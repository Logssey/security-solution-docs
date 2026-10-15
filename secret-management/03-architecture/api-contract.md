# Secret API 계약

기준: [PRD](../01-requirements/PRD.md), [ADR-S008](adr/ADR-S008.md), [데이터 계약](data-contract.md), [대응 정책](../02-functional-design/response-policy.md)

이 문서는 Secret 기능의 목표 계약과 현재 제공하는 MVP 경로를 구분한다. `GET /api/v1/secrets`, `/{id}`, `/connections`, 실행 등록·목록·상세 조회와 연결 테스트는 구현됐다. 10/15 대응 구현 브랜치의 V7에는 대응 건 개설·목록·상세와 담당자 지정의 기초 경로가 추가됐고, 요약·검토·체크리스트·외부 조치·재검증·스냅샷은 후속 설계다. 현재 구현 상태는 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)를 따른다.

## 1. 공통 계약

모든 조회와 변경은 기존 세션 인증 경계를 통과한다. 변경 요청은 POST·PATCH·PUT·DELETE 모두 CSRF 검증과 서버 권한 검사를 적용한다. 공급자 연결 자격 증명과 발견된 키 원문은 요청, 응답, 오류, 로그에 넣지 않는다. 응답 필드는 camelCase, 상태·분류 enum은 대문자, 시각은 UTC ISO 8601 형식을 사용한다. `scope`의 `currentCases`·`inventory`·`history`, `unit`의 `ResponseCase`·`CredentialSet`은 계약 식별 문자열이며 아래 표기를 유지한다. 공통 기준은 [대시보드 API 명세](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)를 따른다.

후속 집계 API는 `source`, `provider`, `environment`, `credentialType`, `validity`, `severity`, `actionStatus`, `exception`, `assignee`, 기간 필터 중 해당 경로에 맞는 값을 받는다. 응답은 적용된 `filters`, `scope`, `unit`, `asOf`, `snapshotId`, `total`, `partialCoverage`를 반환한다. 요약, 매트릭스와 기본 대응 목록은 `scope=currentCases`, `unit=ResponseCase`, 기본 대응 구분 `PRODUCTION`이고 그 구분에서 각 키의 최신 건만 센다. `TEST`는 명시적 시험 조회에만 포함하며 운영·통합 수치와 합치지 않는다. 인벤토리는 `scope=inventory`, `unit=CredentialSet`이다. 이력 목록은 `scope=history`, `unit=ResponseCase`로 과거 건을 각각 센다. 화면은 첫 조회의 `snapshotId`를 매트릭스와 목록에 전달한다. 같은 구분·스냅샷·필터의 카드, 매트릭스, 목록 건수는 일치해야 한다. 만료된 스냅샷은 409와 새 조회 안내를 반환한다. 현재 MVP 실행 조회와 V7 대응 건 목록은 이 스냅샷·필터 계약을 제공하지 않는다.

오류의 필수 필드는 공통 규약과 같은 `code`, `message`다. `requestId`, `retryable`, `details`는 선택 확장 필드이며 기존 클라이언트는 없어도 처리한다. `requestId`는 원문을 포함하지 않는 추적값, `retryable`은 재시도 가능 여부이며 `details`에는 키 원문, 원천 콘텐츠나 임의 내부 식별자를 넣지 않는다. 입력 오류는 400 `INVALID_INPUT`, 세션 없음은 401 `UNAUTHENTICATED`, 권한 없음은 403 `FORBIDDEN`, 대상 없음은 404 `NOT_FOUND`, 메서드 오류는 405 `METHOD_NOT_ALLOWED`, 버전 충돌과 스냅샷 만료는 409 `CONFLICT`, 대시보드 요청 제한은 429 `RATE_LIMITED`, 서버 장애는 5xx로 구분한다. 409의 상세 사유는 안전한 확장 정보로 구분한다. 공급자 검사 결과의 401, 403, 404 등은 대시보드 HTTP 오류와 별도로 ValidationAttempt의 사유로 기록한다.

