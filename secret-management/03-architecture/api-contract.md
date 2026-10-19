# Secret API 계약

기준: [PRD](../01-requirements/PRD.md), [ADR-S008](adr/ADR-S008.md), [ADR-S009](adr/ADR-S009.md), [ADR-S010](adr/ADR-S010.md), [데이터 계약](data-contract.md), [대응 정책](../02-functional-design/response-policy.md)

이 문서는 Secret 기능의 목표 계약과 개발 구현을 구분한다. 인벤토리·연결·실행 조회, V7 대응 건·담당자, V8 수동 범위 분류·노출 판정에 이어 V9는 검토·체크리스트·외부 조치 기록을 제공한다. V10은 승인된 GitHub 위치의 비동기 재검증과 근거가 충족된 해결을 연결한다. 이번 대응 UX 변경은 조치 시작을 필수로 하고, 한 위치의 외부 조치를 여러 건 기록한 뒤 명시적으로 재검증 대기로 넘기는 계약을 추가한다. AI 가이드는 별도 선택 기능이며 기본 안내는 외부 API 없이 동작한다. 심각도·영향 평가, 실제 단계별 요약·스냅샷, 예외와 다른 원천의 재검증은 목표 계약이다. 브랜치별 검증·병합 상태는 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)를 따른다.

