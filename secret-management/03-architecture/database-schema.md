# Secret 데이터베이스 스키마

기준: [PRD](../01-requirements/PRD.md), [데이터 계약](data-contract.md), [API 계약](api-contract.md), [판정 정책](../02-functional-design/triage-policy.md), [대응 정책](../02-functional-design/response-policy.md), [심각도 정책](../02-functional-design/severity-policy.md)

PostgreSQL에 저장하는 Secret 사실 원본, 관계, 제약과 조회 스냅샷의 명세다. 수집·탐지·판정·외부 조치 기록·공급자 확인을 독립 사실로 보존한다. 제품이 따르는 스키마를 정의하며 적용·병합·배포 현황을 나타내지 않는다.

## 1. 공통 데이터와 접근 경계

| 공통 데이터 | Secret의 참조 방식 |
|---|---|
| `dashboard_users` | 담당·제안·승인·요청·감사 행위자 FK. 공통 세션 인증을 사용하며 ADMIN과 Secret 역할은 별개다. 계정 생성·비활성화는 공통 계정 관리 경계에 둔다. |
| `assets` | 명확한 자산 대응 근거가 있을 때 선택 연결. Notion 또는 환경 UNKNOWN을 표현하려고 가짜 자산이나 PROD 기본값을 만들지 않는다. |
| `detection_events` | 일반 탐지 사실 원본. Secret의 위치·미평가·재검사·예외는 전용 모델에 보존하고 통합 API에서 영역·ID·단위를 명시해 합성한다. 일반 탐지에 Secret을 복제 적재하지 않는다. |

테이블은 secret_ 접두사, PK는 별도 표기가 없으면 BIGINT GENERATED ALWAYS AS IDENTITY다. 모든 시각은 TIMESTAMPTZ, 사용자 참조는 dashboard_users의 BIGINT FK다. 아래 표는 주요 저장 컬럼과 관계를 요약한다. 응답의 현재 상태·안전 별칭·집계는 원본과 뷰에서 파생하며 같은 이름의 물리 컬럼이 있다고 가정하지 않는다. 과거 기록에서 추가된 증거 컬럼의 NULL은 근거 부재로 보존한다. 상태는 VARCHAR와 CHECK, 공급자 키 유형·룰은 버전 등록부로 검증한다. 운영 DB 서버·배포 위치는 공통 데이터 저장 정책을 따른다.

키 원문·인증 토큰·원천 본문·전체 공급자 응답은 일반 DB·큐·스냅샷·감사·백업에 넣지 않는다. auth_ref는 DB 밖 비밀 참조다. approved_ref와 재취득 원천 참조는 민감한 메타데이터로 제한 접근하며 일반 응답에는 해당 사용자의 열람 권한과 승인 참조 일치를 확인한 구조화된 대상·발견 위치 좌표를 제공하며, 원본 승인 URI·인증 참조·본문은 반환하지 않는다.

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
| `secret_connections` | `connection_id BIGINT PK`, `provider VARCHAR(20) NOT NULL`, `display_name VARCHAR(200) NOT NULL`, `auth_ref VARCHAR(300) NOT NULL`, `status VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `next_due_at TIMESTAMPTZ`, `last_collection_success_at TIMESTAMPTZ`, `source_retry_at TIMESTAMPTZ`, `source_retry_reason VARCHAR(80)`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL` | 플랫폼은 AWS·KUBERNETES·GITHUB·NOTION, 상태는 ACTIVE·PAUSED·ERROR다. auth_ref는 DB 밖 연결 비밀의 불투명 참조다. 범위 변경은 scope_version을 증가시킨다. last_collection_success_at은 같은 전체 실행에서 모든 필수 범위·단계가 성공한 시각만 가리킨다. |
| `secret_connection_scopes` | `scope_id BIGINT PK`, `connection_id BIGINT NOT NULL FK`, `scope_key VARCHAR(300) NOT NULL`, `revision BIGINT NOT NULL`, `content_type VARCHAR(80) NOT NULL`, `approved_ref VARCHAR(500) NOT NULL`, `history_limit VARCHAR(100)`, `enabled BOOLEAN NOT NULL`, `response_origin VARCHAR(20) NOT NULL`, `response_origin_version BIGINT NOT NULL`, `classification_version VARCHAR(80)`, `safe_display_alias VARCHAR(80)`, `alias_version BIGINT NOT NULL`, `created_at`, `updated_at TIMESTAMPTZ NOT NULL` | UQ(connection_id,scope_key,revision), 활성 scope_key당 revision 하나. 출처 UNCLASSIFIED·PRODUCTION·TEST와 수집 revision·표시 alias_version은 별개다. 사용자 별칭은 승인 참조와 별도 관리한다. 기본 대상 표시는 검증된 승인 좌표에서 파생하고, 좌표 확인이 불가능할 때 내부 ID를 사용한다. 별칭 변경은 별도 이력에 남기고 수집 revision을 늘리지 않는다. |
| `secret_scope_alias_history` | `alias_change_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `safe_display_alias VARCHAR(80)`, `changed_by_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `scope_revision BIGINT NOT NULL` | 별칭 변경·해제의 안전한 값·수행자·시각·수집 revision 보존. alias_version은 scope에서 증가하며 승인 원천 참조를 이력에 복제하지 않는다. |
| `secret_connection_probes` | `probe_id BIGINT PK`, `connection_id BIGINT NOT NULL FK`, `actor_user_id BIGINT NOT NULL FK`, `check_kind VARCHAR(40) NOT NULL`, `status VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80)`, `checked_at TIMESTAMPTZ NOT NULL` | check_kind=IDENTITY_OR_INSTALLATION. CONNECTED·FAILED·UNAVAILABLE과 안전한 사유만 보존한다. 개별 콘텐츠 읽기·키 검증·전체 수집 성공이나 연결 활성 상태를 만들지 않는다. |
| `secret_storage_policies` | `policy_id`, `version BIGINT NOT NULL`, `status VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `created_by_user_id BIGINT NOT NULL FK`, `approved_by_user_id BIGINT FK`, `created_at`, `valid_from TIMESTAMPTZ NOT NULL`, `valid_until`, `approved_at`, `retired_at TIMESTAMPTZ`, 복합 PK `(policy_id,version)` | DRAFT·ACTIVE·RETIRED. 승인자는 작성자와 다르고 ACTIVE에는 승인자·시각 필요. 종료가 있으면 시작보다 뒤다. 승인 버전은 불변이며 변경·폐기는 새 버전/불변 승인 이력으로 남긴다. 상충 허용·금지 규칙 활성화 거부. |
| `secret_storage_policy_rules` | `rule_id BIGINT PK`, `policy_id`, `policy_version BIGINT NOT NULL`, `effect VARCHAR(10) NOT NULL`, `connection_id`, `scope_id`, `scope_revision BIGINT NOT NULL`, `scope_fingerprint VARCHAR(32) NOT NULL`, `content_type VARCHAR(80) NOT NULL`, `branch_condition VARCHAR(200)`, `path_match VARCHAR(10) NOT NULL`, `protected_path_condition VARCHAR(500)`, `created_at TIMESTAMPTZ NOT NULL` | 정책 버전·범위/연결 복합 FK. ALLOW·DENY, SCOPE·EXACT·PREFIX 적용 범위. 승인 revision·보호된 범위 식별·브랜치·경로 조건을 재검사한다. 민감 경로는 일반 응답에 반환하지 않으며 만료·불일치·충돌은 UNKNOWN이다. |
| `secret_scope_classification_proposals` | `proposal_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `expected_scope_version BIGINT NOT NULL`, `proposed_origin VARCHAR(20) NOT NULL`, `policy_id`, `policy_version BIGINT`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `proposed_by_user_id BIGINT NOT NULL FK`, `approval_mode VARCHAR(30) NOT NULL`, `approval_rule_version VARCHAR(80) NOT NULL`, `reviewer_user_id BIGINT FK`, `status VARCHAR(30) NOT NULL`, `version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 범위당 PENDING_APPROVAL 최대 하나. DIRECT는 제안·결정을 같은 트랜잭션에서 APPROVED로 기록하며 대기 행을 만들지 않는다. INDEPENDENT_REVIEW는 제안자와 다른 현재 범위 SECURITY만 검토한다. 검토자 미지정은 대기로 유지한다. 정책 ID·버전은 함께 NULL 또는 활성 버전 FK. 정책 없음은 POLICY_PENDING이며 보관 UNKNOWN 유지. |
| `secret_scope_origin_decisions` | `decision_id BIGINT PK`, `proposal_id BIGINT FK`, `scope_id BIGINT NOT NULL FK`, `from_origin`, `to_origin VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `decision VARCHAR(20) NOT NULL`, `approval_mode VARCHAR(30)`, `approval_rule_version VARCHAR(80)`, `actor_user_id BIGINT NOT NULL FK`, `decided_at TIMESTAMPTZ NOT NULL`, `version BIGINT NOT NULL` | UQ(scope_id,version). 승인·거절·변경 전후 추가 전용. DIRECT의 결정자=제안자는 허용 조건을 서버가 재검사하고 독립 검토에는 다른 결정자를 강제한다. LEGACY의 제안·승인 방식 부재를 정식 승인으로 추정하지 않는다. 적용 시 범위 버전·정책 바인딩·영향 감사·집계 무효화를 원자 확정한다. 거절은 활성 분류 유지. 과거 건 출처를 덮어쓰지 않는다. |
| `secret_scope_policy_bindings` | `binding_id BIGINT PK`, `scope_id BIGINT NOT NULL FK`, `scope_revision`, `scope_origin_version BIGINT NOT NULL`, `scope_fingerprint VARCHAR(32) NOT NULL`, `policy_id`, `policy_version BIGINT NOT NULL`, `classification_decision_id BIGINT NOT NULL FK`, `effective_from TIMESTAMPTZ NOT NULL`, `effective_until TIMESTAMPTZ` | 범위당 effective_until=NULL인 바인딩 최대 하나. 정책 버전·승인 결정/범위 복합 FK. 수집 revision과 출처 승인 버전을 함께 고정하며 과거 바인딩 의미를 바꾸지 않는다. |
| `secret_scope_classification_impacts` | `impact_id BIGINT PK`, `decision_id`, `occurrence_id BIGINT NOT NULL FK`, `case_id BIGINT FK`, `previous_origin`, `new_origin VARCHAR(20) NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 출처 교정이 영향을 준 위치·기존 건과 이전/새 출처를 보존한다. 과거 건의 출처를 직접 바꾸지 않는다. |

### 3.2. 작업·실행·범위 결과

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_jobs` | `job_id BIGINT PK`, `kind VARCHAR(20) NOT NULL`, `purpose VARCHAR(40) NOT NULL`, `connection_id BIGINT NOT NULL FK`, `target_occurrence_id BIGINT FK`, `expected_context JSONB`, `expected_evidence_revision BIGINT`, `requested_by_user_id BIGINT FK`, `idempotency_key VARCHAR(120)`, `scheduled_at TIMESTAMPTZ NOT NULL`, `available_at TIMESTAMPTZ NOT NULL`, `attempt_no INTEGER NOT NULL`, `max_attempts INTEGER NOT NULL`, `status VARCHAR(20) NOT NULL`, `lease_owner VARCHAR(120)`, `lease_until TIMESTAMPTZ`, `last_error_code VARCHAR(80)`, `created_at`, `finished_at TIMESTAMPTZ` | kind=FULL_SCAN·REVALIDATE·ASSESS, 목적은 각각 COLLECTION_AND_DETECTION·CREDENTIAL_VALIDITY/LOCATION_RECHECK·SEVERITY_ASSESSMENT다. 상태는 PENDING·LEASED·SUCCEEDED·FAILED·CANCELLED다. 같은 연결의 외부 호출을 동반하는 FULL_SCAN/REVALIDATE는 합쳐 최대 하나다. ASSESS는 위치/근거 revision/정책별로 멱등화하고 ScanRun을 만들지 않는다. expected_context에는 내부 ID와 승인/동일성/조치 revision만 허용하고 원문·URI를 넣지 않는다. 대상 위치는 같은 connection에 속한다. attempt_no>=0,max_attempts>0이며 만료 lease는 한도 안에서 재예약한다. |
| `secret_scan_runs` | `run_id BIGINT PK`, `job_id BIGINT NOT NULL UQ FK`, `connection_id BIGINT NOT NULL FK`, `trigger_type VARCHAR(20) NOT NULL`, `scope_version BIGINT NOT NULL`, `scheduled_at`, `requested_at TIMESTAMPTZ NOT NULL`, `started_at`, `finished_at TIMESTAMPTZ`, `collection_status`, `detection_status`, `validation_status VARCHAR(20) NOT NULL`, `result_status VARCHAR(20) NOT NULL`, `failure_code VARCHAR(80)`, `created_at TIMESTAMPTZ NOT NULL` | trigger_type=SCHEDULED·MANUAL·REVALIDATION. 당시 connection scope_version과 run_scope 스냅샷을 보존한다. 단계는 여섯 처리 상태, 실행 결과에는 CANCELLED도 포함한다. 현재 연결 표시명은 연결 #ID 등 안전한 별칭으로 제공하며 원문 display_name을 과거 스냅샷처럼 노출하지 않는다. 재검사는 전체 스캔 성공률에서 제외. |
| `secret_run_scopes` | `run_scope_id BIGINT PK`, `run_id`, `scope_id BIGINT NOT NULL FK`, `scope_revision BIGINT`, `snapshot_content_type VARCHAR(80)`, `snapshot_origin VARCHAR(20)`, `safe_display_alias VARCHAR(80)`, `evidence_version SMALLINT`, `collection_status VARCHAR(20) NOT NULL`, `object_count BIGINT`, `excluded_object_count INTEGER NOT NULL DEFAULT 0`, `failure_code VARCHAR(80)`, `detection_stage_status`, `validation_stage_status VARCHAR(20)`, `detection_reason_code`, `validation_reason_code VARCHAR(80)`, `validation_required BOOLEAN`, `started_at`, `finished_at TIMESTAMPTZ` | UQ(run_id,scope_id). 실제 수집 결과와 탐지·검증의 실행/미실행 사유를 보존한다. scope_outcome은 secret_scope_outcomes 뷰에서 필수 단계로 파생한다. 과거 evidence_version 부재는 UNKNOWN·LEGACY_UNAVAILABLE이며 필수 단계 성공을 추정하지 않는다. 원천 참조·리전은 보호된 승인 설정에서만 읽고 별칭은 당시 안전값만 복제한다. |
| `secret_tool_runs` | `tool_run_id BIGINT PK`, `run_scope_id BIGINT NOT NULL FK`, `engine VARCHAR(20) NOT NULL`, `engine_version VARCHAR(100) NOT NULL`, `config_hash VARCHAR(100) NOT NULL`, `fingerprint_key_version VARCHAR(40)`, `scope_origin_version BIGINT`, `status VARCHAR(20) NOT NULL`, `finding_count BIGINT`, `failure_code VARCHAR(80)`, `started_at`, `finished_at TIMESTAMPTZ` | UQ(run_scope_id,engine), finding_count>=0. 실제 탐지 시도의 엔진·설정·HMAC 키 버전과 출처 승인 버전이다. 과거 NULL은 비교 가능한 부재를 증명하지 못한다. 전체 스캔의 미발견도 최신 양성 Finding의 이 버전들과 현재 범위를 비교하며 불일치는 UNKNOWN이다. 원문 HMAC 키·지문을 반환하지 않는다. |

