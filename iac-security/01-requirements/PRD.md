# PRD

## 제품 목적과 해결하려는 문제

**목적: Terraform 변경이 머지되기 전에, 그 변경이 실제 위험을 늘리는지 판정하고 막는 것.**

- **배경**: 인프라를 로컬 작업에서 CI/CD와 PR 협업으로 바꾸면, 레포에 코드를 올릴 수 있는 것만으로 클라우드 권한을 쓰게 됨
- **방법**: 이 길에 관문을 두고 머지 전에 검사
- **결과 위치**: 대시보드의 클라우드 보안 › IaC 보안 탭

| 문제 | 근거 (우리 환경 점검) | 해결 기능 |
|---|---|---|
| `plan`이 코드를 실행해서, 오염된 모듈이나 위험 코드가 CI 자격 증명을 빼갈 수 있음 | 2026년 8월 Coder 레지스트리 사건. 변조된 모듈의 `data "external"`이 클라우드 키와 OIDC 토큰을 훔침 | plan 전 검사 |
| 규칙 도구는 설정 하나만 봐서, 이어졌을 때의 위험과 우선순위를 모름 | 의도된 취약점 7개 중 Checkov·Trivy 1개, KICS 5개만 탐지. 파드 → IMDSv1 → 노드 역할 → 모든 시크릿 → DB 비밀번호 경로는 어느 규칙 도구도 못 잡음 | PR 경로 미리보기 |
| 콘솔 변경과 State 노출을 모름 | grafana DNS 드리프트를 찾았지만 CloudTrail이 꺼져 있어 원인 추적 불가. tfstate 보호는 노드 역할의 거부 정책 하나뿐 | State·드리프트 점검 (보조) |

**사용자**

- 인프라 담당: 코드 작성·조치
- 보안 담당: 차단 리뷰·예외 승인
- 팀원: PR 작성자, 코멘트로 결과 확인

## 포함·제외 범위

| 포함 | 제외 (이유) |
|---|---|
| AWS Terraform 레포의 PR 검사 (plan 전 검사 → plan → 경로 분석 → State) | 런타임 탐지 (Falco 등, 런타임 보안 섹션) |
| 자산 그래프 기반 경로 분석 (인터넷 → 1·2등급 데이터) | 시크릿 탐지·교체 (메인 시크릿 솔루션) |
| 규칙 점검 결과 표시 (KICS + Conftest 보완 규칙) | 다른 클라우드, k8s 매니페스트·Helm 검사 |
| 매일 드리프트·State 점검 | 자동 apply (사람이 승인해야 함) |
| GitHub 연동: 검사 결과, 코멘트, 병합 차단, 수정 PR 생성 | AI 기반 판정·문구 생성 (재현성, 데이터 유출) |
| 대시보드 IaC 보안 탭 | 별도 독립 대시보드 (통합 화면의 한 탭으로만) |
| 코드 밖 변경 탐지 (로컬 apply·콘솔 수정을 드리프트와 State 읽기 주체 증가로) | 관리자·조직 오너 권한 탈취 방지 (신원·권한 관리 영역. 이 솔루션은 탐지만, ADR-12) |

**전제 조건: 사람에게는 AWS 쓰기 권한이 없고, 인프라 변경은 CI apply 역할로만 함 (ADR-12).**

- 팀원은 읽기 전용 권한으로 로컬 plan까지만 가능
- 이 전제가 없으면 plan 전 검사·경로 분석·State·드리프트 점검은 실수와 공급망 공격은 막지만, 쓰기 권한이 있는 사람은 로컬 apply로 우회 가능
- 관리자·조직 오너는 보호 설정 자체를 바꿀 수 있어 막지 못하고(Secret도 같은 전제), 인원 최소화·MFA·감사 로그로 줄임
- 현재는 팀원 5명이 IAM 사용자 장기 키를 가지고 CloudTrail이 꺼져 있어 이 전제가 아직 없음(적용은 팀 일정)
- 실제 사고 배경, 위협별 대응, 한계, 범위 밖 체크리스트는 [위협 모델·범위](threat-model.md)

## 기능 요구사항

