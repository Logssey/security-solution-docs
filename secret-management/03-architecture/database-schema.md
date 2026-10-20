# Secret 데이터베이스 스키마

기준: [PRD](../01-requirements/PRD.md), [데이터 계약](data-contract.md), [API 계약](api-contract.md), [판정 정책](../02-functional-design/triage-policy.md), [대응 정책](../02-functional-design/response-policy.md), [심각도 정책](../02-functional-design/severity-policy.md)

PostgreSQL에 저장하는 Secret 사실 원본, 관계, 제약과 조회 스냅샷의 명세다. 수집·탐지·판정·외부 조치 기록·공급자 확인을 독립 사실로 보존한다. 제품이 따르는 스키마를 정의하며 적용·병합·배포 현황을 나타내지 않는다.

## 1. 공통 데이터와 접근 경계

| 공통 데이터 | Secret의 참조 방식 |
|---|---|
| `dashboard_users` | 담당·제안·승인·요청·감사 행위자 FK. 공통 세션 인증을 사용하며 ADMIN과 Secret 역할은 별개다. 계정 생성·비활성화는 공통 계정 관리 경계에 둔다. |
| `assets` | 명확한 자산 대응 근거가 있을 때 선택 연결. Notion 또는 환경 UNKNOWN을 표현하려고 가짜 자산이나 PROD 기본값을 만들지 않는다. |
| `detection_events` | 일반 탐지 사실 원본. Secret의 위치·미평가·재검사·예외는 전용 모델에 보존하고 통합 API에서 영역·ID·단위를 명시해 합성한다. 일반 탐지에 Secret을 복제 적재하지 않는다. |

테이블은 secret_ 접두사, PK는 별도 표기가 없으면 BIGINT GENERATED ALWAYS AS IDENTITY다. 모든 시각은 TIMESTAMPTZ, 사용자 참조는 dashboard_users의 BIGINT FK다. 표에서 타입을 묶어 적은 연속 컬럼에는 각각 같은 타입·NULL 조건을 적용한다. NOT NULL 표기가 없는 컬럼은 NULL을 허용한다. 상태는 VARCHAR와 CHECK, 공급자 키 유형·룰은 버전 등록부로 검증한다. 운영 DB 서버·배포 위치는 공통 데이터 저장 정책을 따른다.

키 원문·인증 토큰·원천 본문·전체 공급자 응답은 일반 DB·큐·스냅샷·감사·백업에 넣지 않는다. auth_ref는 DB 밖 비밀 참조다. approved_ref와 재취득 원천 참조는 민감한 메타데이터로 제한 접근하며 일반 응답에는 승인한 안전한 별칭만 제공한다.

## 2. 관계와 집계 단위

- Connection → 승인 Scope revision → Job → ScanRun → RunScope → ToolRun으로 실행 사실을 연결한다.
- CredentialSet 하나에 Occurrence 여러 개, 각 위치에 Finding·판정·평가 이력을 보존한다. 발급자와 수집 플랫폼은 독립이다.
- 유효 ABNORMAL 결정은 같은 키·출처의 현재 ResponseCase 생성 또는 위치 연결과 원자 확정한다.
- 대응 건은 위치별 담당·조치 주기·외부 기록·재검사를 갖고 키 수명주기·해결·교정 종료·예외 근거는 별도로 보존한다.
- 현황의 건·위치·키·범위·경고는 다른 단위다. Mixed 공통 합계 없이 caseSummary만 ResponseCase 분포를 사용한다.

## 3. 테이블 명세

