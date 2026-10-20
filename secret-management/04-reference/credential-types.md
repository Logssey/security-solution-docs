# 자격 증명 유형과 수집 경계

비밀 값·공개 식별자·복합 자격 증명을 구분하는 분류 기준이다. 연결별 허용 콘텐츠와 실제 capability는 [수집원과 검증기 계약](../03-architecture/source-and-validator-contract.md)을 따른다.

## 1. 관리 대상으로 판단하는 기준

| 분류 | 기준, 예 | 처리 기준 |
|---|---|---|
| 비밀 인증 값 | 토큰, 비밀번호, 개인키, 클라이언트 secret, 공유 secret | 원문 보호, 마스킹, 유형별 검증 가능성, 만료, 폐기, 노출 추적 |
| 복합 자격 증명 | AWS ID+Secret(+SessionToken), 인증서+개인키, 사용자명+암호 | 구성요소와 관계를 보존. 일부만 발견해도 누락시키지 않음 |
| 접근 권한을 담은 값 | 서명 URL, 쿠키, 세션 값, 등록 토큰 | 만료가 짧아도 범위 포함. 임의 URL 실행이나 세션 재현은 기본 검증으로 삼지 않음 |
| 암호화, 서명 재료 | 대칭키, 평문 data key, 서명용 개인키, 복구 코드 | 민감 자산으로 추적. 일반 로그인 검증과 다른 수명주기 |
| 제한 공개 값 | 공개 클라이언트용 API key 등 | 용도별 도메인, 앱, 기능, 쿼터 제한을 확인. 노출만으로 사고 단정 금지 |
| 식별자, 공개 재료 | AccessKeyId, ClientId, ARN, 공개키, 인증서, CA 인증서 | 관련 자격 증명과 연결하되 단독 비밀로 오분류하지 않음 |
| 내부 키 자원 | KMS 키, HSM 내부 키 등 | 키 재료 추출을 전제하지 않고 정책, 권한, 수명주기를 관리 |

문자열 이름보다 실제 사용 기능이 기준이다. 임의 사용자 정의 secret과 미래 서비스까지 모든 문자열, 변수명을 닫힌 목록으로 나열할 수는 없다. 등록된 포맷·문맥 룰과 유형별 검사 capability를 사용하고 미지원 유형은 이유를 표시한다.

## 2. 플랫폼별 키 계열

| 플랫폼 | 분류 계열 | 유의점 |
|---|---|---|
| AWS | IAM 장기 키, STS 임시 키, 콘솔 암호, SSO 토큰, 서비스 전용 자격 증명, Bedrock, SES, ECR, CodeArtifact, API Gateway, AppSync, Location, Cognito 토큰, client secret, DB, 캐시, 디렉터리 암호, 임시 인증, SSH, TLS, IoT, VPN, 서명 URL, 쿠키, 암호화 재료, 등록 코드, 사용자 정의 secret | IAM 키 메타데이터를 조회한다고 기존 Secret 값을 모두 복원할 수 있는 것은 아님 |
| GitHub | fine-grained/classic PAT, OAuth 사용자 토큰, 앱 client secret, GitHub App 개인키, JWT, 사용자/설치 토큰, refresh token, Actions GITHUB_TOKEN, SSH 개인키, deploy key의 개인키, webhook secret, 패키지, 레지스트리 인증에 쓰는 토큰, 세션, 복구 수단, 저장소/환경/조직에 저장한 임의 secrets | Actions Secret은 저장 위치이고 발급자는 AWS 등일 수 있음. 공개키, 앱 ID, client ID를 비밀과 구별 |
| Notion | 내부 연결 토큰, OAuth 연결 access/refresh token, client secret, PAT, webhook verification 값, 공유된 외부 서비스 키, 암호, 개인키 | 문서 수집용 연결 토큰과 문서에서 발견한 비밀은 별개. 사용자 세션 쿠키를 수집 연결 수단으로 가정하지 않음 |
| K8s | Secret 객체, ServiceAccount 토큰, kubeconfig 인증값, TLS, SSH 개인키, 이미지 레지스트리 인증, 부트스트랩 토큰, 워크로드 안의 임의 외부 키 | 객체 이름보다 내부 데이터 의미가 기준. AWS, GitHub, Notion 키가 그대로 들어갈 수 있음 |

공급자별 분류 계열과 실제 탐지·검증 capability는 구분한다. 카탈로그에 속하더라도 등록된 검사기가 없으면 미지원 이유를 표시한다.

## 3. Kubernetes 자격 증명

