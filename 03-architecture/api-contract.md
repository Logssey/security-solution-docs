# 대시보드 API 계약

기준: [PRD](../01-requirements/PRD.md), [ADR-S008](adr/ADR-S008.md), [데이터 계약](data-contract.md), [대응 정책](../02-functional-design/response-policy.md)

이 문서는 새 Secret 기능의 구현 계약이다. 아래 경로는 현재 코드에 존재한다고 주장하지 않는다. 기존 `/api/v1/overview`의 실제 응답과 호환성은 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)를 따라 점검한다.

## 1. 공통 계약

모든 조회와 변경은 기존 세션 인증 경계를 통과한다. 변경 요청은 CSRF 검증과 서버 권한 검사를 적용한다. 공급자 연결 자격 증명과 발견된 키 원문은 요청, 응답, 오류, 로그에 넣지 않는다. 응답 필드는 camelCase, enum은 대문자, 시각은 UTC ISO 8601 형식을 사용한다.

조회는 `source`, `provider`, `environment`, `credentialType`, `validity`, `severity`, `actionStatus`, `exception`, `assignee`, 기간 필터 중 해당 경로에 맞는 값을 받는다. 응답은 적용된 `filters`, `scope`, `unit`, `asOf`, `snapshotId`, `total`, `partialCoverage`를 반환한다. 요약, 매트릭스와 기본 대응 목록은 `scope=currentCases`, `unit=ResponseCase`이고 각 키의 최신 건만 센다. 인벤토리는 `scope=inventory`, `unit=CredentialSet`이다. 이력 목록은 `scope=history`, `unit=ResponseCase`로 과거 건을 각각 센다. 화면은 첫 조회의 `snapshotId`를 매트릭스와 목록에 전달한다. 같은 스냅샷과 필터의 카드, 매트릭스, 목록 건수는 일치해야 한다. 만료된 스냅샷은 409와 새 조회 안내를 반환한다. 페이지네이션은 안정된 정렬 키와 다음 커서를 사용한다.

오류 형식은 `code`, `message`, `requestId`, `retryable`, `details`다. `details`에는 키 원문이나 원천 콘텐츠를 넣지 않는다. 입력 오류는 400, 세션 없음은 401, 권한 없음은 403, 대상 없음은 404, 버전 충돌은 409, 호출 제한은 429, 서버 장애는 5xx로 구분한다. 공급자 검사 결과의 401, 403, 404 등은 대시보드 HTTP 오류와 별도로 ValidationAttempt의 사유로 기록한다.

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

기존 `/overview` 응답의 숫자 점수는 새 Secret 요약의 근거가 아니다. 기존 클라이언트 호환을 위해 기존 필드를 즉시 재해석하지 않고 새 Secret 응답을 연결한 뒤 화면을 단계적으로 전환한다. 100점 표시 제거 시 해당 필드의 사용처와 계약 테스트를 확인하고 버전 변경 또는 사용 중단을 기록한다. 실제 전환 대상 코드와 순서는 [구현 연동 문서](../04-integration/current-implementation-and-gaps.md)에 둔다.
