# 대시보드 API 계약

기준: [PRD](../01-requirements/PRD.md), [ADR-S008](adr/ADR-S008.md), [데이터 계약](data-contract.md), [대응 정책](../02-functional-design/response-policy.md)

이 문서는 새 Secret 기능의 구현 계약이다. 아래 경로는 현재 코드에 존재한다고 주장하지 않는다. 기존 `/api/v1/overview`의 실제 응답과 호환성은 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)를 따라 점검한다.

## 1. 공통 계약

모든 조회와 변경은 기존 세션 인증 경계를 통과한다. 변경 요청은 POST·PATCH·PUT·DELETE 모두 CSRF 검증과 서버 권한 검사를 적용한다. 공급자 연결 자격 증명과 발견된 키 원문은 요청, 응답, 오류, 로그에 넣지 않는다. 응답 필드는 camelCase, 상태·분류 enum은 대문자, 시각은 UTC ISO 8601 형식을 사용한다. `scope`의 `currentCases`·`inventory`·`history`, `unit`의 `ResponseCase`·`CredentialSet`은 계약 식별 문자열이며 아래 표기를 유지한다. 공통 기준은 [대시보드 API 명세](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)를 따른다.

조회는 `source`, `provider`, `environment`, `credentialType`, `validity`, `severity`, `actionStatus`, `exception`, `assignee`, 기간 필터 중 해당 경로에 맞는 값을 받는다. 응답은 적용된 `filters`, `scope`, `unit`, `asOf`, `snapshotId`, `total`, `partialCoverage`를 반환한다. 요약, 매트릭스와 기본 대응 목록은 `scope=currentCases`, `unit=ResponseCase`이고 각 키의 최신 건만 센다. 인벤토리는 `scope=inventory`, `unit=CredentialSet`이다. 이력 목록은 `scope=history`, `unit=ResponseCase`로 과거 건을 각각 센다. 화면은 첫 조회의 `snapshotId`를 매트릭스와 목록에 전달한다. 같은 스냅샷과 필터의 카드, 매트릭스, 목록 건수는 일치해야 한다. 만료된 스냅샷은 409와 새 조회 안내를 반환한다. 페이지네이션은 안정된 정렬 키와 다음 커서를 사용한다.

오류의 필수 필드는 공통 규약과 같은 `code`, `message`다. `requestId`, `retryable`, `details`는 선택 확장 필드이며 기존 클라이언트는 없어도 처리한다. `requestId`는 원문을 포함하지 않는 추적값, `retryable`은 재시도 가능 여부이며 `details`에는 키 원문, 원천 콘텐츠나 임의 내부 식별자를 넣지 않는다. 입력 오류는 400 `INVALID_INPUT`, 세션 없음은 401 `UNAUTHENTICATED`, 권한 없음은 403 `FORBIDDEN`, 대상 없음은 404 `NOT_FOUND`, 메서드 오류는 405 `METHOD_NOT_ALLOWED`, 버전 충돌과 스냅샷 만료는 409 `CONFLICT`, 대시보드 요청 제한은 429 `RATE_LIMITED`, 서버 장애는 5xx로 구분한다. 409의 상세 사유는 안전한 확장 정보로 구분한다. 공급자 검사 결과의 401, 403, 404 등은 대시보드 HTTP 오류와 별도로 ValidationAttempt의 사유로 기록한다.

`source`는 발견 위치의 수집 플랫폼, `provider`는 키 발급자 필터다. 예를 들어 Notion에서 발견한 AWS 키는 `source=NOTION`, `provider=AWS`다. 환경은 `ALL`, `PROD`, `DEV`, `UNKNOWN`이며 `ALL`은 UNKNOWN을 포함한다. 기존 Overview는 UNKNOWN을 지원하지 않으므로 화면 전환 전 그 경로에 UNKNOWN을 전달하지 않는다.

## 2. 조회

| Method | 경로 | 응답의 핵심 정보 |
|---|---|---|
| GET | `/api/v1/secrets/summary` | 심각도 분포와 미평가, 조치 네 단계와 별도 예외, 유효성 분포, 미지정, 수집 및 검사 최신성, 집계 기준 |
| GET | `/api/v1/secrets/cases` | 기본 현재 대응 건 목록. `scope=history`이면 과거 대응 건 목록. 마스킹한 식별자, 발견 소스와 발급자, 심각도, 유효성, 담당자, 단계, 예외, 시각, 커서 |
| GET | `/api/v1/secrets/cases/{caseId}` | 대응 건, 위치별 근거와 상태, 검사 및 평가 이력, 담당자, 체크리스트, 조치 및 감사 이력 |
| GET | `/api/v1/secrets/matrix` | 심각도와 유효성의 교차 건수, 미평가 행, 동일 필터와 기준시점 |
| GET | `/api/v1/secrets/inventory` | 전체 고유 키 목록에서 정상 보관, 미검증, 대응 중, 만료 및 폐기 기록을 구분 |
| GET | `/api/v1/secrets/connections` | 승인 범위, 지원 및 미지원 콘텐츠, 최근 수집 결과, 누락과 지연. 연결 비밀은 제외 |
| GET | `/api/v1/secrets/runs/{runId}` | 수집 또는 검증 실행의 범위, 시각, 결과, 부분 실패와 재시도 정보 |