### 3.1. 연결·승인·정책

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_connections` | `connection_id BIGINT PK`, `provider VARCHAR(20) NOT NULL`, `display_name VARCHAR(200) NOT NULL`, `auth_ref VARCHAR(300) NOT NULL`, `status VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `next_due_at TIMESTAMPTZ`, `last_collection_success_at TIMESTAMPTZ`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL` | 플랫폼은 AWS·KUBERNETES·GITHUB·NOTION, 상태는 ACTIVE·PAUSED·ERROR다. auth_ref는 DB 밖 연결 비밀의 불투명 참조다. 범위 변경은 scope_version을 증가시킨다. last_collection_success_at은 같은 전체 실행에서 모든 필수 범위·단계가 성공한 시각만 가리킨다. |
| `secret_connection_scopes` | `scope_id BIGINT PK`, `connection_id BIGINT NOT NULL FK`, `scope_key VARCHAR(300) NOT NULL`, `revision BIGINT NOT NULL`, `content_type VARCHAR(80) NOT NULL`, `approved_ref VARCHAR(500) NOT NULL`, `history_limit VARCHAR(100)`, `enabled BOOLEAN NOT NULL`, `response_origin VARCHAR(20) NOT NULL`, `response_origin_version BIGINT NOT NULL`, `classification_version VARCHAR(80)`, `safe_display_alias VARCHAR(80)`, `display_changed_by_user_id BIGINT FK`, `display_changed_at TIMESTAMPTZ`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL` | UQ(connection_id,scope_key,revision), 활성 scope_key당 revision 하나. 출처는 UNCLASSIFIED·PRODUCTION·TEST다. 승인 대상·브랜치·콘텐츠·이력 한도는 불변 revision에 고정한다. 별칭은 승인 참조에서 자동 추출하지 않으며 없으면 범위 #ID를 쓴다. 수집 허가·출처·보관 정책은 별개다. |
| `secret_connection_probes` | `probe_id BIGINT PK`, `connection_id BIGINT NOT NULL FK`, `check_kind VARCHAR(40) NOT NULL`, `requested_by_user_id BIGINT NOT NULL FK`, `request_id VARCHAR(120) NOT NULL`, `status VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80)`, `attempted_at`, `completed_at TIMESTAMPTZ NOT NULL` | check_kind=IDENTITY_OR_INSTALLATION. 결과 CONNECTED·FAILED·UNAVAILABLE, completed_at>=attempted_at. 안전한 사유만 저장하고 연결 활성 상태·전체 수집 성공을 변경하지 않는다. 공급자 신원·토큰·오류 본문 금지. |
| `secret_storage_policies` | `policy_id`, `version BIGINT NOT NULL`, `status VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `created_by_user_id BIGINT NOT NULL FK`, `approved_by_user_id BIGINT FK`, `created_at`, `valid_from TIMESTAMPTZ NOT NULL`, `valid_until`, `approved_at`, `retired_at TIMESTAMPTZ`, 복합 PK `(policy_id,version)` | DRAFT·ACTIVE·RETIRED. 승인자는 작성자와 다르고 ACTIVE에는 승인자·시각 필요. 종료가 있으면 시작보다 뒤다. 승인 버전은 불변이며 변경·폐기는 새 버전/불변 승인 이력으로 남긴다. 상충 허용·금지 규칙 활성화 거부. |
| `secret_storage_policy_rules` | `rule_id BIGINT PK`, `policy_id`, `policy_version BIGINT NOT NULL`, `effect VARCHAR(10) NOT NULL`, `connection_id`, `scope_id BIGINT FK`, `content_type VARCHAR(80) NOT NULL`, `branch_condition VARCHAR(200)`, `protected_path_condition VARCHAR(500)`, `created_at TIMESTAMPTZ NOT NULL` | 정책 버전 복합 FK. effect=ALLOW·DENY. 범위·콘텐츠·브랜치·정확한 경로/접두사 적용 검사. 민감 경로는 제한된 정책 접근 경계에서만 읽는다. 만료·불일치·충돌은 UNKNOWN이다. |
| `secret_scope_classification_proposals` | `proposal_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `expected_scope_version BIGINT NOT NULL`, `proposed_origin VARCHAR(20) NOT NULL`, `policy_id`, `policy_version BIGINT`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `proposed_by_user_id BIGINT NOT NULL FK`, `status VARCHAR(30) NOT NULL`, `version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 범위당 PENDING_APPROVAL 최대 하나. 정책 ID·버전은 함께 NULL 또는 활성 버전 FK. 정책 없음은 POLICY_PENDING이며 보관 UNKNOWN 유지. TEST 신규 승인·PRODUCTION→TEST 교정은 제안자와 다른 승인자 필요. |
| `secret_scope_origin_decisions` | `decision_id BIGINT PK`, `proposal_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `from_origin`, `to_origin VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `decision VARCHAR(20) NOT NULL`, `actor_user_id BIGINT NOT NULL FK`, `decided_at TIMESTAMPTZ NOT NULL`, `version BIGINT NOT NULL` | UQ(scope_id,version). 승인·거절·변경 전후 추가 전용. 승인 시 범위 버전·정책 바인딩·영향 대상 감사·집계 무효화 원자 확정. 거절은 활성 분류 유지. 과거 건의 출처를 덮어쓰지 않는다. |
| `secret_scope_policy_bindings` | `binding_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `scope_revision BIGINT NOT NULL`, `policy_id`, `policy_version BIGINT NOT NULL`, `classification_decision_id BIGINT NOT NULL FK`, `effective_from TIMESTAMPTZ NOT NULL`, `effective_until TIMESTAMPTZ` | 범위 revision·정책 버전 복합 FK. 유효 구간이 겹치는 활성 바인딩 금지. 실행·판정은 당시 버전 참조. 과거 바인딩 의미를 수정하지 않는다. |
| `secret_asset_classifications` | `classification_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `resource_ref VARCHAR(500) NOT NULL`, `environment VARCHAR(10) NOT NULL`, `criticality VARCHAR(20) NOT NULL`, `sensitive_data BOOLEAN`, `version BIGINT NOT NULL`, `evidence_ref VARCHAR(500)`, `classified_at TIMESTAMPTZ NOT NULL`, `classified_by_user_id BIGINT FK` | `UQ(scope_id, resource_ref, version)`. 환경은 `PROD`, `DEV`, `UNKNOWN`; 중요도는 `CRITICAL`, `IMPORTANT`, `LIMITED`, `MINIMAL`, `UNKNOWN`. 이름만 보고 분류하지 않는다. 과거 버전을 보존한다. |

### 3.2. 작업·실행·범위 결과

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_jobs` | `job_id BIGINT PK`, `kind VARCHAR(20) NOT NULL`, `connection_id BIGINT NOT NULL FK`, `requested_by_user_id BIGINT FK`, `idempotency_key VARCHAR(120)`, `scheduled_at TIMESTAMPTZ NOT NULL`, `available_at TIMESTAMPTZ NOT NULL`, `attempt_no INTEGER NOT NULL`, `max_attempts INTEGER NOT NULL`, `status VARCHAR(20) NOT NULL`, `lease_owner VARCHAR(120)`, `lease_until TIMESTAMPTZ`, `last_error_code VARCHAR(80)`, `created_at`, `finished_at TIMESTAMPTZ` | kind=FULL_SCAN·REVALIDATE. 상태 PENDING·LEASED·SUCCEEDED·FAILED·CANCELLED. 같은 연결의 활성 전체 스캔/재검사는 합쳐 최대 하나. attempt_no>=0, max_attempts>0. 대상은 case_revalidations로 연결. 만료 임대는 재시도 한도 내 재예약한다. |
| `secret_scan_runs` | `run_id BIGINT PK`, `job_id BIGINT NOT NULL UQ FK`, `connection_id BIGINT NOT NULL FK`, `trigger_type VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `scheduled_at`, `requested_at TIMESTAMPTZ NOT NULL`, `started_at`, `finished_at TIMESTAMPTZ`, `collection_status`, `detection_status`, `validation_status VARCHAR(20) NOT NULL`, `result_status VARCHAR(20) NOT NULL`, `failure_code VARCHAR(80)`, `created_at TIMESTAMPTZ NOT NULL`; `connection_display_snapshot VARCHAR(200) NOT NULL` | trigger_type=SCHEDULED·MANUAL·REVALIDATION. 단계는 PENDING·RUNNING·SUCCEEDED·PARTIAL·FAILED·SKIPPED, 실행 결과는 이들과 CANCELLED. 실행 당시 연결·범위 표시 보존. 취소 때 완료 범위 유지, 미착수 SKIPPED/JOB_CANCELLED, 중단 FAILED/JOB_CANCELLED. 재검사는 전체 스캔 성공률에서 제외. |
| `secret_run_scopes` | `run_scope_id BIGINT PK`, `run_id`, `scope_id BIGINT NOT NULL FK`, `scope_revision BIGINT NOT NULL`, `safe_display_alias VARCHAR(80)`, `content_type VARCHAR(80) NOT NULL`, `region VARCHAR(80)`, `required_stage_config_hash VARCHAR(100)`, `stage_requirements JSONB`, `collection_status`, `detection_status`, `validation_status VARCHAR(20) NOT NULL`, `scope_outcome VARCHAR(20)`, `coverage_complete BOOLEAN`, `object_count BIGINT`, `failure_code`, `skip_reason_code VARCHAR(80)`, `started_at`, `finished_at TIMESTAMPTZ` | UQ(run_id,scope_id), object_count>=0. 당시 revision·필수/선택/해당 없음·안전한 미실행 사유 고정. 필수 탐지/검증 실패는 수집 성공이어도 PARTIAL. 선택 비활성·메타데이터 해당 없음은 성공을 막지 않는다. 복원 불가 과거 구성은 NULL/LEGACY_UNAVAILABLE. 별칭은 원천 참조를 복제하지 않는다. |
| `secret_tool_runs` | `tool_run_id BIGINT PK`, `run_scope_id BIGINT NOT NULL FK`, `engine VARCHAR(20) NOT NULL`, `engine_version VARCHAR(100) NOT NULL`, `config_hash VARCHAR(100) NOT NULL`, `status VARCHAR(20) NOT NULL`, `finding_count BIGINT`, `failure_code VARCHAR(80)`, `started_at`, `finished_at TIMESTAMPTZ` | UQ(run_scope_id,engine), finding_count>=0. 등록된 엔진별 결과를 보존한다. 한 엔진 실패를 다른 엔진 0건으로 덮지 않으며 탐지기 자체 검증을 제품 VALID로 자동 채택하지 않는다. |

### 3.3. 키·위치·탐지·평가

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_credential_sets` | `credential_set_id BIGINT PK`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `component_state VARCHAR(20) NOT NULL`, `issuer_connection_id BIGINT FK`, `fingerprint BYTEA`, `candidate_locator BYTEA`, `fingerprint_key_version VARCHAR(40)`, `masked_label VARCHAR(120)`, `external_key_ref VARCHAR(300)`, `owner_ref VARCHAR(200)`, `purpose_label VARCHAR(200)`, `issued_at TIMESTAMPTZ`, `expiry_policy VARCHAR(10) NOT NULL`, `expires_at TIMESTAMPTZ`, `lifecycle_status VARCHAR(20) NOT NULL`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `created_at TIMESTAMPTZ NOT NULL`; `lifecycle_evidence_id BIGINT FK`, `first_seen_run_id BIGINT FK` | CONFIRMED는 보호키 HMAC fingerprint, CANDIDATE는 원천 위치를 포함한 HMAC candidate_locator, METADATA_ONLY는 값 지문 없이 보호된 목록 참조다. 마스킹·접두사 병합 금지. FIXED는 expires_at 필수. 최초 동일성 삽입 실행만 first_seen_run_id를 고정하고 구성 승격·새 위치로 덮지 않는다. 근거 불명 과거 행은 NULL. 메타데이터·재검사는 신규 값 후보 집계 제외. |
| `secret_occurrences` | `occurrence_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `connection_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `asset_id BIGINT FK`, `source_object_ref VARCHAR(500) NOT NULL`, `location_key VARCHAR(500) NOT NULL`, `latest_object_version VARCHAR(200)`, `environment VARCHAR(10) NOT NULL`, `exposure_status VARCHAR(10) NOT NULL`, `exposure_range VARCHAR(10) NOT NULL`, `presence_status VARCHAR(10) NOT NULL`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `last_seen_run_id BIGINT FK`, `last_complete_coverage_at TIMESTAMPTZ`, `exposure_assessment_version BIGINT NOT NULL`; `version BIGINT NOT NULL`, `latest_decision_id BIGINT FK`, `safe_location_alias VARCHAR(80)`, `last_comparable_run_scope_id`, `latest_attempt_run_scope_id BIGINT FK`, `last_successful_presence VARCHAR(10)`, `last_successful_observed_at TIMESTAMPTZ` | UQ(credential_set_id,connection_id,source_object_ref,location_key), scope는 같은 연결 소속. 환경 PROD·DEV·UNKNOWN, 노출 ABNORMAL·NORMAL·UNKNOWN, 범위 BROAD·BOUNDED·UNKNOWN, 존재 PRESENT·MISSING·UNKNOWN을 독립 보존한다. 실패는 마지막 성공·수동 판정을 지우지 않는다. 재취득 참조는 제한 접근하고 일반 표시는 승인 별칭 또는 위치 #ID다. |
| `secret_findings` | `finding_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `tool_run_id BIGINT NOT NULL FK`, `finding_key VARCHAR(100) NOT NULL`, `rule_id VARCHAR(120) NOT NULL`, `rule_version VARCHAR(100) NOT NULL`, `masked_context TEXT`, `detected_at TIMESTAMPTZ NOT NULL` | `UQ(tool_run_id, finding_key)`. 마스킹된 문맥만 저장한다. 동일 위치의 두 엔진 결과를 둘 다 남기되 위치를 두 번 만들지 않는다. `tool_run_id`와 `run_id`의 관계를 적재 트랜잭션에서 검증한다. |
| `secret_validation_attempts` | `attempt_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `job_id BIGINT NOT NULL FK`, `validator_id VARCHAR(120) NOT NULL`, `validator_version VARCHAR(80) NOT NULL`, `endpoint_class VARCHAR(80) NOT NULL`, `requested_at`, `completed_at TIMESTAMPTZ NOT NULL`, `provider_checked_at TIMESTAMPTZ`, `result VARCHAR(10) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `next_due_at TIMESTAMPTZ`; `evidence_id BIGINT FK` | (run_id,job_id) 복합 FK. VALID·INVALID·UNKNOWN·UNCHECKED와 안전한 사유. VALID·INVALID에는 실제 공급자 관측 시각 필요, UNCHECKED에는 없다. 최신 실패와 마지막 성공은 별도 조회. 원문·공급자 본문 저장 금지. |
| `secret_aws_access_key_observations` | `credential_set_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `key_status VARCHAR(10) NOT NULL`, `issued_at`, `last_used_at TIMESTAMPTZ`, `last_used_service VARCHAR(120)`, `last_used_region VARCHAR(80)`, `observed_at TIMESTAMPTZ NOT NULL` | PK(credential_set_id,run_id). 키 상태 ACTIVE·INACTIVE·EXPIRED·UNKNOWN. 목록 ACTIVE는 VALID·노출·권한의 증거가 아니며 기록 부재는 미사용이 아니다. 사용자명·키 ID 원문·Secret 값 금지. |
| `secret_evidence` | `evidence_id BIGINT PK`, `credential_set_id BIGINT FK`, `occurrence_id BIGINT FK`, `run_id BIGINT FK`, `kind VARCHAR(40) NOT NULL`, `observation_status VARCHAR(20) NOT NULL`, `source VARCHAR(80) NOT NULL`, `source_ref VARCHAR(500)`, `observed_at TIMESTAMPTZ`, `collected_at TIMESTAMPTZ NOT NULL`, `mapping_version VARCHAR(80)`, `classification_id BIGINT FK`, `safe_facts JSONB NOT NULL` | `CHECK(credential_set_id IS NOT NULL OR occurrence_id IS NOT NULL)`. 근거 종류는 `EXPOSURE`, `IDENTITY`, `PERMISSION`, `RESOURCE`, `USAGE`, `ABUSE`, `LIFECYCLE`, `REVALIDATION` 등이다. 상태는 `CONFIRMED`, `NO_RECORD`, `UNSUPPORTED`, `UNAVAILABLE`, `CONFLICT`. `NO_RECORD`는 미사용 또는 악용 없음의 확정이 아니다. `safe_facts`는 허용 필드만 서버가 생성한다. |
| `secret_assessments` | `assessment_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `run_id BIGINT FK`, `policy_version VARCHAR(80) NOT NULL`, `rule_id VARCHAR(20) NOT NULL`, `assessment_kind VARCHAR(10) NOT NULL`, `exposure_status VARCHAR(10) NOT NULL`, `exposure_range VARCHAR(10) NOT NULL`, `credential_state VARCHAR(20) NOT NULL`, `impact VARCHAR(10) NOT NULL`, `abuse VARCHAR(20) NOT NULL`, `validity VARCHAR(10) NOT NULL`, `severity VARCHAR(10)`, `missing_reason VARCHAR(120)`, `mapping_version VARCHAR(80)`, `assessed_at TIMESTAMPTZ NOT NULL`, `approved_by_user_id BIGINT FK`, `supersedes_assessment_id BIGINT FK` | `assessment_kind`는 `AUTO`, `MANUAL`; 등급은 `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `UNASSESSED` 또는 정상 보관의 NULL. `impact`는 `ADMIN`, `IMPORTANT`, `LIMITED`, `MINIMAL`, `UNKNOWN`. 수동 하향에는 승인자와 반증 근거가 필요하다. 과거 평가를 UPDATE하지 않는다. |
| `secret_assessment_evidence` | `assessment_id BIGINT NOT NULL FK`, `evidence_id BIGINT NOT NULL FK`, `input_name VARCHAR(40) NOT NULL`, 복합 PK | 평가 입력마다 증거와 관측 시각을 `secret_evidence`에 연결한다. 없는 근거는 `missing_reason`으로 남긴다. |
| `secret_exposure_assessments` | `assessment_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `finding_id`, `finding_run_id BIGINT NOT NULL`, `finding_rule_id VARCHAR(120) NOT NULL`, `finding_rule_version VARCHAR(100) NOT NULL`, `from_status`, `decision VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `scope_origin VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `policy_id`, `policy_version BIGINT`, `policy_result VARCHAR(10) NOT NULL`, `decision_source VARCHAR(20) NOT NULL`, `evidence_refs JSONB NOT NULL`, `observed_at TIMESTAMPTZ`, `actor_user_id BIGINT FK`, `automatic_policy_version VARCHAR(80)`, `assessed_at TIMESTAMPTZ NOT NULL`, `version BIGINT NOT NULL` | UQ(occurrence_id,version). HUMAN은 actor, POLICY는 자동 정책 버전 필요. 당시 Finding·룰·실행·정책·시각 스냅샷 보존. 자동 결정은 정확한 ALLOWED의 NORMAL만 허용. DENIED·UNKNOWN 자동 ABNORMAL 금지. Finding 재생성 후에도 판정 근거 유지. |
| `secret_finding_dispositions` | `disposition_id BIGINT PK`, `finding_id BIGINT NOT NULL`, `occurrence_id`, `scope_id BIGINT NOT NULL FK`, `object_version VARCHAR(200)`, `rule_version`, `engine_version VARCHAR(100) NOT NULL`, `protected_evidence_identity BYTEA`, `identity_key_version VARCHAR(40)`, `reason_code VARCHAR(80) NOT NULL`, `decision_source VARCHAR(30) NOT NULL`, `reused_from_disposition_id BIGINT FK`, `reviewed_by_user_id BIGINT FK`, `created_at TIMESTAMPTZ NOT NULL`, `reuse_expires_at`, `revoked_at TIMESTAMPTZ`, `is_current BOOLEAN NOT NULL` | Finding당 현재 교정 최대 하나. FALSE_POSITIVE는 자격 증명 아님이며 NORMAL·INVALID가 아니다. 같은 범위·객체 버전·위치·룰/엔진·후보 동일성이 모두 같은 재탐지만 최대 30일 EXACT_EVIDENCE_REUSE 행을 새로 쓴다. 근거 부재·만료·취소 시 재사용 금지. 원문 원천 참조를 재사용 색인에 넣지 않는다. |

### 3.4. 조사·원자 명령

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_investigation_tasks` | `task_id BIGINT PK`, `target_type VARCHAR(20) NOT NULL`, `occurrence_id`, `scope_id BIGINT FK`, `missing_reason_code VARCHAR(80) NOT NULL`, `creation_source VARCHAR(30) NOT NULL`, `assignee_user_id BIGINT FK`, `due_at TIMESTAMPTZ`, `status VARCHAR(20) NOT NULL`, `evidence_ref VARCHAR(120)`, `version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL`, `closed_at TIMESTAMPTZ`, `resolving_decision_id`, `restored_run_scope_id BIGINT FK` | OCCURRENCE·SCOPE와 해당 FK 배타 지정. HUMAN_MORE_INFO는 담당·기한 필수, AUTO_TEST_ALERT는 각각 nullable. 기본값 부재로 경고 누락 금지. 기한 NULL은 기한 초과 제외. 판정/범위 복구 근거 없이 닫지 않고 과제 종료로 정상 판정을 만들지 않는다. |
| `secret_investigation_history` | `history_id BIGINT PK`, `task_id BIGINT NOT NULL FK`, `event_type VARCHAR(40) NOT NULL`, `actor_user_id BIGINT FK`, `safe_changes JSONB NOT NULL`, `evidence_ref VARCHAR(120)`, `task_version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 배정·기한·증거·종료 추가 전용. 서버 허용 필드만 저장하고 과제 변경으로 판정/실행 사실을 만들지 않는다. |
| `secret_command_idempotency` | `command_id BIGINT PK`, `actor_user_id BIGINT NOT NULL FK`, `command_kind VARCHAR(80) NOT NULL`, `idempotency_key VARCHAR(120) NOT NULL`, `safe_request_digest BYTEA NOT NULL`, `decision_id`, `case_id BIGINT`, `safe_result JSONB NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | UQ(actor_user_id,command_kind,idempotency_key). 원문 본문 없이 안전한 해시·결과를 명령과 원자 저장. 같은 본문은 같은 결과, 다른 본문은 409. |

