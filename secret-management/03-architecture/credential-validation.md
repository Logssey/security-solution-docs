# 발견 자격 증명의 능동 유효성 검증

기준: [수집원과 검증기 계약](source-and-validator-contract.md), [데이터 계약](data-contract.md), [API 계약](api-contract.md), [DB 스키마](database-schema.md), [보안과 실행 정책](security-and-runtime.md), [ADR-S002](adr/ADR-S002.md), [ADR-S005](adr/ADR-S005.md). 권한·영향·노출·심각도는 [위험 평가 계약](risk-assessment-contract.md)의 독립 결과다.

## 1. 검증이 확인하는 사실

능동 검증은 **발견한 원문 자격 증명으로 승인된 최소 인증 요청이 성립하는지** 확인한다. 수집 연결의 GitHub App·설치 토큰·AWS 역할·프로필로 요청한 성공은 발견 키의 성공이 아니다. 발견 키를 공급하지 못하면 수집 자격 증명이나 SDK 기본 자격 증명 체인으로 대체하지 않는다.

다음 사실은 서로 대체하지 않는다.

| 사실 | 근거와 경계 |
|---|---|
| 연결 probe | 수집용 STS 신원 또는 GitHub App 설치 인증 확인. `secret_connection_probes`에 기록하며 발견 키·본문 읽기·전체 수집 성공을 만들지 않음 |
| 발견 키 유효성 | 해당 `CredentialSet`과 동일함을 확인한 원문으로 최소 인증 요청을 수행한 `ValidationAttempt` |
| 위치 존재 | 현재 승인 위치의 비교 가능한 재수집·재탐지. 인증 성공/거절로 `PRESENT`·`MISSING`을 변경하지 않음 |
| 수명주기 | 명시적인 만료·폐기 근거. 인증 거절만으로 폐기·만료 원인을 확정하지 않음 |
| 사용·권한·영향·노출·심각도 | 별도 출처·관측·승인 계약. 검증 요청 자체를 업무 사용으로 세지 않음 |

결과는 `VALID`, `INVALID`, `UNKNOWN`, `UNCHECKED` 네 가지다. `INVALID`는 이 검증에서 인증이 명확히 거절되었다는 뜻이며 안전한 보관·폐기 완료·위치 해결을 뜻하지 않는다. 인증 가능성은 관측 시점의 사실이고 향후 사용 가능성을 보장하지 않는다.

## 2. 검증기 등록과 호출 경계

실행 가능한 검증기 등록에는 발급자, 정확한 자격 증명 유형, 필수 구성요소, 검증기 ID·버전, 승인 호스트·포트·경로·메서드, 고정 요청 헤더와 API 버전, 응답 분류 버전, 호출 한도, timeout, 재시도 정책, 합성 시험 결과를 포함한다. 등록만으로 활성화하지 않으며 공급자별 명시적 활성화와 현재 범위의 능동 검증 승인이 모두 필요하다. 둘 중 하나라도 없으면 외부 호출하지 않는다. 승인 범위는 수집 권한과 별도로 버전 관리한다.

연결 SECURITY는 `키 검증 설정`에서 연결 활성화와 허용할 키 발급 서비스 `AWS|GITHUB|NOTION`을 명시한다. 발급 서비스는 원문을 읽는 플랫폼과 독립이다. GitHub 저장소에서 발견한 AWS 키는 AWS 허용 여부로 검사한다. 서버의 등록 profile은 `credential-validation-v1`이며 사용자에게 profile·endpoint 편집을 요구하지 않는다. 실제 호출에는 서버 검증기 활성화, 연결 활성화·허용 발급 서비스, 해당 범위의 현재 승인, 지원되는 완전한 자격 증명과 현재 권한이 모두 필요하다. 발급 서비스 미허용은 `UNCHECKED/VALIDATOR_NOT_ALLOWED`로 기록한다.

연결 설정 변경은 검증 revision·감사를 갱신하고 이전 관측의 현재 적용 여부와 대기 검증 작업을 재검사한다. 호출 직전과 결과 저장 전에 현재 연결·발급 서비스·범위 승인을 다시 확인한다. 설정 저장은 이후 검증 요청에 적용되며 저장 자체가 외부 인증 요청을 등록하지 않는다. 서버가 꺼져 있어도 허용할 서비스를 저장할 수 있고 화면은 남은 서버 설정을 안내한다.

| 등록 capability | 입력 | 최소 요청과 결과 범위 |
|---|---|---|
| GitHub Classic PAT | 완전한 `GITHUB_CLASSIC_PAT` 원문 | `GET https://api.github.com/user`. 인증 사용자 확인 |
| GitHub Fine-grained PAT | 완전한 `GITHUB_FINE_GRAINED_PAT` 원문 | 같은 `GET /user`. 이 endpoint에는 별도 fine-grained permission이 필요하지 않음 |
| GitHub installation token | 해당 유형의 원문·별도 설치 검증기 등록 | 사용자 PAT capability와 분리. PAT 검증기로 보내거나 수집용 설치 토큰 발급 성공으로 대체하지 않음. 별도 등록·시험·승인 전에는 `UNCHECKED/UNSUPPORTED_TYPE` |
| AWS 장기 키 쌍 | AccessKeyId·SecretAccessKey가 같은 구성이라는 근거 | 기존 STS `GetCallerIdentity`. 발견한 쌍을 명시적 `StaticCredentialsProvider`로 공급 |
| AWS 임시 자격 증명 | 같은 키 쌍과 session token | 같은 STS 요청. session token 누락은 `UNCHECKED/MISSING_SESSION_TOKEN` |
| Notion 콘텐츠 API 토큰 | 완전한 `NOTION_CONNECTION_TOKEN` 후보와 현재 승인 원천의 동일성 근거 | `NOTION_GET_SELF`, endpoint class `NOTION_SELF`, `GET https://api.notion.com/v1/users/me`. 현재 인증과 제한된 주체 정보 확인 |
| 기타 유형·불완전 후보·메타데이터 | 안전한 최소 인증 방법 또는 필수 구성 미충족 | 등록된 capability가 없으면 호출하지 않음. `METADATA_ONLY`에는 값 인증 시도를 생성하지 않음 |

