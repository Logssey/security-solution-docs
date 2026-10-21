# ADR

## 결정 목록

| ID | 이슈 | 결정 | 상태 |
|---|---|---|---|
| [ADR-01](#adr-01) | 판정과 문구를 무엇으로 만드나 | 규칙과 템플릿. AI 미사용 | 결정 |
| [ADR-02](#adr-02) | ②를 언제, 무엇으로 검사하나 | plan 전에 `.tf`·lock 텍스트를 Conftest(직접 쓴 Rego)로 검사 | 결정 |
| [ADR-03](#adr-03) | 자산 그래프를 어떻게 만드나 | Cartography(AWS + Kubernetes 모듈) + Neo4j. 기본 모델에 없는 관계만 워커의 보강 단계에서 추가 | 결정 |
| [ADR-04](#adr-04) | 실효 권한을 어떻게 계산하나 | Cartography 권한 매핑(사용자 정의 매핑 파일)으로 1차 계산, 1등급은 IAM 시뮬레이터로 확정 | 결정 |
| [ADR-05](#adr-05) | 분석을 어디서 돌리나 | ②·plan은 GitHub 호스티드 러너, ①③ 분석은 클러스터 안 Java 워커 | 결정 |
| [ADR-06](#adr-06) | plan JSON의 민감값 처리 | 러너에서 민감 표시(`*_sensitive`)된 값을 지운 뒤 전송, 분석 후 삭제 | 결정 |
| [ADR-07](#adr-07) | 매일 작업의 AWS 자격 증명 | 전용 읽기 역할 scanner-ro를 k8s 서비스 계정 OIDC(IRSA 방식)로 받음 | 결정 (10/14 시범 검증 완료) |
| [ADR-08](#adr-08) | PR 봇의 신원 | GitHub App (이름, 최소 권한, 검사 결과 API) | 결정 |
| [ADR-09](#adr-09) | 규칙 도구 선택 | KICS 주력 + Conftest 보완 규칙. Checkov·Trivy는 선택. MVP는 Conftest만, KICS 연동은 다음 단계 | 결정 |
| [ADR-10](#adr-10) | 그래프·결과 저장소 | 그래프는 전용 Neo4j, 결과는 대시보드 PostgreSQL의 `iac_` 테이블 | 결정 |
| [ADR-11](#adr-11) | 어디에, 무슨 언어로 구현하나 | dashboard-backend(Spring Boot)에 `iac` 모듈로 통합, 분석은 별도 Java 워커 | 결정 |
| [ADR-12](#adr-12) | 사람의 AWS·State 권한 | 사람은 읽기 전용(plan-ro와 같은 정책), 로컬 plan은 래퍼로 허용, apply는 CI apply 역할만, 비상시 break-glass | 결정 (적용은 팀 일정, 로컬 래퍼는 10/15 적용) |

**상태 값**

- 결정: 확정, 그대로 구축
- 결정 (구축 초반 검증·시범 적용): 방향은 확정, 구축 초반에 실제로 확인한 뒤 확정하거나 대안으로 전환
- 결정 대기: 확정 필요
- 결정 (적용은 팀 일정): 방향은 확정, 팀 전체 작업 방식이 바뀌어 구축 스프린트와 별도 일정으로 적용

새 결정은 표에 한 줄, 아래에 같은 형식의 섹션 하나를 추가. 한 건이 길어지면 `adr/ADR-NN.md`로 분리하고 이 문서에는 표 한 줄과 링크만 남김.

## 결정별 상세

### ADR-01

**판정과 문구를 무엇으로 만드나** · 상태: 결정

- **결정**: 규칙과 템플릿. AI 미사용
- **근거**: 보안 판정은 같은 입력에 같은 결과가 나와야 감사·재현 가능. 외부 LLM에 인프라 구성을 보내는 것 자체가 유출 위험
- **버린 대안**: LLM 판정, LLM 요약 코멘트

### ADR-02

**②를 언제, 무엇으로 검사하나** · 상태: 결정

- **결정**: plan 전에 `.tf`·lock 텍스트를 Conftest(직접 쓴 Rego)로 검사
- **근거**: `init`·`plan`은 외부 코드를 내려받아 실행함 (Coder 2026.8). 10/8 검증: 위험 시나리오 6/6 탐지, 실제 레포 오탐 0/348
- **버린 대안**: plan JSON 기반 정책 검사 (이미 실행된 뒤라 늦음)

### ADR-03

**자산 그래프를 어떻게 만드나** · 상태: 결정

- **결정**: Cartography(AWS + Kubernetes 모듈) + Neo4j. 기본 모델에 없는 관계만 워커의 보강 단계에서 추가
- **근거**: 10/7 Cartography가 의도된 취약점 7개를 모두 관계 데이터로 수집. 공식 스키마에 `KubernetesPod -RUNS_ON-> KubernetesNode -IS_INSTANCE-> AWSEC2Instance`, `AWSLoadBalancerV2 -EXPOSE-> AWSEC2Instance`, `AWSCloudFrontDistribution`, `AWSSecretsManagerSecret`이 있음
- **보강 대상**: CloudFront→NLB(S3 오리진만 연결), IMDS 자격 증명, `providerID`가 없는 노드의 EC2 연결
- **버린 대안**: 상용 CNAPP(비용), Terraform 코드만으로 그래프 구성(코드 밖 자산 누락), 파드 연결을 kubectl로 직접 수집(중복 구현)

### ADR-04

**실효 권한을 어떻게 계산하나** · 상태: 결정 (10/13 검증 완료)

- **결정**: Cartography 권한 매핑(사용자 정의 매핑 파일)으로 1차 계산, 1등급은 IAM 시뮬레이터로 확정
- **근거**: 공식 문서상 권한 매핑은 매핑 파일에 적힌 동작만 계산하고, 거부는 반영(조건부 거부는 무조건 거부로 처리), 조건은 관계에 표시만 함. 조건·권한 경계·리소스 정책까지 반영한 정답은 시뮬레이터
- **검증 결과 (10/13)**: 10/10 T2·T3에서 S3 권한 관계가 하나도 안 생긴 원인을 확인
    - 원인: 기본 매핑 파일 경로가 상대경로(`cartography/data/...`)라, 공식 이미지의 작업 디렉터리에서 파일을 못 찾고 권한 단계를 건너뜀
    - 조치: 사용자 정의 매핑 파일(S3 `GetObject`·`PutObject`, `secretsmanager:GetSecretValue` → `CAN_READ`·`CAN_WRITE`)을 절대경로(`--permission-relationships-file`)로 넘겨 Cartography 0.143.0으로 재수집
    - 결과: 노드 역할 → 시크릿 7개, tfstate 읽기 주체 IAM 사용자 5명(노드 역할은 거부가 반영돼 제외). 10/9 IAM 시뮬레이터 결과와 일치
- **버린 대안**: 정책 평가기 직접 구현 (매핑이 안 될 때의 대체안으로 보류), 모든 쌍을 시뮬레이터로 (호출 수 과다)

### ADR-05

**분석을 어디서 돌리나** · 상태: 결정

- **결정**: ②·plan은 GitHub 호스티드 러너, ①③ 분석은 클러스터 안 Java 워커 (ADR-11)
- **근거**: PR 코드는 매번 버려지는 VM에서만 돌고, Neo4j는 사설망에 둠
- **버린 대안**: 클러스터 안 셀프 호스티드 러너 (악성 PR이 클러스터에서 실행됨)

### ADR-06

**plan JSON의 민감값 처리** · 상태: 결정

- **결정**: 러너에서 민감 표시(`*_sensitive`)된 값을 지운 뒤 전송, 분석 후 삭제
- **근거**: 공식 문서상 `terraform show -json`은 민감값을 평문으로 출력. 경로 계산에는 값이 필요 없음
- **지울 범위**: `resource_changes`의 `before`·`after` 중 `*_sensitive`로 표시된 값만으로는 부족함. sensitive 변수가 평문으로 남는 `variables`, `sensitive_values`로 표시된 `prior_state`·`planned_values`, sensitive output까지 같이 지움
- **지우는 방법 (10/15 구현, infrastructure `policy/scrub-plan.sh`)**: 두 단계로 지움
    1. 표시 기준: 위 범위에서 sensitive로 표시된 자리의 값을 `(sensitive)`로 바꿈. 키와 구조는 그대로 둠
    2. 복사본 제거: Terraform은 민감값이 복사된 자리(프로바이더가 돌려준 속성 등)에 표시를 붙이지 않을 때가 있음. 1단계에서 찾은 값을 모아 문서 전체에서 같은 문자열과, 8자 이상이면 그 값을 품은 문자열까지 지움
- **부작용**: 민감값과 우연히 같은 값도 가려짐 (실제 plan에서 SSM 파라미터로 받은 AMI ID가 가려짐). 경로 계산에는 값이 필요 없어 그대로 둠
- **받는 쪽 확인**: 백엔드는 업로드된 plan에서 sensitive 표시가 있는 자리에 `(sensitive)`가 아닌 값이 있으면 422(`UNPROCESSABLE`)로 거부
- **버린 대안**: 원본 전송·보관

### ADR-07

**매일 작업의 AWS 자격 증명** · 상태: 결정 (10/14 시범 검증 완료)

- **결정**: 전용 읽기 역할 scanner-ro를 k8s 서비스 계정 OIDC(IRSA 방식)로 받음
- **근거**: 10/10에 노드 역할이 모든 시크릿을 읽는 것을 문제로 찾음. 스캐너가 같은 역할을 쓰면 안 됨
- **버린 대안**: 노드 역할 재사용, 장기 액세스 키
- **전제 조건**: 비 EKS 클러스터에서 IRSA 방식을 쓰려면 아래가 필요. Kubespray 설정 변경이 필요하므로 구축 초반에 시범 적용
    1. 서비스 계정 발급자의 OIDC discovery 문서와 JWKS를 AWS가 읽을 수 있는 공개 주소(예: S3)에 올림
    2. API 서버의 issuer를 그 주소로 맞춤
    3. IAM OIDC 공급자를 등록
    4. 파드에 projected 토큰을 마운트하고 `AWS_ROLE_ARN`·`AWS_WEB_IDENTITY_TOKEN_FILE`을 넣음
- **어려울 때 대안**: 매일 작업을 GitHub Actions 스케줄 + 읽기 전용 OIDC 역할로 옮기고 결과는 API로 업로드(Neo4j는 클러스터 안 작업이 적재)
- **시범 결과 (10/14)**: 전제 조건 1~4 모두 적용, 대안 없이 본안으로 진행
    - 발급자 `https://k8s-oidc.re-used.store`: 비공개 S3 + CloudFront OAC로 discovery 문서·JWKS 공개 (`infrastructure`의 `modules/k8s-oidc`)
    - IAM OIDC 공급자 등록
    - API 서버 3대를 두 단계로 전환: 발급자 [기존, 새] → [새, 기존]. 기존 토큰도 계속 인정하고 `api-audiences`는 기존 값으로 고정해 운영 중단 없음
    - 시험 파드의 `aws sts get-caller-identity`가 노드 역할이 아니라 시험 역할로 나옴. 시험 역할은 삭제
    - 주의: Kubespray `upgrade-cluster.yml`은 kubeadm 1.35에서 apiserver `extraArgs` 변경을 반영하지 않음(kubeadm-config ConfigMap을 읽음). 설정 변경은 ConfigMap을 먼저 바꾸고 노드별로 적용

### ADR-08

**PR 봇의 신원** · 상태: 결정

- **결정**: GitHub App (이름, 최소 권한, 검사 결과 API)
- **근거**: 봇 이름과 권한을 따로 관리하고, 대시보드에서도 같은 신원으로 재검사·수정 PR 생성 가능
- **버린 대안**: Actions 기본 토큰 (이름이 github-actions, 워크플로 밖에서 못 씀)

### ADR-09

**규칙 도구 선택** · 상태: 결정 (10/13)

- **결정**: KICS 주력 + Conftest 보완 규칙. Checkov·Trivy는 선택
    - MVP는 Conftest만 씀(② 관문, 보완 규칙). KICS 연동은 설정 오류 목록(FR-R1)과 함께 다음 단계
    - IaC 기능 담당이 결정
- **근거**: 10/7 비교: KICS 5/7, Checkov·Trivy 1/7. 놓친 항목은 Conftest 몇 줄로 보완 가능. Trivy는 2026.3 공급망 사고로 버전 고정 필수
- **버린 대안**: 규칙 도구 3개 모두 사용 (중복 경고)

### ADR-10

**그래프·결과 저장소** · 상태: 결정

- **결정**: 그래프는 전용 Neo4j, 결과는 대시보드 PostgreSQL에 `iac_` 테이블로 추가 (Flyway 후속 마이그레이션)
- **근거**
    - 그래프 탐색은 Neo4j, 이력·상태 관리는 관계형 DB가 맞음
    - 대시보드 DB에 두면 담당자·감사 행위자를 `dashboard_users`, 자산을 `assets`에 FK로 연결할 수 있음
    - Secret도 같은 DB에 `secret_` 테이블로 추가함 (ADR-S008)
- **버린 대안**
    - 모두 Neo4j에 저장 (이력·상태 관리와 제약 걸기가 어려움)
    - 기존 RDS(Re:Used 서비스 DB)의 별도 DB `logssey_iac` (이전 초안). 서비스 DB를 장악한 공격자가 탐지 기록까지 지울 수 있고(ADR-D005), 대시보드 사용자·자산과 FK로 연결할 수 없음
- **남은 결정**: 운영 DB 서버 위치는 ADR-D005를 따름

### ADR-11

**어디에, 무슨 언어로 구현하나** · 상태: 결정

- **결정**
    - API: dashboard-backend(Java 21, Spring Boot)에 `iac` 모듈로 통합. 경로는 `/api/v1/iac`
    - 분석: API와 분리된 Java 워커가 `iac_jobs` 대기열에서 작업을 가져가 ①③을 계산. Neo4j Java 드라이버, AWS SDK for Java(IAM 시뮬레이터) 사용
    - 외부 도구: Cartography·KICS·Conftest·Terraform은 실행 파일로 두고 다시 구현하지 않음
- **근거**
    - 대시보드는 Spring Boot 앱 하나(dashboard-backend)이고, 로그인 세션·사용자·자산을 이미 가지고 있음
    - Secret이 같은 구조(Spring API + 별도 Java 워커 + 같은 PostgreSQL)를 이미 정함 (ADR-S008). 언어와 배포 단위를 맞추면 운영할 것이 줄어듦
    - 통합 현황의 심각도 집계를 한 API 안에서 합성할 수 있음
- **버린 대안**
    - 별도 IaC API 서비스 (이전 초안). 인증, 사용자, 통합 현황 합성을 서비스 간 호출로 따로 만들어야 함
    - Python 워커. Cartography와 같은 언어라 편하지만 ADR-S008의 "Python을 제품 실행 언어로 도입하지 않음"과 어긋남
- **영향**
    - CI 요청이 dashboard-backend로 들어옴. CI 경로만 CloudFront → 공개 NLB로 열고, OIDC 토큰 검증과 요청 크기·횟수 제한을 둠 (SR-9)
    - API와 워커는 DB 계정과 권한을 나눔 (ERD의 DB 계정 표)

### ADR-12

**사람의 AWS·State 권한** · 상태: 결정 (적용은 팀 일정)

- **결정**
    - 사람은 IAM Identity Center 읽기 전용 권한만 가짐(plan-ro와 같은 정책)
    - 로컬 plan은 래퍼(`make plan`)로 허용: ② 검사·2b → 홈 디렉터리 없는 컨테이너, 읽기 전용 임시 키만, `-lock=false`
        - 10/15 적용: SSO 전환 전까지는 IAM 사용자가 plan-ro 역할을 assume (`iac_plan_local_users`, 세션 이름 = 사용자 이름)
    - 남의 PR은 로컬에서 plan하지 않고 CI plan 결과로 리뷰
    - apply는 CI apply 역할만(GitHub OIDC, main + environment `prod`)
    - 비상시 break-glass 역할(MFA, 사용 즉시 알림)
    - IAM 사용자 장기 키 폐기, CloudTrail 켬
- **근거**
    - 10/9·10/13 점검: tfstate 읽기 주체가 IAM 사용자 5명(팀원 전원, 장기 키), CloudTrail 꺼짐
    - 사람에게 쓰기 권한이 있으면 로컬 apply로 ②①③을 건너뛸 수 있어, apply 경로가 CI 하나여야 검사가 의미가 있음
    - 로컬 plan을 막으면 확인할 때마다 CI를 거쳐야 해서 느림
    - State 속 실제 시크릿은 0건(10/9)이고 `state.secret_added`로 유지하므로 사람의 State 읽기는 허용
    - 읽기 전용 역할은 S3 `use_lockfile` 잠금을 못 쓰므로 로컬 plan은 `-lock=false`
- **버린 대안**
    - 로컬 plan도 금지하고 PR에서만 plan (Atlantis 방식. 피드백이 느리고 팀 규모에 과함)
    - 원격 실행 서비스 (HCP Terraform 등. 새 서비스 도입)
    - 현재처럼 합의만 (기술적으로 우회 가능)
- **한계**: 조직 오너·AWS 관리자는 보호 설정을 바꿀 수 있어 막지 못함. 인원 최소화·MFA·감사 로그로 줄이고, 이 솔루션은 드리프트·State 읽기 주체 증가로 탐지만 함
- **적용 순서와 체크리스트**: [위협 모델·범위](../01-requirements/threat-model.md)의 범위 밖 전제 조건 (apply CI 전환 → 사람 권한 전환 → tfstate 버킷 정책)

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |
| 수정일 | 2026-10-21 |
