# Secret API 계약

기준: [PRD](../01-requirements/PRD.md), [데이터 계약](data-contract.md), [데이터베이스 스키마](database-schema.md), [워크플로 상태](../02-functional-design/workflow-and-states.md), [대응 정책](../02-functional-design/response-policy.md), [자격 증명 유형](../04-reference/credential-types.md)

Secret 기능은 `/api/v1/secrets` 아래에서 승인된 수집 범위, 키 인벤토리, 위치별 근거와 대응을 제공한다. API는 비동기 작업 등록·조회와 권한 있는 판정·기록을 담당하고 별도 Java 워커가 정기·수동 수집, 탐지와 승인 검증을 실행한다. 서버 시작은 스캔을 실행하지 않는다. 이 문서는 제품의 단일 API 명세이며 구현 진행률이나 배포 상태를 나타내지 않는다.

## 1. 공통 계약

모든 조회·변경은 공통 대시보드 세션 인증을 사용한다. POST·PATCH·PUT·DELETE는 CSRF와 서버 측 역할·연결·위치 권한을 검사한다. `ADMIN`이라는 로그인 역할만으로 `SECURITY`, `REMEDIATOR`, `OPERATIONS`를 부여하지 않는다. 기존 계정을 식별 주체로 사용하며 공개 회원가입이나 별도 로그인 방식을 도입하지 않는다. 공통 형식은 [대시보드 API 명세](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)를 따른다.

응답 필드는 camelCase, enum은 대문자, 시각은 UTC ISO 8601이다. 키 원문·연결 비밀·HMAC 지문·전체 공급자 응답·비신뢰 원천 문맥을 요청·응답·오류·일반 로그에 넣지 않는다. 제한된 증거 참조는 `TICKET:SEC-123`, `CHANGE:OPS-123`, `POLICY:SECRET-12`처럼 서버가 등록한 유형과 식별 형식만 허용한다. 임의 URL·자유 메모·파일 업로드는 받지 않는다. 권한 없는 존재·수·활동 시각을 노출하지 않는다.

조회 scope와 unit은 `workbench/Mixed`, `currentCases/ResponseCase`, `inventory/CredentialSet`으로 구분한다. 대응 과거 목록의 단위는 `ResponseCase`, 실행 이력의 단위는 `ScanRun`이며 실행 목록은 `HISTORY` 읽기 스냅샷을 사용한다. 운영 기본은 `origin=PRODUCTION`이며 `TEST`는 명시적 시험 보기로 분리한다. `UNCLASSIFIED`는 출처 확인 대상으로 별도 조회하고 운영·시험 대응 수에 섞지 않는다. `source`는 발견 플랫폼, `provider`는 키 발급자다. 스캔 이력의 `provider`만 수집 연결 플랫폼이라는 용어 예외를 갖는다. 환경은 `ALL|PROD|DEV|UNKNOWN`이고 `ALL`은 UNKNOWN을 포함한다.

페이지는 0부터, 크기는 1–100(기본 20)이다. 목록은 `{items,total,page,size,filters,scope,unit,asOf,snapshotId,expiresAt}`와 해당 조회의 확인 범위를 반환한다. `total`은 같은 권한·필터의 전체 대상 수다. 목록·카드·분포의 같은 스냅샷은 같은 단위·필터에서 일치한다. workbench 최상위 `total`은 null이며 서로 다른 단위의 지표를 합산하지 않는다. 필터·시간·스냅샷은 5.6·5.12절을 따른다.

변경은 인증 사용자로 수행자를 채우고 최신 `expectedVersion` 또는 명시된 대상별 버전을 검사한다. 저장 결과·버전·안전한 감사·멱등 결과를 같은 트랜잭션에 확정한다. 멱등 범위는 요청자·명령·키이며 동일 키·다른 본문은 409, 같은 본문의 재시도는 같은 결과다. 읽기 스냅샷은 쓰기 허가 근거가 아니다. 변경 성공 시 `requiresSnapshotRefresh=true`를 반환한다.

