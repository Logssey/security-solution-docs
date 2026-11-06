# 수집원과 검증기 계약

기준: [PRD](../01-requirements/PRD.md) FR-01부터 FR-09, [ADR-S001](adr/ADR-S001.md), [ADR-S002](adr/ADR-S002.md), [ADR-S006](adr/ADR-S006.md). 키 유형·구성요소 분류는 [자격 증명 유형](../04-reference/credential-types.md)을 따른다.

## 연결별 수집 경계

| 연결 | 수집 계약 | 독립적으로 승인·표시할 범위 |
|---|---|---|
| GitHub | 승인 저장소·브랜치의 현재 커밋 파일과 Actions 워크플로 설정 파일 | 현재 파일·과거 이력을 구분한다. 실행 로그·아티팩트·저장 비밀 값은 별도 수집 권한과 지원 계약 없이는 읽지 않는다. |
| Notion 확장 계약 | 승인된 현재 페이지와 지원 텍스트 블록 | 수집 연결이 접근할 수 있는 자료와 서비스의 승인 목록을 교차한다. 하위 페이지·페이지 나눔·누락·미지원 범위를 기록한다. |
| AWS | 승인 계정·리전·서비스·리소스별 메타데이터와 정확한 단일 자원 본문 | 목록 메타데이터·값·객체·로그는 독립 범위다. 계정 읽기 승인을 모든 콘텐츠 수집으로 확대하지 않는다. |
| K8s 확장 계약 | 승인 클러스터·네임스페이스의 내부 Secret 객체 | Kubernetes API·최소 RBAC를 사용한다. projected 토큰·외부 저장소·노드·etcd는 Secret 객체 범위에 포함하지 않는다. |

실행은 등록된 수집기 capability와 승인 콘텐츠 유형이 일치할 때만 허용한다. GitHub의 CURRENT_COMMIT_ONLY와 AWS의 명시적 메타데이터/단일 값 계약을 구별하고 Notion·K8s·과거 커밋 이력은 그에 맞는 별도 수집기가 등록되기 전까지 지원 완료로 표시하지 않는다. 연결마다 콘텐츠 유형·원천 객체 ID와 버전·읽은 범위·최초/최근 성공·부분 실패·거부·미지원 사유를 공통 결과로 반환한다. 승인 대상이 없는 정상 0건과 접근 실패의 0건을 구분한다. 발견 URL을 이유로 새 원천을 자동 추가하지 않는다. 수집기는 스케줄러나 대응 저장 모델을 독자적으로 정의하지 않는다.

GitHub는 승인 객체의 파일 바이트를 읽고 객체·커밋·줄 위치를 추적한다. Git 저장소를 워커 디스크에 복제하거나 탐지 도구에 연결 자격 증명을 전달하지 않는다. 현재 파일과 이력 검사는 별도 승인·지원 범위이며 한쪽의 성공을 다른 쪽 완료로 표시하지 않는다.

Notion 수집 연결은 콘텐츠 읽기 권한의 내부 연결을 사용하고 DB에는 비밀 보관소의 `auth_ref`만 저장한다. 승인 원천은 `NOTION_PAGE_CONTENT`, `notion://pages/<page UUID>`, `CURRENT_VERSION_ONLY`로 등록한다. UUID를 정규화하고 임의 호스트·query·본문·다른 URI 형식을 거부한다. 승인 상위 페이지의 하위 페이지를 추가하려면 현재 승인 목록과 하위 포함 정책을 확인한다. API 자체의 공유 상속만으로 검사 범위를 넓히지 않는다. 수집기에는 별도 스케줄러·결과 DB를 두지 않는다.

현재 페이지 메타데이터와 본문·지원 블록을 제한된 메모리에서 읽고 원천 객체 버전·블록 좌표 또는 보호된 문서 위치 근거를 함께 보존한다. Markdown의 `truncated`, 누락·지원 밖 블록, 다음 페이지 실패, 접근 해제와 시간·크기 상한은 범위별 부분 실패와 누락 사유로 기록한다. 문장·설정값·코드 블록의 후보를 같은 정규화 경로로 전달하며 재발견 시 키는 하나로, 발견 위치는 각각 보존한다. 첨부·OCR·댓글·과거 페이지 이력은 등록된 콘텐츠 capability에서 각각 범위를 선언한다.

페이지의 last_edited_time·제한된 본문 버전 근거와 지원 블록 범위를 비교해 현재 관측의 비교 가능성을 기록한다. 읽는 중 수정·페이지 나눔 결과 변동·필수 블록 누락이 확인되면 위치 부재 판정은 UNKNOWN으로 보존하고 재수집을 안내한다. 공유 검색 결과는 현재 관측 목록과 누락 한계를 함께 기록한다. [검색의 완전성 한계](https://developers.notion.com/reference/search-optimizations-and-limitations)

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
