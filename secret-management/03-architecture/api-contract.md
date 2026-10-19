# Secret API 계약

기준: [PRD](../01-requirements/PRD.md), [ADR-S008](adr/ADR-S008.md), [ADR-S009](adr/ADR-S009.md), [데이터 계약](data-contract.md), [대응 정책](../02-functional-design/response-policy.md)

이 문서는 Secret 기능의 목표 계약과 개발 구현을 구분한다. 인벤토리·연결·실행 조회, V7 대응 건·담당자, V8 수동 범위 분류·노출 판정에 이어 V9는 검토·체크리스트·외부 조치 기록을 제공한다. V10은 승인된 GitHub 위치의 비동기 재검증과 근거가 충족된 해결을 연결한다. AI 가이드는 별도 선택 기능이며 기본 안내는 외부 API 없이 동작한다. 심각도·영향 평가, 실제 단계별 요약·스냅샷, 예외와 다른 원천의 재검증은 목표 계약이다. 브랜치별 검증·병합 상태는 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)를 따른다.

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
| V7 | GET | `/api/v1/secrets/cases?page={page}&size={size}&origin=PRODUCTION` | 현재 대응 건의 마스킹 목록 `{items,total,page,size}`. `origin`은 `PRODUCTION` 또는 `TEST`, 기본은 `PRODUCTION`. 페이지는 0부터, 크기는 1–100. 집계 스냅샷·후속 필터는 아직 없음 |
| V7–V10 | GET | `/api/v1/secrets/cases/{caseId}` | 현재 대응 건의 마스킹 키, `caseOrigin`, 단계·검토 상태, 담당자, 버전, 개설 위치와 연결된 위치의 ID·조치 상태, 담당자·검토·체크리스트·외부 조치·단계 변경 이력 및 호출자별 `capabilities`. V10은 `occurrences[]`에 `canRequestRevalidation`·`canResume`, 건 `capabilities`에 `canRequestRevalidation`·`canResolve`·`canResume`을 추가. 키 원문·원천 본문 제외 |
| V7 | GET | `/api/v1/secrets/assignees?caseId={caseId}` | 해당 건의 모든 연결 범위에 `REMEDIATOR` 권한이 있는 기존 계정과 활성 팀 소속. `SECURITY` 권한 필요 |
| V8 | GET | `/api/v1/secrets/triage-candidates?page={page}&size={size}` | 연결별 `SECURITY` 권한이 있는 활성 GitHub 범위의 후보 키만 `{items,total,page,size}`로 조회. 항목은 키 ID, 마스킹 이름, 발급자, 유형, 구성 상태, 허용된 위치 수만 반환 |
| V8 | GET | `/api/v1/secrets/triage-candidates/{secretId}` | 같은 권한 경계의 후보 키와 선택 가능한 발견 위치를 조회. 위치 ID, 연결·범위 ID, 원천 위치, 환경, 노출·존재 상태와 판정·범위 버전 제공 |
| V8 | GET | `/api/v1/secrets/scopes/{scopeId}/response-origin` | 승인 범위의 `responseOrigin`, `responseOriginVersion`, 서버가 허용한 `allowedOrigins`, 분류 수행자·시각과 추가 전용 분류 이력. 비밀이 될 수 있는 원천 참조는 제외 |
| V8 | GET | `/api/v1/secrets/occurrences/{occurrenceId}/triage` | 위치의 `secretId`, `scopeId`, 구성·존재·노출 상태, `assessmentVersion`, 범위 구분·버전, 같은 키·구분의 현재 건 `openCaseId`(없으면 null), 해당 위치에 속하는 Finding ID·룰·탐지 시각과 수동 판정 이력. Finding 문맥과 원문은 제외 |
| V10 | GET | `/api/v1/secrets/cases/{caseId}/revalidations` | 해당 건의 위치별 재검증 요청·작업·결과 이력. 항목은 `revalidationId`, `occurrenceId`, `runId`, `jobStatus`, `result`, `failureCode`, `requestedAt`, `finishedAt`의 안전한 정보만 제공 |
| 후속 | GET | `/api/v1/secrets/summary` | 심각도 분포와 미평가, 조치 네 단계와 별도 예외, 유효성 분포, 미지정, 수집 및 검사 최신성, 집계 기준 |
| 후속 확장 | GET | `/api/v1/secrets/cases` | 범위·스냅샷·필터를 갖춘 대응 목록. `scope=history`이면 과거 대응 건 목록. 발견 소스, 심각도, 유효성, 예외, 시각과 근거 추가 |
| 후속 확장 | GET | `/api/v1/secrets/cases/{caseId}` | 아직 없는 위치별 심각도·영향 근거, 예외, 실행 스냅샷과 과거 건 상세를 추가 |
| 후속 | GET | `/api/v1/secrets/matrix` | 심각도와 유효성의 교차 건수, 미평가 행, 동일 필터와 기준시점 |

