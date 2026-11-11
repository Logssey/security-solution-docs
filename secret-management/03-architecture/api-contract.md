# Secret API 계약

기준: [PRD](../01-requirements/PRD.md), [데이터 계약](data-contract.md), [데이터베이스 스키마](database-schema.md), [워크플로 상태](../02-functional-design/workflow-and-states.md), [대응 정책](../02-functional-design/response-policy.md), [자격 증명 유형](../04-reference/credential-types.md), [검증 계약](credential-validation.md), [심각도 평가 계약](risk-assessment-contract.md)

Secret 기능은 `/api/v1/secrets` 아래에서 승인된 수집 범위, 키 인벤토리, 위치별 근거와 대응을 제공한다. API는 비동기 작업 등록·조회와 권한 있는 판정·기록을 담당하고 별도 Java 워커가 정기·수동 수집, 탐지와 승인 검증을 실행한다. 서버 시작은 스캔을 실행하지 않는다. 이 문서는 제품의 단일 API 명세이며 구현 진행률이나 배포 상태를 나타내지 않는다.

## 1. 공통 계약

모든 조회·변경은 공통 대시보드 세션 인증을 사용한다. POST·PATCH·PUT·DELETE는 CSRF와 서버 측 역할·연결·위치 권한을 검사한다. `ADMIN`이라는 로그인 역할만으로 `SECURITY`, `REMEDIATOR`, `OPERATIONS`를 부여하지 않는다. 기존 계정을 식별 주체로 사용하며 공개 회원가입이나 별도 로그인 방식을 도입하지 않는다. 공통 형식은 [대시보드 API 명세](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)를 따른다.

응답 필드는 camelCase, enum은 대문자, 시각은 UTC ISO 8601이다. 키 원문·연결 비밀·HMAC 지문·전체 공급자 응답·비신뢰 원천 문맥을 요청·응답·오류·일반 로그에 넣지 않는다. 제한된 증거 참조는 `TICKET:SEC-123`, `CHANGE:OPS-123`, `POLICY:SECRET-12`처럼 서버가 등록한 유형과 식별 형식만 허용한다. 임의 URL·자유 메모·파일 업로드는 받지 않는다. 권한 없는 존재·수·활동 시각을 노출하지 않는다.

조회 scope와 unit은 `workbench/Mixed`, `currentCases/ResponseCase`, `inventory/CredentialSet`으로 명시한다. 대응 건은 ResponseCase, 실행 이력은 ScanRun이며 HISTORY 스냅샷을 사용한다. 일반 화면은 `origin=ALL`로 전체 허용 자료를 조회하고 기존 명시적 origin 조회는 호환한다. `source`는 발견 플랫폼, `provider`는 키 발급 서비스다. 실행 이력의 provider는 수집 플랫폼이다. 자원 환경은 `ALL|PROD|DEV|UNKNOWN`이며 ALL은 UNKNOWN을 포함한다.

페이지는 0부터, 크기는 1–100(기본 20)이다. 목록은 `{items,total,page,size,filters,scope,unit,asOf,snapshotId,expiresAt}`와 해당 조회의 확인 범위를 반환한다. `total`은 같은 권한·필터의 전체 대상 수다. 목록·카드·분포의 같은 스냅샷은 같은 단위·필터에서 일치한다. workbench 최상위 `total`은 null이며 서로 다른 단위의 지표를 합산하지 않는다. 필터·시간·스냅샷은 5.6·5.12절을 따른다.

변경은 인증 사용자로 수행자를 채우고 최신 `expectedVersion` 또는 명시된 대상별 버전을 검사한다. 저장 결과·버전·안전한 감사·멱등 결과를 같은 트랜잭션에 확정한다. 멱등 범위는 요청자·명령·키이며 동일 키·다른 본문은 409, 같은 본문의 재시도는 같은 결과다. 읽기 스냅샷은 쓰기 허가 근거가 아니다. 변경 성공 시 `requiresSnapshotRefresh=true`를 반환한다.

오류 필수 필드는 `code`, `message`, 선택 필드는 안전한 `requestId`, `retryable`, `details`다. 400 `INVALID_INPUT`, 401 `UNAUTHENTICATED`, 403 `FORBIDDEN`, 404 `NOT_FOUND`, 405 `METHOD_NOT_ALLOWED`, 409 `CONFLICT`, 422 `EVIDENCE_REQUIRED`, 429 `RATE_LIMITED`, 5xx 서버 장애를 사용한다. `SNAPSHOT_EXPIRED`, `STALE_REVIEW_CONTEXT`, `CLOSURE_BLOCKED` 같은 충돌 사유는 `details.reasonCode`에 둔다. 공급자 401·403·404는 대시보드 인증 오류와 구분해 안전한 검증 사유로 기록한다.

### 1.1. 통합 자료 조회와 검색

제품의 일반 화면은 읽기 선택 `origin=ALL`을 명시한다. ALL은 현재 열람 가능한 PRODUCTION·TEST·UNCLASSIFIED 자료를 포함하며 실제 scope·위치·대응의 origin을 바꾸지 않는다. 기존 API의 명시적 분류 조회와 생략 시 기본값은 호환 경로다. ALL은 변경 명령의 actual origin으로 허용하지 않는다.

고유 키 응답은 `readOrigin`, 실제 허용 분류의 `origins[]`, 단일 분류일 때의 `origin`을 제공한다. 혼합이면 origin은 null이다. `currentCaseId`는 가시적인 현재 대응이 하나일 때만 제공하고 복수 대응은 `casesByOrigin`의 실제 caseId·origin·상태로 구분한다. 위치 응답의 item.origin은 실제 분류다. 단일 분류의 보관·폐기 근거를 다른 분류의 위치에 적용하지 않는다. 여러 분류가 각각 관리 조건을 충족한 경우 MANAGED_LOCATIONS 읽기 분류로 보관·폐기 확인을 설명한다.

인벤토리·대응 목록·확인 대기열은 `q`를 지원한다. 서버는 trim·소문자 정규화·100자 상한·제어 문자와 자격증명 모양 입력 검사를 적용한다. 원문 형태 입력은 반영·echo·스냅샷 생성 전에 거부한다. 검색은 현재 ACL과 다른 조건에 맞는 같은 위치의 허용된 표시 정보에서 적용한 뒤 고유 키·대응·위치를 집계하고 페이지를 나눈다. 가려진 원천 경로·다른 조건의 위치·키 원문을 검색 근거로 사용하지 않는다. overview·검사 누락·대상 설정 대기열처럼 검색 미지원인 계약은 q를 명확히 거부한다.

CURRENT_CASES·INVENTORY·WORKBENCH는 origin·q와 전체 정규화 필터를 스냅샷에 결합한다. history·배정 상태의 기존 LIVE 대응 목록은 LIVE 사실과 같은 검색·조건의 total을 제공한다. 목록에서 스냅샷을 바꾸어 재사용하지 않는다. 상세 기준 버전에는 실제 origin과 해당 revision을 포함한다. CSV는 q를 포함한 동일 INVENTORY 스냅샷의 전체 페이지를 내보내고 사용자용 분류 표시를 포함하지 않는다.

현재 활성 연결·enabled 등록 범위·동일 revision의 실제 current finding으로 수집 근거가 확인되면 SECURITY가 세 actual origin 모두 근거와 CAS를 통해 판단·대응 생성할 수 있다. 기존 PRODUCTION·TEST도 분류 원장 부재 자체를 수집 미승인으로 취급하지 않는다. 정책 부재는 보관 판단 UNKNOWN이다. 자동 NORMAL은 승인된 정책 binding을 요구하고 합성·보호 원천의 인증 가드는 적용한다. 내부 actual origin UNCLASSIFIED의 대응·수명주기·해결도 원본 값으로 연계한다.

보관 정책만 바꾸는 요청은 현재 actual origin을 그대로 전송한다. UNCLASSIFIED의 현재값과 다른 분류로 자동 보정하지 않는다. 정책 승인에는 기존의 독립 검토·정책 버전·scope revision을 적용하고 같은 origin의 승인 원장을 정책 binding에 연결한다. UNCLASSIFIED의 origin_set_by·origin_set_at은 null을 유지하고 기존 분류의 원본 수행자·시각을 덮어쓰지 않는다.

## 2. 조회 자원

| Method·경로 | 목적·핵심 응답 |
|---|---|
| `GET /api/v1/secrets` | 고유 키 인벤토리. `origin`, `contentKind`, `workBucket`, `source`, `provider`, `environment`, `componentState`, `credentialType`, `validity`, `severity`, 페이지·스냅샷 필터. 구성·마스킹 이름·허용 대표 위치·위치 수·검증 상태와 이유. 5.7·5.12절 적용 |
| `GET /api/v1/secrets/{id}` | 호출자에게 허용된 키 요약. 유형·구성·유효성·심각도·casesByOrigin. IAM 관측은 `iamMetadata`에 분리 |
| `GET /api/v1/secrets/{id}/locations` | 위치별 독립 사실·업무·해결 확인. 5.7절의 필터·detailVersion·페이지 계약 |
| `GET /api/v1/secrets/summary` | Mixed 현황 지표와 하위 caseSummary. 5.6절 |
| `GET /api/v1/secrets/matrix` | 동일 currentCases 스냅샷의 심각도·유효성 교차 분포와 미평가·확인 범위 |
| `GET /api/v1/secrets/cases` | `origin`, `scope=currentCases\|history`, `stages`, `assignmentStatus`, `assignee`, 소스·환경·발급자·심각도·검증·예외·변경 기간 필터. 현재·과거 대응 건은 단위를 유지 |
| `GET /api/v1/secrets/cases/{caseId}` | 마스킹 키·개설 결정·단계·건 조정 담당자·버전·권한 있는 위치·capabilities. 상세는 최신 readAt와 changedSinceSnapshot 제공 |
| `GET /api/v1/secrets/cases/{caseId}/guide` | 외부 호출 없는 발급자·단계별 기본 안내와 선택 AI의 현재 사용 조건. 3.3절 |
| `GET /api/v1/secrets/cases/{caseId}/activities` | 허용된 담당·조치·체크리스트·재검사·예외·해결의 시간순 이력. happenedAt DESC, activityId DESC, 페이지 분리 |
| `GET /api/v1/secrets/assignees?caseId=&occurrenceId=` | 해당 위치 연결의 적격 REMEDIATOR와 활성 팀 소속. SECURITY 권한과 후보 대상의 역할을 서버에서 확인 |
| `GET /api/v1/secrets/cases/{caseId}/revalidations` | 허용 위치의 재검사 이력. revalidationId·occurrenceId·runId·canViewRun·jobStatus·result·failureCode·requestedAt·finishedAt. 실행 링크는 canViewRun=true일 때만 제공 |
| `GET /api/v1/secrets/cases/{caseId}/closure-checks` | 최신 근거의 해결 검사·버전·허용 확인 수준. 5.3·5.9·5.11절 |
| `GET /api/v1/secrets/connections` | 사용자가 수동 실행할 수 있는 활성 연결의 ID·안전한 표시명·범위 수. 연결 비밀 제외 |
| `GET /api/v1/secrets/connections/manage` | 연결 운영·승인 상태와 읽기 결과. 5.8절 |
| `GET /api/v1/secrets/runs`, `GET /api/v1/secrets/runs/{runId}` | 실행 당시 범위·단계·결과·검사 누락. 실행 목록의 단위 ScanRun·HISTORY 스냅샷. 5.4절 |
| `GET /api/v1/secrets/review-queue`, `GET /api/v1/secrets/occurrences/{occurrenceId}/review-context` | 역할과 단위가 고정된 대기열·위치별 판단 근거. 5.2절 |
| `GET /api/v1/secrets/runs/{runId}/progress` | 실제 단계·구조화 실행 이벤트·cursor·시도·보존 상태. [스캔 실행 계약](scan-execution-contract.md) |
| `GET /api/v1/secrets/coverage-gaps` | 현재 승인 revision의 검사 누락 고유 범위. 5.8절 |

`assignmentStatus=UNASSIGNED|TEAM_ONLY|PERSONAL`은 팀·개인 모두 없음, 팀만 있음, 개인 지정을 뜻한다. 필터는 서버 바인딩으로 AND 결합하고 알 수 없는 enum·날짜·필터는 400이다. 키 구성 `CONFIRMED|CANDIDATE|METADATA_ONLY`, 유효성 `VALID|INVALID|UNKNOWN|UNCHECKED`, 심각도 `UNASSESSED`와 환경 `UNKNOWN`은 독립적이다. 검증 기록이 없으면 UNCHECKED와 사유를 반환한다. IAM의 ACTIVE나 사용 기록 부재는 VALID·미사용·노출·심각도의 근거가 아니다. AWS 목록 자원은 안전한 범위별 수로만 제공하며 값 발견으로 집계하지 않는다.

## 3. 실행과 대응 명령

### 3.1. 수동 스캔과 연결 테스트

`POST /api/v1/secrets/runs`는 `{connectionId,idempotencyKey}`로 OPERATIONS 또는 명시적으로 실행 권한이 부여된 사용자의 활성 승인 범위를 비동기 예약한다. 지원 콘텐츠·범위·설정 capability를 서버에서 확인하고 미지원 범위는 409와 안전한 사유로 거부한다. `202 Accepted`, `Location: /api/v1/secrets/runs/{runId}`, `{runId,status:"PENDING"}`를 반환한다. 수동 실행은 6시간 정기 예정 시각을 바꾸지 않으며 API는 원천 콘텐츠를 읽거나 도구를 실행하지 않는다. 워커가 비활성이면 작업은 대기로 남는다. 원문 대신 내부 ID만 큐에 넣는다.

AWS 목록 메타데이터는 탐지·검증을 SKIPPED와 해당 없음 사유로 기록한다. 정확한 단일 AWS 값 범위는 별도 승인·`CURRENT_VALUE_ONLY`·값 읽기 opt-in이 필요하며, 검증도 독립 승인 설정을 따른다. 선택 비활성은 안전한 사유와 SKIPPED로 표시하고 VALID나 검사 성공으로 치환하지 않는다. 필수 검증 실패는 PARTIAL이다.

`POST /api/v1/secrets/connections/{connectionId}/test`는 본문 없이 OPERATIONS와 CSRF로 호출한다. 제한 시간 내 신원·설치 확인 후 동기 200 `{connectionId,provider,checkKind:"IDENTITY_OR_INSTALLATION",status,reasonCode,checkedAt}`를 반환한다. 상태는 CONNECTED·FAILED·UNAVAILABLE이며 성공 사유는 null이다. 안전한 실패 코드는 INVALID_APPROVED_SCOPE, AWS_ACCOUNT_MISMATCH, AWS_ACCESS_DENIED, AWS_RATE_LIMITED, AWS_UNAVAILABLE, GITHUB_APP_UNAVAILABLE, GITHUB_ACCESS_DENIED, GITHUB_RATE_LIMITED, GITHUB_UNAVAILABLE다. 연결 없음은 404, 일시중지·미지원은 409, 권한 없음은 403이다.

AWS는 승인 계정과 STS GetCallerIdentity 신원을 비교하되 개별 IAM·Secrets Manager·SSM·S3 권한이나 값을 읽지 않는다. GitHub는 승인 저장소 한 곳으로 제한한 contents:read 설치 토큰과 반환 권한을 확인하되 파일을 읽지 않는다. 테스트는 스캔·연결 상태 변경·전체 수집 성공·발견 키 검증을 만들지 않는다. auth_ref·계정·저장소명·원천 승인 참조·공급자 응답을 노출하지 않고 안전한 probe 이력만 남긴다.

### 3.2. 위치별 조치·재검사

건 조정 담당자는 SECURITY가 `PATCH /cases/{caseId}/assignment`의 `{team,assigneeUserId,reasonCode,expectedVersion}`으로 지정한다. 사유는 INITIAL_ASSIGNMENT·OWNERSHIP_CHANGE·TEAM_TRANSFER·UNASSIGNED·OTHER이며 자유 문장은 받지 않는다. 위치 담당자는 5.3절의 별도 명령으로 지정하고 다른 연결 권한으로 확대하지 않는다.

| Method·경로 (`/api/v1/secrets` 아래) | 요청·서버 조건 |
|---|---|
| `POST /cases/{caseId}/actions` | `{actionType,occurrenceId,performedAt,evidenceRef,expectedCaseVersion,expectedOccurrenceVersion,idempotencyKey}`. 할당된 위치의 REMEDIATOR가 외부 조치를 추가 전용 기록. 첫 실질 조치로 위치와 건 OPEN→IN_PROGRESS를 함께 전이 |
| `POST /cases/{caseId}/occurrences/{occurrenceId}/complete-actions` | 기대 건·위치 버전과 멱등 키. 현재 조치 주기에서 ROTATED·REVOKED·REMOVED_FROM_SOURCE 중 하나 이상 필요. PERMISSION_REDUCED만으로 불가. 위치를 REVALIDATION_PENDING으로 변경하고 남은 위치로 건 단계 산출 |
| `POST /cases/{caseId}/revalidations` | `{occurrenceId,expectedCaseVersion,expectedOccurrenceVersion,idempotencyKey}`. 조치 기록 마침·원천 capability·할당 권한을 확인하고 승인 위치의 비동기 작업 등록. 202와 재검사 이력 형태 반환 |
| `POST /cases/{caseId}/revalidation-resume` | `{occurrenceId,expectedCaseVersion,expectedOccurrenceVersion,reasonCode:"REVALIDATION_PRESENT",idempotencyKey}`. PRESENT 재검사 결과를 가진 위치에 새 조치 주기·이력을 만들고 IN_PROGRESS 전이 |
| `PATCH /cases/{caseId}/checklist/{itemId}` | `{completed,guideVersion,evidenceRef,expectedCaseVersion}`. 허용 담당자가 보조 체크 기록. 완료는 티켓·변경 참조를 요구하되 체크만으로 조치 마침·해결 불가 |
| `GET /cases/{caseId}/exceptions` | 현재·과거 예외, 허용 위치 ID, `currentlyApplicable`, `canApprove`, `allowedReasonCodes`, `allowedRevokeReasonCodes`, `eligibleOccurrences`, `maxDurationDays`, `fullyRiskAccepted`, `restrictedCoverage`, `blockedReasonCodes`. 기본 OFF에서는 canApprove=false, RISK_ACCEPTANCE_DISABLED, maxDurationDays=7이며 과거 기록·취소는 유지. 비활성 정책으로 적용되지 않는 과거 ACTIVE는 POLICY_DISABLED, currentlyApplicable=false이며 만료·범위 변경 상태는 별도 보존 |
| `POST /cases/{caseId}/exceptions` | 위치 집합·사유 코드·안전한 근거·만료 시각·기대 버전. 현재 권한 검사 후 기본 OFF의 HTTP 422 RISK_ACCEPTANCE_DISABLED로 거부. 정책 활성화와 자원 책임자·독립 SECURITY 검토를 모두 갖추기 전 단독 승인 경로 제공 금지 |
| `POST /cases/{caseId}/exceptions/revoke` | 예외 ID·취소 사유·기대 버전. 만료·취소 뒤 남은 위험에 맞게 조치 재개 |

위험 수용 OFF에서는 `eligibleOccurrences=[]`, `canApprove=false`, `fullyRiskAccepted=false`, `blockedReasonCodes=[RISK_ACCEPTANCE_DISABLED]`를 반환한다. `closure-checks.capabilities.canApproveException`도 false다. 현재 권한 검사를 먼저 수행하므로 다른 사용자의 건 존재를 비활성 오류로 노출하지 않는다. 과거 예외 취소는 기존 권한·버전 검사를 유지한다.
| `POST /cases/{caseId}/guide` | 본문 없이 선택 AI를 명시적으로 요청. 허용 조건·설정·응답 검사를 통과하지 못하면 기본 안내 반환. 상태·DB 변경 없음. 3.3절 |

외부 조치 종류는 ROTATED·REVOKED·REMOVED_FROM_SOURCE·PERMISSION_REDUCED다. 외부 수행 시각은 요청보다 최대 5분 뒤부터 최근 30일 사이만 허용한다. 한 위치에 여러 기록을 남길 수 있고 PERMISSION_REDUCED는 임시 완화일 뿐 제거·폐기 증거를 대신하지 않는다. 체크리스트 기본 코드는 IDENTIFY_OWNER·ROTATE_OR_REVOKE·REMOVE_EXPOSURE·CHECK_ACCESS_LOGS이며 상태는 TODO·DONE이다. 체크리스트는 조치 주기·재검사·해결의 필수 사실을 대체하지 않는다.