### 3.3. 키·위치·탐지·평가

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_credential_sets` | `credential_set_id BIGINT PK`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `component_state VARCHAR(20) NOT NULL`, `issuer_connection_id BIGINT FK`, `fingerprint BYTEA`, `candidate_locator BYTEA`, `fingerprint_key_version VARCHAR(40)`, `masked_label VARCHAR(120)`, `external_key_ref VARCHAR(300)`, `owner_ref VARCHAR(200)`, `purpose_label VARCHAR(200)`, `issued_at TIMESTAMPTZ`, `expiry_policy VARCHAR(10) NOT NULL`, `expires_at TIMESTAMPTZ`, `lifecycle_status VARCHAR(20) NOT NULL`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `created_at TIMESTAMPTZ NOT NULL`; `first_seen_run_id BIGINT FK` | CONFIRMED는 보호키 HMAC fingerprint, CANDIDATE는 원천 위치를 포함한 HMAC candidate_locator, METADATA_ONLY는 값 지문 없이 보호된 목록 참조다. 마스킹·접두사 병합 금지. FIXED는 expires_at 필수. 최초 동일성 삽입 실행만 first_seen_run_id를 고정하고 구성 승격·새 위치로 덮지 않는다. 근거 불명 과거 행은 NULL. 메타데이터·재검사는 신규 값 후보 집계 제외. |
| `secret_occurrences` | `occurrence_id BIGINT PK`, `credential_set_id`, `connection_id`, `scope_id BIGINT NOT NULL FK`, `asset_id BIGINT FK`, `source_object_ref VARCHAR(500) NOT NULL`, `location_key VARCHAR(500) NOT NULL`, `latest_object_version VARCHAR(200)`, `environment`, `exposure_status`, `exposure_range`, `presence_status VARCHAR(10) NOT NULL`, `first_seen_at`, `last_seen_at TIMESTAMPTZ NOT NULL`, `last_seen_run_id BIGINT FK`, `last_complete_coverage_at TIMESTAMPTZ`, `exposure_assessment_version BIGINT NOT NULL`, `policy_assessed_binding_id BIGINT FK`, `policy_assessed_origin_version BIGINT`, `policy_recheck_required BOOLEAN NOT NULL` | UQ(credential_set_id,connection_id,source_object_ref,location_key), scope는 같은 연결 소속. 구성·존재·노출은 독립이며 현재 판정 ID/근거와 커버리지 상세는 원본 이력에서 조회한다. 실패는 UNKNOWN으로 기록하되 마지막 완전 검사 시각·수동 판정은 보존한다. 재취득 참조는 제한 접근하고 일반 표시는 위치 #ID다. |
| `secret_findings` | `finding_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `tool_run_id BIGINT NOT NULL FK`, `finding_key VARCHAR(100) NOT NULL`, `rule_id VARCHAR(120) NOT NULL`, `rule_version VARCHAR(100) NOT NULL`, `masked_context TEXT`, `detected_at TIMESTAMPTZ NOT NULL` | `UQ(tool_run_id, finding_key)`. 마스킹된 문맥만 저장한다. 동일 위치의 두 엔진 결과를 둘 다 남기되 위치를 두 번 만들지 않는다. `tool_run_id`와 `run_id`의 관계를 적재 트랜잭션에서 검증한다. |
| `secret_validation_attempts` | `attempt_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `job_id BIGINT NOT NULL FK`, `validator_id VARCHAR(120) NOT NULL`, `validator_version VARCHAR(80) NOT NULL`, `endpoint_class VARCHAR(80) NOT NULL`, `attempt_state VARCHAR(20) NOT NULL`, `requested_at TIMESTAMPTZ NOT NULL`, `request_sent_at`, `completed_at`, `provider_checked_at TIMESTAMPTZ`, `result VARCHAR(10)`, `reason_code VARCHAR(80)`, `next_due_at TIMESTAMPTZ`, 보호된 실행 동일성·승인/조치 revision·현재 적용 적격성 | (run_id,job_id) 복합 FK. 공급자별 테이블을 만들지 않고 [검증 계약](credential-validation.md)의 RESERVED·SENT·COMPLETED·ABANDONED를 사용한다. 완료 시에만 4상태 결과와 사유가 필요하며 진행 예약을 최신 결과로 조회하지 않는다. 실제 관측·전송·종료 시각을 분리하고 미전송·timeout을 인증 성공 시각으로 채우지 않는다. 승인 취소·동일성 변경의 응답은 과거 시도로만 보존한다. 조치 이후 관측 부재는 POST_ACTION_VALIDATION_REQUIRED이며 과거 이력을 덮지 않는다. |
| `secret_aws_access_key_observations` | `credential_set_id BIGINT NOT NULL FK`, `run_id BIGINT NOT NULL FK`, `key_status VARCHAR(10) NOT NULL`, `issued_at`, `last_used_at TIMESTAMPTZ`, `last_used_service VARCHAR(120)`, `last_used_region VARCHAR(80)`, `observed_at TIMESTAMPTZ NOT NULL` | PK(credential_set_id,run_id). 키 상태 ACTIVE·INACTIVE·EXPIRED·UNKNOWN. 목록 ACTIVE는 VALID·노출·권한의 증거가 아니며 기록 부재는 미사용이 아니다. 사용자명·키 ID 원문·Secret 값 금지. |
| `secret_exposure_assessments` | `assessment_id BIGINT PK`, `occurrence_id BIGINT NOT NULL FK`, `finding_id BIGINT`, `finding_run_id BIGINT`, `finding_rule_id VARCHAR(120)`, `finding_rule_version VARCHAR(100)`, `from_status`, `decision VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `scope_origin VARCHAR(20) NOT NULL`, `scope_version`, `scope_revision BIGINT`, `policy_id`, `policy_version`, `policy_binding_id BIGINT FK`, `policy_result VARCHAR(10)`, `decision_source VARCHAR(20) NOT NULL`, `evidence_refs JSONB NOT NULL`, `observed_at TIMESTAMPTZ`, `actor_user_id BIGINT FK`, `automatic_policy_version BIGINT`, `assessed_at TIMESTAMPTZ NOT NULL`, `version BIGINT NOT NULL` | UQ(occurrence_id,version). API decisionId의 단일 원본이며 NORMAL·ABNORMAL·NEEDS_MORE_INFO·FALSE_POSITIVE를 보존한다. HUMAN은 수행자, POLICY는 정확한 ALLOWED의 NORMAL과 정책 버전 필요. 불완전 후보·추가 조사나 과거 근거가 없는 Finding 필드는 NULL이다. 자동 ABNORMAL 금지. |
| `secret_finding_dispositions` | `disposition_id BIGINT PK`, `decision_id BIGINT NOT NULL UQ FK`, `finding_id BIGINT NOT NULL`, `occurrence_id`, `scope_id BIGINT NOT NULL FK`, `scope_revision`, `scope_origin_version BIGINT NOT NULL`, `scope_fingerprint VARCHAR(32) NOT NULL`, `source_object_digest`, `location_digest BYTEA NOT NULL`, `object_version VARCHAR(200)`, `rule_version`, `engine_version VARCHAR(100) NOT NULL`, `protected_evidence_identity BYTEA`, `identity_key_version VARCHAR(40)`, `reason_code VARCHAR(80) NOT NULL`, `decision_source VARCHAR(30) NOT NULL`, `reused_from_disposition_id BIGINT FK`, `reviewed_by_user_id BIGINT FK`, `created_at TIMESTAMPTZ NOT NULL`, `reuse_expires_at`, `revoked_at TIMESTAMPTZ`, `is_current BOOLEAN NOT NULL` | Finding당 현재 교정 최대 하나. FALSE_POSITIVE는 자격 증명 아님이며 NORMAL·INVALID가 아니다. 같은 범위·객체 버전·위치·룰/엔진·후보 동일성이 모두 같은 재탐지만 최대 30일 EXACT_EVIDENCE_REUSE 행을 새로 쓴다. 근거 부재·만료·취소 시 재사용 금지. 원문 원천 참조를 재사용 색인에 넣지 않는다. |