`source`는 발견 위치의 수집 플랫폼, `provider`는 키 발급자 필터다. 예를 들어 Notion에서 발견한 AWS 키는 `source=NOTION`, `provider=AWS`다. 환경은 `ALL`, `PROD`, `DEV`, `UNKNOWN`이며 `ALL`은 UNKNOWN을 포함한다. 기존 Overview는 UNKNOWN을 지원하지 않으므로 화면 전환 전 그 경로에 UNKNOWN을 전달하지 않는다.

## 2. 조회

| 단계 | Method | 경로 | 응답의 핵심 정보 |
|---|---|---|---|
| MVP | GET | `/api/v1/secrets` | 고유 인벤토리 목록. `page`, `size`, `issuer`, `componentState` 필터와 `items`, `total`, `page`, `size` 응답. `CONFIRMED`·`CANDIDATE`·`METADATA_ONLY`를 구분하고 마스킹 식별자, 미평가 심각도, 최신 검증 결과·사유, 위치 수 표시 |
| MVP | GET | `/api/v1/secrets/{id}` | 마스킹 식별자, 유형·구성 상태, 최신 검증 결과, 위치별 원천·환경·존재 상태, 최초·최근 발견 시각. IAM 메타데이터 항목은 최신 관측의 `iamMetadata`에 키 상태, 발급·최근 사용·관측 시각과 최근 사용 서비스·리전을 별도 제공 |
| MVP | GET | `/api/v1/secrets/connections` | 수동 실행용 활성 GitHub 연결과 지원 형식의 AWS 메타데이터 또는 단일 자원 값 범위가 있는 활성 AWS 연결의 ID·표시 이름·상태·승인 범위 수. 연결 비밀 제외. 값 범위가 보여도 워커의 별도 값 읽기 설정이 활성이라는 뜻은 아님 |
| MVP | GET | `/api/v1/secrets/runs?page={page}&size={size}` | 0부터 시작하는 페이지, 크기 1–100(기본 20). `requestedAt` 최신순, 같은 시각이면 `runId` 내림차순. `{items,total,page,size}`를 반환하며 종료된 실행과 대기 중인 실행, 현재 일시중지된 연결의 과거 실행을 함께 조회 |
| MVP | GET | `/api/v1/secrets/runs/{runId}` | 실행 전체·단계별 상태, 범위와 객체 수, 시각·재시도·안전한 실패 코드. `scopeResults`에 범위별 내부 ID·지원 콘텐츠 유형·리전·수집 상태·항목 수·안전한 누락 코드만 포함 |
| V7 브랜치 | GET | `/api/v1/secrets/cases?page={page}&size={size}&origin=PRODUCTION` | 현재 대응 건의 마스킹 목록 `{items,total,page,size}`. `origin`은 `PRODUCTION` 또는 `TEST`, 기본은 `PRODUCTION`. 페이지는 0부터, 크기는 1–100. 집계 스냅샷·후속 필터는 아직 없음 |
| V7 브랜치 | GET | `/api/v1/secrets/cases/{caseId}` | 현재 대응 건의 마스킹 키, `caseOrigin`, 단계, 담당 팀·개인, 버전, 개설 위치 ID·사유, 담당자 변경 이력 |
| V7 브랜치 | GET | `/api/v1/secrets/assignees?caseId={caseId}` | 해당 건의 모든 연결 범위에 `REMEDIATOR` 권한이 있는 기존 계정과 활성 팀 소속. `SECURITY` 권한 필요 |
| 후속 | GET | `/api/v1/secrets/summary` | 심각도 분포와 미평가, 조치 네 단계와 별도 예외, 유효성 분포, 미지정, 수집 및 검사 최신성, 집계 기준 |
| 후속 확장 | GET | `/api/v1/secrets/cases` | 범위·스냅샷·필터를 갖춘 대응 목록. `scope=history`이면 과거 대응 건 목록. 발견 소스, 심각도, 유효성, 예외, 시각과 근거 추가 |
| 후속 확장 | GET | `/api/v1/secrets/cases/{caseId}` | 위치별 근거와 상태, 검사 및 평가 이력, 체크리스트, 외부 조치 및 감사 이력 추가 |
| 후속 | GET | `/api/v1/secrets/matrix` | 심각도와 유효성의 교차 건수, 미평가 행, 동일 필터와 기준시점 |