재검사 작업 상태는 PENDING·LEASED·SUCCEEDED·FAILED·CANCELLED, 결과는 PENDING·PRESENT·MISSING·UNKNOWN이다. MISSING은 마지막 실질 조치 이후 같은 승인 범위 revision을 정상 읽고 필수 탐지가 성공했으며 설정·엔진·HMAC 버전이 비교 가능한 경우에만 인정한다. 불완전·실패·비교 불가는 UNKNOWN과 안전한 사유다. 한 위치 결과를 전체 원천·과거 Git 이력·다른 복사본의 해소로 확장하지 않는다. 지원하지 않는 원천의 재검사는 capability=false로 등록을 거부한다.

### 3.3. 기본 대응 안내와 선택 AI

가이드 응답은 `{version,source:"DEFAULT|GROQ",title,steps:[{code,title,description}],notice,readiness}`다. `version=secret-response-guide-v2`는 발급자·진행 단계별 안내의 버전이며 심각도 정책과 저장된 체크리스트 `guideVersion=S2_V1`을 변경하지 않는다. 안내는 발급 서비스의 키 교체·폐기, 발견 위치의 원문 제거·안전한 저장 전환, 조치 기록과 재검사를 구분한다. 재검사 대기·해결 상태에서는 기존 근거 확인을 안내하며 새 조치나 완료 사실을 생성하지 않는다.

`GET /cases/{caseId}/guide`는 현재 건을 읽을 수 있는 사용자에게 `source=DEFAULT`와 사용 조건을 반환한다. 외부 호출·AI 예산 소비·이전 AI 결과 재생이 없으며 `aiRequested=false,providerCalled=false,cached=false`다. `POST`는 기존 세션·CSRF·대상 권한으로 선택 AI를 명시적으로 요청하고, 제한·비활성·설정 부족·공급자 실패에도 사용 가능한 기본 안내를 반환한다. 두 경로 모두 키·건 상태·체크리스트·감사를 변경하지 않는다.

`readiness={status,reasonCode,aiAvailable,aiRequested,providerCalled,cached}`다. 상태는 `READY|UNAVAILABLE|GENERATED|FALLBACK`이며 `aiAvailable`은 현재 대상·로컬 설정이 요청을 허용한다는 뜻이다. 공급자 인증·할당량·네트워크 성공을 보장하지 않는다. `providerCalled`는 해당 요청의 공급자 통신 시도, `cached`는 이전 결과 또는 실패 대기 상태의 재사용 여부다. 수용된 AI 선택만 `source=GROQ`이며 캐시 재사용은 `CACHE_HIT,providerCalled=false`다. 실패 캐시는 원래 실패 사유와 `source=DEFAULT,cached=true,providerCalled=false`를 유지하므로 AI 결과 생성으로 표시하지 않는다.