심각도·권한 영향의 입력과 등급 기준은 [심각도 정책](../02-functional-design/severity-policy.md), 규칙·정규화 근거·평가 저장은 [심각도 평가 계약](risk-assessment-contract.md)을 따른다. 판정·검증·수명주기 원장은 FK/검증된 참조로 재사용하며 일반 평가 테이블에 복제하지 않는다. 유효한 영향 근거·집계가 없으면 응답은 UNASSESSED와 집계 미제공 사유를 보존한다.

| 심각도 전용 테이블 | 저장 책임 | 핵심 제약 |
|---|---|---|
| `secret_severity_policies` | 불변 정책 버전·활성화·검토/수명주기 | 활성 정책과 과거 평가 재현을 분리. 활성 버전의 조건 수정 금지 |
| `secret_severity_rules` | 정책별 규칙 ID·우선순위·타입 제한 조건·결과 | 정책/규칙과 정책/우선순위 고유, 명시적 ANY와 UNKNOWN 구분, 임의 SQL·스크립트 금지 |
| `secret_risk_evidence` | 권한·실제 대상·자산 중요도·악용/조사의 정규화 근거·시각·적용 범위·매핑 완전성 | 같은 키/위치·현재 승인·ACL 검사, 추가 전용 보완·취소 이력, 원문·공급자 응답 제외 |
| `secret_severity_assessments` | 위치별 자동 평가·수동 조정·승인/반려·적용과 당시 입력·근거 참조 | ASSESSED/UNASSESSED/NOT_APPLICABLE과 등급 NULL 제약, 동일 입력 멱등, revision CAS, 과거 평가 불변 |

이 네 테이블의 컬럼·FK·CHECK·색인과 유효 head는 [평가 계약](risk-assessment-contract.md)이 정의한다. 화면용 상태 이동·공급자별 중복 원장·별도 재평가 큐는 추가하지 않는다. 정책·규칙과 평가 기능의 실제 스키마 변경은 하나의 일관된 변경 단위로 검토하며 이 문서에 마이그레이션 번호를 미리 고정하지 않는다.

`secret_exposure_assessments`에는 `exposure_range VARCHAR(10)`과 `access_snapshot JSONB`를 함께 보존한다. access_snapshot은 공개성·접근 범위·완전성·관측 시각·승인 revision·검증된 typed 원본 참조의 닫힌 schema다. 원래 ACL·사용자 명단·공급자 응답은 넣지 않는다. 과거 행의 NULL은 UNKNOWN 근거 부재로 보존하며 현재 범위 컬럼만 읽어 과거 결정을 소급 재현하지 않는다. 이는 기존 노출 원장의 확장이며 위험 근거 테이블에 노출 사실을 다시 적재하지 않는다.