MVP 인벤토리 목록은 고유 `CredentialSet` 단위 페이지네이션이며 `UNASSESSED`와 환경 `UNKNOWN`을 임의 값으로 치환하지 않는다. 최신 검증 시도가 없으면 `UNCHECKED`로 표시한다. AWS IAM의 `METADATA_ONLY`는 키의 존재 정보이지 비밀 값의 발견·노출·유효성 증거가 아니다. `iamMetadata.keyStatus`의 `ACTIVE`도 STS 검증 결과 `VALID`와 다르며 최근 사용 기록 부재는 미사용의 증명이 아니다. 정확한 AWS 본문에서 키 후보를 탐지해도 그 자원이 외부에 노출됐다는 뜻은 아니며 노출·영향·심각도는 별도 근거로 평가한다. V7은 대응 건의 기초 기록만 제공하며 집계 스냅샷과 실제 단계별 수는 아직 제공하지 않으므로 아래 목표 요약 예시를 현행 목록 응답으로 사용하지 않는다.

실행 이력 목록의 각 `items[]`는 `runId`, `connectionId`, `provider`, 현재 연결 `displayName`, `triggerType`, `status`, `collectionStatus`, `detectionStatus`, `validationStatus`, `scopeCount`, `completedScopeCount`, `incompleteScopeCount`, `collectedObjectCount`, `requestedAt`, `startedAt`, `finishedAt`, `failureCode`만 포함한다. 전체·단계 상태와 시각은 상세 API와 같은 의미다. `failureCode`는 허용된 코드만 노출하고 알 수 없는 내부 문구는 `UNSPECIFIED_FAILURE`로 바꾼다. 계정 번호, `auth_ref`, `approved_ref`, 자원 이름·ARN, 원문 값, 공급자 오류 본문과 범위별 상세는 목록에 넣지 않는다. `collectedObjectCount`는 확인한 원천 항목 수이며 **새로 발견한 키 수가 아니다**. 스캔 이력은 `secret_scan_runs` 단위이고 `/cases?scope=history`의 과거 대응 건과 다르다. 현재 `displayName`은 실행 당시 이름의 스냅샷이 아니라 연결의 현행 표시 이름이다.

향후 상세의 발견 위치 링크를 만들 때에는 허용된 경로만 생성하고 대상 사용자의 원천 접근 권한을 다시 확인한다. 현재 MVP는 위치 메타데이터를 반환하며 원천으로 자동 리다이렉트하지 않는다. 마스킹된 문맥도 비신뢰 텍스트로 렌더링한다.

요약 응답의 핵심 형태는 다음과 같다. 수치는 계약 예시이며 실제 운영 결과가 아니다.

```json
{
  "scope": "currentCases",
  "unit": "ResponseCase",
  "asOf": "2026-10-09T00:00:00Z",
  "snapshotId": "example-snapshot-1",
  "filters": {"environment": "ALL", "caseOrigin": "PRODUCTION"},
  "total": 3,
  "actionCounts": {"OPEN": 1, "IN_PROGRESS": 1, "REVALIDATION_PENDING": 0, "RESOLVED": 0, "RISK_ACCEPTED": 1},
  "severityCounts": {"CRITICAL": 1, "HIGH": 0, "MEDIUM": 1, "LOW": 0, "UNASSESSED": 1},
  "partialCoverage": false,
  "missingSources": []
}
```

`actionCounts`의 네 조치 단계와 별도 `RISK_ACCEPTED` 예외 수, `severityCounts`의 다섯 값은 각각 `total`과 같다. `RISK_ACCEPTED`는 다섯 번째 진행 단계가 아니다. 수집 실패가 있으면 `partialCoverage=true`와 실패한 범위, 사유, 시각을 `missingSources`에 포함한다. 일부 수집 실패를 정상 0건으로 합산하지 않는다.

## 3. 변경

### 수동 전체 스캔