### 3.5. 대응·위치 조치·해결

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_response_cases` | `case_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `previous_case_id BIGINT FK`, `opening_occurrence_id BIGINT NOT NULL FK`, `opening_decision_id BIGINT NOT NULL FK`, `opened_by_user_id BIGINT NOT NULL FK`, `opening_reason VARCHAR(80) NOT NULL`, `case_origin VARCHAR(20) NOT NULL`, `is_current BOOLEAN NOT NULL`, `disposition VARCHAR(20) NOT NULL`, `action_status VARCHAR(30) NOT NULL`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT FK`, `version BIGINT NOT NULL`, `opened_at`, `updated_at TIMESTAMPTZ NOT NULL`, `resolved_at`, `closed_at TIMESTAMPTZ` | 같은 키·PRODUCTION 또는 TEST 출처 현재 건 최대 하나. 개설 결정·위치는 같은 키·출처의 유효 ABNORMAL 근거다. OPEN·IN_PROGRESS·REVALIDATION_PENDING·RESOLVED를 사용하고 예외는 독립 보존한다. 개설 결정을 다시 승인하는 검토/빈 시작 게이트 없음. 해결 후 재발은 previous_case_id를 가진 새 건이다. |
| `secret_case_occurrences` | `case_id`, `occurrence_id`, `credential_set_id BIGINT NOT NULL FK`, `linked_decision_id BIGINT NOT NULL FK`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT FK`, `response_status VARCHAR(30) NOT NULL`, `version BIGINT NOT NULL`, `current_action_cycle_id BIGINT FK`, `linked_at TIMESTAMPTZ NOT NULL`, `resolved_at TIMESTAMPTZ`, 복합 PK `(case_id,occurrence_id)` | 복합 FK/잠금 검사로 건·위치·결정의 키·출처 일치. 위치 담당자는 해당 연결의 적격 REMEDIATOR다. 새 비정상 위치 연결은 원자적이며 재검사 대기 건이면 새 주기·IN_PROGRESS를 함께 확정한다. 기존 위치 근거 보존. |
| `secret_assignment_history` | `assignment_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT FK`, `from_team`, `to_team VARCHAR(100)`, `from_user_id`, `to_user_id BIGINT FK`, `reason_code VARCHAR(80) NOT NULL`, `changed_by_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL`, `occurrence_version BIGINT` | occurrence NULL은 건 조정자, 있으면 위치 담당 변경. 활성 팀·연결 역할·대상 관계 검사. 현재 담당·버전·이력·감사 원자 저장. |
| `secret_case_action_cycles` | `cycle_id BIGINT PK`, `case_id`, `occurrence_id BIGINT NOT NULL FK`, `cycle_no BIGINT NOT NULL`, `started_at TIMESTAMPTZ NOT NULL`, `last_substantive_action_id BIGINT FK`, `completed_at`, `closed_at TIMESTAMPTZ`, `created_case_version BIGINT NOT NULL` | UQ(case_id,occurrence_id,cycle_no), 위치당 열린 주기 최대 하나. 첫 실질 조치·기록 마침·재조치를 구분해 마지막 실질 조치 뒤 비교 가능한 재검사만 선택한다. 빈 시작 클릭은 필수 사건이 아니다. |
| `secret_action_records` | `action_id BIGINT PK`, `case_id`, `occurrence_id`, `cycle_id BIGINT NOT NULL FK`, `action_type VARCHAR(40) NOT NULL`, `performed_at`, `recorded_at TIMESTAMPTZ NOT NULL`, `recorded_by_user_id BIGINT NOT NULL FK`, `evidence_ref VARCHAR(120) NOT NULL`, `case_version`, `occurrence_version BIGINT NOT NULL` | 건·위치·주기 복합 FK. ROTATED·REVOKED·REMOVED_FROM_SOURCE·PERMISSION_REDUCED와 제한된 티켓/변경 참조. 지정 담당자의 외부 수행 기록이며 제품 실행·공급자 확인이 아니다. 수행 시각은 최근 30일~요청+5분이다. |
| `secret_checklist_items` | `item_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `guide_version VARCHAR(80) NOT NULL`, `item_code VARCHAR(80) NOT NULL`, `status VARCHAR(20) NOT NULL`, `completed_at TIMESTAMPTZ`, `completed_by_user_id BIGINT FK`, `evidence_ref VARCHAR(120)`, `updated_at TIMESTAMPTZ NOT NULL` | UQ(case_id,guide_version,item_code). IDENTIFY_OWNER·ROTATE_OR_REVOKE·REMOVE_EXPOSURE·CHECK_ACCESS_LOGS의 TODO·DONE. DONE에는 수행자·시각·근거 필요. 보조 기록이며 체크 수는 조치 마침·재검사·해결을 대신하지 않는다. |
| `secret_checklist_history` | `history_id BIGINT PK`, `case_id`, `item_id BIGINT NOT NULL FK`, `from_status`, `to_status VARCHAR(20) NOT NULL`, `evidence_ref VARCHAR(71)`, `changed_by_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 체크·취소·근거 변경을 버전과 함께 추가 전용 이력으로 보존한다. |
| `secret_case_stage_history` | `stage_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `from_status`, `to_status VARCHAR(30) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `actor_user_id BIGINT FK`, `source_command_id BIGINT FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 개설·첫 실질 조치·기록 마침·새 위치 연결·재조치·근거 해결 전이 추가 전용. 체크·배정만으로 단계 전이를 만들지 않는다. |
| `secret_case_revalidations` | `revalidation_id BIGINT PK`, `case_id`, `occurrence_id`, `cycle_id`, `action_id`, `connection_id`, `scope_id`, `job_id`, `run_id BIGINT NOT NULL FK`, `requested_by_user_id BIGINT NOT NULL FK`, `idempotency_digest VARCHAR(65) NOT NULL`, `scope_revision BIGINT NOT NULL`, `detector_version`, `detector_config_hash`, `fingerprint_key_version VARCHAR(100) NOT NULL`, `case_version BIGINT NOT NULL`, `result VARCHAR(20) NOT NULL`, `failure_code VARCHAR(80)`, `requested_at TIMESTAMPTZ NOT NULL`, `finished_at TIMESTAMPTZ` | UQ(case_id,idempotency_digest). 같은 연결 위치·현재 주기·마지막 실질 조치·작업/실행 관계를 검사한다. 결과 PENDING·PRESENT·MISSING·UNKNOWN은 job 상태와 별개. 필수 수집/탐지 성공·동일 revision·비교 가능한 구성·조치 이후 조건 없이 MISSING 금지. |
| `secret_case_resume_events` | `resume_id BIGINT PK`, `case_id`, `occurrence_id`, `revalidation_id`, `resumed_by_user_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `resumed_at TIMESTAMPTZ NOT NULL` | 재검증에서 키가 남은 `PRESENT` 위치를 명시적으로 다시 조치 단계로 전환한 이력을 남긴다. |
| `secret_case_resolution_checks` | `check_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `target_origin VARCHAR(20) NOT NULL`, `version_digest VARCHAR(100) NOT NULL`, `target_snapshot`, `check_results`, `allowed_coverage_levels JSONB NOT NULL`, `checked_at TIMESTAMPTZ NOT NULL` | 연결 비정상·미연결 비정상·현재 미판정/미확인·정상 보관·과거 비교 가능 미발견·다른 출처 안내를 분리하고 정책/판정/조치/실행/증거 버전 고정. 필수 BLOCKED·UNKNOWN을 LIMITED로 우회하지 못한다. 응답은 별도 필드 권한 적용. |
| `secret_case_resolutions` | `resolution_id BIGINT PK`, `case_id BIGINT NOT NULL UQ FK`, `closure_check_id BIGINT NOT NULL FK`, `coverage_level VARCHAR(20) NOT NULL`, `resolved_by_user_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `evidence_snapshot`, `limitations JSONB NOT NULL`, `resolved_at TIMESTAMPTZ NOT NULL` | FULL·LIMITED·SIMULATION과 실제 근거·확인 한계 불변 보존. FULL은 모든 필수 확인·공급자 확인, LIMITED는 승인 수동 근거, SIMULATION은 정확한 합성 TEST에 한정. 해결 시 버전·전체 권한·현재 근거를 원자 재검사한다. |
| `secret_case_resolution_locations` | `resolution_id BIGINT NOT NULL FK`, `occurrence_id BIGINT NOT NULL FK`, `revalidation_id BIGINT NOT NULL FK`, `cycle_id BIGINT NOT NULL FK`, `last_substantive_action_id BIGINT NOT NULL FK`, 복합 PK `(resolution_id,occurrence_id)` | 연결 비정상 위치마다 마지막 실질 조치 뒤 비교 가능한 MISSING 고정. 정상 보관 위치는 제거 요구 없음. 키 수명주기와 위치 부재는 별개다. |
| `secret_case_dismissals` | `dismissal_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `reason_code VARCHAR(20) NOT NULL`, `evidence_snapshot JSONB NOT NULL`, `accepted_transfer_ref VARCHAR(120)`, `approved_by_user_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | FALSE_POSITIVE는 모든 관련 Finding 교정, OUT_OF_SCOPE는 승인된 범위·소유자 이관·인수 확인 필요. 실제 미해결 노출이 남으면 거부. disposition·is_current=false·closed_at·감사 원자 저장. RESOLVED·위험 수용 수에 넣지 않는다. |
| `secret_exceptions` | `exception_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `approved_by_user_id BIGINT NOT NULL FK`, `approved_at`, `expires_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ`, `revoked_by_user_id BIGINT FK`, `revoke_reason_code VARCHAR(80)` | expires_at>approved_at. 위치 범위·승인 권한·활성 중복 검사, 취소/만료 이력 보존. 남은 미해결 위치 모두 예외에 포함될 때만 RISK_ACCEPTED 집계. 새 위치 자동 포함 금지. |
| `secret_exception_occurrences` | `exception_id BIGINT NOT NULL FK`, `occurrence_id BIGINT NOT NULL FK`, 복합 PK | 승인 당시의 위치 범위를 고정한다. 나중에 새 위치가 발견돼도 예외 범위에 자동 포함되지 않는다. |

### 3.6. 수명주기·합성 근거·교차 환경 경고

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_credential_lifecycle_evidence` | `evidence_id BIGINT PK`, `case_id`, `credential_set_id BIGINT NOT NULL FK`, `origin VARCHAR(20) NOT NULL`, `kind VARCHAR(30) NOT NULL`, `verification_method`, `verification_level VARCHAR(30) NOT NULL`, `evidence_ref VARCHAR(120)`, `provider_validator_id VARCHAR(120)`, `validator_version`, `endpoint_class VARCHAR(80)`, `performed_at`, `observed_at TIMESTAMPTZ NOT NULL`, `submitted_by_user_id BIGINT FK`, `version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 외부 제출 EXTERNAL_RECORD/PENDING, SECURITY 승인도 APPROVED_MANUAL. 승인 워커·실제 공급자 관측만 PROVIDER/PROVIDER_VERIFIED 생성. ROTATED·REVOKED·RETIRED는 실제 키, FIXTURE_RETIRED는 합성 TEST만. 조합 CHECK·DB 역할로 공급자 확인 위조 금지. 원문·전체 응답 금지. |
| `secret_lifecycle_evidence_reviews` | `review_id BIGINT PK`, `evidence_id BIGINT NOT NULL FK`, `decision VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `expected_evidence_version`, `case_version BIGINT NOT NULL`, `reviewed_by_user_id BIGINT NOT NULL FK`, `limitations JSONB NOT NULL`, `reviewed_at TIMESTAMPTZ NOT NULL` | APPROVE·REJECT 추가 전용. 승인 수동 수준은 APPROVED_MANUAL이며 LIMITED 상한. 공급자 확인으로 승격 금지. 취소·상충·적용 시각은 해결 시 재검사한다. |
| `secret_lifecycle_evidence_applications` | `application_id BIGINT PK`, `evidence_id`, `case_id`, `credential_set_id BIGINT NOT NULL FK`, `origin VARCHAR(20) NOT NULL`, `reviewed_by_user_id BIGINT NOT NULL FK`, `reviewed_at TIMESTAMPTZ NOT NULL` | UQ(evidence_id,case_id). 다른 출처 건 재사용은 키·시각·현재 건·권한을 검토한 관계만 추가하고 원 증거 복제/변경 금지. 위치별 조치 후 부재는 별도 필수. |
| `secret_fixture_manifests` | `manifest_id`, `version BIGINT NOT NULL`, `protected_manifest_ref VARCHAR(300) NOT NULL`, `approval_status VARCHAR(20) NOT NULL`, `approved_by_user_id BIGINT FK`, `valid_from TIMESTAMPTZ NOT NULL`, `valid_until`, `revoked_at TIMESTAMPTZ`, 복합 PK `(manifest_id,version)` | 승인·버전 고정 합성 등록물의 보호 참조. 유효 기간·승인/취소 이력 보존. 일반 API에는 값·지문·다른 표본 목록 금지. |
| `secret_fixture_match_evidence` | `match_evidence_id BIGINT PK`, `credential_set_id`, `scope_id BIGINT NOT NULL FK`, `scope_revision BIGINT NOT NULL`, `manifest_id`, `manifest_version BIGINT NOT NULL`, `comparator_id VARCHAR(120) NOT NULL`, `comparator_version VARCHAR(80) NOT NULL`, `match_result VARCHAR(20) NOT NULL`, `protected_match_ref VARCHAR(300)`, `observed_at TIMESTAMPTZ NOT NULL` | manifest 복합 FK. 승인 워커/비교기가 보호된 정확한 키 동일성 비교. TEST 표식·마스킹 끝자리·사용자 선언은 근거 아님. 키·scope revision·manifest 버전에 고정. |
| `secret_test_fixture_attestations` | `attestation_id BIGINT PK`, `credential_set_id`, `scope_id BIGINT NOT NULL FK`, `scope_revision BIGINT NOT NULL`, `manifest_id`, `manifest_version BIGINT NOT NULL`, `match_evidence_id BIGINT NOT NULL FK`, `confirmed_by_user_id BIGINT FK`, `confirmation_source VARCHAR(20) NOT NULL`, `status VARCHAR(20) NOT NULL`, `confirmed_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ` | 매칭 근거 중복 확인 금지. HUMAN 또는 승인된 자동 매칭 정책. manifest 만료/취소·scope 개정·상충 실제 키 근거가 있으면 기존 확인으로 SIMULATION 불가. 위치 제거·조치 후 재검사를 대신하지 않는다. |
| `secret_cross_environment_alerts` | `alert_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `previous_alert_id BIGINT FK`, `status VARCHAR(30) NOT NULL`, `assignee_user_id BIGINT FK`, `due_at TIMESTAMPTZ`, `test_case_id`, `investigation_task_id BIGINT FK`, `version BIGINT NOT NULL`, `first_observed_at`, `last_observed_at TIMESTAMPTZ NOT NULL`, `acknowledged_at`, `closed_at TIMESTAMPTZ`, `closure_evidence_snapshot JSONB` | 키당 열린 경고 최대 하나. 완전한 현재 TEST 값 후보·유효 Finding·합성 미확인에서 SUSPECTED_REAL, 실제 발급자 근거로 CONFIRMED_REAL. CANDIDATE·METADATA_ONLY·모든 Finding 오탐 제외. 건/과제 하나 이상 연결하고 둘 다 없으면 과제 원자 생성. 담당·기한 각각 nullable. 인지는 종료가 아니며 반증 또는 실제 수명주기 근거로 닫는다. |
| `secret_cross_environment_alert_locations` | `alert_id`, `occurrence_id BIGINT NOT NULL FK`, `last_observed_at TIMESTAMPTZ NOT NULL`, 복합 PK `(alert_id,occurrence_id)` | 반복 스캔·추가 위치는 열린 경고 갱신. 조회별 위치 ACL 검사, 운영 ResponseCase 수에 합산 금지. |

### 3.7. 역할·감사

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_role_grants` | `grant_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `role VARCHAR(30) NOT NULL`, `connection_id BIGINT FK`, `granted_by_user_id BIGINT NOT NULL FK`, `grant_source VARCHAR(20) NOT NULL`, `granted_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ` | 역할 SECURITY·REMEDIATOR·OPERATIONS, 부여 출처 BOOTSTRAP·MANUAL. 연결 NULL은 명시 승인 전역 권한. 활성 중복 금지. ADMIN과 별개이며 취소를 재시작으로 복구하지 않는다. 권한 변경은 스냅샷 ACL revision 무효화. |
| `secret_team_memberships` | `membership_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `team_name VARCHAR(100) NOT NULL`, `added_by_user_id BIGINT NOT NULL FK`, `added_at TIMESTAMPTZ NOT NULL`, `removed_at TIMESTAMPTZ` | 팀 담당자 지정 권한 확인에 사용한다. 팀 문자열만으로 소속을 추정하지 않는다. 활성 중복 소속을 막는다. |
| `secret_audit_events` | `audit_id BIGINT PK`, `request_id VARCHAR(120)`, `run_id`, `case_id BIGINT FK`, `target_type VARCHAR(40) NOT NULL`, `target_id BIGINT NOT NULL`, `event_type VARCHAR(80) NOT NULL`, `actor_user_id BIGINT FK`, `worker_id VARCHAR(120)`, `target_version BIGINT`, `safe_changes JSONB NOT NULL`, `occurred_at TIMESTAMPTZ NOT NULL` | 사람/승인 워커 구분. 결정·승인·담당·조치·증거·해결·실패를 추가 전용 보존. 일반 API 역할 UPDATE·DELETE 금지. 원문·자유 메모·공급자 응답 금지. |