### 3.4. 조사·원자 명령

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_investigation_tasks` | `task_id BIGINT PK`, `target_type VARCHAR(20) NOT NULL`, `occurrence_id`, `scope_id BIGINT FK`, `missing_reason_code VARCHAR(80) NOT NULL`, `creation_source VARCHAR(30) NOT NULL`, `assignee_user_id BIGINT FK`, `due_at TIMESTAMPTZ`, `status VARCHAR(20) NOT NULL`, `evidence_ref VARCHAR(120)`, `version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL`, `closed_at TIMESTAMPTZ`, `resolving_decision_id`, `restored_run_scope_id BIGINT FK`, `scope_origin_decision_id BIGINT FK`, `policy_binding_id BIGINT FK`, `accepted_evidence_history_id BIGINT FK` | OCCURRENCE·SCOPE와 해당 FK 배타 지정. HUMAN_MORE_INFO는 담당·기한 필수, AUTO_TEST_ALERT·HUMAN_HANDOFF는 각각 nullable이며 배정 대기로 보존. 같은 대상·질문의 열린 요청은 재사용한다. 기한 NULL은 기한 초과 제외. 종류에 맞는 동일 대상·현재 버전의 판정·복구 실행·분류 결정·정책 연결 또는 적격 SECURITY가 수용한 동일 과제의 소유자·자료 보완 이력으로만 닫으며 과제 종료가 정상 판정을 만들지 않는다. |
| `secret_investigation_history` | `history_id BIGINT PK`, `task_id BIGINT NOT NULL FK`, `event_type VARCHAR(40) NOT NULL`, `actor_user_id BIGINT FK`, `safe_changes JSONB NOT NULL`, `evidence_ref VARCHAR(120)`, `task_version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 배정·기한·증거·종료 추가 전용. 서버 허용 필드만 저장하고 과제 변경으로 판정/실행 사실을 만들지 않는다. |
| `secret_command_idempotency` | `command_id BIGINT PK`, `actor_user_id BIGINT NOT NULL FK`, `command_kind VARCHAR(80) NOT NULL`, `idempotency_key VARCHAR(120) NOT NULL`, `safe_request_digest BYTEA NOT NULL`, `safe_result JSONB NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | UQ(actor_user_id,command_kind,idempotency_key). 원문 본문 없이 안전한 SHA-256 해시·결과 객체를 명령과 원자 저장한다. 같은 본문은 같은 결과, 다른 본문은 409이며 대상 ID는 명령별 safe_result에 둔다. |

### 3.5. 대응·위치 조치·해결

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_response_cases` | `case_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `previous_case_id BIGINT FK`, `opening_occurrence_id BIGINT NOT NULL FK`, `opening_decision_id BIGINT FK`, `opened_by_user_id BIGINT NOT NULL FK`, `opening_reason VARCHAR(80) NOT NULL`, `case_origin VARCHAR(20) NOT NULL`, `is_current BOOLEAN NOT NULL`, `disposition VARCHAR(20) NOT NULL`, `action_status VARCHAR(30) NOT NULL`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT FK`, `version BIGINT NOT NULL`, `opened_at`, `updated_at TIMESTAMPTZ NOT NULL`, `resolved_at`, `closed_at TIMESTAMPTZ` | 같은 키·PRODUCTION 또는 TEST 출처 현재 건 최대 하나. 새 개설 결정·위치는 같은 키·출처의 유효 ABNORMAL 근거다. 과거 건의 결정 ID는 근거 없이 소급 생성하지 않는다. OPEN·IN_PROGRESS·REVALIDATION_PENDING·RESOLVED를 사용하고 예외는 독립 보존한다. 개설 결정을 다시 승인하는 검토/빈 시작 게이트 없음. 해결 후 재발은 previous_case_id를 가진 새 건이다. |
| `secret_case_occurrences` | `case_id`, `occurrence_id`, `credential_set_id BIGINT NOT NULL FK`, `linked_decision_id BIGINT FK`, `assigned_team VARCHAR(100)`, `assigned_user_id BIGINT FK`, `response_status VARCHAR(30) NOT NULL`, `version`, `action_cycle BIGINT NOT NULL`, `last_material_action_at`, `actions_completed_at`, `linked_at TIMESTAMPTZ NOT NULL`, `resolved_at TIMESTAMPTZ`, 복합 PK `(case_id,occurrence_id)` | 건·위치·결정의 키·출처를 검사한다. action_cycle은 해당 위치의 현재 조치 주기 번호이며 별도 주기 테이블은 없다. 새 주기에서 현재 완료/조치 시각은 초기화하고 이전 기록의 주기 번호·활동을 보존한다. 위치 담당자는 해당 연결의 적격 REMEDIATOR다. |
| `secret_location_activities` | `activity_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT`, `activity_type VARCHAR(40) NOT NULL`, `actor_user_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `occurrence_version BIGINT`, `reference_id BIGINT`, `reason_code VARCHAR(40)`, `happened_at TIMESTAMPTZ NOT NULL` | 위치 담당·조치·기록 마침·재개·체크·이관·예외·해결 활동. occurrence_id가 있으면 건/위치 복합 FK. 조치 주기별 원본 기록과 참조 ID를 연결하고 일반 API는 허용 위치 활동만 읽는다. |
| `secret_assignment_history` | `assignment_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `from_team`, `to_team VARCHAR(100)`, `from_user_id`, `to_user_id BIGINT FK`, `reason VARCHAR(80) NOT NULL`, `changed_by_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 건 조정자 변경 이력이다. 위치 담당 변경은 secret_location_activities에 위치 버전·사유와 함께 보존한다. 건 조정자를 지정해도 위치 조치 권한은 늘어나지 않는다. |
| `secret_action_records` | `action_id BIGINT PK`, `case_id`, `occurrence_id BIGINT NOT NULL FK`, `action_cycle BIGINT NOT NULL`, `action_type VARCHAR(40) NOT NULL`, `performed_at`, `recorded_at TIMESTAMPTZ NOT NULL`, `recorded_by_user_id BIGINT NOT NULL FK`, `evidence_ref VARCHAR(71) NOT NULL`, `case_version BIGINT NOT NULL` | 건/위치 복합 FK와 현재 action_cycle로 연결한다. ROTATED·REVOKED·REMOVED_FROM_SOURCE·PERMISSION_REDUCED의 지정 담당자 외부 수행 기록이며 제품 실행·공급자 확인이 아니다. 티켓/변경 참조와 최근 30일~요청+5분 시각만 허용한다. |
| `secret_checklist_items` | `item_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `occurrence_id BIGINT`, `guide_version`, `item_code VARCHAR(80) NOT NULL`, `status VARCHAR(20) NOT NULL`, `completed_at TIMESTAMPTZ`, `completed_by_user_id BIGINT FK`, `evidence_ref VARCHAR(71)`, `updated_at TIMESTAMPTZ NOT NULL` | UQ NULLS NOT DISTINCT(case_id,guide_version,item_code,occurrence_id), 건/위치 복합 FK. 새 위치에는 위치별 네 기본 항목을 생성한다. 과거 건 단위 항목은 보존하되 같은 새 항목을 중복 생성하지 않는다. 체크는 조치 마침·재검사·해결을 대체하지 않는다. |
| `secret_checklist_history` | `history_id BIGINT PK`, `case_id`, `item_id BIGINT NOT NULL FK`, `from_status`, `to_status VARCHAR(20) NOT NULL`, `evidence_ref VARCHAR(71)`, `changed_by_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 체크·취소·근거 변경을 버전과 함께 추가 전용 이력으로 보존한다. |
| `secret_case_stage_history` | `stage_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `from_status`, `to_status VARCHAR(30) NOT NULL`, `reason_code VARCHAR(40) NOT NULL`, `actor_user_id BIGINT NOT NULL FK`, `changed_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | 첫 실질 조치·기록 마침·새 위치 연결·재조치·근거 해결의 전이 이력. 개설 결정은 별도 opening_decision_id로 보존하며 체크·배정만으로 단계를 바꾸지 않는다. |
| `secret_case_revalidations` | `revalidation_id BIGINT PK`, `case_id`, `occurrence_id`, `action_id`, `connection_id`, `scope_id`, `job_id`, `run_id BIGINT NOT NULL FK`, `requested_by_user_id BIGINT NOT NULL FK`, `idempotency_digest VARCHAR(65) NOT NULL`, `baseline_config_hash`, `baseline_engine_version VARCHAR(100)`, `baseline_fingerprint_version VARCHAR(40) NOT NULL`, `runtime_fingerprint_version VARCHAR(40)`, `case_version BIGINT NOT NULL`, `status VARCHAR(20) NOT NULL`, `scope_revision`, `scope_origin_version BIGINT`, `baseline_identity BYTEA`, `action_cycle`, `occurrence_version BIGINT`, `last_material_action_at`, `actions_completed_at TIMESTAMPTZ`, `failure_code VARCHAR(80)`, `requested_at TIMESTAMPTZ NOT NULL`, `finished_at TIMESTAMPTZ` | UQ(case_id,idempotency_digest), 작업·실행 각각 UQ. API result는 물리 status=PENDING·PRESENT·MISSING·UNKNOWN이며 job 상태와 별개다. 범위·출처 승인·HMAC/엔진/설정·정확한 동일성·위치 버전·주기·조치 완료 이후 비교를 고정한다. 과거 NULL은 해결용 미발견 근거가 아니다. |
| `secret_case_resume_events` | `resume_id BIGINT PK`, `case_id`, `occurrence_id`, `revalidation_id`, `resumed_by_user_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `resumed_at TIMESTAMPTZ NOT NULL` | 재검증에서 키가 남은 `PRESENT` 위치를 명시적으로 다시 조치 단계로 전환한 이력을 남긴다. |
| `secret_case_resolution_checks` | `check_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `target_origin VARCHAR(20) NOT NULL`, `version_digest VARCHAR(100) NOT NULL`, `target_snapshot`, `check_results`, `allowed_coverage_levels JSONB NOT NULL`, `checked_at TIMESTAMPTZ NOT NULL` | 해결 명령 때 재검사한 대상·정책/판정/조치/실행/증거와 허용 수준을 불변 보존한다. GET closure-checks는 최신 검사를 계산하며 조회만으로 이 테이블의 승인 기록을 생성하지 않는다. 필수 BLOCKED·UNKNOWN을 LIMITED로 우회하지 못한다. |
| `secret_case_resolutions` | `case_id BIGINT PK FK`, `resolution_id BIGINT NOT NULL UQ`, `resolved_by_user_id BIGINT NOT NULL FK`, `reason_code VARCHAR(80) NOT NULL`, `case_version BIGINT NOT NULL`, `closure_check_id BIGINT FK`, `coverage_level VARCHAR(20)`, `evidence_snapshot`, `limitations JSONB`, `resolved_at TIMESTAMPTZ NOT NULL` | 새 해결은 검사 ID·FULL/LIMITED/SIMULATION·실제 근거·한계를 불변 보존한다. 과거 해결에 없는 검사 ID·확인 수준을 소급 채우지 않는다. 모든 위치의 현재 권한·버전·필수 근거를 같은 트랜잭션에서 다시 검사한다. |
| `secret_case_resolution_locations` | `case_id BIGINT NOT NULL FK`, `occurrence_id`, `revalidation_id BIGINT NOT NULL FK`, `resolution_id BIGINT FK`, `action_cycle BIGINT`, `last_material_action_id BIGINT FK`, 복합 PK `(case_id,occurrence_id)` | 연결 비정상 위치마다 마지막 실질 조치 뒤 비교 가능한 MISSING·실제 주기와 조치 ID를 고정한다. 과거 미보유 컬럼은 NULL로 보존한다. 정상 보관 위치 삭제와 키 수명주기를 대신하지 않는다. |
| `secret_case_dismissals` | `dismissal_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `reason_code VARCHAR(20) NOT NULL`, `evidence_snapshot JSONB NOT NULL`, `accepted_transfer_ref VARCHAR(120)`, `approved_by_user_id BIGINT NOT NULL FK`, `case_version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | FALSE_POSITIVE는 모든 관련 Finding 교정, OUT_OF_SCOPE는 승인된 범위·소유자 이관·인수 확인 필요. 실제 미해결 노출이 남으면 거부. disposition·is_current=false·closed_at·감사 원자 저장. RESOLVED·위험 수용 수에 넣지 않는다. |
| `secret_case_transfer_evidence` | `evidence_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `reason_code VARCHAR(40) NOT NULL`, `evidence_ref`, `accepted_transfer_ref VARCHAR(120) NOT NULL`, `submitted_by_user_id BIGINT NOT NULL FK`, `submitted_at TIMESTAMPTZ NOT NULL`, `case_version BIGINT NOT NULL` | PROGRAM_SCOPE_ENDED·OWNER_TRANSFER의 안전한 티켓/변경 근거와 인수 확인을 별도 불변 보존한다. 제출만으로 건을 종료하지 않는다. |
| `secret_case_transfer_locations` | `evidence_id`, `occurrence_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `scope_revision`, `scope_origin_version`, `assessment_version BIGINT NOT NULL`, 복합 PK `(evidence_id,occurrence_id)` | 이관 당시 관련 위치·수집 revision·출처 승인·판정 버전을 고정한다. 이후 불일치는 적용 가능한 종료 근거가 아니다. |
| `secret_case_transfer_reviews` | `review_id BIGINT PK`, `evidence_id BIGINT NOT NULL FK`, `decision VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `expected_evidence_version`, `case_version BIGINT NOT NULL`, `reviewed_by_user_id BIGINT NOT NULL FK`, `reviewed_at TIMESTAMPTZ NOT NULL` | UQ(evidence_id,expected_evidence_version), 추가 전용. 제출자와 다른 전체 SECURITY의 APPROVE·REJECT. 최신 승인이며 범위·판정 버전이 같은 증거만 OUT_OF_SCOPE 검토에서 선택한다. |
| `secret_exceptions` | `exception_id BIGINT PK`, `case_id BIGINT NOT NULL FK`, `reason_code VARCHAR(80) NOT NULL`, `evidence_ref VARCHAR(120) NOT NULL`, `approved_by_user_id BIGINT NOT NULL FK`, `approved_at`, `expires_at TIMESTAMPTZ NOT NULL`, `approved_case_version BIGINT NOT NULL`, `revoked_at TIMESTAMPTZ`, `revoked_by_user_id BIGINT FK`, `revoke_reason_code VARCHAR(80)` | 위험 수용 기본 OFF. 활성 정책은 근거가 완전한 MEDIUM/LOW에 최대 7일과 자원 책임자·독립 SECURITY 검토를 요구한다. BUSINESS_CONTINUITY·TEMPORARY_DEPENDENCY·COMPENSATING_CONTROL 사유만 허용하며 과거 승인 이력을 삭제·기간 교정하지 않는다. 비활성 정책에서는 기존 행도 현재 위험 수용 집계에 적용하지 않는다. 위치별 활성 중복 금지. 원래 action_status는 보존하고 전체 관련 위치의 SECURITY 접근에서만 RISK_ACCEPTED를 파생한다. 만료·취소·버전 변경은 효력을 없애며 과거 승인 이력을 보존한다. |
| `secret_exception_occurrences` | `exception_id`, `occurrence_id BIGINT NOT NULL FK`, `scope_id BIGINT NOT NULL FK`, `scope_revision`, `scope_origin_version`, `assessment_version BIGINT NOT NULL`, 복합 PK `(exception_id,occurrence_id)` | 승인 당시 위치·수집 revision·출처 승인·판정 버전 불변 고정. 현재 버전이 달라지거나 새 위치가 생기면 기존 예외로 자동 포함하지 않는다. |