정식 수동 전체 스캔은 `POST /api/v1/secrets/runs`로 비동기 등록한다. 요청 본문은 `connectionId`와 선택적 `idempotencyKey`다. 세션·CSRF와 현재 `ADMIN` 권한 검사를 거쳐 승인 범위가 있는 **활성 GitHub 연결** 또는 IAM·Secrets Manager·SSM·S3의 **지원되는 메타데이터 범위나 정확한 단일 자원 값 범위가 있는 활성 AWS 연결**을 작업으로 예약한다. 그 밖의 원천과 지원되지 않는 AWS 범위는 409로 거부한다. 응답은 `202 Accepted`, `Location: /api/v1/secrets/runs/{runId}`, `{ "runId": 123, "status": "PENDING" }` 형태다. 같은 사용자와 멱등 키의 재요청은 기존 실행 ID를 반환한다. 화면은 `GET /api/v1/secrets/runs/{runId}`로 상태와 단계별 결과, 범위·객체 수, 재시도 시각, 안전한 실패 코드를 조회한다. AWS 메타데이터 실행의 객체 수는 각 범위에서 관측한 접근키 또는 목록 항목 수의 합계이며 탐지·검증 단계는 `SKIPPED`다. 값 범위는 `CURRENT_VALUE_ONLY`와 별도 `SECRET_AWS_CONTENT_READ_ENABLED` 설정이 필요하다. 기본 비활성이면 본문 SDK 호출을 하지 않고 범위에 안전한 미실행 코드를 남긴다. 읽기를 켠 범위만 Gitleaks 탐지·정규화 경로로 전달하며 검증은 별도 기본 비활성 설정에 따른다. 범위별 결과에는 계정 ID, 승인 참조, 자원 이름·ARN, 값 또는 공급자 오류 메시지를 반환하지 않는다. 실행기가 비활성이면 등록된 작업은 대기한다. API는 원천 콘텐츠를 읽거나 탐지 도구를 실행하지 않는다. 현재 권한 검사는 임시 `ADMIN` 경계이며 역할·연결 범위 권한은 후속이다.

현재 AWS 검증이 비활성이거나 합성 실험 저장소에서 차단된 경우에는 개별 키에 `UNCHECKED` 시도와 사유가 남더라도 실행의 검증 단계는 `SKIPPED`일 수 있다. 화면은 `SKIPPED`를 `VALID`나 검사 성공으로 바꾸지 않는다. 공급자 응답이 불확실하거나 기록에 실패한 경우는 `PARTIAL`로 구분한다.

### 연결 테스트

`POST /api/v1/secrets/connections/{connectionId}/test`는 본문 없이 현재 `ADMIN` 세션과 CSRF 토큰으로 호출한다. 동기 `200 OK`의 응답은 `{ "connectionId", "provider", "status", "reasonCode", "checkedAt" }`이며 `status`는 `CONNECTED`, `FAILED`, `UNAVAILABLE` 중 하나다. 성공의 `reasonCode`는 `null`이다. 실패·확인 불가 시에는 `INVALID_APPROVED_SCOPE`, `AWS_ACCOUNT_MISMATCH`, `AWS_ACCESS_DENIED`, `AWS_RATE_LIMITED`, `AWS_UNAVAILABLE`, `GITHUB_APP_UNAVAILABLE`, `GITHUB_ACCESS_DENIED`, `GITHUB_RATE_LIMITED`, `GITHUB_UNAVAILABLE` 중 하나만 반환한다. 알 수 없는 코드는 화면에 일반 오류로 표시한다. 연결이 없으면 404, 일시중지·오류·미지원 연결이면 409, 비관리자면 403이다.

AWS 연결은 활성 승인 범위가 단일 계정을 가리키는지 검사한 다음 제한 시간 안에 STS `GetCallerIdentity`로 프로필의 신원을 확인한다. 개별 IAM·Secrets Manager·SSM·S3 권한이나 비밀 값은 읽지 않는다. GitHub 연결은 첫 번째 활성 승인 저장소 한 곳으로 제한한 `contents:read` 설치 토큰 발급과 반환된 저장소·권한을 확인하며 파일은 읽지 않는다. `CONNECTED`를 모든 승인 범위의 수집 성공이나 키의 유효성으로 표현하지 않는다. 이 호출은 스캔을 등록하거나 연결 상태를 바꾸지 않으며 `auth_ref`, 승인 참조, 저장소명, 계정 번호, 자격 증명과 공급자 오류 본문을 응답·일반 로그에 넣지 않는다. 기존 `GET /connections`는 활성 연결의 스캔 선택 목록으로 유지하고 일시중지·오류 연결은 포함하지 않는다.