### 3.8. 일관된 읽기 스냅샷

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_query_snapshots` | `snapshot_id UUID PK`, `scope VARCHAR(20) NOT NULL`, `owner_user_id BIGINT NOT NULL FK`, `case_origin VARCHAR(20)`, `as_of`, `created_at`, `expires_at TIMESTAMPTZ NOT NULL`, `authorization_revision`, `scope_version_digest VARCHAR(100) NOT NULL`, `normalized_filters JSONB NOT NULL`, `filter_digest VARCHAR(100) NOT NULL`, `partial_coverage BOOLEAN`, `restricted_coverage BOOLEAN NOT NULL`, `coverage_status VARCHAR(30) NOT NULL`, `missing_sources JSONB NOT NULL`, `retained_from TIMESTAMPTZ`, `history_complete BOOLEAN NOT NULL` | WORKBENCH·CURRENT_CASES·INVENTORY·HISTORY. TTL 15분, 사용자당 유효 5개. 사용자·역할/범위 revision·필터·출처 종속. REPEATABLE READ에서 헤더·행·membership 확정. 근거/권한 부족이면 partial_coverage NULL, 실패를 false로 만들지 않는다. |
| `secret_snapshot_subjects` | `snapshot_id UUID NOT NULL FK`, `subject_type VARCHAR(30) NOT NULL`, `subject_id BIGINT NOT NULL`, `credential_set_id BIGINT`, `safe_sort_key VARCHAR(200) NOT NULL`, `safe_facts JSONB NOT NULL`, `subject_version VARCHAR(100)`, 복합 PK `(snapshot_id,subject_type,subject_id)` | ResponseCase·CredentialSet·Occurrence·Scope·CrossEnvironmentAlert 타입 구분. 같은 숫자 ID 충돌 금지. 부모·ACL은 생성 트랜잭션 검사. 허용 사실만 복제하며 원천 참조·권한 밖 수/시각 금지. |
| `secret_snapshot_metrics` | `snapshot_id UUID NOT NULL FK`, `metric_key VARCHAR(80) NOT NULL`, `unit VARCHAR(30) NOT NULL`, `status VARCHAR(30) NOT NULL`, `value BIGINT`, `reason_code VARCHAR(80)`, `applied_filters`, `not_applicable_filters JSONB NOT NULL`, 복합 PK `(snapshot_id,metric_key)` | AVAILABLE만 value>=0. NOT_AUTHORIZED·LEGACY_UNAVAILABLE·UNAVAILABLE은 NULL. Mixed 최상위 합계 없음. 적용/해당 없음 필터·권한 제한 분리. |
| `secret_snapshot_metric_memberships` | `snapshot_id UUID NOT NULL`, `metric_key VARCHAR(80) NOT NULL`, `subject_type VARCHAR(30) NOT NULL`, `subject_id BIGINT NOT NULL`, 복합 PK `(snapshot_id,metric_key,subject_type,subject_id)` | metrics·subjects 복합 FK. 카드·도착 대기열·목록은 동일 membership. 미지정은 대응 건 부분집합이며 추가 노출 수가 아니다. |
| `secret_snapshot_rows` | `snapshot_id UUID NOT NULL FK`, `subject_id BIGINT NOT NULL`, `credential_set_id BIGINT NOT NULL`, `action_bucket VARCHAR(30)`, `disposition VARCHAR(20)`, `severity VARCHAR(10)`, `validity VARCHAR(10) NOT NULL`, `lifecycle_status VARCHAR(20) NOT NULL`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `content_kind VARCHAR(20) NOT NULL`, `inventory_source_provider VARCHAR(20)`, `first_detected_at TIMESTAMPTZ`, 복합 PK `(snapshot_id,subject_id)` | 같은 헤더의 typed projection. currentCases는 현재 IN_SCOPE 건, inventory는 키마다 하나, history는 과거 건마다 하나. 네 단계·별도 RISK_ACCEPTED는 배타적. 남은 미해결 위치 모두 예외일 때만 예외 bucket. 대표 심각도는 근거 있는 최고 미해결 등급이며 전부 미평가면 UNASSESSED. |
| `secret_snapshot_row_locations` | `snapshot_id UUID NOT NULL`, `subject_id BIGINT NOT NULL`, `occurrence_id BIGINT NOT NULL FK`, `source_provider VARCHAR(20) NOT NULL`, `environment VARCHAR(10) NOT NULL`, `linked_at TIMESTAMPTZ NOT NULL`, 복합 PK `(snapshot_id, subject_id, occurrence_id)` | 복합 FK `(snapshot_id, subject_id)`가 `secret_snapshot_rows`를 참조한다. 소스와 환경은 같은 위치 행에서 함께 필터한다. 위치가 여러 개여도 대상 행을 한 번만 센다. |

## 4. 원자성과 보존 불변 조건

| 변경 경계 | 불변 조건 |
|---|---|
| 판정·건 연결 | 최신 구성·존재·Finding 소속·위치/범위/정책 버전·ACL을 검사하고 결정·현재 건·위치 연결·조치 주기·감사·멱등 결과를 한 트랜잭션에 확정한다. 같은 키·출처 현재 건 고유성이 최후 방어선이다. |
| 새 위치·해결 경합 | 같은 키·현재 건 잠금 순서로 직렬화한다. 새 비정상 위치가 붙으면 해결을 재검사하고 재검사 대기 건은 IN_PROGRESS로 전이한다. 기존 위치의 증거는 보존한다. |
| 담당·조치 | 위치가 같은 건·키·연결·출처인지 복합 FK 또는 잠금 검사로 강제한다. 역할·활성 팀·할당은 매 명령 재검사한다. 첫 실질 조치로 IN_PROGRESS를 기록하며 현재 주기의 실질 조치 없이 기록 마침을 허용하지 않는다. |
| 해결 | 같은 출처의 연결 비정상 위치별 마지막 실질 조치 후 MISSING, 미연결 유효 ABNORMAL 없음, 현재 미판정/미확인 위치 없음, 필수 검사 누락 없음, 수명주기 또는 합성 근거를 현재 버전으로 검사한다. 정상 보관 삭제·다른 출처 합산은 요구하지 않는다. LIMITED는 필수 실패 우회 수단이 아니다. |
| 증거 이력 | 판정·교정·승인·조치·수명주기·해결은 추가 전용. Finding 재생성·별칭·정책 변경 뒤에도 당시 ID·유형·버전·관측 시각·안전한 참조·확인 한계를 보존한다. 과거 근거 없는 사실은 소급 생성하지 않는다. |
| 스냅샷 | REPEATABLE READ에서 원본·ACL·필터·membership 확정. 만료·사용자/필터 불일치·권한 변경은 거부. 권한 밖 전체 수를 먼저 계산해 차감하지 않는다. 읽기 스냅샷은 쓰기 허가 근거가 아니다. |

전체 관련 위치의 SECURITY 접근이 없는 사용자는 건 전체 해결을 요청할 수 없다. restrictedCoverage는 비완전 권한 사용자에게 일관된 일반 신호로 제공하고 숨겨진 위치 존재·수·시각에 따라 달라지는 신호를 만들지 않는다. 결과 열람과 원 근거 열람 권한은 각각 검사한다.

## 5. 고유성·FK·인덱스

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
CREATE UNIQUE INDEX uq_secret_current_case
  ON secret_response_cases (credential_set_id, case_origin) WHERE is_current;
CREATE UNIQUE INDEX uq_secret_active_connection_job
  ON secret_jobs (connection_id) WHERE status IN ('PENDING', 'LEASED');
CREATE UNIQUE INDEX uq_secret_scheduled_job
  ON secret_jobs (connection_id, kind, scheduled_at) WHERE requested_by_user_id IS NULL;
CREATE UNIQUE INDEX uq_secret_enabled_scope
  ON secret_connection_scopes (connection_id, scope_key) WHERE enabled;
CREATE UNIQUE INDEX uq_secret_open_real_key_alert
  ON secret_cross_environment_alerts (credential_set_id) WHERE closed_at IS NULL;
CREATE INDEX idx_secret_jobs_available
  ON secret_jobs (available_at, job_id) WHERE status = 'PENDING';
```

