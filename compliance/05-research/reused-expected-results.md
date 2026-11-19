# Re:Used 점검 예상 결과표 (Prowler 결과 검증용)

Re:Used 인프라를 Prowler로 점검했을 때 나와야 하는 결과를 코드 근거와 함께 정리했다. 컴플라이언스 기능의 첫 Fixture를 만들 때 이 표와 실제 결과를 대조한다.

- 작성일: 2026-11-05
- 근거 커밋: `infrastructure@ff58e13`(main), `service-backend@6583020`(develop), `dashboard-backend@b2d5ecd`(develop)
- 방법: Terraform, kubespray, K8s 매니페스트를 정적으로 읽어 기대값을 추정했다. Prowler를 실제로 실행한 결과는 아니다.

## 이 표를 쓰는 법

1. Prowler AWS·Kubernetes를 실행해 JSON-OCSF 결과를 받는다.
2. 아래 표의 Check별로 실제 결과와 예상 결과를 비교한다.
3. 결과가 다르면 둘 중 하나다.
   - **IaC와 실제 배포가 다르다.** 콘솔에서 수동으로 바꿨거나, 조직(Organization) 수준 설정이 있는 경우다. 원인을 기록한다.
   - **Check나 매핑이 기대와 다르게 동작한다.** 정규화 로직이나 Check 선택을 다시 본다.

Check ID는 후보다. 사용하는 Prowler 버전에서 `prowler aws --list-checks`, `prowler kubernetes --list-checks`로 실제 이름을 확인한 뒤 고친다.

## 전제: security_mode

prod의 `security_mode` 기본값은 `permissive`다(`terraform/environments/prod/variables.tf:107-114`). permissive는 스캔으로 찾을 취약 설정을 일부러 남기는 모드라서, 아래 일부 항목은 모드에 따라 결과가 바뀐다. 표의 "조건" 열에 표시했다.

IMDS 설정은 `security_mode`와 별개 변수다. strict로 바꿔도 IMDS 결과는 바뀌지 않는다(A1).

## AWS

| # | 점검 항목 | Prowler Check 후보 | 예상 결과 | 조건 | 근거 |
| --- | --- | --- | --- | --- | --- |
| A1 | EC2 IMDSv2 강제 | `ec2_instance_imdsv2_enabled` | FAIL | 모드 무관. prod가 `imds_tokens`를 넘기지 않아 모듈 기본값 `optional`이 적용됨 | `terraform/modules/compute/variables.tf:115-128`, `terraform/modules/compute/instances.tf:45` |
| A2 | SSH(22) 인터넷 공개 | `ec2_securitygroup_allow_ingress_from_internet_to_tcp_port_22` | permissive: FAIL / strict: PASS | `security_mode` | `terraform/modules/security/rules_permissive.tf:49-51` |
| A3 | DB·Redis·K8s API 포트 인터넷 공개 | 포트별 `ec2_securitygroup_allow_ingress_from_internet_to_*` | permissive: FAIL / strict: PASS | `security_mode` | `terraform/modules/security/rules_permissive.tf:27-113` |
| A4 | CloudTrail 다중 리전 | `cloudtrail_multi_region_enabled` | FAIL | 조직 수준 Trail이 있으면 PASS | Terraform에 `aws_cloudtrail` 없음 |
| A5 | AWS Config 기록 | `config_recorder_all_regions_enabled` | FAIL | 콘솔에서 켰으면 PASS | Terraform에 `aws_config_*` 없음 |
| A6 | VPC Flow Logs | `vpc_flow_logs_enabled` | FAIL | — | Terraform에 `aws_flow_log` 없음 |
| A7 | EBS 볼륨 암호화 | `ec2_ebs_volume_encryption` | PASS | — | `terraform/modules/compute/instances.tf:36,81,126,167` |
| A8 | EBS 기본 암호화(계정 설정) | `ec2_ebs_default_encryption` | FAIL | 콘솔에서 켰으면 PASS. 볼륨은 개별로 암호화돼 있어 A7과 결과가 다를 수 있음 | Terraform에 `aws_ebs_encryption_by_default` 없음 |
| A9 | RDS 저장 암호화 | `rds_instance_storage_encrypted` | PASS | — | `terraform/modules/rds/main.tf:84` |
| A10 | RDS 공개 접근 차단 | `rds_instance_no_public_access` | PASS | — | `terraform/modules/rds/main.tf:100` |
| A11 | RDS Multi-AZ | `rds_instance_multi_az` | FAIL | 비용 때문에 의도적으로 꺼 둠 | `terraform/environments/prod/variables.tf:189-192`, `terraform/modules/rds/main.tf:105` |
| A12 | RDS 자동 백업 | `rds_instance_backup_enabled` | PASS(7일) | — | `terraform/modules/rds/main.tf:111`, `terraform/modules/rds/variables.tf:105-106` |
| A13 | S3 버킷 퍼블릭 액세스 차단 | `s3_bucket_level_public_access_block` | PASS | — | `terraform/modules/s3/main.tf:38-70` |
| A14 | S3 버저닝 | `s3_bucket_object_versioning` | FAIL | 의도적으로 끔 | `terraform/modules/s3/main.tf:72-84` |
| A15 | IAM 계정 암호 정책 | `iam_password_policy_*` | 판단 불가 | 계정 설정이라 코드 밖에 있음 | Terraform에 `aws_iam_account_password_policy` 없음 |

