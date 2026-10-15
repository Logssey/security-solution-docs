# 위협 모델과 범위

**이 솔루션은 인프라 쓰기 경로를 "PR → 검사 → CI apply" 하나로 만들고, 그 길에서 위험한 코드와 변경을 막고, 길 밖 변경은 탐지함.**

- 관리자·조직 오너 권한의 탈취·악용은 신원·권한 관리 영역이라 막지 않고 탐지만 지원
- 이 문서는 왜 그렇게 정했는지(실제 사고, 우리 환경), 무엇을 막고 무엇을 못 막는지, 범위 밖에서 해야 할 일을 정리

## 배경: 실제 사고 2건

**둘 다 내부자가 직접 빼간 게 아니라, 외부 공격자가 배포 경로를 오염시키고 피해자의 자동화(CI·provisioner)가 평소처럼 실행하면서 털린 사건.** 공통점은 "받은 코드를 검사 없이 실행" + "실행 환경에 강한 자격 증명".

| 항목 | Coder 레지스트리 (2026-08-31) | Trivy GitHub Action (2026-03-19~20) |
|---|---|---|
| 침입 | Coder의 Cloudflare 인프라 침입(방법 비공개). `registry.coder.com` 트래픽을 받는 IP 목록에 공격자 서버 추가 | Trivy 레포의 취약한 `pull_request_target` 워크플로로 조직 범위 PAT 탈취(2/28), 자격 증명 교체가 불완전해 다시 악용 |
| 오염된 것 | 같은 버전 번호로 내용만 바꾼 Terraform 모듈. `data "external" "telemetry"`가 스크립트 실행 | trivy-action 태그 76개 중 75개를 악성 커밋으로 강제 변경, setup-trivy 태그 7개, Trivy v0.69.4 바이너리 |
| 노출 시간 | 07:35~21:45 UTC, 약 14시간 | 약 12시간 |
| 실행 시점 | 피해 고객사의 provisioner가 템플릿·워크스페이스를 만들며 Terraform 실행. data 소스는 plan에서 읽히므로 자동 실행 | 태그로 Action을 쓰던 CI가 스캔 단계에서 실행. 정상 스캔 전에 악성 코드가 먼저 돎 |
| 털린 것 | 클라우드·AI 도구 키, CI/CD 자격 증명, OIDC 토큰, SSH 키, 환경변수, 설정 파일, 터미널 기록, 일부 구성에서 Coder DB 비밀번호 | 러너 프로세스 메모리의 시크릿, 환경변수, 클라우드 자격 증명, kubeconfig, `terraform.tfstate`·`tfvars`, SSH 키 등 |
| 보낸 곳 | 비슷한 도메인 `coder-infra[.]com`(3일 전 등록) | 비슷한 도메인 `scan[.]aquasecurtiy[.]org`, 실패 시 피해자 GitHub에 공개 레포를 만들어 업로드 |
| 안 털린 경우 | (해당 시간에 모듈을 내려받지 않은 곳) | Action을 커밋 SHA로 고정한 파이프라인 |

**근본 원인 (이 설계가 겨냥하는 것)**

1. 배포 경로를 그대로 믿음. 버전 고정은 "게시자가 악성 버전을 올리는 것"만 막고 "같은 버전, 다른 내용"은 못 막음. Terraform lock 파일은 프로바이더 해시만 기록하고 모듈 내용은 검증 안 함
2. 받은 코드를 검사 없이 실행. Terraform은 plan에서도 코드를 실행함(`data "external"` 등)
3. 실행 환경에 만료 없는 키와 넓은 권한이 있음
4. `pull_request_target`, 조직 범위 PAT, 태그 참조 같은 CI 설정 약점
5. 나가는 통신 제한 없음 (이 설계 범위 밖)

## 우리 환경에서 확인한 노출 (10/6~10/13 점검)

**지금은 사고 원인 1~4가 모두 우리에게도 있음.** 원격 모듈만 아직 안 써서 모듈 오염 경로가 없을 뿐.

