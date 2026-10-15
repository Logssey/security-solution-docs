# Secret 데이터베이스 스키마

기준: [PRD](../01-requirements/PRD.md), [데이터 계약](data-contract.md), [API 계약](api-contract.md), [심각도 정책](../02-functional-design/severity-policy.md), [대응 정책](../02-functional-design/response-policy.md), [ADR-S002](adr/ADR-S002.md), [ADR-S003](adr/ADR-S003.md), [ADR-S005](adr/ADR-S005.md), [ADR-S008](adr/ADR-S008.md).

이 문서는 PostgreSQL의 Secret 물리 스키마와 후속 확장 목표를 구분한다. 2026-10-15 확인 기준은 [dashboard-backend](https://github.com/Logssey/dashboard-backend)의 `develop`에 병합된 V1–V8이다. V1의 `dashboard_users`, `assets`, `detection_events`와 공유 V2의 기존 단일 `credentials`는 유지한다. V3은 연결·작업, V4는 키·위치·탐지, V5는 검증 시도, V6은 AWS IAM 접근키 메타데이터 관측을 추가했다. Secrets Manager·SSM·S3의 개수 수집과 정확한 AWS 단일 값 수집 경로는 기존 V3–V5를 사용한다. V7은 대응 건·담당자·권한의 기초를, V8은 수동 범위 분류·노출 판정의 버전과 이력을 추가한다. 심각도 평가·검토·체크리스트·외부 조치·재검증·집계 스냅샷은 아직 목표다. 현재 상태와 전환 차이는 [구현 연동 현황](../04-integration/current-implementation-and-gaps.md)에 기록한다.

| 마이그레이션 | 현재 구현된 테이블 | 아직 포함하지 않는 범위 |
|---|---|---|
| V3 | `secret_connections`, `secret_connection_scopes`, `secret_jobs`, `secret_scan_runs`, `secret_run_scopes` | `REVALIDATE` 작업, 역할·담당자 |
| V4 | `secret_credential_sets`, `secret_occurrences`, `secret_tool_runs`, `secret_findings` | 평가·대응·사용 근거 |
| V5 | `secret_validation_attempts` | 공급자별 영향 평가와 수동 재검증 API |
| V6 | `secret_aws_access_key_observations`, IAM 메타데이터 식별 고유 인덱스 | IAM 비밀 키 원문, 유효성·노출·권한·심각도 평가, Secrets Manager·SSM·S3 목록의 자원별 개별 기록 |
| V7 | `secret_response_cases`, `secret_case_occurrences`, `secret_assignment_history`, `secret_audit_events`, `secret_role_grants`, `secret_team_memberships`; 승인 범위의 대응 구분 | 기존 발견의 자동 대응 건 생성, 노출 판정 API·근거 이력, 검토·체크리스트·외부 조치·재검증, 평가·스냅샷 |
| V8 | 범위의 `response_origin_version`, 위치의 `exposure_assessment_version`, `secret_scope_origin_decisions`, `secret_exposure_assessments` | 기존 범위의 자동 운영·시험 분류나 기존 발견의 자동 노출 판정·대응 건 생성 없음 |

로컬 개발 DB는 기존 대시보드 ADR-D004를 사용하며 별도 개발용 ADR을 만들지 않는다. 운영 저장 위치는 ADR-D005가 별도로 결정한다. 이 문서는 테이블·논리 접근 경계를 정의하며 운영 DB 서버나 배포 위치를 확정하지 않는다.

## 1. 기존 스키마와 경계

| 기존 테이블 | 이 설계의 사용 방식 |
|---|---|
| `dashboard_users` | 담당자, 승인자, 요청자, 감사 행위자의 FK 대상이다. 기존 로그인 역할은 `ADMIN` 하나지만 V7 대응 API는 별도 `secret_role_grants`의 Secret 역할과 연결 범위를 검사한다. `ADMIN`이라는 값만으로 보안 담당자 권한을 부여하지 않는다. 운영용 계정 생성·비활성화와 권한 부여 절차는 후속이다. |
| `assets` | AWS, K8s, GitHub에서 기존 자산과 명확히 대응할 때만 선택적으로 연결한다. 현재 `provider`는 `NOTION`을, `environment`는 `UNKNOWN`을 허용하지 않는다. Notion 위치와 환경 미확인 위치를 `PROD`로 바꾸거나 가짜 자산을 만들지 않는다. |
| `detection_events` | 현재 일반 탐지와 기존 `/api/v1/overview` 조회용으로 유지한다. 필수 심각도와 환경, 네 가지 조치 상태로는 Secret의 미평가, UNKNOWN 환경, 재검증 대기, 위치별 예외를 표현할 수 없다. Secret 사실의 원본 테이블로 사용하지 않는다. 새 통합 현황은 API에서 두 출처의 조회 결과를 합성한다. |

Secret 전용 테이블 이름은 `secret_` 접두사를 쓴다. 기본 키는 기존 DB와 같은 `BIGINT GENERATED ALWAYS AS IDENTITY`다. 모든 시각은 `TIMESTAMPTZ`로 저장하고 API는 UTC로 반환한다. 화면 표시용 문자열, 원천 객체 참조, JSON 근거에도 원문 자격 증명, 연결 토큰, 인증 헤더, 전체 공급자 응답을 넣지 않는다.

새 통합 현황은 API 조회에서 일반 `detection_events` 한 행과 Secret 현재 대응 건 한 행을 각각 위험 항목으로 합성한다. 응답에 영역과 집계 단위를 명시하고, Secret을 `detection_events`에 다시 적재하지 않는다. 기존 `/api/v1/overview`의 점수와 4상태 계약은 그대로 둔 상태에서 새 통합 응답을 별도 버전으로 제공하고 화면을 전환한다. 심각도 미평가와 환경 UNKNOWN을 임의 등급과 PROD로 치환하지 않는다.

## 2. 관계와 집계 단위

| 관계 | 기준 |
|---|---|
| 연결 1개에서 범위 여러 개, 실행 여러 개 | `secret_connections` → `secret_connection_scopes`, `secret_scan_runs` |
| 실행 1개에서 범위별 수집 결과와 엔진별 결과 여러 개 | `secret_scan_runs` → `secret_run_scopes`, `secret_tool_runs` |
| 자격 증명 1개에서 발견 위치 여러 개 | `secret_credential_sets` → `secret_occurrences`; 위치와 엔진 탐지는 별개 |
| 발견 위치 1개에서 엔진 발견과 심각도 평가 여러 개 | `secret_occurrences` → `secret_findings`, `secret_assessments` |
| 자격 증명 1개에서 검사 시도와 대응 이력 여러 개 | `secret_credential_sets` → `secret_validation_attempts`, `secret_response_cases` |
| 대응 건 1개에서 위치, 작업, 예외와 감사 여러 개 | `secret_response_cases` → 위치 연결 및 대응 기록 |

목표 위험 화면의 단위는 `ResponseCase`다. 현재 MVP의 `GET /api/v1/secrets`는 `CredentialSet` 단위 인벤토리이며 아직 대응 모델을 제공하지 않는다. 키 수는 `CredentialSet`, 노출 위치 수는 `Occurrence`, 엔진 결과 수는 `Finding`으로 따로 센다. 보존 중인 만료 및 폐기 키에 대응 건이 없으면 목표 위험 화면에 넣지 않고 인벤토리에서 조회한다.

## 3. 테이블 명세

V3–V8은 `develop`에 반영된 제약이다. 후속 목표는 별도로 표시한다. `NOT NULL`을 적지 않은 현재 컬럼은 NULL을 허용한다. 목표 표에서 연속으로 적은 시각 컬럼은 각각 `TIMESTAMPTZ`이며, 연속으로 적은 사용자 ID는 각각 `BIGINT FK`다. 코드형 값은 `VARCHAR`와 `CHECK`를 쓰되 공급자별 키 유형과 룰 ID는 버전 관리하는 애플리케이션 등록부에서 검증한다.

### 3.1. 연결, 범위, 실행

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_connections` | `connection_id BIGINT PK`, `provider VARCHAR(20) NOT NULL`, `display_name VARCHAR(200) NOT NULL`, `auth_ref VARCHAR(300) NOT NULL`, `status VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `next_due_at TIMESTAMPTZ`, `last_collection_success_at TIMESTAMPTZ`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL` | `provider`는 `AWS`, `KUBERNETES`, `GITHUB`, `NOTION`; `status`는 `ACTIVE`, `PAUSED`, `ERROR`. `auth_ref`는 비밀 저장소의 불투명 참조이며 값 자체는 넣지 않는다. 범위 변경 때 `scope_version`을 증가시킨다. |
| `secret_connection_scopes` | `scope_id BIGINT PK`, `connection_id BIGINT NOT NULL FK`, `scope_key VARCHAR(300) NOT NULL`, `revision BIGINT NOT NULL`, `content_type VARCHAR(80) NOT NULL`, `approved_ref VARCHAR(500) NOT NULL`, `history_limit VARCHAR(100)`, `enabled BOOLEAN NOT NULL`, `classification_version VARCHAR(80)`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL`; V7의 `response_origin VARCHAR(20) NOT NULL`, `response_origin_set_by_user_id BIGINT FK`, `response_origin_set_at TIMESTAMPTZ`; V8의 `response_origin_version BIGINT NOT NULL` | `UQ(connection_id, scope_key, revision)`. 범위 정의는 실행 이력의 근거이므로 승인 대상과 이력 한도를 제자리에서 바꾸지 않고 새 revision 행을 만든다. 저장소, 브랜치, 페이지, AWS 계정과 리전, K8s 네임스페이스처럼 승인된 단위를 표현한다. 대응 구분은 `UNCLASSIFIED`가 기본이며 V8은 허용된 코드 사유와 버전 확인을 거쳐 `PRODUCTION`·`TEST`로 한 번만 분류하고 별도 이력을 추가한다. |
| `secret_asset_classifications` | `classification_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `resource_ref VARCHAR(500) NOT NULL`, `environment VARCHAR(10) NOT NULL`, `criticality VARCHAR(20) NOT NULL`, `sensitive_data BOOLEAN`, `version BIGINT NOT NULL`, `evidence_ref VARCHAR(500)`, `classified_at TIMESTAMPTZ NOT NULL`, `classified_by_user_id BIGINT FK` | `UQ(scope_id, resource_ref, version)`. 환경은 `PROD`, `DEV`, `UNKNOWN`; 중요도는 `CRITICAL`, `IMPORTANT`, `LIMITED`, `MINIMAL`, `UNKNOWN`. 이름만 보고 분류하지 않는다. 과거 버전을 보존한다. |
| `secret_jobs` | `job_id BIGINT PK`, `kind VARCHAR(20) NOT NULL`, `connection_id BIGINT NOT NULL FK`, `requested_by_user_id BIGINT FK`, `idempotency_key VARCHAR(120)`, `scheduled_at TIMESTAMPTZ NOT NULL`, `available_at TIMESTAMPTZ NOT NULL`, `attempt_no INTEGER NOT NULL`, `max_attempts INTEGER NOT NULL`, `status VARCHAR(20) NOT NULL`, `lease_owner VARCHAR(120)`, `lease_until TIMESTAMPTZ`, `last_error_code VARCHAR(80)`, `created_at`, `finished_at TIMESTAMPTZ` | V3은 `FULL_SCAN`만 허용한다. `REVALIDATE`와 `credential_set_id`는 후속 마이그레이션 대상이다. 상태는 `PENDING`, `LEASED`, `SUCCEEDED`, `FAILED`, `CANCELLED`. 정기 예약·수동 멱등·연결별 활성 작업 중복은 별도 고유 인덱스로 막는다. |
| `secret_scan_runs` | `run_id BIGINT PK`, `job_id BIGINT NOT NULL UQ FK`, `connection_id BIGINT NOT NULL FK`, `trigger_type VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `scheduled_at`, `requested_at TIMESTAMPTZ NOT NULL`, `started_at`, `finished_at TIMESTAMPTZ`, `collection_status`, `detection_status`, `validation_status VARCHAR(20) NOT NULL`, `result_status VARCHAR(20) NOT NULL`, `failure_code VARCHAR(80)`, `created_at TIMESTAMPTZ NOT NULL` | V3에서 `scheduled_at`도 `TIMESTAMPTZ NOT NULL`이다. 단계 상태는 `PENDING`, `RUNNING`, `SUCCEEDED`, `PARTIAL`, `FAILED`, `SKIPPED`; 전체 결과도 같은 범위다. `trigger_type`은 `SCHEDULED`, `MANUAL`. 수동 재검증은 별도 작업과 검사 시도로 추적하며 전체 범위 스캔으로 꾸미지 않는다. |
| `secret_run_scopes` | `run_scope_id BIGINT PK`, `run_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `collection_status VARCHAR(20) NOT NULL`, `object_count BIGINT`, `failure_code VARCHAR(80)`, `unsupported_reason VARCHAR(120)`, `started_at`, `finished_at TIMESTAMPTZ` | `UQ(run_id, scope_id)`, `CHECK(object_count >= 0)`. `SUCCEEDED`와 `object_count=0`은 정상 0건이다. `PARTIAL`이나 `FAILED`는 남은 개수와 무관하게 수집 누락을 뜻한다. IAM은 관측 접근키 수, Secrets Manager·SSM·S3 메타데이터는 목록 항목 수, 정확한 값 범위는 처리한 단일 본문 수를 기록한다. 실행 당시 범위와 버전을 기록한다. |
| `secret_tool_runs` | `tool_run_id BIGINT PK`, `run_scope_id BIGINT NOT NULL FK`, `engine VARCHAR(20) NOT NULL`, `engine_version VARCHAR(100) NOT NULL`, `config_hash VARCHAR(100) NOT NULL`, `status VARCHAR(20) NOT NULL`, `finding_count BIGINT`, `failure_code VARCHAR(80)`, `started_at`, `finished_at TIMESTAMPTZ` | `UQ(run_scope_id, engine)`. MVP 실행 엔진은 `GITLEAKS`다. `TRUFFLEHOG`는 향후 병행 운영을 선택할 때 구분해 기록할 값이며 현재 실행을 뜻하지 않는다. 탐지기의 자체 검증 결과는 제품 유효성으로 쓰지 않는다. |

`secret_jobs`의 워커 선점은 도래한 행을 `FOR UPDATE SKIP LOCKED`로 읽어 같은 트랜잭션에서 `LEASED`, `lease_owner`, `lease_until`로 바꾼다. 만료된 임대는 재시도 한도 내에서 다시 `PENDING`으로 돌린다. 정기 작업의 다음 예정은 이전 예정 시각과 6시간 정책을 기준으로 만들고 수동 작업은 그 시각을 바꾸지 않는다. 공급자별 호출 제한 설정이 없으면 능동 검증 작업을 실행하지 않는다.

AWS의 Secrets Manager·SSM·S3 메타데이터 범위는 `secret_connection_scopes`에 승인 참조와 리전을 기록한다. 목록 결과의 자원 이름·ARN·설명·태그와 페이지 토큰은 영속 저장하지 않고 `secret_run_scopes`에 개수·상태·안전한 실패 코드만 남긴다. API의 `scopeResults`도 내부 실행 범위 ID, 허용된 콘텐츠 유형, 승인 참조에서 추출한 리전, 개수·상태·안전한 누락 코드만 반환한다. 계정 ID와 승인 참조 자체는 반환하지 않는다.

정확한 AWS 값 범위는 별도 `content_type`인 `AWS_SECRETS_MANAGER_VALUE`, `AWS_SSM_PARAMETER_VALUE`, `AWS_S3_OBJECT_CONTENT`로 승인하고 `history_limit=CURRENT_VALUE_ONLY`를 사용한다. `approved_ref`에는 한 Secret·Parameter ARN 또는 한 S3 버킷·객체 키가 포함되므로 일반 응답·로그에 노출하지 않고 연결 DB와 백업의 접근을 제한한다. 현행 `VARCHAR(500)` 범위를 넘는 URI는 거부한다. 값 읽기 설정이 꺼진 범위는 안전한 미실행 코드로 기록한다. 켠 범위에서 텍스트 한 건을 읽으면 V4 도구 실행·발견 위치·Finding과 필요한 V5 검증 시도에 안전한 결과를 기록한다. 위치의 `source_object_ref`는 원본 ARN·이름·객체 키 대신 `aws-content-scope:<scope_id>` 별칭이다. DB에는 읽은 본문 원문을 저장하지 않으며 발견 자체로 노출을 확정하지 않는다.

### 3.2. 키, 위치, 탐지와 근거

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_credential_sets` | `credential_set_id BIGINT PK`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `component_state VARCHAR(20) NOT NULL`, `issuer_connection_id BIGINT FK`, `fingerprint BYTEA`, `candidate_locator BYTEA`, `fingerprint_key_version VARCHAR(40)`, `masked_label VARCHAR(120)`, `external_key_ref VARCHAR(300)`, `owner_ref VARCHAR(200)`, `purpose_label VARCHAR(200)`, `issued_at TIMESTAMPTZ`, `expiry_policy VARCHAR(10) NOT NULL`, `expires_at TIMESTAMPTZ`, `lifecycle_status VARCHAR(20) NOT NULL`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | V4는 완전한 `CONFIRMED`에 보호키 HMAC `fingerprint`, 불완전한 `CANDIDATE`에 원천 위치를 포함한 보호키 HMAC `candidate_locator`를 둔다. 마스킹 문자열이나 접두사만으로 후보를 병합하지 않는다. V6의 IAM 접근키 목록은 `METADATA_ONLY`와 `AWS_IAM_ACCESS_KEY`로 기록하고 원문 키 ID 대신 보호키 HMAC `external_key_ref`, 끝 4자리만 남긴 `masked_label`을 사용한다. `expiry_policy`는 `FIXED`, `NEVER`, `UNKNOWN`; `lifecycle_status`는 `ACTIVE`, `EXPIRED`, `REVOKED`, `UNKNOWN`이다. `lifecycle_evidence_id`는 후속 근거 테이블과 함께 추가한다. |
| `secret_occurrences` | `occurrence_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `connection_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `asset_id BIGINT FK`, `source_object_ref VARCHAR(500) NOT NULL`, `location_key VARCHAR(500) NOT NULL`, `latest_object_version VARCHAR(200)`, `environment VARCHAR(10) NOT NULL`, `exposure_status VARCHAR(10) NOT NULL`, `exposure_range VARCHAR(10) NOT NULL`, `presence_status VARCHAR(10) NOT NULL`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `last_seen_run_id BIGINT FK`, `last_complete_coverage_at TIMESTAMPTZ`; V8의 `exposure_assessment_version BIGINT NOT NULL` | `UQ(credential_set_id, connection_id, source_object_ref, location_key)`. 환경은 `PROD`, `DEV`, `UNKNOWN`; 노출은 `ABNORMAL`, `NORMAL`, `UNKNOWN`; 범위는 `BROAD`, `BOUNDED`, `UNKNOWN`; 존재는 `PRESENT`, `MISSING`, `UNKNOWN`. `scope_id`는 같은 연결에 속해야 한다. 자산 연결은 선택이다. 수집 재실행은 수동 노출 판정 상태를 덮어쓰지 않는다. |
| `secret_findings` | `finding_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `tool_run_id BIGINT NOT NULL FK`, `finding_key VARCHAR(100) NOT NULL`, `rule_id VARCHAR(120) NOT NULL`, `rule_version VARCHAR(100) NOT NULL`, `masked_context TEXT`, `detected_at TIMESTAMPTZ NOT NULL` | `UQ(tool_run_id, finding_key)`. 마스킹된 문맥만 저장한다. 동일 위치의 두 엔진 결과를 둘 다 남기되 위치를 두 번 만들지 않는다. `tool_run_id`와 `run_id`의 관계를 적재 트랜잭션에서 검증한다. |
| `secret_evidence` | `evidence_id BIGINT PK`, `credential_set_id BIGINT FK`, `occurrence_id BIGINT FK`, `run_id BIGINT FK`, `kind VARCHAR(40) NOT NULL`, `observation_status VARCHAR(20) NOT NULL`, `source VARCHAR(80) NOT NULL`, `source_ref VARCHAR(500)`, `observed_at TIMESTAMPTZ`, `collected_at TIMESTAMPTZ NOT NULL`, `mapping_version VARCHAR(80)`, `classification_id BIGINT FK`, `safe_facts JSONB NOT NULL` | `CHECK(credential_set_id IS NOT NULL OR occurrence_id IS NOT NULL)`. 근거 종류는 `EXPOSURE`, `IDENTITY`, `PERMISSION`, `RESOURCE`, `USAGE`, `ABUSE`, `LIFECYCLE`, `REVALIDATION` 등이다. 상태는 `CONFIRMED`, `NO_RECORD`, `UNSUPPORTED`, `UNAVAILABLE`, `CONFLICT`. `NO_RECORD`는 미사용 또는 악용 없음의 확정이 아니다. `safe_facts`는 허용 필드만 서버가 생성한다. |
| `secret_validation_attempts` | `attempt_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `job_id BIGINT NOT NULL FK`, `validator_id VARCHAR(120) NOT NULL`, `validator_version VARCHAR(80) NOT NULL`, `endpoint_class VARCHAR(80) NOT NULL`, `requested_at`, `completed_at TIMESTAMPTZ NOT NULL`, `provider_checked_at TIMESTAMPTZ`, `result VARCHAR(10) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `next_due_at TIMESTAMPTZ` | V5는 `(run_id, job_id)` 복합 FK로 실제 실행과 연결한다. 결과는 `VALID`, `INVALID`, `UNKNOWN`, `UNCHECKED`이고 안전한 사유 코드만 저장한다. 실제 공급자 확인이 없는 `UNCHECKED`의 `provider_checked_at`은 NULL이다. 근거 테이블과 `evidence_id`는 후속이다. |
| `secret_aws_access_key_observations` | `credential_set_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `key_status VARCHAR(10) NOT NULL`, `issued_at`, `last_used_at TIMESTAMPTZ`, `last_used_service VARCHAR(120)`, `last_used_region VARCHAR(80)`, `observed_at TIMESTAMPTZ NOT NULL` | V6의 기본 키는 `(credential_set_id, run_id)`이며 같은 키의 실행별 관측을 보존한다. `key_status`는 `ACTIVE`, `INACTIVE`, `EXPIRED`, `UNKNOWN`만 허용한다. 최근 사용 기록이 없거나 조회 실패한 경우를 미사용으로 단정하지 않는다. 키 ID 원문·사용자명·Secret 값은 저장하지 않는다. |
| `secret_scope_origin_decisions` | V8: `decision_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `from_origin VARCHAR(20) NOT NULL`, `to_origin VARCHAR(20) NOT NULL`, `reason_code VARCHAR(40) NOT NULL`, `actor_user_id BIGINT NOT NULL FK`, `decided_at TIMESTAMPTZ NOT NULL`, `version BIGINT NOT NULL` | `(scope_id, version)` 고유. `UNCLASSIFIED`에서 `TEST`·`APPROVED_TEST_FIXTURE` 또는 `PRODUCTION`·`APPROVED_PRODUCTION_SOURCE`로 한 번 분류한다. 알려진 합성 표본 저장소를 운영으로 분류하는 요청은 서버가 거부한다. 변경 전후, 코드 사유, 수행자와 시각을 추가 전용 이력으로 남긴다. 자동 기본 분류는 없다. |
| `secret_exposure_assessments` | V8: `assessment_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `finding_id BIGINT NOT NULL`, `finding_rule_id VARCHAR(120) NOT NULL`, `finding_run_id BIGINT NOT NULL`, `from_status VARCHAR(20) NOT NULL`, `decision VARCHAR(20) NOT NULL`, `reason_code VARCHAR(40) NOT NULL`, `scope_origin VARCHAR(20) NOT NULL`, `actor_user_id BIGINT NOT NULL FK`, `assessed_at TIMESTAMPTZ NOT NULL`, `version BIGINT NOT NULL` | `(occurrence_id, version)` 고유. `NORMAL`·`AUTHORIZED_LOCATION`, `ABNORMAL`·`UNAPPROVED_SOURCE_LOCATION`, 시험 범위에서만 `ABNORMAL`·`SIMULATED_EXPOSURE`를 허용한다. 현재 노출 상태와 버전 변경을 한 트랜잭션에 저장한다. Finding은 재탐지 시 행이 재생성될 수 있어 FK 대신 판정 당시 ID·룰·실행 ID를 스냅샷으로 보존한다. 형식 탐지는 노출 판정의 자동 증거가 아니다. |
| `secret_assessments` | `assessment_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `run_id BIGINT FK`, `policy_version VARCHAR(80) NOT NULL`, `rule_id VARCHAR(20) NOT NULL`, `assessment_kind VARCHAR(10) NOT NULL`, `exposure_status VARCHAR(10) NOT NULL`, `exposure_range VARCHAR(10) NOT NULL`, `credential_state VARCHAR(20) NOT NULL`, `impact VARCHAR(10) NOT NULL`, `abuse VARCHAR(20) NOT NULL`, `validity VARCHAR(10) NOT NULL`, `severity VARCHAR(10)`, `missing_reason VARCHAR(120)`, `mapping_version VARCHAR(80)`, `assessed_at TIMESTAMPTZ NOT NULL`, `approved_by_user_id BIGINT FK`, `supersedes_assessment_id BIGINT FK` | `assessment_kind`는 `AUTO`, `MANUAL`; 등급은 `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `UNASSESSED` 또는 정상 보관의 NULL. `impact`는 `ADMIN`, `IMPORTANT`, `LIMITED`, `MINIMAL`, `UNKNOWN`. 수동 하향에는 승인자와 반증 근거가 필요하다. 과거 평가를 UPDATE하지 않는다. |
| `secret_assessment_evidence` | `assessment_id BIGINT NOT NULL FK`, `evidence_id BIGINT NOT NULL FK`, `input_name VARCHAR(40) NOT NULL`, 복합 PK | 평가 입력마다 증거와 관측 시각을 `secret_evidence`에 연결한다. 없는 근거는 `missing_reason`으로 남긴다. |

`secret_credential_sets.lifecycle_evidence_id`는 `secret_evidence`를 구현할 때 FK로 추가한다. 발급자와 유형은 수집 플랫폼과 독립적이다. 예를 들어 Notion 문서에 있는 AWS 키는 `secret_connections.provider=NOTION`, `secret_credential_sets.issuer=AWS`다. AWS Access Key ID처럼 공개 식별자가 코드에서 단독으로 발견되면 비밀 구성요소가 부족하므로 `CANDIDATE`로 남긴다. 발급처 목록에서 존재를 확인했지만 값을 반환하지 않으면 `METADATA_ONLY`이며 이는 유효성 `VALID`나 비정상 노출의 증거가 아니다. 원문 없이 확인된 플랫폼 목록과 발견 값을 동일한 키로 자동 병합하지 않는다. 최근 사용 메타데이터는 V6 관측 이력에 저장하며 확인 권한과 자원의 일반 `secret_evidence` 기록은 후속 목표다.

### 3.3. 대응과 변경 이력

V7에서 구현된 테이블은 `secret_response_cases`, `secret_case_occurrences`, `secret_assignment_history`, `secret_role_grants`, `secret_team_memberships`, `secret_audit_events`다. V8은 별도 수동 범위 분류·노출 판정 이력을 추가했다. 아래 표의 나머지 대응·심각도 평가·예외 테이블과 V7 행에 적은 후속 필드는 아직 목표다. V7·V8은 기존 발견을 대응 건으로 소급 생성하지 않는다. 현재 수집 위치의 노출은 기본 `UNKNOWN`, 기존 범위의 대응 구분은 기본 `UNCLASSIFIED`이므로 별도의 판정·분류 전에는 대응 건을 열 수 없다.

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_response_cases` | `case_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `previous_case_id BIGINT FK`, `opening_occurrence_id BIGINT NOT NULL FK`, `opened_by_user_id BIGINT NOT NULL FK`, `opening_reason VARCHAR(80) NOT NULL`, `case_origin VARCHAR(20) NOT NULL`, `is_current BOOLEAN NOT NULL`, `disposition VARCHAR(20) NOT NULL`, `action_status VARCHAR(30) NOT NULL`, `review_status VARCHAR(20) NOT NULL`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT FK`, `version BIGINT NOT NULL`, `opened_at`, `updated_at TIMESTAMPTZ NOT NULL`, `resolved_at`, `closed_at TIMESTAMPTZ` | V7은 `case_origin`을 `PRODUCTION`·`TEST`로 분리하고 `(credential_set_id, case_origin)`마다 현재 건을 최대 하나 허용한다. 개설 위치는 같은 키에 속해야 한다. 현재 개설 단계는 `OPEN`, 검토는 `UNREVIEWED`다. 조치·검토의 이후 전이, 해결·재발은 후속 구현이다. `RISK_ACCEPTED`는 조치 상태가 아니라 후속 예외에서 계산한다. |
| `secret_case_occurrences` | `case_id`, `occurrence_id`, `credential_set_id BIGINT NOT NULL FK`, `response_status VARCHAR(30) NOT NULL`, `linked_at TIMESTAMPTZ NOT NULL`, `resolved_at TIMESTAMPTZ`, 복합 PK `(case_id, occurrence_id)` | V7은 복합 FK로 위치가 대응 건과 같은 키에 속하는지 확인한다. 현재 개설 위치의 상태는 `OPEN`이다. 위치별 조치·재검증 증거 FK는 후속이다. 일부 위치 해결이 다른 위치를 해결하지 않는다. |
| `secret_assignment_history` | `assignment_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `from_team`, `to_team VARCHAR(100)`, `from_user_id`, `to_user_id BIGINT FK`, `reason VARCHAR(80) NOT NULL`, `changed_by_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | V7은 허용된 사유 코드만 받는다. 현재 담당자는 `secret_response_cases`에 두고 버전 변경, 이력, 감사 행을 한 트랜잭션에 추가한다. 서버가 인증 사용자 ID를 채운다. |
| `secret_case_reviews` | `review_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT FK`, `decision VARCHAR(30) NOT NULL`, `reason VARCHAR(500) NOT NULL`, `reviewed_by_user_id BIGINT NOT NULL FK`, `reviewed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 결정은 `CONFIRMED`, `FALSE_POSITIVE`, `OUT_OF_SCOPE`, `NEEDS_MORE_INFO`. 위치별 판단을 연결한다. 오탐과 대상 제외는 위험 수용과 구분한다. |
| `secret_action_records` | `action_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT FK`, `action_type VARCHAR(40) NOT NULL`, `performed_at TIMESTAMPTZ NOT NULL`, `recorded_at TIMESTAMPTZ NOT NULL`, `performed_by_label VARCHAR(200) NOT NULL`, `recorded_by_user_id BIGINT NOT NULL FK`, `evidence_ref VARCHAR(500)`, `safe_note TEXT` | 실제 외부 작업을 사람이 기록한다. `performed_at`은 서버 기록 시각과 다르다. 제품이 교체나 폐기를 실행했다는 플래그는 두지 않는다. |
| `secret_checklist_items` | `item_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `guide_version VARCHAR(80) NOT NULL`, `item_code VARCHAR(80) NOT NULL`, `status VARCHAR(20) NOT NULL`, `completed_at TIMESTAMPTZ`, `completed_by_user_id BIGINT FK`, `evidence_ref VARCHAR(500)`, `updated_at TIMESTAMPTZ NOT NULL` | `UQ(case_id, guide_version, item_code)`. `status`는 `TODO`, `DONE`, `NOT_APPLICABLE`. 체크 완료만으로 대응 건을 해결하지 않는다. |
| `secret_exceptions` | `exception_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `reason VARCHAR(1000) NOT NULL`, `approved_by_user_id BIGINT NOT NULL FK`, `approved_at TIMESTAMPTZ NOT NULL`, `expires_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ`, `revoked_by_user_id BIGINT FK`, `revoke_reason VARCHAR(500)` | `CHECK(expires_at > approved_at)`. 취소와 만료를 삭제 대신 상태로 보존한다. 승인 권한은 서버가 검사한다. |
| `secret_exception_occurrences` | `exception_id BIGINT NOT NULL FK`, `occurrence_id BIGINT NOT NULL FK`, 복합 PK | 승인 당시의 위치 범위를 고정한다. 나중에 새 위치가 발견돼도 예외 범위에 자동 포함되지 않는다. |
| `secret_role_grants` | `grant_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `role VARCHAR(30) NOT NULL`, `connection_id BIGINT FK`, `granted_by_user_id BIGINT NOT NULL FK`, `grant_source VARCHAR(20) NOT NULL`, `granted_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ` | V7의 역할은 `SECURITY`, `REMEDIATOR`, `OPERATIONS`이고 부여 출처는 `BOOTSTRAP`, `MANUAL`이다. 연결 범위를 비우면 전체 범위다. 기존 `dashboard_users.role=ADMIN`과 별도로 서버가 대상 연결 범위까지 검사한다. 활성 중복 권한 부여를 막는다. |
| `secret_team_memberships` | `membership_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `team_name VARCHAR(100) NOT NULL`, `added_by_user_id BIGINT NOT NULL FK`, `added_at TIMESTAMPTZ NOT NULL`, `removed_at TIMESTAMPTZ` | 팀 담당자 지정 권한 확인에 사용한다. 팀 문자열만으로 소속을 추정하지 않는다. 활성 중복 소속을 막는다. |
| `secret_audit_events` | V7: `audit_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `event_type VARCHAR(30) NOT NULL`, `actor_user_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `occurred_at TIMESTAMPTZ NOT NULL` | V7은 `CASE_OPENED`, `ASSIGNMENT_CHANGED`만 추가 기록한다. 비밀이나 자유 형식 원문을 담지 않는다. 요청 ID, 변경 전후 안전한 값, 다른 종류의 감사와 보존·접근 정책은 후속이다. |

`expectedVersion` 변경 요청은 `UPDATE secret_response_cases ... WHERE case_id=:id AND version=:expectedVersion`의 영향 행 수가 1인지 확인하고 같은 트랜잭션에서 버전 증가, 자식 기록, 감사 이벤트를 저장한다. 0행이면 409를 반환한다. 해결 전이는 모든 `ABNORMAL` 위치의 해결 증거와 외부 조치 이후 재확인 증거를 검사한다. 검증 실패, 체크리스트 완료, 예외 승인만으로 `RESOLVED`로 바꾸지 않는다.

예외 승인 시에는 대상 위치가 해당 대응 건에 속하는지, 중복된 활성 승인이 없는지, 승인자에게 해당 연결 범위 권한이 있는지를 같은 트랜잭션에서 검사한다. 예외 만료나 취소 후에는 해당 위치를 조치 상태로 되돌리고 새 스냅샷을 만든다. 감사 행은 UPDATE 및 DELETE를 일반 API 계정에 허용하지 않는다.

오탐이나 대상 제외 판단은 검토 근거를 남긴다. 일부 위치만 제외되면 다른 비정상 노출 위치의 대응을 계속한다. 모든 위치가 오탐 또는 대상 제외로 확정되면 대응 건의 `disposition`을 해당 결과로 바꾸고 `is_current=false`, `closed_at`을 기록해 위험 화면에서 제외한다. 이를 `RESOLVED`나 `RISK_ACCEPTED`로 표시하지 않고 이력에서 조회한다.

### 3.4. 일관된 화면 스냅샷

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_query_snapshots` | `snapshot_id UUID PK`, `scope VARCHAR(20) NOT NULL`, 대응 조회의 `case_origin VARCHAR(20)`, `as_of TIMESTAMPTZ NOT NULL`, `scope_version_digest VARCHAR(100) NOT NULL`, `partial_coverage BOOLEAN NOT NULL`, `missing_sources JSONB NOT NULL`, `created_at TIMESTAMPTZ NOT NULL`, `expires_at TIMESTAMPTZ NOT NULL` | `scope`는 `CURRENT_CASES`, `INVENTORY`, `HISTORY`. 대응 조회는 `PRODUCTION` 또는 명시적 `TEST` 구분을 스냅샷에 고정하고, 인벤토리는 키 기준 범위로 둔다. 하나의 DB 반복 읽기 트랜잭션에서 아래 행과 함께 생성한다. 누락 범위와 사유도 고정한다. 만료된 ID는 409다. |
| `secret_snapshot_rows` | `snapshot_id UUID NOT NULL FK`, `subject_id BIGINT NOT NULL`, `credential_set_id BIGINT NOT NULL`, `action_bucket VARCHAR(30)`, `disposition VARCHAR(20)`, `review_status VARCHAR(20)`, `severity VARCHAR(10)`, `validity VARCHAR(10) NOT NULL`, `lifecycle_status VARCHAR(20) NOT NULL`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `inventory_source_provider VARCHAR(20)`, `first_detected_at TIMESTAMPTZ`, 복합 PK `(snapshot_id, subject_id)` | `subject_id`는 범위에 따라 대응 건 또는 키 ID다. 네 조치 단계와 별도 예외 중 하나를 `action_bucket`에 확정한다. 인벤토리의 만료와 폐기, 이력의 오탐과 대상 제외도 구분한다. 같은 대응 구분에서 키가 여러 소스에 있어도 행은 하나다. 목록, 카드, 매트릭스가 이 행만 읽는다. |
| `secret_snapshot_row_locations` | `snapshot_id UUID NOT NULL`, `subject_id BIGINT NOT NULL`, `occurrence_id BIGINT NOT NULL FK`, `source_provider VARCHAR(20) NOT NULL`, `environment VARCHAR(10) NOT NULL`, `linked_at TIMESTAMPTZ NOT NULL`, 복합 PK `(snapshot_id, subject_id, occurrence_id)` | 복합 FK `(snapshot_id, subject_id)`가 `secret_snapshot_rows`를 참조한다. 소스와 환경은 같은 위치 행에서 함께 필터한다. 위치가 여러 개여도 대상 행을 한 번만 센다. |

목표 `CURRENT_CASES`는 기본 `PRODUCTION` 구분에서 `is_current=true`이고 `disposition=IN_SCOPE`인 대응 건마다 한 행을 만든다. 명시적 `TEST` 조회는 별도 스냅샷으로 분리하고 운영 카드·매트릭스·통합 현황에 합치지 않는다. `INVENTORY`는 키마다 한 행이다. `HISTORY`는 각 과거 대응 건을 따로 만든다. 구분, 범위, 소스, 환경, 발급자, 유효성, 심각도, 담당자 필터를 먼저 동일하게 적용한 행 집합에서 카드, 매트릭스, 목록을 계산한다. 소스와 환경을 동시에 고른 경우에는 `secret_snapshot_row_locations`의 같은 위치가 두 조건을 모두 만족해야 한다. 위치가 전혀 없는 발급처 목록 키는 인벤토리에서만 `inventory_source_provider`로 필터하며 환경은 UNKNOWN이다. 한 건에 해당하는 미해결 위치가 모두 유효한 승인 예외 범위 안에 있을 때만 `action_bucket=RISK_ACCEPTED`다. 대표 심각도는 미해결 위치의 최신 확인 등급 중 최고 등급이며 전부 미평가이면 `UNASSESSED`다. 미평가 위치 수는 상세와 요약의 별도 필드로 제공한다. 지난 `VALID` 뒤에 외부 조치가 있으면 스냅샷의 유효성은 최신 시도 결과와 조치 이후 재검증 대기를 함께 설명할 수 있도록 API가 마지막 성공 시각과 조치 시각을 별도 조회한다.

## 4. 인덱스와 제약 구현

다음은 V3–V6에 적용된 대표 인덱스다. V7은 `(credential_set_id, case_origin) WHERE is_current`의 고유 인덱스로 구분별 현재 건 중복을 막고, 개설·연결 위치가 해당 키에 속하도록 복합 FK를 둔다. 평가·예외·스냅샷 인덱스와 CHECK는 해당 기능의 후속 마이그레이션에서 적용한다. 현재 FK에는 별도 삭제 연쇄 정책을 두지 않는다.

```sql
CREATE UNIQUE INDEX uq_secret_fingerprint
  ON secret_credential_sets (issuer, credential_type, fingerprint_key_version, fingerprint)
  WHERE component_state = 'CONFIRMED';
CREATE UNIQUE INDEX uq_secret_candidate_locator
  ON secret_credential_sets (fingerprint_key_version, candidate_locator)
  WHERE component_state = 'CANDIDATE';
CREATE UNIQUE INDEX uq_secret_aws_metadata_identity
  ON secret_credential_sets (issuer, credential_type, issuer_connection_id, external_key_ref)
  WHERE component_state = 'METADATA_ONLY' AND issuer = 'AWS';
CREATE UNIQUE INDEX uq_secret_job_idempotency
  ON secret_jobs (idempotency_key)
  WHERE idempotency_key IS NOT NULL;
CREATE UNIQUE INDEX uq_secret_job_scheduled
  ON secret_jobs (connection_id, kind, scheduled_at)
  WHERE requested_by_user_id IS NULL;
CREATE UNIQUE INDEX uq_secret_job_active_connection
  ON secret_jobs (connection_id, kind)
  WHERE status IN ('PENDING', 'LEASED');
CREATE INDEX idx_secret_jobs_available
  ON secret_jobs (available_at, job_id)
  WHERE status = 'PENDING';
CREATE INDEX idx_secret_occurrences_connection_presence
  ON secret_occurrences (connection_id, scope_id, presence_status, last_seen_at DESC);
CREATE INDEX idx_secret_validation_latest
  ON secret_validation_attempts (credential_set_id, completed_at DESC, attempt_id DESC);
CREATE UNIQUE INDEX uq_secret_enabled_scope
  ON secret_connection_scopes (connection_id, scope_key)
  WHERE enabled;
```

V4의 구성요소 CHECK는 `CONFIRMED`의 `fingerprint`, `CANDIDATE`의 `candidate_locator`, `METADATA_ONLY`의 자격 증명 지문 부재를 강제한다. `expiry_policy='FIXED'`에는 `expires_at`이 필요하다. V5는 결과 코드, 사유 코드 형식, 시각 순서와 `VALID`·`INVALID`의 실제 확인 시각을 검사한다. V6은 같은 AWS 연결·유형·보호 참조의 IAM 메타데이터 항목을 하나로 묶고 실행마다 관측을 추가한다. 평가의 `severity='UNASSESSED'`와 `missing_reason`, 대응 해결 시각, 예외 기간 등은 해당 후속 테이블을 만들 때 적용할 목표다. `scope_id`와 `connection_id` 및 실행과 작업의 연결은 현재 복합 FK로 강제한다. 연결된 `assets`의 provider와 환경을 무조건 복사하지 않는다.

아래 FK 목록은 현재 V3–V8과 후속 목표 테이블을 함께 나타낸다. 심각도 평가·예외·스냅샷 등 후속 테이블의 FK는 아직 DB에 없다.

| FK 열 | 참조 대상 |
|---|---|
| 모든 `connection_id`, `scope_id`, `run_id`, `run_scope_id`, `tool_run_id`, `job_id` | 각각 `secret_connections`, `secret_connection_scopes`, `secret_scan_runs`, `secret_run_scopes`, `secret_tool_runs`, `secret_jobs`의 PK |
| `credential_set_id`, `occurrence_id`, `case_id` 참조 | 각각 `secret_credential_sets`, `secret_occurrences`, `secret_response_cases`의 PK. V8의 범위 분류 이력은 `scope_id`, 노출 판정 이력은 `occurrence_id`를 FK로 참조 |
| 후속 `assessment_id`, `exception_id`, `evidence_id`, `classification_id` 참조 | 각각 목표 `secret_assessments`, `secret_exceptions`, `secret_evidence`, `secret_asset_classifications`의 PK. V8의 `secret_exposure_assessments.assessment_id`는 그 테이블 자체의 PK이며 목표 심각도 평가 ID와 다른 기록 |
| `previous_case_id`, `supersedes_assessment_id`, `lifecycle_evidence_id` | 각각 `secret_response_cases.case_id`, `secret_assessments.assessment_id`, `secret_evidence.evidence_id` |
| `asset_id`, 모든 `*_user_id` | 각각 기존 `assets.asset_id`, `dashboard_users.user_id` |
| `secret_snapshot_rows.snapshot_id`, `secret_snapshot_row_locations.occurrence_id` | 각각 `secret_query_snapshots.snapshot_id`, `secret_occurrences.occurrence_id` |

V8의 `secret_exposure_assessments.finding_id`는 재탐지 때 Finding 행이 교체될 수 있어 FK로 두지 않는다. 저장 요청에서 같은 위치에 속한 Finding인지 확인하고 ID·룰·실행 ID를 판정 당시 스냅샷으로 남긴다. 그 Finding이 가장 최근 실행에서 생성됐는지까지 강제하지는 않는다.

`secret_query_snapshots`의 생성은 `REPEATABLE READ`에서 원본을 읽고 스냅샷 헤더와 행을 한 번에 커밋한다. `subject_id`는 범위에 따라 다른 부모 테이블을 가리키므로 FK 대신 생성 트랜잭션의 대상 조회로 무결성을 보장한다. 범위마다 별도 스냅샷을 쓰고 스냅샷 ID를 다른 범위로 재사용하지 않는다.

## 5. 적재와 상태 변경 규칙

다음은 전체 설계의 적재 순서다. 현재 V3–V8과 GitHub·AWS 메타데이터 및 정확한 값 워커가 수행하는 부분은 [구현 연동 현황](../04-integration/current-implementation-and-gaps.md)에서 구분한다.

1. 연결과 승인 범위를 등록할 때 범위 버전, 수집 가능한 콘텐츠, 분류 근거와 비밀 저장소 참조를 기록한다.
2. 정기 작업을 예약하고 워커가 선점한 뒤 `secret_scan_runs`를 만든다. 범위별 성공, 실패, 미지원과 엔진별 성공, 실패를 독립적으로 확정한다.
3. 도구 출력은 워커 메모리에서 정규화한다. 키 원문은 HMAC 계산과 승인된 검증에만 쓰고 DB나 큐에 싣지 않는다. 불완전 후보는 다른 후보와 합치지 않는다.
4. 같은 키와 원천 위치는 기존 `secret_occurrences`를 갱신하고 새 발견은 추가한다. 엔진 결과는 `secret_findings`에 별도 기록한다. 완료된 범위에서만 미발견 위치를 `MISSING`으로 갱신한다. 일부 실패 범위의 기존 위치는 `UNKNOWN`으로 남긴다.
5. GitHub 처리기는 발견한 AWS 키·후보의 시도를 실행 주기 내 중복 제거하고 V5 `secret_validation_attempts`에 안전한 상태·사유·시각을 저장한다. 기본 비활성 또는 합성 실험 저장소에서는 `UNCHECKED`를 기록하며 STS를 호출하지 않는다. 사용 정보와 영향 근거의 `secret_evidence` 저장은 후속 마이그레이션 대상이다. 최신 시도와 마지막 성공 판정은 다른 조회다.
   AWS IAM 처리기는 별도 분기로 접근키 메타데이터를 `METADATA_ONLY` 인벤토리와 V6 관측 이력에 기록한다. 이 분기에서는 탐지·검증을 수행하지 않고 실행 단계 상태를 `SKIPPED`로 남긴다. 완전한 IAM 목록 수집에서만 이전에 보이던 접근키의 부재를 `MISSING`으로 표시하고 부분 실패 때는 부재를 확정하지 않는다.
   AWS의 정확한 값 범위는 별도 opt-in이 켜진 경우에만 한 본문을 Gitleaks에 전달해 V4 탐지·정규화 결과를 기록한다. 발견한 완전한 AWS 키의 V5 검증은 독립된 기본 비활성 설정에 따른다. 이 경로의 원천 참조는 내부 범위 ID 별칭이며 결과나 검증 상태로 외부 노출을 확정하지 않는다.
6. 각 비정상 노출 위치에 `severity-v2` 평가를 추가한다. 자동 판정과 승인된 수동 변경, 사용한 근거를 분리한다. 신규 발견의 만료 또는 폐기가 근거로 확인되면 인벤토리에 보존하고 신규 대응 건은 만들지 않는다.
7. 대응 대상은 현재 건에 위치를 연결한다. 해결 뒤 재노출이면 기존 건을 이력으로 닫고 새 건을 만든다. 할당, 검토, 작업, 체크리스트, 예외, 해결 변경은 버전 확인과 감사 이벤트를 같은 트랜잭션에 쓴다.
8. 조회 스냅샷은 모든 단계의 확정 데이터를 기준으로 생성한다. 실패 범위가 있으면 `partialCoverage`와 `missingSources`를 생성 시점에 계산해 스냅샷에 고정한다. 실패를 0건 정상 수집으로 치환하지 않는다.

## 6. 보안, 보존과 마이그레이션

API DB 계정은 Secret 조회 및 대응 변경 테이블에 필요한 권한만 갖고, 워커 DB 계정은 작업 선점과 탐지 및 검사 적재 권한을 갖는다. 연결용 비밀은 DB 밖의 비밀 저장소에 두며 `auth_ref`만 저장한다. HMAC 보호키는 DB 밖에서 관리하고 버전 교체 시 이전 버전 지문을 안전하게 재계산할 수 있는 시점까지 병행 조회한다. 원천에서 키를 다시 읽을 수 없으면 키 동일성 재계산과 재검증이 불가능하므로 임의 병합하지 않는다. 데이터, 백업, 감사 로그의 접근 권한과 보존 기간은 범주별 운영 설정으로 확정한다. 보존 기간이 정해지지 않은 범주는 자동 삭제하지 않는다.

마이그레이션 순서는 V3 연결·작업, V4 키·위치·탐지, V5 검증 이력, V6 IAM 메타데이터 관측, V7 대응 건·담당자 기초, V8 수동 범위 분류·노출 판정 이력이다. V7은 기존 범위를 `UNCLASSIFIED`로 두고 V8도 기존 위치를 자동 `ABNORMAL`로 바꾸거나 대응 건을 생성하지 않는다. 기존 V1 데이터도 신규 Secret 테이블로 일괄 이관하지 않는다. 샘플 `detection_events`를 실탐지로 간주할 수 없고 필수 환경과 심각도를 Secret의 UNKNOWN 또는 UNASSESSED에 안전하게 대응시킬 수 없기 때문이다. 서버 시작 시 외부 Notion 스캔을 호출하던 동작은 제거했고 새 Secret 화면은 비동기 실행 API를 사용한다. 전환 기간의 `/api/v1/credentials/scan` 직접 실행은 다른 사용처를 확인한 뒤 정리한다. 통합 현황의 새 버전은 위의 API 합성 규칙을 사용하고 기존 점수의 제거와 응답 필드 전환은 [API 계약](api-contract.md)에 맞춰 클라이언트와 함께 배포한다.

공유 `V2__create_credentials.sql`은 유지하고 충돌했던 미공유 스케줄러 마이그레이션은 V3로 반영했다. V4–V8도 `develop`에 순서대로 병합됐다. 격리된 Testcontainers PostgreSQL에서 V8까지 적용하는 백엔드·워커 테스트를 통과했고, 공유 로컬 개발 DB에도 V1–V8 Flyway 적용을 확인했다. 마이그레이션은 기존 인벤토리의 범위·위치를 자동 분류하거나 대응 건을 소급 생성하지 않았다. 이어서 로컬 API에서 승인된 합성 저장소 범위를 보안 담당자가 `TEST`로 분류하고 해당 위치를 수동 판정한 뒤 시험 대응 건을 개설·담당자 지정까지 확인했다. 이 시험은 운영 노출 판정이나 브라우저 클릭 흐름의 검증을 뜻하지 않는다. 기존 개발 DB의 적용 이력은 파일명 변경으로 덮어쓰지 않는다. 별도 개발 DB에서는 GitHub·AWS IAM 작업 실행도 확인했다. 향후 필요한 데이터 이전은 대상, 적용 순서, 동일성·FK·시퀀스 검증과 전환 절차를 정한 뒤 실행한다. 후속 마이그레이션 번호는 공통 DB 담당자가 조율한다.

구현 검증은 Flyway 빈 DB 적용과 기존 V1 위 적용, 잘못된 FK와 중복 현재 건 및 중복 작업의 거부, 6시간 정기와 수동 실행의 독립성, 부분 실패 후 위치 유지, 동일 키의 다중 위치와 다중 엔진 중복 제거, 최신 실패와 지난 성공 보존, 부분 예외 집계, 스냅샷의 카드 및 목록 수 일치, 원문이 DB와 로그에 남지 않는지를 확인한다. V8에서는 미분류·미판정 건의 개설 거부, 알려진 합성 표본의 운영 분류 거부, 다른 위치의 Finding 거부, 판정·분류의 오래된 버전 거부, 재스캔 후 수동 판정 보존, 시험 건의 운영 목록 제외를 추가로 확인한다.