| 특수 FK | 참조 대상 |
|---|---|
| opening_decision_id·linked_decision_id·latest_decision_id·resolving_decision_id | secret_exposure_assessments.assessment_id. 같은 위치·키·출처 적용을 별도 검사 |
| lifecycle_evidence_id | secret_credential_lifecycle_evidence.evidence_id. 해당 키·관측 시점·확인 수준이 일치하는 유효 근거 |
| validation_attempts.evidence_id·assessment_evidence.evidence_id | secret_evidence.evidence_id |
| previous_case_id·test_case_id·case_id | secret_response_cases.case_id |
| previous_alert_id | secret_cross_environment_alerts.alert_id |
| current_action_cycle_id·cycle_id | secret_case_action_cycles.cycle_id |
| action_id·last_substantive_action_id | secret_action_records.action_id |
| closure_check_id·resolution_id·revalidation_id | 각각 secret_case_resolution_checks·secret_case_resolutions·secret_case_revalidations의 PK |
| match_evidence_id·manifest_id/manifest_version | 각각 secret_fixture_match_evidence의 PK·secret_fixture_manifests의 복합 PK |
| supersedes_assessment_id·classification_id | 각각 secret_assessments의 PK·secret_asset_classifications의 PK |

수동 작업 멱등 범위는 요청자·명령·키다. secret_command_idempotency의 고유성·작업 결과 연결로 중복을 막는다. connection_id·scope_id·run_id·run_scope_id·tool_run_id·job_id는 해당 테이블 PK를 참조하며 scope/connection·run/job·case/occurrence/credential의 일치는 복합 FK로 강제한다. FK 삭제 연쇄로 과거 증거를 지우지 않는다.

