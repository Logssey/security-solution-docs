# 현재 구현과 Secret 기능 연동 차이

확인일: 2026-10-12, 프론트 `3602e57`, 백엔드 `5d2d1b1`, 대시보드 설계 문서 개정 전 `3bac546`, Secret 문서 개정 전 `acb9b1d`의 정적 확인 기준. 네 레포를 원격 `main`과 대조했으며 기준 커밋 차이는 없었다. 아래 문서 정리로 코드가 변경된 것은 아니며 빌드, 앱 실행, 외부 연결과 Mock 시험은 검증하지 않았다.

## 1. 기준 저장소

프론트는 React, TypeScript, Vite, 백엔드는 Java 21, Spring Boot, DB는 PostgreSQL, Flyway를 사용한다. 별도 `dashboard-frontend`, `dashboard-backend`, `dashboard-design-docs`와 이 문서 레포가 같은 개발 작업 폴더에 연결된 상태에서 정적으로 확인했다. 각 레포는 독립 저장소이며 대시보드 코드를 이 문서 레포에 복사하지 않는다. 감시 대상 Re:Used의 거래 기능은 본 솔루션 기능과 구분한다.

새 Secret 기능도 이 기술 기반을 따른다. [ADR-S001](../03-architecture/adr/ADR-S001.md)은 Gitleaks와 TruffleHog의 단독 또는 병행 운영을 비교 대상으로 두며, [ADR-S008](../03-architecture/adr/ADR-S008.md)은 Java API와 분리된 Java 워커를 실행 구조로 정한다. Python 서비스는 추가하지 않는다.

## 2. 확인된 구현과 추가 작업

| 영역 | 현재 코드에서 확인한 내용 | 요구사항 반영 시 필요한 변경 |
|---|---|---|
| 화면 라우트 | `/overview` 구현, `/secrets`는 ComingSoonPage | 시크릿 요약, 목록, 상세, 지정, 대응 흐름 구현 |
| 상세 검토 | DetailDrawer의 검토 완료는 준비 중 알림 | 저장 API, 권한, 감사 이력, 실패 처리 |
| 플랫폼 | 기존 Overview와 assets는 AWS, KUBERNETES, GITHUB | Secret 연결·발급자 타입에 NOTION 등 지원 유형 추가. 기존 assets에 가짜 Notion·PROD 자산을 만들지 않으며 타입 존재와 실제 수집기 구현을 구별 |
| 수집, 검증 | 확인한 코드에 외부 수집, 유효성 워커 없음 | AWS, K8s, GitHub, Notion 승인 범위의 6시간 주기 재수집과 탐지, 발견한 자격 증명의 유형별 검증, 실행 관리 |
| 담당자 | ownerTeam 문자열 | 개인 담당자 참조, 지정 가능 범위, 변경 이력 |
| 상태 | 심각도 4개, 검토 2개, 조치 4개 | 유효성, 재검증 대기, 심각도 미평가, 예외 세부 정책 |
| 환경 | ALL 조회 필터, 이벤트는 PROD/DEV 필수 | 환경 미확인 SaaS, 키의 표현. 임의 환경 지정 방지 |
| 최신성 | collectedAt이 서버 기동 시 생성되는 표본 시각 | 실행별 수집, 검사, 성공, 실패, 다음 예정 시각 |
| 점수 | `max(0, 100 - 4 × Critical - High)` | PRD 기준: 100점 표시를 Critical/High/Medium/Low 분포와 건수 중심으로 전환 |
| 집계 | COUNTED_EVENTS가 TRUE로 조치 상태 전체 포함 | 탐지 이력, 현재 대응, 만료폐기 신규사고 제외, 예외의 모수 정의 |
| 샘플 | local 프로파일의 LocalSampleDataLoader가 시작 시 assets와 detection_events를 전부 삭제 | Secret FK 연결 전 샘플 행만 교체하도록 변경. 샘플 표시와 실수집 연결을 분리하고 코드 존재로 운영 실데이터를 주장하지 않음 |
| 인증 | HttpSession, 세션 쿠키, CSRF, DB 역할 ADMIN | 담당자, 예외 승인, 연결 관리의 서버 권한 정책 추가 설계 |
| 오류 | 400·401·403·500 기본 JSON, 일부 404·405는 본문 없음 | 공통 code·message 형식 보완, Secret 409·429 등 추가. 선택 확장 필드는 하위 호환 유지 |