GitHub PAT의 `/user` 지원과 permission 조건은 [인증 사용자 조회 공식 계약](https://docs.github.com/en/rest/users/users#get-the-authenticated-user)을 따른다. 공개 저장소 조회 성공이나 특정 저장소 접근 실패로 PAT 유효성을 판정하지 않는다. GitHub Enterprise 등 다른 인스턴스는 별도 호스트·검증기 등록 전까지 지원 범위가 아니다.

GitHub 요청은 `api.github.com:443`, HTTPS, 정확한 `/user`, `GET`, query/body 없음으로 고정한다. 발견 PAT를 `Authorization: Bearer`에 넣고 `Accept: application/vnd.github+json`, `User-Agent: logssey-secret-worker`, `X-GitHub-Api-Version: 2026-03-10`을 검증기 등록의 고정값으로 사용한다. 발견 문서의 URL·호스트·헤더·프록시 지시를 요청에 반영하지 않는다. redirect는 따라가지 않으며 TLS 인증서·호스트 검증을 끄지 않는다. 승인 egress 설정과 호스트 allowlist를 함께 적용하고 인증 헤더를 다른 호스트로 전송하지 않는다.

AWS는 기존 승인 STS endpoint·서명 리전을 고정하고 SDK 내부 재시도도 전체 호출 예산에 포함한다. `SECRET_AWS_VALIDATION_ENABLED`는 기본 false이며 AWS 수집·본문 읽기 활성화와 독립이다. AWS 목록 메타데이터만으로 키 쌍을 구성하거나 검증하지 않는다. 기본 자격 증명 체인·개인 로컬 프로필·수집 연결 역할로 발견 키 검증에 fallback하지 않는다.

Notion 요청은 `api.notion.com:443`, HTTPS, 정확한 `/v1/users/me`, `GET`, query/body 없음으로 등록한다. 헤더는 발견 원문의 `Authorization: Bearer`, `Notion-Version: 2026-03-11`, `Accept: application/json`, 고정된 워커 User-Agent로 구성한다. `SECRET_NOTION_VALIDATION_ENABLED`의 기본값은 false이며 원천 수집 플랫폼·수집 연결의 토큰과 별개로 적용한다. redirect·임의 endpoint·쿠키 인증은 허용하지 않는다. Notion 콘텐츠 API의 내부 연결·PAT·OAuth access token 후보는 같은 최소 인증 계약을 사용할 수 있다. `ntn_`·기존 `secret_`는 포맷 후보 근거이며 세부 발급 유형은 등록·발급 근거로 확인한다. refresh token·client secret·Admin·SCIM 자격 증명은 각 등록 capability로 구분한다. [인증 요청](https://developers.notion.com/reference/get-self), [PAT](https://developers.notion.com/guides/connections-and-tokens/personal-access-tokens)

## 3. 원문 전달·재취득과 동일성

정규화기는 저장 가능한 안전 메타데이터와 **메모리 전용 검증 후보**를 같은 파싱에서 분리한다. 후보는 발급자·유형·필수 구성, 해당 `CredentialSet`의 보호된 동일성, 원천 객체·버전·Finding 연결을 갖는다. GitHub 후보도 AWS 후보와 같은 경계를 사용한다. 원문 포함 객체의 직렬화·일반 `toString()`·자동 디버그 출력은 금지하며 `toString()`은 고정된 숨김 표현과 안전한 유형만 반환한다.

`FULL_SCAN`은 원천 콜백 안에서 읽은 동일 바이트로 탐지·정규화한 뒤 키·위치·Finding을 짧은 트랜잭션으로 저장한다. 검증 예약을 확보한 후보만 같은 콜백의 제한된 메모리에서 검증한다. DB 트랜잭션을 연 채 공급자 HTTP를 기다리지 않는다. 예산·lease·승인 때문에 지금 검증할 수 없으면 원문을 버리고 내부 ID로만 후속 작업을 보존한다.

비동기 요청·재시도·프로세스 재시작 후에는 원문을 큐에서 복원하지 않는다. 현재 승인 원천에서 다음 절차로 재취득한다.

비동기 검증 시도는 재취득 직전에 원문 없는 예약을 확보한다. 따라서 실제 재취득 실패도 해당 실행의 완료된 UNKNOWN 시도로 기록할 수 있다. `FULL_SCAN`의 최초 검증은 이미 읽은 바이트·Finding을 저장한 뒤 예약하며 다시 읽기 위해 원문을 보관하지 않는다.

1. 대상 `CredentialSet`, 대상 위치, 연결 소속, 현재 scope revision, 위치 기대 버전, 수집·검증 승인과 요청 권한을 확인한다.
2. 현재 승인 범위 안의 정확한 객체만 읽고 실제 읽은 버전을 고정한다. GitHub의 `CURRENT_COMMIT_ONLY`에서는 현재 승인 브랜치의 커밋을 해석해 그 커밋의 blob을 읽는다. 과거 blob SHA를 알고 있다는 이유로 과거 이력을 읽지 않는다. AWS 본문은 별도 값 읽기 승인·크기·버전/ETag 조건을 그대로 적용한다.
3. 재취득한 바이트를 등록된 정규화 규칙으로 다시 해석한다. 발급자·유형·구성요소 역할을 포함한 같은 정규형과 **같은 HMAC 키 버전**으로 계산한 fingerprint가 요청한 키와 일치해야 한다. 안전한 내부 키 회전 매핑이 있을 때만 그 매핑을 사용한다.
4. 일치한 원문만 검증기에 전달하고 실제 원천 버전·Finding을 시도 근거로 연결한다. 다른 토큰으로 바뀌었거나 대응되는 HMAC 키/회전 근거가 없으면 호출하지 않는다. 접두사·마스킹 suffix·같은 줄·같은 사용자만으로 동일성을 인정하지 않는다.

같은 파일에 다른 키가 생겼을 때 그 키의 응답을 이전 키에 붙이지 않는다. 새 후보는 별도 탐지·동일성 계약을 따른다. 원래 키를 다시 얻지 못한 사실은 `INVALID`가 아니다. 사전 검사에서 재취득 참조 자체가 없으면 `UNCHECKED/SOURCE_VALUE_UNAVAILABLE`, 승인된 재취득을 시도했으나 삭제·접근 실패·값 불일치로 얻지 못하면 `UNKNOWN/SOURCE_UNAVAILABLE` 또는 `UNKNOWN/CREDENTIAL_MISMATCH`다. 이 결과만으로 위치 `MISSING`이나 대응 완료를 만들지 않는다.

발급자와 수집 플랫폼은 독립적으로 선택한다. GitHub 파일 또는 승인된 AWS 본문에서 발견한 Notion 후보의 재검증은 해당 원천의 현재 재취득 경로와 Notion 검증기를 결합한다. Notion 문서에서 발견한 AWS·GitHub 후보도 각 발급자 검증기로 전달한다. Notion 원천은 승인된 현재 페이지·블록의 버전과 비교 가능한 위치 근거를 재취득한다. 수집 capability와 검증기 등록은 각각 검사한다.

원문·전체 공급자 응답·인증/응답 헤더·예외 메시지는 DB, 큐, 임시 파일, 로그, trace, APM, 오류 응답, 화면, CSV, 백업, AI 입력에 넣지 않는다. 응답은 크기 제한을 둔 메모리 파서에서 필요한 필드만 확인하고 버린다. HTTP wire logging·SDK wire logging·클라이언트 자동 request/response 캡처를 끈다. 자체 소유 byte 배열은 완료·오류·취소 경로에서 덮어쓰며 Java 불변 문자열·SDK 복사본까지 즉시 소거할 수 있다는 보장은 하지 않는다. 워커의 dump·프로세스·메모리 접근을 제한한다.

## 4. 합성 fixture와 TEST 출처

저장소·조직·폴더 이름과 표시용 자료 분류는 공급자 호출을 일괄 차단하는 기준으로 사용하지 않는다. `test-find-key`를 포함한 각 승인 원천은 같은 연결·발급 서비스·범위 검증 정책을 적용한다. 과거의 `SOURCE_GUARD_BLOCKED`는 당시 설정에 따른 미실행 이력으로 표시한다.

승인된 합성 fixture 원천과 유효한 manifest·보호된 일치 근거로 합성이 확인된 후보는 공급자 활성화 여부와 관계없이 `UNCHECKED/SYNTHETIC_FIXTURE`로 강제 생략한다. fixture guard는 수집 직후와 호출 직전에 적용하고 사용자 요청·재시도·관리자 옵션으로 우회하지 않는다. 승인된 fixture 원천은 일반 운영 범위로 자동 전환하지 않으며 guard를 해제하려면 명시적인 새 승인 revision이 필요하다.

`origin=TEST` 표식만으로 합성을 인정하지 않는다. 합성 확인이 없는 완전한 TEST 후보는 현재 검증 승인·지원 조건에 따라 처리하고 시험 실키 의심 경고와 조사 계약을 유지한다. 유효 manifest·scope revision·키 동일성·attestation이 맞는지는 [데이터 계약](data-contract.md)의 합성 근거 규칙을 따른다. manifest의 만료·취소·상충 근거는 현재 합성 인정과 capability를 다시 계산하게 한다.

fixture 생략은 **그 원천·발견 근거**에 묶는다. 같은 fingerprint를 다른 승인 운영 원천에서 발견했다고 해서 fixture 생략 결과를 재사용하지 않는다. 반대로 실제 키 검증 결과가 존재하더라도 fixture 원천에서 검증했다고 표시하지 않는다. 합성 여부가 충돌하면 자동으로 조용히 병합하거나 합성 면제를 확장하지 않고 조사 사유를 남긴다.

한 실행의 검증 중복 억제는 같은 범위 안의 같은 키에 적용한다. 앞선 범위의 합성 판정·발급 서비스 미허용·승인 부재가 다른 적격 범위의 검증을 막지 않게 한다. 실행 전체의 공급자 호출 예산과 저장 예약의 중복 방지는 계속 적용한다.

## 5. 결과 분류

`UNCHECKED`는 사전 조건 때문에 검증을 시작하지 않은 결과다. `UNKNOWN`은 승인 재취득 또는 공급자 요청을 실제 시도했으나 확정할 수 없는 결과다. `VALID`·`INVALID`에는 공급자가 응답한 관측 시각이 필요하다. `UNKNOWN`에는 timeout·전송 실패처럼 공급자 관측 시각이 없는 경우가 있으므로 완료 시각을 공급자 관측 시각으로 복사하지 않는다.

### GitHub PAT

| 관측 | 결과·안전한 reason | 규칙 |
|---|---|---|
| 200, 제한 내 JSON의 인증 사용자 `id`와 `login` 구조 확인 | `VALID/AUTHENTICATED` | 인증 주체가 확인된 사실만 기록. 사용자 전체 응답·원문 ID·login을 일반 검증 이력에 저장하지 않음 |
| 승인 TLS endpoint의 401, 원문 PAT 헤더가 정상 구성됐고 등록된 파서가 명확한 `Bad credentials` 인증 거부로 확인 | `INVALID/AUTHENTICATION_REJECTED` | 만료·폐기·소유자 변경 원인을 추정하지 않음. 공급자 원문 메시지는 저장하지 않음 |
| 401이지만 헤더 구성 오류·예상 밖 응답·client 문제를 배제할 수 없음 | `UNKNOWN/CLIENT_FAILURE` 또는 `UNKNOWN/RESPONSE_UNEXPECTED` | 모든 401을 자동 무효로 취급하지 않음 |
| 403·429의 승인된 rate-limit 신호 | `UNKNOWN/RATE_LIMITED` | `Retry-After`·reset 시각은 메모리에서 검증해 안전한 다음 시도 시각만 남김 |
| 그 밖의 403, 404 | `UNKNOWN/ACCESS_AMBIGUOUS` | 접근 제한·일시 인증 차단·잘못된 endpoint 등을 토큰 무효로 단정하지 않음 |
| timeout·DNS/TLS/연결 실패·5xx | `UNKNOWN/PROVIDER_UNAVAILABLE` | 인증 거부나 만료로 해석하지 않음 |
| 3xx, 과대 응답, malformed JSON, 200인데 주체 필드 누락 | `UNKNOWN/RESPONSE_UNEXPECTED` | redirect하지 않으며 200만으로 인증 성공을 만들지 않음 |
| 비활성·미지원·불완전·합성·현재 승인 없음·호출 예산 미확보 | `UNCHECKED`와 `DISABLED`·`UNSUPPORTED_TYPE`·`INCOMPLETE_CREDENTIAL`·`SYNTHETIC_FIXTURE`·`NOT_APPROVED`·`RATE_BUDGET_EXHAUSTED` 중 실제 사유 | 공급자 호출 없음 |

GitHub는 잘못된 자격 증명에 처음 401을 반환한 뒤 반복 실패 시 일시적으로 403을 반환할 수 있다. 따라서 명확한 인증 거부 후 즉시 재시도하지 않고 403을 폐기 근거로 승격하지 않는다. [GitHub REST 인증 문서](https://docs.github.com/en/rest/authentication/authenticating-to-the-rest-api)의 구분을 적용한다. SSO·조직 승인·특정 저장소 permission의 확인 여부는 `/user` 결과와 별개다.

### AWS STS

기존 분류를 유지한다. 성공은 `VALID/AUTHENTICATED`, `InvalidClientTokenId`는 `INVALID/INVALID_ACCESS_KEY`, 명시적 `ExpiredToken`은 `INVALID/EXPIRED_SESSION`이다. `SignatureDoesNotMatch`는 서명·환경 원인도 가능하므로 `UNKNOWN/SIGNATURE_AMBIGUOUS`다. `Throttling` 계열·429는 `UNKNOWN/RATE_LIMITED`, 다른 401·403·404는 `UNKNOWN/ACCESS_AMBIGUOUS`, 공급자/네트워크/client 실패는 대응하는 `UNKNOWN` 사유다. AWS의 명시적인 만료 근거를 GitHub의 일반 401에 적용하지 않는다.

### Notion 콘텐츠 API 토큰

| 관측 | 결과·안전한 reason | 기록할 범위 |
|---|---|---|
| 200, 제한 내 JSON의 `object=user`, 지원되는 `type=bot\|person`, 유효한 주체 UUID | `VALID/AUTHENTICATED` | 현재 인증 관측. 선택 주체 필드의 누락·지원 밖 값은 해당 필드의 미확인으로 기록 |
| 401, 정상 오류 JSON의 `code=unauthorized` | `INVALID/AUTHENTICATION_REJECTED` | 현재 인증 거부. 만료·폐기·변조·관리 정책 변경의 세부 원인은 별도 근거로 관리 |
| 그 밖의 401, 200 주체 형식 불일치·잘린/과대/비정상 JSON·3xx | `UNKNOWN/UNEXPECTED_RESPONSE` | 등록 응답 구조 확인 실패. redirect는 따르지 않음 |
| 403·404 | `UNKNOWN/ACCESS_LIMITED` | endpoint 권한·자료 접근 제한·요청 경계의 불확실성 |
| 429, `rate_limit_reason=public_api_request_blocked` | `UNKNOWN/PROVIDER_ACCESS_BLOCKED` | 운영 확인이 필요한 공급자 제한. 자동 재시도에서 제외 |
| 그 밖의 429 | `UNKNOWN/RATE_LIMITED` | 검증된 `Retry-After`와 제한 단위에 따른 대기 |
| 529·그 밖의 5xx·timeout·연결/TLS 실패 | `UNKNOWN/PROVIDER_UNAVAILABLE` | 허용 횟수·전체 실행 기한 안에서 재시도 |
| 요청 구성·검증기 내부 오류 | `UNKNOWN/CLIENT_FAILURE` | 검사 설정·처리 오류를 안전한 사유로 보존 |
| 비활성·미지원·불완전·합성·미승인 | `UNCHECKED`와 실제 사전 차단 사유 | 공통 미실행·원문 보호 계약 적용 |

기능 권한에 따라 `/users/me`의 사용자 정보가 제한될 수 있다. 정상 User 구조를 확인한 뒤 선택 메타데이터가 부족하면 인증 성공과 해당 필드의 미확인을 함께 보존한다. 봇·owner·workspace 정보에서 전체 capability·접근 가능한 모든 페이지·토큰 만료일을 추정하지 않는다. `bot.owner.type=workspace`는 소유 범위 근거이며 내부 연결·PAT·OAuth 세부 유형 확정에는 발급 근거를 사용한다. PAT 정책 변경도 인증 거부의 원인이 될 수 있다. [인증 응답](https://developers.notion.com/reference/get-self), [오류 코드](https://developers.notion.com/reference/status-codes)

페이지 정보·본문 읽기는 등록된 대상에 대한 `ENRICH/PERMISSION_ASSISTANCE`의 별도 관측이다. 기본 인증과 다른 요청 목적·승인 대상·결과·관측 시각을 갖는다. OAuth introspection은 정확한 발급 앱과 관리 연결이 확인된 별도 capability로 등록하며, 응답의 active/scope/iat와 토큰 만료 자료를 구분한다. refresh·revoke는 외부 조치 절차에서 다룬다.

reason은 검증기별 승인된 코드 집합이며 `result`와 일치해야 한다. UI는 코드에서 고정된 한국어 설명을 선택하고 공급자 자유문을 그대로 표시하지 않는다. 예를 들어 GitHub의 `AUTHENTICATION_REJECTED`는 ‘인증이 거절됨 · 만료/폐기 원인 미확인’으로 표시한다. 범위가 다른 오류를 하나의 ‘키 무효’ 문구로 합치지 않는다.

## 6. 기존 작업·주기·중복·lease

새 provider별 스케줄러나 검증 전용 작업 종류를 만들지 않는다. 기존 `secret_jobs`·`secret_scan_runs`·`secret_run_scopes`를 사용한다.

| 진입 | 실행 계약 |
|---|---|
| 정기·수동 `FULL_SCAN` | 승인 원천 수집→탐지→정규화→안전 저장→해당 후보 검증. 정기는 기본 6시간이며 다음 예정은 해당 정기 작업의 실제 예약 시각+6시간으로 계산. 중단 후 놓친 슬롯은 한 번만 예약하고 연속 재생하지 않음 |
| `REVALIDATE`, `purpose=CREDENTIAL_VALIDITY` | 키·지정 위치·기대 revision을 가진 유효성 재검증. 현재 원천 재취득과 HMAC 일치 후 인증만 확인 |
| `REVALIDATE`, `purpose=LOCATION_RECHECK` | 기존 대응 위치 재검사. 인증 검증과 위치 부재·비교 가능성 결과를 분리하며 한쪽 성공으로 다른 쪽을 완료하지 않음 |

별도 Java 워커의 도래 작업 확인 간격은 기본 30초다. 30초는 수집·검증 주기나 완료 SLA가 아니다. 최초 발견은 해당 전체 검사 안에서 첫 검증을 준비하며 별도 작업이 필요해도 30초 polling·lease 계약을 따른다. 수동 요청·일시 오류·최근 성공으로 6시간 정기 예정 시각을 뒤로 밀지 않는다.

중복 키는 `(CredentialSet, credentialRevision, 동일성 HMAC 키 버전, 검증기 버전, 검증 승인 revision, 검증 주기/목적)`으로 묶는다. 통상 주기는 실제 정기 예약으로 시작한 6시간 구간이며 같은 구간의 중복 위치·원천은 실제 인증 요청 하나의 근거에 연결할 수 있다. 연결 가능한 각 위치에는 현재 허용된 동일 키·발견 근거가 있어야 한다. fixture·비활성의 생략 결과를 실제 요청의 중복 제거에 사용하지 않는다. 조치 이후 재검증은 조치 ID/버전과 목적을 별도 주기 키로 사용해 조치 전 결과를 재사용하지 않는다. 같은 요청자·명령·멱등 키의 같은 본문은 완료 후에도 원래 요청·결과를 `reused`로 반환한다. 다른 멱등 키로 접수한 새 수동 명령만 새 요청 epoch를 사용하며 호출 예산·공급자 backoff를 통과해야 한다. 현재 진행 중인 같은 대상·목적 요청을 재사용할 때도 원래 요청 참조와 `reused`를 반환한다. 재사용한 과거 응답을 새 관측으로 표시하거나 관측 시각을 갱신하지 않는다.

외부 호출 전 기존 `secret_validation_attempts`에 안전한 예약을 원자적으로 확보한다. 같은 중복 키의 활성 예약은 하나이며 다른 워커는 완료된 유효 결과를 재사용하거나 대기한다. `secret_jobs`의 현재 lease token과 검증 예약 세대를 함께 확인한다. heartbeat·결과 저장은 lease가 살아 있을 때만 허용하고 만료된 워커는 요청을 새로 보내거나 결과를 현재 사실로 저장하지 않는다. 새 워커의 lease가 이전 워커를 fence한다.

전송 직후 프로세스가 죽으면 공급자 요청의 실행 여부를 완전히 알 수 없다. 따라서 외부 요청의 exactly-once를 약속하지 않으며 lease 복구 후에도 재시도 횟수·호출 예산·같은 키의 다음 허용 시각을 적용한다. HTTP 전송과 DB의 `request_sent_at` 갱신도 원자적이지 않다. `request_sent_at`은 전송을 시작했다고 로컬 transport가 확인한 시각이고, NULL은 crash 시 ‘확실히 미전송’의 증명이 아니다. 예약·전송 가능성을 포함한 감사 근거를 보존하고 무한 반복 호출을 금지한다.

같은 연결의 외부 호출을 직렬화하는 활성 작업 unique 제약은 `FULL_SCAN`·`REVALIDATE`에만 적용한다. 외부 호출 없는 `ASSESS`는 이 제약의 대상에서 제외하고 위치·근거 revision·정책별 멱등 및 CAS를 적용한다. 동일 대상·목적·요청 의미의 중복은 멱등 재사용하고 다른 활성 외부 호출 작업과 충돌하면 안전한 `ACTIVE_JOB_EXISTS`를 반환한다. 다른 작업의 대상·수행자·시각을 열람 권한 없이 내보내지 않는다.

## 7. 승인 취소와 TOCTOU

작업 등록 때의 승인·capability는 실행 권한을 영구 부여하지 않는다. 작업은 scope revision, 위치 기대 버전, 키의 `credentialRevision`, 능동 검증 승인 revision, 검증기 버전과 요청 주체의 권한 근거를 고정한다. 요청 주체의 권한이 취소되면 그 사용자가 요청한 대기 수동 작업을 실행하지 않는다. 정기 작업은 현재 서비스 주체와 범위 승인으로 검사한다.

다음 시점마다 현재 상태를 확인한다.

1. 작업 선점과 원천 읽기 직전: 연결 활성, 현재 scope revision·수집 권한·요청 대상 소속을 확인한다.
2. 외부 요청 직전: 짧은 트랜잭션에서 live lease, 현재 검증 승인 revision, fixture guard, 동일성, 한도와 활성 예약을 확인하고 최대 1초의 호출 허가를 발급한다. 허가가 만료되면 재검사한다. 원문은 허가에 넣지 않는다.
3. 완료 저장 직전: 같은 lease·예약 세대·scope/검증 승인 revision을 CAS로 확인한다. 현재 근거로 사용할 수 있는지를 함께 확정한다.

승인 취소·범위 변경·연결 중지는 새 허가와 새 요청을 즉시 차단하고 대기·미전송 작업을 취소한다. 이미 전송된 요청은 회수할 수 없으므로 transport 취소는 최선의 조치이며 ‘취소 직후 공급자가 받는 요청이 0건’이라고 보장하지 않는다. 취소와 응답이 경합한 경우 공급자 관측의 원래 결과는 추가 전용 이력으로 보존하되 `currentEligible=false`와 `APPROVAL_CHANGED` 등 안전한 사유를 붙인다. 현재 판정·freshness·마지막 인증 성공·영향 평가·해결 근거에 승격하지 않는다. lease를 잃은 워커는 완료를 쓰지 못하며 예약은 복구 주체가 `ABANDONED`로 종료한다.

한 번의 HTTP 성공을 승인 revision 변경 이후의 새 요청 허가로 재사용하지 않는다. 범위 축소나 사용자 권한 취소 이후의 조회도 현재 ACL로 다시 필터하며 과거 키 ID를 안다는 이유로 원천·다른 위치의 활동을 공개하지 않는다.

## 8. 공통 검증 이력과 읽기 투영

V5의 `secret_validation_attempts`를 예약 상태와 미완료 NULL 제약을 허용하도록 확장해 모든 발급자의 공통 이력으로 일반화한다. GitHub 전용 시도 테이블이나 별도 최신 결과 저장소를 만들지 않는다. 기존 run/job/credential FK, validator ID·버전, endpoint class, result·reason과 시각을 유지하고 다음 의미를 저장한다. 물리 컬럼·제약의 기준은 [DB 스키마](database-schema.md)다.

키 동일성 revision의 원본은 `secret_credential_sets.credential_revision BIGINT NOT NULL DEFAULT 1`이다. 계약의 `credentialRevision`은 이 컬럼을 뜻하며 HMAC 보호키 버전과 다르다. 구성 승격·동일성 변경·승인된 동일성 전환 때 원자적으로 증가시키고, 요청 context·작업의 기대 snapshot·검증 예약/완료의 동일성 epoch는 모두 이 값을 사용한다. 스냅샷 값이 현재 원본과 다르면 과거 원문·응답을 현재 키 검증으로 승격하지 않는다.

| 저장 의미 | 필수 규칙 |
|---|---|
| 예약 상태 | `attempt_state=RESERVED\|SENT\|COMPLETED\|ABANDONED`. 이는 작업 수명이며 네 가지 검증 결과와 다름. 활성 중복 키에 하나의 RESERVED/SENT 예약만 허용 |
| 시각 | `requested_at`은 예약 시각, `request_sent_at`은 로컬 전송 시작 확인, `completed_at`은 종료 시각. `provider_checked_at`은 실제 공급자 응답 관측 시각이며 timeout·재취득 실패에는 NULL |
| NULL·결과 제약 | RESERVED/SENT는 `result`, `reason_code`, `completed_at`이 NULL. RESERVED의 `request_sent_at`은 NULL, SENT는 전송 시각 필수. COMPLETED는 네 결과 중 하나와 reason·완료 시각 필수. VALID/INVALID는 `request_sent_at`·`provider_checked_at` 필수, UNCHECKED는 두 시각 모두 NULL. ABANDONED는 결과가 NULL이며 안전한 종료 사유·종료 시각을 기록. 확인한 시각은 요청 이상·완료 이하이고 공급자 관측은 전송 이후여야 함 |
| 동일성·중복 근거 | credential ID·원본의 `credential_revision`, 유형에 맞는 HMAC 키 버전, 검증 주기/목적·재시도 번호, 활성 예약 세대. HMAC 값·원문은 일반 API에 노출하지 않음 |
| 실행·승인 근거 | run/job, 현재 lease의 예약 세대, scope·scope revision, 검증 승인 revision, 실제 원천 버전의 보호된 근거, 참조한 occurrence/Finding의 내부 ID. 같은 키·실행·범위 소속을 저장 트랜잭션에서 확인 |
| 현재 사용 적격성 | `current_eligible`와 부적격 사유. 승인 취소·revision 변경·lease 상실 결과를 현재 판정으로 승격하지 않음. 과거 시도·근거는 덮어쓰지 않음 |
| 제한된 응답 메타데이터 | 시도 원장에 정수 HTTP status, 정규화 reason, `next_due_at`을 저장. VALID에 결합한 주체 정보는 [DB 스키마](database-schema.md#9-notion-주체와-근거-저장)의 공급자별 닫힌 schema로 저장. response body·헤더·예외 원문·사용자 프로필 저장 금지 |

검증기 ID·유형·endpoint class·result/reason의 허용 조합은 등록부로 검증한다. 예를 들어 PAT에 AWS method를 쓰거나 `CANDIDATE`에 `VALID`를 저장할 수 없다. COMPLETED의 실제 응답은 재시도로 덮지 않고 새 attempt로 남긴다. 진행 중·포기한 예약을 완료된 최신 결과로 조회하지 않는다.

profile 미설정·미등록으로 사전 차단한 COMPLETED/UNCHECKED에는 profile NULL을 허용하며 실제 전송에는 등록 profile을 요구한다. 미등록 유형의 안전한 비호출 분류자는 `UNREGISTERED/registry-v1`, endpoint class는 `NONE`이다. 가짜 검증기·profile로 지원 여부를 채우지 않는다. 연결·범위의 검증 설정 및 예약·NULL CHECK의 물리 원본은 [DB 스키마 8절](database-schema.md#8-검증-일반화와-평가-revision의-물리-제약)이다.

읽기 응답은 현재 ACL로 볼 수 있고 `currentEligible`인 근거를 대상으로 다음을 구분한다. 이 적격성은 저장된 허용 표식만으로 결정하지 않고 현재 키 동일성·scope/검증 승인 revision·ACL을 매번 결합한다. 완료 후 승인 취소도 즉시 현재 투영에서 제외하며 과거 이력만 허용된 범위에서 보존한다.

| 읽기 정보 | 계산 |
|---|---|
| `latestRequest` | 현재 요청·작업의 대기/실행·예정·중단 사실. 아직 결과가 없음을 별도로 표시 |
| `latestAttempt` | `attempt_state=COMPLETED`인 최근 적격 시도의 네 결과·reason·요청/완료/공급자 관측 시각 |
| `lastAuthenticatedAt` | 적격한 `VALID/AUTHENTICATED`의 마지막 공급자 관측 시각. INVALID·UNKNOWN·UNCHECKED는 갱신하지 않음 |
| `lastConclusiveValidation` | 마지막 적격 VALID/INVALID와 관측 시각. 인증 성공과 인증 거부를 같은 ‘마지막 성공’으로 표시하지 않음 |
| `nextDueAt`, `freshness` | 승인된 유효성 재확인 주기로 계산. 기본 6시간. `CURRENT\|STALE\|POST_ACTION_REQUIRED\|UNAVAILABLE`의 파생 표시이며 공급자 토큰 만료시각이 아님 |
| `postActionValidationPending` | 현재 조치 이후의 적격·확정 관측이 없는지. 과거 VALID의 시각과 조치 후 대기를 함께 표시 |

기존 `lastSuccessfulValidationAt`을 제공하는 호환 응답은 마지막 **인증 성공**이라는 의미로 한정하고 `lastAuthenticatedAt`과 같은 시각을 반환한다. 최신 UNKNOWN·UNCHECKED를 숨기고 과거 VALID만 현재 결과로 표시하지 않는다. 최근 시도가 미확정이면 ‘최근 확인 불가’와 이전 인증 성공 시각을 함께 보여준다. 기한이 지났거나 조치 이후라면 과거 VALID에 현재 확정 표현을 붙이지 않는다. 재취득만 성공하거나 probe가 성공해도 인증 신선도를 갱신하지 않는다.

새 승인 revision에는 이전 revision의 관측을 현재 근거로 자동 승격하지 않는다. `freshness`는 최신 실패와 독립된 마지막 확정 공급자 관측·다음 예정 시각·현재 승인에서 계산하되 최신 UNKNOWN/UNCHECKED와 차단 사유를 함께 반환한다. 수집 freshness와 유효성 freshness는 별개이며 하나의 성공 시각으로 합치지 않는다.

`freshness`의 결정 순서는 현재 승인·동일성·관측이 없어 사용할 수 없으면 `UNAVAILABLE`, 마지막 확정 관측 뒤 실질 조치가 있으면 `POST_ACTION_REQUIRED`, 마지막 확정 관측보다 새로운 UNKNOWN/UNCHECKED가 있으면 `UNAVAILABLE`, 확인 기한을 넘었으면 `STALE`, 나머지는 `CURRENT`다. 관측을 사용할 수 없는 이유는 별도 코드로 제공한다. 진행 중 요청만으로 기존 확정 관측을 실패로 바꾸지는 않는다. 확인 기한은 마지막 확정 공급자 관측+승인 주기이며 실패·접수로 연장하지 않는다.

## 9. API·capability 계약

발견 키 유효성 조회·요청·진행 조회는 [API 계약](api-contract.md)의 기존 키 리소스와 검증 요청 계약을 사용한다. API는 원문을 받거나 직접 외부 호출하지 않는다. 요청 본문은 `{occurrenceId,expectedContextVersion,idempotencyKey}`이며 `expectedContextVersion`은 키의 `credentialRevision`·위치·scope/검증 승인·조치 revision을 묶은 서버 발행 불투명 버전이다. 서버는 검증된 기대 snapshot으로 `REVALIDATE/CREDENTIAL_VALIDITY` 작업을 만든다. 접수는 202와 request/job·run 참조이며 검증 성공이 아니다.

`capabilities.canRequestValidation`은 등록 지원, 유형·구성, 현재 대상 ACL, 현재 범위/검증 승인, 기능 활성화, fixture guard, 재취득 가능 경로, 활성 작업·한도를 모두 고려한다. false일 때는 안전한 `capabilities.blockedReasonCodes`와 다음 행동을 반환한다. 이 사전 capability는 실행 보장이 아니며 등록·실행 때 현재 조건을 재검사한다. SUPPORTED와 ENABLED, APPROVED, 지금 실행 가능을 하나의 boolean으로 합치지 않는다.

요청자는 해당 범위 SECURITY 또는 해당 위치에 할당된 REMEDIATOR여야 한다. OPERATIONS의 연결 관리 권한만으로 발견 키 인증 호출을 요청할 수 없다. 연결의 `canTest`와 키의 `canRequestValidation`은 별개다. 권한 밖 키는 존재·최근 시도·다른 연결의 작업을 누출하지 않는 오류를 반환한다.

응답은 네 결과·고정 reason 설명, 검증기/방법·버전, `latestRequest`, `latestAttempt`, 마지막 인증 성공·마지막 확정 관측, 다음 예정·freshness, 조치 후 대기·차단 사유, 근거가 적용되는 범위를 제공한다. 원문·HMAC·전체 endpoint URL·원천 참조·프로필·공급자 자유문은 반환하지 않는다. 조회 실패·권한 제한을 `INVALID` 또는 위치 부재로 바꾸지 않는다. 별도 유효성 요청이 위치 부재·건 해결 검사도 수행했다고 표현하지 않는다.

## 10. 운영 한도와 실패 처리

능동 검증은 한도가 설정·검증되지 않으면 시작하지 않는다. 아래 값은 초기 승인 기본값이며 변경은 버전이 있는 운영 설정과 감사에 남긴다. 더 큰 한도가 필요해도 무제한 설정은 허용하지 않는다.

| 항목 | 기본값·상한 규칙 |
|---|---|
| 동시 호출 | 워커당 최대 2개. 동일 공급자·egress에 공유된 limiter를 적용해 워커 증설로 예산이 늘지 않음 |
| 호출량 | 공급자별 조직당 분당 30회, 공유 egress당 분당 60회. 하나의 키에는 통상 6시간 주기당 최초 1회와 허용된 일시 오류 재시도만 허용 |
| 실행 예산 | 한 실행에서 실제 공급자 요청 최대 500회, 검증 단계 경과 시간 최대 30분. 먼저 도달한 한도에서 새 전송을 멈추고 나머지는 미검증 이유·다음 예정으로 보존 |
| HTTP | 연결 timeout 3초, 단일 요청 전체 deadline 8초, 응답 본문 최대 64 KiB. AWS SDK 내부 재시도를 포함한 전체 deadline/횟수로 계산 |
| 일시 오류 재시도 | 최초 포함 최대 3회. 30초부터 지수 backoff, 최대 15분, jitter 적용. `Retry-After`/reset이 있으면 더 늦은 시각 적용. 30분 실행 예산 밖이면 기존 job의 도래 시각으로 지연 |
| 인증 거부·영구 사전 차단 | INVALID와 합성·미지원·불완전·미승인은 즉시 자동 반복하지 않음. 새 승인·새 구성·정기 재확인·명시적 적격 요청 전까지 대기 |
| 연속 인증 거부 | 같은 GitHub egress에서 60초 내 명확한 거부 5회면 신규 PAT 요청을 15분 중지하고 안전한 경보·지연 사유 기록. 중지 중 INVALID를 추정하지 않음 |

provider rate limit이 더 낮으면 그 제한이 우선이다. timeout·전송 실패의 자동 재시도도 호출 예산을 소비한 것으로 계산한다. retry 대기와 예약을 `Thread.sleep`으로 장시간 점유하지 않고 기존 job의 `available_at`·`max_attempts`·시도 이력·`next_due_at`에 영속화한다. 미검증 대상은 내부 ID·기대 revision만 유지하며 재실행 때 원문을 다시 취득한다.

Notion의 전송은 상태·검증된 대기 헤더를 본문보다 먼저 처리하고 내부 자동 재전송을 끈다. 8초 전체 deadline과 64 KiB 상한은 헤더 이후의 본문 정지에도 적용한다. 최초 포함 최대 3회·30분 실행 기한을 적용하고, 더 긴 공급자 대기는 공유 상태에 보존한 채 해당 요청을 확인 불가로 종료한다. 다음 요청도 남은 공급자 대기를 적용한다. 대기의 최솟값은 검증된 `Retry-After`이며 이를 로컬 backoff 상한으로 줄이지 않는다.

Notion의 공유 대기는 `NOTION_SELF` endpoint의 공통 cooldown에 검증된 가장 늦은 `Retry-After`를 보존한다. 키·작업·워커가 바뀌어도 이 기한을 적용해 workspace 제한 우회를 차단한다. 같은 workspace·연결을 정확히 결합하는 근거가 있는 세분화 limiter는 공통 대기 이상의 제한으로 적용한다. `public_api_request_blocked`는 운영 확인 필요 사유로 종료하고 운영 해제·새 검증 승인에 따른 적격 요청에서 재평가한다. [Notion 요청 제한](https://developers.notion.com/reference/request-limits)

검증 단계는 실제 호출의 VALID/INVALID를 처리 성공으로 보고 UNKNOWN·필수 검증 누락은 부분 실패로 남긴다. 선택적으로 비활성인 검증·합성 fixture 생략·메타데이터의 해당 없음은 이유를 기록한 SKIPPED이며 수집 실패를 만들지 않는다. 검증 budget 소진은 이미 성공한 수집·탐지를 실패나 정상 0건으로 덮지 않고 검증 누락을 표시한다. 실행 완료 상태는 결과의 유효성 값과 별개다.

운영 지표는 원문 없는 공급자·검증기 버전별 대기량, 예약/전송/완료/포기 수, 인증 거부·미확정 비율, timeout, rate-limit 대기, lease 복구, 승인 취소, 재취득 실패, 동일성 불일치, fixture 생략, 예산 소진과 freshness 지연이다. 키·사용자·원천 경로를 metric label에 넣지 않는다. 경보·오류도 reason code와 내부 실행 ID만 사용한다.

## 11. 수용 시험

기본 자동 시험은 provider client stub·loopback fake·합성 sentinel로 수행한다. 운영 키·원문 파일을 시험 입력으로 요구하지 않는다. 실제 공급자 계약 확인은 별도 승인된 시험 계정과 원천·호출 예산에서만 허용하며 fixture는 실제 endpoint로 보내지 않는다.

| 시험 | 수용 조건 |
|---|---|
| PAT 두 유형의 인증 성공 | Classic/Fine-grained 각각 발견 원문으로 정확히 GET `/user` 한 번. 200 주체 파싱 후 VALID. 수집 App·설치 토큰은 요청 인증에 사용되지 않음 |
| GitHub 거부·비정상 응답 | 명확한 401만 INVALID. 헤더 오류 401·403·404·429·timeout·5xx·redirect·malformed/과대 응답은 정의된 UNKNOWN. 원인 불명 401로 만료·폐기 증거 생성 불가 |
| AWS 회귀 | 기본 비활성·불완전 쌍·ASIA session 누락은 미호출. 명시적 발견 쌍으로 STS 호출. 기존 명시 오류·서명 모호성·rate limit 분류 유지, 기본 profile fallback 없음 |
| Notion 인증 분류 | 내부 연결·PAT·OAuth access의 등록 후보에서 200 정상 주체·401 unauthorized·403/404·잘린 200·잘못된 JSON을 구분. 선택 owner/workspace 누락은 인증 성공과 별도 미확인으로 보존 |
| Notion 주체와 동일성 | 같은 봇·workspace의 서로 다른 키는 별도 CredentialSet. 유형 보완이 기존 HMAC 동일성을 조용히 변경하지 않음. 과거 200→사용자 폐기 신고→현재 401은 세 기록으로 보존 |
| Notion 원천과 발급자 | GitHub/AWS 원천의 Notion 키는 현재 원천 재취득·HMAC 일치 후 Notion 인증기로 전달. Notion 원천의 AWS/GitHub 키는 해당 발급자로 전달. 새 키로 교체된 원본은 이전 키의 인증 결과를 생성하지 않음 |
| Notion 대기와 영구 제한 | 연결·workspace·endpoint별 429, 529, 긴 Retry-After, 헤더 이후 본문 정지, 재시도 소진을 시험. 새 키·작업의 대기 우회와 내부 재전송을 차단하고 public_api_request_blocked는 운영 확인 필요로 종료 |
| 원문 수명 | 최초 콜백의 동일 바이트를 검증기에 전달. 재시작·재시도는 내부 ID에서 승인 원천 재취득. 원문이 큐·파일·DB에 복원 경로로 남지 않음 |
| 동일성·현재 원천 | 같은 suffix의 다른 PAT, 원천 값 변경, 다른 HMAC 버전·회전 근거 없음, 현재 범위 밖 과거 blob은 미호출. 같은 HMAC으로 확인한 원문만 해당 키에 연결 |
| 합성·출처 | fixture guard는 활성화·수동 요청·재시도에서도 실제 호출 0회. TEST 표식만으로 fixture 생략 불가. fixture 생략이 다른 운영 원천의 실제 검증을 중복 제거하지 않음 |
| 중복·작업 | 여러 위치·동시 워커가 같은 주기 키를 처리해도 활성 예약 하나. 같은 목적 요청은 멱등, 다른 활성 작업은 안전한 409. 수동 요청이 정기 6시간 예정 시각을 바꾸지 않음 |
| lease·crash | 만료 워커의 추가 전송/저장 차단. 전송 후 crash 복구에서 포기 이력·세대·예산 보존. 한도 안의 중복 가능성을 기록하며 무한 재호출 없음 |
| 승인·TOCTOU | 등록 후 권한 취소·scope/검증 승인 revision 변경·fixture 변경을 읽기/전송/완료 경계마다 차단. 전송 중 취소의 응답은 이력에만 남고 currentEligible=false. 새 승인에 옛 성공 자동 승격 불가 |
| 시간·투영 | timeout/재취득 실패의 provider_checked_at=NULL. 최근 UNKNOWN/UNCHECKED·진행 요청과 이전 VALID 시각을 함께 표시. INVALID는 lastAuthenticatedAt을 갱신하지 않음. 조치 이후 검사 전 과거 VALID를 현재 확정으로 표시하지 않음 |
| 한도·장애 | 분당/동시/실행 예산·8초/64 KiB·3회 상한·Retry-After·인증거부 circuit break를 결정적 시계로 확인. 예산 없는 설정에서 호출 0회. DB 장애를 검증 성공으로 처리하지 않음 |
| 권한·사실 분리 | probe 성공이 키 검증·수집 성공을 만들지 않음. 인증 결과로 위치·노출·권한·심각도·대응 해결이 변하지 않음. 권한 밖 근거·다른 작업 메타데이터 조회 불가 |
| 원문 모의 sentinel 누출 | PAT·AWS 구성요소·헤더·응답·예외에 서로 다른 sentinel을 넣고 정상/오류/timeout/취소/retry 경로 실행. 로그·trace/APM payload·DB·큐·임시 파일·API/CSV·백업 대상·객체 문자열에서 sentinel 0건. 발견 시 수용 실패 |

검증기 capability는 이 시험의 해당 유형·응답·보안 경계를 통과한 버전에서만 지원으로 등록한다. 합성 성공률을 운영 인증 성공률로 표현하지 않으며 활성화된 기능·현재 승인·관측 결과를 각각 표시한다.