- **MVP**: plan 전 검사 관문, 경로 비교, GitHub 연동, IaC 탭 기본 화면
- **다음 단계**: 수정 PR 자동 생성, 예외 관리, 설정 오류 목록
- 규칙 상세는 규칙 명세 문서 참고

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-G1 | PR의 `.tf`에서 실행 data 소스(`external`·`http`·`dns`), provisioner, `aws_lambda_invocation`을 찾아 차단 | MVP |
| FR-G2 | 허용 목록(프로바이더 단위) 밖 프로바이더는 차단, 모듈 ref·version과 프로바이더 버전 미고정은 경고 | MVP |
| FR-G3 | lock·backend·워크플로 파일 변경 감지. backend 변경은 중단, 나머지는 보안 리뷰 필수 | MVP |
| FR-G4 | `init -lockfile=readonly`로 프로바이더 해시 검증, 원격 모듈은 `terraform get`으로 내려받기만 한 뒤 재검사와 내용 해시 비교(Coder 사건 대응). 원격 모듈이 없는 지금부터 CI·로컬 래퍼에 포함 | MVP |
| FR-G5 | 로컬 plan 래퍼(`make plan`): CI와 같은 plan 전 검사(2b 포함) 후, 홈 디렉터리 없는 컨테이너에서 읽기 전용 임시 키로 `plan -lock=false`. 강제가 아니라 기본 경로 ([위협 모델·범위](threat-model.md)) | MVP |
| FR-P1 | 매일 Cartography 수집(AWS + Kubernetes) 후 부족한 관계(CloudFront→NLB, IMDS 자격 증명, 노드→EC2(`providerID`가 비어 내부 IP 매칭))를 보강 | MVP |
| FR-P2 | plan JSON을 그래프 복사본에 반영해 머지 전·후 "인터넷 → 데이터" 경로 차이 계산 | MVP |
| FR-P3 | 새 경로의 끝이 1등급이면 차단, 2등급·네트워크 도달만이면 경고, 경로 감소는 정보 | MVP |
| FR-P4 | 1등급 데이터에 닿는 권한은 IAM 시뮬레이터로 확인 후 확정 | MVP |
| FR-S1 | 매일 refresh-only로 드리프트 탐지, 노이즈 제거 후 높음·중간·낮음 분류 | MVP |
| FR-S2 | State를 속성 단위로 점검, State 읽기 주체 수 변화 추적 | MVP |
| FR-I1 | 검사 결과 2개(plan 전 검사·경로 분석, State는 경로 분석 절에 포함)와 요약 코멘트를 PR에 등록, 차단이면 병합 차단 | MVP |
| FR-I2 | 요약 코멘트 하나를 커밋마다 갱신, 원인 코드 줄에 코멘트 | MVP |
| FR-D1 | IaC 탭에 열린 경로, PR 판정, plan 전 검사 관문 상태, State·드리프트, 검사 주기 표시 | MVP |
| FR-D2 | 정해진 조치(IMDSv2 강제, 시크릿 읽기 축소, tfstate 버킷 정책, CloudTrail)를 수정 PR로 생성 | 다음 |
| FR-D3 | 예외를 사유·승인자·만료일과 함께 등록, 만료되면 다시 차단 | 다음 |
| FR-R1 | KICS 결과와 보완 규칙(0.0.0.0/0 인그레스, `*FullAccess` 연결)을 설정 오류 목록으로 표시 | 다음 |

## 비기능 요구사항

| ID | 항목 | 기준 |
|---|---|---|
| NFR-1 | 속도 | plan 전 검사 관문 1분 이내, PR 검사 전체(plan 전 검사 → plan → 경로 분석 → State) 10분 이내. 목표치, 구축 후 측정 |
| NFR-2 | 재현성 | 같은 커밋·같은 그래프면 항상 같은 판정. 판정과 문구는 규칙·템플릿으로만 생성 |
| NFR-3 | 오탐 | 현재 레포 기준 오탐 0 유지 (10/8 Conftest 348개 검사가 기준선). 반복 노이즈는 무시 목록으로 관리 |
| NFR-4 | 실패 처리 | 필수 검사가 오류로 끝나면 통과가 아니라 실패 처리(fail-closed). 재실행 가능 |
| NFR-5 | 최신성 | 자산 그래프는 24시간 이내. 더 오래되면 PR 코멘트에 기준 시각 표시 |
| NFR-6 | 유지보수 | 규칙은 버전을 붙여 레포로 관리, 위험 시나리오(S1~S6)를 회귀 테스트로 실행 |
| NFR-7 | 비용 | 오픈소스와 기존 클러스터·대시보드 PostgreSQL 사용(별도 DB 서버 없음). CloudTrail은 관리 이벤트 첫 복사본(무료)만 |