상세의 발견 위치 링크는 허용된 경로만 생성하고 대상 사용자의 원천 접근 권한을 다시 확인한다. 임의 원천 URL을 그대로 리다이렉트하지 않는다. 마스킹된 문맥도 비신뢰 텍스트로 렌더링한다.

요약 응답의 핵심 형태는 다음과 같다. 수치는 계약 예시이며 실제 운영 결과가 아니다.

```json
{
  "scope": "currentCases",
  "unit": "ResponseCase",
  "asOf": "2026-10-09T00:00:00Z",
  "snapshotId": "example-snapshot-1",
  "filters": {"environment": "ALL"},
  "total": 3,
  "actionCounts": {"OPEN": 1, "IN_PROGRESS": 1, "REVALIDATION_PENDING": 0, "RESOLVED": 0, "RISK_ACCEPTED": 1},
  "severityCounts": {"CRITICAL": 1, "HIGH": 0, "MEDIUM": 1, "LOW": 0, "UNASSESSED": 1},
  "partialCoverage": false,
  "missingSources": []
}
```

`actionCounts`의 다섯 값과 `severityCounts`의 다섯 값은 각각 `total`과 같다. 수집 실패가 있으면 `partialCoverage=true`와 실패한 범위, 사유, 시각을 `missingSources`에 포함한다. 일부 수집 실패를 정상 0건으로 합산하지 않는다.

## 3. 변경

### 수동 전체 스캔

정식 수동 전체 스캔은 `/api/v1/secrets` 아래에서 비동기 작업으로 등록하고 ScanRun의 실행 ID를 반환한다. 화면은 `GET /api/v1/secrets/runs/{runId}`로 상태를 조회한다. API 요청에서 원천 콘텐츠를 읽거나 탐지 도구를 실행하지 않는다. 등록 엔드포인트의 세부 경로, 요청·응답 필드와 HTTP 상태는 공통 검토 후 구현하며 이 문서가 구현 완료를 뜻하지 않는다.

현재 `/api/v1/credentials`와 `/api/v1/credentials/scan`은 기존 화면 전환 기간에만 유지한다. 화면과 다른 호출자를 정식 API로 전환하고 사용처가 없음을 확인한 뒤 제거한다. 전환 기간의 필드와 현재 구현의 차이는 [구현 연동 현황](../04-integration/current-implementation-and-gaps.md)에 기록하며 정식 모델이나 실행 경계로 채택하지 않는다.

### 대응 건 변경과 재검증

| Method | 경로 | 요청과 불변 조건 |
|---|---|---|
| PATCH | `/api/v1/secrets/cases/{caseId}/assignment` | 팀과 개인, 사유, `expectedVersion`. 수행자, 변경 전후, 시각 감사 |
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

계정 인증은 기존 `dashboard_users`와 세션을 사용한다. `ADMIN`이라는 값만으로 Secret의 `SECURITY`, `REMEDIATOR`, `OPERATIONS` 권한을 자동 부여하지 않고 `secret_role_grants`와 연결 범위를 확인한다. 초기 관리자에게 필요한 역할을 부여하는 절차, 추가 담당자 계정의 생성·비활성화, 최초 역할 부여의 초기화 방식과 감사는 다중 사용자 연동 전에 구현 계획으로 확정한다. 현재 자동 등록 관리자 1개와 이름 문자열만으로 여러 개인 담당자가 준비됐다고 표시하지 않는다. 공개 회원가입이나 새로운 로그인 방식은 이 문서에서 도입하지 않는다.

기존 `/overview` 응답의 숫자 점수는 새 Secret 요약의 근거가 아니다. 기존 클라이언트 호환을 위해 기존 필드를 즉시 재해석하지 않고 새 Secret 응답을 연결한 뒤 화면을 단계적으로 전환한다. 100점 표시 제거 시 해당 필드의 사용처와 계약 테스트를 확인하고 버전 변경 또는 사용 중단을 기록한다. 실제 전환 대상 코드와 순서는 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)에 둔다.

### 통합 현황의 전환 경계

- 첫 연동은 `/api/v1/secrets`를 독립적으로 구현하고 기존 Overview v1의 필드 의미를 유지한다. 새 Secret 요약 카드에는 해당 범위와 단위를 표시한다.
- 새 통합 조회는 일반 탐지의 `detection_events` 한 행과 Secret의 현재 ResponseCase 한 행을 각각 위험 항목으로 합성한다. 항목의 영역, 원본 ID와 집계 단위를 명시하고 서로 다른 ID를 혼용하지 않는다. Secret 대응 건을 `detection_events`에 다시 적재하여 두 번 세지 않는다.
- 통합 심각도 분포는 포함된 위험 항목의 확인 등급과 미평가를 구분한다. Secret 부분은 현재 대응 건 필터·기준시점·스냅샷을 유지하며 고유 키·위치·Finding 수를 위험 항목 수에 합산하지 않는다. 기존 위험 추이의 날짜별 발생 이벤트는 현재 대응 건 수와 별도 지표다.
- 새 통합 응답은 별도 버전으로 제공한다. 정확한 URL, 필드, 일반 탐지의 포함·제외 조건과 두 출처의 기준시점 고정 방식은 통합 API 구현 전에 정의한다. 기존 API 필드를 새 의미로 바꾸거나 구현 완료로 표시하지 않는다.
- 새 화면과 통합 계약을 검증한 뒤 100점 카드를 등급 분포로 교체하고 구 클라이언트 지원 종료를 기록한다. 확장 전에는 기존 샘플 화면 표시를 유지한다.
