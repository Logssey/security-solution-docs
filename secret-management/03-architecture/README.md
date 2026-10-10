# 03. 아키텍처

PRD의 수집, 검증, 평가, 대응 요구사항을 구성요소와 처리 경계로 구체화한다. 그림과 단계별 설명은 설계 문서에, 선택의 근거와 대안은 개별 ADR에 둔다.

## 설계 문서

| 문서 | 내용 |
|---|---|
| [전체 시스템 구성](system-overview.md) | 수집원, 스케줄러, 탐지, 검증, 영향 평가, 대응, 데이터 및 대시보드의 책임 |
| [처리 흐름](processing-flow.md) | 수집부터 판정, 대응, 재검증과 감사 이력까지의 분기 |
| [수집원과 검증기 계약](source-and-validator-contract.md) | 네 연결의 수집 범위와 유형별 검증, 실패 처리 |
| [데이터 계약](data-contract.md) | 키, 발견 위치, 검증, 대응 기록의 관계와 집계 기준 |
| [데이터베이스 스키마](database-schema.md) | 기존 대시보드 DB와의 경계, Secret 테이블, 제약, 인덱스와 마이그레이션 기준 |
| [API 계약](api-contract.md) | 대시보드 조회, 변경, 오류와 권한 경계 |
| [보안과 실행 정책](security-and-runtime.md) | 수집 권한, 검증 작업, 원문 보호와 운영 기록 |
| [ADR 목록](adr/README.md) | 기술 결정의 상태, 근거, 대안과 남은 선택 |

## 관련 문서

- [PRD](../01-requirements/PRD.md): 제품 범위와 보호 요건의 기준
- [기능 설계](../02-functional-design/README.md): 상태, 집계, 화면 동작
- [구현 차이](../04-integration/current-implementation-and-gaps.md): 현재 코드와 설계 사이의 차이