오류 필수 필드는 `code`, `message`, 선택 필드는 안전한 `requestId`, `retryable`, `details`다. 400 `INVALID_INPUT`, 401 `UNAUTHENTICATED`, 403 `FORBIDDEN`, 404 `NOT_FOUND`, 405 `METHOD_NOT_ALLOWED`, 409 `CONFLICT`, 422 `EVIDENCE_REQUIRED`, 429 `RATE_LIMITED`, 5xx 서버 장애를 사용한다. `SNAPSHOT_EXPIRED`, `STALE_REVIEW_CONTEXT`, `CLOSURE_BLOCKED` 같은 충돌 사유는 `details.reasonCode`에 둔다. 공급자 401·403·404는 대시보드 인증 오류와 구분해 안전한 검증 사유로 기록한다.

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
| `GET /api/v1/secrets/cases/{caseId}/activities` | 허용된 담당·조치·체크리스트·재검사·예외·해결의 시간순 이력. happenedAt DESC, activityId DESC, 페이지 분리 |
| `GET /api/v1/secrets/assignees?caseId=&occurrenceId=` | 해당 위치 연결의 적격 REMEDIATOR와 활성 팀 소속. SECURITY 권한과 후보 대상의 역할을 서버에서 확인 |
| `GET /api/v1/secrets/cases/{caseId}/revalidations` | 허용 위치의 재검사 이력. revalidationId·occurrenceId·runId·canViewRun·jobStatus·result·failureCode·requestedAt·finishedAt. 실행 링크는 canViewRun=true일 때만 제공 |
| `GET /api/v1/secrets/cases/{caseId}/closure-checks` | 최신 근거의 해결 검사·버전·허용 확인 수준. 5.3·5.9·5.11절 |
| `GET /api/v1/secrets/connections` | 사용자가 수동 실행할 수 있는 활성 연결의 ID·안전한 표시명·범위 수. 연결 비밀 제외 |
| `GET /api/v1/secrets/connections/manage` | 연결 운영·승인 상태와 읽기 결과. 5.8절 |
| `GET /api/v1/secrets/runs`, `GET /api/v1/secrets/runs/{runId}` | 실행 당시 범위·단계·결과·검사 누락. 실행 목록의 단위 ScanRun·HISTORY 스냅샷. 5.4절 |
| `GET /api/v1/secrets/review-queue`, `GET /api/v1/secrets/occurrences/{occurrenceId}/review-context` | 역할과 단위가 고정된 대기열·위치별 판단 근거. 5.2절 |
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
| `GET /cases/{caseId}/exceptions` | 현재·과거 예외, 허용 위치 ID, `currentlyApplicable`, `canApprove`, `allowedReasonCodes`, `allowedRevokeReasonCodes`, `eligibleOccurrences`, `maxDurationDays`, `fullyRiskAccepted`, `restrictedCoverage`. 승인·취소·만료·범위 변경 상태를 구분 |
| `POST /cases/{caseId}/exceptions` | 위치 집합·사유 코드·안전한 근거·만료 시각·기대 버전. SECURITY가 위치별 범위·승인 권한·중복 활성 예외 검사 |
| `POST /cases/{caseId}/exceptions/revoke` | 예외 ID·취소 사유·기대 버전. 만료·취소 뒤 남은 위험에 맞게 조치 재개 |
| `POST /cases/{caseId}/guide` | 본문 없이 허용 사용자에게 버전 고정 안내 반환. 상태·DB 변경 없음 |

외부 조치 종류는 ROTATED·REVOKED·REMOVED_FROM_SOURCE·PERMISSION_REDUCED다. 외부 수행 시각은 요청보다 최대 5분 뒤부터 최근 30일 사이만 허용한다. 한 위치에 여러 기록을 남길 수 있고 PERMISSION_REDUCED는 임시 완화일 뿐 제거·폐기 증거를 대신하지 않는다. 체크리스트 기본 코드는 IDENTIFY_OWNER·ROTATE_OR_REVOKE·REMOVE_EXPOSURE·CHECK_ACCESS_LOGS이며 상태는 TODO·DONE이다. 체크리스트는 조치 주기·재검사·해결의 필수 사실을 대체하지 않는다.

재검사 작업 상태는 PENDING·LEASED·SUCCEEDED·FAILED·CANCELLED, 결과는 PENDING·PRESENT·MISSING·UNKNOWN이다. MISSING은 마지막 실질 조치 이후 같은 승인 범위 revision을 정상 읽고 필수 탐지가 성공했으며 설정·엔진·HMAC 버전이 비교 가능한 경우에만 인정한다. 불완전·실패·비교 불가는 UNKNOWN과 안전한 사유다. 한 위치 결과를 전체 원천·과거 Git 이력·다른 복사본의 해소로 확장하지 않는다. 지원하지 않는 원천의 재검사는 capability=false로 등록을 거부한다.

가이드 응답은 `{version,source:"DEFAULT|GROQ",title,steps:[{code,title,description}],notice}`다. 기본 가이드는 외부 API 없이 동작한다. 선택 AI는 별도 활성화·키·확인된 합성 TEST 조건을 모두 충족할 때만 호출하며 장애 시 기본 안내로 대체한다. 키 원문·마스킹 끝자리·HMAC·저장소·계정·경로·자유 메모는 외부 AI에 보내지 않는다.

## 4. 공유 데이터 경계

인증 주체는 공통 dashboard_users를 참조하고 Secret 역할·활성 팀·연결/위치 권한을 별도로 검사한다. SECURITY는 근거 검토·판정·조정·예외·해결, REMEDIATOR는 할당 위치의 외부 조치·재검사, OPERATIONS는 연결·승인 범위 운영을 수행한다. 화면 capability는 보조 정보이며 모든 명령은 서버에서 다시 검사한다.

일반 탐지 detection_events와 Secret 전용 모델은 분리한다. 통합 위험 조회의 Secret 단위는 현재 PRODUCTION ResponseCase 한 건이며 TEST·고유 키·Finding·위치 수를 위험 수에 더하지 않는다. Secret 기록을 일반 탐지에 복제 적재하지 않고 API에서 원본 영역·ID·단위를 명시해 합성한다. 일반 탐지와 Secret의 기준시점·필터는 공통 통합 계약으로 고정하고, 미평가·UNKNOWN을 점수나 임의 기본값으로 바꾸지 않는다. 공통 통합 URL·응답의 소유권은 대시보드 설계에 둔다.

## 5. 위치별 판단·증거·집계

### 5.1. 연결 승인과 출처 교정

`OPERATIONS`가 범위를 제안하고 별도의 `SECURITY` 권한자가 승인한다. 출처를 `TEST`로 신규 승인하거나 `PRODUCTION`에서 `TEST`로 교정하는 경우 제안자와 승인자는 달라야 한다. 연결 승인 화면은 수집 허가, 대응 출처, 보관 허용 정책을 별도 필드와 설명으로 보여준다. 승인 정책이 적용되는 범위·브랜치·경로·콘텐츠 유형과 만료일을 검사하고 혼합된 시험·운영 범위는 분리한다.

| Method | 경로 | 요청 또는 응답의 핵심 |
|---|---|---|
| GET | `/api/v1/secrets/scopes/{scopeId}/classification` | 현재 `origin`, `originVersion`, nullable `storagePolicyId`, `storagePolicyVersion`, 승인 상태·시각·수행자, 안전한 정책 요약과 변경 이력. `approved_ref`·계정·자원명 제외 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals` | `{origin,storagePolicyId?,storagePolicyVersion?,reasonCode,evidenceRef,expectedScopeVersion}`. `OPERATIONS` 제안, 검증 가능한 내부 정책·티켓 참조 필수. 정책이 없으면 두 필드를 모두 null로 두고 `reasonCode=POLICY_PENDING`; 보관 판정은 `UNKNOWN`. `PENDING_APPROVAL` 반환 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals/{proposalId}/approve` | `{expectedScopeVersion,expectedProposalVersion}`. 승인자는 서버 세션에서 결정. 승인 순간 범위 버전 증가·추가 전용 변경 이력·영향 대상 목록·집계 무효화가 원자적 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals/{proposalId}/reject` | `{reasonCode,expectedProposalVersion}`. `SECURITY`가 거절 사유와 이력을 남기고 이전 승인 분류를 유지. 승인 대기 제안을 활성 분류로 읽지 않음 |

미분류 또는 승인 대기 범위는 `/api/v1/secrets/scopes?classificationStatus=NEEDS_REVIEW`에서 운영·시험 집계와 별도로 찾는다. 승인 전에도 이전 탐지·실패 기록은 열람할 수 있고 조사 과제를 만들 수 있다. 범위 교정은 과거 대응 건의 출처를 직접 UPDATE하지 않는다. 영향 대상별 이전·새 건의 관계와 승인 근거를 추가 전용으로 기록하고 현재 분류의 새 스냅샷을 만든다. 이전 판정과 대응 건의 출처는 이력으로 보존한다.

보관 정책은 `SECURITY`가 `POST /api/v1/secrets/storage-policies/drafts`에 연결·범위·콘텐츠 유형·정확한 경로/접두사 조건, `ALLOW|DENY`, 근거 참조, 유효 기간을 제출하고 **다른** `SECURITY`가 `POST /storage-policies/{policyId}/versions/{version}/approve`로 승인한다. 승인된 버전은 불변이며 `GET /storage-policies?status=ACTIVE`에서 권한에 맞는 안전한 요약만 조회·선택한다. 충돌하는 허용·금지 규칙은 활성화 전에 거부하고 평가 중 범위·버전 불일치나 만료는 `UNKNOWN`으로 남긴다. 정책 폐기는 새 승인 버전 또는 `POST /storage-policies/{policyId}/retire`의 사유·승인 기록으로 처리하며 기존 판정에 재검토 표식을 만든다. 화면은 보호된 `approved_ref`와 규칙의 민감한 전체 경로를 일반 사용자에게 노출하지 않는다. 범위 승인 API는 정책을 지정했다면 **활성 정책 버전**만 받으며, 정책 미설정 범위도 수집 승인은 가능하지만 보관 판정은 `UNKNOWN`·정책 작성 과제로 남는다. 정책 등록·승인·바인딩 모두 감사 기록과 버전 충돌 검사를 수행한다.

### 5.2. 검토 대기열과 근거

검토 대기열은 `GET /api/v1/secrets/review-queue?bucket=ACTION_REQUIRED|NEEDS_DECISION|NORMAL_INVENTORY|NEEDS_CASE_LINK&origin=PRODUCTION|TEST|UNCLASSIFIED&page={page}&size={size}`로 조회한다. `bucket`, `origin`, `page`, `size`는 허용 목록으로 검증하고 `ACTION_REQUIRED`는 범위별 `SECURITY` 또는 해당 위치에 할당되고 연결의 `REMEDIATOR` 권한을 가진 사용자에게 허용한다. 담당자 조회는 본인에게 허용된 위치가 있는 고유 건만 반환하며 다른 위치의 근거·수·담당자를 확장 공개하지 않는다. `NEEDS_DECISION`, `NORMAL_INVENTORY`, `NEEDS_CASE_LINK`는 범위별 `SECURITY` 권한을 요구한다. 응답은 `{items,total,page,size,unit,asOf,partialCoverage}`이며 `ACTION_REQUIRED`는 남은 작업이 있는 `OPEN|IN_PROGRESS|REVALIDATION_PENDING` 현재 `ResponseCase`, `NEEDS_DECISION`은 `UNKNOWN`·재검토 필요 `Occurrence`, `NORMAL_INVENTORY`는 `CredentialSet`, `NEEDS_CASE_LINK`는 유효한 비정상 판정이 있으나 현재 건에 연결되지 않은 `ABNORMAL` `Occurrence`가 단위다. 해결·전체 위험 수용 건은 `ACTION_REQUIRED`에 넣지 않는다. 서로 다른 단위의 `total`을 더해 고유 키 수로 표시하지 않는다. 항목에는 `credentialSetId`, 선택 가능한 `occurrenceId`, `caseId`, 안전한 키 이름, 구성·존재 상태, 출처, 노출 판정·심각도·검증 각각의 상태, 마지막 성공·최근 시도, 정책 버전, 근거의 확인 수준, 누락 사유, 담당자·기한, 위치 수, `whyHere`, `nextAction`을 필드별 열람 권한 안에서 제공한다. `whyHere`와 `nextAction`은 사용자 자유문이나 생성형 AI 문장이 아니라 서버가 안전한 코드와 템플릿으로 만든다. 범위별 검사 누락은 키 대기열 수에 섞지 않고 `GET /api/v1/secrets/coverage-gaps`에서 `unit=Scope`로 제공한다.

`NORMAL_INVENTORY`의 키는 같은 출처 구분에 활성 대응 건과 판정 대기 위치가 없고, 위치 보관 `NORMAL`이 정책으로 증명됐거나 `METADATA_ONLY` 또는 폐기·만료가 공급자 근거로 확인된 경우에만 들어간다. 응답의 `inventorySubtype=AUTHORIZED_STORAGE|METADATA_ONLY|RETIRED_HISTORY`를 반드시 표시한다. `CANDIDATE`·사유 불명 `INVALID`·단순 건 없음은 정상 근거가 아니다. 허용 위치와 비정상 위치가 같이 있는 키는 정상 키 한 건으로 중복 표시하지 않고 대응·판정 상세에서 허용 위치를 보여준다. 전체 인벤토리 API는 이 대기열과 달리 해당 키를 계속 조회할 수 있다. `UNCLASSIFIED`는 정상 인벤토리로 자동 분류하지 않는다.

`GET /api/v1/secrets/occurrences/{occurrenceId}/review-context`는 화면의 단일 근거 카드다. 구성·발급자·원천 유형, 권한에 맞는 위치 표시, 해당 Finding의 룰·버전·탐지 시각, 현재 존재와 마지막 성공 검사/최근 시도, 범위 출처·보관 정책 ID/버전·적용 이유, 검증 결과·방법·시각, 확인된 주체·권한·영향, 동일 키의 열람 가능한 다른 위치, 기존 건, 부족 근거 코드와 후속 과제, `allowedDecisions`, 각각의 예상 버전을 반환한다. 각 사실은 `CONFIRMED|UNKNOWN|NOT_CHECKED`와 출처·관측 시각을 따로 가진다. 응답의 `UNKNOWN`은 무효·정상·낮은 심각도로 치환하지 않는다. 비밀 값, 마스킹 전후 문맥, HMAC 지문, 권한 밖 위치 수와 원천 경로는 반환하지 않는다. `404`와 `403`은 안전한 오류 화면 및 복귀 경로로 처리하고 로딩 중 null을 결론처럼 렌더링하지 않는다.

`GET /api/v1/secrets/investigations?status=OPEN|OVERDUE&owner=ME&page=&size=`와 `GET /investigations/{taskId}`는 위치/범위 구분, 부족 근거 코드, 담당자·기한, 제한된 대상 링크와 최근 활동을 조회한다. `PATCH /investigations/{taskId}/assignment`는 `{assigneeUserId,dueAt,reasonCode,expectedVersion}`, `POST /investigations/{taskId}/evidence`는 `{evidenceRef,evidenceKind,observedAt,expectedVersion}`, `POST /investigations/{taskId}/close`는 `{expectedVersion,resolvingDecisionId|restoredRunScopeId}`를 받는다. 관련 위치가 새 판정으로 확정되거나 누락 범위가 성공적으로 다시 검사된 근거가 없으면 과제를 닫을 수 없다. 범위별 역할을 다시 검사하고, 담당자 변경·기한 초과·증거·종료를 추가 전용 이력에 남긴다. 과제 종료만으로 `NORMAL`, `ABNORMAL` 또는 검사 성공을 만들지 않는다.

`TEST` 위치의 `fixtureStatus=NOT_APPLICABLE|VERIFIED_SYNTHETIC|POSSIBLE_REAL|CONFIRMED_REAL`은 출처 구분과 독립이다. `CANDIDATE`·`METADATA_ONLY`와 현재 유효 Finding이 없는 위치는 `NOT_APPLICABLE`이며 조사·인벤토리에는 남긴다. `VERIFIED_SYNTHETIC`은 승인된 fixture 목록·버전과 **해당 키**가 일치한 근거가 있을 때만 반환한다. `POSSIBLE_REAL`은 `CONFIRMED` 값 후보가 성공한 최신 검사에서 `PRESENT`이고 유효한 Finding이 남았으며 fixture 일치를 확인했지만 합성으로 증명되지 않았다는 뜻이다. 목록 확인 실패도 사유와 함께 이 상태로 둔다. 이는 유효성 `VALID`와 같지 않다. **이 조건이 충족되는 즉시** `GET /api/v1/secrets/cross-environment-alerts`에 `SUSPECTED_REAL` 운영 보안 확인 경고를 열고 담당자·기한·권한에 맞는 위치 링크를 제공한다. `testCaseId`는 건이 있는 경우에만, 개설 조건을 충족하지 못한 후보의 `investigationTaskId`는 조사 과제가 있는 경우에 제공한다. 두 ID가 모두 없는 경고는 조사 과제를 원자적으로 만든다. 연결의 기본 적격 SECURITY 담당자·기한 정책이 있는 필드만 채우며 미설정이면 담당자와 기한을 각각 nullable로 두고 경고 생성을 막지 않는다. 배정/기한 미설정을 표시하고 `ASSIGN_INVESTIGATION`으로 안내하며 기한이 없으면 기한 초과 수에 넣지 않는다. 사람이 `NEEDS_MORE_INFO`를 제출하는 경우의 담당자·기한 필수 조건은 그대로다. 같은 CredentialSet의 반복 검사·추가 위치는 열린 경고를 갱신하고 새 경고를 만들지 않는다. 실제 발급자 근거가 들어오면 같은 경고를 `CONFIRMED_REAL`로 승격한다. 경고의 `acknowledged`는 인지 기록이고 조치 완료가 아니다. TEST 건은 TEST 집계에 남지만 운영 화면의 이 경고를 숨기지 않는다. 경고 수는 운영 `ResponseCase` 수와 별도 단위로 표시한다. 합성 여부가 불명확한 동안 시험 종료는 막는다.

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

`GET /api/v1/secrets/summary?origin=PRODUCTION&source=&provider=&environment=&connectionId=&assignee=&snapshotId=`가 현황 진입점이다. `origin=PRODUCTION|TEST`를 받으며 기본은 `PRODUCTION`이다. `UNCLASSIFIED`는 아래 별도 지표와 출처 확인 목록으로만 조회하고 운영·시험 건에 합산하지 않는다. `source`·`environment`·`connectionId`는 **같은 허용된 위치**에서 AND로 적용하고, `provider`는 키 발급자에 적용한다. `assignee=ME|UNASSIGNED|사용자ID`는 건 또는 위치 담당 기록이 있는 대응 지표에만 적용한다. 해당하지 않는 검사 누락 지표에 임의로 같은 필터를 적용하지 않고 `appliedFilters`와 `notApplicableFilters`에 이를 명시한다. 파라미터는 서버 허용 목록으로 검증하며 임의 SQL 정렬·전체 원문 검색은 받지 않는다.

판정·건 연결·정상 인벤토리 지표는 연결/위치별 `SECURITY` 범위에서, 조치 지표는 `SECURITY` 또는 본인 할당 위치의 `REMEDIATOR` 범위에서, 검사 누락·출처 분류 지표는 허용된 `OPERATIONS|SECURITY` 범위에서 산출한다. 역할을 갖지 못한 지표는 `NOT_AUTHORIZED`로 남긴다. 담당자용 응답은 `accessScope=ASSIGNED_LOCATIONS`, 보안 범위 조회는 `AUTHORIZED_LOCATIONS`를 명시하며 자신이 확인한 위치만으로 건 전체가 해결됐다고 결론 내리지 않는다. 기존 로그인 `ADMIN` 값이 이 권한을 대신하지 않는다.

이 요약은 서로 다른 단위를 담으므로 최상위는 `scope=workbench`, `unit=Mixed`, `total=null`이다. `workbench`·`Mixed`는 합성 조회용 새 식별값이며 `currentCases`·`ResponseCase`의 의미를 바꾸지 않는다. 대응 건 전용 요약·분포는 `caseSummary` 하위 블록으로 제공한다. 각 지표는 아래의 고유 단위를 유지한다.

| 지표 키 | 집계 대상·단위 | 같은 스냅샷의 도착 목록 |
|---|---|---|
| `ongoingCases` | 현재 같은 출처의 `OPEN\|IN_PROGRESS`, `ResponseCase` | `review-queue?bucket=ACTION_REQUIRED&stages=OPEN,IN_PROGRESS` |
| `revalidationPendingCases` | 현재 같은 출처의 `REVALIDATION_PENDING`, `ResponseCase` | `review-queue?bucket=ACTION_REQUIRED&stages=REVALIDATION_PENDING` |
| `needsDecisionLocations` | `UNKNOWN` 또는 근거가 낡아 재검토가 필요한, 유효 Finding이 남은 위치, `Occurrence` | `review-queue?bucket=NEEDS_DECISION` |
| `needsCaseLinkLocations` | 유효 `ABNORMAL`이 현재 건에 연결되지 않은 무결성 복구 대상 위치, `Occurrence` | `review-queue?bucket=NEEDS_CASE_LINK` |
| `coverageGapScopes` | 5.8절의 현재 필수 승인 범위 중 검사 누락 사유가 있는 고유 범위, `Scope` | `coverage-gaps` |
| `unclassifiedScopes` | 현재 출처 미분류·승인 대기 범위, `Scope` | `scopes?classificationStatus=NEEDS_REVIEW` |

미지정 건은 `ongoingCases`·`revalidationPendingCases`의 **부분집합**이며 별도 추가 노출 건수로 더하지 않는다. 같은 키의 운영·시험 위치가 있어도 위치는 해당 출처에서만 세고 운영·시험 건은 분리한다. 시험 실키 의심 경고는 사용자에게 열람 권한이 있는 경우 `crossEnvironmentAlerts`에 `unit=CrossEnvironmentAlert`로 따로 제공한다. 이는 운영 출처 필터에서도 숨기지 않지만 운영 `ResponseCase` 수에 더하지 않는다. 조사 과제 수나 인벤토리 전체 키 수를 노출·대응 수로 합산하지 않는다.

`NORMAL_INVENTORY`는 `inventorySubtype=METADATA_ONLY|AUTHORIZED_STORAGE|RETIRED_HISTORY`를 구분한다. 승인 보관은 같은 출처의 허용 위치 전체에 유효 Finding·활성 ALLOWED 정책·NORMAL 근거가 있고 활성 대응/재검토가 없는 키다. RETIRED_HISTORY는 호출자가 열람 가능한 현재 유효 PROVIDER_VERIFIED 폐기·교체 근거가 있으며 같은 출처에 활성 대응·판정 대기·현재 존재/판정 UNKNOWN이 없는 키만 포함한다. 수동 승인·INVALID·만료 시각 추정만으로 이 분류를 만들지 않는다. 소스/환경 필터가 다른 위치를 가렸다는 이유로 승인 보관·폐기 이력에 올리지 않고 숨은 근거로 상태·수를 생성하지 않는다.

`stages`는 `ACTION_REQUIRED`에서 `OPEN|IN_PROGRESS|REVALIDATION_PENDING`의 중복 없는 부분집합, `CURRENT_CASES`에서 이들과 `RESOLVED|RISK_ACCEPTED`의 중복 없는 부분집합을 쉼표로 받는다. 다른 대기열에 stages를 보내거나 빈 값·중복·알 수 없는 단계를 보내면 400이다. 전체 위험 수용은 모든 관련 위치의 SECURITY 권한이 있을 때만 별도 파생 단계로 제공한다. 일부 위치만 열람하는 사용자에게는 원래 조치 단계와 restrictedCoverage를 제공하며 숨은 예외의 승인·만료로 단계를 바꾸지 않는다.

지표의 공통 형태는 `{key,scope,unit,value,status,reasonCode,appliedFilters,notApplicableFilters,queueFilter}`다. 등록된 지표마다 한 항목을 반환하고 `status=AVAILABLE`일 때만 `value`는 0 이상의 정수다. `caseSummary`, `normalInventory`, `crossEnvironmentAlerts`도 각각 자신의 status와 단위를 유지한다. 권한 없는 하위 합계·분포는 null이고 경고 목록은 빈 배열이며 권한 밖 수를 0으로 추정하지 않는다. 역할이 없으면 `NOT_AUTHORIZED`, 과거 자료로 집계를 복원할 수 없으면 `LEGACY_UNAVAILABLE`, 집계 실패는 `UNAVAILABLE`이며 이때 `value=null`이다. 실제 0, 미지원, 권한 없음과 읽기 실패를 구별한다. `queueFilter`는 서버가 만든 허용 필터 객체이며 임의 URL·공급자 링크를 반환하지 않는다. `partialCoverage`는 열람 가능한 필수 범위에서 확인된 누락 여부이며, 집계 근거·권한이 없으면 null과 같은 상태·사유를 반환한다. 권한 제한은 별도 `restrictedCoverage`로 설명하고 누락 0건으로 간주하지 않는다. 화면 라벨은 [문구와 표시 기준](../02-functional-design/content-and-labels.md)을 따른다.

```json
{
  "scope": "workbench",
  "unit": "Mixed",
  "total": null,
  "origin": "PRODUCTION",
  "filters": {"origin":"PRODUCTION"},
  "snapshotId": "example-workbench-snapshot",
  "asOf": "2026-10-21T00:00:00Z",
  "expiresAt": "2026-10-21T00:15:00Z",
  "metrics": [
    {"key":"needsDecisionLocations","scope":"reviewQueue","unit":"Occurrence","value":3,"status":"AVAILABLE","reasonCode":null,"appliedFilters":{"origin":"PRODUCTION"},"notApplicableFilters":[],"queueFilter":{"bucket":"NEEDS_DECISION","origin":"PRODUCTION"}},
    {"key":"coverageGapScopes","scope":"coverage","unit":"Scope","value":1,"status":"AVAILABLE","reasonCode":null,"appliedFilters":{"origin":"PRODUCTION"},"notApplicableFilters":[],"queueFilter":{"gapOnly":true,"origin":"PRODUCTION"}}
  ],
  "caseSummary": {"scope":"currentCases","unit":"ResponseCase","total":2,"actionCounts":{"OPEN":1,"IN_PROGRESS":0,"REVALIDATION_PENDING":1,"RESOLVED":0,"RISK_ACCEPTED":0}},
  "partialCoverage": true
}
```

수치는 응답 모양을 설명하는 예시다. `caseSummary.total`에 대해서만 네 단계·별도 예외 수의 합을 검증하며 `metrics[].value` 합계를 만들지 않는다. `severityCounts`는 CRITICAL·HIGH·MEDIUM·LOW·UNASSESSED를 구분하며 같은 total에 일치한다. 모든 위치가 미평가이면 대표 심각도도 UNASSESSED다. 분포에서 제외한 시험 건·오탐/범위 밖 교정 종료·정상 인벤토리의 수를 끼워 넣지 않는다. 근거 또는 집계 결과가 없으면 심각도·매트릭스는 `UNAVAILABLE`과 사유를 반환하고 기본 등급을 만들지 않는다. 관련 건의 일부 위치만 볼 수 있으면 그 사용자가 열람 가능한 건·위치로 집계한 `visible` 범위임을 응답에 밝힌다. 제한 위치의 정확한 수·이름·미할당 여부를 추론할 수 있는 차감 수치를 주지 않는다. 사용자 전체의 안전 여부나 건 해결 가능성을 이 부분 집계에서 결론 내리지 않는다.

최초 요약 요청은 인증 사용자·역할 부여 revision·허용 연결/범위 revision·정규화 필터·출처에 묶인 스냅샷을 `REPEATABLE READ`에서 만든다. `asOf`는 서버 DB 시각이며 TTL 상한은 **15분**, `expiresAt`을 반환한다. 열람 가능한 정책·오탐 재사용·전체 위험 수용 예외의 효력이 먼저 끝나면 그 시각까지 유효 기간을 줄인다. 같은 ID는 요약, 대기열, 대응 목록, 검사 누락 목록의 동일 필터 투영에만 쓴다. 큐의 `total`은 해당 카드와 같아야 한다. 필터 또는 시험 보기 전환은 새 스냅샷을 요구한다. 같은 사용자·조회 scope에서 필터나 출처가 다르면 409 `SNAPSHOT_MISMATCH`, 역할 취소·범위 권한 변경이면 409 `SNAPSHOT_ACCESS_CHANGED`로 거부한다. 만료·상한 초과로 폐기한 ID는 보존한 폐기 표식으로 409 `SNAPSHOT_EXPIRED`를 구분한다. 존재하지 않거나 다른 사용자·조회 scope의 ID는 404다. 권한이 취소된 이전 행은 만료 전에도 반환하지 않는다. 오류 코드는 `code=CONFLICT` 아래 `details.reasonCode`로 제공한다.

스냅샷을 오래된 상태의 쓰기 근거로 쓰지 않는다. 상세 진입은 현재 상태를 읽고 `readAt`, 버전, `changedSinceSnapshot`을 표시하며, 판정·조치는 최신 버전으로 다시 검사한다. 변경 성공 후에는 `requiresSnapshotRefresh=true`를 반환하고 화면이 요약·목록을 함께 다시 읽는다. 갱신하지 않은 카드만 감소시키는 클라이언트 추정 집계는 금지한다. 스냅샷 생성은 화면 초기 진입·명시 새로고침·변경 후 재조회에 한정하고 초 단위 폴링마다 새로 생성하지 않는다. 사용자별 동시 유효 스냅샷 상한은 5개이며 넘으면 가장 오래된 스냅샷을 만료시키고 기존 조회에 같은 새로고침 흐름을 적용한다.

기간은 지표마다 의미가 다르므로 현황의 현재 상태 카드에는 공통 `from/to`를 받지 않는다. `/cases?changedFrom=&changedTo=`는 `updatedAt`의 `[from,to)`, `/review-queue?observedFrom=&observedTo=`는 위치의 `lastSeenAt`의 `[from,to)`, `/runs?from=&to=`는 `requestedAt`의 `[from,to)`다. 날짜만 있는 입력은 화면이 사용자의 표시 시간대 기준 경계를 UTC로 변환한다. 불완전 경계·역전·형식 오류는 400이다. 기간 필터가 있는 도착 목록은 동일 기간이 적용된 별도 집계로 표시하고 무기간 현황 카드 수와 같다고 주장하지 않는다. 보존된 기간 밖 요청은 빈 결과를 실제 무발생으로 해석하지 않게 `retainedFrom`·`historyComplete=false`를 반환한다.

### 5.7. 키별 위치 시각화의 안전한 읽기 모델

`GET /api/v1/secrets/{id}/locations?origin=PRODUCTION&source=&environment=&presence=&page=&size=`를 키 독립 상세 `/secrets/:secretId`의 집계 경로로 둔다. 위치 권한은 서버에서 적용한다. 이 경로는 `SECURITY`가 허용된 위치, 또는 현재 할당받고 해당 연결의 `REMEDIATOR` 권한이 있는 위치에 접근할 때만 사용할 수 있다. `OPERATIONS`만으로 키 상세·판정·대응 증거의 읽기 권한이 생기지 않는다. 전체 연결 조회의 권한을 발견 키 조회 권한으로 확장하지 않는다.

응답은 `{credential,filters,visibleLocationCount,locationStateCounts,items,total,page,size,readAt,detailVersion,restrictedCoverage,capabilities}`다. `page`는 0부터, `size`는 1–100(기본 20)이며 `total=visibleLocationCount`는 같은 권한·필터로 허용된 위치의 수다. 정렬은 현재 미해결 위치 우선, `lastSeenAt DESC,occurrenceId DESC`로 고정한다. 같은 응답의 대표 위치·상태 수·총수·페이지는 하나의 DB 읽기 시점을 사용한다. 한 페이지의 길이를 전체 위치 수로 쓰지 않는다. 후속 페이지는 첫 응답의 `expectedDetailVersion`을 전달한다. 허용된 위치·권한·정렬 사실이 바뀌면 409 `DETAIL_CHANGED`로 첫 페이지 재조회를 요구해 이전 페이지와 최신 페이지의 중복·누락을 막는다. 이 버전은 허용된 사실의 불투명 버전값이며 숨겨진 위치 활동으로 바뀌지 않는다.

| 블록 | 필수 정보·의미 | 필드별 권한·비노출 |
|---|---|---|
| `credential` | ID, 발급자·유형·구성 상태·마스킹 이름, 최신 유효성 결과·사유·시각과 이전 마지막 성공, 심각도 또는 미평가 | 키 수준 결과도 호출자가 허용된 범위에서 열람할 수 있는 근거만 집계. 보이지 않는 위치의 발견·검증 활동을 시각·수치로 누출하지 않음 |
| 위치 식별 | `occurrenceId`, `source`, `environment`, `origin`, 범위 revision, `safeLocationLabel`은 검증된 범위 별칭과 `위치 #ID` 또는 ID만 사용 | 자동 추출한 원문 경로·원천 URL·ARN·계정 ID·HMAC·문맥 제외. 표시 별칭이 없으면 `위치 #ID`. 현재 별칭이 없거나 과거 별칭이 안전문자·토큰 검사를 통과하지 못하면 `위치 #ID`로 대체 |
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

