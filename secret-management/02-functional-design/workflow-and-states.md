# 상태와 집계 규칙

기준: [PRD](../01-requirements/PRD.md). 수집부터 대응까지의 분기는 [처리 흐름](../03-architecture/processing-flow.md)에 설명한다.

## 1. 용어와 처리 단위

| 단위 | 뜻 |
|---|---|
| Source / Connection | 수집 위치와 연결 범위. GitHub 저장소, Notion 페이지, 블록, AWS 리소스, K8s 네임스페이스 등 |
| CredentialSet / key set | 함께 인증에 쓰이는 구성요소 묶음. AWS ID+Secret(+Session Token) 등. `METADATA_ONLY`는 발급처 목록에서 존재만 확인한 별도 구성 상태이고 원본 비밀 값이 발견됐다는 뜻이 아님 |
| Occurrence | 키가 발견된 위치 한 건 또는 발급처 목록의 메타데이터 관측 위치. 동일 키가 여러 곳에 있으면 여러 위치를 보존하고 목록 관측을 비정상 노출로 간주하지 않음 |
| Finding | 유형, 위치, 엔진, 룰, 탐지 근거를 연결한 결과 |
| ValidationAttempt | 특정 시각에 특정 검증기가 수행한 검사 결과 |
| Assessment | 영향 범위와 심각도 판정, 근거 및 정책 버전 |
| ResponseCase | 한 CredentialSet의 비정상 노출을 관리하는 대응 단위. 생성, 해결과 재발 조건은 [대응 정책](response-policy.md)을 따름 |

이 이름은 논리 모델의 단위다. 물리 테이블과 API 필드는 [데이터 계약](../03-architecture/data-contract.md)과 [API 계약](../03-architecture/api-contract.md)에 정의한다. **발견 원천과 키 발급자를 구분**한다. Notion에서 AWS 키를 찾았다면 source=NOTION, credentialProvider=AWS이다.

## 2. 단위 간 관계

하나의 key set에 여러 발견 위치와 탐지 결과가 연결될 수 있다. 검증 시도, 영향 평가와 대응 이력은 시점별로 보존한다. 수집 위치와 키 발급자는 별도 속성이다. 상세 수집, 탐지, 검증 순서는 [처리 흐름](../03-architecture/processing-flow.md)을 따른다.

## 3. 상태 축

아래 값은 제품의 논리 상태다. 저장 및 응답의 표현은 API 계약과 일치시킨다.

| 축 | 값 | 의미 |
|---|---|---|
| 유효성 | VALID / INVALID / UNKNOWN / UNCHECKED | 유효 / 무효 확인 / 확인 불가 / 미검증 |
| 심각도 | CRITICAL / HIGH / MEDIUM / LOW, 또는 미평가 | [심각도 정책](severity-policy.md)에 따른 조치 우선순위. Secret 목록은 현재 `UNASSESSED`를 반환하지만 평가 이력과 정책 실행은 후속 구현 |
| 검토 | UNREVIEWED / REVIEWED | 담당자가 근거를 검토했는지. 기존 계약 승계 |
| 조치 | OPEN / IN_PROGRESS / REVALIDATION_PENDING / RESOLVED | 신규 및 분석 / 조치 중 / 재검증 대기 / 해결 |
| 예외 | RISK_ACCEPTED | 승인된 위험 수용. 네 막대와 배타적으로 집계하며 오탐, 대상 제외와 구별 |
| 최신성 | 마지막 판정 시각, 최근 시도, 조치 이후 미검증, 지연 | 키의 유효성과 독립적으로 결과가 오래됐는지 표현 |

### 유효성 사유

사유는 `EXPIRED_CONFIRMED`, `REVOKED_CONFIRMED`, `INACTIVE_CONFIRMED`, `INVALID_UNSPECIFIED`, `UNSUPPORTED_TYPE`, `INCOMPLETE_CREDENTIAL`, `SOURCE_UNAVAILABLE`, `PERMISSION_LIMITED`, `RATE_LIMITED`, `NETWORK_ERROR`, `PROVIDER_ERROR`, `VALIDATOR_AUTH_ERROR`로 분류한다. 공급자 응답 근거가 없으면 만료와 폐기를 추측하지 않는다.