MVP 인벤토리 목록은 고유 `CredentialSet` 단위 페이지네이션이며 `UNASSESSED`와 환경 `UNKNOWN`을 임의 값으로 치환하지 않는다. 최신 검증 시도가 없으면 `UNCHECKED`로 표시한다. AWS IAM의 `METADATA_ONLY`는 키의 존재 정보이지 비밀 값의 발견·노출·유효성 증거가 아니다. `iamMetadata.keyStatus`의 `ACTIVE`도 STS 검증 결과 `VALID`와 다르며 최근 사용 기록 부재는 미사용의 증명이 아니다. 정확한 AWS 본문에서 키 후보를 탐지해도 그 자원이 외부에 노출됐다는 뜻은 아니며 노출·영향·심각도는 별도 근거로 평가한다. V7–V9는 대응 건의 개설·담당·검토·외부 조치 이력을 제공하지만 집계 스냅샷과 실제 단계별 수는 아직 제공하지 않으므로 아래 목표 요약 예시를 현행 목록 응답으로 사용하지 않는다.

수동 판정 화면은 기존 `ADMIN` 전용 인벤토리 API를 권한 우회용으로 확장하지 않는다. V8의 후보 목록·상세 API가 활성 GitHub 연결·승인 범위와 현재 사용자의 연결별 `SECURITY` 권한으로 위치를 SQL에서 제한한다. 한 키에 여러 위치가 있어도 권한 밖 위치는 상세 응답에 나타나지 않고, 응답의 위치 수는 허용된 위치의 수다. 원문 키·보호키 식별값은 반환하지 않는다.

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

V8의 수동 판정은 해당 연결의 `SECURITY` 권한을 요구하며 현재 활성 GitHub 범위에 한정한다. `PATCH /api/v1/secrets/scopes/{scopeId}/response-origin`은 `{ "origin": "TEST|PRODUCTION", "reasonCode", "expectedVersion" }`를 받는다. 허용 조합은 `TEST`와 `APPROVED_TEST_FIXTURE`, `PRODUCTION`과 `APPROVED_PRODUCTION_SOURCE`다. 처음에는 `UNCLASSIFIED`이고 한 번만 분류할 수 있으며 공급자·환경·저장소 이름에서 자동으로 운영 분류를 만들지 않는다. 알려진 `github://Logssey/test-find-key/...` 합성 표본 범위는 서버가 `PRODUCTION` 선택을 거부하고 `allowedOrigins`에 `TEST`만 제공한다. 그 밖의 범위도 보안 담당자의 명시적 선택이 필요하다. 버전 충돌은 409다. 분류 전후, 사유 코드, 수행자, 시각과 버전은 추가 전용 이력에 보존한다.

`POST /api/v1/secrets/occurrences/{occurrenceId}/assessments`는 `{ "decision": "ABNORMAL|NORMAL", "reasonCode", "findingId", "expectedVersion" }`를 받는다. `ABNORMAL` 사유는 `UNAPPROVED_SOURCE_LOCATION`, 시험 범위에서만 `SIMULATED_EXPOSURE`다. `NORMAL` 사유는 `AUTHORIZED_LOCATION`이다. `findingId`는 같은 위치의 형식 탐지 기록이어야 하며, 노출 판단의 자동 증명이 아니라 담당자가 검토한 자료를 가리킨다. 활성 GitHub 연결·분류된 범위, `CONFIRMED` 키, `PRESENT` 위치에서만 판정한다. 수동 판정의 변경 전후 상태·사유·Finding ID·수행자·시각·버전을 이력으로 저장하고 최신 판정을 위치에 반영한다. 근거가 부족하면 `UNKNOWN`을 유지한다. 원문 키나 자유 형식 메모는 요청·응답·감사에 담지 않는다.

V8 이후 `POST /api/v1/secrets/{secretId}/cases`는 `{ "occurrenceId", "reason": "ASSESSED_EXPOSURE", "expectedAssessmentVersion", "expectedScopeOriginVersion" }`를 받아 201과 현재 건 상세를 반환한다. 서버는 저장된 최신 수동 판정이 `ABNORMAL`인지, 두 버전이 일치하는지, `CONFIRMED` 키와 `PRESENT` 위치, 활성 GitHub 연결·범위, 해당 연결의 `SECURITY` 권한을 다시 확인한다. 클라이언트는 `caseOrigin`을 정할 수 없고 서버가 분류된 범위에서 결정한다. `UNCLASSIFIED`, `UNKNOWN`, IAM `METADATA_ONLY`, 불완전 후보, AWS 저장소의 값 존재나 검증 `UNKNOWN`·`UNCHECKED`만으로 개설하지 않는다. 판정 저장만으로 대응 건을 자동 생성하지 않으며, 재스캔은 수동 판정을 덮어쓰지 않는다.

