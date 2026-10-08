# 현재 구현과 Secret 기능 연동 차이

확인일: 2026-10-08, 로컬 소스 정적 확인 기준. 앱 실행, 외부 연결 성공 여부는 이번 작업에서 검증하지 않음.

## 1. 기준 저장소

프론트는 React, TypeScript, Vite, 백엔드는 Java 21, Spring Boot, DB는 PostgreSQL, Flyway를 사용한다. 이 확인은 별도 대시보드 코드와 설계 문서를 대상으로 한 정적 확인이며, 해당 저장소는 이 문서 묶음에 포함되지 않는다. 감시 대상 Re:Used의 거래 기능은 본 솔루션 기능과 구분한다.

새 Secret 기능도 이 기술 기반을 따른다. [ADR-S001](../03-architecture/adr/ADR-S001.md)은 Gitleaks CLI를 탐지 엔진으로, [ADR-S008](../03-architecture/adr/ADR-S008.md)은 Java API와 분리된 Java 워커를 실행 구조로 정한다. Python 서비스는 추가하지 않는다.

## 2. 확인된 구현과 추가 작업

| 영역 | 현재 코드에서 확인한 내용 | 요구사항 반영 시 필요한 변경 |
|---|---|---|
| 화면 라우트 | `/overview` 구현, `/secrets`는 ComingSoonPage | 시크릿 요약, 목록, 상세, 지정, 대응 흐름 구현 |
| 상세 검토 | DetailDrawer의 검토 완료는 준비 중 알림 | 저장 API, 권한, 감사 이력, 실패 처리 |
| 플랫폼 | AWS, KUBERNETES, GITHUB | NOTION 추가. 타입 존재와 실제 수집기 구현을 구별 |
| 수집, 검증 | 확인한 코드에 외부 수집, 유효성 워커 없음 | AWS, K8s, GitHub, Notion 수집, 유형별 검증, 6시간 실행 관리 |
| 담당자 | ownerTeam 문자열 | 개인 담당자 참조, 지정 가능 범위, 변경 이력 |
| 상태 | 심각도 4개, 검토 2개, 조치 4개 | 유효성, 재검증 대기, 심각도 미평가, 예외 세부 정책 |
| 환경 | ALL 조회 필터, 이벤트는 PROD/DEV 필수 | 환경 미확인 SaaS, 키의 표현. 임의 환경 지정 방지 |
| 최신성 | collectedAt이 서버 기동 시 생성되는 표본 시각 | 실행별 수집, 검사, 성공, 실패, 다음 예정 시각 |
| 점수 | `max(0, 100 - 4 × Critical - High)` | PRD 기준: 100점 표시를 Critical/High/Medium/Low 분포와 건수 중심으로 전환 |
| 집계 | COUNTED_EVENTS가 TRUE로 조치 상태 전체 포함 | 탐지 이력, 현재 대응, 만료폐기 신규사고 제외, 예외의 모수 정의 |
| 샘플 | local 프로파일의 LocalSampleDataLoader | 샘플 표시와 실수집 연결 분리. 코드 존재로 운영 실데이터를 주장하지 않음 |
| 인증 | HttpSession, 세션 쿠키, CSRF, DB 역할 ADMIN | 담당자, 예외 승인, 연결 관리의 서버 권한 정책 추가 설계 |

정적 확인 당시의 소스 위치: 프론트엔드 `src/App.tsx`, `src/api/types.ts`, `src/components/DetailDrawer.tsx`; 백엔드 `OverviewService.java`, `OverviewQueryRepository.java`, `LocalSampleDataLoader.java`. 원본 저장소가 이 묶음에 포함되지 않으므로 구현 전 해당 파일과 계약을 다시 확인한다.

### 조치 상태 매핑

| 기존 actionStatus | 새 화면 단계 | 필요한 처리 |
|---|---|---|
| OPEN | 신규 및 분석 | 신규와 분석을 하나의 단계로 표시 |
| IN_PROGRESS | 조치 중 | 기존 상태 유지 |
| 없음 | 재검증 대기 | 조치 이후 재검증 대기를 별도 상태로 추가 |
| RESOLVED | 해결 | 해결 근거와 시각을 함께 저장 |
| RISK_ACCEPTED | 예외 | 위험 수용에 한해 매핑하고 다른 예외와 구분 |

## 3. 기존 API 계약