### 3.6. 수명주기·합성 근거·교차 환경 경고

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_credential_lifecycle_evidence` | `evidence_id BIGINT PK`, `case_id`, `credential_set_id`, `source_occurrence_id BIGINT NOT NULL FK`, `origin VARCHAR(20) NOT NULL`, `kind VARCHAR(30) NOT NULL`, `verification_method`, `verification_level VARCHAR(30) NOT NULL`, `evidence_ref VARCHAR(120)`, `provider_validator_id VARCHAR(120)`, `validator_version`, `endpoint_class`, `reason_code VARCHAR(80)`, `performed_at TIMESTAMPTZ`, `observed_at TIMESTAMPTZ NOT NULL`, `submitted_by_user_id BIGINT FK`, `version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 수행 시각 미상은 performed_at=NULL, 관측·서버 접수 시각은 각각 보존. 외부 제출은 EXTERNAL_RECORD/PENDING 불변 원본. 수동 승인은 reviews와 현재 유효 뷰에서 APPROVED_MANUAL로 파생한다. 승인 DB 역할·검증기 capability의 실제 관측만 PROVIDER/PROVIDER_VERIFIED를 생성한다. 이후 상충 VALID로 현재 적용을 취소하며 공급자 본문·키 원문은 저장하지 않는다. |
| `secret_lifecycle_evidence_reviews` | `review_id BIGINT PK`, `evidence_id BIGINT NOT NULL FK`, `decision VARCHAR(20) NOT NULL`, `reason_code VARCHAR(80) NOT NULL`, `expected_evidence_version`, `case_version BIGINT NOT NULL`, `reviewed_by_user_id BIGINT NOT NULL FK`, `limitations JSONB NOT NULL`, `reviewed_at TIMESTAMPTZ NOT NULL` | APPROVE·REJECT 추가 전용. 검토자는 제출자와 다르고 전체 적용 위치의 SECURITY여야 한다. 현재 수동 수준은 APPROVED_MANUAL이며 LIMITED 상한. 공급자 확인으로 승격 금지. 취소·상충·적용 시각은 해결 시 재검사한다. |
| `secret_lifecycle_evidence_applications` | `application_id BIGINT PK`, `evidence_id`, `case_id`, `credential_set_id BIGINT NOT NULL FK`, `origin VARCHAR(20) NOT NULL`, `reviewed_by_user_id BIGINT NOT NULL FK`, `reviewed_at TIMESTAMPTZ NOT NULL` | UQ(evidence_id,case_id). 다른 출처 건 재사용은 키·시각·현재 건·권한을 검토한 관계만 추가하고 원 증거 복제/변경 금지. 위치별 조치 후 부재는 별도 필수. |
| `secret_lifecycle_provider_writers`·`secret_lifecycle_provider_capabilities` | 승인 DB 역할; `provider_validator_id`·`validator_version`·`endpoint_class`·`kind`의 허용 조합 | 운영자가 승인한 DB 쓰기 주체와 검증기만 공급자 수명주기 근거를 적재한다. 기본 허용 목록은 비어 있으며 일반 HTTP나 수동 SECURITY 승인으로 이 수준을 생성할 수 없다. |
| `secret_fixture_manifests` | `manifest_id`, `version BIGINT NOT NULL`, `protected_manifest_ref VARCHAR(300) NOT NULL`, `approval_status VARCHAR(20) NOT NULL`, `approved_by_user_id BIGINT FK`, `valid_from TIMESTAMPTZ NOT NULL`, `valid_until`, `revoked_at TIMESTAMPTZ`, 복합 PK `(manifest_id,version)` | 승인·버전 고정 합성 등록물의 보호 참조. 유효 기간·승인/취소 이력 보존. 일반 API에는 값·지문·다른 표본 목록 금지. |
| `secret_fixture_manifest_identities` | `identity_id BIGINT PK`, `manifest_id`, `manifest_version`, `scope_id`, `scope_revision BIGINT NOT NULL`, `issuer VARCHAR(40) NOT NULL`, `credential_type VARCHAR(80) NOT NULL`, `fingerprint_key_version VARCHAR(40) NOT NULL`, `protected_fingerprint BYTEA NOT NULL` | 내부 전용의 정확한 HMAC 동일성 등록. manifest 버전 복합 FK와 범위 revision에 고정하며 일반 API·로그·AI에 값·지문·다른 표본 목록을 반환하지 않는다. |
| `secret_fixture_match_evidence` | `match_evidence_id BIGINT PK`, `credential_set_id`, `scope_id BIGINT NOT NULL FK`, `scope_revision BIGINT NOT NULL`, `manifest_id`, `manifest_version BIGINT NOT NULL`, `run_scope_id BIGINT NOT NULL FK`, `comparator_id VARCHAR(120) NOT NULL`, `comparator_version VARCHAR(80) NOT NULL`, `match_result VARCHAR(20) NOT NULL`, `protected_match_ref VARCHAR(300)`, `observed_at TIMESTAMPTZ NOT NULL` | manifest 복합 FK. 승인 워커/비교기가 보호된 정확한 키 동일성 비교. TEST 표식·마스킹 끝자리·사용자 선언은 근거 아님. 키·scope revision·manifest 버전에 고정. |
| `secret_test_fixture_attestations` | `attestation_id BIGINT PK`, `credential_set_id`, `scope_id BIGINT NOT NULL FK`, `scope_revision BIGINT NOT NULL`, `manifest_id`, `manifest_version BIGINT NOT NULL`, `match_evidence_id BIGINT NOT NULL FK`, `confirmed_by_user_id BIGINT FK`, `confirmation_source VARCHAR(20) NOT NULL`, `status VARCHAR(20) NOT NULL`, `confirmed_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ` | 매칭 근거 중복 확인 금지. HUMAN 또는 승인된 자동 매칭 정책. manifest 만료/취소·scope 개정·상충 실제 키 근거가 있으면 기존 확인으로 SIMULATION 불가. 위치 제거·조치 후 재검사를 대신하지 않는다. |
| `secret_cross_environment_alerts` | `alert_id BIGINT PK`, `credential_set_id BIGINT NOT NULL FK`, `previous_alert_id BIGINT FK`, `status VARCHAR(30) NOT NULL`, `assignee_user_id BIGINT FK`, `due_at TIMESTAMPTZ`, `test_case_id`, `investigation_task_id BIGINT FK`, `version BIGINT NOT NULL`, `first_observed_at`, `last_observed_at TIMESTAMPTZ NOT NULL`, `acknowledged_at`, `closed_at TIMESTAMPTZ`, `closure_evidence_snapshot JSONB` | 키당 열린 경고 최대 하나. 완전한 현재 TEST 값 후보·유효 Finding·합성 미확인에서 SUSPECTED_REAL, 실제 발급자 근거로 CONFIRMED_REAL. CANDIDATE·METADATA_ONLY·모든 Finding 오탐 제외. 건/과제 하나 이상 연결하고 둘 다 없으면 과제 원자 생성. 담당·기한 각각 nullable. 인지는 종료가 아니며 반증 또는 실제 수명주기 근거로 닫는다. |
| `secret_cross_environment_alert_locations` | `alert_id`, `occurrence_id BIGINT NOT NULL FK`, `last_observed_at TIMESTAMPTZ NOT NULL`, 복합 PK `(alert_id,occurrence_id)` | 반복 스캔·추가 위치는 열린 경고 갱신. 조회별 위치 ACL 검사, 운영 ResponseCase 수에 합산 금지. |
| `secret_cross_environment_alert_history` | `history_id BIGINT PK`, `alert_id BIGINT NOT NULL FK`, `event_type VARCHAR(40) NOT NULL`, `actor_user_id BIGINT FK`, `safe_changes JSONB NOT NULL`, `alert_version BIGINT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL` | 배정·인지·승격·반증 검토·종료의 안전한 변경 이력을 추가 전용 보존한다. 인지는 종료가 아니다. |

현재 유효 Finding·보관 정책·합성 증명·수명주기는 각각 `secret_current_valid_findings`, `secret_occurrence_policy_results`, `secret_test_fixture_status`, `secret_current_lifecycle_evidence` 뷰에서 현재 조건을 다시 평가한다. `secret_current_finding_dispositions`는 만료·취소·상충 실제 키 근거를 배제한 현재 교정을 제공한다. 원본 이력과 호환 조회 뷰는 별도 복제 저장소가 아니다.

### 3.7. 역할·감사

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_role_grants` | `grant_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `role VARCHAR(30) NOT NULL`, `connection_id BIGINT FK`, `granted_by_user_id BIGINT NOT NULL FK`, `grant_source VARCHAR(20) NOT NULL`, `granted_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ` | 역할 SECURITY·REMEDIATOR·OPERATIONS, 부여 출처 BOOTSTRAP·MANUAL. 연결 NULL은 명시 승인 전역 권한. 활성 중복 금지. ADMIN과 별개이며 취소를 재시작으로 복구하지 않는다. 권한 변경은 스냅샷 ACL revision 무효화. |
| `secret_team_memberships` | `membership_id BIGINT PK`, `user_id BIGINT NOT NULL FK`, `team_name VARCHAR(100) NOT NULL`, `added_by_user_id BIGINT NOT NULL FK`, `added_at TIMESTAMPTZ NOT NULL`, `removed_at TIMESTAMPTZ` | 팀 담당자 지정 권한 확인에 사용한다. 팀 문자열만으로 소속을 추정하지 않는다. 활성 중복 소속을 막는다. |
| `secret_audit_events` | `audit_id BIGINT PK`, `case_id BIGINT FK`, `case_version BIGINT`, `event_type VARCHAR(80) NOT NULL`, `actor_user_id BIGINT NOT NULL FK`, `target_type VARCHAR(40)`, `target_id`, `target_version BIGINT`, `command_id BIGINT FK`, `safe_changes JSONB NOT NULL`, `occurred_at TIMESTAMPTZ NOT NULL` | 결정·승인·담당·조치·증거·해결과 안전한 변경을 감사한다. 대상은 CASE·SCOPE·RUN 등의 내부 ID로 식별하고 공통 명령의 멱등 결과와 연결한다. 원문·자유 메모·공급자 응답 금지. |

### 3.8. 일관된 읽기 스냅샷

