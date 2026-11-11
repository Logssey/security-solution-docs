# 03. 아키텍처

Secret 노출 탐지·대응의 구성요소, 처리 경계, 데이터·API 계약과 보안 원칙을 정의한다. 기술 선택의 결정과 근거는 ADR에 둔다.

| 문서 | 정의하는 내용 |
|---|---|
| [전체 시스템 구성](system-overview.md) | 수집원, API, 스케줄러·워커, 탐지·검증·평가, 대응과 저장소의 책임 |
| [처리 흐름](processing-flow.md) | 수집, 위치별 결정, 외부 조치, 재검사와 해결 증거의 처리 순서 |
| [수집원과 검증기 계약](source-and-validator-contract.md) | 연결별 승인 범위, 콘텐츠·메타데이터 경계와 유형별 검증 결과 |
| [발견 키 유효성 검증](credential-validation.md) | 지원 등록, 원문 재취득, 공급자 응답·시각·한도와 검증 작업 |
| [심각도 평가와 규칙 저장](risk-assessment-contract.md) | 정규화 근거, 룰 테이블·버전, 평가 이력과 수동 조정 |
| [사용 이력과 사고 근거](usage-observation-contract.md) | AWS 관리 이벤트, 정확한 키 결합·조회 커버리지와 수동 자료 |
| [AI 조사 요약](risk-briefing-contract.md) | 기본 설명·선택 AI, 전송 정책·출력 검증·큐·예산과 적격성 |
| [데이터 계약](data-contract.md) | 키, 위치, 검사, 판정과 대응 기록의 관계 및 집계 기준 |
| [데이터베이스 스키마](database-schema.md) | Secret 전용 저장 모델, 제약, 인덱스와 스키마 변경 원칙 |
| [API 계약](api-contract.md) | 조회·변경 명령, 권한, 오류와 동시 변경 경계 |
| [보안과 실행 정책](security-and-runtime.md) | 읽기 권한, 원문 보호, 영속 작업과 운영 통제 |
| [ADR 목록](adr/README.md) | 채택한 기술 결정과 선택 근거 |

관련 기준: [PRD](../01-requirements/PRD.md), [기능 설계](../02-functional-design/README.md).

## 스캔 진행 읽기

[스캔 실행 계약](scan-execution-contract.md)은 실제 단계·구조화 이벤트·리스·cursor·보존과 단일 실행 패널의 갱신 경계를 정의한다.