secret_exposure_assessments는 재생성되는 Finding에 의존해 과거 증거를 잃지 않도록 당시 ID·룰·실행을 스냅샷으로 보존한다. 저장 시 같은 위치의 유효 Finding인지 검증한다. 다형 subject_id는 생성 트랜잭션이 타입별 실제 부모·ACL을 검사한다.

| 조회 경로 | 인덱스 기준 |
|---|---|
| 키별 허용 위치 | occurrences(credential_set_id,connection_id,scope_id,last_seen_at DESC,occurrence_id DESC) 및 노출/존재 필터. ACL SQL 선적용 |
| 최신 Finding·교정 | findings(occurrence_id,detected_at DESC,finding_id DESC), finding_dispositions(finding_id,created_at DESC), Finding별 현재 교정 고유성 |
| 대응·담당 | response_cases(case_origin,action_status,updated_at DESC,case_id DESC) WHERE is_current, case_occurrences(assigned_user_id,response_status,case_id,occurrence_id) |
| 검증·재검사 | validation_attempts(credential_set_id,completed_at DESC,attempt_id DESC), case_revalidations(occurrence_id,finished_at DESC,revalidation_id DESC) |
| 현재 revision 검사 | run_scopes(scope_id,scope_revision,finished_at DESC,run_scope_id DESC)와 성공 조건부 인덱스. scan_runs(connection_id,trigger_type,requested_at DESC,run_id DESC) |
| 연결 테스트 | connection_probes(connection_id,completed_at DESC,probe_id DESC) |
| 스냅샷 | query_snapshots(owner_user_id,expires_at), metric_memberships(snapshot_id,metric_key,subject_type,subject_id), subject 안전 정렬 키 |
| 열린 조사 | 열린 대상·부족 사유 고유성, investigation_tasks(assignee_user_id,due_at,task_id). 기한 NULL은 초과 제외 |