## 사용 시나리오

| # | 상황 | 시스템 동작 | 결과 |
|---|---|---|---|
| U1 | 팀원이 AI가 짠 코드로 "Grafana를 내 IP로만 열기" PR을 올림 (`data "http"`로 내 IP 조회) | plan 전 검사가 실행 data 소스와 허용 목록 밖 프로바이더를 찾음. plan과 자격 증명 발급 모두 안 함 | 차단 코멘트 + 대안(변수 `admin_cidr`) 안내 |
| U2 | 팀원이 "중복된 정책 정리"로 `deny_tfstate`를 지우는 PR을 올림 (permissive 상태) | plan 전 검사 통과 → plan → 경로 분석이 파드 → 노드 역할 → tfstate 새 경로를 찾음. State·드리프트 점검이 State 읽기 주체 5 → 6을 붙임 | 병합 차단, 보안 리뷰 요청, 버킷 정책으로 옮기는 수정 제안 |
| U3 | 보안 담당이 IaC 탭에서 "파드 → 노드 역할 → 시크릿 7개" 열린 경로를 확인 | 끊을 지점(IMDSv2 강제)과 방법을 보여줌. MVP는 사람이 PR을 직접 올림(자동 생성은 FR-D2). 이 PR도 plan 전 검사·경로 분석·State·드리프트 점검을 거침 | PR에 "경로 −1" 표시, 병합 후 다음 점검에서 경로 닫힘 |
| U4 | 누군가 콘솔에서 DNS 레코드를 바꿈 | 매일 06:00 드리프트 점검이 "높음"으로 분류하고 그래프 보정 | IaC 탭에 드리프트 표시, 담당 지정 후 코드 반영 또는 되돌리기 |

## 보안 요구사항

**이 솔루션 자체가 CI 자격 증명과 인프라 정보를 다루므로 검사 대상보다 엄격하게 적용.**

| ID | 요구사항 |
|---|---|
| SR-1 | PR용 plan 역할은 읽기 전용, OIDC `sub`는 인프라 레포의 `pull_request`로 제한. apply 역할은 별도로 두고 머지 후 main에서만 사용 |
| SR-2 | plan 전 검사 관문 통과 전에는 클라우드 자격 증명 발급 안 함 (PR 워크플로에서는 `id-token: write`를 plan job에만) |
| SR-3 | `pull_request_target` 금지, 포크 PR에는 자격 증명 미제공, 사용하는 Action은 커밋 SHA로 고정 |
| SR-4 | 시크릿 값은 출력·저장 안 함. 존재 여부, 길이, 주소, 속성 이름만 기록 |
| SR-5 | plan JSON은 민감 표시된 값을 지운 뒤 전송, 분석 후 삭제. 결과만 저장 |
| SR-6 | 스캐너는 노드 역할이 아닌 전용 읽기 역할 사용. State 버킷 읽기는 명시적 허용 목록으로만 |
| SR-7 | 도구 이미지·Action은 버전 고정, 업데이트는 리뷰 후 반영 (2026년 3월 Trivy Action 사건) |
| SR-8 | GitHub App은 최소 권한(checks 쓰기, pull requests 쓰기, contents 읽기). 대시보드에서 다시 검사하지 않으므로 actions 쓰기는 주지 않음 (10/21). 수정 PR 생성용 컨텐츠 쓰기는 별도 App으로 분리. App 개인 키는 Secrets Manager에 보관 |
| SR-9 | 대시보드·API는 인증된 팀원만, CI 호출은 GitHub OIDC 토큰 검증. CI용 엔드포인트만 외부 노출(요청 크기·횟수 제한), Neo4j와 결과 DB는 클러스터 내부에만 |
| SR-10 | 예외 승인, 재검사, 수정 PR 생성은 누가 언제 했는지 기록 |

## 배포 전 점검 결과 (11/04)

**SR 10개 중 6개 충족, 4개는 일부 충족. 남은 부분은 모두 배포(대시보드·CI → API 연결)나 팀 일정(apply 역할)에 묶여 있음.**