같은 키·구분에 현재 대응 건이 이미 있으면 새 건 개설은 409다. V8은 그 건의 `openCaseId`를 보여주지만 다른 비정상 노출 위치를 기존 건에 추가 연결하는 API는 아직 제공하지 않는다. 여러 위치를 한 건으로 묶는 목표 계약과 현재 구현의 차이는 [구현 연동 현황](../04-integration/current-implementation-and-gaps.md)에 남긴다.

V7의 `PATCH /api/v1/secrets/cases/{caseId}/assignment`는 `{ "team", "assigneeUserId", "reason", "expectedVersion" }`를 받는다. 사유는 `INITIAL_ASSIGNMENT`, `OWNERSHIP_CHANGE`, `TEAM_TRANSFER`, `UNASSIGNED`, `OTHER` 중 하나다. 권한 있는 기존 조치 담당자와 팀 소속만 지정하며, 비밀 원문이 들어갈 수 있는 자유 형식 사유를 저장하지 않는다. 같은 트랜잭션에서 담당자, 버전, 변경 전후 이력과 안전한 감사 이벤트를 저장한다. 잘못된 버전은 409이며 검토·조치 단계는 담당자 지정만으로 바뀌지 않는다.

| Method | 경로 | 요청과 불변 조건 |
|---|---|---|
| PATCH (V8) | `/api/v1/secrets/scopes/{scopeId}/response-origin` | 허용된 시험·운영 분류와 사유 코드, `expectedVersion`. `UNCLASSIFIED`에서 한 번만 전이, 수행자·시각·변경 전후 감사 |
| POST (V8) | `/api/v1/secrets/occurrences/{occurrenceId}/assessments` | 해당 위치의 Finding ID, 노출 판정·사유 코드, `expectedVersion`. `UNKNOWN`은 근거 부족 시 유지, 판정 이력 보존 |
| POST (V8 적용) | `/api/v1/secrets/{secretId}/cases` | `occurrenceId`, `ASSESSED_EXPOSURE`, 두 최신 판정·범위 버전. 서버가 분류와 비정상 노출을 재확인하고 운영·시험 구분을 파생 |
| PATCH (V7) | `/api/v1/secrets/cases/{caseId}/assignment` | 팀과 개인, 허용된 사유 코드, `expectedVersion`. 수행자, 변경 전후, 시각 감사 |
| POST (V9) | `/api/v1/secrets/cases/{caseId}/review` | `{decision,reasonCode,expectedVersion}`. `CONFIRMED/EVIDENCE_CONFIRMED`이면 `REVIEWED`, `NEEDS_MORE_INFO/INSUFFICIENT_EVIDENCE`이면 이력만 추가하고 `UNREVIEWED` 유지. `SECURITY` 권한 필요, 조치 단계·해결 상태는 자동 변경하지 않음 |
| POST (V9) | `/api/v1/secrets/cases/{caseId}/start` | `{reasonCode:"RESPONSE_STARTED",expectedVersion}`. 해당 건에 지정된 개인이면서 모든 연결에 `REMEDIATOR` 권한이 있어야 하며, 검토 확정 뒤 `OPEN`에서 `IN_PROGRESS`로 명시적 전이 |
| PATCH (V9) | `/api/v1/secrets/cases/{caseId}/checklist/{itemId}` | `{completed,guideVersion:"S2_V1",evidenceRef,expectedVersion}`. 지정된 `REMEDIATOR`만 변경, `DONE`에는 제한된 티켓/변경 참조 필요. 체크만으로 해결 금지 |
| POST (V9) | `/api/v1/secrets/cases/{caseId}/actions` | `{actionType,occurrenceId,performedAt,evidenceRef,expectedVersion}`. 검토 확정 뒤 지정된 `REMEDIATOR`가 연결 위치에 외부에서 수행한 조치를 기록. 공급자 키 변경 호출 없음. 조치한 위치는 `REVALIDATION_PENDING`; 모든 연결 위치가 그 상태일 때 건도 대기로 전이 |
| POST (V10) | `/api/v1/secrets/cases/{caseId}/revalidations` | `{occurrenceId,expectedVersion,idempotencyKey}`. 조치 기록이 있는 승인된 단일 GitHub 위치의 비동기 작업을 등록하고 재검증 식별자·작업 상태를 반환. 원문은 받지 않음 |
| POST (V10) | `/api/v1/secrets/cases/{caseId}/revalidation-resume` | `{occurrenceId,expectedVersion,reasonCode:"REVALIDATION_PRESENT"}`. 조치 후 재검증에서도 키가 계속 발견된 위치를 다시 `IN_PROGRESS`로 전환해 추가 외부 조치를 받음 |
| POST (V10) | `/api/v1/secrets/cases/{caseId}/resolve` | `{expectedVersion,reasonCode:"POST_ACTION_REVALIDATION"}`. `SECURITY` 권한과 모든 관련 위치의 조치 이후 최신 `MISSING` 근거·범위 버전 등 서버 조건을 검사한 후 `RESOLVED`로 전이 |
| POST (선택) | `/api/v1/secrets/cases/{caseId}/guide` | 본문 없음. `SECURITY` 또는 지정된 `REMEDIATOR`에게 버전 고정 기본 가이드를 반환하고, 별도 활성화·키·합성 `TEST` 범위 조건이 모두 맞을 때만 Groq 호출. 호출 실패는 기본 가이드로 처리; 상태·DB 변경 없음 |
| POST (후속) | `/api/v1/secrets/cases/{caseId}/exceptions` | 승인 범위, 사유, 만료 시각, 승인자 권한, `expectedVersion`. 전체 예외와 일부 위치 예외 구분 |
| POST (후속) | `/api/v1/secrets/cases/{caseId}/exceptions/revoke` | 취소 사유, `expectedVersion`. 남은 위험에 따라 단계 재개 |