Kubernetes Secret은 비밀을 담는 객체이며 `data`의 Base64 인코딩 자체는 암호화가 아니다. 기본 유형과 사용자 정의 유형을 모두 분류해야 한다. [공식 Secret 문서](https://kubernetes.io/docs/concepts/configuration/secret/)

| 객체, 유형 | 관리할 값 | 분류, 제약 |
|---|---|---|
| Opaque, 사용자 정의 Secret | 임의 토큰, 비밀번호, 연결 문자열, 서명키, 외부 API 키 | 값별 발급자, 유형 재분류 필요 |
| kubernetes.io/service-account-token | token | 장기 토큰 계열. ca.crt와 namespace는 비밀 토큰과 구별 |
| kubernetes.io/dockercfg | .dockercfg 내부 인증 값 | 레지스트리 사용자명, 암호/토큰, auth 필드 |
| kubernetes.io/dockerconfigjson | .dockerconfigjson 내부 인증 값 | Base64 auth도 디코딩 후 자격 증명으로 취급 |
| kubernetes.io/basic-auth | password와 대응 username | 사용자명은 식별자, 암호는 비밀 |
| kubernetes.io/ssh-auth | ssh-privatekey | SSH 개인키, 별도 암호문구가 있으면 함께 관리 |
| kubernetes.io/tls | tls.key, 필요시 체인 관련 메타데이터 | 개인키는 비밀, tls.crt는 공개 인증 재료 |
| bootstrap.kubernetes.io/token | token-secret과 token-id 조합 | secret은 비밀, ID는 대응 식별자 |
| projected ServiceAccount 토큰 | 워크로드에 제공되는 단기 토큰 | Secret 객체 목록만으로 전부 포괄할 수 없음 |
| kubeconfig | token, client-key-data, 개인키 참조, 인증 공급자 관련 값 | client certificate, CA, server URL과 비밀 구별. exec 참조가 실제 비밀 문자열은 아닐 수 있음 |

ServiceAccount의 단기 토큰 발급, 회전은 Secret 객체에 저장된 장기 토큰과 다르다. 따라서 ‘Secret 목록 수집’과 ‘모든 런타임 인증 수단 수집’은 지원 범위를 별도로 기록한다. [ServiceAccount 문서](https://kubernetes.io/docs/concepts/security/service-accounts/)

kubeconfig는 인증값뿐 아니라 파일, 플러그인 참조를 포함할 수 있다. 분석할 때 포함된 명령을 자동 실행하지 않는다. [kubeconfig 공식 문서](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

## 4. 발견 위치와 수집 경계

| 소스 | 수집 설계 대상 | 별도 범위, 제약 |
|---|---|---|
| GitHub | 허용 저장소의 파일, 설정, Git 이력 | 브랜치·태그·이력 범위를 명시. 이슈, PR 본문, 댓글, 위키, Actions 로그, 아티팩트는 별도 수집 범위 |
| Notion | 연결에 허용된 페이지, 하위 블록, 지원 텍스트 | 공유 권한, 페이지네이션, 중첩 블록, 첨부 파일/OCR 지원을 구분. 접근 불가 영역은 누락 범위로 보고 |
| AWS | 승인된 리소스, 설정, IAM, 사용 메타데이터 | Secrets Manager/Parameter Store 내용 열람, S3 객체, 로그, 함수 설정 등은 자원별 권한, 수집 범위를 정함 |
| K8s | 승인된 네임스페이스의 Secret | Secret·ConfigMap·워크로드 참조는 각각의 콘텐츠 범위로 구분 |
| 런타임, 외부 저장소 | 환경변수, 마운트 파일, 외부 비밀 저장소, 이미지 레이어 등 | 명시적 승인 범위로 구별. Pod exec, 노드, etcd, 이미지 전체 수집을 기본 수집에 포함하지 않음 |

각 원천은 승인 콘텐츠·권한·이력 범위를 명시한다. 수집 실패와 실제 0건을 구별하고 정상 비밀 저장소의 값 존재만으로 비정상 노출을 판정하지 않는다.

## 5. 유형 등록 필드

유형 등록부는 발급자, 유형 ID, 비밀·식별자 구성요소, 포맷·문맥 규칙, 엔진·룰 버전, 허용 발견 위치, 정적 검사, 승인된 검증 방법, 권한·사용 메타데이터, 미지원 사유, 공식 출처와 확인일을 기록한다. 접두사만으로 유효성을 확정하지 않는다.

탐지·검증 capability는 유형별로 선언한다. 안전한 범용 검증 방법이 없는 개인키·웹훅 secret·임의 암호는 정적 근거와 수동 확인 경로를 사용한다. 임의 URL 접속·재발급·토큰 갱신으로 유효성을 시험하지 않는다.

관련: [수집·검증 계약](../03-architecture/source-and-validator-contract.md), [심각도 정책](../02-functional-design/severity-policy.md).
