# 수집원과 검증기 계약

기준: [PRD](../01-requirements/PRD.md) FR-01부터 FR-09, [ADR-S001](adr/ADR-S001.md), [ADR-S002](adr/ADR-S002.md), [ADR-S006](adr/ADR-S006.md). 키 유형·구성요소 분류는 [자격 증명 유형](../04-reference/credential-types.md)을 따른다.

## 연결별 수집 경계

| 연결 | 수집 계약 | 독립적으로 승인·표시할 범위 |
|---|---|---|
| GitHub | 승인 저장소·브랜치의 현재 커밋 파일과 Actions 워크플로 설정 파일 | 현재 파일·과거 이력을 구분한다. 실행 로그·아티팩트·저장 비밀 값은 별도 수집 권한과 지원 계약 없이는 읽지 않는다. |
| Notion | 승인된 현재 페이지의 제목과 지원 텍스트 블록 | 수집 연결이 접근할 수 있는 자료와 승인 페이지 목록을 교차한다. 중첩 블록과 페이지 나눔을 처리하고, 하위 페이지·데이터베이스·외부 본문은 별도 대상으로 구분한다. |
| AWS | 승인 계정·리전·서비스·리소스별 메타데이터와 정확한 단일 자원 본문 | 목록 메타데이터·값·객체·로그는 독립 범위다. 계정 읽기 승인을 모든 콘텐츠 수집으로 확대하지 않는다. |
| K8s 확장 계약 | 승인 클러스터·네임스페이스의 내부 Secret 객체 | Kubernetes API·최소 RBAC를 사용한다. projected 토큰·외부 저장소·노드·etcd는 Secret 객체 범위에 포함하지 않는다. |

실행은 등록된 수집기 capability와 승인 콘텐츠 유형이 일치할 때만 허용한다. GitHub의 CURRENT_COMMIT_ONLY와 AWS의 명시적 메타데이터/단일 값 계약을 구별하고 Notion·K8s·과거 커밋 이력은 그에 맞는 별도 수집기가 등록되기 전까지 지원 완료로 표시하지 않는다. 연결마다 콘텐츠 유형·원천 객체 ID와 버전·읽은 범위·최초/최근 성공·부분 실패·거부·미지원 사유를 공통 결과로 반환한다. 승인 대상이 없는 정상 0건과 접근 실패의 0건을 구분한다. 발견 URL을 이유로 새 원천을 자동 추가하지 않는다. 수집기는 스케줄러나 대응 저장 모델을 독자적으로 정의하지 않는다.

GitHub는 승인 객체의 파일 바이트를 읽고 객체·커밋·줄 위치를 추적한다. Git 저장소를 워커 디스크에 복제하거나 탐지 도구에 연결 자격 증명을 전달하지 않는다. 현재 파일과 이력 검사는 별도 승인·지원 범위이며 한쪽의 성공을 다른 쪽 완료로 표시하지 않는다.

Notion 수집 연결은 콘텐츠 읽기 권한의 내부 연결을 사용한다. DB에는 고정된 불투명 참조 `auth_ref=notion-internal:default`만 저장하고, 원문은 외부 설정 `SECRET_NOTION_TOKEN_FILE`이 가리키는 파일에서 읽는다. 파일 경로·원문은 연결 응답이나 DB에 복사하지 않는다. `SECRET_NOTION_COLLECTION_ENABLED`는 기본 false이며 API와 워커가 같은 설정을 사용한다. 발견 키의 인증 활성화는 별도 `SECRET_NOTION_VALIDATION_ENABLED`를 따른다.

승인 원천은 `NOTION_PAGE_CONTENT`, `notion://pages/<page UUID>`, `CURRENT_VERSION_ONLY`로 등록한다. UUID는 소문자 하이픈 형식으로 정규화하며 임의 호스트·query·다른 URI 형식은 거부한다. 현재 페이지의 제목과 블록 목록을 읽고 지원 중첩 블록·페이지 나눔을 순회한다. `child_page`, `child_database`, `link_to_page`의 연결 대상과 첨부·외부 파일 본문은 따라가지 않는다. 하위 페이지는 별도 승인 페이지로 등록한다. 자식 페이지·데이터베이스의 제목과 첨부 블록의 캡션처럼 현재 블록에 포함된 지원 텍스트는 검사할 수 있다. 댓글·OCR·과거 페이지 이력은 별도 수집 capability와 승인을 적용한다. 수집기는 공통 DB·스케줄러·작업 선점과 lease 보호를 사용한다.