V9의 체크리스트는 `IDENTIFY_OWNER`, `ROTATE_OR_REVOKE`, `REMOVE_EXPOSURE`, `CHECK_ACCESS_LOGS` 네 코드이며 `TODO`·`DONE`을 저장한다. 외부 조치 종류는 `ROTATED`, `REVOKED`, `REMOVED_FROM_SOURCE`, `PERMISSION_REDUCED`다. 증거 참조는 `TICKET:SEC-123`처럼 `TICKET|CHANGE:대문자 프로젝트 코드-숫자`만 허용한다. 임의 URL·자유 문장·키 원문은 저장하지 않는다. 외부 수행 시각은 최근 30일에서 요청 시각보다 최대 5분 뒤까지만 허용한다. `NEEDS_MORE_INFO`는 검토 확정이 아니며 조치 시작·기록 권한을 열지 않는다. 한 건에 여러 위치가 연결되면 각 위치의 조치를 따로 기록하고 미조치 위치가 남은 건은 `IN_PROGRESS`로 둔다. 현재 개설 API는 위치 하나만 연결하며 추가 위치 연결은 후속이다.

V10의 재검증 결과는 `PENDING`, `PRESENT`, `MISSING`, `UNKNOWN`으로 구분한다. `MISSING`은 해당 위치의 **단일 승인 GitHub 범위**를 조치 이후 정상 수집하고 Gitleaks 실행이 성공했으며, 탐지 설정·버전과 HMAC 키 버전이 비교 가능한 상태에서 해당 키를 찾지 못했을 때만 인정한다. 일부 실패나 비교 불가를 `MISSING`으로 바꾸지 않고 `UNKNOWN`과 안전한 실패 코드로 기록한다. `PRESENT`를 추가 조치 없이 해결 근거로 사용하지 않는다. 다른 수집원·과거 Git 이력의 잔여 노출까지 확인한 것으로 표시하지 않는다.

V10의 `POST /cases/{caseId}/revalidations`는 `202 Accepted`와 `{revalidationId,occurrenceId,runId,jobStatus,result,failureCode,requestedAt,finishedAt}`를 반환한다. `GET`은 같은 형태의 이력 배열이다. 작업 상태는 `PENDING`, `LEASED`, `SUCCEEDED`, `FAILED`, `CANCELLED`이고 결과는 별도 필드다. 실행 중 결과는 `PENDING`, 실패·불완전 비교는 `UNKNOWN` 및 `REVALIDATION_INCOMPLETE` 또는 안전한 워커 실패 코드를 사용한다. 해결·재조치 변경은 최신 버전의 대응 건 상세를 반환한다. 최신 외부 조치 뒤의 근거, 동일한 현재 범위 버전 및 같은 키·대응 구분에 남은 다른 비정상·잔여·확인 불가 위치를 서버가 확인한다.