목록은 같은 시점의 허용 subject를 묶어 조회하고 N+1을 피한다. 현황 GET은 외부 공급자 API 호출이나 대량 상태 변경을 수행하지 않는다.

## 6. 적재와 집계

1. 운영 제안·독립 승인으로 범위 revision·출처·보관 정책을 고정한다. 이름·공급자·환경으로 운영/시험을 추정하지 않고 UNCLASSIFIED는 별도 대기열에 둔다.
2. 도래한 작업은 FOR UPDATE SKIP LOCKED로 선점해 임대 상태·소유자·시각을 원자 저장한다. 6시간 정기는 직전 예정 기준이며 수동 실행은 그 시각을 바꾸지 않는다.
3. 도구 출력·원문은 승인 워커 메모리에서 정규화·HMAC·선택 검증 후 폐기한다. 동일성 고유 제약 아래 최초 삽입 실행을 고정하고 재발견·새 위치·구성 승격에서 변경하지 않는다.
4. 같은 키·원천 위치를 재사용하고 Finding은 엔진별 보존한다. 비교 가능한 필수 단계 성공만 MISSING 근거다. 실패는 UNKNOWN과 마지막 성공을 함께 보존하며 수동 노출 판정을 지우지 않는다.
5. IAM 목록은 METADATA_ONLY·실행별 관측이다. Secrets Manager·SSM·S3 목록은 항목 수·범위 상태·안전한 사유만 남기고 자원명·ARN·설명·태그·페이지 토큰을 결과로 적재하지 않는다. 정확한 단일 값 범위는 별도 승인·CURRENT_VALUE_ONLY·opt-in에서만 탐지하고 위치 참조는 aws-content-scope:<scope_id>다.
6. 최신 검증 시도·마지막 성공·미검증 이유를 분리한다. 존재·VALID·ACTIVE·사용 기록·침해 사실은 서로 추론하지 않는다. 평가는 정책·규칙·근거·시각을 보존하고 근거 부족을 LOW로 채우지 않는다.
7. 판정·조치·재검사·수명주기·해결은 버전·ACL·감사·멱등 결과와 원자 저장한다. 오탐은 Finding 교정, 정상 보관은 위치 판정, 키 무효는 검증 결과다.
8. WORKBENCH는 같은 헤더의 typed subject·metric membership을 대기열·분포로 투영한다. 독립 INVENTORY·CURRENT_CASES·HISTORY 헤더는 다른 scope로 재사용하지 않는다. CSV는 독립 INVENTORY 전체 페이지·고유 키 10,000개 상한을 따른다.

