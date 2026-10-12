# IaC 보안 설계문서

**Terraform 변경을 머지 전에 검사해서, 이 코드를 CI에서 돌려도 되는지(②)와 머지하면 민감 데이터까지 가는 공격 경로가 새로 생기는지(①)를 판정하고, 위험하면 차단.** State·드리프트 점검(③)은 두 기능에 데이터 등급과 현재 상태를 보태는 보조 기능. 결과는 PR과 통합 대시보드의 클라우드 보안 › IaC 보안 탭에 표시.

| 문서 | 담는 내용 |
|---|---|
| PRD | 목적, 해결할 문제, 범위, 기능·비기능·보안 요구사항, 사용 시나리오 |
| 아키텍처 | 구성 요소, 전체 구성도, PR·매일 데이터 흐름 |
| ADR | 아키텍처 결정 10건과 근거 |
| 규칙 명세 | 강도·데이터 등급, ②①③·설정 오류 규칙 전체, 운영 규칙 |
| 화면 설계 | IaC 보안 탭, PR 상세 패널, PR 코멘트 |
| API 명세 | CI용·대시보드용·내부 작업용 API |
| ERD | 결과 DB 테이블과 관계, 그래프 스키마 |


## 검토 결과 (10/12)

| 항목 | 확인 결과 | 반영 |
|---|---|---|
| 파드→EC2 연결 | Cartography Kubernetes 모듈에 `KubernetesPod -RUNS_ON-> KubernetesNode -IS_INSTANCE-> AWSEC2Instance`가 있음. 단, EC2 연결은 노드의 `providerID` 기준 | 직접 만들지 않고 Cartography 모듈 사용. Kubespray라 `providerID`가 비면 내부 IP 매칭만 보강 (ADR-03, ERD) |
| 실효 권한 | Cartography는 권한 매핑 파일로 `CAN_READ` 등을 계산하고 거부도 반영(조건부 거부는 무조건 거부로 처리). 10/10 수집에서 관계가 안 생긴 건 설정 문제일 가능성 | "Cartography가 못 한다" → "사용자 정의 매핑 파일로 계산 + 1등급은 시뮬레이터 확인"으로 수정 (ADR-04) |
| CloudFront·시크릿 노드 | `AWSCloudFrontDistribution`(S3 오리진만 연결), `AWSSecretsManagerSecret` 있음 | CloudFront→NLB만 보강, 시크릿 노드는 그대로 사용 (ERD) |
| plan JSON 민감값 | `terraform show -json`은 민감값을 평문으로 출력 | 제거 후 전송 설계 유지 (ADR-06) |
| lock 검증 | `init -lockfile=readonly`는 lock 파일을 바꾸지 않고 체크섬만 검증 | 설계 유지 |
| 도구 버전 | Conftest 최신 v0.68.x (10/8 검증은 v0.56.0) | 구축 때 재검증 후 고정 (규칙 명세) |
| 스캐너 자격 증명 | 비 EKS 클러스터의 IRSA 방식은 서비스 계정 발급자 문서(OIDC discovery·JWKS)를 공개 주소에 올려야 함 | ADR-07에 전제 조건과 대안 명시 |
| 전체 흐름 그림 (이전 문서) | 규칙 점검 칸이 Checkov·Trivy로 남아 ADR-09와 어긋남 | KICS·Conftest 보완으로 수정 |

**구축 초반에 확인할 것**

1. Cartography 권한 매핑이 돌지 않은 원인 확인, 사용자 정의 파일(`secretsmanager:GetSecretValue`, `s3:GetObject`)로 다시 수집
2. Kubespray 노드의 `providerID` 값 확인 (비면 내부 IP 매칭)
3. 서비스 계정 OIDC(ADR-07) 시범 적용, 어려우면 대안으로 전환

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |