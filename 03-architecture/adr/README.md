# 아키텍처 결정 기록

개별 ADR은 하나의 설계 선택을 맥락, 결정, 근거, 대안, 결과 순서로 기록한다. 전체 구성과 처리 흐름은 [아키텍처 설계 문서](../README.md)에 둔다.

## 상태

| 상태 | 의미 |
|---|---|
| Accepted | PRD를 충족하는 현재 설계 기준 |
| Proposed | 대안 비교나 검증이 남아 있는 선택 |
| Superseded | 후속 결정으로 대체된 기록 |

한 문서에 확정된 경계와 미결정 구현 세부사항이 함께 있으면 아래 표에서 구분한다. PRD의 제품 요구사항은 ADR의 선택지보다 우선한다.

## 결정 목록

| ID | 결정 | 상태 |
|---|---|---|
| [ADR-S001](ADR-S001.md) | 룰 기반 탐지와 지원 범위 | Gitleaks 초기 탐지 엔진 Accepted |
| [ADR-S002](ADR-S002.md) | 6시간 수집, 탐지와 유형별 유효성 재검사 | Java 수집 및 검증 워커와 영속 작업 대기열 Accepted |
| [ADR-S003](ADR-S003.md) | 독립 상태 축과 조치 단계 | 대응 단위와 배타적 집계 Accepted |
| [ADR-S004](ADR-S004.md) | 수동 대응과 외부 키 변경 경계 | Accepted |
| [ADR-S005](ADR-S005.md) | 검증용 원문 보관 | 원문 영속 미보관 Accepted |
| [ADR-S006](ADR-S006.md) | 소스별 수집 경계와 K8s | Java 수집 어댑터와 배치 조건 Accepted |
| [ADR-S007](ADR-S007.md) | 등급 기반 위험 판단 | 네 등급, 미평가와 순서가 정해진 판정 규칙 Accepted |
| [ADR-S008](ADR-S008.md) | 대시보드와 논리 데이터 경계 | Java API, 워커 분리와 PostgreSQL 경계 Accepted |

## 후속 검토

실제 연결 범위와 권한, 공급자별 허용 엔드포인트와 호출 한도, 워커가 접근할 수 있는 K8s 네트워크, 실행 파일 버전은 배포 환경과 시험으로 확인한다. 선택 기준은 해당 ADR에 기록한다. 확정된 계약은 [데이터](../data-contract.md), [API](../api-contract.md), [보안 실행 정책](../security-and-runtime.md)을 따른다.