| 테이블 | 컬럼과 타입 | 핵심 제약과 의미 |
|---|---|---|
| `secret_query_snapshots` | `snapshot_id UUID PK`, `scope VARCHAR(20) NOT NULL`, `owner_user_id BIGINT NOT NULL FK`, `case_origin VARCHAR(20)`, `as_of`, `created_at`, `expires_at TIMESTAMPTZ NOT NULL`, `authorization_revision`, `scope_version_digest VARCHAR(100) NOT NULL`, `normalized_filters JSONB NOT NULL`, `filter_digest VARCHAR(100) NOT NULL`, `partial_coverage BOOLEAN`, `restricted_coverage BOOLEAN NOT NULL`, `coverage_status VARCHAR(30) NOT NULL`, `missing_sources JSONB NOT NULL`, `retained_from TIMESTAMPTZ`, `history_complete BOOLEAN NOT NULL` | WORKBENCH·CURRENT_CASES·INVENTORY·HISTORY. TTL 상한 15분, 사용자당 유효 5개. 허용 근거·전체 위험 수용 효력의 종료가 더 빠르면 만료 시각을 줄인다. 사용자·역할/범위 revision·필터·출처 종속. REPEATABLE READ에서 헤더·행·membership 확정. 근거/권한 부족이면 partial_coverage NULL, 실패를 false로 만들지 않는다. |
| `secret_snapshot_owner_locks` | `user_id BIGINT PK FK`, `epoch BIGINT NOT NULL` | 사용자별 스냅샷 생성·정리 경합을 직렬화한다. |
| `secret_snapshot_retirements` | `snapshot_id UUID PK`, `owner_user_id BIGINT NOT NULL FK`, `retired_at TIMESTAMPTZ NOT NULL` | 만료·상한 초과·무효화된 헤더의 폐기 표식. 24시간 보존해 만료 409와 알 수 없는 ID 404를 구분하고 다른 사용자의 존재는 노출하지 않는다. |
| `secret_snapshot_subjects` | `snapshot_id UUID NOT NULL FK`, `subject_type VARCHAR(30) NOT NULL`, `subject_id BIGINT NOT NULL`, `credential_set_id BIGINT`, `safe_sort_key VARCHAR(200) NOT NULL`, `safe_facts JSONB NOT NULL`, `subject_version VARCHAR(100)`, 복합 PK `(snapshot_id,subject_type,subject_id)` | ResponseCase·CredentialSet·DecisionOccurrence·LinkOccurrence·GapScope·UnclassifiedScope·CrossEnvironmentAlert·ScanRun 등 typed projection 구분. 같은 숫자 ID 충돌 금지. 부모·ACL은 생성 트랜잭션 검사. 허용 사실만 복제하며 원천 참조·권한 밖 수/시각 금지. |
| `secret_snapshot_location_memberships` | `snapshot_id UUID NOT NULL`, `subject_type VARCHAR(30) NOT NULL`, `subject_id BIGINT NOT NULL`, `occurrence_id BIGINT NOT NULL FK`, 복합 PK `(snapshot_id,subject_type,subject_id,occurrence_id)` | subjects 복합 FK. 안전한 투영과 별도의 내부 위치 ACL 집합이며 매 페이지에서 현재 권한을 재검사한다. 소스·환경은 같은 허용 위치에서 필터하고 한 대상은 한 번만 센다. |
| `secret_snapshot_metrics` | `snapshot_id UUID NOT NULL FK`, `metric_key VARCHAR(80) NOT NULL`, `unit VARCHAR(30) NOT NULL`, `status VARCHAR(30) NOT NULL`, `value BIGINT`, `reason_code VARCHAR(80)`, `applied_filters`, `not_applicable_filters JSONB NOT NULL`, 복합 PK `(snapshot_id,metric_key)` | AVAILABLE만 value>=0. NOT_AUTHORIZED·LEGACY_UNAVAILABLE·UNAVAILABLE은 NULL. Mixed 최상위 합계 없음. 적용/해당 없음 필터·권한 제한 분리. |
| `secret_snapshot_metric_memberships` | `snapshot_id UUID NOT NULL`, `metric_key VARCHAR(80) NOT NULL`, `subject_type VARCHAR(30) NOT NULL`, `subject_id BIGINT NOT NULL`, 복합 PK `(snapshot_id,metric_key,subject_type,subject_id)` | metrics·subjects 복합 FK. 카드·도착 대기열·목록은 동일 membership. 미지정은 대응 건 부분집합이며 추가 노출 수가 아니다. |

## 4. 원자성과 보존 불변 조건

| 변경 경계 | 불변 조건 |
|---|---|
| 판정·건 연결 | 최신 구성·존재·Finding 소속·위치/범위/정책 버전·ACL을 검사하고 결정·현재 건·위치 연결·조치 주기·감사·멱등 결과를 한 트랜잭션에 확정한다. 같은 키·출처 현재 건 고유성이 최후 방어선이다. |
| 범위 분류·승인 | 현재 권한·변경 영향으로 DIRECT/INDEPENDENT_REVIEW를 결정한다. 제안·직접 결정 또는 독립 검토, 범위·정책 바인딩·영향 감사·멱등 결과를 같은 명령 경계에서 보존한다. 화면의 미리보기 방식으로 권한을 우회하지 못한다. |
| 내부 담당 요청 | 기존 investigation_tasks/history를 재사용하고 같은 대상·질문의 열린 요청 고유성을 보장한다. 요청 자체가 수신자의 권한을 늘리거나 분류·판정·복구 성공을 만들지 않는다. |
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
  ON secret_jobs (connection_id)
  WHERE kind IN ('FULL_SCAN', 'REVALIDATE') AND status IN ('PENDING', 'LEASED');
CREATE UNIQUE INDEX uq_secret_scheduled_job
  ON secret_jobs (connection_id, kind, scheduled_at)
  WHERE requested_by_user_id IS NULL AND kind = 'FULL_SCAN';
CREATE UNIQUE INDEX uq_secret_enabled_scope
  ON secret_connection_scopes (connection_id, scope_key) WHERE enabled;
CREATE UNIQUE INDEX uq_secret_open_real_key_alert
  ON secret_cross_environment_alerts (credential_set_id) WHERE closed_at IS NULL;
CREATE INDEX idx_secret_jobs_available
  ON secret_jobs (available_at, job_id) WHERE status = 'PENDING';