현재 `/api/v1/credentials`와 `/api/v1/credentials/scan`은 기존 호출자의 전환 기간에만 유지한다. 새 Secret 화면은 정식 API를 사용한다. 다른 사용처가 없음을 확인한 뒤 기존 경로를 제거한다. 전환 기간의 필드와 현재 구현의 차이는 [구현 연동 현황](../04-integration/current-implementation-and-gaps.md)에 기록하며 정식 모델이나 실행 경계로 채택하지 않는다.

### 대응 건 변경과 재검증

V7 브랜치의 `POST /api/v1/secrets/{secretId}/cases`는 `{ "occurrenceId", "reason": "ASSESSED_EXPOSURE" }`를 받아 201과 현재 건 상세를 반환한다. 서버는 저장된 `CONFIRMED` 키, `ABNORMAL`·`PRESENT` 위치, 활성 GitHub 연결·범위, 분류된 범위 대응 구분, 해당 연결의 `SECURITY` 권한을 확인한다. 클라이언트는 `caseOrigin`을 정할 수 없다. 기존 범위는 `UNCLASSIFIED`, 현재 수집 위치는 `UNKNOWN`이므로 별도의 승인된 노출 판정과 범위 분류 없이는 실데이터 건을 열 수 없다. IAM `METADATA_ONLY`, 불완전 후보, AWS 저장소의 값 존재나 검증 `UNKNOWN`·`UNCHECKED`만으로 개설하지 않는다. 이 개설은 노출 평가 절차 자체가 아니며 평가 근거·승인 이력은 후속 과제다.

V7의 `PATCH /api/v1/secrets/cases/{caseId}/assignment`는 `{ "team", "assigneeUserId", "reason", "expectedVersion" }`를 받는다. 사유는 `INITIAL_ASSIGNMENT`, `OWNERSHIP_CHANGE`, `TEAM_TRANSFER`, `UNASSIGNED`, `OTHER` 중 하나다. 권한 있는 기존 조치 담당자와 팀 소속만 지정하며, 비밀 원문이 들어갈 수 있는 자유 형식 사유를 저장하지 않는다. 같은 트랜잭션에서 담당자, 버전, 변경 전후 이력과 안전한 감사 이벤트를 저장한다. 잘못된 버전은 409이며 검토·조치 단계는 담당자 지정만으로 바뀌지 않는다. 아래 다른 변경 경로는 후속 목표다.

| Method | 경로 | 요청과 불변 조건 |
|---|---|---|
| PATCH (V7) | `/api/v1/secrets/cases/{caseId}/assignment` | 팀과 개인, 허용된 사유 코드, `expectedVersion`. 수행자, 변경 전후, 시각 감사 |
| POST | `/api/v1/secrets/cases/{caseId}/review` | 검토 결과와 사유, `expectedVersion`. 검토만으로 RESOLVED 전이 금지 |
| POST | `/api/v1/secrets/cases/{caseId}/actions` | 외부 수행 종류, 대상 위치, 실제 수행 시각, 비밀 제거 증거 참조, `expectedVersion`. 공급자 키 변경 호출 없음 |
| PATCH | `/api/v1/secrets/cases/{caseId}/checklist/{itemId}` | 완료 여부, 가이드 버전, 근거, `expectedVersion`. 체크만으로 해결 금지 |
| POST | `/api/v1/secrets/cases/{caseId}/revalidations` | 요청 사유, `idempotencyKey`, `expectedVersion`. 작업 ID를 반환하고 원문은 받지 않음 |
| POST | `/api/v1/secrets/cases/{caseId}/resolve` | 위치별 해결 근거, 조치 이후 재확인 근거, `expectedVersion`. 서버가 전이 조건 검증 |
| POST | `/api/v1/secrets/cases/{caseId}/exceptions` | 승인 범위, 사유, 만료 시각, 승인자 권한, `expectedVersion`. 전체 예외와 일부 위치 예외 구분 |
| POST | `/api/v1/secrets/cases/{caseId}/exceptions/revoke` | 취소 사유, `expectedVersion`. 남은 위험에 따라 단계 재개 |