원천 객체 참조는 제목 `notion/pages/<page UUID>/title`, 블록 `notion/pages/<page UUID>/blocks/<block UUID>`로 고정한다. 텍스트 순서가 바뀌어도 같은 블록 ID를 사용하며, 버전은 `notion-v1:<last_edited_time>:<현재 객체 텍스트 SHA-256>`으로 기록한다. 각 객체를 독립적으로 Gitleaks·정규화에 전달한다. AWS 키 ID·Secret·세션 토큰은 같은 제목 또는 같은 블록 안의 기존 문맥 근거로만 묶고, 다른 블록의 값은 불완전 후보로 보존한다. 같은 키가 여러 블록에 있으면 키 동일성은 합치고 위치는 각각 남긴다.

알려진 하위 페이지·데이터베이스·링크·첨부 본문 제외는 `excludedObjectCount`로 집계한다. 선언된 현재 페이지 텍스트를 완전하게 읽었다면 이 제외만으로 부분 실패가 되지 않는다. 알 수 없는 블록, 외부 동기화 블록, 중복·반복 커서, 접근 실패와 요청·깊이·블록·시간·크기 상한으로 지원 텍스트를 읽지 못하면 안전한 코드와 함께 `PARTIAL` 또는 읽은 객체가 없는 `FAILED`로 기록한다. 제외 개수와 실제 검사 누락은 분리한다.