관리 연결의 응답에는 현재 `connectionStatus`, 안전한 별칭, 승인 범위 수·revision, `latestProbe`, `lastFullScanAttempt`, `lastSuccessfulFullScan`, `nextDueAt`, `activeJob`, 권한에 맞는 상태별 범위 수를 분리한다. 연결 테스트는 STS 신원/설치 토큰 발급 확인이라는 `checkKind=IDENTITY_OR_INSTALLATION`이고 **수집 범위 권한·본문 접근·발견 키 유효성 확인이 아니다**. 테스트의 시각·안전한 사유 이력을 남겨도 연결 활성 상태나 수집 성공 시각을 변경하지 않는다.

범위별 `coverage`는 현재 `scopeRevision`, `lastAttempt`의 실행·시각·`scopeOutcome`, 동일 revision과 필수 단계 구성에서 마지막으로 성공한 `lastSuccessfulScan`, `nextDueAt`, `isPaused`, `isRunning`, `freshness`와 `gapReasonCodes[]`를 제공한다. `freshness=CURRENT|OVERDUE|NEVER_SCANNED|NOT_SCHEDULED`는 주기와 예정 시각으로 계산하는 읽기 정보이며 기존 수집 상태 enum을 바꾸지 않는다. `lastSuccessfulScan`을 최신 실패 시각으로 덮어쓰지 않고, 범위 변경 전의 성공을 현재 revision의 확인으로 재사용하지 않는다. 실행 ID가 있으면 스캔 상세로 연결하고 대상이 실행 전이면 연결·범위 복구로 안내한다.

