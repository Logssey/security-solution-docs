# 규칙 명세

**판정은 이 문서의 규칙으로만 내림.**

- 규칙마다 ID·엔진·조건·강도·심각도를 고정하고, 버전을 붙여 레포의 `policy/`에서 관리
- 문구는 규칙의 `template_key`로 만들어 PR 코멘트와 대시보드가 같은 문장을 씀

## 강도, 심각도, 데이터 등급

세 기준은 서로 따로 매김.

- **강도**: PR에서 무슨 일이 일어나는지 (병합 차단, 코멘트 등)
- **심각도**: 발견 건의 위험이 얼마나 큰지. Secret 등 다른 솔루션과 통합 대시보드에서 같은 기준으로 집계하기 위한 값
- **데이터 등급**: 경로 끝에 있는 자산이 얼마나 민감한지

| 강도 | 의미 | PR에서 |
|---|---|---|
| 차단 | 머지하면 안 되는 변경 | 필수 검사 실패, 병합 불가 |
| 경고 | 사람이 확인해야 하는 변경 | 검사 중립, 코멘트 |
| 정보 | 위험이 줄어든 변경 | 코멘트에 좋아진 점으로 표시 |
| 무시 | 판정에 영향 없음 | 표시 안 함 |

| 심각도 | 판단 기준 |
|---|---|
| CRITICAL | 그 자체로 자격 증명이 탈취되거나, 1등급 데이터에 닿음 |
| HIGH | 공급망 변조 신호, 또는 권한·접근이 크게 늘어남 |
| MEDIUM | 고정 누락·노출 증가처럼 다른 문제와 겹쳐야 위험해짐 |
| LOW | 영향이 거의 없음 |
| 없음 | 위험이 아닌 실행 오류나 개선. 위험 집계에서 제외 |

강도는 심각도에서 기계적으로 나오지 않음. 같은 HIGH라도 확인이 필요한 변경은 경고로 둠(예: `gate.workflow_changed`).

| 등급 | 대상 (현재 환경) |
|---|---|
| 1등급 | RDS PostgreSQL, Secrets Manager 시크릿 7개(DB 마스터 비밀번호 포함), tfstate 버킷 |
| 2등급 | 이미지 버킷, Loki 버킷(chunks·ruler) |
| 3등급 | 공개 정적 리소스 |

등급표는 대시보드 PostgreSQL의 `iac_data_assets` 테이블에서 관리 (ERD).

## ② plan 전 검사 규칙

| ID | 엔진 | 조건 | 강도 | 심각도 | 상태 |
|---|---|---|---|---|---|
| `gate.exec_data_source` | Conftest (hcl2) | `data "external"`·`"http"`·`"dns"` 존재 | 차단 | CRITICAL | 10/8 검증 |
| `gate.provisioner` | Conftest | `local-exec`·`remote-exec` provisioner | 차단 | CRITICAL | 10/8 검증 |
| `gate.lambda_invocation` | Conftest | `data "aws_lambda_invocation"` (plan 중 Lambda 호출) | 차단 | HIGH | 미구현 |
| `gate.provider_not_allowed` | Conftest | 허용 목록(`hashicorp/aws`) 밖 프로바이더 | 차단 | HIGH | 10/8은 namespace 단위로 검증. 프로바이더 단위로 수정 필요 |
| `gate.provider_unpinned` | Conftest | 프로바이더 `version` 없음 | 경고 | MEDIUM | 10/8 검증 |
| `gate.module_unpinned` | Conftest | git 모듈 `?ref=` 없음, 레지스트리 모듈 `version` 없음 | 경고 | MEDIUM | 10/8 검증 |
| `gate.backend_changed` | git diff | `backend` 블록 변경 | 차단 (plan 중단) | HIGH | 설계 |
| `gate.lock_changed` | git diff | `.terraform.lock.hcl` 변경 | 경고 + 보안 리뷰 필수 | MEDIUM | 설계 |
| `gate.workflow_changed` | git diff + CODEOWNERS | `.github/workflows/` 변경 | 경고 + 보안 리뷰 필수 | HIGH | 설계 |
| `gate.lock_mismatch` | `terraform init -lockfile=readonly` | 해시·버전이 lock 파일과 다름 | 차단 | HIGH | 설계 (공식 옵션 확인함) |
| `gate.module_content_changed` | 해시 비교 | 같은 버전인데 내려받은 모듈 내용이 다름 | 경고 | HIGH | 설계 (원격 모듈 도입 시) |
| `gate.plan_failed` | terraform plan | plan 실패·시간 초과 | 차단 (fail-closed) | 없음 (실행 오류) | 설계 |