## Kubernetes

| # | 점검 항목 | Prowler Check 후보 | 예상 결과 | 조건 | 근거 |
| --- | --- | --- | --- | --- | --- |
| K1 | API 서버 감사로그 | `apiserver_audit_log_path_set` | FAIL | — | `kubespray/group_vars`에 감사로그 설정 없음 |
| K2 | etcd 안 Secret 암호화 | `apiserver_encryption_provider_config_set` | FAIL | — | `kubespray/group_vars`에 `kube_encrypt_secret_data` 없음 |
| K3 | NetworkPolicy 존재 | 확인 필요(Prowler에 해당 Check가 없을 수 있음) | FAIL | 이 레포 기준. 앱 매니페스트가 다른 곳에 있으면 다시 확인 | `k8s/`에 `NetworkPolicy`·`CiliumNetworkPolicy` 없음 |
| K4 | 앱 Pod non-root, capability drop, SA 토큰 automount | `core_minimize_root_containers_admission` 등 | 코드로 확인 불가 | Re:Used 앱 매니페스트가 이 레포에 없음. 위치 확인 필요 | `k8s/`에는 플랫폼 구성요소만 있음 |
| K5 | ArgoCD 서버 TLS | 확인 필요(Prowler 범위 밖일 수 있음) | FAIL(수동 확인) | — | `k8s/platform/argocd/values.yaml:20` `server.insecure: true` |

## Prowler 범위 밖: 수동 검토 후보

아래는 설정 점검으로 잡히지 않는다. 기능 설계의 NEEDS_REVIEW 항목 예시로 쓴다.

| 항목 | 현재 상태 | 근거 |
| --- | --- | --- |
| 노드 Role의 Secrets Manager 읽기 범위 | 계정의 모든 시크릿(`secret:*`)을 읽을 수 있음. Prowler IAM Check가 잡는지 확인 필요 | `terraform/modules/iam/main.tf:93-106` |
| PR 이벤트의 ECR push 권한 | `pull_request` 실행에도 push Role을 허용 | `terraform/modules/iam/github-actions.tf:62` |
| Re:Used 로그인 실패 제한 | 5회/10분 차단 있음 | `service-backend:src/main/resources/application.properties:31-32` |
| 대시보드 로그인 실패 제한·MFA | 없음 | `dashboard-backend:src/main/java/com/logssey/auth/AuthController.java` |
| 접속기록(로그인·IP) | 대시보드는 `last_login_at`만 기록 | `dashboard-backend:src/main/java/com/logssey/auth/AuthService.java` |
| 개인정보 처리방침, 국외 이전 고지 | 없음 | `service-frontend` 라우트에 처리방침 페이지 없음 |

---

**추신.** 이 표와 별도로 Re:Used와 대시보드를 ISMS-P 기준으로 자체 점검했다. Logssey GitHub 8개 레포를 정적으로 읽어 인증기준별 충족 여부를 판정했다. 그 결과를 바탕으로 관리체계 문서 초안도 작성해 두었다. 초안에는 정보보호 정책서, 조직·책임자 지정, 정보자산 목록, 위험평가서, 침해사고 대응 절차서, 재해복구(DR) 계획, 개인정보 흐름도, 법적 요구사항 목록이 들어 있다. 컴플라이언스 기능의 수동 검토 항목을 설계할 때 참고할 수 있으며, 필요하면 작성자에게 요청하면 공유한다.