범위의 PENDING·RUNNING·SUCCEEDED·PARTIAL·FAILED·SKIPPED만 배타적으로 scopeCount에 합산한다. finishedScopeCount=SUCCEEDED+PARTIAL+FAILED+SKIPPED, successfulScopeCount=SUCCEEDED, gapScopeCount=PARTIAL+FAILED+SKIPPED, unverifiedScopeCount=scopeCount-SUCCEEDED다. 현재 coverageGapScopes는 최근 불완전·새 revision 미검사·미실행·기한 초과·일시중지의 고유 범위 수다.

ScopeCoverageProjection은 실행 사실에서 파생한다. 물리 투영은 원본/투영 버전·시각을 갖고 재생 가능해야 한다. 캐시가 판정·해결의 원본은 아니다. 범위 비활성화는 예약 분모만 바꾸고 알려진 노출·과거 증거를 삭제하거나 해결을 통과시키지 않는다.

## 7. 보안·보존·복구

API DB 역할은 허용 조회·업무 변경, 워커 역할은 작업 임대·탐지/검증·공급자 근거 적재로 분리한다. 일반 HTTP·DB 경로는 PROVIDER_VERIFIED를 만들 수 없다. 연결 비밀·HMAC 키는 DB 밖에 두고 버전을 관리한다. 보호키 교체의 재취득·동일성 재계산은 제한 접근으로 수행하며 재취득 불가 항목은 임의 병합하지 않는다.

원문은 영속 저장하지 않는다. AWS SDK가 불변 String을 만들 수 있어 프로세스 메모리 전체 소거를 보장한다고 주장하지 않는다. 메타데이터·감사·증거·백업은 범주별 보존 기간·접근·삭제·복구 범위를 운영 정책에 명시한다. 기간 미정은 자동 삭제나 무기한 보존의 기본값이 아니다. 만료 스냅샷 정리는 사실·감사 보존과 별개다.

복구 후 동일성·FK·현재 건 유일성·scope revision·최신 시도와 마지막 성공·증거 추가 전용성·권한 취소·집계 단위를 검증한다. 근거 부재는 null과 안전한 미확인 사유로 남기고 합성/실키·정상 보관·완전 검사·공급자 폐기를 소급 생성하지 않는다.
