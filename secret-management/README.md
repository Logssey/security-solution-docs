# Secret 관리 솔루션 설계 문서

승인된 개발 및 운영 환경에서 자격 증명의 노출 가능성을 탐지하고 유효성, 영향 범위와 대응 상태를 관리하는 제품의 요구사항과 설계를 정리한다. 제품 범위는 [PRD](01-requirements/PRD.md)를 기준으로 한다.

## 대시보드 문서와의 역할

[대시보드 설계 문서](https://github.com/Logssey/dashboard-design-docs)는 공통 기술 스택, 세션 로그인, 공통 레이아웃, 통합 현황과 API 공통 규약을 관리한다. 이 문서 묶음은 Secret의 요구사항, 수집·탐지·검증, 원문 처리, 상태, 상세 화면, 데이터와 API 계약을 관리한다. Secret 상세 설계를 대시보드 문서에 중복 정의하지 않는다.

2026-10-13 문서 분리에서는 기존 Secret 설계와 일치하는 새 연동 정보만 반영했다. 단일 키 테이블 초안, 임의 위험 판정과 Notion Admin PAT Mock 상세는 정식 설계로 옮기지 않았다. 현재 `/credentials` 구현과 전환 계획은 [구현 연동 현황](04-integration/current-implementation-and-gaps.md)에 구분해 기록한다.

현재 구현과 개발 목표는 [구현 차이](04-integration/current-implementation-and-gaps.md)에서 구분한다. 개발용 DB는 기존 ADR-D004를 유지하고 운영 저장 위치는 ADR-D005의 별도 미결정 사항이다. 단계별 제공 범위는 PRD에 기록하며 전체 필수 요구사항을 첫 MVP의 완료 주장으로 바꾸지 않는다.

2026-10-19의 근거 중심 UX 목표는 [후보 판정과 조사 대기열](02-functional-design/triage-policy.md), [대응 정책](02-functional-design/response-policy.md), [스캔 이력 화면 정책](02-functional-design/scan-history-policy.md)에 정리했다. 설계 선택은 [ADR-S010](03-architecture/adr/ADR-S010.md), 구현과의 차이는 [현재 구현과 차이](04-integration/current-implementation-and-gaps.md)를 따른다. 이 목표는 아직 코드에 반영됐다는 뜻이 아니다.

## 현재 기준 문서

| 영역 | 문서 | 내용 |
|---|---|---|
| 요구사항 | [PRD](01-requirements/PRD.md) | 제품 목적, 적용 범위, 사용자 시나리오, 기능과 보호 요건 |
| 기능 설계 | [기능 설계](02-functional-design/README.md) | 상태, 집계 규칙과 대시보드 동작 |
| 아키텍처 | [아키텍처](03-architecture/README.md) | 전체 구성, 처리 흐름과 결정별 ADR |

## 권장 열람 순서

1. [PRD](01-requirements/PRD.md)에서 제품 범위와 수용 기준을 확인한다.
2. [기능 설계](02-functional-design/README.md)에서 상태, 집계와 화면 흐름을 확인한다.
3. [아키텍처](03-architecture/README.md)에서 전체 구성과 처리 흐름을 본 뒤 필요한 [ADR](03-architecture/adr/README.md)을 읽는다.

## 근거와 작업 문서

| 영역 | 문서 | 용도 |
|---|---|---|
| 구현 연동 | [현재 구현과 차이](04-integration/README.md) | 기존 대시보드 코드의 확인 상태와 추가 작업 |
| 조사 근거 | [키 유형과 검증 조사](05-research/README.md) | 유형, 수집 경계, 공급자 검증 방법과 한계 |

설계 이미지는 해당 문서 본문에서 설명과 함께 보여준다. [이미지 목록](assets/README.md)은 파일을 찾는 용도다.