**목표 계약 우선순위:** 아래 1–4절의 V7–V12 경로는 현재 구현과 전환 호환을 기록한다. 중복 판정 게이트, 한 위치만 연결되는 개설, 범위 구분의 불변성과 스캔 집계 필드의 화면 의미는 [5절](#5-근거-중심-검토와-스캔-이력의-목표-계약)의 ADR-S010 목표로 대체한다. 새 기능을 구현했다고 해석하지 않는다.

## 1. 공통 계약

모든 조회와 변경은 기존 세션 인증 경계를 통과한다. 변경 요청은 POST·PATCH·PUT·DELETE 모두 CSRF 검증과 서버 권한 검사를 적용한다. 공급자 연결 자격 증명과 발견된 키 원문은 요청, 응답, 오류, 로그에 넣지 않는다. 응답 필드는 camelCase, 상태·분류 enum은 대문자, 시각은 UTC ISO 8601 형식을 사용한다. `scope`의 `currentCases`·`inventory`·`history`, `unit`의 `ResponseCase`·`CredentialSet`은 계약 식별 문자열이며 아래 표기를 유지한다. 공통 기준은 [대시보드 API 명세](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)를 따른다.

후속 집계 API는 `source`, `provider`, `environment`, `credentialType`, `validity`, `severity`, `actionStatus`, `exception`, `assignee`, 기간 필터 중 해당 경로에 맞는 값을 받는다. 응답은 적용된 `filters`, `scope`, `unit`, `asOf`, `snapshotId`, `total`, `partialCoverage`를 반환한다. 요약, 매트릭스와 기본 대응 목록은 `scope=currentCases`, `unit=ResponseCase`, 기본 대응 구분 `PRODUCTION`이고 그 구분에서 각 키의 최신 건만 센다. `TEST`는 명시적 시험 조회에만 포함하며 운영·통합 수치와 합치지 않는다. 인벤토리는 `scope=inventory`, `unit=CredentialSet`이다. 이력 목록은 `scope=history`, `unit=ResponseCase`로 과거 건을 각각 센다. 화면은 첫 조회의 `snapshotId`를 매트릭스와 목록에 전달한다. 같은 구분·스냅샷·필터의 카드, 매트릭스, 목록 건수는 일치해야 한다. 만료된 스냅샷은 409와 새 조회 안내를 반환한다. 현재 MVP 실행 조회와 V7 대응 건 목록은 이 스냅샷·필터 계약을 제공하지 않는다.

오류의 필수 필드는 공통 규약과 같은 `code`, `message`다. `requestId`, `retryable`, `details`는 선택 확장 필드이며 기존 클라이언트는 없어도 처리한다. `requestId`는 원문을 포함하지 않는 추적값, `retryable`은 재시도 가능 여부이며 `details`에는 키 원문, 원천 콘텐츠나 임의 내부 식별자를 넣지 않는다. 입력 오류는 400 `INVALID_INPUT`, 세션 없음은 401 `UNAUTHENTICATED`, 권한 없음은 403 `FORBIDDEN`, 대상 없음은 404 `NOT_FOUND`, 메서드 오류는 405 `METHOD_NOT_ALLOWED`, 버전 충돌과 스냅샷 만료는 409 `CONFLICT`, 목표 판정·해결에서 제출한 근거 부족은 422 `EVIDENCE_REQUIRED`, 대시보드 요청 제한은 429 `RATE_LIMITED`, 서버 장애는 5xx로 구분한다. 409의 상세 사유는 안전한 확장 정보로 구분한다. 공급자 검사 결과의 401, 403, 404 등은 대시보드 HTTP 오류와 별도로 ValidationAttempt의 사유로 기록한다.

`source`는 발견 위치의 수집 플랫폼, `provider`는 키 발급자 필터다. 예를 들어 Notion에서 발견한 AWS 키는 `source=NOTION`, `provider=AWS`다. 환경은 `ALL`, `PROD`, `DEV`, `UNKNOWN`이며 `ALL`은 UNKNOWN을 포함한다. 기존 Overview는 UNKNOWN을 지원하지 않으므로 화면 전환 전 그 경로에 UNKNOWN을 전달하지 않는다.

## 2. 조회

| 단계 | Method | 경로 | 응답의 핵심 정보 |
|---|---|---|---|
| MVP | GET | `/api/v1/secrets` | 고유 인벤토리 목록. `page`, `size`, `issuer`, `componentState` 필터와 `items`, `total`, `page`, `size` 응답. `CONFIRMED`·`CANDIDATE`·`METADATA_ONLY`를 구분하고 마스킹 식별자, 미평가 심각도, 최신 검증 결과·사유, 위치 수 표시 |
| MVP | GET | `/api/v1/secrets/{id}` | 마스킹 식별자, 유형·구성 상태, 최신 검증 결과, 위치별 원천·환경·존재 상태, 최초·최근 발견 시각. IAM 메타데이터 항목은 최신 관측의 `iamMetadata`에 키 상태, 발급·최근 사용·관측 시각과 최근 사용 서비스·리전을 별도 제공 |
| MVP | GET | `/api/v1/secrets/connections` | 수동 실행용 활성 GitHub 연결과 지원 형식의 AWS 메타데이터 또는 단일 자원 값 범위가 있는 활성 AWS 연결의 ID·표시 이름·상태·승인 범위 수. 연결 비밀 제외. 값 범위가 보여도 워커의 별도 값 읽기 설정이 활성이라는 뜻은 아님 |
| MVP | GET | `/api/v1/secrets/runs?page={page}&size={size}` | 0부터 시작하는 페이지, 크기 1–100(기본 20). `requestedAt` 최신순, 같은 시각이면 `runId` 내림차순. `{items,total,page,size}`를 반환하며 종료된 실행과 대기 중인 실행, 현재 일시중지된 연결의 과거 실행을 함께 조회 |
| MVP | GET | `/api/v1/secrets/runs/{runId}` | 실행 전체·단계별 상태, 범위와 객체 수, 시각·재시도·안전한 실패 코드. `scopeResults`에 범위별 내부 ID·지원 콘텐츠 유형·리전·수집 상태·항목 수·안전한 누락 코드만 포함 |
| V7 + UX·목록 필터 변경 | GET | `/api/v1/secrets/cases?page={page}&size={size}&origin=PRODUCTION` | 권한 있는 현재 대응 건의 마스킹 목록 `{items,total,page,size}`. `origin`은 `PRODUCTION` 또는 `TEST`, 기본은 `PRODUCTION`. API 페이지는 0부터(기본 0), 크기는 1–100(기본 20). 선택 필터 `stage`, `reviewStatus`, `assignmentStatus`, `changedSince`를 서버에서 적용. 항목에는 `reviewStatus`, `latestReviewDecision`을 추가하며 사유·티켓은 제외. 집계 스냅샷·심각도 등 확장 필터는 아직 없음 |
| V7–V10 + UX·V12 변경 | GET | `/api/v1/secrets/cases/{caseId}` | 현재 대응 건의 마스킹 키, `caseOrigin`, 단계·검토 상태, 담당자, 버전, 개설 위치와 연결된 위치의 ID·조치 상태, 담당자·검토·체크리스트·외부 조치·단계 변경 이력 및 호출자별 `capabilities`. V10은 `occurrences[]`에 `canRequestRevalidation`·`canResume`, 건 `capabilities`에 `canRequestRevalidation`·`canResolve`·`canResume`을 추가. UX 변경에서 위치별 `sourceType`, `safeLocationLabel`, `canCompleteActions` 및 건 `capabilities.canAssign`을 추가. V12는 검토 이력 `reviews[]`에 제한된 근거 참조와 후속 담당자 정보를 추가. `safeLocationLabel`은 호출자의 승인 범위에 맞는 GitHub `owner/repo · 위치 #ID` 또는 제한 표시만 제공하며 파일 경로·줄 번호·키 원문·수집 본문은 제외 |
| V7 | GET | `/api/v1/secrets/assignees?caseId={caseId}` | 해당 건의 모든 연결 범위에 `REMEDIATOR` 권한이 있는 기존 계정과 활성 팀 소속. `SECURITY` 권한 필요 |
| V8 | GET | `/api/v1/secrets/triage-candidates?page={page}&size={size}` | 연결별 `SECURITY` 권한이 있는 활성 GitHub 범위의 후보 키만 `{items,total,page,size}`로 조회. 항목은 키 ID, 마스킹 이름, 발급자, 유형, 구성 상태, 허용된 위치 수만 반환 |
| V8 | GET | `/api/v1/secrets/triage-candidates/{secretId}` | 같은 권한 경계의 후보 키와 선택 가능한 발견 위치를 조회. 위치 ID, 연결·범위 ID, 원천 위치, 환경, 노출·존재 상태와 판정·범위 버전 제공 |
| V8 | GET | `/api/v1/secrets/scopes/{scopeId}/response-origin` | 승인 범위의 `responseOrigin`, `responseOriginVersion`, 서버가 허용한 `allowedOrigins`, 분류 수행자·시각과 추가 전용 분류 이력. 비밀이 될 수 있는 원천 참조는 제외 |
| V8 | GET | `/api/v1/secrets/occurrences/{occurrenceId}/triage` | 위치의 `secretId`, `scopeId`, 구성·존재·노출 상태, `assessmentVersion`, 범위 구분·버전, 같은 키·구분의 현재 건 `openCaseId`(없으면 null), 해당 위치에 속하는 Finding ID·룰·탐지 시각과 수동 판정 이력. Finding 문맥과 원문은 제외 |
| V10 | GET | `/api/v1/secrets/cases/{caseId}/revalidations` | 해당 건의 위치별 재검증 요청·작업·결과 이력. 항목은 `revalidationId`, `occurrenceId`, `runId`, `jobStatus`, `result`, `failureCode`, `requestedAt`, `finishedAt`의 안전한 정보만 제공 |
| 후속 | GET | `/api/v1/secrets/summary` | 심각도 분포와 미평가, 조치 네 단계와 별도 예외, 유효성 분포, 미지정, 수집 및 검사 최신성, 집계 기준 |
| 후속 확장 | GET | `/api/v1/secrets/cases` | 현재 단계·검토·담당·시각 필터 외에 범위·스냅샷 필터를 갖춘 대응 목록. `scope=history`이면 과거 대응 건 목록. 발견 소스, 심각도, 유효성, 예외와 근거 추가 |
| 후속 확장 | GET | `/api/v1/secrets/cases/{caseId}` | 아직 없는 위치별 심각도·영향 근거, 예외, 실행 스냅샷과 과거 건 상세를 추가 |
| 후속 | GET | `/api/v1/secrets/matrix` | 심각도와 유효성의 교차 건수, 미평가 행, 동일 필터와 기준시점 |

MVP 인벤토리 목록은 고유 `CredentialSet` 단위 페이지네이션이며 `UNASSESSED`와 환경 `UNKNOWN`을 임의 값으로 치환하지 않는다. 최신 검증 시도가 없으면 `UNCHECKED`로 표시한다. AWS IAM의 `METADATA_ONLY`는 키의 존재 정보이지 비밀 값의 발견·노출·유효성 증거가 아니다. `iamMetadata.keyStatus`의 `ACTIVE`도 STS 검증 결과 `VALID`와 다르며 최근 사용 기록 부재는 미사용의 증명이 아니다. 정확한 AWS 본문에서 키 후보를 탐지해도 그 자원이 외부에 노출됐다는 뜻은 아니며 노출·영향·심각도는 별도 근거로 평가한다. V7–V9는 대응 건의 개설·담당·검토·외부 조치 이력을 제공하지만 집계 스냅샷과 실제 단계별 수는 아직 제공하지 않으므로 아래 목표 요약 예시를 현행 목록 응답으로 사용하지 않는다.

대응 목록의 `stage`는 `OPEN|IN_PROGRESS|REVALIDATION_PENDING|RESOLVED`, `reviewStatus`는 `UNREVIEWED|REVIEWED`, `assignmentStatus`는 `UNASSIGNED|TEAM_ONLY|PERSONAL` 중 하나다. `UNASSIGNED`는 팀·개인 모두 없음, `TEAM_ONLY`는 팀만 있음, `PERSONAL`은 개인 지정이다. `changedSince`는 시차를 포함한 ISO 8601 시각이며 `updated_at >= changedSince`로 경계를 포함한다. 네 필터를 함께 보내면 AND로 결합하고 권한·`origin` 범위를 먼저 제한한다. `items`와 `total`은 동일한 조건을 사용하므로 `total`은 현재 페이지의 길이가 아닌 필터 결과 전체 건수다. 잘못된 enum·날짜·`origin`은 400이며 데이터베이스에는 입력값을 바인딩한다. `latestReviewDecision`은 `CONFIRMED`, `NEEDS_MORE_INFO`, 검토 이력이 없으면 null이고 부족 사유·근거 참조는 목록에 포함하지 않는다. 운영·시험 탭의 필터와 무관한 건수는 각 `origin`에 필터 없이 별도 조회한 `total`로 표시하며 두 구분을 합산하지 않는다. 집계 스냅샷은 아직 없다.

수동 판정 화면은 기존 `ADMIN` 전용 인벤토리 API를 권한 우회용으로 확장하지 않는다. V8의 후보 목록·상세 API가 활성 GitHub 연결·승인 범위와 현재 사용자의 연결별 `SECURITY` 권한으로 위치를 SQL에서 제한한다. 한 키에 여러 위치가 있어도 권한 밖 위치는 상세 응답에 나타나지 않고, 응답의 위치 수는 허용된 위치의 수다. 원문 키·보호키 식별값은 반환하지 않는다.

현행 실행 이력 목록의 각 `items[]`는 `runId`, `connectionId`, `provider`, 현재 연결 `displayName`, `triggerType`, `status`, `collectionStatus`, `detectionStatus`, `validationStatus`, `scopeCount`, `completedScopeCount`, `incompleteScopeCount`, `collectedObjectCount`, `requestedAt`, `startedAt`, `finishedAt`, `failureCode`만 포함한다. 전체·단계 상태와 시각은 상세 API와 같은 의미다. `failureCode`는 허용된 코드만 노출하고 알 수 없는 내부 문구는 `UNSPECIFIED_FAILURE`로 바꾼다. 계정 번호, `auth_ref`, `approved_ref`, 자원 이름·ARN, 원문 값, 공급자 오류 본문과 범위별 상세는 목록에 넣지 않는다. `collectedObjectCount`는 확인한 원천 항목 수이며 **새로 발견한 키 수가 아니다**. `completedScopeCount`는 성공 수가 아닌 종료 수이고 `incompleteScopeCount`는 종료 중 누락 수라 서로 겹친다. 새 화면에서 각각 `성공`·`미완료`로 부르지 않는다. 스캔 이력은 `secret_scan_runs` 단위이고 `/cases?scope=history`의 과거 대응 건과 다르다. 현재 `displayName`은 실행 당시 이름의 스냅샷이 아니라 연결의 현행 표시 이름이다.

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
| POST (V9, V12 확장) | `/api/v1/secrets/cases/{caseId}/review` | `{decision,reasonCode,evidenceRef,expectedVersion}`. `CONFIRMED/EVIDENCE_CONFIRMED`는 `evidenceRef`를 생략하거나 null로 보내고 `REVIEWED`로 전이. `NEEDS_MORE_INFO`는 `SOURCE_CONTEXT_MISSING`, `OWNER_UNCONFIRMED`, `EXPOSURE_EVIDENCE_MISSING` 중 하나와 허용된 `evidenceRef`가 필수이며 `UNREVIEWED`·`OPEN`을 유지. 해당 연결의 `SECURITY` 권한 필요, 해결 상태는 자동 변경하지 않음 |
| POST (V9, UX 규칙 강화) | `/api/v1/secrets/cases/{caseId}/start` | `{reasonCode:"RESPONSE_STARTED",expectedVersion}`. 해당 건에 지정된 개인이면서 모든 연결에 `REMEDIATOR` 권한이 있어야 하며, 검토 확정 뒤 `OPEN`에서 `IN_PROGRESS`로 명시적 전이. 조치 기록의 필수 선행 단계 |
| PATCH (V9) | `/api/v1/secrets/cases/{caseId}/checklist/{itemId}` | `{completed,guideVersion:"S2_V1",evidenceRef,expectedVersion}`. 지정된 `REMEDIATOR`만 변경, `DONE`에는 제한된 티켓/변경 참조 필요. 체크만으로 해결 금지 |
| POST (V9, UX 규칙 변경) | `/api/v1/secrets/cases/{caseId}/actions` | `{actionType,occurrenceId,performedAt,evidenceRef,expectedVersion}`. 검토 확정과 `start` 이후 지정된 `REMEDIATOR`가 `IN_PROGRESS` 위치에 외부에서 수행한 조치를 추가 전용으로 기록. 같은 위치에 여러 건을 남겨도 상태는 `IN_PROGRESS` 유지. 공급자 키 변경 호출 없음 |
| POST (UX 변경) | `/api/v1/secrets/cases/{caseId}/occurrences/{occurrenceId}/complete-actions` | `{expectedVersion}`. 지정된 `REMEDIATOR`가 위치별 기록을 마침. 마지막 `start` 또는 재조치 재개 이후 `ROTATED`·`REVOKED`·`REMOVED_FROM_SOURCE` 중 하나 이상이 있어야 해당 위치를 `REVALIDATION_PENDING`으로 전이. `PERMISSION_REDUCED`만 있으면 409. 모든 위치가 대기면 건도 대기. 저장 후 최신 대응 건 상세 반환 |
| POST (V10) | `/api/v1/secrets/cases/{caseId}/revalidations` | `{occurrenceId,expectedVersion,idempotencyKey}`. 조치 기록을 마쳐 `REVALIDATION_PENDING`인 승인된 단일 GitHub 위치의 비동기 작업을 등록하고 재검증 식별자·작업 상태를 반환. 원문은 받지 않음 |
| POST (V10) | `/api/v1/secrets/cases/{caseId}/revalidation-resume` | `{occurrenceId,expectedVersion,reasonCode:"REVALIDATION_PRESENT"}`. 조치 후 재검증에서도 키가 계속 발견된 위치를 다시 `IN_PROGRESS`로 전환해 추가 외부 조치를 받음 |
| POST (V10) | `/api/v1/secrets/cases/{caseId}/resolve` | `{expectedVersion,reasonCode:"POST_ACTION_REVALIDATION"}`. `SECURITY` 권한과 모든 관련 위치의 조치 이후 최신 `MISSING` 근거·범위 버전 등 서버 조건을 검사한 후 `RESOLVED`로 전이 |
| POST (선택) | `/api/v1/secrets/cases/{caseId}/guide` | 본문 없음. `SECURITY` 또는 지정된 `REMEDIATOR`에게 버전 고정 기본 가이드를 반환하고, 별도 활성화·키·합성 `TEST` 범위 조건이 모두 맞을 때만 Groq 호출. 호출 실패는 기본 가이드로 처리; 상태·DB 변경 없음 |
| POST (후속) | `/api/v1/secrets/cases/{caseId}/exceptions` | 승인 범위, 사유, 만료 시각, 승인자 권한, `expectedVersion`. 전체 예외와 일부 위치 예외 구분 |
| POST (후속) | `/api/v1/secrets/cases/{caseId}/exceptions/revoke` | 취소 사유, `expectedVersion`. 남은 위험에 따라 단계 재개 |

V9의 체크리스트는 `IDENTIFY_OWNER`, `ROTATE_OR_REVOKE`, `REMOVE_EXPOSURE`, `CHECK_ACCESS_LOGS` 네 코드이며 `TODO`·`DONE`을 저장한다. 체크리스트는 보조 기록이며 4/4 완료는 위치별 기록 마침·재검증·해결의 서버 조건이 아니다. 외부 조치 종류는 `ROTATED`, `REVOKED`, `REMOVED_FROM_SOURCE`, `PERMISSION_REDUCED`다. `PERMISSION_REDUCED`는 임시 위험 완화로 기록하지만 현재 GitHub 현재 파일 미발견의 근거를 대체하지 않는다. 증거 참조는 `TICKET:SEC-123`처럼 `TICKET|CHANGE:대문자 프로젝트 코드-숫자`만 허용한다. 임의 URL·자유 문장·키 원문은 저장하지 않는다. 외부 수행 시각은 최근 30일에서 요청 시각보다 최대 5분 뒤까지만 허용한다. `NEEDS_MORE_INFO`는 검토 확정이 아니며 조치 시작·기록 권한을 열지 않는다. 한 건에 여러 위치가 연결되면 각 위치의 조치를 따로 기록하고 미완료 위치가 남은 건은 `IN_PROGRESS`로 둔다. 현재 개설 API는 위치 하나만 연결하며 추가 위치 연결은 후속이다. 건의 대표 티켓 재사용과 통합 시간순 이력 페이지 나눔은 후속 계약이다.

V12의 새 검토 요청은 `CONFIRMED`와 `EVIDENCE_CONFIRMED`·null `evidenceRef` 또는 `NEEDS_MORE_INFO`와 세 부족 근거 코드·필수 `evidenceRef` 조합만 허용한다. `evidenceRef`는 위 티켓·변경 참조 형식을 사용한다. `reviews[]`에는 `evidenceRef`, `followUpOwnerUserId`, `followUpOwnerName`이 추가된다. 후속 담당자는 별도 지정값이 아니라 해당 검토의 `reviewedByUserId` 보안 담당자에서 파생한다. V12 마이그레이션은 `secret_case_reviews.evidence_ref`를 nullable로 추가하고 기존 `INSUFFICIENT_EVIDENCE`·null 행은 보존한다. 새 API 입력에서는 그 이전 사유를 거부하며 기존 `REVIEW_RECORDED` 감사 이벤트와 버전 검사를 유지한다.

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

## 5. 근거 중심 검토와 스캔 이력의 목표 계약

이 절은 [ADR-S010](adr/ADR-S010.md)의 **미구현 목표**다. 새 경로의 응답과 변경은 1절의 세션·CSRF·범위별 역할·원문 비노출·UTC·안전한 오류 규약을 따른다. 정확한 마이그레이션 번호는 공통 DB 담당자가 정한다. 전환 시각과 `workflowVersion=LEGACY|EVIDENCE_V1`을 건과 감사 이력에 고정한다. 전환 전에는 이전 V8–V12 변경 API가 동작할 수 있지만, **전환 이후** 옛 범위 분류·판정·건 개설 쓰기 API는 `410 LEGACY_WRITE_DISABLED`로 막고 조회·이력만 남긴다. 예전 클라이언트를 위한 느슨한 새 건 개설 경로를 유지하지 않는다. 전환 전 개설된 `LEGACY` 건의 검토·조치는 권한 검사를 거쳐 이어갈 수 있지만 해결은 새 근거 계약으로만 수행한다. 새 화면은 같은 보안 판단을 여러 번 제출하지 않는다.

### 5.1. 연결 승인과 출처 교정

`OPERATIONS`가 범위를 제안하고 별도의 `SECURITY` 권한자가 승인한다. 출처를 `TEST`로 신규 승인하거나 `PRODUCTION`에서 `TEST`로 교정하는 경우 제안자와 승인자는 달라야 한다. 연결 승인 화면은 수집 허가, 대응 출처, 보관 허용 정책을 별도 필드와 설명으로 보여준다. 승인 정책이 적용되는 범위·브랜치·경로·콘텐츠 유형과 만료일을 검사하고 혼합된 시험·운영 범위는 분리한다.

| Method | 목표 경로 | 요청 또는 응답의 핵심 |
|---|---|---|
| GET | `/api/v1/secrets/scopes/{scopeId}/classification` | 현재 `origin`, `originVersion`, nullable `storagePolicyId`, `storagePolicyVersion`, 승인 상태·시각·수행자, 안전한 정책 요약과 변경 이력. `approved_ref`·계정·자원명 제외 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals` | `{origin,storagePolicyId?,storagePolicyVersion?,reasonCode,evidenceRef,expectedScopeVersion}`. `OPERATIONS` 제안, 검증 가능한 내부 정책·티켓 참조 필수. 정책이 없으면 두 필드를 모두 null로 두고 `reasonCode=POLICY_PENDING`; 보관 판정은 `UNKNOWN`. `PENDING_APPROVAL` 반환 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals/{proposalId}/approve` | `{expectedScopeVersion,expectedProposalVersion}`. 승인자는 서버 세션에서 결정. 승인 순간 범위 버전 증가·추가 전용 변경 이력·영향 대상 목록·집계 무효화가 원자적 |
| POST | `/api/v1/secrets/scopes/{scopeId}/classification-proposals/{proposalId}/reject` | `{reasonCode,expectedProposalVersion}`. `SECURITY`가 거절 사유와 이력을 남기고 이전 승인 분류를 유지. 승인 대기 제안을 활성 분류로 읽지 않음 |

미분류 또는 승인 대기 범위는 `/api/v1/secrets/scopes?classificationStatus=NEEDS_REVIEW`에서 운영·시험 집계와 별도로 찾는다. 승인 전에도 이전 탐지·실패 기록은 열람할 수 있고 조사 과제를 만들 수 있다. 범위 교정은 과거 대응 건의 출처를 직접 UPDATE하지 않는다. 영향 대상별 이전·새 건 연결을 승인된 전환 작업으로 기록한 뒤 새 스냅샷을 만든다. 기존 `PATCH /scopes/{scopeId}/response-origin`은 이전 화면에만 남기고 새 범위 분류·교정에 사용하지 않는다.

보관 정책은 `SECURITY`가 `POST /api/v1/secrets/storage-policies/drafts`에 연결·범위·콘텐츠 유형·정확한 경로/접두사 조건, `ALLOW|DENY`, 근거 참조, 유효 기간을 제출하고 **다른** `SECURITY`가 `POST /storage-policies/{policyId}/versions/{version}/approve`로 승인한다. 승인된 버전은 불변이며 `GET /storage-policies?status=ACTIVE`에서 권한에 맞는 안전한 요약만 조회·선택한다. 충돌하는 허용·금지 규칙은 활성화 전에 거부하고 평가 중 범위·버전 불일치나 만료는 `UNKNOWN`으로 남긴다. 정책 폐기는 새 승인 버전 또는 `POST /storage-policies/{policyId}/retire`의 사유·승인 기록으로 처리하며 기존 판정에 재검토 표식을 만든다. 화면은 보호된 `approved_ref`와 규칙의 민감한 전체 경로를 일반 사용자에게 노출하지 않는다. 범위 승인 API는 정책을 지정했다면 **활성 정책 버전**만 받으며, 정책 미설정 범위도 수집 승인은 가능하지만 보관 판정은 `UNKNOWN`·정책 작성 과제로 남는다. 정책 등록·승인·바인딩 모두 감사 기록과 버전 충돌 검사를 수행한다.

### 5.2. 검토 대기열과 근거

검토 대기열은 `GET /api/v1/secrets/review-queue?bucket=ACTION_REQUIRED|NEEDS_DECISION|NORMAL_INVENTORY|NEEDS_CASE_LINK&origin=PRODUCTION|TEST|UNCLASSIFIED&page={page}&size={size}`로 조회한다. `bucket`, `origin`, `page`, `size`는 허용 목록과 범위별 `SECURITY` 권한으로 검증한다. 응답은 `{items,total,page,size,unit,asOf,partialCoverage}`이며 `ACTION_REQUIRED`는 남은 작업이 있는 `OPEN|IN_PROGRESS|REVALIDATION_PENDING` 현재 `ResponseCase`, `NEEDS_DECISION`은 `UNKNOWN`·재검토 필요 `Occurrence`, `NORMAL_INVENTORY`는 `CredentialSet`, 전환용 `NEEDS_CASE_LINK`는 이전 방식으로 확정됐으나 건에 연결되지 않은 `ABNORMAL` `Occurrence`가 단위다. 해결·전체 위험 수용 건은 `ACTION_REQUIRED`에 넣지 않는다. 서로 다른 단위의 `total`을 더해 고유 키 수로 표시하지 않는다. 항목에는 `credentialSetId`, 선택 가능한 `occurrenceId`, `caseId`, 안전한 키 이름, 구성·존재 상태, 출처, 노출 판정·심각도·검증 각각의 상태, 마지막 성공·최근 시도, 정책 버전, 근거의 확인 수준, 누락 사유, 담당자·기한, 위치 수, `whyHere`, `nextAction`을 제공한다. `whyHere`와 `nextAction`은 사용자 자유문이나 생성형 AI 문장이 아니라 서버가 안전한 코드와 템플릿으로 만든다. 범위별 검사 누락은 키 대기열 수에 섞지 않고 `GET /api/v1/secrets/coverage-gaps`에서 `unit=Scope`로 제공한다.

`NORMAL_INVENTORY`의 키는 같은 출처 구분에 활성 대응 건과 판정 대기 위치가 없고, 위치 보관 `NORMAL`이 정책으로 증명됐거나 `METADATA_ONLY` 또는 폐기·만료가 공급자 근거로 확인된 경우에만 들어간다. 응답의 `inventorySubtype=AUTHORIZED_STORAGE|METADATA_ONLY|RETIRED_HISTORY`를 반드시 표시한다. `CANDIDATE`·사유 불명 `INVALID`·단순 건 없음은 정상 근거가 아니다. 허용 위치와 비정상 위치가 같이 있는 키는 정상 키 한 건으로 중복 표시하지 않고 대응·판정 상세에서 허용 위치를 보여준다. 전체 인벤토리 API는 이 대기열과 달리 해당 키를 계속 조회할 수 있다. `UNCLASSIFIED`는 정상 인벤토리로 자동 분류하지 않는다.

`GET /api/v1/secrets/occurrences/{occurrenceId}/review-context`는 화면의 단일 근거 카드다. 구성·발급자·원천 유형, 권한에 맞는 위치 표시, 해당 Finding의 룰·버전·탐지 시각, 현재 존재와 마지막 성공 검사/최근 시도, 범위 출처·보관 정책 ID/버전·적용 이유, 검증 결과·방법·시각, 확인된 주체·권한·영향, 동일 키의 열람 가능한 다른 위치, 기존 건, 부족 근거 코드와 후속 과제, `allowedDecisions`, 각각의 예상 버전을 반환한다. 각 사실은 `CONFIRMED|UNKNOWN|NOT_CHECKED`와 출처·관측 시각을 따로 가진다. 응답의 `UNKNOWN`은 무효·정상·낮은 심각도로 치환하지 않는다. 비밀 값, 마스킹 전후 문맥, HMAC 지문, 권한 밖 위치 수와 원천 경로는 반환하지 않는다. `404`와 `403`은 안전한 오류 화면 및 복귀 경로로 처리하고 로딩 중 null을 결론처럼 렌더링하지 않는다.

`GET /api/v1/secrets/investigations?status=OPEN|OVERDUE&owner=ME&page=&size=`와 `GET /investigations/{taskId}`는 위치/범위 구분, 부족 근거 코드, 담당자·기한, 제한된 대상 링크와 최근 활동을 조회한다. `PATCH /investigations/{taskId}/assignment`는 `{assigneeUserId,dueAt,reasonCode,expectedVersion}`, `POST /investigations/{taskId}/evidence`는 `{evidenceRef,evidenceKind,observedAt,expectedVersion}`, `POST /investigations/{taskId}/close`는 `{expectedVersion,resolvingDecisionId|restoredRunScopeId}`를 받는다. 관련 위치가 새 판정으로 확정되거나 누락 범위가 성공적으로 다시 검사된 근거가 없으면 과제를 닫을 수 없다. 범위별 역할을 다시 검사하고, 담당자 변경·기한 초과·증거·종료를 추가 전용 이력에 남긴다. 과제 종료만으로 `NORMAL`, `ABNORMAL` 또는 검사 성공을 만들지 않는다.

`TEST` 위치의 `fixtureStatus=NOT_APPLICABLE|VERIFIED_SYNTHETIC|POSSIBLE_REAL|CONFIRMED_REAL`은 출처 구분과 독립이다. `CANDIDATE`·`METADATA_ONLY`와 현재 유효 Finding이 없는 위치는 `NOT_APPLICABLE`이며 조사·인벤토리에는 남긴다. `VERIFIED_SYNTHETIC`은 승인된 fixture 목록·버전과 **해당 키**가 일치한 근거가 있을 때만 반환한다. `POSSIBLE_REAL`은 `CONFIRMED` 값 후보가 성공한 최신 검사에서 `PRESENT`이고 유효한 Finding이 남았으며 fixture 일치를 확인했지만 합성으로 증명되지 않았다는 뜻이다. 목록 확인 실패도 사유와 함께 이 상태로 둔다. 이는 유효성 `VALID`와 같지 않다. **이 조건이 충족되는 즉시** `GET /api/v1/secrets/cross-environment-alerts`에 `SUSPECTED_REAL` 운영 보안 확인 경고를 열고 담당자·기한·권한에 맞는 위치 링크를 제공한다. `testCaseId`는 건이 있는 경우에만, 개설 조건을 충족하지 못한 후보의 `investigationTaskId`는 조사 과제가 있는 경우에 제공한다. 두 ID가 모두 없는 경고는 담당자와 조사 과제를 원자적으로 만들어 방치하지 않는다. 같은 CredentialSet의 반복 검사·추가 위치는 열린 경고를 갱신하고 새 경고를 만들지 않는다. 실제 발급자 근거가 들어오면 같은 경고를 `CONFIRMED_REAL`로 승격한다. 경고의 `acknowledged`는 인지 기록이고 조치 완료가 아니다. TEST 건은 TEST 집계에 남지만 운영 화면의 이 경고를 숨기지 않는다. 경고 수는 운영 `ResponseCase` 수와 별도 단위로 표시한다. 합성 여부가 불명확한 동안 시험 종료는 막는다.

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

`ABNORMAL`에는 현재 위치·형식 탐지와 적용 정책 또는 명시적인 비승인 위치 근거가 필요하다. 유효성 `UNKNOWN`·`UNCHECKED`는 이 결정을 자동 차단하거나 안전 판정으로 바꾸지 않는다. 성공 응답은 `201`(건 생성) 또는 `200`(기존 건에 위치 연결)과 `{decisionId,occurrenceId,exposureStatus,caseId,caseOperation:"CREATED|ATTACHED",caseVersion,queueBucket}`를 반환한다. 기존 건에 같은 위치가 이미 있으면 동일 멱등 요청은 기존 결과를 반환하고 새 이력·건을 만들지 않는다. 다른 출처 건에 붙이지 않으며 해결된 건 뒤 새 노출은 `previousCaseId`로 연결한 새 건을 만든다. 새 건은 이 보안 결정으로 검토 근거가 충족됐으므로 별도 `CONFIRMED` 요청 없이 담당자 지정·조치 시작이 가능하다. 레거시 `UNREVIEWED` 건은 소급 확정하지 않는다.

이미 `REVALIDATION_PENDING`인 건에 새 `ABNORMAL` 위치가 붙으면 그 위치의 미완료 조치 주기를 열고 건을 **같은 트랜잭션에서 `IN_PROGRESS`로 되돌린다**. `OPEN`·`IN_PROGRESS` 건은 기존 단계를 유지한다. 건 버전·단계 이력·전이 사유와 요약 집계 무효화를 함께 확정한다. 기존 위치의 과거 재검증은 보존하지만 새 위치가 미조치인 동안 해결은 차단한다. 전체 위험 수용 예외가 새 위치를 포함하지 않으면 예외 집계에서 빼고 미해결 단계로 보여준다.

새 건은 `openingDecisionId`를 보존하고 검토 상태를 `REVIEWED`, 근거 출처를 `OPENING_DECISION`으로 노출한다. 기존 `CONFIRMED` 검토 행을 빈 기록으로 만들지 않는다. 이전 방식 `UNREVIEWED` 건은 권한 있는 보안 담당자가 기존 검토를 완료하거나 새 결정으로 전환한 뒤 진행한다. 새 결정이 기존 건에 위치를 붙이면 그 위치의 근거 ID를 추가하고 다른 위치의 판정·조치 상태는 바꾸지 않는다.

전환용 `POST /api/v1/secrets/occurrences/{occurrenceId}/case-links`는 이미 저장된 V8 `ABNORMAL` 판정의 ID·버전, 현재 위치/범위 버전, 멱등 키를 받는다. `SECURITY`가 원래 판정을 재입력하지 않고 현재 `CONFIRMED` 키·`PRESENT` 위치, 판정 근거의 적용 가능성, 대응 출처와 권한을 확인한 뒤 해당 키·출처의 건에 연결하거나 없으면 새 방식 건을 연다. 근거가 낡았으면 409로 새 근거 카드·재판정으로 안내한다. 성공 시 원래 결정과 전환 연결 이력을 모두 보존하고 중복 위치·건을 만들지 않는다. 이 경로는 이전 방식의 미연결 위치에만 허용하고 신규 판정에는 사용하지 않는다.

`NORMAL`은 `CONFIRMED`·`PRESENT`인 실제 값 위치와 유효한 Finding, 실제 보관 허용 정책·적용 범위와 만료 근거를 요구한다. `CANDIDATE`·`METADATA_ONLY`는 보관 허용 위치로 자동 판정하지 않는다. `FALSE_POSITIVE`는 자격 증명이 아님을 확인한 유형별 근거를 요구하며 대응 건을 새로 만들지 않는다. `FALSE_POSITIVE`는 Finding 교정 이력에만 기록하고 위치 노출을 `NORMAL`로 바꾸지 않는다. 같은 위치에 독립된 유효 Finding이 남으면 그 위치는 검토 대기열에 계속 남는다. `NEEDS_MORE_INFO`는 노출을 `UNKNOWN`으로 유지하고 `missingReasonCode`, `followUpOwnerUserId`, `dueAt`, 안전한 `evidenceRef` 또는 내부 과제 ID를 필수로 받아 대기열에 남긴다. 키 원문·자유 메모는 받지 않는다. 판정 재검토는 기존 이력을 덮지 않고 새 판정으로 추가한다. 정책·범위·Finding이 바뀌어 예상 버전이 오래되면 409 `STALE_REVIEW_CONTEXT`와 안전한 변경 항목을 반환하고 현재 근거를 다시 조회하게 한다. 입력 근거 부족은 422 `EVIDENCE_REQUIRED`, 권한 범위 밖은 403, 대상 없음은 404다. 대기열·건 상세와 판정 결과는 저장 후 같은 새 버전을 가리킨다.

같은 위치의 **모든 최신 Finding**이 `FALSE_POSITIVE`로 교정되고 다른 유효 Finding이 없을 때만 그 위치를 활성 `NEEDS_DECISION`에서 빼며 제외·판정 이력에서는 계속 조회한다. 승인된 오탐 결정은 같은 범위·원천 객체·정확한 객체 버전·위치·엔진/룰 버전·보호된 후보 동일성이 모두 같은 새 Finding에만 최대 30일 재사용한다. 서버는 새 Finding에 원래 결정 ID를 참조한 `EXACT_EVIDENCE_REUSE` 교정 행을 새로 쓰고 감사에 남긴다. 객체 버전이나 동일성 근거가 없거나 하나라도 달라지고, 기한 만료·승인 취소·다른 유효 Finding이 있으면 다시 대기열에 올린다. 구 오탐 이력은 조건을 증명할 수 없으면 재사용하지 않는다. `review-context`는 현재 교정 출처·만료일과 재검토 이유를 표시한다.

정확한 위치·콘텐츠 유형·범위·유효 기간에 일치하는 **명시적 허용 규칙**만 자동 `NORMAL` 판정에 사용할 수 있다. 정책 평가기는 `policyResult=ALLOWED|DENIED|UNKNOWN`과 적용 규칙·정책 버전을 남긴다. `CONFIRMED`·`PRESENT` 값 위치에 유효 Finding이 있고 `ALLOWED`일 때만 `decisionSource=POLICY`와 근거 스냅샷을 가진 새 `NORMAL` 이력을 기록한다. `DENIED`는 근거 있는 보안 검토 권고이지 자동 `ABNORMAL`이나 대응 건 개설이 아니다. 규칙 부재·충돌·정책 만료는 `UNKNOWN`이다. 수동 판정 뒤의 새 자동 결과가 사람의 결정을 조용히 덮지 않으며 정책·원천 버전이 바뀌면 재검토 표식과 이전 근거를 같이 보여준다.

`GET /api/v1/secrets/cases/{caseId}`의 목표 `occurrences[]`는 **호출자가 접근할 수 있는 위치만** 안전한 표시·노출 판정·담당 팀/개인·조치 주기·최근 재검증·막힌 이유와 함께 반환한다. `visibleOccurrenceCount`도 그 배열의 수이며 전체 위치 수가 아니다. 서버는 권한 밖 위치의 행·정확한 수·식별자를 숨기고 필요할 때만 `restrictedCoverage=true`라는 일반 신호를 준다. 건 전체에 권한이 있는 보안 조정자만 모든 연결 위치와 정확한 수를 본다. 보안 담당자는 건 전체를 조정하지만 `REMEDIATOR`는 할당되고 권한이 있는 **위치**만 조치한다. 모든 연결에 일괄 `REMEDIATOR` 권한을 요구하는 현행 담당자 목록/조치 조건은 목표에서 위치별 권한으로 바꾼다. `GET /api/v1/secrets/cases/{caseId}/closure-checks`는 서버 내부에서 모든 관련 위치·미연결 비정상 위치·불확실 위치·검사 누락, 조치 이후 비교 가능한 재검증, 자격 증명의 교체·폐기 근거를 검사한다. 응답은 호출자가 볼 수 있는 범위의 `PASSED|BLOCKED|UNKNOWN`과 안전한 사유·확인 범위를 반환하고 제한 위치의 세부·정확한 수는 내보내지 않는다. 공급자 상태를 자동 확인하지 못한 외부 수행 증거는 `APPROVED_MANUAL` 확인 수준으로 분리한다.

제한 위치가 있는 사용자에게 `capabilities.canResolve=false`를 반환하고 상단에는 ‘권한 있는 보안 조정자의 전체 확인 필요’라는 일반 안내만 제공한다. 해결 명령은 서버가 **모든 연결 위치와 미연결 현재 노출에 대한 `SECURITY` 접근 권한**을 검사하고 어느 하나라도 없으면 403으로 거부한다. 권한이 있더라도 보안 조정자는 모든 필수 해결 근거를 별도로 충족해야 한다. 제한 위치의 이름·수는 거부 응답에 넣지 않는다.

`PATCH /api/v1/secrets/cases/{caseId}/occurrences/{occurrenceId}/assignment`는 `{team,assigneeUserId,reasonCode,expectedCaseVersion,expectedOccurrenceVersion}`으로 위치별 담당자를 바꾼다. `SECURITY`가 해당 연결의 역할과 지정 대상의 연결별 `REMEDIATOR` 권한·팀 소속을 확인한다. 건 조정 담당자와 위치별 조치 담당자는 다른 필드이며, 위치 담당자 한 명의 권한을 다른 연결 위치로 확장하지 않는다. 목표 `POST /cases/{caseId}/actions`는 지정되고 해당 위치에 권한 있는 담당자가 첫 실질 외부 조치를 기록할 때 해당 위치와 건을 `OPEN→IN_PROGRESS`로 함께 전환할 수 있다. 현행 `/start`와 선행 `CONFIRMED` 게이트는 구 방식 건의 호환 경로로 남기되 새 방식 건에는 같은 사실의 두 번째 승인이나 빈 단계 전이 버튼을 요구하지 않는다. 한 위치의 여러 조치 기록은 계속 허용하고 `complete-actions`로 해당 위치의 기록 마침을 선언한다.

목표 `POST /api/v1/secrets/cases/{caseId}/resolution-decisions`는 `{decision:"RESOLVE",expectedCaseVersion,closureCheckVersion,evidenceIds[],coverageLevel:"FULL|LIMITED|SIMULATION",idempotencyKey}`와 `SECURITY` 권한을 요구한다. 서버는 현재 건과 키·범위·연결 버전, **모든** 위치의 마지막 실질 조치 후 비교 가능한 `MISSING`, 미연결 `ABNORMAL` 없음, 현재 `UNKNOWN` 위치 없음, 필수 수집 범위 누락 없음을 같은 트랜잭션에서 다시 검사한다. 이어서 **실제 자격 증명**은 공급자 또는 승인된 수동 교체·폐기 수명주기 근거를 요구하고 `FULL|LIMITED`로 확인 범위를 기록한다. **승인 fixture와 키별로 일치한 합성 TEST**만 fixture 제거 증빙과 비교 가능한 재검사로 `SIMULATION`을 허용하며 공급자 키 폐기로 표시하지 않는다. `POSSIBLE_REAL|CONFIRMED_REAL`은 `SIMULATION`으로 닫을 수 없다. 실제 키가 시험 출처에 있었더라도 실제 수명주기 근거와 관련 교차 환경 경고의 처리가 확인되면 `FULL|LIMITED` 경로를 쓸 수 있다. 막힌 항목은 409 `CLOSURE_BLOCKED`와 안전한 `closureChecks[]`로 반환하고 상태를 바꾸지 않는다. `FULL`은 선언된 승인·지원 범위의 모든 필수 확인이 성공한 때만 가능하다. 지원 밖 이력이나 제한된 수동 확인이 남는 `LIMITED`는 확인 한계를 고정하며 알려진 미해결 노출 또는 필수 검사 실패를 우회하지 못한다. 성공은 해결 근거 스냅샷, 승인자·시각·버전을 추가 전용으로 남긴다. 전환 이후 기존 `POST /cases/{caseId}/resolve`는 모든 건에서 `410 LEGACY_WRITE_DISABLED`를 반환한다. 기존 `LEGACY` 미해결 건도 새 근거 계약을 충족한 뒤에만 해결하며 구 경로로 우회할 수 없다.

원래 노출 판정 자체가 오류였던 건의 종료는 해결과 다른 `POST /api/v1/secrets/cases/{caseId}/dismissal-decisions`에서 `{reasonCode:"FALSE_POSITIVE|OUT_OF_SCOPE",evidenceIds[],expectedCaseVersion,idempotencyKey}`로 요청한다. `FALSE_POSITIVE`는 **모든** 연결 위치의 Finding이 실제 자격 증명이 아님을 유형별 근거로 교정했고 다른 유효 Finding이 없을 때만 허용한다. `OUT_OF_SCOPE`는 대상이 프로그램 범위 밖이라는 승인된 범위·소유자 이관 근거와 인수 확인을 요구하며, 실제 미해결 비승인 노출을 단순히 제외하는 데 쓸 수 없다. `SECURITY`가 원래 판정·새 근거·타 연결의 미연결 위치를 확인하고 `disposition`, `is_current=false`, `closedAt`과 불변 감사 기록을 함께 저장한다. 결과는 `RESOLVED`·위험 수용 수에 넣지 않고 교정 이력에서 찾는다.

### 5.4. 실행 이력과 발견 연결

목표 `GET /api/v1/secrets/runs?type=SCHEDULED|MANUAL|REVALIDATION|ALL&status=&connectionId=&from=&to=&page=&size=`는 세션과 연결별 조회 권한을 먼저 적용하고 필터를 AND로 결합한 뒤 `requestedAt DESC,runId DESC`로 정렬·페이지 처리한다. `page`는 API에서 0부터, `size`는 1–100(기본 20)이며 목록 화면의 1부터 시작하는 페이지를 변환한다. `from`·`to`는 UTC ISO 8601 경계로 받아 요청 시각의 `from <= requestedAt < to`에 적용한다. `status`는 허용된 실행 결과 enum만, `connectionId`는 권한 있는 연결 ID만 받는다. 잘못된 값은 400, 권한 밖 연결은 403이며 응답 `{items,total,page,size,filters,asOf}`의 `total`은 같은 권한·필터가 적용된 **전체 실행 수**다. 목록 상단 집계도 같은 기준시각·필터에서 서버가 계산해 제공하지 않으면 화면은 숫자 카드를 숨긴다. 결과 항목과 `total`은 같은 조회 스냅샷을 사용해 페이지 간 모순을 피한다.

`GET /api/v1/secrets/runs`와 `GET /runs/{runId}`의 목표 응답은 실행 시점의 연결 표시·범위 버전을 보존하고 `triggerType=SCHEDULED|MANUAL|REVALIDATION`을 그대로 표시한다. 재검증 실행은 전체 스캔 성공률 분모에 넣지 않는다. `scopeStatusCounts`는 `PENDING`, `RUNNING`, `SUCCEEDED`, `PARTIAL`, `FAILED`, `SKIPPED`를 모두 포함하며 합이 `scopeCount`다. `finishedScopeCount=SUCCEEDED+PARTIAL+FAILED+SKIPPED`, `successfulScopeCount=SUCCEEDED`, `gapScopeCount=PARTIAL+FAILED+SKIPPED`, `unverifiedScopeCount=scopeCount-SUCCEEDED`를 함께 반환한다. `unverifiedScopeCount`에는 진행 중인 `PENDING|RUNNING`이 들어가므로 최종 실패·누락인 `gapScopeCount`와 다르게 표시한다. 현행 `completedScopeCount`·`incompleteScopeCount`는 전환 기간에 기존 의미로 유지하고 새 화면의 성공·미완료 라벨로 사용하지 않는다.

목표 **실행 결과** `status`에는 `CANCELLED`를 추가하되 범위·단계의 여섯 상태와 섞지 않는다. 작업이 취소되면 실행 결과를 `CANCELLED`로 확정하고, 이미 끝난 범위는 그대로 보존하며 미착수 범위는 `SKIPPED/JOB_CANCELLED`, 진행하다 멈춘 범위는 `FAILED/JOB_CANCELLED`로 확정한다. 기존 실행 결과 enum에 취소가 없던 기록은 근거 없이 다시 쓰지 않는다. 실행 목록의 취소 필터·배지는 이 목표 확장 뒤에만 제공한다.

집계 기준 `scopeOutcome`은 범위의 수집 상태와 해당 범위에 **필수로 적용되는** 탐지·검증 단계의 결과를 합성한다. 필수 탐지/검증 실패는 수집이 성공해도 `PARTIAL`이며, 메타데이터 전용의 `NOT_APPLICABLE` 단계와 명시적으로 선택 비활성인 검증은 범위 성공에서 제외하되 단계별 미실행 사유를 응답에 반드시 남긴다. 검증 설정이 켜졌지만 실패한 경우는 범위 `PARTIAL`이다. 오래된 실행에 필수 단계·적용 여부 기록이 없어 신뢰할 수 없는 경우 `scopeCount`는 제공하되 성공/누락 집계에는 `aggregationStatus=LEGACY_UNAVAILABLE`을 주고 화면은 수치를 추정하지 않는다.

각 `scopeResults[]`는 `runScopeId`, 내부 `scopeId`, 운영자가 별도로 승인한 `safeDisplayAlias` 또는 `범위 #ID`, 콘텐츠 유형·리전, 수집·탐지·검증 단계별 상태, 객체 수, 안전한 실패/건너뜀 코드, 시작·종료 시각, 재시도 가능 여부를 제공한다. 별칭은 `approved_ref`·자원명·ARN·파일 경로에서 자동 생성하지 않으며 권한 범위에 따라 제한한다. 실행의 `collectedObjectCount`와 `findingCount`, `distinctCredentialCount`, `newCredentialCount`는 별도 단위·정의·집계 시점을 가진다. 키를 읽지 않는 IAM/목록 메타데이터 범위는 탐지·새 키 수를 0과 혼동하지 않도록 `SKIPPED`와 이유를 제공한다. 부분 실패로 확인되지 않은 범위는 발견 0건이라고 단정하지 않는다.

`distinctCredentialCount`는 이 실행의 값 후보 Finding에 연결된 고유 `CredentialSet` ID 수, `newCredentialCount`는 그중 동일성 키의 원자적 최초 삽입 때 `firstSeenRunId=runId`로 기록된 ID 수다. 같은 후보의 여러 Finding·위치는 한 번만 센다. 이전에 발견한 후보의 새 위치나 동일 항목의 `CANDIDATE→CONFIRMED` 승격은 신규 후보 수에 넣지 않는다. 메타데이터 전용·재검증 실행에는 `newCredentialCount=null`과 미적용 사유를, 과거 실행에 최초 생성 실행을 복원할 수 없거나 결과 연결이 없으면 `null`과 `aggregationStatus=LEGACY_UNAVAILABLE|RESULT_LINK_UNAVAILABLE`을 준다. 부분 실패에서 계산한 수는 확인된 범위의 하한으로만 표시하고 `coverageComplete=false`를 붙인다. 전체 커버리지와 결과 연결이 확인돼 실제 0건인 때에만 숫자 `0`을 반환한다.

`GET /api/v1/secrets/runs/{runId}/findings?page=&size=`는 해당 실행에 연결된 허용 범위의 고유 키·위치·Finding을 구분해 반환하고, `/secrets/findings?runId={runId}`는 같은 서버 필터를 사용한다. 접근 범위 밖 결과나 원문은 노출하지 않는다. 실행 상세는 실패 범위의 재시도 또는 연결 관리로, 발견 항목의 검토 대기열로, 재검증 실행의 대응 건으로 이동할 안전한 링크 식별자만 반환한다. 404·403·대기·일부 실패·없음은 다른 화면 상태다. 화면 경로 `/secrets/history/:runId`는 직접 열 수 있고 이전 `?run=` 주소는 목록 조건을 보존해 옮긴다.

### 5.5. 집계와 전환 검증

`GET /api/v1/secrets/summary`와 대응 목록은 같은 `snapshotId`, `scope`, `unit`, 출처 구분과 권한 필터를 적용한다. 네 조치 단계·별도 예외·미평가 심각도는 각각 현재 대응 건 수와 맞아야 한다. 노출 검토가 필요한 위치, 정상 인벤토리 키, 검사 누락 범위는 서로 단위가 달라 이 합에 넣지 않는다. 스냅샷에는 부분 수집 실패와 마지막 성공 시각을 고정해 미확인을 실제 0으로 표시하지 않는다. 기존 V8–V12 API는 새 클라이언트로 전환한 뒤 실제 사용처를 확인하고 제거하며, 과거 감사·검토·범위 분류·위치 판정 기록은 읽기 경로에서 보존한다. 빈 DB와 기존 개발 DB의 순차 마이그레이션, 중복 요청·경합·정책 버전 변경·권한 오류·다중 위치·부분 실패·레거시 링크를 계약 시험으로 검증한다.