현재 Overview는 다음 조회 API를 호출한다. 환경 필터는 ALL/PROD/DEV다.

| Method | 경로 | 역할 |
|---|---|---|
| GET | `/api/v1/overview/summary` | 통합 요약 |
| GET | `/api/v1/overview/risk-trend` | 추이 |
| GET | `/api/v1/overview/assets` | 자산 요약 |
| GET | `/api/v1/overview/priority-findings` | 우선 확인 결과 |

세션 인증과 CSRF, camelCase, 대문자 enum, UTC 시각을 연동 기준으로 삼는다. 구체 계약은 별도 백엔드의 `OverviewController.java`, `SecurityConfig.java`와 프론트엔드의 `src/api/client.ts`에서 확인한다. 기존 API 설계 문서의 설명과 실제 타입이 다르면 계약 개정 시 함께 정합성을 맞춘다.

아래는 **신규 기능 계약의 요약**이며 현재 코드에 존재하는 API가 아니다. 경로, 공통 필터, 권한, 오류와 동시 변경 조건은 [API 계약](../03-architecture/api-contract.md)에 정의한다.

| 기능 | 요청, 응답에 필요한 정보 |
|---|---|
| 탐지 요약, 목록 | 필터, 집계 단위, 기준시점, 페이지 정보, 마스킹 결과, 수집 누락 |
| key set 상세 | 유형, 발급자, 여러 발견 위치, 탐지 근거, 검증, 평가 이력 |
| 검토, 담당자, 상태 변경 | 대상, 변경값, 수행자 권한, 사유, 시각, 동시 변경 조건 |
| 조치 체크리스트 | 가이드 버전, 항목 완료 기록, 증거, 실제 외부 조치 시각 |
| 재검증 요청 | 실행 ID, 대기/진행/결과, 중복 요청 제어. 키 원문은 브라우저에 반환하지 않음 |
| 연결, 실행 상태 | 허용 범위, 마지막 성공, 최근 시도, 다음 예정, 실패, 지연 |

## 4. 데이터 모델 차이

정적 확인 당시 백엔드의 `V1__create_tables.sql`은 dashboard_users, assets, detection_events 세 테이블을 정의했다. detection_events에는 선택적 asset 연결, 필수 environment, owner_team, severity, 검토, 조치 상태가 있다. 자격 증명 구성요소, 발견 위치, 검증 이력, 개인 담당자, 변경 이력은 없다.

| 논리 모델 | 목적 |
|---|---|
| Connection / ScanRun | 연결 대상, 권한 범위와 실행, 실패, 최신성 추적 |
| CredentialSet / Occurrence | 같은 인증 수단과 여러 발견 위치를 연결, 발급자와 소스 분리 |
| ValidationAttempt | 시점별 유효성, 사유, 방법, 관측 주체 기록 |
| Assessment | 영향 근거, 심각도, 정책 버전 |
| ResponseCase / AssignmentHistory | 검토, 조치, 담당자, 체크리스트, 예외 이력 |

논리 관계, 위치별 조치와 집계, 원문 영속 미보관은 [데이터 계약](../03-architecture/data-contract.md)에 정의한다. 실제 테이블과 인덱스는 백엔드 코드 확인 후 새 Flyway 버전으로 추가한다. 기존 V1은 수정하지 않는다. 이 문서 저장소에는 대시보드 코드가 없어 DB 변경 자체는 아직 수행할 수 없다.

## 5. 코드 연동 순서

1. 현재 코드 버전과 기존 API, DB 계약을 다시 확인하고 [새 데이터 계약](../03-architecture/data-contract.md)의 마이그레이션을 추가한다.
2. 승인된 연결 범위와 최소 권한을 설정하고 네 수집기, 탐지와 유형별 검증 작업을 연결한다.
3. [새 API 계약](../03-architecture/api-contract.md)을 구현하고 기존 100점 표시의 전환과 호환 범위를 확인한다.
4. 합성 결과로 `/secrets`의 목록, 상세, 검토, 지정, 조치 기록과 재검증 흐름을 연결한다.
5. 승인된 테스트 범위에서 실제 수집과 검증을 시험하고 실패, 누락, 최신성, 원문 비노출을 확인한다.

성공 기준은 API를 호출했다는 사실이 아니라 저장, 새로고침, 실패 복구 후에도 근거, 상태, 집계가 일치하는 것이다.