- 판정 대상은 PR에서 새로 들어온 것만. 기존 코드에 있던 것은 허용 목록으로 관리.
- 화면 S-01 E의 관문 번호 대응:

| 관문 번호 | 규칙 |
|---|---|
| 0 | `workflow_changed` |
| 1 | `backend_changed`·`lock_changed` |
| 2 | `exec_data_source`·`provisioner`·`lambda_invocation`·`provider_*`·`module_unpinned` |
| 2b | `module_content_changed` (+ 내려받은 모듈에 2번 규칙 다시 적용) |
| 3 | `lock_mismatch` |
| 4 | `plan_failed` |

## 규칙 점검 (설정 오류)

| ID | 엔진 | 조건 | 강도 | 심각도 | 상태 |
|---|---|---|---|---|---|
| `rule.sg_open_world` | Conftest (plan JSON) | 인그레스 `0.0.0.0/0`·`::/0`, 포트 무관 | 경고 | MEDIUM | 설계 |
| `rule.iam_full_access` | Conftest (plan JSON) | `*FullAccess` 관리형 정책 연결, `Action: "*"` | 경고 | HIGH | 설계 |
| `rule.kics` | KICS | KICS 결과를 그대로 가져옴. KICS 쿼리 ID는 `iac_findings.params`에 | KICS CRITICAL·HIGH → 경고, 나머지 → 정보 | KICS 값 그대로 (INFO → LOW) | 설계 |

설정 오류만으로는 차단하지 않음. 차단은 ①에서 경로가 생길 때.

## ① 경로 규칙

| ID | 조건 | 강도 | 심각도 |
|---|---|---|---|
| `path.tier1.added` | 인터넷 → 1등급 데이터 `CAN_READ` 경로가 새로 생김 | 차단 | CRITICAL |
| `path.tier1.reach_only` | 1등급에 네트워크로만 도달(`CAN_REACH`), 자격 증명 경로 없음 | 경고 | HIGH |
| `path.tier2.added` | 인터넷 → 2등급 데이터 새 경로 | 경고 | HIGH |
| `path.shortened` | 기존 경로의 단계 수 감소 | 경고 | MEDIUM |
| `path.imds_v1_added` | `http_tokens = optional`인 인스턴스 추가 | 경고 | MEDIUM |
| `path.removed` | 기존 경로 사라짐 | 정보 | 없음 (개선) |

1등급 차단은 IAM 시뮬레이터로 확인한 뒤 확정. 시뮬레이터가 허용이 아니면 경고·HIGH로 낮춤.

## ③ State·드리프트 규칙

| ID | 조건 | 강도 | 심각도 |
|---|---|---|---|
| `state.reader_increase` | tfstate를 읽을 수 있는 주체 수 증가 (시뮬레이터 확인) | 경고 | HIGH |
| `state.secret_added` | 민감 표시이면서 값이 비어 있지 않은 속성이 새로 생김 | 경고 | MEDIUM |
| `state.plaintext_secret_arg` | 평문 `password` 인자, `random_password`·`tls_private_key`·`aws_iam_access_key` 추가 | 경고 + write-only·`manage_*` 제안 | MEDIUM |
| `drift.high` | 보안그룹·IAM·DNS·LB·네트워크 속성이 코드 밖에서 바뀜 | 경고 (높음) | HIGH |
| `drift.medium` | 인스턴스·DB 설정 변경 | 정보 (중간) | MEDIUM |
| `drift.low` | 태그·메타데이터 변경 | 정보 (낮음) | LOW |

- 강도는 위 4단계만 씀. 드리프트 높음·중간·낮음은 규칙 ID로 구분하고, 심각도는 HIGH·MEDIUM·LOW로 대응.
- 무시 목록 시작값 (모두 10/9 점검에서 확인한 오탐):
    - 드리프트: `aws_db_instance.latest_restorable_time`
    - State 검사: `aws_wafv2_web_acl.lock_token`, `aws_ssm_parameter.value`, `aws_instance.password_data`

## 운영 규칙

- 규칙 변경은 PR로만, 변경 시 `version` 올림
- 회귀 테스트: 위험 시나리오 S1~S6 탐지 + 실제 레포 오탐 0을 정책 PR마다 실행
- 예외: `rule_id` + 대상 + 사유 + 승인자 + 만료일. 만료되면 다시 판정에 포함
- 도구 버전 고정: Conftest는 10/8에 v0.56.0으로 검증함. 구축 때 최신 버전(10/12 기준 v0.71.1)으로 같은 테스트를 다시 돌리고 고정. KICS·Terraform도 같은 방식

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |
