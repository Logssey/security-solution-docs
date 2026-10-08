# 유효성, 사용 여부, 위험도 조사

조사 반영일: 2026-10-08, 공식 문서 기반 API 후보. 실제 계정에서 키를 검사한 결과가 아님.

## 1. 구별할 판단

| 판단 | 답하려는 질문 | 주의점 |
|---|---|---|
| 유형 탐지 | 인증, 서명 등에 사용될 수 있는 값인가? | 룰 일치는 유효성 증명이 아님 |
| 유효성 | 특정 시각에 해당 방식으로 인증할 수 있는가? | 인증 성공이 모든 자원 접근을 뜻하지 않음 |
| 사용 여부 | 실제 서비스, 사용자가 사용한 이력이 있는가? | 최근 기록 없음은 미사용 확정이 아님 |
| 영향 범위 | 어떤 주체, 권한, 자원에 연결되는가? | 정책, 조직 제한, 리소스 정책 등으로 달라짐 |
| 심각도 | 어떤 순서로 대응할 것인가? | 유효성, 권한, 노출, 환경, 악용 근거를 함께 고려 |

## 2. 공급자, 유형별 검증 후보

| 발급자, 유형 | 최소 검증 후보 | 필요한 값, 주체 | 알 수 있는 것 / 한계 |
|---|---|---|---|
| AWS IAM, STS 자격 증명 | STS `GetCallerIdentity` | ID+Secret, 임시 자격 증명은 SessionToken 포함 | Account/Arn/UserId. 해당 키의 S3, RDS 등 실제 권한 전체를 증명하지 않음 |
| AWS 장기 접근 키 사용 메타데이터 | IAM `GetAccessKeyLastUsed` | 대상 AccessKeyId, 조회 권한 있는 별도 연결 자격 증명 | 최근 사용 시각, 서비스, 리전. 대상 Secret 없이 메타데이터 조회 가능하며 인증 검증과 별개 |
| GitHub PAT, OAuth 사용자 토큰, App 사용자 토큰 | `GET /user` | 해당 사용자 토큰 | 인증된 사용자. 저장소 전체 권한, 실제 업무 사용을 보장하지 않음 |
| GitHub App installation token | `GET /installation/repositories` | installation token | 설치 토큰으로 접근 가능한 저장소 목록. 사용자 토큰 검증과 별도 계약 |
| Notion 연결 토큰, PAT | `GET /v1/users/me` | Bearer token, 지원 API 버전 | 토큰에 해당하는 bot 또는 사용자. 모든 페이지 접근 가능 여부와 별개 |
| Notion 지원 토큰의 introspection | `POST /v1/oauth/introspect` | 검사 대상 토큰과 올바른 OAuth client 인증 | active 등 토큰 메타데이터. 임의 발견 토큰에 무조건 적용 가능한 경로가 아님 |
| K8s ServiceAccount, 클라이언트 인증 | 허용 클러스터에서 유형별 검증 계약 설계 | 토큰 또는 인증서, 개인키, 신뢰할 서버 정보 | 인증과 RBAC 허가를 구별. 단일 범용 엔드포인트, 권한은 이번 초안에서 확정하지 않음 |
| 개인키, 웹훅 secret, 앱 client secret, refresh token, 임의 암호 | 정적 구조 확인 + 유형별 지원 조사 | 유형별 상이 | 모든 값에 안전한 범용 ‘유효성 API’가 존재하지 않음. 갱신, 발급으로 검증하지 않음 |

AWS STS GetCallerIdentity는 권한을 요구하지 않으며 명시적 거부가 있어도 신원 정보를 반환할 수 있다. 따라서 성공 응답을 업무 리소스 권한 증거로 사용하지 않는다. [AWS 공식 API](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html)