```

ASSESS의 활성 작업은 `(target_occurrence_id, expected_evidence_revision, expected_context의 정책 버전)`별로 유일해야 한다. 정책 버전은 서버 허용 구조의 JSONB scalar로 고정하며 식 인덱스로 고유성을 강제한다. 같은 위치·근거의 자동 평가와 요청은 같은 의미일 때 재사용한다. 외부 호출 직렬화 색인에 ASSESS를 포함하지 않는다. 모든 ASSESS는 대상 위치·기대 revision·정책 버전을 요구하고, FULL_SCAN은 전체 연결 목적이며 대상 위치가 없다. REVALIDATE의 목적·대상 제약은 [검증 계약](credential-validation.md)을 따른다.

| 특수 FK | 참조 대상 |
|---|---|
| opening_decision_id·linked_decision_id·resolving_decision_id | secret_exposure_assessments.assessment_id. 같은 위치·키·출처 적용을 별도 검사 |
| lifecycle evidence 참조 | 해결 evidence_snapshot과 명시적 applications가 secret_credential_lifecycle_evidence.evidence_id를 참조. credential_sets에 단일 현재 근거 ID를 덮어쓰지 않음 |
| previous_case_id·test_case_id·case_id | secret_response_cases.case_id |
| previous_alert_id | secret_cross_environment_alerts.alert_id |
| action_cycle | 현재 case_occurrences의 위치별 주기 번호와 action_records·case_revalidations·resolution_locations의 당시 주기 값. 별도 FK/테이블이 아닌 비교 근거 |
| action_id·last_material_action_id | secret_action_records.action_id |
| closure_check_id·resolution_id·revalidation_id | 각각 secret_case_resolution_checks·secret_case_resolutions·secret_case_revalidations의 PK |
| match_evidence_id·manifest_id/manifest_version | 각각 secret_fixture_match_evidence의 PK·secret_fixture_manifests의 복합 PK |

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

1. 수집 범위 허가와 출처·정책을 분리한다. 영향별 직접 결정·독립 검토로 출처 버전·정책 바인딩·감사를 고정하고, 이름·공급자·환경으로 운영/시험을 추정하지 않는다. 범위 설정 사유는 현재 분류·정책·대기·영향 처리에서 계산하며 일부 사유만 해소돼도 나머지를 유지한다.
2. 도래한 작업은 FOR UPDATE SKIP LOCKED로 선점해 임대 상태·소유자·시각을 원자 저장한다. 6시간 정기는 실제 예약 시각 기준이며 수동 실행은 그 시각을 바꾸지 않는다. 중단 중 놓친 슬롯은 한 번만 예약하고 연속 재생하지 않는다. DB 근거만 평가하는 ASSESS는 원천 스캔 분모·성공 횟수에 넣지 않는다.
3. 도구 출력·원문은 승인 워커 메모리에서 정규화·HMAC·선택 검증 후 폐기한다. 동일성 고유 제약 아래 최초 삽입 실행을 고정하고 재발견·새 위치·구성 승격에서 변경하지 않는다.
4. 같은 키·원천 위치를 재사용하고 Finding은 엔진별 보존한다. 전체 스캔은 작업·실행 전체 성공과 해당 범위 필수 수집·탐지 성공을 확인하고 최신 양성 Finding의 범위·출처 승인·연결·설정·엔진·HMAC 버전이 현재와 같을 때만 미관측 위치를 MISSING으로 기록한다. Notion 현재 페이지는 자동 부재 확정 대상에서 제외하고 미관측 위치를 UNKNOWN으로 보존한다. 실패·부분 실행·증거 부재·불일치는 UNKNOWN이다. 이미 완전한 동일성 키가 있는 과거 partial 후보를 근거 없이 합치지 않으며 그 후보의 현재 부재도 이 비교 절차를 따른다. 수동 노출·수명주기 사실과 마지막 완전 검사 이력은 보존한다.
5. IAM 목록은 METADATA_ONLY·실행별 관측이다. Secrets Manager·SSM·S3 목록은 항목 수·범위 상태·안전한 사유만 남기고 자원명·ARN·설명·태그·페이지 토큰을 결과로 적재하지 않는다. 정확한 단일 값 범위는 별도 승인·CURRENT_VALUE_ONLY·opt-in에서만 탐지하고 위치 참조는 aws-content-scope:<scope_id>다.
6. 최신 검증 시도·마지막 성공·미검증 이유를 분리한다. 존재·VALID·ACTIVE·사용 기록·침해 사실은 서로 추론하지 않는다. 평가는 정책·규칙·근거·시각을 보존하고 근거 부족을 LOW로 채우지 않는다.
7. 판정·조치·재검사·수명주기·해결은 버전·ACL·감사·멱등 결과와 원자 저장한다. 오탐은 Finding 교정, 정상 보관은 위치 판정, 키 무효는 검증 결과다.
8. WORKBENCH는 같은 헤더의 typed subject·metric membership을 대기열·분포로 투영한다. 독립 INVENTORY·CURRENT_CASES·HISTORY 헤더는 다른 scope로 재사용하지 않는다. CSV는 독립 INVENTORY 전체 페이지·고유 키 10,000개 상한을 따른다.

범위의 PENDING·RUNNING·SUCCEEDED·PARTIAL·FAILED·SKIPPED 및 과거/호환 UNKNOWN·CANCELLED를 배타적으로 scopeCount에 합산한다. finishedScopeCount=SUCCEEDED+PARTIAL+FAILED+SKIPPED+CANCELLED, successfulScopeCount=SUCCEEDED, gapScopeCount=PARTIAL+FAILED+SKIPPED+CANCELLED, unverifiedScopeCount=scopeCount-SUCCEEDED다. 과거 revision·단계 근거가 없으면 성공/누락 집계의 미확인 상태를 유지하고 화면에서 성공률을 추정하지 않는다. 현재 coverageGapScopes는 최근 불완전·새 revision 미검사·미실행·기한 초과·일시중지의 고유 범위 수다.

ScopeCoverageProjection은 실행 사실에서 파생한다. 물리 투영은 원본/투영 버전·시각을 갖고 재생 가능해야 한다. 캐시가 판정·해결의 원본은 아니다. 범위 비활성화는 예약 분모만 바꾸고 알려진 노출·과거 증거를 삭제하거나 해결을 통과시키지 않는다.

TaskGuidance·범위 설정 사유·필수 대응 위치 요약은 기존 사실의 읽기 투영이다. 화면 간 수동 이동을 위한 별도 상태 테이블을 만들지 않는다. 승인 방식·검토 담당은 기존 분류 제안·결정에, 내부 담당 요청은 조사 모델에 둔다. 현재 분류의 영향 처리는 해당 decision_id·위치·기존/새 건 연결과 추가 전용 처리 감사로 확인하며 과거 impact 행을 삭제하지 않는다.

## 7. 보안·보존·복구

API DB 역할은 허용 조회·업무 변경, 워커 역할은 작업 임대·탐지/검증·공급자 근거 적재로 분리한다. 일반 HTTP·DB 경로는 PROVIDER_VERIFIED를 만들 수 없다. 연결 비밀·HMAC 키는 DB 밖에 두고 버전을 관리한다. 보호키 교체의 재취득·동일성 재계산은 제한 접근으로 수행하며 재취득 불가 항목은 임의 병합하지 않는다.

원문은 영속 저장하지 않는다. AWS SDK가 불변 String을 만들 수 있어 프로세스 메모리 전체 소거를 보장한다고 주장하지 않는다. 메타데이터·감사·증거·백업은 범주별 보존 기간·접근·삭제·복구 범위를 운영 정책에 명시한다. 기간 미정은 자동 삭제나 무기한 보존의 기본값이 아니다. 만료 스냅샷 정리는 사실·감사 보존과 별개다.

복구 후 동일성·FK·현재 건 유일성·scope revision·최신 시도와 마지막 성공·증거 추가 전용성·권한 취소·집계 단위를 검증한다. 근거 부재는 null과 안전한 미확인 사유로 남기고 합성/실키·정상 보관·완전 검사·공급자 폐기를 소급 생성하지 않는다.

## 8. 검증 일반화와 평가 revision의 물리 제약

다음은 3절 요약을 구체화한 필수 컬럼·제약이다. 별도 큐·공급자별 원장을 추가하지 않고 기존 테이블을 확장한다. 저장된 적격 flag는 현재 권한·승인·동일성의 읽기 재검사를 대체하지 않는다.

| 원본 | 컬럼·타입 | 규칙 |
|---|---|---|
| `secret_jobs` | `lease_token UUID`, `lease_generation BIGINT NOT NULL DEFAULT 0` | 선점 때 token 교체·generation 증가. LEASED는 둘 다 유효하고 lease_owner·lease_until 필수. 결과/heartbeat는 jobId와 현재 token/generation·만료를 CAS 검사. 일반 API에 token 반환 금지 |
| `secret_credential_sets` | `credential_revision BIGINT NOT NULL DEFAULT 1` | 양수. 구성 승격·동일성 교정·보호키 회전의 의미 변경 시 증가. 단순 last_seen_at 갱신과 다른 기준 |
| `secret_occurrences` | `risk_evidence_revision BIGINT NOT NULL DEFAULT 0` | 0 이상. 평가 입력 사실의 확정 변경과 같은 트랜잭션에서 증가·재평가 예약. 입력 원본 head·정책 버전도 함께 확인 |
| `secret_connections` | `validation_enabled BOOLEAN NOT NULL DEFAULT false`, `validation_profile_version VARCHAR(80)`, `validation_revision BIGINT NOT NULL DEFAULT 1` | 서버의 승인 검증 profile 참조. 활성화에는 등록된 profile 필요. profile·활성화 변경은 revision과 감사를 갱신하고 범위 승인을 조용히 확대하지 않음 |
| `secret_connection_scopes` | `validation_approved BOOLEAN NOT NULL DEFAULT false`, `validation_approval_revision BIGINT NOT NULL DEFAULT 1`, `validation_profile_version VARCHAR(80)`, `validation_approved_by_user_id BIGINT FK`, `validation_approved_at TIMESTAMPTZ` | 양수 revision. 승인에는 현재 연결 profile 일치·수행자·시각 필요. 원천 수집 승인을 대체하지 않음. 철회/변경 때 revision 증가·안전한 감사 보존 |
| `secret_validation_attempts` | `scope_id BIGINT NOT NULL FK`, `occurrence_id BIGINT NOT NULL FK`, `source_finding_id BIGINT FK`, `credential_revision BIGINT NOT NULL`, `scope_revision BIGINT NOT NULL`, `validation_revision BIGINT NOT NULL`, `validation_approval_revision BIGINT NOT NULL`, `validation_profile_version VARCHAR(80)`, `action_context_digest BYTEA NOT NULL` | 현재 키·연결·범위·검증 승인과 조치 원본 ID/주기의 안전한 digest를 고정. 같은 키·실행·범위의 실제 부모를 저장 시 검사. source_finding_id는 사전 차단/재취득 실패에 NULL 허용, 실제 인증 관측에는 동일성을 확인한 유효 Finding 필요. profile 미설정의 사전 차단 COMPLETED/UNCHECKED만 profile=NULL 허용하며 실제 전송에는 등록 profile 필수 |
| `secret_validation_attempts` | `dedup_key BYTEA NOT NULL`, `cycle_key VARCHAR(120) NOT NULL`, `attempt_no INTEGER NOT NULL`, `reservation_generation BIGINT NOT NULL`, `reservation_token UUID NOT NULL`, `current_eligible BOOLEAN NOT NULL DEFAULT false`, `ineligible_reason_code VARCHAR(80)` | digest는 32 bytes, 번호·세대 양수. cycle은 승인된 6시간 구간/수동 요청 epoch/조치 주기를 구분한 내부 식별자. raw key나 경로를 넣지 않음. token은 내부 fencing용이며 읽기 API에 반환하지 않음 |

검증 profile은 버전 관리된 서버 설정의 검증기·요청·한도 manifest를 가리킨다. 연결·범위 JSON에 임의 호스트·메서드·헤더를 저장해 실행하지 않는다. 승인 이력은 기존 감사 원장의 대상 scope·이전/새 revision·profile·수행자로 보존한다. 현재 설정 행의 변경으로 과거 시도의 승인 근거가 사라지지 않게 한다.

검증 예약의 부분 고유 색인은 `(credential_set_id, validator_id, validator_version, dedup_key)`에 `attempt_state IN ('RESERVED','SENT')`를 조건으로 둔다. COMPLETED의 재사용 적격성과 새 수동 epoch는 [검증 계약](credential-validation.md)을 따른다. lease 복구자는 이전 예약을 CAS로 ABANDONED 처리한 뒤 새 세대를 확보하고 원문을 재취득한다. 외부 HTTP exactly-once는 보장하지 않으며 실제 시도와 예산을 보존한다.

필수 CHECK는 다음과 같다.

- RESERVED/SENT는 result·reason_code·completed_at이 NULL이다. RESERVED는 request_sent_at=NULL, SENT는 request_sent_at이 있다.
- COMPLETED는 result가 VALID·INVALID·UNKNOWN·UNCHECKED 중 하나이고 reason_code·completed_at이 있다. VALID/INVALID에는 request_sent_at·provider_checked_at이 필수, UNCHECKED에는 둘 다 없다. UNKNOWN의 실제 공급자 관측 시각은 NULL일 수 있다.
- ABANDONED는 result=NULL이며 종료 사유·completed_at이 있다. 진행 예약이나 포기 행을 최신 완료 결과로 선택하지 않는다.
- 모든 알려진 전송·공급자 관측·완료 시각은 requested_at 이후이며 완료 시각 이하이고 provider_checked_at은 request_sent_at 이후다. 로컬 전송 기록의 부재를 crash 시 미전송의 증명으로 해석하지 않는다.
- credential/scope/validation revision·reservation_generation·attempt_no는 양수, action_context_digest와 dedup_key는 32 bytes다. result/reason·발급자/유형/검증기/endpoint 조합은 고정 등록부와 일치해야 한다.
- profile 미설정은 `attempt_state=COMPLETED AND result='UNCHECKED' AND request_sent_at IS NULL AND provider_checked_at IS NULL`인 사전 차단에만 허용한다. 미등록 유형은 고정된 `validator_id=UNREGISTERED`, `validator_version=registry-v1`, `endpoint_class=NONE`의 비호출 분류자로 기록하며 등록된 실제 검증기가 있다고 가장하지 않는다. 가짜 profile·공급자 endpoint를 채우지 않는다.
- COMPLETED/current_eligible=false에는 안전한 부적격 사유가 필요하다. true여도 현재 ACL·동일성·승인 revision을 매번 재검사한다. 과거 이력의 사실을 수정해 재승인하지 않는다.

조회 색인은 `(credential_set_id,completed_at DESC,attempt_id DESC) WHERE attempt_state='COMPLETED'`와 `(job_id,attempt_state)`, 활성 예약 색인으로 구성한다. 마지막 인증 성공은 COMPLETED/VALID의 실제 provider_checked_at에서 구하고 권한/승인 적격성을 함께 검사한다. 위험 평가의 상세 CHECK·FK·색인은 [평가 계약](risk-assessment-contract.md)이 정의한다.

## 9. Notion 주체와 근거 저장

인증·주체·권한·수명 근거는 기존 공통 테이블을 확장한다. Notion 전용 시도 원장이나 최신 상태 캐시는 생성하지 않는다.

Notion 수집 연결의 `auth_ref`는 `notion-internal:default`이며 외부 토큰 파일의 경로와 원문을 저장하지 않는다. `secret_connections.source_retry_at`·`source_retry_reason`은 모두 NULL이거나 기한과 `NOTION_RATE_LIMITED|NOTION_PROVIDER_UNAVAILABLE` 사유를 함께 가진다. 동일 참조의 연결들에 검증된 가장 늦은 대기 기한을 적용하고 작업 재예약·정기 예약·실제 요청에서 확인한다. 발견 키의 인증 시도 원장과 수집 연결의 대기 상태는 별도로 보존한다.

`secret_run_scopes.excluded_object_count`는 0 이상의 정수이며 기본값은 0이다. 현재 페이지 순회에서 명시적으로 제외한 하위 페이지·데이터베이스·링크·첨부 본문 수를 기록하고 실제 수집 객체 수·검사 누락 수와 구별한다. 재시도 시 실행 범위의 개수·제외 개수를 초기화한 뒤 해당 시도의 관측을 저장한다. 지원 텍스트의 읽기 실패·불안정·미지원 블록은 단계 상태·안전한 실패 코드에 남기며 제외 개수로 대체하지 않는다.

Notion `source_object_ref`는 정규화한 페이지·블록 UUID의 제목 또는 블록 참조이며 `latest_object_version`은 페이지 수정 시각과 해당 객체 텍스트 해시를 가진다. 블록 간 AWS 구성요소를 결합하지 않는다. 원천 본문은 메모리에서 처리하고 위치·버전·HMAC 동일성·탐지 규칙과 안전한 결과만 기존 공통 테이블에 기록한다. 수정 시각의 분 단위 정밀도와 커서 순회의 스냅샷 한계를 고려해 `NOTION_PAGE_CONTENT`는 자동 부재 확정 대상에서 제외한다. 전체 검사·대응 위치 재검사에서 미관찰한 기존 위치는 `UNKNOWN`을 유지하며 `MISSING`·제거·회수 완료 근거를 생성하지 않는다. 실제 관찰한 동일 키의 인증 시도는 공통 원장에 별도로 남긴다. [수정 시각의 정밀도](https://developers.notion.com/guides/resources/historical-changelog#june-28-2021), [페이지 나눔 계약](https://developers.notion.com/reference/pagination)

| 기존 원본 | Notion 저장 계약 | 제약 |
|---|---|---|
| `secret_validation_attempts` | NOTION_GET_SELF·검증기 버전·NOTION_SELF, HTTP status·정규화 reason·전송/관측/완료 시각 | 네 결과·동일성·원천/승인 revision·현재 적격성은 공통 계약. 새 관측은 새 행 |
| `secret_credential_identities` | `provider=NOTION`, `principal_type=NOTION_BOT`, UUID인 `principal_id`, 보호된 `principal_ref`, 제한된 `safe_claims`, `observed_at`, `expires_at` | 같은 현재 VALID 부모 `validation_attempt_id`에 유일하게 결합. account_ref의 AWS 계정 의미를 보존하고 Notion에는 NULL 사용 |
| `secret_permission_requests/observations/decisions` | 등록된 Notion 권한 adapter가 있을 때 승인된 페이지·블록 읽기 또는 설정 자료의 출처·대상·safe_facts·완전성·관측 시각·근거 확인 기한 | 미등록이면 UNSUPPORTED이며 작업을 만들지 않음. 등록된 ENRICH/PERMISSION_ASSISTANCE의 PARTIAL 후보와 담당자 수용은 별도 이력 |
| `secret_credential_lifecycle_evidence`와 reviews | 외부 폐기 신고·수명 자료·수동 검토 | 수행 시각 미상을 허용하고 인증 거부와 별도 보존. 기존 확인 수준·독립 검토 적용 |

Notion identity의 닫힌 `safe_claims` schema는 `schemaVersion=1`, 선택적인 `ownerType=workspace|user`, 검증된 보호 `workspaceRef`만 허용한다. 알 수 없는 값·누락 필드는 미확인으로 표현하고 임의 문자열·추가 객체를 복사하지 않는다. 봇 주체 식별이 가능하면 선택 owner/workspace 필드가 없어도 identity를 생성할 수 있다. 지원하는 User 응답에서 주체 부가 자료를 구성할 수 없으면 인증 결과와 제한 사유를 보존한다. 이름·이메일·페이지 제목·전체 응답·원문 인증값은 이 schema에 포함하지 않는다. schema·safe_claims의 길이 상한은 8 KiB다.

identity는 부모의 실제 공급자 관측 시각을 참조하고 `expires_at`은 그 관측의 확인 기한으로 계산한다. 토큰 만료일은 발급/관리 출처가 정확히 해당 키에 결합된 수명 자료로 기록하며, 미확인 값을 기존 TTL·접수·폐기 신고 시각으로 채우지 않는다. 원문 키의 교체는 새 CredentialSet을 생성하고 기존 identity와 과거 검증을 보존한다.

구조화된 교체 근거에는 nullable `replacement_credential_set_id`, `before_validation_attempt_id`, `after_validation_attempt_id`, `replacement_validation_attempt_id` FK를 기존 lifecycle evidence에 연결한다. 참조한 전후 시도는 원래 키, 교체 키 시도는 새 키에 속해야 하며 각 원천·수행 시각·권한을 검사한다. 알려진 performed_at이 없는 근거는 전후 시각 조건을 미확인으로 유지한다. 참조 추가·교정은 기존 증거 version·멱등·감사와 원자 처리하고 원본 관측을 수정하지 않는다.

## 10. 근거 적용 시점

`secret_risk_evidence`는 nullable `fact_valid_from TIMESTAMPTZ`, `fact_valid_until TIMESTAMPTZ`, `source_presence_observed_at TIMESTAMPTZ`, `applicability_basis VARCHAR`를 갖는다. 기간은 둘 다 NULL이거나 from < until이어야 한다. 서버가 위치의 실제 원문 관측·원천 버전과 결합하며 `applicability_basis`는 OBSERVATION_ONLY 또는 EFFECTIVE_INTERVAL이다. 현재 수집기의 비동시 공개 관측은 OBSERVATION_ONLY와 NULL 사실 기간으로 기록한다. 콘텐츠 revision이나 같은 run만으로 ACL 동시성을 확정하지 않는다. 수동 기간은 해당 원문 관측과 평가 기준시점에 적용 가능해야 한다. 기존 행의 NULL은 미확인으로 보존하고 관측·접수·수용 시각을 추정하여 backfill하지 않는다.

현재 source revision·키·위치·부모와 맞지 않으면 적용 실패 사유를 반환한다. API의 applicableFrom/Until은 물리 fact_valid_from/until과 대응하며, 해당 근거의 관측 시각·TTL·만료는 별도로 검사한다. 원천 관찰 시간 결합은 키 원문·본문 저장을 요구하지 않는다.

## 11. 사용 이력과 AI 저장·실행

기존 키·위치·Finding, 인증 시도, 노출 판정, 보관 정책, 권한 후보·결정, 위험 근거·평가, 조사 과제·감사 원장을 재사용한다. AI 출력과 사용 기록을 secret_risk_evidence의 ABUSE_LINK로 저장하지 않는다. 기존 secret_location_activities는 담당자·조치 타임라인이며 키 사용 로그로 재사용하지 않는다.

새 논리 모델은 다음 여섯 가지다. SQL 버전 번호는 공동 DB 담당자와 구현 직전에 조율하며 과거 마이그레이션을 복제하지 않는다. 테이블마다 별도 마이그레이션을 만들지 않고 함께 배포하는 기능의 변경을 묶는다.

| 모델 | 저장 내용과 제약 |
|---|---|
| secret_usage_requests | 기존 ENRICH job_id, 서버 고정 범위·기간·키 결합·원천/승인 revision·멱등키, 실행 상태·페이지/이벤트 수·조회 커버리지·실패/누락 사유. 이벤트 0건과 불가 조회도 요약 행을 보존 |
| secret_usage_observations | sourceKind=PROVIDER 또는 MANUAL, nullable request_id·provider event 참조, 키/주체·적용 기간·결합 수준·행위 결과·안전한 자원 참조. 공급자 관측은 등록 adapter와 event/key 중복 키, 수동 자료는 actor/occurrence/idempotency로 중복 방지 |
| secret_usage_observation_decisions | observation_id·기준 revision·ACCEPT/REJECT/WITHDRAW·사유·typed 근거·담당자·시각. MANUAL 제출 수용과 철회를 추가 전용으로 보존 |
| secret_ai_analysis_requests | 독립 큐. 위치·요청자·평가·근거/인증/사용/ACL/전송 정책 revision·전체 contextDigest·로컬 alias 대응·상태·lease·generation·멱등키·검증 결과 코드 |
| secret_ai_analysis_attempts | 시도 ID·요청 ID·모델/prompt/schema·전송 시각·결과 사유·token 사용·예산 예약·응답 digest. 원문 응답 없음 |
| secret_ai_budget_windows | 배포·분/일 구간의 한도·예약·전송 횟수, 동시 실행 lease. 공급자 호출 전 원자 예약 |

사용 조회 요청은 기존 secret_jobs의 ENRICH 목적을 확장해 USAGE_OBSERVATION으로 구분한다. 범용 job에 존재하지 않는 context/result 필드를 가정하지 않고 조회 기간·대상·revision·완료/부분 실패·0건 요약은 secret_usage_requests에 보존한다. 요청과 job을 같은 트랜잭션에서 연결하고 기존 lease/fencing 계약으로 실행한다. 자동 관측의 event ID·고유 키와 수동 자료의 멱등키는 별도다. 스캔 횟수나 ScanRun을 만들지 않는다. AI 큐는 수집·권한·평가 큐와 별도로 처리하여 외부 AI 지연이 스캔을 막지 않는다.

기존 정기 예약·수집·권한 워커의 기본 polling 30초와 정기 스캔 6시간을 유지한다. 30초는 작업 확인 간격이며 시작·완료 보장이 아니다. 수동 요청은 다음 선점에서 처리하고 대기·실행 상태를 표시한다. 새 AI 큐만 기본 2초 polling을 사용하되 큐가 비었으면 10초까지 완화하며 수동 요청은 즉시 wake-up 신호를 줄 수 있다. wake-up은 성능 최적화이고 DB가 요청 원본이다.

외부 조회의 정상적인 제한·재시도는 기존 검증·수집 계약을 유지한다. 새 CloudTrail 조회는 한 페이지당 일시 오류 최대 2회 재시도, 1초·3초 지연과 jitter를 쓰고 Retry-After가 길면 lease를 반환해 재예약한다. 인증/권한 거절은 재시도하지 않는다. 실패한 페이지 이후를 COMPLETE로 처리하지 않는다. 원천 스캔·인증·사용 조회·AI의 성공 상태와 집계는 분리한다.

신규 안전한 사용 관측과 AI 요청/결과 메타데이터는 초기 365일 보존한다. AI 응답 원문은 보존하지 않으며 cache 적격 기간은 별도 24시간이다. 사고·미해결 대응 또는 평가의 근거로 참조된 기록은 365일을 지나도 참조가 해소되기 전 자동 삭제하지 않는다. 기존 평가·감사 원장은 현재 보존 계약을 유지한다. 정리는 운영 유지보수 권한으로만 수행하며 일반 API에는 삭제 기능을 추가하지 않는다.