`GET /api/v1/secrets/coverage-gaps?origin=PRODUCTION&source=&connectionId=&snapshotId=&page=&size=`는 현재 승인된 필수 범위의 revision마다 한 행을 반환한다. `unit=Scope`, `total`은 동일 권한·필터·스냅샷에서 누락 사유가 있는 고유 범위 수다. 사유는 `LAST_ATTEMPT_INCOMPLETE`, `NEVER_SCANNED`, `REVISION_UNCHECKED`, `OVERDUE`, `CONNECTION_PAUSED` 등 여러 개일 수 있으나 같은 범위는 한 번만 센다. 실패한 최신 시도가 있어도 마지막 성공은 함께 표시한다. 정상적으로 대기/진행 중인 실행 자체는 최종 실패로 세지 않으며, 이전 누락·기한 초과가 이미 있으면 실행 중이라는 이유로 숨기지 않는다. 선택 비활성 검증이나 메타데이터의 해당 없음 단계는 필수 수집 누락이 아니고 단계 사유로 남긴다.

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
| 시험 실키 의심 경고 | 해당 없음. TEST 위치 경고를 별도 표시 | 허용된 관련 위치 | 해당 키 발급자 | 허용된 관련 위치 | 해당 없음. 경고 담당자는 별도 경고 필터 |

`provider`와 `environment`를 미검사 범위의 발견 키·위치에서 역으로 추정하지 않는다. 해당 없음 필터는 지표의 `notApplicableFilters`로 반환하고 UI는 검사 누락 카드에 `환경 필터 미적용`처럼 적용 한계를 표시한다. 없는 대응 건에서 assignee를 추정하거나 작업 담당자 없는 범위를 ‘미지정 대응 건’으로 세지 않는다.

위치 API는 `exposure=ABNORMAL|NORMAL|UNKNOWN`, `actionStatus=OPEN|IN_PROGRESS|REVALIDATION_PENDING|RESOLVED|NO_CASE`, `expectedDetailVersion`을 받는다. 소스·환경·존재·노출·조치 필터는 같은 허용된 위치에 AND로 적용하고 단일 키 요약과 화면 필터의 수를 구분한다. 날짜 필터는 키 위치 표에 추가하지 않고 마지막 관측 시각을 표시한다. 기간은 활동/실행 목록에서 적용한다.

스캔 이력의 `GET /runs`는 5.4절 파라미터 외 `provider=GITHUB|AWS|NOTION|KUBERNETES`로 **수집 연결 플랫폼**을 필터할 수 있다. 실행 응답의 `provider` 필드와 같은 의미이며 키 발급자 필터가 아니다. 이 경로의 용어 예외를 UI에는 `수집 플랫폼`으로 표시한다. 연결·플랫폼·기간·실행 목적·상태는 같은 허용 실행에 AND로 적용하고 미지원 연결을 가짜 실행으로 채우지 않는다.

CSV는 새 export endpoint 없이 고유 키 인벤토리 `GET /api/v1/secrets`의 **동일 INVENTORY 스냅샷 전체 페이지**에서 만든다. 첫 요청의 정규화 필터(`origin`, `contentKind`, `workBucket`, 소스·발급자·환경·구성·검증 등), `scope=inventory`, `unit=CredentialSet`, `asOf`, `snapshotId`, `total`, `expiresAt`을 고정하고 후속 페이지에 같은 ID·필터를 전달한다. 권한은 첫 요청과 매 페이지에서 확인한다. 한 페이지 또는 현재 브라우저 행만 내보내고 ‘현재 조건 전체’로 표시하지 않는다.

내보내기 상한은 **고유 키 10,000개**, 페이지 크기는 최대 100이다. 상한을 넘으면 시작 전에 필터 축소를 안내하고 조용히 잘라 파일을 만들지 않는다. 스냅샷 만료·권한/범위 변경·페이지 오류·전체 행수 불일치가 생기면 내보내기를 중단하고 새 조회를 요구한다. 모든 페이지를 확인한 뒤 한 파일을 내려받으며 일부 파일을 완성본으로 제공하지 않는다. 전체 UI 카드용 WORKBENCH와 독립 생성한 INVENTORY의 ID를 바꾸어 쓰지 않는다.

열은 마스킹 키·발급자·유형·구성·선택 출처·허용 위치 수·검증 상태/시각·작업 이유·기준시각 등 허용된 요약으로 제한한다. 원문·문맥·HMAC·전체 경로·ARN·공급자 응답·권한 밖 위치 수·외부 증거 자유문은 제외한다. 시험 출처는 각 행과 파일의 기준 정보에 표시한다. UTF-8 CSV의 구분자·따옴표·줄바꿈을 올바르게 escape하고, 셀 앞의 공백을 포함해 `=`, `+`, `-`, `@` 등 수식 시작 문자를 탐지해 텍스트로 무력화한다. 필터·기준시점·단위·확인 범위는 파일의 요약 필드로 제공하며 실제 키 원문을 CSV 검증용으로 기록하지 않는다.