| 항목 | 확인 결과 | 의미 |
|---|---|---|
| 사람 자격 증명 | 팀원 5명이 IAM 사용자 장기 키 보유, 5명 모두 tfstate 읽기 가능(10/9, 10/13 그래프로 재확인) | 로컬 apply로 검사 우회 가능. 키가 털리면 만료 없이 악용 |
| 변경 기록 | CloudTrail 꺼짐. grafana DNS 드리프트를 찾았지만 누가 바꿨는지 추적 불가(10/9) | 우회 변경을 찾아도 행위자를 모름 |
| 노드 역할 | 노드 역할 → 시크릿 7개 읽기 가능(시뮬레이터·Cartography 일치), IMDS `optional`·hop 2 | 파드 하나 장악 시 DB 비밀번호까지. ①의 대표 경로 |
| tfstate 보호 | 노드 역할의 거부 정책(`deny_tfstate`) 하나뿐 | 이 정책을 지우는 PR이 위험 (시나리오 U2, #58) |
| State 속 시크릿 | 실제 시크릿 0건(10/9) | 사람의 State 읽기를 허용할 수 있는 근거. ③이 계속 감시 |
| plan 실행 | 로컬에서 검사 없이 plan, CI에는 인프라 plan 검사 없음 | 어느 쪽에서든 실행 구문이 그대로 돎 |
| 모듈 출처 | 전부 레포 안 로컬 모듈 | Coder형 오염은 아직 해당 없음. 원격 모듈 도입 시 2b 필수 |

## 목표와 강제 지점

**강제는 CI에서, 로컬은 실수 방지와 피해 한정, 나머지는 탐지.**

- 남의 코드와 권한 있는 자격 증명이 만나는 곳이 CI라 여기를 우회 불가로 만듦
- 로컬은 강제할 수 없지만, 래퍼를 빼고 돌리면 피해는 본인의 읽기 전용 임시 키뿐이라 공격자 입장에서 얻는 게 거의 없음

| 지점 | 방식 | 강제 여부 | 막는 것 |
|---|---|---|---|
| CI (PR) | ② 관문 0~4(2b 포함) → plan(plan-ro) → ①③. ②·①은 필수 검사 | 강제 (브랜치 보호, CODEOWNERS) | 오염된 모듈·악성 PR 코드의 실행, 위험한 경로를 만드는 변경의 병합 |
| CI (apply) | main 병합 후 apply 역할(GitHub OIDC, environment `prod` 승인) | 강제 (ADR-12 적용 후) | 검사를 안 거친 변경의 적용 |
| 로컬 plan | 래퍼: ② 검사(2b 포함) → 홈 디렉터리 없는 컨테이너에서 읽기 전용 임시 키로 plan | 강제 안 함 (기본 경로로 만듦) | 정직한 팀원이 모르고 오염된 코드를 실행하는 것, 실행돼도 다른 키·토큰이 털리는 것 |
| 매일 점검 | ③ 드리프트, State 읽기 주체, State 속 민감 값. ① 열린 경로 재계산 | 해당 없음 (탐지) | 콘솔·로컬 apply 같은 길 밖 변경, 권한 증가를 다음 날 발견 |

## 위협별 대응

**예방이 실패해도 피해 한정과 탐지가 뒤를 받치도록 위협마다 세 층을 둠.** 피해 한정 칸 중 ADR-12가 필요한 항목은 사람 권한 정리 전까지는 미적용.

| ID | 위협 | 예방 (이 솔루션) | 피해 한정 | 탐지 | 남는 한계 |
|---|---|---|---|---|---|
| TM-1 | 오염된 원격 모듈 (Coder형) | 2b: init 후 내려받은 모듈에 Conftest, 실행 구문이면 plan 전 차단. 같은 버전·다른 내용이면 경고 + 보안 리뷰 | ② 통과 전 자격 증명 없음, plan 역할 읽기 전용·시크릿 거부, 로컬은 컨테이너 | PR 코멘트·대시보드 ② 영역 | 규칙에 없는 새 실행 수법 |
| TM-2 | 악성 PR 코드 (외부 기여자, 탈취된 팀원 GitHub 계정) | ② 관문 2: `data "external"`·`http`·`dns`, provisioner, Lambda 호출, 허용 안 된 프로바이더 차단 | 같음. 포크 PR에는 자격 증명 미제공 | ② 결과, 차단 이력 | 새 실행 수법 |
| TM-3 | CI 공급망 (Trivy형: Action·도구 변조) | Action·이미지 커밋 SHA·버전 고정, `pull_request_target` 금지, 워크플로 변경은 보안 리뷰 필수, lock 파일 해시 검증 | 러너 자격 증명 최소화(레포 범위 토큰, 읽기 전용 plan 역할) | `gate.workflow_changed`, `gate.lock_changed` | 고정한 버전 자체가 이미 오염된 경우 |
| TM-4 | 위험한 인프라 변경 (실수나 의도로 경로를 여는 PR) | ① 1등급 데이터까지 새 경로면 병합 차단(시뮬레이터 확인) | 리뷰 필수 | 대시보드 열린 경로 | 그래프에 없는 관계로 생기는 경로, 그래프 최신성(24시간) |
| TM-5 | 길 밖 변경 (콘솔 수정, 로컬 apply) | 사람에게 쓰기 권한 없음 (ADR-12, 범위 밖) | break-glass는 MFA·사용 즉시 알림 (ADR-12) | ③ 매일 드리프트 | 다음 날까지 모름, CloudTrail 없으면 행위자 모름 |
| TM-6 | State 노출 (읽기 주체 증가, 평문 시크릿 추가) | PR에서 `state.reader_increase`·`state.secret_added` 경고, write-only·`manage_*` 제안 | State에 실제 시크릿을 두지 않음 | ③ 매일 State 점검 | 이미 읽은 사람은 되돌릴 수 없음 |
| TM-7 | 일반 팀원 자격 증명 탈취 (PC 감염, 키 유출) | 해당 없음 (범위 밖) | SSO 짧은 세션·읽기 전용이면 피해가 "몇 시간 조회"로 제한 (ADR-12) | ③ State 읽기 주체·드리프트 | 현재는 장기 키라 피해 큼 |
| TM-8 | 관리자·조직 오너 권한 탈취·악용 | 해당 없음 (범위 밖) | 인원 최소화, 피싱에 강한 MFA, 평소 미사용 (범위 밖) | ③으로 결과만 일부 탐지 | 보호 설정 자체를 끌 수 있어 막지 못함. Secret도 같은 전제 |

## 로컬 plan 방식

**로컬 plan은 허용하되 `make plan` 래퍼를 기본 경로로 둠.** 로컬 plan을 막으면 확인할 때마다 CI를 거쳐야 해서 느리고, 실무에서도 plan은 허용하고 apply만 막는 경우가 많음(ADR-12).

래퍼가 하는 일 (`infrastructure`의 `Makefile`, 10/15 실제 plan까지 확인):

1. ② 관문 2: Conftest(CI와 같은 `policy/`와 같은 버전). 관문 0·1(변경 감지)은 CI에서만
2. 관문 3·2b: `init -backend=false -lockfile=readonly`로 모듈·프로바이더만 받고, 내려받은 모듈에 Conftest + 모듈 해시 비교. 결과는 `.terraform-gate`에 둠
3. 내 IAM 사용자로 plan-ro 역할을 assume해 1시간짜리 읽기 전용 임시 키를 받음 (세션 이름 = 사용자 이름). ADR-12로 SSO에 옮기면 SSO 세션에서 꺼냄
4. 이미지 digest로 고정한 Terraform 컨테이너에서 `init -lockfile=readonly` → `plan -lock=false`. 2번에서 검사한 모듈·프로바이더를 그대로 씀

| 컨테이너에 넣는 것 | 넣지 않는 것 |
|---|---|
| 레포의 `terraform/`(읽기 전용, 검사한 모듈·프로바이더 포함), 읽기 전용 임시 키(최대 1시간) | 홈 디렉터리 전체: `~/.aws`(장기 키, SSO 캐시), `~/.ssh`, GitHub 토큰, Docker 자격 증명, 셸 기록 |

- **컨테이너 제한**: 루트 파일 시스템 읽기 전용, capability 모두 제거, 권한 상승 금지. backend 초기화는 컨테이너 안 임시 디렉터리에서 해서 사람이 쓰는 `.terraform`은 건드리지 않음
- **backend는 분리하지 않음**: 처음엔 State 버킷 정보를 코드에서 빼고 래퍼만 `-backend-config`로 넘기려 했음. 모두가 init을 다시 해야 하는 데 비해 얻는 게 습관 방지뿐이라 뺌. backend 블록 변경은 관문 1이 차단
- **남의 PR**: 로컬에서 plan하지 않고 CI plan 결과로 리뷰
- **래퍼를 빼고 돌리면**: 지금은 PC의 장기 키(쓰기 권한)로 plan이 돌아 실행 코드가 그 키를 가져갈 수 있음. ADR-12 적용 뒤에도 읽기 전용 키와 PC의 다른 비밀이 노출될 수 있음. 피해자는 본인이라 래퍼를 쓸 이유가 분명함
- **편의성**: 명령은 `make plan` 하나. 도구는 첫 실행 때 `.tools/bin`에 받고, 프로바이더는 gate가 받은 것을 재사용해 두 번째 실행부터 속도 차이 거의 없음

## 한계

**알려진 실행 수법과 그래프에 있는 관계까지만 막음.** 아래는 일부러 범위에서 뺀 것과 구조적으로 못 막는 것.

- **새 실행 수법**: ②는 알려진 구문(`external`·`http`·`dns`, provisioner, Lambda 호출, 허용 안 된 프로바이더)을 찾음. 처음 보는 수법은 실행될 수 있고, 그때는 피해 한정(읽기 전용 임시 키)에 기댐
- **나가는 통신 제한 없음**: plan은 AWS와 통신해야 해서 러너·컨테이너의 네트워크를 열어 둠. 임시 키를 밖으로 보내는 것 자체는 못 막음(범위 밖)
- **로컬 강제 불가**: 래퍼는 기본 경로일 뿐 강제가 아님
- **관리자·조직 오너**: 브랜치 보호·IAM·이 솔루션 자체를 끌 수 있음. 어떤 도구도 그 도구를 끌 수 있는 사람은 못 막음
- **사람 권한 정리 전**: ADR-12 적용 전에는 쓰기 키를 가진 팀원이 로컬 apply로 우회 가능. ③이 다음 날 탐지만 함
- **①의 범위**: 그래프에 있는 관계(Cartography + 보강 4개)와 MVP 경로 유형만 봄. 조건·권한 경계·리소스 정책은 1등급만 시뮬레이터로 확인. 그래프는 최대 24시간 전 상태
- **런타임**: 배포 후 파드 안에서 일어나는 공격은 런타임 보안 영역(PRD 제외 범위)

## 범위 밖 전제 조건 (자격 증명·권한)

**이 솔루션이 만들지 않지만, 없으면 위 표의 "피해 한정"이 성립하지 않는 것들.**

- 팀 전체 작업 방식이 바뀌는 일이라 구축 스프린트와 별도로 팀 일정을 잡음(ADR-12)
- 1→2→3 순서를 지킴: apply를 CI로 옮긴 뒤에 사람의 쓰기 권한을 회수

| # | 항목 | 내용 | 관련 위협 | 상태 |
|---|---|---|---|---|
| 1 | apply를 CI로 | apply 역할 신뢰 조건을 GitHub OIDC + main + environment `prod`(승인자 지정)로 제한. apply 워크플로 구축 | TM-5 | 예정 |
| 2 | 사람 권한 전환 | IAM 사용자 장기 키 폐기, IAM Identity Center 읽기 전용 권한(plan-ro와 같은 정책, 세션 1시간) | TM-5, TM-7 | 예정 |
| 3 | tfstate 버킷 정책 | 쓰기는 apply 역할만, 읽기는 apply·plan-ro·scanner-ro·사람 읽기 권한만 | TM-6 | 예정 |
| 4 | break-glass 역할 | 장애 때만 쓰는 쓰기 역할. MFA 필수, 짧은 세션, 사용 즉시 알림 | TM-5, TM-8 | 예정 |
| 5 | CloudTrail | 관리 이벤트 첫 복사본(무료) 켜고 일정 기간 보관. 드리프트의 행위자 추적 | TM-5, TM-8 | 예정 |
| 6 | GitHub 조직 | 오너 최소 인원, 2FA 필수, 브랜치 보호·룰셋에 관리자도 포함, 감사 로그 확인 | TM-2, TM-8 | 예정 |
| 7 | CI 토큰 | 조직 범위 PAT 대신 GitHub App·레포 범위 토큰, 워크플로 권한은 작업별 최소 | TM-3 | 예정 |
| 8 | 운영 관리자 권한 | AWS 관리자 권한 보유자 최소화, 피싱에 강한 MFA(보안 키), 루트 계정 미사용 | TM-8 | 예정 |

노드 역할의 시크릿 읽기 축소·IMDSv2 강제는 전제 조건이 아니라 ①이 찾은 조치 대상이라 여기 없음(제안 조치 목록에 표시).

## 출처

- [CSA 연구 노트: Coder Registry Terraform 공급망 (2026-09-07)](https://labs.cloudsecurityalliance.org/wp-content/uploads/2026/09/CSA_research_note_coder_registry_terraform_supply_chain_20260907-csa-styled.pdf)
- [eSecurity Planet: Coder Registry Compromise](https://esecurityplanet.com/cybersecurity/news-coder-registry-malicious-terraform-modules)
- [Wiz Threats: Coder module registry compromise](https://threats.wiz.io/all-incidents/coder-module-registry-compromise-leads-to-credential-stealing-malware-distribution)
- [Snyk: Trivy GitHub Actions supply chain compromise](https://snyk.io/articles/trivy-github-actions-supply-chain-compromise/)
- 우리 환경: 이전 문서의 10/6~10/10 점검 기록, 구축 스프린트의 10/13 사전 확인

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-13 |
| 수정일 | 2026-10-15 |
