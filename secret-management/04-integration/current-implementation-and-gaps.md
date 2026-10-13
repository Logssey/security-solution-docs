# 현재 구현과 Secret 기능 연동 차이

확인 기준: 2026-10-14, `dashboard-backend`의 `develop` `1a67be3`과 `dashboard-frontend`의 `develop` `d04627d`까지. 아래의 «구현»은 코드와 로컬 검증 범위를 뜻하며 모든 원천의 운영 지원을 뜻하지 않는다. 정식 목표 계약은 [데이터 계약](../03-architecture/data-contract.md), [DB 스키마](../03-architecture/database-schema.md), [API 계약](../03-architecture/api-contract.md)에 둔다.

## 1. 문서와 구현의 경계

[대시보드 설계 문서](https://github.com/Logssey/dashboard-design-docs)는 공통 기술 스택, 로그인, 공통 화면, 통합 현황과 공통 API를 관리한다. Secret의 제품 요구사항, 수집·탐지·검증, 키 모델, 평가·대응, 상세 화면과 전용 API는 이 문서 묶음에서 관리한다. 대시보드 문서에 남았던 단일 키 테이블, 임의 등급 판정, Admin PAT Mock 상세는 Secret의 정식 설계로 채택하지 않았다.

## 2. 현재 코드와 남은 연동

| 영역 | 구현 상태 | 남은 작업 |
|---|---|---|
| 실행 경계 | API는 `POST /api/v1/secrets/runs`로 작업을 등록하고 상태를 조회한다. 서버 시작 시 외부 스캔은 수행하지 않는다. 별도 Java 워커가 예약, 작업 선점과 GitHub 수집·탐지를 실행한다. | 기존 `/credentials/scan` 직접 Notion 실행 경로의 다른 사용처를 확인하고 제거하며 Notion 수집을 워커로 옮긴다. |
| 연결·범위·작업 | V3에 연결, 승인 범위, `FULL_SCAN` 작업, 실행, 실행 범위 5개 테이블이 있다. 6시간 기본 예약, 중복 방지, 리스·재시도 실행기가 있다. 예약기와 실행기는 기본 비활성이다. | 운영용 연결 등록·권한 관리와 원천별 처리기를 확장한다. `REVALIDATE` 작업은 아직 없다. |
| GitHub 수집 | 읽기 전용 GitHub App 설치 토큰으로 승인된 저장소·브랜치·경로의 현재 커밋 파일을 읽는다. `CURRENT_COMMIT_ONLY`를 지원하고 크기·총량·깊이 제한, 부분 실패를 기록한다. GitHub 파이프라인과 작업 실행기는 둘 다 명시적으로 켜야 한다. | 승인된 이력, Actions 설정 등 추가 콘텐츠는 별도 구현·검증한다. |
| 탐지·정규화 | 워커가 Gitleaks 8.30.1에 파일을 `stdin`으로 전달한다. V4에 도구 실행, 고유 키, 발견 위치, 도구 발견 기록이 있으며 완전한 키는 보호키 HMAC으로 묶고 불완전 후보는 위치별로 분리한다. | 지원 룰과 형식을 확장하고 실제 자료 기준 품질을 검증한다. |
| 유효성 검증 | GitHub 처리기가 발견한 AWS 키 후보를 V5 검증 시도 이력에 연결한다. 같은 완전 키는 실행 주기 내 한 번만 처리한다. STS 호출 설정은 기본 `false`이며 비활성은 `UNCHECKED/DISABLED`로 기록한다. 합성 실험 저장소는 설정과 관계없이 호출을 차단하고 `UNCHECKED/SYNTHETIC_FIXTURE`로 기록한다. 이때 실행의 검증 단계는 `SKIPPED`일 수 있다. | 승인된 비합성 테스트 자격 증명과 호출 정책으로 실제 STS 분기를 검증한다. 다른 발급자 유형의 검증기는 후속이다. |
| 조회 화면 | `/secrets` 화면이 새 연결 선택, 비동기 실행 등록·상태 조회, 고유 키 목록·상세를 사용한다. `GET /api/v1/secrets`, `/{id}`, `/connections`는 V4–V5 전용 기록을 읽는다. 기존 `/api/v1/credentials`는 전환 기간에 남아 있다. | 다른 호출자의 기존 API 사용처를 확인하고 제거한다. 요약·대응·스냅샷 조회는 후속이다. |
| 심각도·대응 | Secret의 미평가와 환경 `UNKNOWN`을 임의 등급이나 `PROD`로 바꾸지 않는다. | 권한·영향 근거, 평가 이력, 담당자·검토·체크리스트·외부 조치·재검증을 구현한다. |
| 통합 현황 | 기존 Overview는 일반 `detection_events` 조회와 100점 카드 계약을 유지한다. Secret의 원본 기록은 전용 테이블에 있다. | 통합 조회에서 두 출처를 중복 없이 합성하고 단위·미평가·누락 범위를 명시한다. |

GitHub App을 허용된 `Logssey/test-find-key`의 `fixtures/`에만 연결한 로컬 수집·탐지 실행에서는 파일 16개를 읽고 작업이 `SUCCEEDED`로 끝났다. 합성 표본에서 고유 키·후보 9개(완전 7, 불완전 2), 위치 15개, 도구 발견 23개가 기록됐다. 표본의 긴 합성 토큰 28개가 DB 데이터 덤프와 워커 로그에 원문으로 남지 않은 것도 확인했다. 검증 연결 이후에는 별도 테스트에서 합성 저장소의 STS 차단과 안전한 `UNCHECKED` 기록을 확인했다. 이는 테스트 범위의 결과이며 AWS 실수집이나 운영 데이터 유효성 검증 결과가 아니다.

소스 확인 위치: 백엔드의 `V3__create_secret_scheduler_tables.sql`부터 `V5__create_secret_validation_attempts.sql`, `SecretRunController.java`, `SecretInventoryController.java`, `secret-worker`의 `GitHubSecretJobProcessor.java`, `SecretFindingNormalizer.java`, `AwsCredentialValidator.java`; 프론트 `SecretsPage.tsx`, `src/api/secrets.ts`; 전환 대상은 기존 `CredentialListController.java`와 `NotionLeakScanService.java`다.

## 3. 현재 비동기 API와 기존 API

| 경로 | 현재 계약과 경계 |
|---|---|
| POST `/api/v1/secrets/runs` | 인증된 `ADMIN`이 활성 GitHub 연결의 `connectionId`와 선택적 `idempotencyKey`로 승인 범위의 전체 스캔을 등록한다. 다른 원천은 409로 거부한다. `202 Accepted`, `Location` 헤더와 `{ "runId", "status" }`를 반환한다. API는 원천을 읽거나 탐지를 실행하지 않는다. 같은 멱등 키의 재요청은 기존 실행을 반환한다. |
| GET `/api/v1/secrets/runs/{runId}` | `ADMIN`에게 전체·수집·탐지·검증 상태, 범위와 완료·누락 범위 수, 읽은 객체 수, 재시도 횟수, 시각, 허용된 실패 코드를 반환한다. 작업이 대기 중이면 `PENDING`, 리스 중이면 `RUNNING`으로 표시한다. |
| GET `/api/v1/secrets` | `ADMIN`에게 고유 키 목록을 `items`, `total`, 0부터 시작하는 `page`, `size`로 반환한다. `issuer`, `componentState` 필터와 1–100개 페이지 크기를 지원한다. 최신 검증 시도가 없으면 `UNCHECKED`, 심각도는 `UNASSESSED`다. |
| GET `/api/v1/secrets/{id}` | `ADMIN`에게 마스킹된 키 요약과 개별 발견 위치, 환경·노출·존재 상태를 반환한다. 보호키 지문과 원문은 제외한다. |
| GET `/api/v1/secrets/connections` | `ADMIN`에게 활성 GitHub 연결 중 활성 승인 범위가 있는 항목만 표시한다. 다른 원천의 수집기를 아직 실행할 수 없으므로 선택 목록에서 제외한다. |
| GET `/api/v1/credentials` | 전환 기간의 단일 `credentials` 조회다. 정식 Secret 고유 키·위치 모델과 집계 단위가 다르다. |
| POST `/api/v1/credentials/scan` | 전환 기간에 남은 API 프로세스의 동기 Notion 스캔이다. 새 비동기 작업의 성공이나 검증 결과로 해석하지 않는다. |

`ADMIN` 검사는 현재 임시 접근 경계다. 설계된 Secret 역할·연결 범위 권한과 감사는 아직 적용되지 않았다. 실행 API는 세션·CSRF를 사용하며 원문 자격 증명을 받지 않는다. 새 화면은 정식 경로를 사용하고 기존 API는 다른 사용처가 없어지면 제거한다. 대시보드의 공통 인증·Overview 계약은 [공통 API 명세](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)를 참조한다.

## 4. 원천별 범위와 다음 작업

- GitHub는 승인된 테스트 저장소의 현재 커밋과 파일 경로만 구현·확인했다. 워커가 없거나 관련 활성화 설정이 꺼져 있으면 등록된 작업이 자동으로 수집되지는 않는다.
- AWS는 연결할 계정의 리전·서비스·리소스와 읽기 권한 범위가 아직 정해지지 않았다. AWS **리소스 수집은 수행하지 않았다**. STS 검증은 GitHub에서 발견한 완전한 AWS 키 쌍에 대한 선택 기능이며 AWS 리소스 수집기나 권한·영향 평가가 아니다. 기본값은 꺼져 있고 합성 실험 저장소에는 설정과 관계없이 호출하지 않는다.
- Notion은 기존 API의 페이지 검색·Markdown 읽기·부분 실패 처리 코드가 있지만 워커의 승인 페이지 수집기로 아직 분리되지 않았다. 블록 전체 페이지 나눔과 지원 콘텐츠의 범위를 보완한다. Admin PAT 목록은 이번 작업에서 제외한다.
- K8s Secret 객체 수집과 공급자별 영향 평가, 대응 관리, 통합 현황은 후속 범위다.

공유 V2 `credentials`는 유지하고 Secret 테이블은 V3–V5로 추가했다. 기존 개발 DB의 V2 적용 이력을 파일명 변경으로 덮어쓰지 않는다. 별도 개발 DB에서 V1–V4와 GitHub 작업 실행을, 분리된 PostgreSQL 테스트에서 V5 적용과 제약을 확인했다. 원래 DB의 필요한 데이터 이전은 아직 수행하지 않았다. 자세한 경계는 [마이그레이션 절](../03-architecture/database-schema.md#6-보안-보존과-마이그레이션)을 따른다.
