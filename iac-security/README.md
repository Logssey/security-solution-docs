# IaC 보안 설계문서

**Terraform 변경을 머지 전에 검사해서, 이 코드를 CI에서 돌려도 되는지(②)와 머지하면 민감 데이터까지 가는 공격 경로가 새로 생기는지(①)를 판정하고, 위험하면 차단.** State·드리프트 점검(③)은 두 기능에 데이터 등급과 현재 상태를 보태는 보조 기능. 결과는 PR과 통합 대시보드의 클라우드 보안 › IaC 보안 탭에 표시.

## 솔루션 목표

**IaC 공급망 공격 차단 + 공격 경로 사전 차단 (+ 드리프트 감시)**

| 기능 | 이름 (영문) | 무엇을 막나 | 언제 | 방식 |
|---|---|---|---|---|
| ② | 공급망·실행 코드 차단 (Pre-plan Gate) | 오염된 모듈, 악성 PR 코드, CI 공급망(Coder·Trivy형)의 실행과 자격 증명 탈취 | plan 전 | 차단 (CI 강제, 로컬은 래퍼) |
| ① | 공격 경로 사전 차단 (Attack Path Diff) | 인터넷에서 1등급 데이터(시크릿, DB, tfstate)까지 가는 새 공격 경로를 여는 변경 | 머지 전 | 1등급이면 병합 차단, 나머지는 경고 |
| ③ | 드리프트·State 감시 (Drift & State Monitoring) | 콘솔 수정·로컬 apply처럼 CI를 거치지 않은 변경, State 읽기 주체 증가, State 속 시크릿 | 매일 | 탐지 (대시보드에 표시) |

- **실행 전에 막는다**: Terraform은 plan만 해도 코드를 실행함. 2026년 사고 두 건(Coder, Trivy Action)이 여기서 털렸음. 그래서 자격 증명을 주기 전에 코드부터 검사
- **머지 전에 경로로 판정한다**: 규칙 도구는 설정 하나씩만 봄. 이 솔루션은 "이 PR이 시크릿까지 가는 새 길을 여는가"를 실제 자산 그래프와 IAM 시뮬레이터로 판정
- **길을 벗어나면 다음 날 잡는다**: 막을 수 없는 우회 변경은 드리프트와 State 점검으로 찾아냄
- **범위 경계**: 사람의 쓰기 권한을 없애는 것과 관리자 보호는 신원·권한 관리의 일. 이 솔루션은 그 전제 위에서 동작하고, 전제가 깨졌을 때는 탐지로 알림 ([위협 모델·범위](01-requirements/threat-model.md))

## 문서

| 문서 | 담는 내용 |
|---|---|
| [PRD](01-requirements/PRD.md) | 목적, 해결할 문제, 범위, 기능·비기능·보안 요구사항, 사용 시나리오 |
| [위협 모델·범위](01-requirements/threat-model.md) | 실제 사고(Coder, Trivy Action), 우리 환경의 노출, 강제 지점, 위협별 대응, 로컬 plan 방식, 한계, 범위 밖 전제 조건(자격 증명·권한) |
| [아키텍처](03-architecture/system-overview.md) | 구성 요소, 전체 구성도, PR·매일 데이터 흐름 |
| [ADR](03-architecture/adr.md) | 아키텍처 결정 12건과 근거 |
| [규칙 명세](02-functional-design/rule-spec.md) | 강도·심각도·데이터 등급, ②①③·설정 오류 규칙 전체, 운영 규칙 |
| [화면 설계](02-functional-design/dashboard-flow.md) | 심각도 표시, IaC 보안 탭, PR 상세 패널, PR 코멘트 |
| [API 명세](03-architecture/api-contract.md) | CI용·대시보드용·내부 작업용 API |
| [ERD](03-architecture/database-schema.md) | `iac_` 테이블과 관계, DDL 초안, 그래프 스키마 |

권장 열람 순서: PRD → 위협 모델·범위 → 규칙 명세 → 아키텍처 → ADR → 화면 설계 → API 명세 → ERD

## 구현 위치

**별도 서비스를 만들지 않고 기존 대시보드에 붙임.** Secret과 같은 구조 (ADR-11, Secret ADR-S008).

| 구성 | 위치 |
|---|---|
| API | [dashboard-backend](https://github.com/Logssey/dashboard-backend)의 `iac` 모듈, `/api/v1/iac`. 대시보드 세션 인증을 그대로 씀 |
| 분석 | API와 분리된 Java 워커. PostgreSQL 대기열 `iac_jobs`에서 작업을 가져감 |
| 결과 DB | 대시보드 PostgreSQL에 `iac_` 테이블 추가 (Flyway 후속 마이그레이션). `dashboard_users`·`assets`와 FK 연결 |
| 자산 그래프 | 별도 Neo4j (Cartography) |
| 외부 도구 | Cartography, KICS, Conftest, Terraform (실행 파일) |

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |
| 수정일 | 2026-10-13 |