페이지 메타데이터를 읽기 전후로 확인하고 `last_edited_time`·게시 상태가 바뀌거나 순회가 불완전하면 객체를 비교 불가로 기록한다. 내부 `comparable=true`는 이 관측에서 변경·누락을 감지하지 못했다는 의미다. 수정 시각은 분 단위로 절삭되며 커서 순회 계약만으로 페이지 전체의 원자적 스냅샷을 입증할 수 없으므로, 성공한 전체 스캔·위치 재검사에서도 Notion 미관찰을 `MISSING`이나 제거·회수 완료로 확정하지 않는다. 기존 위치는 `UNKNOWN`으로 보존하고 실제 관찰한 키·위치와 제한된 관측 버전을 기록한다. [수정 시각의 정밀도](https://developers.notion.com/guides/resources/historical-changelog#june-28-2021), [커서 페이지 나눔 계약](https://developers.notion.com/reference/pagination)

현재 승인 페이지 순회에서 같은 객체의 원문을 관찰하고 보호된 키 동일성이 일치하면 해당 키의 인증 확인을 허용한다. 이 양성 관찰과 공급자 인증 결과는 페이지 전체의 부재 증명과 별도로 보존한다. 원문·전체 응답은 저장하지 않으며 수집 연결의 인증 테스트는 `GET /v1/users/me`만 수행한다. 페이지 읽기 결과는 스캔 이력에서 확인한다.

공급자의 검증된 `Retry-After`는 동일 `auth_ref`의 연결들이 공유하는 수집 대기로 보존한다. 새 작업·다른 워커·다른 연결도 남은 대기를 적용하며 예약기와 실제 전송 직전에 다시 검사한다. 부분 수집 자료와 누락 사유는 보존하고 대기 기한을 로컬 재시도 간격으로 줄이지 않는다.

Enterprise Admin PAT 목록은 별도 관리 관측 계약이다. 수집 연결용 키와 문서에서 발견한 키는 사용 목적·동일성·승인을 분리하고, 문서에 포함된 AWS·GitHub·Notion 후보는 각 발급자의 검증기로 전달한다. [내부 연결과 공유 범위](https://developers.notion.com/guides/connections-and-tokens/internal-connections)

### AWS 메타데이터

실행 전 연결 자격 증명의 STS 계정이 승인 계정과 일치하는지 검사한다. 승인된 리전만 사용하며 리전을 자동 발견해 범위를 확대하지 않는다.

| content_type | 승인 참조 | 읽는 정보 |
|---|---|---|
| IAM_ACCESS_KEY_METADATA | `aws://<계정 ID>/global/iam/access-keys` | 현재 IAM 사용자 접근키 목록·상태·발급 시각·최근 사용 및 루트 접근키 개수 확인 |
| AWS_SECRETS_MANAGER_METADATA | `aws://<계정 ID>/<리전>/secrets-manager/all` | ListSecrets 항목 수 |
| AWS_SSM_PARAMETER_METADATA | `aws://<계정 ID>/<리전>/ssm/parameters` | DescribeParameters 항목 수 |
| AWS_S3_BUCKET_METADATA | `aws://<계정 ID>/global/s3/buckets` | ListBuckets 버킷 수. 객체 목록 제외 |

Secrets Manager·SSM·S3의 history_limit은 CURRENT_STATE_ONLY이고 IAM은 같은 값 또는 NULL을 허용한다. IAM ID·사용자명은 일반 DB·API·로그에 저장하지 않고 보호키 HMAC과 마스킹 표지만 기록한다. 루트 접근키가 있거나 확인할 수 없으면 사용자 목록만으로 전체 확인을 선언하지 않는다. 다른 세 목록은 이름·ARN·설명·태그를 작업 메모리에서 집계 후 버린다. 페이지 제한·권한 오류는 부분 수집이다.

IAM만 METADATA_ONLY 인벤토리와 실행별 관측을 만든다. 다른 세 목록은 실행 범위의 개수·상태·안전한 사유만 기록하며 CredentialSet·Occurrence·ValidationAttempt를 생성하지 않는다. 메타데이터의 탐지·검증 단계는 SKIPPED이며 값 발견·유효성·비정상 노출·심각도로 승격하지 않는다.

### AWS 정확한 단일 값

본문은 정확한 단일 자원 승인과 `SECRET_AWS_CONTENT_READ_ENABLED=true`를 요구한다. 기본 false이며 AWS 수집·작업 실행도 별도로 활성화한다. history_limit은 CURRENT_VALUE_ONLY만 허용한다. 전체 목록·접두사·와일드카드 승인을 본문 승인으로 해석하지 않는다.

| content_type | 승인 참조 | 읽는 현재 값 |
|---|---|---|
| AWS_SECRETS_MANAGER_VALUE | `aws://<계정 ID>/<리전>/secrets-manager/secret/<URL 인코딩한 정확한 Secret ARN>` | GetSecretValue의 AWSCURRENT 텍스트 |
| AWS_SSM_PARAMETER_VALUE | `aws://<계정 ID>/<리전>/ssm/parameter/<URL 인코딩한 정확한 Parameter ARN>` | 복호화 옵션을 사용한 GetParameter 값 |
| AWS_S3_OBJECT_CONTENT | `aws://<계정 ID>/<리전>/s3/object/<버킷>/<URL 인코딩한 정확한 객체 키>` | 소유 계정·크기·ETag 확인 후 If-Match 조건부 GetObject. HeadObject에 버전 ID가 있으면 해당 버전 지정 |

approved_ref는 VARCHAR(500) 한도를 초과하면 거부한다. ARN·Parameter 이름·객체 키는 민감 메타데이터로 접근을 제한한다. 기본 256 KiB·절대 상한 1 MiB의 UTF-8 텍스트만 제한된 메모리에서 동기 탐지하며 S3는 시간·크기도 제한한다. 결과 위치에는 자원명 대신 `aws-content-scope:<scope_id>`를 기록한다. 본문 탐지는 그 보관 위치의 외부 노출이나 권한·영향 판단이 아니다.

## 탐지와 정규화

Gitleaks 8.30.1을 후보 추출기로 사용한다. 승인된 콘텐츠를 원천 객체별로 stdin에 전달하고 원천 ID·버전·줄을 연결한다. 입력 크기·형식·파서 출력을 제한하며 크기 초과·본문 누락·도구 실패를 정상 0건으로 처리하지 않는다. 도구·실행 파일 해시·룰 ID·설정 해시를 기록한다.

유형별 형식·접두사·정규식·문맥으로 후보를 찾되 비밀 구성요소와 공개 식별자를 구분한다. 불완전 AWS AccessKeyId·공개 인증서만으로 완전한 자격 증명을 만들지 않는다. 동일성·구성요소 연결은 [데이터 계약](data-contract.md)의 근거를 요구한다.

Notion 후보는 현재 `ntn_[A-Za-z0-9_]{46}`와 기존 `secret_[A-Za-z0-9]{43}` 지원 형식을 탐지·정규화·검증 후보 생성에서 일치시킨다. 길이·문자열 경계와 엔진 Finding의 위치 근거를 확인한다. 기존 정규형은 `issuer=NOTION`, `credential_type=NOTION_CONNECTION_TOKEN`을 사용하고 내부 연결·PAT·OAuth 세부 분류는 별도 발급 근거로 보완한다. 새 포맷은 탐지 등록부 버전과 회귀 시험으로 추가한다. 인증 상태와 세부 유형은 각각 실제 인증 관측·발급 근거로 확인한다. refresh·client secret·관리용으로 식별된 문맥은 해당 유형의 지원 capability로 분류한다.

정상·노출·변형·불완전·중복 합성 표본과 승인 시험 자료로 유형별 누락·오탐·위치 복원을 검증한다. 합성 표본 발견률을 운영 재현율로 표현하지 않는다. 다른 탐지기를 채택할 때도 같은 입력·원문 보호·실패·중복 처리 기준을 적용한다. TruffleHog 평가에서는 `--no-verification`으로 도구 자체 능동 검증을 끈다. 도구 결과를 공급자 유효성으로 복사하지 않는다.

## 검증기 등록

실행 예약·원문 재취득·공급자별 결과·시간·호출 한도의 세부 명세는 [발견 키 유효성 검증](credential-validation.md)을 따른다. 권한·대상·자산 중요도 근거와 등급 평가의 저장·실행은 [심각도 평가 계약](risk-assessment-contract.md)을 따른다. 두 계약은 인증 가능성과 영향 범위를 별개의 사실로 기록한다.

검증기마다 발급자·키 유형·필수 구성요소·승인 호스트와 엔드포인트·허용 요청·연결 권한·호출 한도·응답 해석·버전을 등록한다. 미등록 유형과 엔드포인트에는 네트워크 요청을 하지 않는다. 검증은 실제 데이터 다운로드나 키·권한 변경을 포함하지 않는다.

| 유형 | 승인 가능한 검증 방법 | 독립적으로 구분할 사실 |
|---|---|---|
| AWS 장기·임시 자격 증명 | 완전한 키 쌍, 임시 키는 세션 토큰을 포함해 STS GetCallerIdentity | 인증 가능성과 업무 리소스 권한은 별개. 세션 토큰 누락은 불완전 |
| GitHub 사용자 계열 토큰 | 인증 사용자 확인 | 공개 API 성공·특정 저장소 권한 부재와 토큰 유효성을 혼동하지 않음 |
| GitHub 설치 토큰 | 설치 범위의 승인 조회 | 사용자 토큰 검증과 분리 |
| Notion 콘텐츠 API 토큰 | 고정 `GET /v1/users/me`, `NOTION_GET_SELF/NOTION_SELF` | 내부 연결·PAT·OAuth access의 현재 인증과 제한된 주체 관측. 페이지 접근·capability·만료·폐기 근거는 각각 기록 |
| K8s 토큰·개인키·웹훅 secret·임의 암호 | 유형별 안전한 승인 방법이 등록된 경우만 능동 검사 | 범용 검증 API를 가정하지 않고 미지원·불완전·수동 확인을 명시 |

검증기 지원에는 성공·명시적 무효·권한 부족·호출 제한·네트워크 오류·필수 구성 누락의 승인된 시험 결과가 필요하다. 표에 있다는 이유만으로 운영 지원 완료를 표시하지 않는다.

AWS STS 검증은 본문 읽기와 독립적인 `SECRET_AWS_VALIDATION_ENABLED` 설정이며 기본 false에서 UNCHECKED/DISABLED를 기록한다. 승인된 합성 fixture 원천은 설정과 관계없이 능동 호출하지 않고 UNCHECKED/SYNTHETIC_FIXTURE다. 실행 주기 내 같은 자격 증명은 중복 검증하지 않고 시도를 관련 위치에 연결한다. 검증기는 자원 수집·권한·영향 분석기를 대체하지 않는다.

## 결과와 실패

결과는 VALID·INVALID·UNKNOWN·UNCHECKED와 사유·방법·시각·검증기 버전이다. 시도 전 미지원·원문 부재는 UNCHECKED, 시도 후 공급자 오류·재취득 실패는 UNKNOWN이다. 공급자 근거가 명확할 때만 만료·폐기를 별도 사유로 기록한다. 상태 코드만으로 일괄 무효·만료·폐기를 판정하지 않는다.

조치 이후 검사 전에는 과거 VALID를 현재 확정 결과처럼 표시하지 않고 조치 이후 재검사 대기를 표시한다. 최근 시도와 마지막 성공을 분리하고 실패는 마지막 성공 시각을 갱신하지 않는다. 최근 사용 메타데이터도 독립 출처·권한·시각을 갖는다. 재시도·스케줄은 [보안과 실행 정책](security-and-runtime.md)을 따른다.

## 영향 범위 증거

유효성 검증기와 영향 분석기는 별도 결과를 낸다. 영향 분석기는 발급자·토큰 유형별 매핑 버전, 확인 권한·적용 대상·제한 통제·자산 분류 버전·증거 ID·시각을 반환한다. [심각도 정책](../02-functional-design/severity-policy.md)의 ADMIN·IMPORTANT·LIMITED·MINIMAL·UNKNOWN으로 정규화한다.

권한 목록·대상 조회의 완전성을 명시한다. 권한 부족·미지원 토큰·일부 대상 누락·상충 자산 분류는 UNKNOWN과 사유를 반환한다. 인증 성공·유형 이름만으로 권한 범주를 채우지 않는다. 공급자 매핑은 합성 사례와 승인된 시험 계정의 권한 경계 사례를 통과한 버전만 활성화한다.