AI는 AWS·GitHub 발급자, TEST 건, 현재 유효한 키별 합성 표본 근거, 실키 확인 근거 부재, 모든 관련 위치의 열람 권한, 서버 opt-in과 유효한 키 설정을 모두 요구한다. 저장소 이름이나 TEST 표지만으로 허용하지 않으며 캐시 사용 전에도 대상 조건을 다시 검사한다. 사유는 `TEST_ONLY|SYNTHETIC_FIXTURE_REQUIRED|ALL_LOCATIONS_ACCESS_REQUIRED|UNSUPPORTED_ISSUER`, `DISABLED|KEY_MISSING|KEY_INVALID|ENDPOINT_NOT_ALLOWED|MODEL_NOT_ALLOWED|CONFIGURATION_INVALID`, `REQUEST_IN_PROGRESS|CALL_BUDGET_EXHAUSTED` 및 `PROVIDER_TIMEOUT|PROVIDER_REJECTED|PROVIDER_RATE_LIMITED|PROVIDER_UNAVAILABLE|PROVIDER_INVALID_RESPONSE|PROVIDER_RESPONSE_TOO_LARGE|PROVIDER_INTERRUPTED`로 구분한다. 공급자 원문 오류와 설정값은 반환하지 않는다. 전송·응답·호출량 경계는 [보안 실행 정책](security-and-runtime.md#선택-ai-가이드)을 따른다.

### 3.4. 발견 키 권한 자료와 담당자 확인

다음 경로는 `/api/v1/secrets/occurrences/{occurrenceId}/permission-assistance` 아래의 기존 위치별 자원이다. 조회는 현재 위치 열람 권한, 요청·확인·제외는 해당 연결 SECURITY·CSRF와 최신 context·근거 revision을 검사한다. 키 원문·principal ARN·계정 식별자·사용자 이름·정책 원문·임의 endpoint는 입력·응답에 포함하지 않는다.

| Method·하위 경로 | 계약 |
|---|---|
| `GET` | 현재 `identity`, `collectorOptions`, `latestRequest`, `latestObservation`, `supplementation`, `capabilities`와 `requestReadiness` 반환. 외부 호출 없음 |
| `POST /requests` | `{expectedContextVersion,idempotencyKey,collectorConnectionId?}`로 `ENRICH/PERMISSION_ASSISTANCE` 작업 등록. 202와 requestId·jobId·상태. 수집 연결이 하나로 확정되지 않으면 적격 연결을 명시 |
| `GET /requests/{requestId}` | 같은 위치의 작업 상태·안전한 실패 사유·관측 후보 조회 |
| `GET /observations/{observationId}/preview` | 현재 contextVersion·evidenceRevision, 후보·범위 한계, 실제 사용 가능한 부모 근거와 확인/제외 조건 조회 |
| `POST /observations/{observationId}/confirm` | `{expectedContextVersion,expectedEvidenceRevision,idempotencyKey,attestation:true,evidenceRef,observedAt,validUntil,facts,sourceRefs,applicableFrom?,applicableUntil?}`. 별도 자료를 검토한 사람의 수동 근거 제출 |
| `POST /observations/{observationId}/reject` | `{expectedContextVersion,idempotencyKey,reasonCode}`. 사유는 NOT_APPLICABLE·INSUFFICIENT_EVIDENCE·IDENTITY_MISMATCH |

`requestReadiness={nextAction,checks:[{code,status,action}]}`는 기능 설정·SECURITY 역할·지원 유형·완전한 키 구성·원천 접근·현재 키 주체·승인 조회 연결의 기존 요청 조건을 설명한다. 상태는 `READY|BLOCKED|WAITING|UNSUPPORTED`다. 실제 최신 검증에 결합된 주체가 없으면 `KEY_IDENTITY`는 BLOCKED, `COLLECTOR`는 WAITING이며 먼저 `VALIDATE_CREDENTIAL`을 안내한다. 이때 적격 연결을 아직 대조할 수 없으므로 `APPROVED_COLLECTOR_REQUIRED`로 단정하지 않는다. 주체 확인 뒤 실제 승인 연결이 없을 때 `CHOOSE_APPROVED_COLLECTOR`를 안내한다. 준비 완료는 권한 수집·확정 완료가 아니다.

AWS는 확인한 주체와 같은 계정의 승인 연결로 결합된 사용자·역할 정책만 읽는다. GitHub는 발견 PAT의 실제 검증에서 얻은 사용자·정규화된 scope를 사용하며 수집 App의 권한을 대신하지 않는다. 자동 관측은 PARTIAL 후보이며 전체 유효 권한·영향·심각도를 확정하지 않는다. 확인에는 관측에 있는 capability와 기존 `PERMISSION_MAPPING`의 네 coverage에 대한 COMPLETE 근거, 현재 키 주체를 결합한 실제 검증 시도, 승인된 자산 분류, 수락된 조사 자료 등 적격 `sourceRefs`를 요구한다. 체크박스나 후보 이름 변경으로 이를 대신하지 못한다. 사람의 수동 근거만 기존 평가·보류·독립 검토 절차로 전달하며 [심각도 평가 계약](risk-assessment-contract.md)의 v2 정책과 신규 720개 정책 보류를 변경하지 않는다.

Notion의 인증 주체는 현재 VALID 시도에 결합한 봇·소유 범위·워크스페이스 확인 수준으로 제공한다. 주체가 있어도 해당 권한 관측기가 등록되지 않았으면 권한 자료의 readiness는 `UNSUPPORTED`와 안전한 지원 사유를 반환한다. 사용자에게 불필요한 조회 연결 선택을 요구하지 않는다. 기본 인증의 실행 가능성은 별도로 계산한다.

Notion 발급 토큰의 권한 관측기는 등록 전 `UNSUPPORTED`다. 등록된 adapter가 있을 때만 승인 페이지 정보·본문 읽기를 기존 permission-assistance의 `ENRICH/PERMISSION_ASSISTANCE`로 등록할 수 있다. 대상은 현재 권한으로 열람 가능한 승인 페이지·블록의 서버 등록 식별자에서 선택하고 context에 대상·원천·승인 revision을 고정한다. 기본 인증 승인으로 추가 대상을 허용하지 않는다. 읽기 성공은 해당 대상·동작·시각의 PARTIAL 관측이며, 설정상 capability·전체 접근 목록·실효 쓰기 권한·토큰 만료 근거는 별도로 표시한다. 담당자 확인은 기존 권한 자료의 provenance·완전성·자산·v2 매핑 조건을 적용한다.

## 4. 공유 데이터 경계

인증 주체는 공통 dashboard_users를 참조하고 Secret 역할·활성 팀·연결/위치 권한을 별도로 검사한다. SECURITY는 근거 검토·판정·조정·예외·해결, REMEDIATOR는 할당 위치의 외부 조치·재검사, OPERATIONS는 연결·승인 범위 운영을 수행한다. 화면 capability는 보조 정보이며 모든 명령은 서버에서 다시 검사한다.

일반 탐지 detection_events와 Secret 전용 모델은 분리한다. 통합 위험 조회의 Secret 단위는 현재 열람 가능한 actual 세 분류의 ResponseCase 한 건이며 metadata origin은 ALL이다. 같은 caseId의 여러 위치를 중복 합산하지 않고 고유 키·Finding·위치 수를 대응 수에 더하지 않는다. Secret 기록을 일반 탐지에 복제 적재하지 않고 API에서 원본 영역·ID·단위를 명시해 합성한다. 일반 탐지와 Secret의 기준시점·필터는 공통 통합 계약으로 고정하고, 미평가·UNKNOWN을 점수나 임의 기본값으로 바꾸지 않는다. 공통 통합 URL·응답의 소유권은 대시보드 설계에 둔다.

## 5. 위치별 판단·증거·집계

### 5.1. 보관 정책 승인과 출처 교정 호환

일반 연결 화면은 수집 대상·보관 정책을 제공하고 운영·시험 분류 입력은 제공하지 않는다. 보관 정책 변경은 현재 actual origin을 유지한 `classification-preview`·`classification-proposals` 호환 계약을 사용하며 제안자와 다른 적격 SECURITY가 검토한다. 현재 UNCLASSIFIED인 대상의 정책-only 제안도 같은 값으로 허용한다. 기존 명시적 분류 교정 API의 source-label-v2 직접 기록·scope-change-v1 영향별 승인·합성 대상 제한은 보존한다. 명령 시점에 현재 권한·범위·정책·근거로 승인 방식을 재검사하고, 분류 교정만으로 기존 노출·건·경고를 자동 종료하지 않는다.

| Method | 경로 | 요청 또는 응답의 핵심 |
|---|---|---|
| GET | `/api/v1/secrets/scopes/{scopeId}/classification` | 현재 `origin`, `originVersion`, nullable `storagePolicyId`, `storagePolicyVersion`, 결정 상태·방식·시각·수행자, 정책 요약·변경 이력, `setupReasonCodes[]`, `taskGuidance`, `pendingProposal`의 허용 검토 담당·요청 시각. `approved_ref`·계정·자원명 제외 |
| PATCH | `/api/v1/secrets/scopes/{scopeId}/classification` | 호환 API의 명시적 분류 교정. 일반 화면에는 입력을 제공하지 않음. `{origin,expectedScopeVersion,idempotencyKey}`. 해당 대상 SECURITY가 직접 저장하며 외부 티켓 불필요. `source-label-v2` 변경 이력을 자동 생성. 정책·수집 범위·노출/해결 판정은 변경하지 않음 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-preview` | `{origin,storagePolicyId?,storagePolicyVersion?,expectedScopeVersion}`. 부작용 없는 미리보기. `approvalMode`, `approvalReasonCodes[]`, 규칙 버전, 허용 영향 요약·검토 담당 후보·`taskGuidance` 반환. 공급자 호출·제안 생성·승인·스캔 없음 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals` | `{origin,storagePolicyId?,storagePolicyVersion?,reasonCode,evidenceRef,expectedScopeVersion,expectedApprovalMode,reviewerUserId?,idempotencyKey}`. 해당 범위 SECURITY 또는 OPERATIONS. 정책 미설정은 두 필드 null과 `POLICY_PENDING`; 보관 UNKNOWN 유지. 서버가 DIRECT이면 제안·결정을 원자 적용하고, INDEPENDENT_REVIEW이면 PENDING_APPROVAL로 보존. 현재 Classification과 `changeResult` 반환 |
| PATCH | `/api/v1/secrets/scopes/{scopeId}/classification-proposals/{proposalId}/assignment` | `{reviewerUserId,expectedProposalVersion,idempotencyKey}`. 해당 범위 SECURITY 또는 제안 권한을 유지한 요청자만 적격·비제안자 SECURITY를 지정. 권한 부여가 아니며 변경 감사·버전 증가. 담당 미지정도 대기 내역에 명시 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals/{proposalId}/approve` | `{expectedScopeVersion,expectedProposalVersion,idempotencyKey}`. 제안자와 다른 현재 범위 SECURITY. 지정 검토자가 있으면 해당 사용자만 승인하며 재배정은 별도 감사. 현재 조건을 재검사하고 범위 버전·추가 전용 결정·영향·집계 무효화를 원자 확정 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals/{proposalId}/reject` | `{reasonCode,expectedProposalVersion}`. `SECURITY`가 거절 사유와 이력을 남기고 이전 승인 분류를 유지. 승인 대기 제안을 활성 분류로 읽지 않음 |

범위 설정 사유가 남은 대상은 `/api/v1/secrets/scopes?classificationStatus=NEEDS_REVIEW`에서 대응·발견 위치 집계와 다른 Scope 단위로 찾는다. 승인 전에도 이전 탐지·실패 기록은 열람할 수 있고 조사 과제를 만들 수 있다. 범위 교정은 과거 대응 건의 출처를 직접 UPDATE하지 않는다. 영향 대상별 이전·새 건의 관계와 승인 근거를 추가 전용으로 기록하고 현재 분류의 새 스냅샷을 만든다. 이전 판정과 대응 건의 출처는 이력으로 보존한다.

`changeResult={effect,proposalId,decisionId,approvalMode,remainingReasonCodes}`에서 `effect=APPLIED|PENDING_REVIEW|UNCHANGED`를 사용한다. DIRECT의 제안은 저장 트랜잭션 안에서 APPROVED로 확정하고 직접 결정자·승인 방식·규칙 버전을 남긴다. 독립 검토 대기는 현재 출처·정책을 바꾸지 않는다. 미리보기와 제출의 승인 방식이 달라지면 409 `APPROVAL_REQUIREMENT_CHANGED`로 재확인을 요구하며 클라이언트의 기대 방식은 권한 근거로 사용하지 않는다. 동일 본문의 멱등 재시도와 의미상 무변경은 기존 결과를 반환하고 결정을 증식하지 않는다.

무변경은 현재 유효한 결정·근거까지 동일한 경우다. 출처 표식만 있고 정식 결정이 없는 LEGACY 범위의 근거 보완을 UNCHANGED로 되돌려 재확인을 막지 않는다. DIRECT에는 reviewerUserId를 받지 않는다. preview는 해당 범위 SECURITY 또는 OPERATIONS의 권한과 안전한 영향만 조회하며 읽기 권한 밖 위치·건·정확한 수를 노출하지 않는다.

`setupReasonCodes[]`는 `POLICY_MISSING`, `POLICY_EXPIRED`, `POLICY_RETIRED`, `POLICY_NOT_YET_VALID`, `POLICY_SCOPE_MISMATCH`, `PENDING_REVIEW`, `CLASSIFICATION_IMPACT_PENDING`의 현재 사유 집합이다. 값 보관 판단 대상이 아닌 메타데이터 범위에는 POLICY_MISSING을 만들지 않는다. NEEDS_REVIEW는 하나 이상의 사유가 남은 고유 범위다. 실제 정책·변경 검토·기존 건 영향에서 같은 집합을 계산한다. 미분류 값·분류 원장 부재 자체는 사유가 아니다. 정책 변경 저장과 남은 재검토 사유를 구별한다. 목록·현황과 갱신한 상세에서 같은 사유·단위·다음 행동을 제공한다.

보관 정책은 `SECURITY`가 `POST /api/v1/secrets/storage-policies/drafts`에 연결·범위·콘텐츠 유형·정확한 경로/접두사 조건, `ALLOW|DENY`, 근거 참조, 유효 기간을 제출하고 **다른** `SECURITY`가 `POST /storage-policies/{policyId}/versions/{version}/approve`로 승인한다. 승인된 버전은 불변이며 `GET /storage-policies?status=ACTIVE`에서 권한에 맞는 안전한 요약만 조회·선택한다. 충돌하는 허용·금지 규칙은 활성화 전에 거부하고 평가 중 범위·버전 불일치나 만료는 `UNKNOWN`으로 남긴다. 정책 폐기는 새 승인 버전 또는 `POST /storage-policies/{policyId}/retire`의 사유·승인 기록으로 처리하며 기존 판정에 재검토 표식을 만든다. 화면은 보호된 `approved_ref`와 규칙의 민감한 전체 경로를 일반 사용자에게 노출하지 않는다. 범위 승인 API는 정책을 지정했다면 **활성 정책 버전**만 받으며, 정책 미설정 범위도 수집 승인은 가능하지만 보관 판정은 `UNKNOWN`·정책 작성 과제로 남는다. 정책 등록·승인·바인딩 모두 감사 기록과 버전 충돌 검사를 수행한다.

### 5.2. 검토 대기열과 근거

검토 대기열은 `GET /api/v1/secrets/review-queue?bucket=ACTION_REQUIRED|CURRENT_CASES|REVIEW_REQUIRED|NEEDS_DECISION|NORMAL_INVENTORY|NEEDS_CASE_LINK&origin=ALL|PRODUCTION|TEST|UNCLASSIFIED&q={query}&page={page}&size={size}`로 조회한다. `bucket`, `origin`, `page`, `size`는 허용 목록으로 검증하고 `ACTION_REQUIRED`는 범위별 `SECURITY` 또는 해당 위치에 할당되고 연결의 `REMEDIATOR` 권한을 가진 사용자에게 허용한다. 담당자 조회는 본인에게 허용된 위치가 있는 고유 건만 반환하며 다른 위치의 근거·수·담당자를 확장 공개하지 않는다. `REVIEW_REQUIRED`, `NEEDS_DECISION`, `NORMAL_INVENTORY`, `NEEDS_CASE_LINK`는 범위별 `SECURITY` 권한을 요구한다. 응답은 `{items,total,page,size,unit,asOf,partialCoverage}`이며 `ACTION_REQUIRED`는 남은 작업이 있는 `OPEN|IN_PROGRESS|REVALIDATION_PENDING` 현재 `ResponseCase`, `REVIEW_REQUIRED`는 판정과 대응 연결이 필요한 occurrenceId 합집합이며 대응 연결 행동을 우선한다. `NEEDS_DECISION`은 `UNKNOWN`·재검토 필요 `Occurrence`, `NORMAL_INVENTORY`는 `CredentialSet`, `NEEDS_CASE_LINK`는 유효한 비정상 판정이 있으나 현재 건에 연결되지 않은 `ABNORMAL` `Occurrence`가 단위다. 해결 건과 현재 정책상 적격인 전체 위험 수용 건만 `ACTION_REQUIRED`에서 제외한다. 위험 수용 OFF에서는 과거 예외로 미해결 건을 숨기지 않는다. 서로 다른 단위의 `total`을 더해 고유 키 수로 표시하지 않는다. 항목에는 `credentialSetId`, 선택 가능한 `occurrenceId`, `caseId`, 안전한 키 이름, 구성·존재 상태, 출처, 노출 판정·심각도·검증 각각의 상태, 마지막 성공·최근 시도, 정책 버전, 근거의 확인 수준, 누락 사유, 담당자·기한, 위치 수, `whyHere`, `nextAction`을 필드별 열람 권한 안에서 제공한다. `whyHere`와 `nextAction`은 사용자 자유문이나 생성형 AI 문장이 아니라 서버가 안전한 코드와 템플릿으로 만든다. 범위별 검사 누락은 키 대기열 수에 섞지 않고 `GET /api/v1/secrets/coverage-gaps`에서 `unit=Scope`로 제공한다.

`NORMAL_INVENTORY`는 5.6절의 인벤토리 적격 조건으로 계산한다. `inventorySubtype=AUTHORIZED_STORAGE|METADATA_ONLY|RETIRED_HISTORY|MANAGED_LOCATIONS`를 반환하며, 전체 인벤토리는 이 작업 대기열보다 넓은 집합이다. `UNCLASSIFIED` 자체는 포함·제외 근거가 아니고 해당 위치의 현재 정책·Finding·대응·수명주기 근거를 검사한다.

`GET /api/v1/secrets/occurrences/{occurrenceId}/review-context`는 화면의 단일 근거 카드다. 구성·발급자·원천 유형, 권한에 맞는 위치 표시, 해당 Finding의 룰·버전·탐지 시각, 현재 존재와 마지막 성공 검사/최근 시도, 범위 출처·보관 정책 ID/버전·적용 이유, 검증 결과·방법·시각, 확인된 주체·권한·영향, 동일 키의 열람 가능한 다른 위치, 기존 건, 부족 근거 코드와 후속 과제, `allowedDecisions`, 각각의 예상 버전을 반환한다. 각 사실은 `CONFIRMED|UNKNOWN|NOT_CHECKED`와 출처·관측 시각을 따로 가진다. 응답의 `UNKNOWN`은 무효·정상·낮은 심각도로 치환하지 않는다. 비밀 값, 마스킹 전후 문맥, HMAC 지문, 권한 밖 위치 수와 원천 경로는 반환하지 않는다. `404`와 `403`은 안전한 오류 화면 및 복귀 경로로 처리하고 로딩 중 null을 결론처럼 렌더링하지 않는다.

`GET /api/v1/secrets/investigations?status=OPEN|OVERDUE&owner=ME&page=&size=`와 `GET /investigations/{taskId}`는 위치/범위 구분, 부족 근거 코드, 담당자·기한, 제한된 대상 링크와 최근 활동을 조회한다. `PATCH /investigations/{taskId}/assignment`는 `{assigneeUserId,dueAt,reasonCode,expectedVersion}`, `POST /investigations/{taskId}/evidence`는 `{evidenceRef,evidenceKind,observedAt,expectedVersion}`, `POST /investigations/{taskId}/close`는 `{expectedVersion,resolvingDecisionId|restoredRunScopeId}`를 받는다. 관련 위치가 새 판정으로 확정되거나 누락 범위가 성공적으로 다시 검사된 근거가 없으면 과제를 닫을 수 없다. 범위별 역할을 다시 검사하고, 담당자 변경·기한 초과·증거·종료를 추가 전용 이력에 남긴다. 과제 종료만으로 `NORMAL`, `ABNORMAL` 또는 검사 성공을 만들지 않는다.

`TEST` 위치의 `fixtureStatus=NOT_APPLICABLE|VERIFIED_SYNTHETIC|POSSIBLE_REAL|CONFIRMED_REAL`은 출처 구분과 독립이다. `CANDIDATE`·`METADATA_ONLY`와 현재 유효 Finding이 없는 위치는 `NOT_APPLICABLE`이며 조사·인벤토리에는 남긴다. `VERIFIED_SYNTHETIC`은 승인된 fixture 목록·버전과 **해당 키**가 일치한 근거가 있을 때만 반환한다. `POSSIBLE_REAL`은 `CONFIRMED` 값 후보가 성공한 최신 검사에서 `PRESENT`이고 유효한 Finding이 남았으며 fixture 일치를 확인했지만 합성으로 증명되지 않았다는 뜻이다. 목록 확인 실패도 사유와 함께 이 상태로 둔다. 이는 유효성 `VALID`와 같지 않다. **이 조건이 충족되는 즉시** `GET /api/v1/secrets/cross-environment-alerts`에 `SUSPECTED_REAL` 자격증명 확인 경고를 열고 담당자·기한·권한에 맞는 위치 링크를 제공한다. `testCaseId`는 건이 있는 경우에만, 개설 조건을 충족하지 못한 후보의 `investigationTaskId`는 조사 과제가 있는 경우에 제공한다. 두 ID가 모두 없는 경고는 조사 과제를 원자적으로 만든다. 연결의 기본 적격 SECURITY 담당자·기한 정책이 있는 필드만 채우며 미설정이면 담당자와 기한을 각각 nullable로 두고 경고 생성을 막지 않는다. 배정/기한 미설정을 표시하고 `ASSIGN_INVESTIGATION`으로 안내하며 기한이 없으면 기한 초과 수에 넣지 않는다. 사람이 `NEEDS_MORE_INFO`를 제출하는 경우의 담당자·기한 필수 조건은 그대로다. 같은 CredentialSet의 반복 검사·추가 위치는 열린 경고를 갱신하고 새 경고를 만들지 않는다. 실제 발급자 근거가 들어오면 같은 경고를 `CONFIRMED_REAL`로 승격한다. 경고의 `acknowledged`는 인지 기록이고 조치 완료가 아니다. TEST 원장과 해당 대응 관계는 보존하고 일반 화면에서는 자격증명 여부 확인 업무로 표시한다. 경고 수는 ALL의 `ResponseCase` 수와 별도 단위이며 중복 합산하지 않는다. 합성 여부가 불명확하면 SIMULATION 종료는 허용하지 않는다.

`PATCH /cross-environment-alerts/{alertId}/assignment`는 담당자·기한·기대 버전, `POST /cross-environment-alerts/{alertId}/acknowledge`는 기대 버전과 안전한 인지 사유를 받는다. `POST /cross-environment-alerts/{alertId}/close`는 `SUSPECTED_REAL`을 반증한 키별 fixture 증빙 ID 또는 연결된 모든 최신 Finding의 오탐 교정 ID, **혹은** `CONFIRMED_REAL`의 실제 키 폐기·교체 근거 ID, 관련 건 ID 또는 조사 과제 ID와 기대 버전을 요구하며 `SECURITY`가 서버에서 다시 확인한다. 열린 동일 키 경고의 재검사 결과와 다른 현재 위치도 확인한 뒤 닫고, 새 의심 근거가 나타나면 이전 경고에 연결한 새 경고를 연다. 인지·담당자 배정만으로 경고를 닫지 않는다. 해당 사용자에게 권한 없는 시험 위치의 세부 정보는 운영 경고에도 포함하지 않는다.

### 5.3. 위치별 단일 결정과 대응 건 연결

이 절에서 `ABNORMAL`로 새 건을 만들거나 기존 건에 위치를 붙이는 공통 선행 조건은 **`CONFIRMED` 자격 증명, 마지막 성공 검사에서 `PRESENT`인 위치, 그 위치에 속한 유효 Finding**이다. `CANDIDATE`·`METADATA_ONLY`·현재 존재가 확인되지 않은 위치는 건 개설 명령을 거부하고 검토·인벤토리로 안내한다. 유효성 `UNKNOWN`·`UNCHECKED`는 이 선행 조건과 별도다.

`POST /api/v1/secrets/occurrences/{occurrenceId}/decisions`는 다음 형태의 단일 변경 명령이다. `SECURITY`가 해당 위치·연결에 권한이 있어야 하며, 서버가 최신 키 구성, 현재 존재, Finding 소속, 분류·보관 정책의 적용과 버전, 현재 건, 멱등 키를 같은 트랜잭션에서 확인한다.

```json
{
  "decision": "ABNORMAL",
  "reasonCode": "POLICY_PROHIBITED_LOCATION",
  "findingId": 123,
  "evidenceRefs": ["POLICY:SECRET-12", "TICKET:SEC-123"],
  "expectedOccurrenceVersion": 4,
  "expectedScopeVersion": 2,
  "expectedPolicyVersion": 3,
  "idempotencyKey": "client-generated-request-id"
}
```

`ABNORMAL`에는 현재 위치·형식 탐지와 적용 정책 또는 명시적인 비승인 위치 근거가 필요하다. 유효성 `UNKNOWN`·`UNCHECKED`는 이 결정을 자동 차단하거나 안전 판정으로 바꾸지 않는다. 성공 응답은 `201`(건 생성) 또는 `200`(기존 건에 위치 연결)과 `{decisionId,occurrenceId,exposureStatus,caseId,caseOperation:"CREATED|ATTACHED",caseVersion,queueBucket}`를 반환한다. 기존 건에 같은 위치가 이미 있으면 동일 멱등 요청은 기존 결과를 반환하고 새 이력·건을 만들지 않는다. 다른 출처 건에 붙이지 않으며 해결된 건 뒤 새 노출은 `previousCaseId`로 연결한 새 건을 만든다. 건은 이 보안 결정을 개설 근거로 사용하며 담당자 지정 뒤 첫 실질 외부 조치를 기록할 수 있다. 기존 판정에 없는 근거를 소급 생성하지 않는다.

같은 결정 명령은 선택적인 `exposureRange=BROAD|BOUNDED|UNKNOWN`, `accessEvidenceRefs[]`를 받을 수 있다. 보안 담당자에게 항상 접근 명단을 다시 입력시키지 않고 현재 적격 접근 snapshot을 먼저 제시·재사용한다. BROAD/BOUNDED는 해당 위치·revision의 공개성/접근 주체·완전성·관측 시각을 확인한 구조화 근거가 있어야 하며 없으면 UNKNOWN이다. 원본 정책·접근 ACL 응답·사용자 명단을 일반 화면 또는 요청 본문에 넣지 않는다. 서버가 선택한 typed 참조·정규화 access_snapshot과 exposure_range를 기존 노출 결정 원장에 고정한다. 범위 미확인은 노출 결정·담당자 조치를 일괄 차단하지 않고 심각도의 부족 정보로 남긴다.

이미 `REVALIDATION_PENDING`인 건에 새 `ABNORMAL` 위치가 붙으면 그 위치의 미완료 조치 주기를 열고 건을 **같은 트랜잭션에서 `IN_PROGRESS`로 되돌린다**. `OPEN`·`IN_PROGRESS` 건은 기존 단계를 유지한다. 건 버전·단계 이력·전이 사유와 요약 집계 무효화를 함께 확정한다. 기존 위치의 과거 재검증은 보존하지만 새 위치가 미조치인 동안 해결은 차단한다. 전체 위험 수용 예외가 새 위치를 포함하지 않으면 예외 집계에서 빼고 미해결 단계로 보여준다.

새 건은 `openingDecisionId`를 개설 근거로 보존한다. 같은 노출 판단을 확인하는 별도 재승인이나 빈 검토 행을 요구하지 않는다. 새 결정이 기존 건에 위치를 붙이면 그 위치의 근거 ID를 추가하고 다른 위치의 판정·조치 상태는 바꾸지 않는다.

`POST /api/v1/secrets/occurrences/{occurrenceId}/case-links`는 저장된 유효 `ABNORMAL` 판정이 현재 건과 연결되지 않은 무결성 복구 대상에 한해 판정 ID·버전, 현재 위치/범위 버전, 멱등 키를 받는다. `SECURITY`가 현재 `CONFIRMED` 키·`PRESENT` 위치, 판정 근거의 적용 가능성, 같은 키·대응 출처와 권한을 확인한 뒤 건에 연결하거나 현재 건이 없으면 원래 결정을 개설 근거로 건을 만든다. 근거가 낡았으면 409로 현재 근거를 다시 확인하게 한다. 기존 결정과 복구 이력은 보존하고 중복 위치·건을 만들지 않는다. 신규 판정은 단일 `/decisions` 명령으로 판정과 건 연결을 원자적으로 수행하며 이 복구 경로로 나누어 실행하지 않는다.

`NORMAL`은 `CONFIRMED`·`PRESENT`인 실제 값 위치와 유효한 Finding, 실제 보관 허용 정책·적용 범위와 만료 근거를 요구한다. `CANDIDATE`·`METADATA_ONLY`는 보관 허용 위치로 자동 판정하지 않는다. `FALSE_POSITIVE`는 자격 증명이 아님을 확인한 유형별 근거를 요구하며 대응 건을 새로 만들지 않는다. `FALSE_POSITIVE`는 Finding 교정 이력에만 기록하고 위치 노출을 `NORMAL`로 바꾸지 않는다. 같은 위치에 독립된 유효 Finding이 남으면 그 위치는 검토 대기열에 계속 남는다. `NEEDS_MORE_INFO`는 노출을 `UNKNOWN`으로 유지하고 `missingReasonCode`, `followUpOwnerUserId`, `dueAt`, 안전한 `evidenceRef` 또는 내부 과제 ID를 필수로 받아 대기열에 남긴다. 키 원문·자유 메모는 받지 않는다. 판정 재검토는 기존 이력을 덮지 않고 새 판정으로 추가한다. 정책·범위·Finding이 바뀌어 예상 버전이 오래되면 409 `STALE_REVIEW_CONTEXT`와 안전한 변경 항목을 반환하고 현재 근거를 다시 조회하게 한다. 입력 근거 부족은 422 `EVIDENCE_REQUIRED`, 권한 범위 밖은 403, 대상 없음은 404다. 대기열·건 상세와 판정 결과는 저장 후 같은 새 버전을 가리킨다.

같은 위치의 **모든 최신 Finding**이 `FALSE_POSITIVE`로 교정되고 다른 유효 Finding이 없을 때만 그 위치를 활성 `NEEDS_DECISION`에서 빼며 제외·판정 이력에서는 계속 조회한다. 승인된 오탐 결정은 같은 범위·원천 객체·정확한 객체 버전·위치·엔진/룰 버전·보호된 후보 동일성이 모두 같은 새 Finding에만 최대 30일 재사용한다. 서버는 새 Finding에 원래 결정 ID를 참조한 `EXACT_EVIDENCE_REUSE` 교정 행을 새로 쓰고 감사에 남긴다. 객체 버전이나 동일성 근거가 없거나 하나라도 달라지고, 기한 만료·승인 취소·다른 유효 Finding이 있으면 다시 대기열에 올린다. 구 오탐 이력은 조건을 증명할 수 없으면 재사용하지 않는다. `review-context`는 현재 교정 출처·만료일과 재검토 이유를 표시한다.

정확한 위치·콘텐츠 유형·범위·유효 기간에 일치하는 **명시적 허용 규칙**만 자동 `NORMAL` 판정에 사용할 수 있다. 정책 평가기는 `policyResult=ALLOWED|DENIED|UNKNOWN`과 적용 규칙·정책 버전을 남긴다. `CONFIRMED`·`PRESENT` 값 위치에 유효 Finding이 있고 `ALLOWED`일 때만 `decisionSource=POLICY`와 근거 스냅샷을 가진 새 `NORMAL` 이력을 기록한다. `DENIED`는 근거 있는 보안 검토 권고이지 자동 `ABNORMAL`이나 대응 건 개설이 아니다. 규칙 부재·충돌·정책 만료는 `UNKNOWN`이다. 수동 판정 뒤의 새 자동 결과가 사람의 결정을 조용히 덮지 않으며 정책·원천 버전이 바뀌면 재검토 표식과 이전 근거를 같이 보여준다.

`GET /api/v1/secrets/cases/{caseId}`의 `occurrences[]`는 **호출자가 접근할 수 있는 위치만** 안전한 표시·노출 판정·담당 팀/개인·조치 주기·최근 재검증·막힌 이유와 함께 반환한다. `visibleOccurrenceCount`도 그 배열의 수이며 전체 위치 수가 아니다. 서버는 권한 밖 위치의 행·정확한 수·식별자를 숨긴다. 완전한 위치 접근 권한이 없는 사용자에게는 숨겨진 위치의 실제 존재·수·활동과 무관하게 일관된 `restrictedCoverage=true` 신호를 제공한다. 건 전체에 권한이 있는 보안 조정자만 모든 연결 위치와 정확한 수를 본다. 보안 담당자는 건 전체를 조정하지만 `REMEDIATOR`는 할당되고 권한이 있는 **위치**만 조치한다. 담당자 후보와 조치 권한은 각 위치의 연결 범위로 검사한다. `GET /api/v1/secrets/cases/{caseId}/closure-checks`는 서버 내부에서 모든 관련 위치·미연결 비정상 위치·불확실 위치·검사 누락, 조치 이후 비교 가능한 재검증, 자격 증명의 교체·폐기 근거를 검사한다. 응답은 호출자가 볼 수 있는 범위의 `PASSED|BLOCKED|UNKNOWN`과 안전한 사유·확인 범위를 반환하고 제한 위치의 세부·정확한 수는 내보내지 않는다. 공급자 상태를 자동 확인하지 못한 외부 수행 증거는 `APPROVED_MANUAL` 확인 수준으로 분리한다.

제한 위치가 있는 사용자에게 `capabilities.canResolve=false`를 반환하고 상단에는 ‘권한 있는 보안 조정자의 전체 확인 필요’라는 일반 안내만 제공한다. 해결 명령은 서버가 **모든 연결 위치와 미연결 현재 노출에 대한 `SECURITY` 접근 권한**을 검사하고 어느 하나라도 없으면 403으로 거부한다. 권한이 있더라도 보안 조정자는 모든 필수 해결 근거를 별도로 충족해야 한다. 제한 위치의 이름·수는 거부 응답에 넣지 않는다.

`PATCH /api/v1/secrets/cases/{caseId}/occurrences/{occurrenceId}/assignment`는 `{team,assigneeUserId,reasonCode,expectedCaseVersion,expectedOccurrenceVersion}`으로 위치별 담당자를 바꾼다. `SECURITY`가 해당 연결의 역할과 지정 대상의 연결별 `REMEDIATOR` 권한·팀 소속을 확인한다. 건 조정 담당자와 위치별 조치 담당자는 다른 필드이며, 위치 담당자 한 명의 권한을 다른 연결 위치로 확장하지 않는다. `POST /cases/{caseId}/actions`는 지정되고 해당 위치에 권한 있는 담당자가 첫 실질 외부 조치를 기록할 때 해당 위치와 건을 `OPEN→IN_PROGRESS`로 함께 전환할 수 있다. 첫 실질 조치 앞에 동일 사실의 두 번째 승인이나 빈 단계 전이 명령을 요구하지 않는다. 한 위치의 여러 조치 기록은 계속 허용하고 `complete-actions`로 해당 위치의 기록 마침을 선언한다.

`POST /api/v1/secrets/cases/{caseId}/resolution-decisions`는 `{decision:"RESOLVE",expectedCaseVersion,closureCheckVersion,evidenceIds[],coverageLevel:"FULL|LIMITED|SIMULATION",idempotencyKey}`와 `SECURITY` 권한을 요구한다. 서버는 현재 건과 키·범위·연결 버전, 같은 키·같은 대응 출처의 연결된 비정상 위치마다 마지막 실질 조치 후 비교 가능한 `MISSING`, 미연결 유효 `ABNORMAL` 없음, 현재 값이 있거나 존재 확인 불가인 미판정 위치 없음, 필수 수집 범위 누락 없음을 같은 트랜잭션에서 다시 검사한다. `presence=UNKNOWN`과 `exposure=UNKNOWN`은 다른 차단 사유이며, 활성 정책의 `NORMAL` 위치에는 제거를 요구하지 않는다. 과거 비교 가능한 미발견·다른 출처 위치의 경계는 5.11절을 따른다. 이어서 **실제 자격 증명**은 공급자 또는 승인된 수동 교체·폐기 수명주기 근거를 요구한다. 공급자 확인을 포함한 모든 필수 검사가 갖춰진 경우에만 `FULL`을 검토하고, 승인된 수동 증거만 있는 경우에는 `APPROVED_MANUAL`·`LIMITED`로 확인 범위를 제한한다. **승인 fixture와 키별로 일치한 합성 TEST**만 fixture 제거 증빙과 비교 가능한 재검사로 `SIMULATION`을 허용하며 공급자 키 폐기로 표시하지 않는다. `POSSIBLE_REAL|CONFIRMED_REAL`은 `SIMULATION`으로 닫을 수 없다. 실제 키가 시험 출처에 있었더라도 실제 수명주기 근거와 관련 교차 환경 경고의 처리가 확인되면 `FULL|LIMITED` 경로를 쓸 수 있다. 막힌 항목은 409 `CLOSURE_BLOCKED`와 안전한 `closureChecks[]`로 반환하고 상태를 바꾸지 않는다. `FULL`은 선언된 승인·지원 범위의 모든 필수 확인이 성공한 때만 가능하다. 지원 밖 이력이나 제한된 수동 확인이 남는 `LIMITED`는 확인 한계를 고정하며 알려진 미해결 노출 또는 필수 검사 실패를 우회하지 못한다. 성공은 해결 근거 스냅샷, 승인자·시각·버전을 추가 전용으로 남긴다.

범위·소유자 이관 근거는 `GET|POST /api/v1/secrets/cases/{caseId}/transfer-evidence`와 `POST /cases/{caseId}/transfer-evidence/{evidenceId}/reviews`로 관리한다. 제출은 `{occurrenceIds,reasonCode:"PROGRAM_SCOPE_ENDED|OWNER_TRANSFER",evidenceRef,acceptedTransferRef,expectedCaseVersion,idempotencyKey}`, 검토는 `{decision:"APPROVE|REJECT",reasonCode,expectedEvidenceVersion,expectedCaseVersion,idempotencyKey}`다. 모든 관련 위치의 `SECURITY` 권한을 검사하고 제출자와 다른 보안 담당자만 승인할 수 있다. 승인 사유는 `TRANSFER_AND_ACCEPTANCE_CONFIRMED`, 거절 사유는 `INSUFFICIENT_EVIDENCE|WRONG_SCOPE`다. 티켓·변경 근거와 인수 확인을 따로 받고 당시 위치·범위 revision·출처 승인·판정 버전을 고정한다. 이관 승인 자체는 해결이나 제외를 실행하지 않으며, 현재 적용 가능한 승인 증거만 `closure-checks`의 `eligibleDismissalEvidenceByReason`에서 선택한다.

판정 오류의 교정 또는 승인된 범위·소유자 이관에 따른 종료는 해결과 다른 `POST /api/v1/secrets/cases/{caseId}/dismissal-decisions`에서 `{reasonCode:"FALSE_POSITIVE|OUT_OF_SCOPE",evidenceIds[],expectedCaseVersion,idempotencyKey}`로 요청한다. `FALSE_POSITIVE`는 **모든** 연결 위치의 Finding이 실제 자격 증명이 아님을 유형별 근거로 교정했고 다른 유효 Finding이 없을 때만 허용한다. `OUT_OF_SCOPE`는 대상이 프로그램 범위 밖이라는 승인된 범위·소유자 이관 근거와 인수 확인을 요구하며, 실제 미해결 비승인 노출을 단순히 제외하는 데 쓸 수 없다. 현재 모든 관련 Finding이 유효한 오탐 근거로 교정됐거나, 위치 제거·필수 검사·키 수명주기와 해당 시험 경고 검토가 모두 충족돼야 한다. `SECURITY`가 원래 판정·새 근거·타 연결의 미연결 위치를 확인하고 `disposition`, `is_current=false`, `closedAt`과 불변 감사 기록을 함께 저장한다. 결과는 `RESOLVED`·위험 수용 수에 넣지 않고 교정 이력에서 찾는다.

### 5.4. 실행 이력과 발견 연결

`GET /api/v1/secrets/runs?type=SCHEDULED|MANUAL|REVALIDATION|ALL&status=&provider=&connectionId=&from=&to=&snapshotId=&page=&size=`는 세션과 연결별 조회 권한을 먼저 적용하고 필터를 AND로 결합한 뒤 `requestedAt DESC,runId DESC`로 정렬·페이지 처리한다. `page`는 API에서 0부터, `size`는 1–100(기본 20)이며 목록 화면의 1부터 시작하는 페이지를 변환한다. `from`·`to`는 UTC ISO 8601 경계로 받아 요청 시각의 `from <= requestedAt < to`에 적용한다. `status`는 허용된 실행 결과 enum만, `connectionId`는 권한 있는 연결 ID만 받는다. 잘못된 값은 400, 권한 밖 연결은 403이며 응답 `{items,total,page,size,unit:"ScanRun",filters,snapshotId,asOf,expiresAt,historyComplete,retainedFrom}`의 `total`은 같은 권한·필터가 적용된 **전체 실행 수**다. 최초 조회는 사용자·권한 revision·정규화 필터에 고정된 `HISTORY` 스냅샷을 만들고 후속 페이지는 같은 ID를 전달한다. 보존 범위를 증명할 수 없으면 `historyComplete=false`, `retainedFrom=null`을 반환한다. 목록 상단 집계도 같은 기준시각·필터에서 서버가 계산해 제공하지 않으면 화면은 숫자 카드를 숨긴다. 결과 항목과 `total`은 같은 조회 스냅샷을 사용해 페이지 간 모순을 피한다.

`GET /api/v1/secrets/runs`와 `GET /runs/{runId}`의 응답은 안전한 연결 식별과 실행 당시의 범위 표시·버전을 제공하고 `triggerType=SCHEDULED|MANUAL|REVALIDATION`을 그대로 표시한다. 재검증 실행은 전체 스캔 성공률 분모에 넣지 않는다. `scopeStatusCounts`는 `PENDING`, `RUNNING`, `SUCCEEDED`, `PARTIAL`, `FAILED`, `SKIPPED`와 호환·근거 부재를 위한 `CANCELLED`, `UNKNOWN` 키를 반환하며 모든 키의 합이 `scopeCount`다. 새 실행의 취소 범위는 아래 SKIPPED/FAILED 규칙으로 정리하고, 과거 UNKNOWN을 여섯 상태로 추정 변환하지 않는다. `finishedScopeCount=SUCCEEDED+PARTIAL+FAILED+SKIPPED+CANCELLED`, `successfulScopeCount=SUCCEEDED`, `gapScopeCount=PARTIAL+FAILED+SKIPPED+CANCELLED`, `unverifiedScopeCount=scopeCount-SUCCEEDED`를 함께 반환한다. 과거 revision·단계 근거를 복원할 수 없으면 성공/누락 집계는 null 또는 aggregationStatus와 함께 미확인으로 처리하고 상태 수만으로 성공률을 계산하지 않는다. `unverifiedScopeCount`에는 진행 중인 `PENDING|RUNNING`이 들어가므로 최종 실패·누락인 `gapScopeCount`와 다르게 표시한다.

**실행 결과** `status`는 여섯 처리 상태와 `CANCELLED`를 허용하며, 범위·단계의 여섯 상태와 구분한다. 작업이 취소되면 실행 결과를 `CANCELLED`로 확정하고, 이미 끝난 범위는 그대로 보존하며 미착수 범위는 `SKIPPED/JOB_CANCELLED`, 진행하다 멈춘 범위는 `FAILED/JOB_CANCELLED`로 확정한다. 과거 기록에 취소 사실을 증명할 근거가 없으면 임의로 다시 쓰지 않는다.

집계 기준 `scopeOutcome`은 범위의 수집 상태와 해당 범위에 **필수로 적용되는** 탐지·검증 단계의 결과를 합성한다. 필수 탐지/검증 실패는 수집이 성공해도 `PARTIAL`이며, 메타데이터 전용의 `NOT_APPLICABLE` 단계와 명시적으로 선택 비활성인 검증은 범위 성공에서 제외하되 단계별 미실행 사유를 응답에 반드시 남긴다. 검증 설정이 켜졌지만 실패한 경우는 범위 `PARTIAL`이다. 오래된 실행에 필수 단계·적용 여부 기록이 없어 신뢰할 수 없는 경우 `scopeCount`는 제공하되 성공/누락 집계에는 `aggregationStatus=LEGACY_UNAVAILABLE`을 주고 화면은 수치를 추정하지 않는다.

각 `scopeResults[]`는 `runScopeId`, 내부 `scopeId`, `scopeRevision`, 운영자가 별도로 승인한 `safeDisplayAlias` 또는 `범위 #ID`, 콘텐츠 유형·리전, 수집·탐지·검증 단계별 상태, 객체 수, 안전한 실패/건너뜀 코드, 시작·종료 시각, 재시도 가능 여부를 제공한다. 별칭은 `approved_ref`·자원명·ARN·파일 경로에서 자동 생성하지 않으며 권한 범위에 따라 제한한다. 실행의 `collectedObjectCount`와 `findingCount`, `distinctCredentialCount`, `newCredentialCount`는 별도 단위·정의·집계 시점을 가진다. 키를 읽지 않는 IAM/목록 메타데이터 범위는 탐지·새 키 수를 0과 혼동하지 않도록 `SKIPPED`와 이유를 제공한다. 부분 실패로 확인되지 않은 범위는 발견 0건이라고 단정하지 않는다.

`distinctCredentialCount`는 이 실행의 값 후보 Finding에 연결된 고유 `CredentialSet` ID 수, `newCredentialCount`는 그중 동일성 키의 원자적 최초 삽입 때 `firstSeenRunId=runId`로 기록된 ID 수다. 같은 후보의 여러 Finding·위치는 한 번만 센다. 이전에 발견한 후보의 새 위치나 동일 항목의 `CANDIDATE→CONFIRMED` 승격은 신규 후보 수에 넣지 않는다. 메타데이터 전용·재검증 실행에는 `newCredentialCount=null`과 미적용 사유를, 과거 실행에 최초 생성 실행을 복원할 수 없거나 결과 연결이 없으면 `null`과 `aggregationStatus=LEGACY_UNAVAILABLE|RESULT_LINK_UNAVAILABLE`을 준다. 부분 실패에서 계산한 수는 확인된 범위의 하한으로만 표시하고 `coverageComplete=false`를 붙인다. 전체 커버리지와 결과 연결이 확인돼 실제 0건인 때에만 숫자 `0`을 반환한다.

`GET /api/v1/secrets/runs/{runId}/findings?page=&size=`는 `{items,total,unit:"Finding",page,size,runId,asOf}`를 반환한다. 각 항목은 `findingId`, `occurrenceId`, `credentialSetId`, `secretId`, 마스킹 이름·구성, 룰 ID·버전, 출처로 제한한다. `SECURITY` 또는 실제 할당 위치의 `REMEDIATOR` 권한을 결과 행에 먼저 적용하고 같은 읽기 시점에서 수와 페이지를 계산한다. `GET /api/v1/secrets?runId={runId}`와 화면 `/secrets/findings?runId={runId}`는 그 실행의 허용 위치와 연결된 고유 키만 조회한다. 실행 상세의 `capabilities.canViewFindings`와 재검사 이력의 `canViewRun`은 서로 다른 권한이다. 후자가 false이면 위치별 재검사 결과는 보존하되 연결 운영 정보를 담은 실행 상세 링크는 만들지 않는다. 접근 범위 밖 결과나 원문은 노출하지 않는다. 실행 상세는 누락 범위의 연결 관리로, 발견 항목의 검토 대기열로, 재검증 실행의 대응 건으로 이동할 안전한 링크 식별자만 반환한다. 404·403·대기·일부 실패·없음은 다른 화면 상태다. 화면 경로 `/secrets/history/:runId`는 직접 열 수 있으며 목록으로 돌아오면 필터·페이지·포커스를 복원한다.

### 5.5. 집계 일관성

`GET /api/v1/secrets/summary`의 대응 건 분포와 대응 목록은 같은 `snapshotId`, 출처 구분과 권한 필터를 적용한다. 분포의 `scope=currentCases`, `unit=ResponseCase`와 혼합 현황의 지표별 단위를 구분한다. 네 조치 단계·별도 예외·미평가 심각도는 각각 현재 대응 건 수와 맞아야 한다. 노출 검토가 필요한 위치, 인벤토리 키, 검사 누락 범위는 서로 단위가 달라 이 합에 넣지 않는다. 스냅샷에는 부분 수집 실패와 마지막 성공 시각을 고정해 미확인을 실제 0으로 표시하지 않는다. 과거 감사·판정·분류·조치 근거와 그 확인 한계는 추가 전용 이력으로 보존한다.

### 5.6. 행동 중심 요약과 같은 시점의 목록

`GET /api/v1/secrets/summary?origin=ALL&source=&provider=&environment=&connectionId=&assignee=&snapshotId=`가 현황 진입점이다. 일반 현황은 검색·상세 필터 없이 ALL로 시작한다. API는 `ALL|PRODUCTION|TEST|UNCLASSIFIED`를 지원하며 기존 생략 시 기본값은 호환 경로로 유지한다. `source`·`environment`·`connectionId`는 **같은 허용된 위치**에서 AND로 적용하고, `provider`는 키 발급자에 적용한다. `assignee=ME|UNASSIGNED|사용자ID`는 건 또는 위치 담당 기록이 있는 대응 지표에만 적용한다. 검사·설정 지표에 적용되지 않는 조건은 `appliedFilters`와 `notApplicableFilters`에 명시한다. q는 이 요약에서 허용하지 않는다.

판정·건 연결·정상 인벤토리 지표는 연결/위치별 `SECURITY` 범위에서, 조치 지표는 `SECURITY` 또는 본인 할당 위치의 `REMEDIATOR` 범위에서, 검사 누락·설정 보완 지표는 허용된 `OPERATIONS|SECURITY` 범위에서 산출한다. 역할을 갖지 못한 지표는 `NOT_AUTHORIZED`로 남긴다. 담당자용 응답은 `accessScope=ASSIGNED_LOCATIONS`, 보안 범위 조회는 `AUTHORIZED_LOCATIONS`를 명시하며 자신이 확인한 위치만으로 건 전체가 해결됐다고 결론 내리지 않는다. 기존 로그인 `ADMIN` 값이 이 권한을 대신하지 않는다.

이 요약은 서로 다른 단위를 담으므로 최상위는 `scope=workbench`, `unit=Mixed`, `total=null`이다. `workbench`·`Mixed`는 합성 조회용 새 식별값이며 `currentCases`·`ResponseCase`의 의미를 바꾸지 않는다. 대응 건 전용 요약·분포는 `caseSummary` 하위 블록으로 제공한다. 각 지표는 아래의 고유 단위를 유지한다.

| 지표 키 | 집계 대상·단위 | 같은 스냅샷의 도착 목록 |
|---|---|---|
| `ongoingCases` | 현재 같은 출처의 `OPEN\|IN_PROGRESS`, `ResponseCase` | `review-queue?bucket=ACTION_REQUIRED&stages=OPEN,IN_PROGRESS` |
| `revalidationPendingCases` | 현재 같은 출처의 `REVALIDATION_PENDING`, `ResponseCase` | `review-queue?bucket=ACTION_REQUIRED&stages=REVALIDATION_PENDING` |
| `reviewRequiredLocations` | NEEDS_DECISION과 NEEDS_CASE_LINK의 occurrenceId 합집합, `Occurrence` | `review-queue?bucket=REVIEW_REQUIRED` |
| `needsDecisionLocations` | `UNKNOWN` 또는 근거가 낡아 재검토가 필요한, 유효 Finding이 남은 위치, `Occurrence` | `review-queue?bucket=NEEDS_DECISION` |
| `needsCaseLinkLocations` | 유효 `ABNORMAL`이 현재 건에 연결되지 않은 무결성 복구 대상 위치, `Occurrence` | `review-queue?bucket=NEEDS_CASE_LINK` |
| `coverageGapScopes` | 5.8절의 현재 필수 승인 범위 중 검사 누락 사유가 있는 고유 범위, `Scope` | `coverage-gaps` |
| `unclassifiedScopes` | 선택 출처의 활성 연결·활성 범위 중 설정 사유가 남은 고유 범위, `Scope`. 이름은 호환용이며 미분류 출처만 뜻하지 않음 | `scopes?origin={selectedOrigin}&classificationStatus=NEEDS_REVIEW` |

미지정 건은 `ongoingCases`·`revalidationPendingCases`의 **부분집합**이며 추가 노출 건수로 더하지 않는다. ALL은 현재 허용된 세 actual origin의 위치·건을 함께 읽되 occurrenceId·caseId로 중복을 제거하고 각 건의 origin·근거를 보존한다. `crossEnvironmentAlerts`는 자격증명 여부 확인 업무로 연결하며 `unit=CrossEnvironmentAlert`를 유지한다. 경고·조사·위치·고유 키 수를 대응 수에 합산하지 않는다. 기존 exact-origin API에서도 열람 가능한 자격증명 확인 경고를 누락하지 않는다.

#### 현황의 실제 업무·검사 상태·설정 보조 조회

요약은 숫자만으로 업무 행이나 검사 성공을 추정하지 않도록 다음 읽기 모델을 함께 제공한다. 새 업무 상태·원장이나 새로운 위험 점수를 만들지 않는다.

- `workPreviews.review`, `workPreviews.response`, `workPreviews.revalidation`, `workPreviews.decision`: `{status,items,total,unit,snapshotId}`. 같은 스냅샷·권한·필터의 대기열 순서에서 최대 5개를 반환한다. 화면은 최대 3개씩 요약하고 같은 스냅샷의 전체 목록으로 연결한다. 응답 실패·권한 없음은 빈 업무로 단정하지 않는다.
- `collectionHealth`: `{status,origin,approvedScopeCount,configuredConnectionCount,lastAttempt,lastCompleteSuccess,counts,items,providers}`. `counts`는 success·partial·failed·notRun·paused·running·incomplete이며 items는 범위별 미리보기다. providers는 미리보기 일부가 아닌 전체 허용 범위에서 source·approvedScopeCount·counts·gapScopeCount·lastAttempt·lastScopeSuccess·lastCompleteSuccess를 계산한다. gapScopeCount는 실패뿐 아니라 지연·미검사 등 현재 누락 범위를 중복 제거한 수다. 최신 현재 scope revision·필수 단계 근거만 사용하며 lastAttempt의 범위 결과와 실행 전체 결과를 구별한다. lastCompleteSuccess는 동일한 전체 실행에서 모든 필수 범위가 성공한 사실로만 제공하고 범위별 다른 날의 성공을 합성하지 않는다. configuredConnectionCount를 기록하지 않은 과거 스냅샷은 null이며 연결 없음 0으로 추정하지 않는다. 적용되지 않는 필터를 설명하고 원문·보호 승인 참조는 반환하지 않는다.
- `setupSummary`: `status,selectedOrigin,selectedOriginScopeCount,unclassifiedScopeCount,pausedScopeCount`와 사유별 `reasonCounts,selectedReasonCounts,unclassifiedReasonCounts,pausedReasonCounts` 및 `items,unclassifiedItems,pausedItems`를 제공한다. 각 미리보기는 최대 10개다. 선택 출처의 활성 설정, 출처 미분류인 활성 설정, 중지된 연결의 설정을 별개 집합으로 구분한다. 전역 두 그룹은 `globalGroupsIgnoreOrigin=true`로 출처 필터 미적용을 밝힌다. 발견 소스·연결·ACL은 모든 그룹에 적용하고 발급자·환경·담당은 `notApplicableFilters`로 표시한다. 같은 범위의 여러 사유는 중복 집계될 수 있으므로 사유별 수의 합을 총 범위 수로 쓰지 않는다.

중지된 연결은 현재 수행 가능한 설정 수나 선택 출처 지표에 끼워 넣지 않는다. 활성화 요청과 남아 있는 과거 설정은 별도 안내한다. 미분류 전역 수를 운영·시험 각각의 적용 지표처럼 표시하지 않는다. 적용 필터와 `queueFilter`가 같은 집합을 가리켜야 한다.

위치 행과 위험 조회의 `legacyExposureAssessment`는 기존 판정의 `assessmentId,decision,scopeRevision,currentScopeRevision,needsReconfirmation,reasonCodes`를 허용된 범위에서 제공한다. 기존 판정의 revision이 없거나 재확인 표식이 있으면 원판정을 현재 평가 근거로 위조하지 않고 재확인 이유를 설명한다. 해당 판정·현재 대응의 존재를 새 심각도 UNKNOWN으로 지우지 않는다.

`NORMAL_INVENTORY`는 `inventorySubtype=METADATA_ONLY|AUTHORIZED_STORAGE|RETIRED_HISTORY|MANAGED_LOCATIONS`를 구분한다. 승인 보관은 같은 actual origin의 허용 위치 전체에 유효 Finding·활성 ALLOWED 정책·NORMAL 근거가 있고 활성 대응·재검토가 없는 키다. RETIRED_HISTORY는 현재 열람 가능한 유효 PROVIDER_VERIFIED 폐기·교체 근거가 있고 같은 origin에 활성 대응·판정 대기·현재 존재/판정 UNKNOWN이 없는 키다. ALL에서는 각 actual origin의 가시적인 위치 집합이 독립적으로 이 조건을 통과해야 한다. 모두 같은 하위 유형이면 해당 유형을 유지하고, 서로 다른 적격 하위 유형이 함께 있으면 MANAGED_LOCATIONS로 표시한다. 한 origin의 보관·폐기 근거를 다른 origin에 적용하지 않는다. 수동 승인·INVALID·만료 시각 추정은 폐기 확인 근거가 아니며 소스·환경 필터로 다른 확인 필요 위치를 가리거나 권한 밖 근거로 상태를 만들지 않는다.

`stages`는 `ACTION_REQUIRED`에서 `OPEN|IN_PROGRESS|REVALIDATION_PENDING`의 중복 없는 부분집합, `CURRENT_CASES`에서 이들과 `RESOLVED|RISK_ACCEPTED`의 중복 없는 부분집합을 쉼표로 받는다. 다른 대기열에 stages를 보내거나 빈 값·중복·알 수 없는 단계를 보내면 400이다. 전체 위험 수용은 모든 관련 위치의 SECURITY 권한이 있을 때만 별도 파생 단계로 제공한다. 일부 위치만 열람하는 사용자에게는 원래 조치 단계와 restrictedCoverage를 제공하며 숨은 예외의 승인·만료로 단계를 바꾸지 않는다.

지표의 공통 형태는 `{key,scope,unit,value,status,reasonCode,appliedFilters,notApplicableFilters,queueFilter}`다. 등록된 지표마다 한 항목을 반환하고 `status=AVAILABLE`일 때만 `value`는 0 이상의 정수다. `caseSummary`, `normalInventory`, `crossEnvironmentAlerts`도 각각 자신의 status와 단위를 유지한다. 권한 없는 하위 합계·분포는 null이고 경고 목록은 빈 배열이며 권한 밖 수를 0으로 추정하지 않는다. 역할이 없으면 `NOT_AUTHORIZED`, 과거 자료로 집계를 복원할 수 없으면 `LEGACY_UNAVAILABLE`, 집계 실패는 `UNAVAILABLE`이며 이때 `value=null`이다. 실제 0, 미지원, 권한 없음과 읽기 실패를 구별한다. `queueFilter`는 서버가 만든 허용 필터 객체이며 임의 URL·공급자 링크를 반환하지 않는다. `partialCoverage`는 열람 가능한 필수 범위에서 확인된 누락 여부이며, 집계 근거·권한이 없으면 null과 같은 상태·사유를 반환한다. 권한 제한은 별도 `restrictedCoverage`로 설명하고 누락 0건으로 간주하지 않는다. 화면 라벨은 [문구와 표시 기준](../02-functional-design/content-and-labels.md)을 따른다.

```json
{
  "scope": "workbench",
  "unit": "Mixed",
  "total": null,
  "origin": "ALL",
  "filters": {"origin":"ALL"},
  "snapshotId": "example-workbench-snapshot",
  "asOf": "2026-10-21T00:00:00Z",
  "expiresAt": "2026-10-21T00:15:00Z",
  "metrics": [
    {"key":"reviewRequiredLocations","scope":"reviewQueue","unit":"Occurrence","value":3,"status":"AVAILABLE","reasonCode":null,"appliedFilters":{"origin":"ALL"},"notApplicableFilters":[],"queueFilter":{"bucket":"REVIEW_REQUIRED","origin":"ALL"}},
    {"key":"coverageGapScopes","scope":"coverage","unit":"Scope","value":1,"status":"AVAILABLE","reasonCode":null,"appliedFilters":{"origin":"ALL"},"notApplicableFilters":[],"queueFilter":{"gapOnly":true,"origin":"ALL"}}
  ],
  "caseSummary": {"scope":"currentCases","unit":"ResponseCase","total":2,"actionCounts":{"OPEN":1,"IN_PROGRESS":0,"REVALIDATION_PENDING":1,"RESOLVED":0,"RISK_ACCEPTED":0}},
  "partialCoverage": true
}
```

수치는 응답 모양을 설명하는 예시다. `caseSummary.total`에 대해서만 네 단계·별도 예외 수의 합을 검증하며 `metrics[].value` 합계를 만들지 않는다. `severityCounts`는 CRITICAL·HIGH·MEDIUM·LOW·UNASSESSED를 구분하며 같은 total에 일치한다. 모든 위치가 미평가이면 대표 심각도도 UNASSESSED다. 오탐·범위 밖 교정 종료·정상 인벤토리를 현재 대응 분포에 끼워 넣지 않는다. ALL에서 actual origin만으로 현재 대응을 제외하지 않는다. 근거 또는 집계 결과가 없으면 심각도·매트릭스는 `UNAVAILABLE`과 사유를 반환하고 기본 등급을 만들지 않는다. 관련 건의 일부 위치만 볼 수 있으면 그 사용자가 열람 가능한 건·위치로 집계한 `visible` 범위임을 응답에 밝힌다. 제한 위치의 정확한 수·이름·미할당 여부를 추론할 수 있는 차감 수치를 주지 않는다. 사용자 전체의 안전 여부나 건 해결 가능성을 이 부분 집계에서 결론 내리지 않는다.

최초 요약 요청은 인증 사용자·역할 부여 revision·허용 연결/범위 revision·정규화 필터·출처에 묶인 스냅샷을 `REPEATABLE READ`에서 만든다. `asOf`는 서버 DB 시각이며 TTL 상한은 **15분**, `expiresAt`을 반환한다. 열람 가능한 정책·오탐 재사용·전체 위험 수용 예외의 효력이 먼저 끝나면 그 시각까지 유효 기간을 줄인다. 같은 ID는 요약, 대기열, 대응 목록, 검사 누락 목록의 동일 필터 투영에만 쓴다. 큐의 `total`은 해당 카드와 같아야 한다. 필터 또는 호환 API의 읽기 origin 변경은 새 스냅샷을 요구한다. 같은 사용자·조회 scope에서 필터나 출처가 다르면 409 `SNAPSHOT_MISMATCH`, 역할 취소·범위 권한 변경이면 409 `SNAPSHOT_ACCESS_CHANGED`로 거부한다. 만료·상한 초과로 폐기한 ID는 보존한 폐기 표식으로 409 `SNAPSHOT_EXPIRED`를 구분한다. 존재하지 않거나 다른 사용자·조회 scope의 ID는 404다. 권한이 취소된 이전 행은 만료 전에도 반환하지 않는다. 오류 코드는 `code=CONFLICT` 아래 `details.reasonCode`로 제공한다.

스냅샷을 오래된 상태의 쓰기 근거로 쓰지 않는다. 상세 진입은 현재 상태를 읽고 `readAt`, 버전, `changedSinceSnapshot`을 표시하며, 판정·조치는 최신 버전으로 다시 검사한다. 변경 성공 후에는 `requiresSnapshotRefresh=true`를 반환하고 화면이 요약·목록을 함께 다시 읽는다. 갱신하지 않은 카드만 감소시키는 클라이언트 추정 집계는 금지한다. 스냅샷 생성은 화면 초기 진입·명시 새로고침·변경 후 재조회에 한정하고 초 단위 폴링마다 새로 생성하지 않는다. 사용자별 동시 유효 스냅샷 상한은 5개이며 넘으면 가장 오래된 스냅샷을 만료시키고 기존 조회에 같은 새로고침 흐름을 적용한다.

WORKBENCH·INVENTORY·HISTORY의 동시 스냅샷 생성에서 SQLSTATE `40001`이 발생하면 해당 읽기 트랜잭션을 전부 롤백하고 권한 확인·원천 조회·스냅샷 저장을 새 REPEATABLE READ 트랜잭션에서 최대 3회 시도한다. 재시도 소진과 외부 트랜잭션이 소유한 충돌은 409 `SNAPSHOT_RETRY`와 `retryable=true`로 안내한다. 다른 SQL 오류와 권한 거부는 이 재시도 대상에서 제외한다.

기간은 지표마다 의미가 다르므로 현황의 현재 상태 카드에는 공통 `from/to`를 받지 않는다. `/cases?changedFrom=&changedTo=`는 `updatedAt`의 `[from,to)`, `/review-queue?observedFrom=&observedTo=`는 위치의 `lastSeenAt`의 `[from,to)`, `/runs?from=&to=`는 `requestedAt`의 `[from,to)`다. 날짜만 있는 입력은 화면이 사용자의 표시 시간대 기준 경계를 UTC로 변환한다. 불완전 경계·역전·형식 오류는 400이다. 기간 필터가 있는 도착 목록은 동일 기간이 적용된 별도 집계로 표시하고 무기간 현황 카드 수와 같다고 주장하지 않는다. 보존된 기간 밖 요청은 빈 결과를 실제 무발생으로 해석하지 않게 `retainedFrom`·`historyComplete=false`를 반환한다.

### 5.7. 키별 위치 시각화의 안전한 읽기 모델

`GET /api/v1/secrets/{id}/locations?origin=ALL&source=&environment=&presence=&page=&size=`를 키 독립 상세 `/secrets/:secretId`의 집계 경로로 둔다. 위치 권한은 서버에서 적용한다. 이 경로는 `SECURITY`가 허용된 위치, 또는 현재 할당받고 해당 연결의 `REMEDIATOR` 권한이 있는 위치에 접근할 때만 사용할 수 있다. `OPERATIONS`만으로 키 상세·판정·대응 증거의 읽기 권한이 생기지 않는다. 전체 연결 조회의 권한을 발견 키 조회 권한으로 확장하지 않는다.

키 상세에서는 대응 건이 없어도 AWS·GitHub의 기본 조치 절차를 외부 호출 없이 확인할 수 있다. 안내를 여는 동작은 대응 건 생성·노출 판정·조치 기록·AI 요청이 아니며, 건별 가이드 API를 쓰기 위해 건을 임의 생성하지 않는다. 실제 유효성·합성 표본·권한·해결 근거는 각 자원에서 별도로 확인한다.

응답은 `{credential,filters,visibleLocationCount,locationStateCounts,items,total,page,size,readAt,detailVersion,restrictedCoverage,capabilities}`다. `page`는 0부터, `size`는 1–100(기본 20)이며 `total=visibleLocationCount`는 같은 권한·필터로 허용된 위치의 수다. 정렬은 현재 미해결 위치 우선, `lastSeenAt DESC,occurrenceId DESC`로 고정한다. 같은 응답의 대표 위치·상태 수·총수·페이지는 하나의 DB 읽기 시점을 사용한다. 한 페이지의 길이를 전체 위치 수로 쓰지 않는다. 후속 페이지는 첫 응답의 `expectedDetailVersion`을 전달한다. 허용된 위치·권한·정렬 사실이 바뀌면 409 `DETAIL_CHANGED`로 첫 페이지 재조회를 요구해 이전 페이지와 최신 페이지의 중복·누락을 막는다. 이 버전은 허용된 사실의 불투명 버전값이며 숨겨진 위치 활동으로 바뀌지 않는다.

| 블록 | 필수 정보·의미 | 필드별 권한·비노출 |
|---|---|---|
| `credential` | ID, 발급자·유형·구성 상태·마스킹 이름, 최신 유효성 결과·사유·시각과 이전 마지막 성공, 심각도 또는 미평가 | 키 수준 결과도 호출자가 허용된 범위에서 열람할 수 있는 근거만 집계. 보이지 않는 위치의 발견·검증 활동을 시각·수치로 누출하지 않음 |
| 위치 식별 | `occurrenceId`, `source`, `environment`, `origin`, 범위 revision. `safeLocationLabel`은 현재 승인 범위와 일치하고 안전 검사를 통과한 `sourceDetails.label`을 우선 사용하며, 없으면 검증된 별칭 또는 `위치 #ID` | 검증되지 않은 원문 경로·임의 원천 URL·보호 승인 참조·ARN·계정 ID·HMAC·문맥 제외. 좌표가 현재 승인 대상과 다르거나 안전문자·토큰 검사를 통과하지 못하면 별칭 또는 `위치 #ID`로 대체 |
| 독립 사실 | `presence`, `exposure`, `storagePolicy`, `validity`, `coverage`, `fixtureStatus` 각각의 값·근거 ID·확인 수준·관측 시각 | 유효성·허용 여부·수집 성공·운영/시험을 서로 추론하지 않음. 검사 누락이 기존 `ABNORMAL` 판정 사실을 지우지 않음 |
| 위치 업무 | `caseId?`, 위치별 담당자·조치 상태, 조치 주기, 마지막 실질 조치 시각, 재검사 작업·결과·비교 가능 여부, 부족 근거, `whyHereCode`, `nextActionCode` | SECURITY는 허용 위치의 판정·조정 근거, 할당 REMEDIATOR는 자신의 위치 조치·필요 최소 증거만 제공. 다른 역할에서 얻은 필드를 합집합으로 모두 공개하지 않음 |
| 해결 확인 | `locationAbsenceCheck`, `credentialLifecycleCheck`, `coverageCheck`의 `PASSED\|BLOCKED\|UNKNOWN`, 안전한 이유·확인 범위 | 외부 조치 기록을 공급자 폐기 확인으로 표시하지 않음. 건 전체 해결 검사는 별도 `/closure-checks`의 전체 권한 조건 유지 |
| `capabilities` | `canReview`, `canViewCase`, `canRecordAction`, `canRequestRevalidation`, `canViewCoverage`, `canOpenSafeSource` 등 허용된 위치별 작업 | 버튼 숨김 보조. 실제 변경 명령과 원천 열기는 매 요청마다 권한 재검사 |

`locationStateCounts`는 존재·노출·조치 등 **각 축별**로 센다. 예를 들어 존재 세 값의 합은 같은 필터의 `visibleLocationCount`와 맞지만 존재 수와 노출 수를 합치지 않는다. 건에 연결되지 않은 위치에는 조치 단계를 `null`, `caseLinkStatus=NOT_REQUIRED|NEEDS_DECISION|NEEDS_CASE_LINK`로 반환하고 임의 `OPEN`을 만들지 않는다. 같은 위치의 여러 엔진 Finding·조치 기록을 위치 수에 중복 반영하지 않는다. 현재 확인 불가라도 과거 마지막 성공 관측과 시각을 따로 반환한다.

`whyHereCode`는 이 위치가 표시된 근거, `nextActionCode`는 호출자가 다음으로 할 수 있는 작업 한 개다. 두 필드는 서버의 고정된 우선순위 규칙으로 계산하고 원시 `facts[]`도 함께 제공한다. 알려진 노출이 있으면 추가 검사 누락과 동시에 표시하며 `COVERAGE_GAP`을 노출 판정의 대체 상태로 쓰지 않는다. 심각도 평가가 없다는 이유로 해당 위치를 낮은 위험·해결로 표시하지 않는다. 안전한 카드·표 렌더링이 우선이며, 관계도는 이 응답에서 확인된 키–위치 연결만 사용한다. 권한·자산 연결은 별도 실제 증거가 없으면 노드를 만들지 않는다.

`restrictedCoverage`는 전체 위치를 확인할 권한이 없을 수 있다는 일반 신호일 뿐, 제한 위치의 수나 존재를 추론시키는 값이 아니다. 제한 대상의 배열·정확한 수·키 활동 시각·담당자·집계 차이를 내보내지 않는다. 동일 키가 다른 출처에도 있는지 역시 허용된 위치만으로 판단한다. 판정·대응·연결·유효성 증거의 원본 접근 권한을 각각 검사하며, 허용된 관계가 하나 있다는 이유로 연결된 모든 이력을 공개하지 않는다.

키 상세의 표시 영역 `인벤토리|확인 필요|조치 필요`는 여러 사실에서 파생한 보기이며 기존 구성·존재·노출 상태를 대체하지 않는다. `GET /api/v1/secrets`는 `contentKind=ALL|VALUE_FINDINGS|METADATA` 필터를 지원하고 `componentState`, 대표 허용 위치, 열람 가능한 위치 수, 확인 필요 이유를 반환한다. `/secrets/findings`의 모수는 **고유 키 인벤토리**이고 기본 `contentKind=VALUE_FINDINGS`는 값 후보 `CONFIRMED|CANDIDATE`를 뜻하며 비정상 노출 확정을 뜻하지 않는다. IAM `METADATA_ONLY`는 `contentKind=METADATA`에서 별도로 본다. `ALL`도 사용자 권한 범위 안의 고유 키이며 `NORMAL_INVENTORY` 대기열의 정책상 보관 근거가 있는 부분집합과 혼동하지 않는다.

같은 고유 키 목록 안의 `workBucket=ALL|NEEDS_DECISION|ACTION_REQUIRED|NORMAL_INVENTORY|NEEDS_CASE_LINK`는 별도 작업 필터다. 키가 여러 위치에 있어도 항목은 한 번이고 해당 위치·건에 맞는 `matchedLocationCount`와 사실들을 제공한다. 여러 업무가 있는 키는 각각의 작업 보기에서 보일 수 있으므로 작업 보기 수를 합산하지 않는다. 이 목록은 `unit=CredentialSet`이며 위치 단위 검토 대기열과 대응 건 단위 대기열의 수와 같을 필요가 없다. 현황의 위치·건 카드 도착지는 먼저 **같은 단위의 대기열**이고, 그곳에서 고유 키 상세로 이동한다. 단위를 바꾸는 링크에는 새 모수를 표시한다.

### 5.8. 연결·범위의 읽기 상태와 검사 누락

실행 선택용 `/connections`와 관리용 `GET /api/v1/secrets/connections/manage?status=&origin=&page=&size=`와 `GET /api/v1/secrets/connections/{connectionId}/scopes?origin=&classificationStatus=&gapOnly=&page=&size=`를 분리한다. `OPERATIONS`는 승인 범위 운영·실행 사실, `SECURITY`는 자신에게 허용된 범위의 승인·분류·정책 요약을 본다. 일반 REMEDIATOR의 키 상세에는 필요한 범위 읽기 상태만 포함하고 전체 연결 목록 권한을 주지 않는다. 실행 선택 목록과 관리 목록은 목적과 필드를 구분한다.

관리 연결의 응답에는 현재 `connectionStatus`, 안전한 별칭, 승인 범위 수·revision, `latestProbe`, `lastFullScanAttempt`, `lastSuccessfulFullScan`, `nextDueAt`, `activeJob`, 권한에 맞는 상태별 범위 수를 분리한다. 연결 테스트는 AWS STS 신원·GitHub 설치 토큰 발급·Notion 인증 확인을 나타내는 `checkKind=IDENTITY_OR_INSTALLATION`이고 **수집 범위 권한·본문 접근·발견 키 유효성 확인이 아니다**. 테스트의 시각·안전한 사유 이력을 남겨도 연결 활성 상태나 수집 성공 시각을 변경하지 않는다.

범위별 `coverage`는 현재 `scopeRevision`, `lastAttempt`의 실행·시각·`scopeOutcome`, 동일 revision과 필수 단계 구성에서 마지막으로 성공한 `lastSuccessfulScan`, `nextDueAt`, `isPaused`, `isRunning`, `freshness`와 `gapReasonCodes[]`를 제공한다. `freshness=CURRENT|OVERDUE|NEVER_SCANNED|NOT_SCHEDULED`는 주기와 예정 시각으로 계산하는 읽기 정보이며 기존 수집 상태 enum을 바꾸지 않는다. `lastSuccessfulScan`을 최신 실패 시각으로 덮어쓰지 않고, 범위 변경 전의 성공을 현재 revision의 확인으로 재사용하지 않는다. 실행 ID가 있으면 스캔 상세로 연결하고 대상이 실행 전이면 연결·범위 복구로 안내한다.

`GET /api/v1/secrets/coverage-gaps?origin=ALL&source=&connectionId=&snapshotId=&page=&size=`는 현재 승인된 필수 범위의 revision마다 한 행을 반환한다. `unit=Scope`, `total`은 동일 권한·필터·스냅샷에서 누락 사유가 있는 고유 범위 수다. 사유는 `LAST_ATTEMPT_INCOMPLETE`, `NEVER_SCANNED`, `REVISION_UNCHECKED`, `OVERDUE`, `CONNECTION_PAUSED` 등 여러 개일 수 있으나 같은 범위는 한 번만 센다. 실패한 최신 시도가 있어도 마지막 성공은 함께 표시한다. 정상적으로 대기/진행 중인 실행 자체는 최종 실패로 세지 않으며, 이전 누락·기한 초과가 이미 있으면 실행 중이라는 이유로 숨기지 않는다. 선택 비활성 검증이나 메타데이터의 해당 없음 단계는 필수 수집 누락이 아니고 단계 사유로 남긴다.

비활성화·승인 종료된 범위는 현재 예약 분모에서 제외하되 과거 실행과 알려진 노출을 삭제하지 않는다. 범위 비활성화만으로 미해결 건의 해결 조건을 통과시키지 않고 범위 종료·이관 근거를 요구한다. 연결 전체 성공은 **같은 전체 스캔 실행에서 당시 모든 필수 범위·필수 단계가 성공한 경우**에만 기록하며 서로 다른 날짜의 범위별 성공을 합쳐 하나의 연결 전체 성공으로 만들지 않는다. 재검사 작업은 한 위치의 증거이므로 전체 연결·전체 승인 범위의 최신 성공으로 쓰지 않는다. 원천 후보의 부재를 확정할 때에는 같은 원천 버전·탐지 설정 등 비교 조건을 별도로 만족해야 한다.

관리용 표시 별칭은 `PATCH /api/v1/secrets/scopes/{scopeId}/display-alias`의 `{safeDisplayAlias,expectedScopeRevision,expectedAliasVersion}`으로 변경한다. 해당 연결의 `OPERATIONS`가 두 현재 버전을 검사한다. null은 별칭 해제이며 문자열은 1–80자, 첫 글자는 영문 또는 한글, 나머지는 영문·숫자·한글·공백·밑줄·하이픈만 허용한다. 자격 증명 접두사·24자 이상 연속 ASCII 토큰형 문자열·10자리 이상 연속 숫자도 거부한다. 원문·URL·제어문자·HTML은 표시 별칭으로 받지 않는다. 성공은 `{scopeId,safeDisplayAlias,scopeRevision,aliasVersion,requiresSnapshotRefresh:true}`이며 `aliasVersion`만 증가하고 읽기 스냅샷을 무효화한다. 별칭 변경 이력·수행자·시각을 추가 전용으로 남기며 수집 revision·승인 참조·과거 실행 별칭은 바꾸지 않는다. 실제 연결 추가·원천 참조·권한 비밀 등록은 기존 운영 등록 경계를 유지하며, 자격 증명 원문을 받는 일반 대시보드 API를 추가하지 않는다. 범위 수정은 승인·새 revision·분류 정책 계약을 거친다.

### 5.9. 능력과 필드별 권한

- `/cases`와 `/review-queue`의 단계 필터는 모두 `stages=OPEN,IN_PROGRESS`처럼 중복 없는 쉼표 구분 목록으로 통일한다. `/cases`는 네 조치 단계를 허용하고, `/review-queue`는 `bucket=ACTION_REQUIRED`에서만 `OPEN|IN_PROGRESS|REVALIDATION_PENDING`의 부분집합을 허용한다. 다른 bucket에 `stages`를 보내면 400이다. 카드의 대응 중·재검사 대기 링크와 서버 목록에 동일 단계 조건을 적용한다. 단일 `stage`와 `stages`를 혼용하지 않으며 명세 밖 필터는 400이다.
- `closure-checks`는 버전·근거별 검사 결과와 서버가 산출한 `allowedCoverageLevels[]`, 각 수준의 확인 범위·한계를 제공한다. 해결 요청의 `coverageLevel`은 이 허용 수준과 대조하며 사용자가 `LIMITED`를 골라 알려진 실패·노출을 제외할 수 없다. 동일 키의 외부 교체·폐기 근거를 여러 위치에서 참조하더라도 각 위치의 조치 후 부재 근거는 별도로 요구한다. 재발한 새 건은 근거의 적용 키·관측 시각·조치 이후 조건을 다시 검사한다.
- 수집·능동 검증·위치 재검사는 원천·유형별 등록 능력에 따라 별개로 허용한다. [자격 증명 유형](../04-reference/credential-types.md)의 지원 조건을 적용하며 지원하지 않는 능력은 capability=false와 안전한 사유를 반환하고 등록을 거부한다. 외부 키 교체·폐기는 담당자가 수행하며 일반 대시보드 API가 공급자 키를 변경하지 않는다.
- 키별 조회의 최신 검증·수명주기·영향 근거도 원본 실행·근거에 접근할 권한이 있어야 한다. 허용된 키 ID만으로 모든 원천의 키 활동 시각을 노출하지 않는다. 권한 때문에 현재 근거를 표시할 수 없으면 해당 블록을 `NOT_AUTHORIZED`로 반환하고 위치 상태를 `INVALID`·`MISSING`으로 바꾸지 않는다.
- 5절의 `SNAPSHOT_EXPIRED`, `STALE_REVIEW_CONTEXT`, `CLOSURE_BLOCKED` 등 409 사유는 공통 오류 `code=CONFLICT`의 안전한 `details.reasonCode`다. 403·404·422는 공통 상태 규약을 유지하고, 위치·필터가 권한 밖이라는 상세 원천 정보는 오류에 포함하지 않는다.

### 5.10. 수명주기·합성 표본 증거의 제출과 검토

자기 신고인 외부 조치 기록, 보안 담당자가 검토한 수동 증거, 공급자가 확인한 키 상태는 다른 확인 수준이다. `/actions`에 티켓을 적거나 체크리스트를 완료했다는 이유로 수명주기 검사가 통과하지 않는다. 다음 경로는 원문 없이 제한된 증거를 추가·검토한다. 목록은 허용 증거와 `allowedKinds`, `allowedReviewReasonCodes`, `capabilities`를 제공하고 공급자 폐기 검증을 제공하지 않는 검증기는 `providerVerificationStatus=NOT_SUPPORTED_BY_CURRENT_VALIDATORS`로 표시한다.

| Method·경로 | 요청·권한 | 결과와 불변 조건 |
|---|---|---|
| `GET /cases/{caseId}/lifecycle-evidence` | 해당 위치에 권한 있는 SECURITY 또는 할당 REMEDIATOR. 허용된 증거·적용 위치만 조회 | 근거 ID, `ROTATED\|REVOKED\|RETIRED\|FIXTURE_RETIRED`, 수행/관측 시각, 안전한 참조, 제출·검토 상태, 확인 수준·한계. 원문·공급자 응답·권한 밖 참조 제외 |
| `POST /cases/{caseId}/lifecycle-evidence` | `{kind,performedAt,evidenceRef,expectedCaseVersion,idempotencyKey}`. SECURITY 또는 허용 위치에 할당된 REMEDIATOR | `201`과 `{evidenceId,version,status:"PENDING",requiresSnapshotRefresh:true}`. 상세는 GET으로 다시 읽으며 제출 원본은 `verificationMethod=EXTERNAL_RECORD`, `reviewStatus=PENDING`. 키는 건에서 결정하며 임의 다른 키·출처에 적용할 수 없음. 티켓/변경 참조의 허용 형식만 받고 원문·자유 URL·파일 업로드는 받지 않음 |
| `POST /cases/{caseId}/lifecycle-evidence/{evidenceId}/reviews` | `{decision:APPROVE\|REJECT,reasonCode,expectedEvidenceVersion,expectedCaseVersion,idempotencyKey}`. 적용 대상 전체의 SECURITY이며 제출자와 다른 사용자 | 실제 참조·적용 키·시각·확인 범위를 검토한 추가 전용 기록. 승인해도 `APPROVED_MANUAL`이며 `PROVIDER_VERIFIED`로 바뀌지 않음. 실키 수동 증거의 해결 상한은 `LIMITED`이며 정확한 합성 표본의 FIXTURE_RETIRED는 `SIMULATION`에만 사용 |
| `GET /{credentialSetId}/fixture-attestations` | 해당 TEST 위치의 SECURITY | 승인 manifest ID·버전, 위치 scope revision, 보호된 일치 검사의 안전한 결과 ID, 확인자·시각·상태만 반환. manifest의 값·지문·다른 키 목록은 제외 |
| `POST /{credentialSetId}/fixture-attestations` | `{scopeId,expectedScopeVersion,manifestId,manifestVersion,matchEvidenceId,idempotencyKey}`. 해당 TEST 범위의 SECURITY | 서버에 존재하는 승인 manifest와 정확한 키·scope revision·보호된 일치 근거를 재검사해 attestation ID를 생성. 사용자 선언·TEST 표식·키 ID만으로 합성 인정 불가 |

위 경로는 모두 `/api/v1/secrets` 아래다. 수명주기 증거의 요청 키 ID는 별도 입력받지 않고 현재 건의 `credentialSetId`를 사용한다. 제출·검토 버전과 감사·멱등 결과는 같은 트랜잭션에 저장한다. 원 제출 행의 `verificationLevel=PENDING`은 바꾸지 않고 별도 검토 이력에서 현재 유효 수준을 계산한다. 승인된 수동 증거는 `APPROVED_MANUAL` 확인 수준을 유지하고 공급자 확인이 없다는 한계를 화면·해결 스냅샷에 남긴다. 공급자 근거로 키 상태를 확인할 수 없는 버전도 이 수동 검토 경로로 제한 확인을 할 수 있으나, 알려진 미해결 위치·조치 후 검사 실패를 우회하지 못한다. 같은 키의 다른 출처 건에서 증거를 적용할 때도 적용 키·시각·권한·현재 건을 별도로 검사하고 추가 전용 적용 관계로 참조한다. 이후의 실제 VALID 관측이 폐기 주장과 상충하면 과거 승인을 현재 유효 폐기 근거로 사용하지 않는다.

`verificationMethod=PROVIDER`, `verificationLevel=PROVIDER_VERIFIED`는 승인된 읽기 검증기·워커만 생성한다. 일반 HTTP 요청은 이 필드를 지정할 수 없다. 검증기 ID·버전·승인 엔드포인트 분류·실제 공급자 관측 시각·안전한 확정 사유가 필요하며 인증 거절·사용 불가·연결 테스트만으로 폐기 확인을 만들지 않는다. 공급자 검증기가 없는 유형이면 이 수준을 생성할 수 없다. 실키 수동 증거를 SECURITY가 승인한 것만으로 FULL 수준이 생기지 않는다. FIXTURE_RETIRED도 실제 키 폐기 확인으로 승격하지 않는다.

fixture manifest는 별도로 승인·버전 고정된 합성 표본 등록물이며 보호된 일치 자료는 일반 API에 노출하지 않는다. 워커 또는 신뢰된 내부 비교기가 현재 키의 보호된 동일성을 manifest와 정확히 비교하고 결과 ID·키·scope revision·manifest 버전을 보존한다. SECURITY는 그 일치 결과를 검토·승인하거나 이미 승인된 manifest의 매칭 정책에 따라 자동 attestation을 허용할 수 있다. 원문 키를 폼에 붙여 넣거나 마스킹 suffix를 비교하는 방식은 허용하지 않는다. manifest 만료·취소·키/위치 revision 변경·실제 키임을 보이는 상충 근거가 생기면 이전 attestation으로 SIMULATION을 통과할 수 없다. 최신 유효 attestation과 실제 fixture 제거·조치 후 비교 가능한 재검사가 있을 때만 시험 종료를 허용한다.

수명주기 신고의 `performedAt`은 정확한 수행 시각을 알 때 입력하고 미상이면 null을 허용한다. 서버는 `observedAt`과 접수 시각을 독립 기록하며 조회에서 `performedAtStatus=KNOWN|UNKNOWN`을 파생한다. 시각 미상 신고는 등록·검토·조회할 수 있고, 조치 이후의 근거가 필요한 해결 조건은 추가 확인될 때까지 UNKNOWN으로 유지한다. 공급자의 인증 거부 시각이나 접수 시각을 수행 시각으로 대입하지 않는다.

교체 증거에 선택적인 `replacementCredentialSetId`, `beforeValidationAttemptId`, `afterValidationAttemptId`, `replacementValidationAttemptId`를 연결할 때 기존 키·새 키·각 관측의 소속·현재 ACL·시각·revision을 검증한다. 참조한 관측은 당시 사실로 보존하고 현재 사용 적격성은 조회·적용 시 다시 검사한다. 제출 원본·검토·인증 결과·교체 관계를 서로 다른 항목으로 반환하며 일반 401 관측은 `AUTHENTICATION_REJECTED`와 원인 미확인 설명으로 제공한다.

### 5.11. 같은 출처의 해결 대상과 표시 집계

해결·건 집계는 같은 `CredentialSet`과 같은 **대응 건 출처**를 기준으로 계산한다. 승인 위치에서 값을 발견한 사실만으로 그 값을 모두 지우게 하지 않는다.

| 대상 | 해결 검사에서의 처리 |
|---|---|
| 현재 건에 연결된 같은 출처의 비정상 위치 | 각 위치의 마지막 실질 조치 이후 비교 가능한 `MISSING` 필수. `presence=UNKNOWN`은 검사 차단이며 `exposure=UNKNOWN`과 다른 사실 |
| 같은 출처의 유효 `ABNORMAL`이지만 건에 미연결 | `UNLINKED_ABNORMAL`로 차단하고 `LINK_EXISTING_ASSESSMENT` 안내. 기존 노출 판정을 UNKNOWN으로 낮추거나 다시 입력시키지 않음 |
| 같은 출처의 아직 판정하지 못한 값 위치 | 유효 Finding이 있고 `PRESENT` 또는 현재 존재 확인 불가인 경우 `UNASSESSED_CURRENT_LOCATION`으로 차단. 판단 근거·조사 작업으로 안내 |
| 노출 미판정이나 최신 완전·비교 가능한 검사에서 `MISSING`인 과거 위치 | 현재 판정을 강제하지 않고 해결 범위·이력·한계에 보존. 이후 실패·범위 개정·비교 불가로 현재 부재 근거가 낡으면 다시 현재 확인 불가로 포함 |
| 활성 보관 정책으로 `NORMAL`이 유지되는 위치 | 위치 삭제·미발견을 필수로 요구하지 않음. 정책 만료·새 상충 근거로 재확인이 필요해지면 위 미판정/불확실 위치로 처리 |
| 다른 출처의 같은 키 위치·현재 건 | 운영/시험 건·단계·해결 수에 합치지 않음. 키 상세의 허용된 다른 출처 경고·관련 건 링크로 보여주고, 시험 실키 경고는 별도 안전성 검사를 계속 요구 |

`closure-checks`는 `caseId`, 허용된 `caseVersion`, `closureCheckVersion`, `checkedAt`, `allowedCoverageLevels`, `coverageOptions[]`(수준·필요 근거 ID·한계), `closureChecks[]`(코드·상태·사유·허용 위치 ID·근거 ID)를 제공한다. 화면은 서버의 허용 수준·종료 사유·적격 근거만 선택하고 검사를 통과하지 못한 수준을 임의 구성하지 않는다. `lastResolution`·`lastDismissal`은 과거 확정 결과를 현재 검사와 분리한다. 대상은 `targetOrigin`, `requiredResponseLocations`, `unlinkedAbnormalLocations`, `unassessedCurrentLocations`, `authorizedStorageLocations`, `historicalMissingLocations`, `otherOriginNotice`를 각 단위·권한 범위에 맞춰 분리한다. 제한 위치의 행·정확한 수는 내보내지 않으며 전체 해결 권한이 없으면 일반 차단 이유만 제공한다. 해결 검사는 위 대상 구분을 동일하게 적용한다. 같은 키의 모든 정상 보관 위치를 삭제하거나, 다른 출처의 불확실성을 운영 건에 조용히 합산하는 방식으로 해석하지 않는다.

키 독립 상세·인벤토리에서 출처 필터는 **해당 출처의 위치·건·대표 행동**에 적용한다. 키 자체의 구성·발급자는 바꾸지 않는다. 키의 검증·수명주기 사실이 다른 출처에서 생성됐더라도 열람할 권한과 같은 키에 적용되는 근거가 있으면 `evidenceOrigin`·`observedAt`·확인 범위를 명시해 보여줄 수 있다. 다른 출처의 `UNCHECKED` 또는 합성 차단 시도를 현재 출처의 성공 검사로 설명하지 않는다. 위치 수·단계·정책·최근 원천 검사 시각을 다른 출처 결과로 덮지 않는다. 대응 링크도 단일 `caseId` 대신 허용된 `casesByOrigin[]`와 선택 출처의 `currentCaseId`를 구분한다.

### 5.12. 필터 적용과 마스킹 CSV

| 현황 대상 | `origin` | `source`·`connectionId` | `provider`(키 발급자) | `environment` | `assignee` |
|---|---|---|---|---|---|
| 현재 대응 건·조치 지표 | 대응 건 출처 | 같은 허용 위치 | 해당 키 발급자 | 같은 허용 위치 | 해당 건/할당 위치의 담당자 |
| 확인·건 연결 필요 위치 | 해당 위치 출처 | 해당 위치 | 해당 키 발급자 | 해당 위치 | 해당 없음. 조사 담당자는 별도 조사 필터 |
| 검사 누락 범위 | 승인 범위 출처 | 해당 범위 | 해당 없음 | 해당 없음 | 해당 없음 |
| 출처 미분류 범위 | 해당 없음. UNCLASSIFIED 고정 | 해당 범위 | 해당 없음 | 해당 없음 | 해당 없음 |
| 자격증명 여부 확인 경고 | 내부 경고 원장의 실제 분류·허용 위치를 사용 | 허용된 관련 위치 | 해당 키 발급자 | 허용된 관련 위치 | 해당 없음. 경고 담당자는 별도 경고 필터 |

`provider`와 `environment`를 미검사 범위의 발견 키·위치에서 역으로 추정하지 않는다. 해당 없음 필터는 지표의 `notApplicableFilters`로 반환하고 UI는 검사 누락 카드에 `환경 필터 미적용`처럼 적용 한계를 표시한다. 없는 대응 건에서 assignee를 추정하거나 작업 담당자 없는 범위를 ‘미지정 대응 건’으로 세지 않는다.

위치 API는 `exposure=ABNORMAL|NORMAL|UNKNOWN`, `actionStatus=OPEN|IN_PROGRESS|REVALIDATION_PENDING|RESOLVED|NO_CASE`, `expectedDetailVersion`을 받는다. 소스·환경·존재·노출·조치 필터는 같은 허용된 위치에 AND로 적용하고 단일 키 요약과 화면 필터의 수를 구분한다. 날짜 필터는 키 위치 표에 추가하지 않고 마지막 관측 시각을 표시한다. 기간은 활동/실행 목록에서 적용한다.

스캔 이력의 `GET /runs`는 5.4절 파라미터 외 `provider=GITHUB|AWS|NOTION|KUBERNETES`로 **수집 연결 플랫폼**을 필터할 수 있다. 실행 응답의 `provider` 필드와 같은 의미이며 키 발급자 필터가 아니다. 이 경로의 용어 예외를 UI에는 `수집 플랫폼`으로 표시한다. 연결·플랫폼·기간·실행 목적·상태는 같은 허용 실행에 AND로 적용하고 미지원 연결을 가짜 실행으로 채우지 않는다.

CSV는 새 export endpoint 없이 고유 키 인벤토리 `GET /api/v1/secrets`의 **동일 INVENTORY 스냅샷 전체 페이지**에서 만든다. 첫 요청의 정규화 필터(`origin`, `q`, `contentKind`, `workBucket`, 소스·발급자·환경·구성·검증 등), `scope=inventory`, `unit=CredentialSet`, `asOf`, `snapshotId`, `total`, `expiresAt`을 고정하고 후속 페이지에 같은 ID·필터를 전달한다. 권한은 첫 요청과 매 페이지에서 확인한다. 한 페이지 또는 현재 브라우저 행만 내보내고 ‘현재 조건 전체’로 표시하지 않는다.

내보내기 상한은 **고유 키 10,000개**, 페이지 크기는 최대 100이다. 상한을 넘으면 시작 전에 필터 축소를 안내하고 조용히 잘라 파일을 만들지 않는다. 스냅샷 만료·권한/범위 변경·페이지 오류·전체 행수 불일치가 생기면 내보내기를 중단하고 새 조회를 요구한다. 모든 페이지를 확인한 뒤 한 파일을 내려받으며 일부 파일을 완성본으로 제공하지 않는다. 전체 UI 카드용 WORKBENCH와 독립 생성한 INVENTORY의 ID를 바꾸어 쓰지 않는다.

열은 마스킹 키·발급자·유형·구성·허용 위치 수·검증 상태/시각·작업 이유·기준시각 등 허용된 요약으로 제한한다. 원문·문맥·HMAC·전체 경로·ARN·공급자 응답·권한 밖 위치 수·외부 증거 자유문은 제외한다. 조회 범위는 전체 자료 기준으로 표시하고 내부 자료 분류는 사용자용 열에서 제외한다. UTF-8 CSV의 구분자·따옴표·줄바꿈을 올바르게 escape하고, 셀 앞의 공백을 포함해 `=`, `+`, `-`, `@` 등 수식 시작 문자를 탐지해 텍스트로 무력화한다. 필터·기준시점·단위·확인 범위는 파일의 요약 필드로 제공하며 실제 키 원문을 CSV 검증용으로 기록하지 않는다.

### 5.13. 다음 행동·내부 담당 요청과 처리 결과

업무 행과 키·위치·범위·대응 상세의 `taskGuidance`는 `{whyHereCode,priorityReasonCodes[],evidenceObservedAt,nextActionCode,target,canExecute,blockedReasonCode,requiredRole,waitingSince,relatedTaskId,remainingReasonCodes[]}`다. `target`은 서버가 허용한 대상 종류·내부 ID만 제공하고 화면은 허용된 내부 주소로 연결한다. 실행할 수 없으면 차단 이유·필요 역할·대기·허용 요청 경로를 제공하며 원천 URL·AI 문장을 행동으로 실행하지 않는다. `whyHereCode`와 `nextActionCode` 등 기존 단일 필드는 같은 투영의 호환 표현이고 서로 다른 다음 행동을 반환하지 않는다.

`priorityReasonCodes[]`는 `POST_ACTION_REDETECTED`, `CONFIRMED_EXPOSURE`, `VALID_AND_POLICY_DENIED`, `ASSIGNEE_REQUIRED`, `FOLLOWUP_OVERDUE`, `REVALIDATION_REQUIRED`, `COVERAGE_BLOCKED`, `EVIDENCE_REQUIRED`의 허용된 근거다. 실제로 확인한 조치·검사·정책·검증·시각에 한정하며 새 위험 점수나 노출 판정이 아니다. 같은 단위·권한·스냅샷 안의 정렬에 적용하고 필터 뒤 `total`·membership을 보존한다. 필요한 근거 시각을 확인할 수 없으면 해당 이유를 생성하지 않는다. 동률은 해당 자원의 내부 ID로 고정하고 숨겨진 사실로 순서를 바꾸지 않는다.

`POST /api/v1/secrets/investigations`는 내부 담당 요청을 등록한다. 요청은 `{targetType:SCOPE|OCCURRENCE,targetId,missingReasonCode,assigneeUserId?,dueAt?,evidenceRef?,expectedTargetVersion,idempotencyKey}`이며 서버가 `creationSource=HUMAN_HANDOFF`를 채운다. SCOPE는 해당 연결 OPERATIONS 또는 SECURITY, OCCURRENCE는 해당 위치 SECURITY 또는 할당된 REMEDIATOR 중 `canRequestAssistance`가 있는 사용자만 가능하다. 허용 질문은 출처 확인·보관 정책 보완·소유자 확인·원천 접근 복구·근거 보완이다. 수신자의 현재 대상 권한·질문에 맞는 역할을 검사하고 요청자가 사용자 ID로 권한을 부여하지 못하게 한다. 적격 담당자·기한이 없으면 배정 대기로 보존한다. 수동 노출 판단의 NEEDS_MORE_INFO에는 기존 담당·기한 필수 조건을 유지한다.

해당 조회의 `capabilities.canRequestAssistance`와 `allowedMissingReasonCodes`를 제공한다. 질문 코드는 `SOURCE_UNCLASSIFIED|POLICY_UNKNOWN|OWNER_UNKNOWN|SOURCE_PERMISSION|EVIDENCE_INCOMPLETE` 중 대상·역할에 허용된 부분집합이다. SOURCE_UNCLASSIFIED는 기존 조사 과제·호환 명령의 이유 코드이며 일반 화면의 분류 필수 입력이나 현재 설정 미완료의 자동 근거가 아니다. SCOPE의 expectedTargetVersion은 현재 분류 버전, OCCURRENCE는 현재 위치 버전이며 서버가 대상 소속도 확인한다. 양식의 담당·기한 선택은 과제 기록이며 권한·기한 정책을 바꾸는 명령이 아니다.

새 요청은 201, 같은 대상·질문의 열린 요청 재사용은 200과 `{taskId,created,requiresSnapshotRefresh}`다. 기존 요청을 열람할 수 없으면 taskId·담당·활동 시각을 내보내지 않고 안전한 접수 상태만 반환한다. 원문·자유 메모·외부 메시지 발송은 포함하지 않는다. 요청 접수는 분류·판정·조치 완료가 아니며 반복 검사·재전송으로 과제를 증식하지 않는다.

과제 종료는 기존 `/investigations/{taskId}/close`에서 요청 종류에 맞는 `resolvingDecisionId|restoredRunScopeId|scopeOriginDecisionId|policyBindingId|acceptedEvidenceHistoryId`와 기대 버전을 검사한다. 근거는 동일 대상·현재 버전·권한과 질문의 완료 조건에 맞아야 한다. 소유자 확인·자료 보완 요청의 acceptedEvidenceHistoryId는 같은 과제에 제출한 근거를 적격 SECURITY가 검토한 이력만 허용하며 UNKNOWN을 자동 판정하지 않는다. 이 근거로 분류·정책 적용·검사 복구의 필수 결과를 대신하지 못한다. 단순 상태 입력으로 닫거나 분류 결정으로 검사 복구를 대신하지 않는다.

변경 결과에는 적용된 결정·과제·건·실행의 허용 ID, 실제 효과와 `remainingReasonCodes`를 제공한다. UI는 적용 결과에 맞는 상세로 이동하고 원래 목록의 조건·선택 행을 유지한다. 변경된 읽기 스냅샷은 명시적으로 새 기준을 선택하며 옛 스냅샷의 수를 조용히 바꾸지 않는다. 키 상세의 필수 대응 위치 요약은 기존 위치·건·해결 검사에서 계산하고, 전체 위치 수·승인 보관·메타데이터와 다른 모수임을 표시한다. 권한·기간·출처 변경으로 감소한 표시 수를 위험 감소율로 반환하지 않는다.

## 발견 위치와 연결 실행 표시

키 위치 응답은 기존 occurrence ACL을 적용한 뒤 `sourceDetails`와 `detectionEvidence[]`를 제공한다. `sourceDetails`는 승인 범위와 일치하는 저장소·브랜치·파일·줄 또는 허용된 AWS 자원 좌표이며, `status=UNAVAILABLE`은 원본 좌표 확인 불가다. 같은 좌표·표시 규칙을 대응 상세의 허용된 `occurrences[]`와 위치 `review-context`에도 적용한다. 이 조회에서는 ACL을 통과한 각 위치의 현재 승인 참조로 좌표를 검증하고 원본 참조를 응답하지 않는다. 원본 `approved_ref`, 인증 참조·파일 내용·키 원문은 반환하지 않는다. 규칙 기록은 `{ruleId,detectedAt,runId}`다. GitHub 링크는 고정 호스트와 인코딩한 좌표만 사용한다.

연결 관리 응답은 `targetSummary[]`, `testBlockedReasonCode`, `scanBlockedReasonCode`와 기존 `capabilities.canTest/canRun`을 제공한다. 사용 설정 상태와 실제 연결 테스트 결과를 구분하며, 실행 버튼을 숨기는 대신 권한·중지·대상 누락 등 사유를 표시한다. 일반 분류 저장 권한은 `capabilities.canClassifySource`다.

## 6. 발견 키의 유효성 확인

검증 방법·사유·원문 재취득·시간·진행 상태의 원본은 [검증 계약](credential-validation.md)이다. 연결 probe, 발견 키의 인증 확인, 조치 후 위치 재검사는 다른 자원이다. 일반 조회는 공급자 호출을 시작하지 않는다.

| Method·경로 | 계약 |
|---|---|
| `GET /api/v1/secrets/{id}/validation?occurrenceId={occurrenceId}` | 허용된 키·선택 위치의 지원 상태, 최근 적격 완료 시도, 마지막 인증 성공, 진행 중 요청, 최신성·한계·capability |
| `POST /api/v1/secrets/{id}/validation-requests` | 키 원문 없이 선택 위치의 유효성 확인 작업 등록. 202와 `requestId`, `jobId`, `runId`, `status`, `reused`, `requiresSnapshotRefresh` |
| `GET /api/v1/secrets/validation-requests/{requestId}` | 해당 목적의 작업만 조회. 대기·선점·종료·생략·실패, 단계별 시각과 허용된 결과 ID |
| `GET /api/v1/secrets/{id}/validation-history?occurrenceId={occurrenceId}&page={page}&size={size}` | 해당 원천 근거에 대한 현재 ACL을 적용한 시도 이력. `requestedAt DESC, attemptId DESC` 정렬 |

등록 본문은 `{occurrenceId, expectedContextVersion, idempotencyKey}`다. `expectedContextVersion`은 위 GET이 발행한 불투명 버전이며 키 동일성·위치·scope revision·검증 승인 revision·조치 revision에 묶인다. 클라이언트가 공급자·endpoint·원문·인증 참조·검증 결과를 지정하지 못한다. 한 키에 위치가 여러 곳이면 권한 있는 적격 위치를 명시하며 서버가 임의로 권한 밖 위치로 바꾸지 않는다.

SECURITY 또는 해당 위치의 현재 지정 REMEDIATOR가 승인된 능력 안에서 요청할 수 있다. OPERATIONS의 수집 권한과 로그인 ADMIN은 발견 키 검증 권한을 대신하지 않는다. 검증 설정·허용 endpoint·호출 예산을 확대하는 일은 이 명령의 효과가 아니다. 등록과 전송 직전에 현재 역할·연결·승인 revision을 다시 확인한다.

작업은 기존 `secret_jobs`의 `kind=REVALIDATE`, `purpose=CREDENTIAL_VALIDITY`를 사용하고 선택 위치·안전한 기대 revision만 저장한다. 기존 대응 재검사는 `purpose=LOCATION_RECHECK`다. `requestId=jobId`로 매핑하며 별도의 요청 원장·공급자별 큐를 만들지 않는다. 같은 요청자·명령·멱등 키의 같은 본문은 기존 결과를 반환한다. 다른 활성 작업은 409 `details.reasonCode=ACTIVE_JOB_EXISTS`이며, 그 작업을 볼 권한이 있을 때만 진행 링크를 제공한다. 자동 검증과 수동 검증의 외부 호출 중복은 검증 계약의 실행 예약으로 제어한다.

읽기 응답의 필수 블록은 다음과 같다.

- `support`: 지원·부분 지원·미지원과 안전한 사유, 허용 검증 방법. 미지원은 공급자 인증 실패가 아니다.
- `latestAttempt`: 현재 ACL·동일성·승인 조건에 적격인 최근 완료 시도. `result`, `reasonCode`, `method`, `validatorVersion`, `requestedAt`, `requestSentAt`, `providerCheckedAt`, `completedAt`, `currentEligible`를 제공한다. 없으면 null이다.
- `lastAuthenticatedAt`: 적격한 VALID의 실제 공급자 관측 시각. UNKNOWN·UNCHECKED·대기·실패로 갱신하지 않는다. 승인 또는 동일성 조건이 바뀌면 현재 블록에 승격하지 않고 허용된 과거 이력으로만 보존한다.
- `lastConclusiveValidation`: 마지막 적격 VALID/INVALID의 결과·실제 관측 시각·시도 ID. 마지막 인증 성공과 분리하며 현재 응답 오류를 숨기는 용도로 사용하지 않는다.
- `latestRequest`: 진행 중 작업의 ID·상태·접수/시작/예정 시각. 진행 중 작업을 완료된 유효성 결과로 만들지 않는다.
- `freshness`: `CURRENT|STALE|POST_ACTION_REQUIRED|UNAVAILABLE`과 이유·확인 기한. 확인 기한은 토큰 만료일이 아니다.
- `nextDueAt`, `postActionValidationPending`: 다음 유효성 확인 기한과 조치 이후 확정 관측 부재. 마지막 성공·작업 접수 시각과 혼용하지 않는다.
- `capabilities`: `canRequestValidation`, `blockedReasonCodes`, 허용된 설정/담당자 요청 행동. 단순 boolean만 반환하지 않는다.
- `readAt`, `contextVersion`, `coverage`, `limitations`: 기준 시각·필드별 권한·관측 범위. 제한된 다른 위치의 결과·수·시각을 노출하지 않는다.

기존 키·위치 요약의 `validationStatus`는 같은 적격 근거에서 투영하고, 시도 전이면 `UNCHECKED`와 `UNSUPPORTED_TYPE|DISABLED|SYNTHETIC_FIXTURE|INCOMPLETE_PAIR|SOURCE_VALUE_UNAVAILABLE|NO_VERIFICATION_ATTEMPT` 등의 정확한 사유를 제공한다. `validatedAt`은 실제 공급자 관측 시각인 `providerCheckedAt`에만 대응하며 접수·생략·시도 종료 시각으로 채우지 않는다. 과거 VALID의 확인 기한이 지났으면 원래 결과와 당시 시각은 보존하되 `freshness=STALE`과 `다시 확인 필요`를 함께 표시한다. 조치 후에는 기존 POST_ACTION_VALIDATION_REQUIRED 경계를 적용한다.

권한 없는 대상은 기존 404/필드 차단 규칙, 지원·구성·승인 부족은 422의 안전한 사유, 오래된 context는 409, 대시보드 요청 한도는 429를 사용한다. 공급자의 401/403/429는 대시보드 세션 오류로 중계하지 않는다.

검증 설정 조회는 기존 연결·범위 설정 응답에 현재 서버 검증 profile 버전·활성화·scope 승인 revision·승인 여부·지원 유형과 `canConfigureValidation`을 추가한다. `PATCH /api/v1/secrets/scopes/{scopeId}/validation-policy`는 `{expectedValidationApprovalRevision,idempotencyKey,enabled,reasonCode}`를 받으며 현재 범위 SECURITY만 수행한다. 이는 이미 승인된 범위의 등록 검증기를 허용/철회하는 직접 결정이고 수집 범위를 넓히거나 endpoint·원문·권한을 지정하지 못한다. 서버 profile 활성화는 승인된 운영 배포 설정의 책임이다. 연결·서버 profile이 꺼져 있으면 범위 허용만으로 외부 호출하지 않는다. 결정자·시각·승인 revision·감사를 원자 저장하고 철회는 대기 요청 취소와 현재 투영 무효화를 수행한다. 서버 profile이 바뀌면 기존 scope 승인을 자동 확대하지 않는다.

### 6.1. Notion 인증과 원천 capability

Notion 발급자의 `support.methods`는 등록된 `NOTION_GET_SELF`를 제공한다. 원천이 GitHub 파일·승인 AWS 본문·승인 Notion 본문인지와 발급자 검증기 지원을 각각 판정한다. `REVALIDATE/CREDENTIAL_VALIDITY`는 현재 원천 재취득과 보호된 동일성 확인 후 실행한다. 원천 수집기가 미등록이면 원천 미지원 사유, Notion 검증기 미등록·비활성이면 해당 사유를 구분한다.

Notion 연결 테스트는 연결의 불투명 참조 `notion-internal:default`에 대응하는 외부 토큰으로 `GET /v1/users/me`만 수행하고 `secret_connection_probes`에 결과를 기록한다. 화면은 `Notion 인증 확인`과 페이지 읽기 결과를 구분한다. `capabilities.canTest/canRun`, `canRequestValidation`, 권한 관측 readiness는 등록된 각 capability와 현재 승인에서 독립 계산한다. 수집 비활성은 `NOTION_COLLECTION_DISABLED`, 토큰 파일 접근·형식 실패는 `NOTION_CREDENTIALS_UNAVAILABLE`, 인증·접근 거부는 `NOTION_ACCESS_DENIED`, 공급자 대기는 `NOTION_RATE_LIMITED`, 나머지 통신·응답 실패는 `NOTION_UNAVAILABLE`로 제공한다. 외부 경로·토큰·전체 응답은 반환하지 않는다.

Notion scope는 `NOTION_PAGE_CONTENT`, 정규화한 페이지 UUID, `CURRENT_VERSION_ONLY`와 현재 페이지 텍스트 경계를 제공한다. 임의 URL·추가 페이지를 스캔 요청 본문에서 승인하지 않는다. 발견 위치는 승인 페이지와 일치하는 제목·블록 ID에서 구조화하고, 원본 열기 주소는 `https://www.notion.so/<하이픈 없는 페이지 UUID>#<하이픈 없는 블록 UUID>`로 생성한다. 제목은 블록 fragment를 생략한다. 페이지·블록 불일치나 확인 불가 좌표에는 원본 링크를 제공하지 않는다.

실행 상세의 `scopeResults[].excludedObjectCount`는 선언된 하위 페이지·데이터베이스·링크·첨부 본문 제외 수다. 해당 값은 기본 0이며 수집한 텍스트 객체 수와 별개다. 화면은 0보다 큰 제외 개수와 `현재 페이지 텍스트 · 하위 페이지·외부 본문은 별도 대상` 경계를 표시한다. 알려진 제외만으로 성공 범위를 검사 누락으로 세지 않는다. 지원 텍스트를 읽지 못했거나 변경·불완전 순회가 있으면 기존 단계·범위 결과에 `PARTIAL|FAILED`와 `NOTION_*` 안전한 실패 코드를 제공한다.

Notion에서 성공한 순회·변경 미검출·새 후보 0건도 기존 위치의 부재 증명으로 제공하지 않는다. 분 단위 수정 시각과 커서 순회의 한계에 따라 전체 스캔·대응 위치 재검사의 미관찰은 `UNKNOWN`으로 표시하고 제거·회수 완료나 `MISSING`으로 전환하지 않는다. 실제 관찰한 위치·동일 키에 대한 인증 결과는 독립적으로 제공하며 이 경계는 기존 `severity-v2`를 변경하지 않는다. [수정 시각의 정밀도](https://developers.notion.com/guides/resources/historical-changelog#june-28-2021), [페이지 나눔 계약](https://developers.notion.com/reference/pagination)

Notion의 현재 결과·마지막 인증 성공·이력·시각·재시도는 이 절의 공통 응답을 사용한다. 정규화 사유는 `AUTHENTICATED`, `AUTHENTICATION_REJECTED`, `ACCESS_LIMITED`, `UNEXPECTED_RESPONSE`, `RATE_LIMITED`, `PROVIDER_UNAVAILABLE`, `PROVIDER_ACCESS_BLOCKED`, `CLIENT_FAILURE`와 실제 사전 차단 사유로 제한한다. 인증 거부는 현재 사용 불가와 세부 원인 미확인, 공급자 제한은 운영 확인 필요를 나타낸다. 원천 오류·공급자 오류를 대시보드 로그인 오류로 변환하지 않는다.

## 7. 위치별 심각도·근거·재평가

규칙·입력·증거·불변 기록·승인 조건의 원본은 [심각도 평가 계약](risk-assessment-contract.md), 표시와 행동은 [검증·심각도 화면](../02-functional-design/validation-and-risk-experience.md)이다. 규칙과 정책 활성화는 버전이 고정된 검토·배포 절차로 관리하며 일반 화면에 임의 규칙 편집·SQL 실행 API를 제공하지 않는다.

| Method·경로 | 계약 |
|---|---|
| `GET /api/v1/secrets/occurrences/{occurrenceId}/risk` | 현재 위치 평가·적용 중 판단·이전 확정 등급·부족 정보·근거·capability·`evidenceRevision` |
| `GET /api/v1/secrets/occurrences/{occurrenceId}/assessment-history?page={page}&size={size}` | 위치별 자동 평가·수동 조정·승인/반려·적용 이력. `assessedAt DESC, assessmentId DESC` 정렬 |
| `POST /api/v1/secrets/occurrences/{occurrenceId}/risk-evidence` | 허용된 구조화 자료와 제한된 참조 제출. 201과 `evidenceId`, 수용 상태, `evidenceRevision` |
| `POST /api/v1/secrets/occurrences/{occurrenceId}/risk-evidence/{evidenceId}/decisions` | SECURITY의 같은 위치 자료 수용·거절·교정. 원본 대신 새 결정 기록 |
| `POST /api/v1/secrets/occurrences/{occurrenceId}/assessment-requests` | 외부 호출 없이 현재 DB 근거의 규칙 재평가 예약. 202와 `requestId`, `jobId`, `status`, `reused` |
| `GET /api/v1/secrets/assessment-requests/{requestId}` | ASSESS 목적 작업의 상태·안전한 결과/실패 사유. 허용될 때만 assessmentId 제공 |
| `POST /api/v1/secrets/occurrences/{occurrenceId}/severity-overrides` | SECURITY의 별도 수동 등급 제안. 자동 평가·여섯 축의 사실을 덮지 않음 |
| `POST /api/v1/secrets/occurrences/{occurrenceId}/severity-overrides/{assessmentId}/decisions` | 계약에 따른 독립 SECURITY의 승인·반려. 승인 이후에도 현재 근거·버전을 재검사 |

위치 조회 권한·증거 자체의 ACL을 모두 검사하고 같은 키 ID만으로 다른 원천의 권한·악용 근거를 공개하지 않는다. 평가 요청·수동 조정은 SECURITY, 자료 제출은 SECURITY 또는 해당 위치의 지정 REMEDIATOR에 허용한다. REMEDIATOR 제출은 자료이며 공급자 자동 확인이나 적용된 영향 범주가 아니다. SECURITY가 직접 수용한 일반 자료에 다시 자기 자신과 다른 승인자를 일괄 요구하지 않는다. 명시적 등급 조정과 기존 높은 등급 하향은 별도 승인 계약을 적용한다.

자료 본문은 `{expectedEvidenceRevision,idempotencyKey,evidenceType,subject,observedAt,validUntil,evidenceRef,facts,applicableFrom?,applicableUntil?}`다. optional applicableFrom/applicableUntil은 함께 없거나 함께 있는 유효 UTC 반개구간 [from, until)이다. from < until, from <= observedAt, validUntil이 있으면 until <= validUntil을 검사하고 원문 관찰 시각이 구간에 포함되어야 한다. 역사적 ABUSE_LINK는 현재 사실의 적용 기간 필수 조건에서 제외한다. observedAt 확인 시각과 사실의 효력 기간은 별도이며, 누락 자료는 저장 호환을 유지하되 적용 시점 미확인으로 평가 입력에서 구분한다. `facts`는 등록된 유형별 필드·enum·개수·길이만 허용하며 `evidenceRef`는 기존 TICKET/CHANGE/POLICY 등의 제한된 형식이다. 클라이언트는 `PROVIDER_VERIFIED`·검증 결과·자동 매핑 버전·자동 노출 결정·행위자를 지정하지 못한다. 외부 응답·인증 재료·임의 URL·자유 본문은 400으로 거부한다. 자료 결정은 `{expectedEvidenceRevision,idempotencyKey,decision,reasonCode}`이며 수용 대상·유형·같은 키/위치·근거 시각을 재검사한다.

평가 요청 본문은 `{expectedEvidenceRevision,idempotencyKey}`다. 정책 버전은 현재 활성 정책을 서버가 선택하며 사용자가 과거 정책을 현재 판단에 적용하지 못한다. 기존 `secret_jobs`에 `kind=ASSESS`, `purpose=SEVERITY_ASSESSMENT`, 내부 위치 ID·근거 revision을 저장한다. 이 작업은 DB 근거만 평가하며 ScanRun·수집·탐지·검증 성공 기록을 생성하지 않는다. 결과에서 `runId`는 null이다. 같은 위치·근거 revision·정책의 진행 작업은 멱등 재사용한다. 연결의 외부 호출 직렬화와 별개로 위치 잠금·revision CAS를 적용한다.

새 노출 결정·자료 수용/취소·권한 근거·자산 분류·정책 활성화는 해당 사실과 재평가 예약을 원자 확정한다. 유효성 갱신은 결과 설명과 해결 근거의 최신성도 갱신하되 VALID/INVALID만으로 심각도를 바꾸지 않는다. 평가 중 근거 revision이 바뀌면 과거 기준 결과를 현재 head로 적용하지 않고 새 기준으로 재예약한다. 과거 결과는 당시 입력과 함께 보존한다.

위치 risk 응답은 `{latestAutomaticAssessment,effectiveAssessment,lastConfirmedSeverity,evidenceRevision,capabilities,nextAction,readAt}`와 허용된 `inputs,evidenceSummary,missingReasonCodes,conflictReasonCodes,freshness`다. `latestAutomaticAssessment`는 `{assessmentId,assessmentStatus,severity,ruleId,policyVersion,assessedAt}`이고 `assessmentStatus=ASSESSED|UNASSESSED|NOT_APPLICABLE`, 뒤 두 상태의 `severity=null`이다. `effectiveAssessment`는 적용 근거의 같은 필드와 `held,holdReasonCodes,sourceAssessmentId,reviewRequired`를 갖는다. 최신 R99가 기존 CRITICAL과 동시에 존재할 때 각각의 규칙·시각·원평가를 구분한다. 호환 평면 필드 `assessmentStatus,severity,ruleId,policyVersion,assessmentId,assessedAt`를 제공하면 모두 latestAutomaticAssessment에 대응시키고 `effectiveSeverity`는 effectiveAssessment.severity에만 대응시킨다. 규칙별 입력은 [심각도 정책](../02-functional-design/severity-policy.md)을 따른다. 현재 미평가·오래된 근거와 이전 확정/적용 중 등급은 별개이며 새 UNKNOWN으로 과거 악용 등급을 조용히 삭제하지 않는다. 근거 부족을 LOW로 투영하지 않는다.

수동 조정 본문은 `{expectedEvidenceRevision,idempotencyKey,targetAssessmentStatus,targetSeverity,basisAssessmentId,reasonCode,evidenceIds,validUntil}`이고 결정 본문은 `{expectedEvidenceRevision,idempotencyKey,decision,reasonCode}`다. `targetAssessmentStatus=ASSESSED`이면 네 등급 중 하나가 필수다. `NOT_APPLICABLE`이면 `targetSeverity=null`과 현재 적격 R00 등 자동 평가의 `basisAssessmentId`를 요구한다. 이는 잘못된 노출 판단을 근거 있게 교정한 뒤 기존 등급 보류를 독립 검토로 해제하는 경로이며 자료 부재·권한 철회·원천 삭제만으로는 사용할 수 없다. 확정 사고 근거와 충돌하면 반증·정식 교정 없이 보류를 해제하지 않는다. 제안·승인·적용은 서로 다른 기록으로 남기며 자동 평가를 수정하지 않는다. 같은 범위의 독립 승인자·현재 권한·충돌·근거 만료를 적용 시 다시 검사한다. 예외 만료는 자동 LOW 또는 해결이 아니라 자동 근거 재평가와 필요한 검토를 만든다. 등급 보류 해제는 대응 건 해결·키 폐기를 만들지 않는다. 대응 건·현황·목록은 같은 출처·허용 위치·스냅샷 기준의 유효 대표 등급과 미평가 수를 사용한다.

## 8. 판단 경험 읽기 모델

`GET /api/v1/secrets/occurrences/{occurrenceId}/risk`는 기존 필드를 유지하고 `riskExperience`를 추가한다. 모든 하위 사실·대상은 같은 위치의 현재 권한으로 필터링한다. 새로운 등급 원장을 만들거나 GET에서 재평가·외부 호출을 시작하지 않는다.

| 필드 | 계약 |
|---|---|
| version, readAt | `risk-experience-v1`, 이 읽기 모델을 계산한 UTC 시각. 평가 시각과 다름 |
| severityLabel | `시크릿 노출 심각도` |
| sourcePresenceObservedAt | 같은 위치의 실제 원문 관측 시각. 미확인은 null |
| authentication | `{currentResult,freshness,latestAttempt,lastConclusiveValidation}`. 기존 validation.read의 허용된 응답 재사용. 현재 인증과 과거의 마지막 확정 결과 분리 |
| summary | `{status:CURRENT\|STALE\|UNASSESSED\|METADATA_ONLY,reasonCode,ruleId,severity,automaticSeverity,held}`. severity는 실제 적용값, automaticSeverity는 최신 자동값. 저장된 평가와 현재 적격성 구분 |
| criteria[] | `{axis,label,value,status,evidenceIds,reasonCodes,nextAction}`. axis는 기존 여섯 축, status는 CONFIRMED/CANDIDATE/UNKNOWN/STALE/CONFLICT. 허용 근거만 참조 |
| missingFacts[] | `{code,evidenceType,action}`. 현재 미확인·오래됨·충돌인 축에 필요한 질문과 허용된 보완 행동만 제공. 과거 탈락 근거 전체를 질문으로 복사하지 않음 |
| workPriority | `{policyVersion:'work-priority-v1',level,reasonCodes,clockMode,timeZone,targetBasis,targetBasisAt,acknowledgeDueAt,planDueAt,operatingHoursNow,holidayCalendarConfigured}` |
| riskAcceptance | `{enabled:false,reasonCode:'RISK_ACCEPTANCE_DISABLED',allowedSeverities:['MEDIUM','LOW'],maximumDays:7}`. 등급 override와 별도 |
| support.permissionAssistance | `{status:PARTIAL\|UNSUPPORTED,reasonCodes}`. 발급자·수집 원천 조합의 등록 capability 반영 |
| support.automaticUsage | `{status,reasonCodes}`. 사용 adapter가 미등록이면 UNSUPPORTED. 인증 성공으로 지원 상태를 만들지 않음 |

workPriority.level은 `IMMEDIATE|PRIORITY|GENERAL|PLANNED|INVENTORY`다. 기본 clockMode는 BUSINESS_HOURS, timeZone은 Asia/Seoul, 월~금 09~18시다. targetBasis는 GRADE_ACTIVATED(현재 적용 등급 전환의 첫 APPLICATION 시각), FIRST_OBSERVED(일반 미평가의 최초 관측), PRIORITY_SIGNAL_OBSERVED(현재 유효한 공개성 관측과 인증 성공 또는 높은 권한 후보의 연속 관측 구간이 겹치기 시작한 시각), UNCONFIRMED(신호의 시작 시점 증명 부족), NOT_APPLICABLE(관리 기록)이다. 같은 등급의 반복 평가·조회로 기산을 갱신하지 않는다. targetBasisAt·acknowledgeDueAt·planDueAt은 nullable이며 기산을 입증할 자료가 없으면 기한 미확인으로 표시하고 조회 시각으로 만들지 않는다. 공휴일 캘린더 미연결은 false로 표시하고 실제 운영 체계가 없는 ALWAYS_ON을 허용하지 않는다. 운영시간 밖에도 긴급 배지는 유지한다.

`evidenceSummary`의 안전한 항목은 `applicableFrom,applicableUntil,sourcePresenceObservedAt,applicabilityBasis`를 포함한다. `applicabilityBasis=OBSERVATION_ONLY|EFFECTIVE_INTERVAL`이며, 비동시 수집 관측은 사실 기간을 NULL로 유지한다. 서버가 실제 같은 원천 관측을 결합할 때만 해당 값을 제공한다. 누락·불일치·source revision 미확인은 `EVIDENCE_APPLICABILITY_UNCONFIRMED|EVIDENCE_APPLICABILITY_MISMATCH|SOURCE_REVISION_UNCONFIRMED`로 표시하며 기존 행을 추정하지 않는다.

`authentication.currentResult`는 freshness=CURRENT일 때 latestAttempt.result를 사용한다. 시도 기록이 없으면 UNCHECKED이며, 과거 VALID/INVALID만 있고 만료·조치 이후 미검증·최근 오류로 현재성이 없으면 UNKNOWN이다. latestAttempt와 lastConclusiveValidation은 당시 결과·시각으로 별도 보존한다. 현재 인증 표시와 업무 우선 신호는 이 현재성 결과를 사용하고 과거 평가 snapshot의 validity를 현재 인증으로 재사용하지 않는다. v2 등급 계산과는 독립이다.

`analysisContext.excludedEvidence`는 `{evidenceId,kind,reasonCode}`의 안전한 배열이다. 과거 또는 현재 부적격 근거가 왜 제외됐는지 설명하며 현재 부족 질문과 다른 항목이다. 적격한 새 근거가 같은 축을 확인하면 이전 만료 근거를 이유로 STALE 보완 질문을 반복하지 않는다. `analysisContext.access`의 repositoryPublicObserved·publicObservationAt·coexistenceStatus는 공개 메타데이터 관측과 원문 공존 확인을 구분한다. ACCESS_COEXISTENCE_UNCONFIRMED의 missingFacts는 evidenceType=ACCESS_SCOPE, action=REVIEW_ACCESS로 연결하고 원문 관찰 시점의 공유 근거·효력 기간을 확인하게 한다. 단순 재스캔을 동시성 부족의 해소로 안내하지 않는다.

PRIORITY_SIGNAL_OBSERVED는 업무 기산용 관측 연결이며 공개 키 노출의 실제 동시성을 확정하지 않는다. 현재 사건의 시작·조치 재개 이전으로 기산을 소급하지 않고, 반복 관측으로 이어지는 같은 신호는 최초 겹침을 유지한다. 반대 관측·기한 만료·관측 공백이 있으면 연속 구간을 끊는다. 수용·서버 기록 이전의 수동 주장 시각으로 기산하지 않으며 현재 구간의 시작을 입증할 수 없으면 UNCONFIRMED와 null 기한을 반환한다.

우선순위에 사용하는 권한 후보도 원천 위치뿐 아니라 조회 연결의 현재 열람 권한·ACTIVE 상태·범위 활성·인증 참조·승인 baseline·정확한 키 주체 일치를 통과해야 한다. 오래된 권한·철회된 연결·다른 키의 관측으로 우선 신호를 만들지 않는다.

GitHub PAT 권한 보조는 GitHub 수집 원천과 지원 PAT 유형의 조합에서만 실행 가능하다. Notion에서 발견한 GitHub PAT는 인증 지원과 별도로 권한 보조를 미지원으로 표시한다. Notion 발급 토큰도 관측기 등록 전 미지원이며 연결 선택을 요구하지 않는다.

원천 공개 메타데이터와 키 원문이 동시에 관측됐다는 증거가 없으면 `ACCESS_COEXISTENCE_UNCONFIRMED`를 반환한다. GitHub blob SHA만으로 ACL revision 일치를 주장하지 않는다. 공개 후보는 확인 질문으로 보존하고 현재 접근 범위는 UNKNOWN으로 표시한다.

평가 snapshot의 적용 시점 해석 버전은 risk-applicability-v1이다. 기존 snapshot에 해당 버전이 없으면 과거 결과는 보존하고 현재 적격성을 false로 반환하며 재평가를 안내한다.

## 9. AI 조사 요약 API

다음 경로는 `/api/v1/secrets` 아래다. 기존 `/cases/{caseId}/guide`의 합성 TEST 제한·응답은 유지하며 새 자원에 적용하지 않는다. 전체 계약은 [AI 조사 요약](risk-briefing-contract.md)을 따른다.

| Method·경로 | 입력과 결과 |
|---|---|
| GET /occurrences/{occurrenceId}/risk-briefing | 서버 기본 설명과 현재 적격 AI 결과. source=DEFAULT/AI, providerCalled, cached, reasonCode, 생성 시각·기준 평가·적격성. 외부 호출 없음 |
| POST /occurrences/{occurrenceId}/risk-briefing-requests | SECURITY·현재 위치 권한·CSRF. expectedAssessmentId, expectedEvidenceRevision, idempotencyKey. 202와 requestId·상태 |
| GET /occurrences/{occurrenceId}/risk-briefing-requests/{requestId} | 같은 위치에 결합된 요청 상태·허용 결과. QUEUED/RUNNING/SUCCEEDED/FALLBACK/SUPERSEDED/CANCELLED |

입력은 내부 대상·기준 버전만 받는다. 키·사실 본문·등급·prompt·endpoint·자유 메모를 받지 않는다. 평가가 없으면 ASSESSMENT_REQUIRED, 메타데이터 전용이면 대상 제외, 현재 권한 결합 불가이면 CONTEXT_NOT_AUTHORIZED로 처리한다. 숨은 근거의 존재·이름·개수를 오류에 포함하지 않는다. 동일 멱등키·동일 본문은 같은 요청, 다른 본문 또는 기준 버전 불일치는 409다. 기존 성공 요청의 현재 적격성이 만료돼도 과거 성공 기록은 유지한다.

## 10. 사용 이력 API

다음 경로도 `/api/v1/secrets` 아래다. 전체 계약은 [사용 이력](usage-observation-contract.md)을 따른다.

| Method·경로 | 입력과 결과 |
|---|---|
| GET /{credentialId}/usage?occurrenceId=... | page=1·pageSize=20(최대 100). 선택 위치·키의 허용 사용 사실, 요청 상태·조회 기간/범위·커버리지·결합 수준·미지원 사유, 별도 iamLastUsed |
| POST /{credentialId}/usage-requests | occurrenceId, serverRegisteredTargetId, windowStart?, windowEnd?, expectedCredentialRevision, expectedScopeRevision, idempotencyKey. 생략 기간은 최근 7일. 현재 역할·원천·승인 범위 재검사 후 202와 요청 상태 |
| GET /usage-requests/{requestId} | 허용 요청의 대기·실행·완료·부분 실패·0건 결과 요약. raw provider body 없음 |
| POST /{credentialId}/usage-observations | SECURITY 또는 할당 REMEDIATOR. occurrenceId, 현재 credential/scope revision, typed sourceRef, 기간·행위 코드·결과·대상 참조·idempotencyKey. 201, MANUAL 자료 제출 |
| POST /usage-observations/{observationId}/decisions | SECURITY. ACCEPT/REJECT/WITHDRAW, expectedObservationRevision, reasonCode, idempotencyKey. 200, 원행 수정 없이 결정 추가 |

작업 등록은 임의 계정·리전·키 ID·endpoint·로그 본문을 받지 않는다. 별도 수동 자료 API도 승인된 구조화 사실만 받고 원문 로그·토큰·자유 URL은 거부한다. 요청과 결과 모두 현재 권한을 검사하고 등록 후 변경·철회 시 취소하거나 현재 적용을 막는다. 동일 멱등키의 다른 본문·기준 버전 불일치는 409다. 수동 자료는 PROVIDER 관측으로 승격하지 않으며 EXACT_CREDENTIAL 결합은 서버 증명이 있어야 한다.

수동 자료 본문은 `{occurrenceId,expectedCredentialRevision,expectedScopeRevision,sourceRef:{type,id},targetRef:{type,id},windowStart,windowEnd,eventAt,actionCode,actionResult,actorAttribution,idempotencyKey}`다. sourceRef는 같은 위치 또는 승인 범위의 INVESTIGATION_HISTORY/ACTION_RECORD, targetRef는 현재 APPROVED_SCOPE/연결 ASSET만 허용한다. 등록 자료·대상 목록은 서버가 제공하며 읽기용 표제·시각을 제출 본문에 복사하지 않는다. 시각은 UTC, 수동 자료의 발생 시각은 [windowStart, windowEnd)에 포함된다. 자료 제출로 공급자 이벤트나 정확한 키 결합을 만들지 않는다.

자료와 원본 조사·조치 기록의 읽기 권한을 각각 검사한다. 대응 건 종료만으로 SECURITY에게 허용된 기존 사용 관측과 결정 이력을 숨기지 않는다. 종료된 조치 기록을 새 자료의 근거로 선택하거나 새로 수용하는 조건과 과거 이력 열람·철회 조건은 별도로 적용한다. 권한 철회와 자격 증명·원천 기준 변경은 현재 적격성에 계속 반영한다.

조회는 `support`, `capabilities`, `keyIdentifierStatus`, `targetOptions`, `availableSourceRefs`, `availableTargetRefs`, `latestRequest`, `iamLastUsed`, `observations:{items,page,pageSize,totalElements}`를 제공한다. 요청 상태는 PENDING/RUNNING/COMPLETE/PARTIAL/UNAVAILABLE/CANCELLED다. 관측은 actionCode와 등록 operationCode, serviceCode, 결합 수준·커버리지·행위 결과·출처·수동 검토 상태·결정 이력을 제공한다. allowedDecisions와 observationRevision에 따라 검토하고 과거 조건의 자료는 현재 사실과 구분한다. 계정은 안전한 별칭으로 표시하고 수집 연결 인증 참조·원문 승인 URI·키 지문은 응답하지 않는다.
