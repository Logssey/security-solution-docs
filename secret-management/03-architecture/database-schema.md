# Secret 데이터베이스 스키마

기준: [PRD](../01-requirements/PRD.md), [데이터 계약](data-contract.md), [API 계약](api-contract.md), [심각도 정책](../02-functional-design/severity-policy.md), [대응 정책](../02-functional-design/response-policy.md), [ADR-S002](adr/ADR-S002.md), [ADR-S003](adr/ADR-S003.md), [ADR-S005](adr/ADR-S005.md), [ADR-S008](adr/ADR-S008.md).

이 문서는 PostgreSQL에 추가할 물리 스키마의 구현 기준이다. 2026-10-13 정적 확인 기준은 [dashboard-backend](https://github.com/Logssey/dashboard-backend) `14b1812`다. V1에는 `dashboard_users`, `assets`, `detection_events`가 있으며 공유 V2에는 기존 단일 `credentials`가 있다. 공유 main에는 이 문서의 정식 분리 모델이 아직 구현되지 않았다. 로컬에는 연결·작업 예약용 Secret 테이블 5개의 미공유 구현이 있으므로 공유 코드와 구분한다. 현재 상태와 전환 차이는 [구현 연동 현황](../04-integration/current-implementation-and-gaps.md)에 기록한다. 기존 V1과 공유 V2를 수정하지 않고 후속 Flyway 마이그레이션으로 정식 모델을 추가한다.

로컬 개발 DB는 기존 대시보드 ADR-D004를 사용하며 별도 개발용 ADR을 만들지 않는다. 운영 저장 위치는 ADR-D005가 별도로 결정한다. 이 문서는 테이블·논리 접근 경계를 정의하며 운영 DB 서버나 배포 위치를 확정하지 않는다.

## 1. 기존 스키마와 경계

| 기존 테이블 | 이 설계의 사용 방식 |
|---|---|
| `dashboard_users` | 담당자, 승인자, 요청자, 감사 행위자의 FK 대상이다. 현재 DB의 역할은 `ADMIN` 하나이므로 세부 권한은 서버 정책과 후속 역할 확장으로 구현한다. `ADMIN`이라는 값만으로 보안 담당자 승인 권한이 검증됐다고 간주하지 않는다. |
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

기본 위험 화면의 단위는 현재 `ResponseCase`다. 키 수는 `CredentialSet`, 노출 위치 수는 `Occurrence`, 엔진 결과 수는 `Finding`으로 따로 센다. 보존 중인 만료 및 폐기 키에 대응 건이 없으면 위험 화면에 넣지 않고 인벤토리에서 조회한다.

## 3. 테이블 명세

아래 `PK`, `FK`, `UQ`, `CHECK`는 백엔드 마이그레이션에 적용할 제약이다. `NOT NULL`을 적지 않은 컬럼은 근거가 없거나 지원하지 않는 경우를 나타내기 위해 NULL을 허용한다. 표에서 연속으로 적은 시각 컬럼은 모두 각각 `TIMESTAMPTZ`이며, 연속으로 적은 사용자 ID는 각각 `BIGINT FK`다. 팀과 메모 컬럼은 별도 표기가 없으면 각각 `VARCHAR(100)`, `TEXT`다. 코드형 값은 `VARCHAR`와 `CHECK`를 쓰되 공급자별 키 유형과 룰 ID는 버전 관리하는 애플리케이션 등록부에서 검증한다.

### 3.1. 연결, 범위, 실행

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_connections` | `connection_id BIGINT PK`, `provider VARCHAR(20) NOT NULL`, `display_name VARCHAR(200) NOT NULL`, `auth_ref VARCHAR(300) NOT NULL`, `status VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `next_due_at TIMESTAMPTZ`, `last_collection_success_at TIMESTAMPTZ`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL` | `provider`는 `AWS`, `KUBERNETES`, `GITHUB`, `NOTION`; `status`는 `ACTIVE`, `PAUSED`, `ERROR`. `auth_ref`는 비밀 저장소의 불투명 참조이며 값 자체는 넣지 않는다. 범위 변경 때 `scope_version`을 증가시킨다. |
| `secret_connection_scopes` | `scope_id BIGINT PK`, `connection_id BIGINT NOT NULL FK`, `scope_key VARCHAR(300) NOT NULL`, `revision BIGINT NOT NULL`, `content_type VARCHAR(80) NOT NULL`, `approved_ref VARCHAR(500) NOT NULL`, `history_limit VARCHAR(100)`, `enabled BOOLEAN NOT NULL`, `classification_version VARCHAR(80)`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL` | `UQ(connection_id, scope_key, revision)`. 범위 정의는 실행 이력의 근거이므로 승인 대상과 이력 한도를 제자리에서 바꾸지 않고 새 revision 행을 만든다. 저장소, 브랜치, 페이지, AWS 계정과 리전, K8s 네임스페이스처럼 승인된 단위를 표현한다. |
| `secret_asset_classifications` | `classification_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `resource_ref VARCHAR(500) NOT NULL`, `environment VARCHAR(10) NOT NULL`, `criticality VARCHAR(20) NOT NULL`, `sensitive_data BOOLEAN`, `version BIGINT NOT NULL`, `evidence_ref VARCHAR(500)`, `classified_at TIMESTAMPTZ NOT NULL`, `classified_by_user_id BIGINT FK` | `UQ(scope_id, resource_ref, version)`. 환경은 `PROD`, `DEV`, `UNKNOWN`; 중요도는 `CRITICAL`, `IMPORTANT`, `LIMITED`, `MINIMAL`, `UNKNOWN`. 이름만 보고 분류하지 않는다. 과거 버전을 보존한다. |
| `secret_jobs` | `job_id BIGINT PK`, `kind VARCHAR(20) NOT NULL`, `connection_id BIGINT FK`, `credential_set_id BIGINT FK`, `requested_by_user_id BIGINT FK`, `idempotency_key VARCHAR(120)`, `scheduled_at`, `available_at TIMESTAMPTZ NOT NULL`, `attempt_no INTEGER NOT NULL`, `max_attempts INTEGER NOT NULL`, `status VARCHAR(20) NOT NULL`, `lease_owner VARCHAR(120)`, `lease_until TIMESTAMPTZ`, `last_error_code VARCHAR(80)`, `created_at`, `finished_at TIMESTAMPTZ` | `kind`는 `FULL_SCAN`, `REVALIDATE`; 대상은 전자에서 연결 ID, 후자에서 키 ID가 필수다. `status`는 `PENDING`, `LEASED`, `SUCCEEDED`, `FAILED`, `CANCELLED`. `CHECK(attempt_no >= 0 AND max_attempts > 0)`. 정기 스캔의 중복 예약과 수동 요청의 멱등 키는 별도 고유 인덱스로 막는다. |
| `secret_scan_runs` | `run_id BIGINT PK`, `job_id BIGINT UQ FK`, `connection_id BIGINT NOT NULL FK`, `trigger_type VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `scheduled_at`, `requested_at TIMESTAMPTZ NOT NULL`, `started_at`, `finished_at TIMESTAMPTZ`, `collection_status`, `detection_status`, `validation_status VARCHAR(20) NOT NULL`, `result_status VARCHAR(20) NOT NULL`, `failure_code VARCHAR(80)`, `created_at TIMESTAMPTZ NOT NULL` | 단계 상태는 `PENDING`, `RUNNING`, `SUCCEEDED`, `PARTIAL`, `FAILED`, `SKIPPED`; 전체 결과도 같은 범위다. `trigger_type`은 `SCHEDULED`, `MANUAL`. 수동 재검증은 별도 작업과 검사 시도로 추적하며 전체 범위 스캔으로 꾸미지 않는다. |
| `secret_run_scopes` | `run_scope_id BIGINT PK`, `run_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `collection_status VARCHAR(20) NOT NULL`, `object_count BIGINT`, `failure_code VARCHAR(80)`, `unsupported_reason VARCHAR(120)`, `started_at`, `finished_at TIMESTAMPTZ` | `UQ(run_id, scope_id)`, `CHECK(object_count >= 0)`. `SUCCEEDED`와 `object_count=0`은 정상 0건, `FAILED`나 `PARTIAL`과 0 또는 NULL은 수집 누락이다. 실행 당시 범위와 버전을 기록한다. |
| `secret_tool_runs` | `tool_run_id BIGINT PK`, `run_scope_id BIGINT NOT NULL FK`, `engine VARCHAR(20) NOT NULL`, `engine_version VARCHAR(100) NOT NULL`, `config_hash VARCHAR(100) NOT NULL`, `status VARCHAR(20) NOT NULL`, `finding_count BIGINT`, `failure_code VARCHAR(80)`, `started_at`, `finished_at TIMESTAMPTZ` | `UQ(run_scope_id, engine)`. MVP 실행 엔진은 `GITLEAKS`다. `TRUFFLEHOG`는 향후 병행 운영을 선택할 때 구분해 기록할 값이며 현재 실행을 뜻하지 않는다. 탐지기의 자체 검증 결과는 제품 유효성으로 쓰지 않는다. |

`secret_jobs`의 워커 선점은 도래한 행을 `FOR UPDATE SKIP LOCKED`로 읽어 같은 트랜잭션에서 `LEASED`, `lease_owner`, `lease_until`로 바꾼다. 만료된 임대는 재시도 한도 내에서 다시 `PENDING`으로 돌린다. 정기 작업의 다음 예정은 이전 예정 시각과 6시간 정책을 기준으로 만들고 수동 작업은 그 시각을 바꾸지 않는다. 공급자별 호출 제한 설정이 없으면 능동 검증 작업을 실행하지 않는다.

### 3.2. 키, 위치, 탐지와 근거

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_credential_sets` | `credential_set_id BIGINT PK`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `component_state VARCHAR(20) NOT NULL`, `issuer_connection_id BIGINT FK`, `fingerprint BYTEA`, `fingerprint_key_version VARCHAR(40)`, `masked_label VARCHAR(120)`, `external_key_ref VARCHAR(300)`, `owner_ref VARCHAR(200)`, `purpose_label VARCHAR(200)`, `issued_at TIMESTAMPTZ`, `expiry_policy VARCHAR(10) NOT NULL`, `expires_at TIMESTAMPTZ`, `lifecycle_status VARCHAR(20) NOT NULL`, `lifecycle_evidence_id BIGINT`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | `component_state`는 `CONFIRMED`, `CANDIDATE`, `METADATA_ONLY`; 후보를 마스킹 문자열이나 접두사만으로 병합하지 않는다. 발급처 목록에서 원문 없이 확인한 키는 `METADATA_ONLY`다. `fingerprint`는 보호키 HMAC만 허용한다. `expiry_policy`는 `FIXED`, `NEVER`, `UNKNOWN`이고 `FIXED`이면 만료 시각이 필수다. `lifecycle_status`는 `ACTIVE`, `EXPIRED`, `REVOKED`, `UNKNOWN`; `INVALID`만으로 EXPIRED 또는 REVOKED로 바꾸지 않는다. |
| `secret_occurrences` | `occurrence_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `connection_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `asset_id BIGINT FK`, `source_object_ref VARCHAR(500) NOT NULL`, `location_key VARCHAR(500) NOT NULL`, `latest_object_version VARCHAR(200)`, `environment VARCHAR(10) NOT NULL`, `exposure_status VARCHAR(10) NOT NULL`, `exposure_range VARCHAR(10) NOT NULL`, `presence_status VARCHAR(10) NOT NULL`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `last_seen_run_id BIGINT FK`, `last_complete_coverage_at TIMESTAMPTZ` | `UQ(credential_set_id, connection_id, source_object_ref, location_key)`. 환경은 `PROD`, `DEV`, `UNKNOWN`; 노출은 `ABNORMAL`, `NORMAL`, `UNKNOWN`; 범위는 `BROAD`, `BOUNDED`, `UNKNOWN`; 존재는 `PRESENT`, `MISSING`, `UNKNOWN`. `scope_id`는 같은 연결에 속해야 한다. 자산 연결은 선택이다. |
| `secret_findings` | `finding_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `tool_run_id BIGINT NOT NULL FK`, `finding_key VARCHAR(100) NOT NULL`, `rule_id VARCHAR(120) NOT NULL`, `rule_version VARCHAR(100) NOT NULL`, `masked_context TEXT`, `detected_at TIMESTAMPTZ NOT NULL` | `UQ(tool_run_id, finding_key)`. 마스킹된 문맥만 저장한다. 동일 위치의 두 엔진 결과를 둘 다 남기되 위치를 두 번 만들지 않는다. `tool_run_id`와 `run_id`의 관계를 적재 트랜잭션에서 검증한다. |
| `secret_evidence` | `evidence_id BIGINT PK`, `credential_set_id BIGINT FK`, `occurrence_id BIGINT FK`, `run_id BIGINT FK`, `kind VARCHAR(40) NOT NULL`, `observation_status VARCHAR(20) NOT NULL`, `source VARCHAR(80) NOT NULL`, `source_ref VARCHAR(500)`, `observed_at TIMESTAMPTZ`, `collected_at TIMESTAMPTZ NOT NULL`, `mapping_version VARCHAR(80)`, `classification_id BIGINT FK`, `safe_facts JSONB NOT NULL` | `CHECK(credential_set_id IS NOT NULL OR occurrence_id IS NOT NULL)`. 근거 종류는 `EXPOSURE`, `IDENTITY`, `PERMISSION`, `RESOURCE`, `USAGE`, `ABUSE`, `LIFECYCLE`, `REVALIDATION` 등이다. 상태는 `CONFIRMED`, `NO_RECORD`, `UNSUPPORTED`, `UNAVAILABLE`, `CONFLICT`. `NO_RECORD`는 미사용 또는 악용 없음의 확정이 아니다. `safe_facts`는 허용 필드만 서버가 생성한다. |
| `secret_validation_attempts` | `attempt_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `run_id BIGINT FK`, `job_id BIGINT FK`, `validator_id VARCHAR(120) NOT NULL`, `validator_version VARCHAR(80) NOT NULL`, `endpoint_class VARCHAR(80) NOT NULL`, `requested_at`, `completed_at TIMESTAMPTZ`, `result VARCHAR(10) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `evidence_id BIGINT FK`, `next_due_at TIMESTAMPTZ` | 결과는 `VALID`, `INVALID`, `UNKNOWN`, `UNCHECKED`. 지원하지 않는 유형이나 원문 재취득 실패는 사유를 기록한다. 최근 시도와 마지막 성공 판정은 이력에서 각각 조회하며 실패가 과거 성공 시각을 바꾸지 않는다. |
| `secret_assessments` | `assessment_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `run_id BIGINT FK`, `policy_version VARCHAR(80) NOT NULL`, `rule_id VARCHAR(20) NOT NULL`, `assessment_kind VARCHAR(10) NOT NULL`, `exposure_status VARCHAR(10) NOT NULL`, `exposure_range VARCHAR(10) NOT NULL`, `credential_state VARCHAR(20) NOT NULL`, `impact VARCHAR(10) NOT NULL`, `abuse VARCHAR(20) NOT NULL`, `validity VARCHAR(10) NOT NULL`, `severity VARCHAR(10)`, `missing_reason VARCHAR(120)`, `mapping_version VARCHAR(80)`, `assessed_at TIMESTAMPTZ NOT NULL`, `approved_by_user_id BIGINT FK`, `supersedes_assessment_id BIGINT FK` | `assessment_kind`는 `AUTO`, `MANUAL`; 등급은 `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `UNASSESSED` 또는 정상 보관의 NULL. `impact`는 `ADMIN`, `IMPORTANT`, `LIMITED`, `MINIMAL`, `UNKNOWN`. 수동 하향에는 승인자와 반증 근거가 필요하다. 과거 평가를 UPDATE하지 않는다. |
| `secret_assessment_evidence` | `assessment_id BIGINT NOT NULL FK`, `evidence_id BIGINT NOT NULL FK`, `input_name VARCHAR(40) NOT NULL`, 복합 PK | 평가 입력마다 증거와 관측 시각을 `secret_evidence`에 연결한다. 없는 근거는 `missing_reason`으로 남긴다. |

`secret_credential_sets.lifecycle_evidence_id`는 `secret_evidence` 생성 뒤 FK를 추가한다. 발급자와 유형은 수집 플랫폼과 독립적이다. 예를 들어 Notion 문서에 있는 AWS 키는 `secret_connections.provider=NOTION`, `secret_credential_sets.issuer=AWS`다. AWS Access Key ID처럼 공개 식별자가 있어도 비밀 구성요소가 부족하면 `CANDIDATE`로 남긴다. 발급처가 목록으로 존재를 확인했지만 값을 반환하지 않으면 `METADATA_ONLY`이며 이는 유효성 `VALID`의 증거가 아니다. 원문 없이 확인된 플랫폼 목록과 발견 값을 동일한 키로 자동 병합하지 않는다. 마지막 사용 시각, 확인 권한과 자원은 출처와 관측 시각을 잃지 않도록 `secret_evidence`에 저장한다.

### 3.3. 대응과 변경 이력

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_response_cases` | `case_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `previous_case_id BIGINT FK`, `is_current BOOLEAN NOT NULL`, `disposition VARCHAR(20) NOT NULL`, `action_status VARCHAR(30) NOT NULL`, `review_status VARCHAR(20) NOT NULL`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT FK`, `version BIGINT NOT NULL`, `opened_at`, `updated_at TIMESTAMPTZ NOT NULL`, `resolved_at`, `closed_at TIMESTAMPTZ` | `disposition`은 `IN_SCOPE`, `FALSE_POSITIVE`, `OUT_OF_SCOPE`. 조치는 `OPEN`, `IN_PROGRESS`, `REVALIDATION_PENDING`, `RESOLVED`; 검토는 `UNREVIEWED`, `REVIEWED`. `RISK_ACCEPTED`는 조치 상태가 아니라 아래 예외에서 계산한다. 키마다 `is_current=true` 최대 한 건. 재발 시 이전 건을 false로 바꾸고 새 건을 연다. |
| `secret_case_occurrences` | `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT NOT NULL FK`, `response_status VARCHAR(30) NOT NULL`, `linked_at TIMESTAMPTZ NOT NULL`, `action_evidence_id BIGINT FK`, `revalidation_evidence_id BIGINT FK`, `resolved_at TIMESTAMPTZ`, 복합 PK | 위치 상태는 `OPEN`, `IN_PROGRESS`, `REVALIDATION_PENDING`, `RESOLVED`. 위치가 해당 건의 키에 속해야 한다. 일부 위치 해결이 다른 위치를 해결하지 않는다. |
| `secret_assignment_history` | `assignment_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `from_team`, `to_team VARCHAR(100)`, `from_user_id`, `to_user_id BIGINT FK`, `reason VARCHAR(500) NOT NULL`, `changed_by_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 현재 담당자는 `secret_response_cases`에 두고 변경 이력을 추가 기록한다. 서버가 인증 사용자 ID를 채운다. |
| `secret_case_reviews` | `review_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT FK`, `decision VARCHAR(30) NOT NULL`, `reason VARCHAR(500) NOT NULL`, `reviewed_by_user_id BIGINT NOT NULL FK`, `reviewed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 결정은 `CONFIRMED`, `FALSE_POSITIVE`, `OUT_OF_SCOPE`, `NEEDS_MORE_INFO`. 위치별 판단을 연결한다. 오탐과 대상 제외는 위험 수용과 구분한다. |
| `secret_action_records` | `action_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT FK`, `action_type VARCHAR(40) NOT NULL`, `performed_at TIMESTAMPTZ NOT NULL`, `recorded_at TIMESTAMPTZ NOT NULL`, `performed_by_label VARCHAR(200) NOT NULL`, `recorded_by_user_id BIGINT NOT NULL FK`, `evidence_ref VARCHAR(500)`, `safe_note TEXT` | 실제 외부 작업을 사람이 기록한다. `performed_at`은 서버 기록 시각과 다르다. 제품이 교체나 폐기를 실행했다는 플래그는 두지 않는다. |
| `secret_checklist_items` | `item_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `guide_version VARCHAR(80) NOT NULL`, `item_code VARCHAR(80) NOT NULL`, `status VARCHAR(20) NOT NULL`, `completed_at TIMESTAMPTZ`, `completed_by_user_id BIGINT FK`, `evidence_ref VARCHAR(500)`, `updated_at TIMESTAMPTZ NOT NULL` | `UQ(case_id, guide_version, item_code)`. `status`는 `TODO`, `DONE`, `NOT_APPLICABLE`. 체크 완료만으로 대응 건을 해결하지 않는다. |
| `secret_exceptions` | `exception_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `reason VARCHAR(1000) NOT NULL`, `approved_by_user_id BIGINT NOT NULL FK`, `approved_at TIMESTAMPTZ NOT NULL`, `expires_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ`, `revoked_by_user_id BIGINT FK`, `revoke_reason VARCHAR(500)` | `CHECK(expires_at > approved_at)`. 취소와 만료를 삭제 대신 상태로 보존한다. 승인 권한은 서버가 검사한다. |
| `secret_exception_occurrences` | `exception_id BIGINT NOT NULL FK`, `occurrence_id BIGINT NOT NULL FK`, 복합 PK | 승인 당시의 위치 범위를 고정한다. 나중에 새 위치가 발견돼도 예외 범위에 자동 포함되지 않는다. |
| `secret_role_grants` | `grant_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `role VARCHAR(30) NOT NULL`, `connection_id BIGINT FK`, `granted_by_user_id BIGINT NOT NULL FK`, `granted_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ` | 역할은 `SECURITY`, `REMEDIATOR`, `OPERATIONS`. 연결 범위를 비우면 전체 범위다. 기존 `dashboard_users.role=ADMIN`과 별도로 서버가 대상 연결 범위까지 검사한다. 활성 중복 권한 부여를 막는다. |
| `secret_team_memberships` | `membership_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `team_name VARCHAR(100) NOT NULL`, `added_by_user_id BIGINT NOT NULL FK`, `added_at TIMESTAMPTZ NOT NULL`, `removed_at TIMESTAMPTZ` | 팀 담당자 지정 권한 확인에 사용한다. 팀 문자열만으로 소속을 추정하지 않는다. 활성 중복 소속을 막는다. |
| `secret_audit_events` | `audit_id BIGINT PK`, `request_id VARCHAR(100)`, `run_id BIGINT FK`, `actor_user_id BIGINT FK`, `actor_kind VARCHAR(20) NOT NULL`, `entity_type VARCHAR(50) NOT NULL`, `entity_id BIGINT NOT NULL`, `event_type VARCHAR(80) NOT NULL`, `before_safe JSONB`, `after_safe JSONB`, `occurred_at TIMESTAMPTZ NOT NULL` | API 변경, 워커 상태, 실패와 승인 근거를 추가 전용으로 남긴다. 비밀, 인증 헤더, 전체 도구 출력을 JSON에 복사하지 않는다. 보존 및 접근 권한은 별도 설정한다. |

`expectedVersion` 변경 요청은 `UPDATE secret_response_cases ... WHERE case_id=:id AND version=:expectedVersion`의 영향 행 수가 1인지 확인하고 같은 트랜잭션에서 버전 증가, 자식 기록, 감사 이벤트를 저장한다. 0행이면 409를 반환한다. 해결 전이는 모든 `ABNORMAL` 위치의 해결 증거와 외부 조치 이후 재확인 증거를 검사한다. 검증 실패, 체크리스트 완료, 예외 승인만으로 `RESOLVED`로 바꾸지 않는다.

예외 승인 시에는 대상 위치가 해당 대응 건에 속하는지, 중복된 활성 승인이 없는지, 승인자에게 해당 연결 범위 권한이 있는지를 같은 트랜잭션에서 검사한다. 예외 만료나 취소 후에는 해당 위치를 조치 상태로 되돌리고 새 스냅샷을 만든다. 감사 행은 UPDATE 및 DELETE를 일반 API 계정에 허용하지 않는다.

오탐이나 대상 제외 판단은 검토 근거를 남긴다. 일부 위치만 제외되면 다른 비정상 노출 위치의 대응을 계속한다. 모든 위치가 오탐 또는 대상 제외로 확정되면 대응 건의 `disposition`을 해당 결과로 바꾸고 `is_current=false`, `closed_at`을 기록해 위험 화면에서 제외한다. 이를 `RESOLVED`나 `RISK_ACCEPTED`로 표시하지 않고 이력에서 조회한다.

### 3.4. 일관된 화면 스냅샷

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_query_snapshots` | `snapshot_id UUID PK`, `scope VARCHAR(20) NOT NULL`, `as_of TIMESTAMPTZ NOT NULL`, `scope_version_digest VARCHAR(100) NOT NULL`, `partial_coverage BOOLEAN NOT NULL`, `missing_sources JSONB NOT NULL`, `created_at TIMESTAMPTZ NOT NULL`, `expires_at TIMESTAMPTZ NOT NULL` | `scope`는 `CURRENT_CASES`, `INVENTORY`, `HISTORY`. 하나의 DB 반복 읽기 트랜잭션에서 아래 행과 함께 생성한다. 누락 범위와 사유도 스냅샷에 고정한다. 만료된 ID는 409다. |
| `secret_snapshot_rows` | `snapshot_id UUID NOT NULL FK`, `subject_id BIGINT NOT NULL`, `credential_set_id BIGINT NOT NULL`, `action_bucket VARCHAR(30)`, `disposition VARCHAR(20)`, `review_status VARCHAR(20)`, `severity VARCHAR(10)`, `validity VARCHAR(10) NOT NULL`, `lifecycle_status VARCHAR(20) NOT NULL`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `inventory_source_provider VARCHAR(20)`, `first_detected_at TIMESTAMPTZ`, 복합 PK `(snapshot_id, subject_id)` | `subject_id`는 범위에 따라 대응 건 또는 키 ID다. 네 조치 단계와 별도 예외 중 하나를 `action_bucket`에 확정한다. 인벤토리의 만료와 폐기, 이력의 오탐과 대상 제외도 구분한다. 같은 키가 여러 소스에 있어도 행은 하나다. 목록, 카드, 매트릭스가 이 행만 읽는다. |
| `secret_snapshot_row_locations` | `snapshot_id UUID NOT NULL`, `subject_id BIGINT NOT NULL`, `occurrence_id BIGINT NOT NULL FK`, `source_provider VARCHAR(20) NOT NULL`, `environment VARCHAR(10) NOT NULL`, `linked_at TIMESTAMPTZ NOT NULL`, 복합 PK `(snapshot_id, subject_id, occurrence_id)` | 복합 FK `(snapshot_id, subject_id)`가 `secret_snapshot_rows`를 참조한다. 소스와 환경은 같은 위치 행에서 함께 필터한다. 위치가 여러 개여도 대상 행을 한 번만 센다. |

`CURRENT_CASES`는 `is_current=true`이고 `disposition=IN_SCOPE`인 대응 건마다 한 행을 만든다. `INVENTORY`는 키마다 한 행이다. `HISTORY`는 각 과거 대응 건을 따로 만든다. 범위, 소스, 환경, 발급자, 유효성, 심각도, 담당자 필터를 먼저 동일하게 적용한 행 집합에서 카드, 매트릭스, 목록을 계산한다. 소스와 환경을 동시에 고른 경우에는 `secret_snapshot_row_locations`의 같은 위치가 두 조건을 모두 만족해야 한다. 위치가 전혀 없는 발급처 목록 키는 인벤토리에서만 `inventory_source_provider`로 필터하며 환경은 UNKNOWN이다. 한 건에 해당하는 미해결 위치가 모두 유효한 승인 예외 범위 안에 있을 때만 `action_bucket=RISK_ACCEPTED`다. 대표 심각도는 미해결 위치의 최신 확인 등급 중 최고 등급이며 전부 미평가이면 `UNASSESSED`다. 미평가 위치 수는 상세와 요약의 별도 필드로 제공한다. 지난 `VALID` 뒤에 외부 조치가 있으면 스냅샷의 유효성은 최신 시도 결과와 조치 이후 재검증 대기를 함께 설명할 수 있도록 API가 마지막 성공 시각과 조치 시각을 별도 조회한다.

## 4. 인덱스와 제약 구현

다음 인덱스와 CHECK는 마이그레이션의 필수 항목이다. 표의 FK는 기본적으로 `ON DELETE RESTRICT`를 사용해 근거가 남은 부모 행의 삭제를 막는다. 스냅샷 만료 행만 보존 정책에 따라 자식부터 삭제한다.

```sql
CREATE UNIQUE INDEX uq_secret_fingerprint
  ON secret_credential_sets (issuer, credential_type, fingerprint_key_version, fingerprint)
  WHERE fingerprint IS NOT NULL AND component_state = 'CONFIRMED';

CREATE UNIQUE INDEX uq_secret_current_case
  ON secret_response_cases (credential_set_id)
  WHERE is_current;

CREATE UNIQUE INDEX uq_secret_manual_idempotency
  ON secret_jobs (requested_by_user_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;

CREATE UNIQUE INDEX uq_secret_scheduled_scan
  ON secret_jobs (connection_id, scheduled_at)
  WHERE kind = 'FULL_SCAN' AND requested_by_user_id IS NULL;

CREATE INDEX idx_secret_jobs_due
  ON secret_jobs (available_at, job_id)
  WHERE status = 'PENDING';

CREATE INDEX idx_secret_occurrences_connection_presence
  ON secret_occurrences (connection_id, scope_id, presence_status, last_seen_at DESC);

CREATE INDEX idx_secret_validation_latest
  ON secret_validation_attempts (credential_set_id, completed_at DESC, attempt_id DESC);

CREATE INDEX idx_secret_assessment_latest
  ON secret_assessments (occurrence_id, assessed_at DESC, assessment_id DESC);

CREATE INDEX idx_secret_case_list
  ON secret_response_cases (is_current, action_status, opened_at DESC, case_id DESC);

CREATE INDEX idx_secret_snapshot_bucket
  ON secret_snapshot_rows (snapshot_id, action_bucket, severity, validity);

CREATE INDEX idx_secret_snapshot_locations
  ON secret_snapshot_row_locations (snapshot_id, source_provider, environment, subject_id);

CREATE UNIQUE INDEX uq_secret_active_scope
  ON secret_connection_scopes (connection_id, scope_key)
  WHERE enabled;

CREATE UNIQUE INDEX uq_secret_active_role_grant
  ON secret_role_grants (user_id, role, COALESCE(connection_id, 0))
  WHERE revoked_at IS NULL;

CREATE UNIQUE INDEX uq_secret_active_membership
  ON secret_team_memberships (user_id, team_name)
  WHERE removed_at IS NULL;
```

추가 `CHECK`는 `component_state='CONFIRMED'`일 때 `fingerprint`와 `fingerprint_key_version`의 동시 존재, `expiry_policy='FIXED'`일 때 `expires_at`의 존재, `severity='UNASSESSED'`일 때 `missing_reason`의 존재, `RESOLVED` 건의 `resolved_at` 존재, 양수 `version`, 정기 작업의 `scheduled_at` 존재, 검사 시작 전 완료 시각 금지, 예외 기간의 순서를 포함한다. `scope_id`와 `connection_id`, `occurrence_id`와 `credential_set_id`, `case_id`와 `occurrence_id`, `run_id`와 `tool_run_id`의 일치는 복합 FK 또는 동일 트랜잭션의 명시적 검증으로 강제한다. 연결된 `assets`의 provider와 환경을 무조건 복사하지 않는다. 두 값의 의미가 다른 경우 Secret 위치의 분류 근거를 우선한다.

| FK 열 | 참조 대상 |
|---|---|
| 모든 `connection_id`, `scope_id`, `run_id`, `run_scope_id`, `tool_run_id`, `job_id` | 각각 `secret_connections`, `secret_connection_scopes`, `secret_scan_runs`, `secret_run_scopes`, `secret_tool_runs`, `secret_jobs`의 PK |
| 모든 `credential_set_id`, `occurrence_id`, `case_id`, `assessment_id`, `exception_id`, `evidence_id`, `classification_id` | 각각 `secret_credential_sets`, `secret_occurrences`, `secret_response_cases`, `secret_assessments`, `secret_exceptions`, `secret_evidence`, `secret_asset_classifications`의 PK |
| `previous_case_id`, `supersedes_assessment_id`, `lifecycle_evidence_id` | 각각 `secret_response_cases.case_id`, `secret_assessments.assessment_id`, `secret_evidence.evidence_id` |
| `asset_id`, 모든 `*_user_id` | 각각 기존 `assets.asset_id`, `dashboard_users.user_id` |
| `secret_snapshot_rows.snapshot_id`, `secret_snapshot_row_locations.occurrence_id` | 각각 `secret_query_snapshots.snapshot_id`, `secret_occurrences.occurrence_id` |

`secret_query_snapshots`의 생성은 `REPEATABLE READ`에서 원본을 읽고 스냅샷 헤더와 행을 한 번에 커밋한다. `subject_id`는 범위에 따라 다른 부모 테이블을 가리키므로 FK 대신 생성 트랜잭션의 대상 조회로 무결성을 보장한다. 범위마다 별도 스냅샷을 쓰고 스냅샷 ID를 다른 범위로 재사용하지 않는다.

## 5. 적재와 상태 변경 규칙

1. 연결과 승인 범위를 등록할 때 범위 버전, 수집 가능한 콘텐츠, 분류 근거와 비밀 저장소 참조를 기록한다.
2. 정기 작업을 예약하고 워커가 선점한 뒤 `secret_scan_runs`를 만든다. 범위별 성공, 실패, 미지원과 엔진별 성공, 실패를 독립적으로 확정한다.
3. 도구 출력은 워커 메모리에서 정규화한다. 키 원문은 HMAC 계산과 승인된 검증에만 쓰고 DB나 큐에 싣지 않는다. 불완전 후보는 다른 후보와 합치지 않는다.
4. 같은 키와 원천 위치는 기존 `secret_occurrences`를 갱신하고 새 발견은 추가한다. 엔진 결과는 `secret_findings`에 별도 기록한다. 완료된 범위에서만 미발견 위치를 `MISSING`으로 갱신한다. 일부 실패 범위의 기존 위치는 `UNKNOWN`으로 남긴다.
5. 유형별 검증은 자격 증명당 실행 주기 내 한 번만 수행하고 `secret_validation_attempts`에 기록한다. 지원되는 사용 정보와 영향 근거는 `secret_evidence`에 따로 저장한다. 최신 시도와 마지막 성공 판정은 다른 조회다.
6. 각 비정상 노출 위치에 `severity-v2` 평가를 추가한다. 자동 판정과 승인된 수동 변경, 사용한 근거를 분리한다. 신규 발견의 만료 또는 폐기가 근거로 확인되면 인벤토리에 보존하고 신규 대응 건은 만들지 않는다.
7. 대응 대상은 현재 건에 위치를 연결한다. 해결 뒤 재노출이면 기존 건을 이력으로 닫고 새 건을 만든다. 할당, 검토, 작업, 체크리스트, 예외, 해결 변경은 버전 확인과 감사 이벤트를 같은 트랜잭션에 쓴다.
8. 조회 스냅샷은 모든 단계의 확정 데이터를 기준으로 생성한다. 실패 범위가 있으면 `partialCoverage`와 `missingSources`를 생성 시점에 계산해 스냅샷에 고정한다. 실패를 0건 정상 수집으로 치환하지 않는다.

## 6. 보안, 보존과 마이그레이션

API DB 계정은 Secret 조회 및 대응 변경 테이블에 필요한 권한만 갖고, 워커 DB 계정은 작업 선점과 탐지 및 검사 적재 권한을 갖는다. 연결용 비밀은 DB 밖의 비밀 저장소에 두며 `auth_ref`만 저장한다. HMAC 보호키는 DB 밖에서 관리하고 버전 교체 시 이전 버전 지문을 안전하게 재계산할 수 있는 시점까지 병행 조회한다. 원천에서 키를 다시 읽을 수 없으면 키 동일성 재계산과 재검증이 불가능하므로 임의 병합하지 않는다. 데이터, 백업, 감사 로그의 접근 권한과 보존 기간은 범주별 운영 설정으로 확정한다. 보존 기간이 정해지지 않은 범주는 자동 삭제하지 않는다.

마이그레이션 순서는 연결 및 작업, 키 및 발견 위치, 근거 및 검사와 평가, 대응과 역할, 스냅샷, 인덱스와 권한 순이다. 기존 V1 데이터는 신규 Secret 테이블로 일괄 이관하지 않는다. 샘플 `detection_events`를 실탐지로 간주할 수 없고, 필수 환경과 심각도를 Secret의 UNKNOWN 또는 UNASSESSED에 안전하게 대응시킬 수 없기 때문이다. 10/13 main의 로컬 샘플 적재기는 전체 삭제 대신 조건부 정리를 수행한다. 기존 `credential_id` 연결 이벤트도 삭제하므로 이전 대상 기록과 정리 SQL을 대조하고 비샘플 데이터 보존을 일괄 가정하지 않는다. 시작 시 외부 스캔은 여전히 남아 있어 [실행 정책](security-and-runtime.md)에 맞게 제거해야 한다. 새 `/api/v1/secrets` 조회를 먼저 구현하고 통합 현황의 새 버전은 위의 API 합성 규칙을 사용한다. 기존 점수의 제거와 응답 필드 전환은 [API 계약](api-contract.md)에 맞춰 클라이언트와 함께 배포한다.

공유 `V2__create_credentials.sql`은 유지하고 미공유 스케줄러 마이그레이션은 V3로 변경한다. 기존 개발 DB를 보존한 채 별도 개발 DB에 전체 마이그레이션 적용을 검증하고 필요한 데이터를 이전한다. 기존 DB에는 스케줄러 V2가 이미 적용돼 있어 파일 이름만 바꿔 재실행하지 않는다. 이전 대상, 적용 순서, 동일성·FK·시퀀스 검증과 전환 절차를 정한 뒤 실행하며 이후 번호는 공통 DB 담당자가 조율한다. 이 문서 분리 작업에서는 번호 변경이나 DB 이전을 수행하지 않는다.

구현 검증은 Flyway 빈 DB 적용과 기존 V1 위 적용, 잘못된 FK와 중복 현재 건 및 중복 작업의 거부, 6시간 정기와 수동 실행의 독립성, 부분 실패 후 위치 유지, 동일 키의 다중 위치와 다중 엔진 중복 제거, 최신 실패와 지난 성공 보존, 부분 예외 집계, 스냅샷의 카드 및 목록 수 일치, 원문이 DB와 로그에 남지 않는지를 확인한다.
