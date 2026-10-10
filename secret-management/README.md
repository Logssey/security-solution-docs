# Secret 관리 솔루션 설계 문서

승인된 개발 및 운영 환경에서 자격 증명의 노출 가능성을 탐지하고 유효성, 영향 범위와 대응 상태를 관리하는 제품의 요구사항과 설계를 정리한다. 제품 범위는 [PRD](01-requirements/PRD.md)를 기준으로 한다.

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