모든 변경은 서버가 인증된 사용자 ID를 감사 기록에 넣는다. 클라이언트가 보낸 수행자 ID를 신뢰하지 않는다. `expectedVersion`이 다르면 409를 반환하고 저장된 현재 상태를 다시 조회하게 한다. 재검증 요청은 `idempotencyKey`로 중복 작업을 막고 이전 요청의 작업 ID를 반환한다. 변경 결과는 저장 완료 후의 버전과 상태를 반환한다.

## 4. 권한과 전환

보안 담당자는 검토, 심각도 근거 확인, 예외 승인을 수행한다. 인프라 및 개발 담당자는 할당된 대응 건의 외부 조치 기록, 체크리스트와 재검증 요청을 수행한다. 운영 관리자는 연결 범위와 실행 상태를 관리한다. 담당자 지정과 해결은 보안 담당자 또는 명시적으로 위임된 역할에 한정한다. 서버는 대상 범위까지 검사하며 화면 숨김만으로 권한을 구현하지 않는다.

계정 인증은 기존 `dashboard_users`와 세션을 사용한다. `ADMIN`이라는 값만으로 Secret의 `SECURITY`, `REMEDIATOR`, `OPERATIONS` 권한을 자동 부여하지 않고 V7의 `secret_role_grants`와 연결 범위를 확인한다. V7은 개발 환경에서 명시적으로 지정한 기존 계정에 전역 Secret 역할을 한 번 부여하는 선택적 초기화 경로를 제공한다. 빈 설정은 권한을 주지 않고, 취소된 부여를 재시작으로 복구하지 않는다. 운영 환경의 승인된 역할 부여·취소, 연결별 범위 관리, 새 담당자 계정 생성·비활성화와 그 감사 절차는 후속이다. 현재 자동 등록 관리자 1개와 이름 문자열만으로 여러 개인 담당자가 준비됐다고 표시하지 않는다. 공개 회원가입이나 새로운 로그인 방식은 이 문서에서 도입하지 않는다.

기존 `/overview` 응답의 숫자 점수는 새 Secret 요약의 근거가 아니다. 기존 클라이언트 호환을 위해 기존 필드를 즉시 재해석하지 않고 새 Secret 응답을 연결한 뒤 화면을 단계적으로 전환한다. 100점 표시 제거 시 해당 필드의 사용처와 계약 테스트를 확인하고 버전 변경 또는 사용 중단을 기록한다. 실제 전환 대상 코드와 순서는 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)에 둔다.

### 통합 현황의 전환 경계

- 첫 연동은 `/api/v1/secrets`를 독립적으로 구현하고 기존 Overview v1의 필드 의미를 유지한다. 새 Secret 요약 카드에는 해당 범위와 단위를 표시한다.
- 새 통합 조회는 일반 탐지의 `detection_events` 한 행과 Secret의 현재 `PRODUCTION` ResponseCase 한 행을 각각 위험 항목으로 합성한다. `TEST` 시뮬레이션 건은 제외한다. 항목의 영역, 원본 ID와 집계 단위를 명시하고 서로 다른 ID를 혼용하지 않는다. Secret 대응 건을 `detection_events`에 다시 적재하여 두 번 세지 않는다.
- 통합 심각도 분포는 포함된 위험 항목의 확인 등급과 미평가를 구분한다. Secret 부분은 현재 대응 건 필터·기준시점·스냅샷을 유지하며 고유 키·위치·Finding 수를 위험 항목 수에 합산하지 않는다. 기존 위험 추이의 날짜별 발생 이벤트는 현재 대응 건 수와 별도 지표다.
- 새 통합 응답은 별도 버전으로 제공한다. 정확한 URL, 필드, 일반 탐지의 포함·제외 조건과 두 출처의 기준시점 고정 방식은 통합 API 구현 전에 정의한다. 기존 API 필드를 새 의미로 바꾸거나 구현 완료로 표시하지 않는다.
- 새 화면과 통합 계약을 검증한 뒤 100점 카드를 등급 분포로 교체하고 구 클라이언트 지원 종료를 기록한다. 확장 전에는 기존 샘플 화면 표시를 유지한다.
