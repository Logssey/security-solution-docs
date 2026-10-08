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
| [ADR-S001](ADR-S001.md) | 룰 기반 탐지와 지원 범위 | 탐지 방식 Accepted, 엔진 구성 Proposed |
| [ADR-S002](ADR-S002.md) | 유형별 유효성 검증과 6시간 재검사 | 검사 계약 Accepted, 재시도 수치 Proposed |
| [ADR-S003](ADR-S003.md) | 독립 상태 축과 조치 단계 | 상태와 배타적 집계 Accepted, 물리 모델 Proposed |
| [ADR-S004](ADR-S004.md) | 수동 대응과 외부 키 변경 경계 | Accepted |
| [ADR-S005](ADR-S005.md) | 검증용 원문 보관 | Proposed, 보관 방식 결정 대기 |
| [ADR-S006](ADR-S006.md) | 소스별 수집 경계와 K8s | 수집 범위 Accepted, 배포 방식 Proposed |
| [ADR-S007](ADR-S007.md) | 등급 기반 위험 판단 | 네 등급과 미평가 Accepted, 산정 규칙 Proposed |
| [ADR-S008](ADR-S008.md) | 대시보드와 논리 데이터 경계 | 논리 경계 Accepted, 물리 배치 Proposed |

## 후속 검토

원문 보관 방식, K8s 수집기 배포 위치, 대응 단위와 대표 상태, 등급 산정 규칙, API 필드 및 물리 저장소 배치를 구체화해야 한다. 각 항목의 선택 기준은 해당 ADR과 [검증 계획](../../06-planning/open-decisions-and-validation.md)에 기록한다.