최근 사용 메타데이터는 유효성 검사와 별개로 수집한다. 검증 호출 자체가 사용 기록에 섞일 가능성을 고려해 검사 시각을 함께 남기는 것은 본 프로젝트의 설계 제안이다. [GetAccessKeyLastUsed](https://docs.aws.amazon.com/IAM/latest/APIReference/API_GetAccessKeyLastUsed.html)

GitHub는 사용자 계열과 설치 계열 토큰의 API를 구분한다. 공개 엔드포인트의 200 응답만으로 토큰이 유효하다고 판정하지 않는다. [인증 사용자 조회](https://docs.github.com/en/rest/users/users#get-the-authenticated-user), [설치 저장소 조회](https://docs.github.com/en/rest/apps/installations#list-repositories-accessible-to-the-app-installation)

Notion의 self 조회와 introspection은 사용 조건이 다르다. introspection의 client 인증 오류는 검증기 설정 문제일 수 있으므로 대상 토큰 무효로 단정하지 않는다. [self 조회](https://developers.notion.com/reference/get-self), [introspection](https://developers.notion.com/reference/introspect-token)

## 3. 실패 응답의 해석

| 결과 | 설계상 처리 원칙 |
|---|---|
| 인증된 주체 확인 성공 | 해당 시각, 방법의 VALID 근거 기록 |
| 401 등 명시적 인증 거절 | 엔드포인트별 계약과 검증기 구성을 확인한 후 INVALID 가능. 만료, 폐기 원인을 추측하지 않음 |
| 403 | 권한 부족, 정책 제한, 호출 제한 가능. 유형별 분석 없이 무효 처리 금지 |
| 404 | 대상 부재 또는 비공개 자원 접근 제한 가능. 인증 무효와 구분 |
| 429, 제한 헤더 | UNKNOWN/RATE_LIMITED, 재시도 시각 기록 |
| 5xx, 타임아웃, DNS, 네트워크 오류 | UNKNOWN, 공급자/네트워크 사유 기록 |
| 검증기 client 인증 실패 | UNKNOWN/VALIDATOR_AUTH_ERROR, 운영 점검 |
| 필요한 구성요소, 원문 없음 | 미검증 또는 시도 후 UNKNOWN, 사유 명시 |

상태 코드는 공통 초안이며 최종 매핑은 공급자, 유형, 엔드포인트별로 검증해야 한다. Notion, GitHub 오류 문서가 권한, 호출 제한과 인증 실패를 구별한다. [Notion 상태 코드](https://developers.notion.com/reference/status-codes), [GitHub REST 오류 해결](https://docs.github.com/en/rest/using-the-rest-api/troubleshooting-the-rest-api)

## 4. 도구 결과 해석

Gitleaks의 정규식, 키워드, 엔트로피, 허용 목록은 후보 탐지를 돕는다. TruffleHog는 탐지기별 검증 지원이 다르며 verified, unverified, unknown을 구별한다. 특히 unverified는 ‘폐기된 키’와 동의어가 아니다. 지원 유형, 룰, 버전을 고정해 자체 매핑 표를 작성한다. [Gitleaks](https://github.com/gitleaks/gitleaks), [TruffleHog](https://github.com/trufflesecurity/trufflehog/blob/main/README.md)

GitHub Secret scanning의 지원 패턴과 validity 기능도 모든 임의 비밀값에 대한 완전한 검증을 보장하지 않는다. 이를 자체 엔진의 지원 목록과 동일하게 취급하지 않는다. [지원 패턴](https://docs.github.com/en/code-security/reference/secret-security/supported-secret-scanning-patterns), [유효성 확인](https://docs.github.com/en/code-security/concepts/secret-security/validity-checks)

## 5. 위험 등급 조사 예시

점수 대신 **CRITICAL/HIGH/MEDIUM/LOW** 네 등급을 사용한다. 아래 표는 초기 조사에서 작성한 예시이며 외부 기관이 정한 공식 등급표가 아니다. 현재 제품 판정 기준은 [심각도 정책](../02-functional-design/severity-policy.md)을 따른다.

| 등급 | 예시 조건 | 함께 필요한 근거 |
|---|---|---|
| CRITICAL | 공개 노출과 운영 관리자 권한이 확인되거나, 중대한 악용 근거가 확인됨 | 노출 범위, 권한, 대상, 시점, 악용 사실 |
| HIGH | 비정상 노출과 중요 운영 자원 접근 가능성이 충분한 근거로 확인됨 | 제한된 범위라도 중요 데이터, 서비스 영향 |
| MEDIUM | 비정상 노출이 있으나 영향이 제한된 권한, 자원으로 확인됨 | 실제 제한 범위와 노출 조건 |
| LOW | 낮은 영향의 제한된 자원, 환경, 권한으로 확인됨 | 낮은 영향이라는 긍정적 근거. 단순 ‘정보 없음’ 제외 |
| 미평가 | 판정에 필요한 증거 부족 | 부족한 정보와 수동 검토 사유. 다섯 번째 심각도 등급이 아님 |

유효성 UNKNOWN이어도 알려진 노출, 영향 근거를 보존한다. 기존 높은 심각도를 검사 오류 때문에 LOW로 낮추지 않는다. 무효가 된 키도 과거 악용, 후속 영향이 있으면 기존 사고 검토를 유지한다. 실제 권한의 의미와 정책 적용 결과는 공급자별 검증 사례로 확인한다.

NIST SP 800-61r3의 사고 영향, 우선순위 관점과 OWASP의 비밀 수명주기, 최소 권한, 감사 원칙을 참고했다. 이 출처들이 위 네 등급 예시나 특정 점수를 규정한다는 뜻은 아니다. [NIST SP 800-61r3](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r3.pdf), [OWASP Secrets Management](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

## 6. 실제 검증으로 남은 일

테스트 전용 자격 증명으로 정상, 무효, 구성요소 누락, 권한 제한, 호출 제한을 비교한다. 키 값은 보고서에 남기지 않고 검증기 버전, 상태, 이유, 시각만 기록한다. 실제 운영 키를 발급, 폐기하는 실험은 이번 문서 작업에 포함되지 않는다.

## 7. 상용 제품의 공개된 우선순위 판단 방식

다음 내용은 공개 문서에 명시된 기능을 비교한 것이다. 내부 모델의 가중치나 기본 규칙의 전체 조건은 공개 자료만으로 재현할 수 없다.

| 제품 | 공개 문서에서 확인한 방식 | 이 설계에 반영한 점 |
|---|---|---|
| GitGuardian Internal Monitoring | 유효성, 공개 노출, 탐지 위치와 원천 맥락을 조사한다. 심각도는 사용자 지정 가능한 규칙을 위에서 아래로 평가해 처음 일치한 규칙을 적용하고, 여러 발견 위치에는 가장 높은 심각도를 사용한다. 별도 ML 위험 점수는 0에서 100까지이며 비밀 유형, 유효성, 탐지 맥락, 노출 패턴 등을 입력으로 쓴다. | 등급과 점수를 구별하고 규칙 순서, 위치별 평가, 근거와 수동 변경을 명시했다. 독자적인 수치 점수는 도입하지 않는다. |
| GitHub Secret Scanning | 알림에 토큰 유효성, 공개 유출 여부, 여러 저장소 발견 여부, 지원되는 토큰 메타데이터와 발견 위치를 제공한다. 유효성 검사 결과로 대응 우선순위를 판단하도록 안내한다. 공개된 Secret Scanning 문서는 모든 비밀에 적용되는 공통 4등급 위험 산식을 제시하지 않는다. | 유효성과 노출 맥락을 별도로 수집하고, 인증 성공을 권한 또는 최종 위험 등급의 증거로 간주하지 않는다. |
| TruffleHog Enterprise Analyze | 검증된 키라도 실제 위험 판단에는 연결된 자원의 소유자, 접근 가능한 데이터 범위, 권한을 조사한다. 공개된 설명은 AWS 테스트 버킷만 접근하는 키와 운영 저장소에 접근하는 GitHub 키의 영향이 다르다는 사례를 든다. | 공급자별 권한과 대상 조회가 없으면 영향 범위를 임의로 채우지 않는다. |

근거: [GitGuardian 사고 우선순위와 심각도 규칙](https://docs.gitguardian.com/internal-monitoring/remediate/prioritize-incidents), [GitGuardian 위험 점수 입력](https://docs.gitguardian.com/secrets-detection/secrets-detection-engine/machine_learning), [GitHub Secret Scanning 알림 평가](https://docs.github.com/en/code-security/tutorials/remediate-leaked-secrets/evaluating-alerts), [GitHub Secret Scanning API 필드](https://docs.github.com/en/rest/secret-scanning/secret-scanning), [TruffleHog Analyze의 영향 분석 설명](https://trufflesecurity.com/blog/why-trufflehog-analyze-is-a-game-changer-for-security-teams).

제품에 적용할 결정적 규칙은 [심각도 판정 정책](../02-functional-design/severity-policy.md)의 `severity-v2`를 따른다. 공급자별 권한과 자산 중요도를 읽지 못하는 경우 미평가로 둔다는 제한도 그 정책의 일부다.