- 근거는 infrastructure develop, dashboard-backend·dashboard-frontend develop + 11/04 브랜치, 로컬 데모 기준

| ID | 결과 | 근거 | 남은 것 |
|---|---|---|---|
| SR-1 | 일부 | plan-ro: `ReadOnlyAccess` + 시크릿 값·버킷 객체 거부, 신뢰 정책 `sub`는 `…:pull_request`만 | apply 역할 없음. CI apply는 ADR-12 팀 일정 |
| SR-2 | 일부 | 자격 증명 단계가 gate 단계 뒤에 있고, gate가 실패하면 받지 않음 | `id-token: write`가 gate·plan이 같이 도는 job 전체에 걸림. plan을 별도 job(`needs: gate`)으로 나누면 충족 |
| SR-3 | 충족 | `pull_request_target` 없음, 포크 PR은 plan 안 함, Action 8곳 모두 커밋 SHA 고정 | |
| SR-4 | 충족 | API·화면·PR 코멘트에 주소·속성 이름·개수만. plan은 2단계 지우기(ADR-06) 후 전송 | |
| SR-5 | 충족 | `scrub-plan.sh` 2단계, 분석이 끝나면 `iac_plan_payloads` 삭제 (워커 테스트로 확인) | |
| SR-6 | 충족 | 수집은 전용 수집 역할(`logssey-dashboard-role-iac-collector`, 대시보드 호스트 1시간 세션), 시크릿 값·객체 읽기 거부. State 내용은 읽지 않음. 예전 `scanner-ro`(IRSA)는 11/19 삭제 | |
| SR-7 | 충족 | conftest·terraform 버전 + 체크섬, 컨테이너 이미지 digest, Action SHA | |
| SR-8 | 일부 | App 권한 checks·pull requests 쓰기, contents 읽기. actions 쓰기 없음 | 개인 키는 Secrets Manager가 아니라 main 전용 environment(`iac-app`)의 GitHub secret |
| SR-9 | 일부 | 대시보드 API는 로그인한 관리자만, Neo4j는 NetworkPolicy로 클러스터 안만 | CI OIDC 검증, 요청 횟수 제한, 결과 DB 위치는 배포 때 |
| SR-10 | 충족 (지금 있는 쓰기 기능 기준) | 발견 건 담당·상태 변경은 `iac_audit_events`에 사용자와 전후 값 기록, 분석·점검 작업도 기록 | 예외 승인·수정 PR은 다음 단계 기능 |

**NFR-1 속도**

**운영 끝에서 끝 검증 (2026-11-19)**: #58 시나리오(노드 역할의 `deny_tfstate` 삭제)를 테스트 PR #75로 올려 머지하지 않고 닫음. plan 전 검사 통과 → plan(삭제 1) → 경로 분석 실패(`path.tier1.added` 6개: 인터넷 → CloudFront → NLB → EC2 → 노드 역할 → tfstate, IAM 시뮬레이터 explicitDeny → allowed) + State 읽기 주체 증가 경고 → 대시보드 PR 목록·상세에 차단·Critical·경로 +6. 오판 없음. main 매일 점검(iac-drift) 수동 실행: plan 전 검사 main 위반 0, 드리프트 5건 탐지

| 구간 | 목표 | 측정 | 방법 |
|---|---|---|---|
| plan 전 검사(gate) | 1분 이내 | 1.1~2.6초 (3회, 도구 설치 제외) | 로컬 `GATE_BASE=origin/develop policy/gate.sh` |
| 경로 분석 + State 영향 | | 1.2초 (단독), 10.4~10.7초 (그래프 보강과 겹쳐 대기한 경우, IAM 시뮬레이터 호출 포함) | 로컬 데모, `iac_jobs` 등록 → 완료 시각 |
| 그래프 보강 / State 점검 | | 6.2초 / 11.3초 | 같은 데모 (열린 경로 60개 그래프) |
| PR 검사 전체 (plan 전 검사 → plan → 경로 분석 → State) | 10분 이내 | **1분 41초** (PR 열림 → "경로 분석" 등록). gate·plan 54초, "plan 전 검사" 등록 1분 24초 | 2026-11-19 운영 끝에서 끝 검증 (infrastructure 테스트 PR #75, GitHub 검사 시각) |

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |
| 수정일 | 2026-11-04 |