- 미검증: 아직 검사를 수행하지 않은 상태.
- 확인 불가: 시도했으나 현재 결론을 낼 수 없는 상태.
- 무효 확인: 해당 검증 방식으로 인증 불가가 확인된 상태. 구체적 이유가 만료, 폐기인지는 별도 근거가 필요하다.
- TruffleHog `unverified`를 무효로 직접 매핑하지 않는다.

### 사고 생성 및 보존

- 만료, 폐기 확인: 이력 보존, 신규 사고 제외.
- 검증 불가: 사유 표시와 수동 검토. 비정상 노출 근거가 있으면 대응 건을 생성한다.
- 유효한 비정상 노출: 대응 건을 생성하고 담당자를 지정한다.
- 안전한 저장소의 정상 키 목록을 읽은 것만으로 사고를 생성하지 않는다.
- AWS IAM 목록의 `METADATA_ONLY` 항목은 노출·유효성·권한을 확인한 결과가 아니므로 그 사실만으로 사고나 심각도를 만들지 않는다.
- 기존 사고가 이후 무효로 바뀌어도 과거 악용, 잔여 노출까지 해결됐다고 가정하지 않는다. 종결 조건은 [대응 정책](response-policy.md)을 따른다.

## 4. 4단계 조치 현황

조치 상태는 **신규 및 분석 / 조치 중 / 재검증 대기 / 해결**로 표시하고 예외는 별도로 집계한다.

전이 조건과 일부 위치만 조치한 경우는 [대응 정책](response-policy.md)을 따른다. 키 변경은 담당자가 외부 시스템에서 수행하며 체크리스트 완료나 재검증 요청만으로 해결되지 않는다. 기존 Overview의 상태는 정식 Secret 상태로 일괄 변환하지 않으며 현재 계약과 남은 연동은 [구현 차이](../04-integration/current-implementation-and-gaps.md)에 기록한다.

## 5. 집계 규칙

- 조치 막대, 매트릭스, 미지정 목록의 기본 단위는 **각 CredentialSet의 최신 운영 ResponseCase 한 건**이다. 별도 `TEST` 시뮬레이션 범위의 건은 운영 집계에 넣지 않으며, 시험 조회에서만 같은 기준으로 한 건씩 센다. 같은 키의 여러 엔진 결과와 노출 위치는 상세에서 보여준다.
- 같은 범위의 네 막대 합계 + 예외 수 = 현재 대응 건 목록 수다. 정상 인벤토리와 대응 건이 없는 만료, 폐기 이력은 이 모수에서 제외한다.
- 같은 키에 정상 보관 위치와 비정상 노출 위치가 함께 있을 수 있다. 위치별 노출, 조치 근거를 보존하고 한 위치의 삭제나 예외로 전체 키를 해결, 예외 처리하지 않는다.
- 원문 미보관, 재취득 실패로 재검증할 수 없으면 확인 불가 사유를 남긴다. 키가 무효이거나 해결됐다는 근거로 사용하지 않는다.
- image 72의 탐지, 분류, 우선순위, 검토 완료는 **중복 가능한 진행 지표**다. 네 수의 합을 전체로 취급하지 않는다.
- image 74의 심각도×유효성 셀은 기본 운영 범위에서 같은 기준시점의 현재 대응 건을 센다. 이 범위에는 CredentialSet당 한 건만 존재한다. 시험 건은 명시적인 시험 범위에서만 따로 센다. 미평가 심각도는 별도 수로 표시한다.
- 무효, 만료, 폐기 이력 조회와 현재 사고 집계를 분리한다. 이력 조회에 무효 키가 보인다고 신규 사고로 집계하지 않는다.
- 필터: 환경, 발견 소스, 키 발급자, 유형, 유효성, 심각도, 조치 단계, 담당자. 기존 PROD/DEV로 식별할 수 없는 소스는 임의 PROD로 넣지 않는다.

## 6. 검사 이후 조치가 일어난 경우

예시 이미지 71은 06:00 유효 판정 이후 09:05 비활성화 체크가 있다. 두 기록은 시점이 다르다. 마지막 확인 결과와 실제 조치 시각을 함께 보여주고 ‘조치 이후 재검증 대기’를 표시한다. 재검증이 실패하면 과거 성공 판정을 덮어쓰지 않고 최신 시도의 실패 사유를 추가한다.

관련: [ADR-S002](../03-architecture/adr/ADR-S002.md), [ADR-S003](../03-architecture/adr/ADR-S003.md), [검증 조사](../05-research/verification-and-risk.md).