정적 확인 소스 위치: 프론트엔드 `src/App.tsx`, `src/api/types.ts`, `src/api/client.ts`, `src/components/DetailDrawer.tsx`; 백엔드 `OverviewService.java`, `OverviewQueryRepository.java`, `LocalSampleDataLoader.java`, `SecurityConfig.java`, `GlobalExceptionHandler.java`. 이후 코드가 변경되면 확인일과 기준 커밋, 구현·실행 검증 상태를 함께 갱신한다.

### 문서 정합성 정리

- 대시보드는 공통 구조와 기존 샘플 구현, Secret 문서는 상세 요구사항·상태·데이터·API를 관리한다.
- `credentials-draft.md`는 대체된 과거 초안이며 ADR-D009는 Secret ADR-S005로 대체한다. UNKNOWN 환경을 PROD로 처리하거나 만료·폐기만으로 신규 사고를 만드는 과거 규칙은 사용하지 않는다.
- ADR-D006의 네 상태와 ADR-D007의 준비 중 동작은 기존 샘플 계약이다. Secret 상세 상태·화면으로 확장하되 기존 API 필드 의미를 바꾸지 않는다.
- API 공통 명세의 예전 필드 변환표를 제거하고 오류 기본 필드와 선택 확장 필드를 통일한다. 문서 수정은 코드 수정이나 실행 검증 완료를 뜻하지 않는다.
- Notion Enterprise Admin API Mock은 후속 읽기 전용 목록 조회 검토이며 페이지·블록 수집 MVP와 구분한다. 목록 관리 안내를 Secret 등급이나 대응 건으로 변환하지 않는다.
- 로컬 개발은 기존 ADR-D004를 유지한다. 운영 저장 위치 ADR-D005는 Proposed로 남기고 승인된 실데이터 연결 전에 확정한다.

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

논리 관계, 위치별 조치와 집계, 원문 영속 미보관은 [데이터 계약](../03-architecture/data-contract.md)에 정의한다. 기존 백엔드 V1과 대시보드 설계 문서를 확인한 확장 테이블, 제약, 인덱스와 마이그레이션 순서는 [데이터베이스 스키마](../03-architecture/database-schema.md)에 정의한다. 기존 V1은 수정하지 않는다. 이 문서 정리 작업에서는 DB 마이그레이션이나 대시보드 코드를 변경하지 않았다.

## 5. 코드 연동 순서

1. [데이터베이스 스키마](../03-architecture/database-schema.md)에 따라 기존 V1을 유지한 후속 마이그레이션을 추가하고 빈 DB와 기존 DB에 적용한다. Secret FK 연동 전 샘플 적재기 삭제 범위를 보완한다.
2. 합성 표본으로 탐지·정규화·마스킹 결과와 최소 목록·상세 API를 연결하고 스프린트 1의 GitHub·AWS 수집·검증을 승인된 테스트 범위에서 시험한다. 실데이터 연결 전 ADR-D005의 저장 위치와 접근 권한을 확정한다.
3. 스프린트 2에서 담당자·역할, 검토·체크리스트·외부 조치 기록·수동 재검증·해결 조건을 저장·감사와 연결한다. AI 가이드 보조 없이 기본 흐름이 동작해야 한다.
4. 스프린트 3에서 Notion·K8s 수집과 지원 유형을 확대하고 6시간 반복 실행·재시도·중복 방지·실패 최신성을 검증한다.
5. 스프린트 4에서 영향 근거와 기존 severity-v2 규칙, 매트릭스·통합 API를 연결한다. 새 통합 계약·기준시점·하위 호환을 정의하고 클라이언트 전환 후 100점 표시를 제거한다.
6. 각 단계에서 실패·누락·원문 비노출을 확인하고 전체 출시 수용 기준은 모든 필수 기능과 운영 검증이 충족된 뒤 판단한다. 단계별 목표와 지원 범위는 [PRD](../01-requirements/PRD.md)의 제공 계획을 따른다.

성공 기준은 API를 호출했다는 사실이 아니라 저장, 새로고침, 실패 복구 후에도 근거, 상태, 집계가 일치하는 것이다.