모든 변경은 서버가 인증된 사용자 ID를 감사 기록에 넣는다. 클라이언트가 보낸 수행자 ID를 신뢰하지 않는다. `expectedVersion`이 다르면 409를 반환하고 저장된 현재 상태를 다시 조회하게 한다. 재검증 요청은 `idempotencyKey`로 중복 작업을 막고 기존 요청을 다시 연결한다. 변경 결과는 저장 완료 후의 버전과 상태를 반환한다. 가이드 응답은 `{version:"secret-response-guide-v1",source:"DEFAULT|GROQ",title,steps:[{code,title,description}],notice}`이며 키 원문, 마스킹 끝자리, 지문, 저장소·계정·경로와 임의 메모를 외부 AI에 전송하지 않는다.

## 4. 권한과 전환

보안 담당자는 검토, 심각도 근거 확인, 예외 승인을 수행한다. 인프라 및 개발 담당자는 할당된 대응 건의 외부 조치 기록, 체크리스트와 재검증 요청을 수행한다. 운영 관리자는 연결 범위와 실행 상태를 관리한다. 담당자 지정과 해결은 보안 담당자 또는 명시적으로 위임된 역할에 한정한다. 서버는 대상 범위까지 검사하며 화면 숨김만으로 권한을 구현하지 않는다.

계정 인증은 기존 `dashboard_users`와 세션을 사용한다. `ADMIN`이라는 값만으로 Secret의 `SECURITY`, `REMEDIATOR`, `OPERATIONS` 권한을 자동 부여하지 않고 V7의 `secret_role_grants`와 연결 범위를 확인한다. V7은 개발 환경에서 명시적으로 지정한 기존 계정에 전역 Secret 역할을 한 번 부여하는 선택적 초기화 경로를 제공한다. 빈 설정은 권한을 주지 않고, 취소된 부여를 재시작으로 복구하지 않는다. V8의 수동 판정 후보는 연결별 `SECURITY` 범위로 제한하며 기존 `ADMIN` 인벤토리를 판정 접근 경계로 사용하지 않는다. 운영 환경의 승인된 역할 부여·취소, 범위별 역할 관리, 잘못 분류한 범위의 교정, 새 담당자 계정 생성·비활성화와 그 감사 절차는 후속이다. 현재 자동 등록 관리자 1개와 이름 문자열만으로 여러 개인 담당자가 준비됐다고 표시하지 않는다. 공개 회원가입이나 새로운 로그인 방식은 이 문서에서 도입하지 않는다.

기존 `/overview` 응답의 숫자 점수는 새 Secret 요약의 근거가 아니다. 기존 클라이언트 호환을 위해 기존 필드를 즉시 재해석하지 않고 새 Secret 응답을 연결한 뒤 화면을 단계적으로 전환한다. 100점 표시 제거 시 해당 필드의 사용처와 계약 테스트를 확인하고 버전 변경 또는 사용 중단을 기록한다. 실제 전환 대상 코드와 순서는 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)에 둔다.

### 통합 현황의 전환 경계

- 첫 연동은 `/api/v1/secrets`를 독립적으로 구현하고 기존 Overview v1의 필드 의미를 유지한다. 새 Secret 요약 카드에는 해당 범위와 단위를 표시한다.
- 새 통합 조회는 일반 탐지의 `detection_events` 한 행과 Secret의 현재 `PRODUCTION` ResponseCase 한 행을 각각 위험 항목으로 합성한다. `TEST` 시뮬레이션 건은 제외한다. 항목의 영역, 원본 ID와 집계 단위를 명시하고 서로 다른 ID를 혼용하지 않는다. Secret 대응 건을 `detection_events`에 다시 적재하여 두 번 세지 않는다.
- 통합 심각도 분포는 포함된 위험 항목의 확인 등급과 미평가를 구분한다. Secret 부분은 현재 대응 건 필터·기준시점·스냅샷을 유지하며 고유 키·위치·Finding 수를 위험 항목 수에 합산하지 않는다. 기존 위험 추이의 날짜별 발생 이벤트는 현재 대응 건 수와 별도 지표다.
- 새 통합 응답은 별도 버전으로 제공한다. 정확한 URL, 필드, 일반 탐지의 포함·제외 조건과 두 출처의 기준시점 고정 방식은 통합 API 구현 전에 정의한다. 기존 API 필드를 새 의미로 바꾸거나 구현 완료로 표시하지 않는다.
- 새 화면과 통합 계약을 검증한 뒤 100점 카드를 등급 분포로 교체하고 구 클라이언트 지원 종료를 기록한다. 확장 전에는 기존 샘플 화면 표시를 유지한다.
