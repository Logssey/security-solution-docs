# Secret 위험 근거와 심각도 평가 계약

기준: [심각도 정책](../02-functional-design/severity-policy.md), [ADR-S007](adr/ADR-S007.md), [데이터 계약](data-contract.md), [DB 스키마](database-schema.md), [API 계약](api-contract.md), [자격 증명 검증 계약](credential-validation.md).

이 문서는 `Occurrence`의 여섯 축을 근거에서 정규화하고 `severity-v2`의 R00~R99를 재현하는 저장·평가기 계약이다. 수집 플랫폼, 자격 증명 발급자, 인증 유효성, 보관 허용, 영향과 사고 사실은 독립이다. 정책·규칙·접근/영향 근거·평가기록을 네 테이블에 분리하며 기존 판정·검증·수명주기·조사·감사 원장은 재사용한다.

## 1. 평가 단위와 결과

평가 단위는 `Occurrence`다. 같은 키의 다른 위치에서 얻은 노출 상태나 접근 범위를 대신 사용하지 않는다. 키 공통 권한 근거를 재사용할 때에도 동일한 `CredentialSet`임을 검증하고 해당 위치의 근거 접근 권한과 적용 범위를 다시 확인한다.

| 입력 축 | 정확한 값 | 정규화의 원본 |
|---|---|---|
| `exposure` | `ABNORMAL`, `NORMAL`, `UNKNOWN` | 해당 위치의 유효 `secret_exposure_assessments`와 보관 정책·바인딩·범위 revision |
| `exposureRange` | `BROAD`, `BOUNDED`, `UNKNOWN` | 해당 위치의 접근 명단·공개성·공유 정책과 관측 범위 |
| `credential` | `CONFIRMED`, `CANDIDATE` | 키 구성·정확한 동일성·유효 Finding. `CONFIRMED`는 인증 성공을 뜻하지 않음 |
| `impact` | `ADMIN`, `IMPORTANT`, `LIMITED`, `MINIMAL`, `UNKNOWN` | 발견 키에 결합된 권한·실제 대상·제한 통제·자산 중요도와 매핑 완전성 |
| `abuse` | `CONFIRMED`, `NOT_CONFIRMED` | 동일 키·위치와 연관된 중대한 비인가 사용 및 영향의 증거 |
| `validity` | `VALID`, `INVALID`, `UNKNOWN`, `UNCHECKED` | 해당 키의 검증 시도·공급자 관측 시각. 심각도 등급과 독립 |

`METADATA_ONLY`는 여섯 축의 자격 증명 값이 아니다. 위험 GET에서 `NOT_APPLICABLE`, `severity=null`, `assessmentId=null`, `ruleId=null`, `reasonCode=METADATA_ONLY`로 투영하고 R00~R99의 입력에 넣지 않는다. 메타데이터만을 위한 ASSESS 작업·AUTO_RESULT·대응 건을 만들지 않으며 평가 요청에는 대상 제외 사유를 반환한다. 원래 메타데이터 원장은 유지한다. 값 후보가 별도 발견되어 정확한 동일성 근거로 구성 상태가 전환되면 새 revision으로 평가한다. 과거 확정 등급이 있는 키를 구성 상태 교정만으로 하향시키지는 않는다.

| `assessmentStatus` | `severity` | 의미 |
|---|---|---|
| `ASSESSED` | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` 중 하나 | R01~R07 또는 승인된 수동 override의 등급 |
| `UNASSESSED` | `null` | R99. 필요한 근거 부재·지원 한계·충돌 때문에 보류 |
| `NOT_APPLICABLE` | `null` | R00 정상 보관 또는 `METADATA_ONLY` 등 평가 대상 밖 |

`UNASSESSED`와 `NOT_APPLICABLE`는 위험 등급이 아니다. 정상 보관의 R00과 근거 부족의 R99를 같은 상태로 반환하지 않는다. 기존 응답의 `severity="UNASSESSED"` 표현이 필요한 투영은 `assessmentStatus=UNASSESSED`에서만 만든다. 저장되는 `severity`에는 이 문자열을 넣지 않는다. 근거가 없는 행을 LOW나 MEDIUM으로 채우지 않는다.

결과는 `latestAutomaticAssessment`, `effectiveAssessment`, `lastConfirmedSeverity`를 구분한다. 첫 항목은 최신 계산 사실, 두 번째는 현재 적용된 등급과 보류 사유, 세 번째는 과거의 마지막 확정 등급·평가 ID·시각이다. 최신 자동 결과가 미평가여도 이전 등급이 사라진 것으로 표시하지 않는다. `lastConfirmedSeverity`만으로 현재 등급이나 건을 생성하지 않는다.

## 2. 근거의 타입·동일성과 접근 경계

### 2.1. 원본 재사용과 추가 근거

| 사실 | 사실 원본 | 위험 계약이 저장하는 것 |
|---|---|---|
| 정상·비정상 노출과 오탐 교정 | `secret_exposure_assessments`, `secret_finding_dispositions`, 보관 정책 원장 | 평가 당시 ID·version·적용 revision의 typed 참조 |
| 키 구성·탐지·존재 | `secret_credential_sets`, `secret_findings`, `secret_occurrences`, 실행 원장 | 동일성·구성 revision, Finding·실행 참조 |
| 인증 유효성 | `secret_validation_attempts` | 검증 시도 FK와 기준 시각. 유효성 결과의 별도 원장을 만들지 않음 |
| 교체·폐기·해결 | 수명주기 근거·review·application, 조치·해결 원장 | 해당 근거의 참조와 현재 적용 가능성. 심각도 하향의 자동 증거로 사용하지 않음 |
| 실제 접근 범위·권한·대상·제한·자산 분류·사고 연결·조사 커버리지 | `secret_risk_evidence` | 안전한 타입별 관측과 provenance, 해석·분류 버전, 완전성·시각. 노출 판정은 별도 원장 유지 |
| 자료 보완·담당 요청 | `secret_investigation_tasks`, `secret_investigation_history` | 과제·수용 이력 참조. 같은 위치·질문의 열린 과제를 재사용 |
| 명령·권한·정책 활성화 | 기존 명령 멱등·감사 원장 | 대상 ID·해시·변경 전후 ID·사유. 본문·키·공급자 응답 제외 |

`secret_risk_evidence`의 허용 `kind`는 `ACCESS_SCOPE`, `PERMISSION_MAPPING`, `ASSET_CLASSIFICATION`, `ABUSE_LINK`, `ABUSE_COVERAGE`다. ACCESS_SCOPE는 실제 공개성·접근 범위의 관측이며 정상/비정상 노출 판정이 아니다. 노출 판정이나 `VALID`·`INVALID`를 이 타입에 새 사실로 적재하지 못한다. 외부 자료의 참조만 제출한 것은 입증된 영향이 아니며 수용 전 상태를 보존한다.

| kind | safe_payload의 필수 타입별 필드 |
|---|---|
| `ACCESS_SCOPE` | schemaVersion, 해당 위치/객체·scope revision, accessSourceProvider, 관측 audienceClass·coverageStatus·공개 접근 확인 참조 또는 보호된 제한 주체/권한 참조, 실제 관측 시각·효력 만료·누락/상속/공유 제한 코드. 자격 증명 권한이나 노출 판정과 구분 |
| `PERMISSION_MAPPING` | schemaVersion, provider/type/mappingVersion, identityBindingStatus·동일성 참조, 인식한 permissions의 코드·effect·보호된 targetRef·적용 제약 코드, 네 coverage 상태·누락, positiveCapabilities, 자산 분류 참조. 원시 정책 본문·토큰·임의 API 필드 제외 |
| `ASSET_CLASSIFICATION` | schemaVersion, assetId 또는 승인된 보호 대상 참조, classificationVersion, 분류된 환경·민감도·중요도·테스트 전용 여부의 등록 enum, 확인한 대상 경계·분류 원본 참조. 이름·태그에서 추론한 분류 금지 |
| `ABUSE_LINK` | schemaVersion, 보호된 사건 참조, 사건 observationWindow, 비인가 사용 확인 참조, 중대한 영향 확인 참조, 같은 키 결합 참조, 해당 노출과의 연결 참조. 각각의 필수 참조가 없으면 CONFIRMED 주장 생성 불가 |
| `ABUSE_COVERAGE` | schemaVersion, investigationStatus, coverageStatus, 조사 observationWindow, 자료 종류별 수집 성공/실패·확인 기간, 누락 코드. COMPLETE와 무관하게 악용 없음의 주장으로 해석하지 않음 |

각 payload는 해당 schema version에서 필수/선택 필드와 enum을 닫아 검증하며 알 수 없는 필드를 보존하지 않는다. 배열 상한과 문자열 길이 제한은 승인된 schema에 포함하고 제한을 넘으면 일부만 저장해 COMPLETE로 기록하지 않는다.

노출 판정 당시의 ACL 근거는 기존 `secret_exposure_assessments`의 `exposure_range VARCHAR(10)`와 `access_snapshot JSONB`에 불변 snapshot으로 고정한다. 과거 행의 NULL은 `UNKNOWN` 근거 부재다. exposure_range CHECK는 `BROAD|BOUNDED|UNKNOWN`이고 access_snapshot은 `schemaVersion,sourceRefs,observedAt,scopeRevision,sourceObjectVersion,policyBindingId,audienceClass,coverageStatus,limitationCodes,collectorIdentityRef`의 닫힌 schema다. audienceClass는 당시 exposure_range와 같고 coverageStatus는 `COMPLETE|PARTIAL|UNKNOWN`이다. sourceRefs는 해당 ACCESS_SCOPE 관측의 typed 부모를 포함할 수 있다. 전체 principal 목록·이메일·원천 URL을 snapshot에 복제하지 않는다.

새 ACL 관측은 `secret_risk_evidence/ACCESS_SCOPE`에 추가하고 같은 위치·객체·scope revision 및 관측 시각을 검증한 뒤 현재 exposureRange를 정규화한다. 승인된 수집 capability가 이 관측을 갱신할 수 있으며 6시간마다 사람이 같은 노출 판정을 반복하지 않는다. BOUNDED는 제한된 주체·실제 권한 범위를 입증한 보호된 참조와 COMPLETE를 요구한다. 누락된 페이지·상속 공유·외부 접근 여부 때문에 범위를 제한할 수 없으면 UNKNOWN이다. 입증된 공개/조직 전체 접근은 나머지 주체를 모두 열거하지 못해도 BROAD의 양성 근거가 될 수 있다. ACCESS_SCOPE는 기존 NORMAL/ABNORMAL을 자동 전환하지 않는다. 새 접근 관측이 NORMAL의 승인 근거와 모순되면 판정 원본을 보존하고 현재 입력은 UNKNOWN·재검토로 처리한다. 기존 ABNORMAL에서 제한 접근이 새로 관측돼도 NORMAL이나 해결로 자동 바꾸지 않는다.

근거 참조 형식은 `{type,id,version,role}`이며 `type`은 서버가 등록한 테이블별 enum, `id`와 `version`은 양의 정수, `role`은 `ASSERTS|BINDS_IDENTITY|LIMITS|CLASSIFIES|CONTRADICTS|REVIEWS` 중 허용 조합이다. 자유 URL·테이블명·SQL·실행 코드는 받지 않는다. 참조 배열은 순서와 중복을 정규화한다. 저장 트랜잭션은 타입별 실제 부모의 존재, 같은 키·위치·출처, 적용 범위와 ACL을 검사한다. JSON 내부의 ID가 있다는 이유만으로 FK 검증을 마친 것으로 간주하지 않는다.

### 2.2. provenance와 두 신원

모든 근거는 아래 값을 갖는다. 알려지지 않은 값은 `null`과 명시적 상태로 표현하며 표시명이나 마스킹 suffix로 동일성을 만들지 않는다.

| 값 | 의미와 제약 |
|---|---|
| `sourceKind`, `sourceRefs[]` | 승인 워커 관측, 자산 분류, 수동 제출·수용을 구분. 원본 레코드와 버전까지 고정 |
| `provider`, `credentialType` | 발견 자격 증명의 발급자·유형. 원천 수집 플랫폼과 독립 |
| `collectorConnectionId`, `collectorIdentityRef` | 사실을 조회한 연결과 수집용 신원의 보호된 참조 |
| `credentialIdentityRef`, `identityBindingStatus`, `identityEvidenceRefs[]` | 발견 키가 행사하는 신원과 결합 근거. `MATCHED\|UNRESOLVED\|CONFLICT\|NOT_APPLICABLE` |
| `observedAt`, `recordedAt`, `effectiveFrom`, `expiresAt` | 실제 관측, 서버 수신, 적용 시작, 근거 효력 만료. 네 시각을 대체하지 않음 |
| `observationWindow`, `coverage`, `missingReasonCodes[]` | 관측 기간·대상·로그/권한 조회의 범위와 누락 |
| `normalizerVersion`, `mappingVersion`, `assetClassificationVersion` | 해석 및 자산 중요도 버전. 입력을 재현할 수 있게 고정 |
| `verificationLevel` | `PROVIDER_OBSERVED\|ASSET_CLASSIFIED\|MANUAL_SUBMITTED\|MANUAL_ACCEPTED`. 서로 자동 승격하지 않음 |
| `safePayload`, `safePayloadDigest` | 타입별 허용 필드와 canonical SHA-256. 키 원문·일반 비밀 지문을 해싱해 저장하지 않음 |

수집용 계정의 관리자 권한은 발견 키의 영향 근거가 아니다. 권한을 수집용 신원으로 조회했더라도 결과가 발견 키·주체·대상에 결합됐음을 별도로 입증해야 한다. `PERMISSION_MAPPING`에서 결합 상태가 `MATCHED`가 아니면 영향은 `UNKNOWN`이다. `NOT_APPLICABLE`는 ACCESS_SCOPE나 자산 분류처럼 발견 키 신원 결합이 필요 없는 종류에만 허용하며 해당 위치/대상 결합은 별도로 요구한다. 검증의 신원 조회 결과만으로 권한·대상·업무 사용을 추정하지 않는다.

`scope_id`는 근거를 읽을 수 있는 범위의 ACL 기준점이다. 키 전체에 적용할 수 있는 근거도 해당 `Occurrence`를 읽을 권한과 근거의 허용 범위를 모두 통과해야 한다. 다른 위치에 근거를 적용할 때는 원래 안전한 근거 ID를 참조하며 원문을 복제하거나 키 전체 열람 권한을 부여하지 않는다. 일부 근거를 볼 수 없는 사용자에게 근거 ID·보호된 대상·제한 위치의 수를 노출하지 않는다.

### 2.3. 공급자별 지원·권한 매핑 완전성

승인된 정책의 `permission_mapping_manifest`는 `(provider,credentialType,mappingVersion)`마다 아래 계약을 고정한다. 단순히 공급자를 지원한다고 표시하는 것은 충분하지 않다.

| 선언 | 필수 내용 |
|---|---|
| `support` | `SUPPORTED\|PARTIAL\|UNSUPPORTED`. 유효성 지원과 영향 분석 지원을 별도 선언 |
| 조회 capability | 읽을 수 있는 권한·대상·제한 필드, 안전한 조회 종류와 실패 코드. API 결과의 존재와 완전성 보장은 별개 |
| 권한 매핑 | 인식한 권한 이름·버전, 영향 분류 근거, 알 수 없는 권한 처리 |
| 대상 매핑 | 적용 주체, 조직·계정·저장소·자원 범위, 포함·제외의 확인 방법과 페이지/열거 완료 조건 |
| 제한 매핑 | 적용 가능한 조직 제한·경계·세션/설치 제한·환경 통제의 지원 여부 및 관측 상태 |
| 자산 분류 | 승인된 자산 분류의 버전·적용 대상·민감도·운영 여부와 누락 처리 |
| 완전성 | `permissionCoverage`, `targetCoverage`, `restrictionCoverage`, `classificationCoverage`: 각각 `COMPLETE\|PARTIAL\|UNKNOWN\|NOT_APPLICABLE`과 근거·누락 사유 |

`NOT_APPLICABLE`는 매핑 manifest가 해당 유형에 그 통제가 적용되지 않음을 정의하고 증명한 경우에만 허용한다. API가 필드를 생략하거나 수집 신원이 읽을 수 없다는 뜻으로 쓰지 않는다. 허용된 필드의 `COMPLETE` 또는 입증된 `NOT_APPLICABLE`, 발견 키 신원 결합, 지원되는 매핑·자산 버전이 함께 있어야 `ADMIN|IMPORTANT|LIMITED|MINIMAL`을 확정한다. 불완전 상태의 확인된 높은 권한은 별도 `positiveCapabilities` 사실로 보존하되 완전한 영향 범주로 바꾸지 않는다. 미지원 권한·대상·제한이 하나라도 평가에 영향을 줄 수 있으면 `impact=UNKNOWN`과 정확한 누락 사유를 반환한다.

GitHub의 사용자 인증·헤더·단일 저장소 접근 응답은 토큰 전체 권한이나 전체 대상을 증명하지 않는다. fine-grained PAT와 App/설치 토큰에 대해 모든 권한·선택 대상·조직 제한을 자체 조회 API로 열거할 수 있다고 가정하지 않는다. 권한 문서나 수동 자료를 보완해도 실제 해당 키에 대한 결합과 누락 범위의 해소가 필요하다. 인증만 지원되는 유형은 유효성만 반환하고 영향 분석은 `PARTIAL` 또는 `UNSUPPORTED`, `impact=UNKNOWN`을 유지한다.

AWS의 주체 확인·목록 상태·마지막 사용 관측은 유효 권한의 전체 평가가 아니다. 발견 키에 적용되는 정책·경계·조직·세션/자원 조건과 실제 대상·자산 중요도를 확인할 수 없는 부분을 누락으로 기록한다. 수집용 연결의 넓은 권한을 대입하거나 조회 실패를 낮은 권한으로 바꾸지 않는다. 구체 검증 방법과 허용 요청은 [자격 증명 검증 계약](credential-validation.md)에 따른다.

`abuse=NOT_CONFIRMED`에는 별도 `investigationStatus=NOT_STARTED|IN_PROGRESS|COMPLETED|UNAVAILABLE`, `coverageStatus=COMPLETE|PARTIAL|UNKNOWN`, 조사 기간·자료 유형·누락을 보존한다. 이 값은 악용이 없다는 뜻이 아니다. 로그 없음·미수집·조사 종료·최근 사용 기록 없음은 `CONFIRMED`의 반증이 아니다. `ABUSE_LINK`의 `CONFIRMED`에는 비인가 행위와 중대한 영향, 동일 키 및 해당 노출과의 연결을 입증한 참조가 필요하다.

## 3. 정규화와 결정적 평가

### 3.1. 근거 선택·최신성·충돌

평가기 입력은 `{policyVersion,policyHash,engineVersion,normalizerVersion,freshnessProfileVersion,evaluationAsOf,occurrenceId,credentialRevision,scopeRevision,evidenceRevision,sourceHeadRefs,evidenceRefs,inputs}`다. 서버가 DB 사실에서 만든다. 클라이언트는 등급·축 값·공급자 응답·관측 시각을 임의 입력해 자동 결과를 만들 수 없다.

1. 대상·현재 범위·키 동일성을 고정한다. 메타데이터는 평가 진입에서 분리한다. 자료 취소·효력 기간·부모 삭제/오탐 교정·적용 범위·revision을 검사한다.
2. 각 축을 독립 정규화한다. 한 축의 실패로 다른 축의 확정 사실을 지우지 않는다. `NEEDS_MORE_INFO`는 노출 `UNKNOWN`으로, `FALSE_POSITIVE`는 해당 Finding의 교정으로 다룬다. 다른 유효 Finding이 남으면 위치 전체를 임의 제외하지 않는다.
3. 현재 주장에 필요한 최신성은 `evaluationAsOf`와 버전 고정된 freshness profile로 판정한다. 권한·접근 명단·자산 분류에는 타입별 최대 나이와 revision 일치를 검사한다. 관측 시각 부재·허용 시계 오차 초과·만료는 `STALE_EVIDENCE` 또는 `TIME_UNCERTAIN`이다. 수신 시각으로 실제 관측을 새롭게 만들지 않는다.
4. 역사적 `ABUSE_LINK`는 사건 시점의 사실이다. 키 폐기·조회 실패·일반 최신성 TTL만으로 효력이 사라지지 않는다. 반증·정식 취소는 새 근거와 검토로 기록하며 원래 사실과 R01 평가를 보존한다. 현재 주장과 역사적 주장의 시간 범위를 구분해 비교한다.
5. 같은 대상·유효 시간에 적용되는 모순된 주장을 수용 시각이나 작성자 직급으로 덮어쓰지 않는다. 근거가 명시적으로 이전 것을 교정하고 필요한 권한을 통과한 경우에만 supersession을 적용한다. 그렇지 않으면 `CONFLICTING_EVIDENCE`와 축별 미확정값을 사용한다. 노출·범위·영향은 `UNKNOWN`, 자격 증명 확인을 입증할 수 없으면 `CANDIDATE`, 악용을 확정할 수 없으면 `NOT_CONFIRMED`다. 이미 확정된 역사적 사고를 현재 조회 실패로 미확정으로 되돌리지는 않는다. 유효성의 최신성·미실행 표현은 별도 검증 계약을 따른다.
6. `NORMAL`과 동일 키·해당 노출에 연결된 중대한 악용 `CONFIRMED`가 함께 있으면 `EXPOSURE_INCIDENT_CONFLICT`다. 저장된 NORMAL 판정은 보존하고 평가용 `exposure=UNKNOWN`으로 둔다. R00을 적용하지 않고 R99로 보낸다. 자동으로 `ABNORMAL`로 바꾸지 않는다. 기존 사고의 CRITICAL 적용·이력은 하향하지 않고 보안 담당자 조사로 연결한다.

정책은 실제 수집 주기와 종류별 허용 나이를 숫자·단위로 명시한 freshness profile 없이는 활성화할 수 없다. 평가기는 숨겨진 기본 TTL이나 실행 시점의 현재 시각을 사용하지 않는다. 동일 snapshot과 `evaluationAsOf`의 replay는 당시의 최신성 판정을 재현한다. 나중의 현재 상황을 확인하는 재평가는 새로운 `evaluationAsOf`와 snapshot을 갖는다.

초기 `severity-v2`의 profile은 `risk-freshness-v1`이며 다음 값을 사용한다. 나이는 `evaluationAsOf-observedAt`으로 계산하고 최대 나이와 같은 시각부터 오래된 근거로 처리한다. 승인된 보관 정책·수동 결정의 고유 효력 기간은 별도로 검사한다.

| 종류·경계 | `risk-freshness-v1` 값 |
|---|---|
| 현재 접근 명단·공개성·노출 범위의 관측 | 최대 6시간 |
| 현재 권한·대상·제한 매핑 관측 | 최대 24시간 |
| 자산 중요도 분류 관측/확인 | 최대 24시간. 분류 버전 변경 시 즉시 재검사 |
| 수동 제출한 현재 영향/접근 주장 | 관측 후 최대 24시간이며 원래 자료 만료가 더 이르면 그 시각 적용 |
| 수동 override | 승인 시점부터 최대 7일. 제안된 validUntil과 기반 근거 만료 중 더 이른 시각 적용 |
| 미래 관측 시각의 허용 오차 | 서버 기준 +5분. 초과 시 TIME_UNCERTAIN |
| 역사적 사건·중대한 악용 | 일반 최대 나이 없음. 사건 기간·동일성·반증/취소의 유효성 검사 |

scope/정책 바인딩·키 동일성·원본 권한/자산 revision 변경은 위 시간이 남아 있어도 즉시 재평가 사유다. 현재성 확인이 필요한 NORMAL의 보관 정책은 자신의 유효 기간·정확한 범위/위치·revision과 실제 접근의 근거를 확인하며, 최신 범위 관측이 필요하면 동일 6시간 제한을 적용한다. 최대 7일은 최신 근거가 7일간 유효하다는 뜻이 아니다. profile 변경은 작성자와 다른 정책 운영 승인자의 검토를 거친 새 policy version의 PR/배포로만 수행하며 개별 자료 제출·수용 요청으로 TTL을 늘릴 수 없다.

### 3.2. 규칙 언어와 first-match

규칙은 여섯 축의 조건을 AND로 결합한 선언 데이터다. 허용 연산자는 `EQ`, `IN`, `ANY`뿐이다. 각 축은 조건에 정확히 한 번 존재한다. `EQ`는 해당 enum 한 개, `IN`은 해당 enum의 비어 있지 않은 중복 없는 집합, `ANY`는 값 필드가 없는 명시적 비참조 연산자다. `UNKNOWN`은 실제 enum 값이며 wildcard가 아니다. `null`, 빈 배열, 조건 생략, 문자열 `*`를 ANY로 해석하지 않는다. 규칙 데이터로 SQL·스크립트·정규식·함수명·외부 호출·가중치 산식을 실행하지 않는다.

예를 들어 R01의 선언은 아래와 같다. `ANY`는 그 축의 증거가 완전해야 한다는 요구를 만들지 않는다.

```json
{
  "exposure": {"op": "EQ", "value": "ABNORMAL"},
  "exposureRange": {"op": "ANY"},
  "credential": {"op": "ANY"},
  "impact": {"op": "ANY"},
  "abuse": {"op": "EQ", "value": "CONFIRMED"},
  "validity": {"op": "ANY"}
}
```

필수 근거는 규칙 경로별로 검사한다. 모든 축의 COMPLETE를 일괄 전제하지 않는다. R00은 유효 NORMAL 근거와 충돌 부재가 있으면 범위·영향·현재 인증이 미확인이어도 적용한다. R01은 유효 ABNORMAL 및 연결된 중대한 악용 증거가 있으면 credential이 CANDIDATE이거나 범위·영향·현재 인증이 UNKNOWN/UNCHECKED여도 적용한다. R02~R07은 `CONFIRMED`, 확인된 노출 범위와 해당 영향 분류 근거가 필요하다. 유효성은 모든 등급 규칙에서 ANY다.

| 순서·ID | `exposure` | `exposureRange` | `credential` | `impact` | `abuse` | `validity` | 상태·등급 |
|---|---|---|---|---|---|---|---|
| 0 R00 | EQ NORMAL | ANY | ANY | ANY | ANY | ANY | NOT_APPLICABLE, null |
| 1 R01 | EQ ABNORMAL | ANY | ANY | ANY | EQ CONFIRMED | ANY | ASSESSED, CRITICAL |
| 2 R02 | EQ ABNORMAL | EQ BROAD | EQ CONFIRMED | EQ ADMIN | ANY | ANY | ASSESSED, CRITICAL |
| 3 R03 | EQ ABNORMAL | EQ BOUNDED | EQ CONFIRMED | EQ ADMIN | ANY | ANY | ASSESSED, HIGH |
| 4 R04 | EQ ABNORMAL | IN BROAD, BOUNDED | EQ CONFIRMED | EQ IMPORTANT | ANY | ANY | ASSESSED, HIGH |
| 5 R05 | EQ ABNORMAL | IN BROAD, BOUNDED | EQ CONFIRMED | EQ LIMITED | ANY | ANY | ASSESSED, MEDIUM |
| 6 R06 | EQ ABNORMAL | EQ BROAD | EQ CONFIRMED | EQ MINIMAL | ANY | ANY | ASSESSED, MEDIUM |
| 7 R07 | EQ ABNORMAL | EQ BOUNDED | EQ CONFIRMED | EQ MINIMAL | ANY | ANY | ASSESSED, LOW |
| 99 R99 | ANY | ANY | ANY | ANY | ANY | ANY | UNASSESSED, null |

평가 순서는 `(ordinal,rule_id)`로 고정하며 `ordinal`은 정책 안에서 유일하다. 첫 일치 결과에서 종료한다. R99는 마지막의 유일한 전역 fallback이다. 여러 규칙에 일치하는 경우가 있어도 더 높은 등급을 검색해 선택하지 않는다. R01과 R02의 중첩은 선언된 우선순위로 처리한다. 활성화 검사는 의도하지 않은 도달 불가 규칙, R99 뒤 규칙, enum/결과 불일치, `UNKNOWN`이 확인값을 대신하는 조건을 거부한다.

정규화 결과의 canonical 직렬화는 키 정렬, enum 대문자, ID 정렬·중복 제거, UTC 시각 정밀도, `null`의 의미를 버전으로 고정한다. `input_digest=SHA-256(canonicalInput)`이며 정책 내용 hash·engine/normalizer/freshness/mapping 버전·기준 시각·부모 head ID를 포함한다. 평가의 서버 수신 시각·worker ID는 digest에서 제외한다. 같은 digest의 AUTO 결과·규칙 ID·상태·사유 배열은 같아야 한다. 사유 배열도 enum 순서로 정렬한다.

R99는 `EXPOSURE_UNKNOWN`, `EXPOSURE_RANGE_UNKNOWN`, `CREDENTIAL_UNCONFIRMED`, `IMPACT_UNKNOWN`, `IDENTITY_BINDING_UNRESOLVED`, `PERMISSION_MAPPING_INCOMPLETE`, `TARGET_COVERAGE_INCOMPLETE`, `ASSET_CLASSIFICATION_UNKNOWN`, `UNSUPPORTED_IMPACT_ANALYSIS`, `STALE_EVIDENCE`, `CONFLICTING_EVIDENCE`, `EXPOSURE_INCIDENT_CONFLICT` 등 확인된 사유만 보존한다. R00/R01이 요구하지 않는 축의 누락을 평가 차단 사유로 만들지 않는다. 사유·추가 조사·다음 행동은 서버 enum과 템플릿에서 생성한다.

## 4. 네 테이블의 물리 계약

모든 PK는 `BIGINT GENERATED ALWAYS AS IDENTITY`, 시각은 `TIMESTAMPTZ`, 사용자 FK는 `dashboard_users`다. 별도 허용 외에는 `ON DELETE RESTRICT`를 사용한다. 테이블별 `CHECK`는 같은 행의 타입·조합을 강제한다. 다른 행·다른 테이블·현재 ACL을 조회하는 조건은 일반 CHECK로 표현하지 않고 복합 FK 또는 지연 constraint trigger와 권한 제한 쓰기 트랜잭션으로 강제한다. 원문 키·토큰·원천 본문·전체 공급자 응답·자유 문장을 어느 컬럼에도 저장하지 않는다.

### 4.1. `secret_severity_policies`

| 컬럼 | 타입·제약 |
|---|---|
| `policy_id` | BIGINT PK |
| `policy_version` | VARCHAR(80) NOT NULL UQ. 최초 규칙 집합은 `severity-v2` |
| `state` | VARCHAR(10) NOT NULL CHECK IN (`DRAFT`,`ACTIVE`,`RETIRED`) |
| `condition_schema_version`, `engine_version`, `normalizer_version`, `freshness_profile_version` | VARCHAR(80) NOT NULL |
| `freshness_profile`, `permission_mapping_manifest` | JSONB NOT NULL. 등록 schema로 검증된 선언 데이터 |
| `content_hash` | BYTEA NOT NULL CHECK octet_length=32. 규칙 전체와 모든 선언 포함 |
| `created_by_user_id`, `approved_by_user_id` | BIGINT FK. 작성자 NOT NULL, 승인자는 활성화 이후 NOT NULL |
| `created_at`, `approved_at`, `activated_at`, `retired_at` | created_at NOT NULL, 나머지는 수명주기별 필수 |
| `valid_from`, `valid_until` | TIMESTAMPTZ. 활성화 이후 valid_from 필수, 종료가 있으면 시작보다 뒤 |

`ACTIVE`는 승인자·승인 시각·활성화 시각·valid_from 필수, `approved_by_user_id<>created_by_user_id`, retired_at NULL이다. `RETIRED`는 승인/활성화 이력과 retired_at 필수다. `DRAFT`는 activated_at·retired_at NULL이다. 전역 평가 정책은 `UNIQUE ((true)) WHERE state='ACTIVE'`로 최대 하나다. 현재 적용 시작·종료와 활성화 트랜잭션에서 공백/중첩을 검사하며 단순 타임스탬프만으로 ACTIVE가 자동 생성되지 않는다.

초안은 배포 관리 경계에서 편집할 수 있다. 승인 시 계산한 content_hash와 승인 대상 hash가 같아야 한다. 활성화 이후 규칙·선언·작성자·승인·hash는 불변이다. 승인된 버전의 오타 수정도 같은 `policy_version`의 내용 변경으로 처리하지 않는다. 허용된 state·retired_at·valid_until 전이만 관리용 DB 역할이 수행하고 기존 감사 원장에 이전/새 상태와 hash를 남긴다. 규칙은 PR과 권한을 분리한 운영 승인을 통해 배포하며 임의 웹 규칙 편집 API를 제공하지 않는다.

### 4.2. `secret_severity_rules`

| 컬럼 | 타입·제약 |
|---|---|
| `severity_rule_id` | BIGINT PK |
| `policy_id` | BIGINT NOT NULL FK policies |
| `rule_id`, `ordinal` | VARCHAR(10) NOT NULL, SMALLINT NOT NULL CHECK ordinal>=0 |
| `conditions` | JSONB NOT NULL. 3.2절의 정확한 여섯 축·EQ/IN/ANY schema |
| `required_evidence` | JSONB NOT NULL. 등록된 근거 역할·종류·완전성 요구만 선언 |
| `result_status`, `result_severity`, `reason_code` | VARCHAR(20) NOT NULL, VARCHAR(10) nullable, VARCHAR(80) NOT NULL |
| `created_at` | TIMESTAMPTZ NOT NULL |

`UQ(policy_id,rule_id)`와 `UQ(policy_id,ordinal)`를 둔다. 상태·등급 조합은 1절과 동일 CHECK다. R00은 NOT_APPLICABLE/null, R99는 UNASSESSED/null, R01~R07은 ASSESSED/허용 등급이어야 한다. `severity-v2`의 ID·순서·조건·결과는 3.2절과 정확히 일치해야 한다. 등록 schema 검증과 활성화 검증으로 임의 필드·코드·외부 참조·조건 누락을 거부한다. 부모가 활성화된 후 INSERT·UPDATE·DELETE를 차단한다.

### 4.3. `secret_risk_evidence`

| 컬럼 | 타입·제약 |
|---|---|
| `evidence_id` | BIGINT PK |
| `record_kind` | VARCHAR(12) NOT NULL CHECK IN (`ASSERTION`,`REVIEW`,`RETRACTION`) |
| `kind` | VARCHAR(30) NOT NULL. 2.1절의 다섯 종류 |
| `occurrence_id`, `credential_set_id`, `scope_id` | BIGINT NOT NULL FK. 실제 동일 위치·키·범위 관계를 지연 constraint trigger로 검사 |
| `related_evidence_id` | BIGINT self FK. REVIEW/RETRACTION의 원 ASSERTION. ASSERTION에서는 NULL |
| `source_kind`, `verification_level` | VARCHAR(30) NOT NULL. 승인 워커·자산 분류·수동 제출/수용 구분 |
| `source_refs`, `identity_evidence_refs`, `safe_payload`, `limitations`, `missing_reason_codes` | JSONB NOT NULL. 타입별 닫힌 schema, 빈 경우 명시적인 배열/객체 |
| `provider`, `credential_type` | VARCHAR(40), VARCHAR(80). PERMISSION_MAPPING에는 NOT NULL |
| `collector_connection_id`, `binding_validation_attempt_id` | BIGINT FK. 전자는 관측 수집 연결, 후자는 발견 키 신원 결합의 검증 원본 참조이며 유효성 결과를 복제하지 않음 |
| `collector_identity_ref`, `credential_identity_ref`, `identity_binding_status` | VARCHAR(120), VARCHAR(120), VARCHAR(20). 보호된 불투명 참조만 저장 |
| `normalizer_version`, `mapping_version`, `asset_classification_version` | VARCHAR(80). 종류별 필수 여부를 CHECK/schema로 검증 |
| `observed_at`, `recorded_at`, `effective_from`, `expires_at` | observed_at·recorded_at NOT NULL, 현재 주장에는 effective_from·expires_at 필수. 역사적 사건은 observationWindow 필수이며 expires_at nullable |
| `submitted_by_user_id`, `reviewed_by_user_id`, `writer_service_id` | 사용자 FK와 VARCHAR(120) 서비스 식별자. 인간/서비스 생성 경로별 배타 제약 |
| `review_decision`, `reason_code` | VARCHAR(10) nullable CHECK IN (`ACCEPT`,`REJECT`), VARCHAR(80) NOT NULL |
| `safe_payload_digest`, `ingestion_key` | BYTEA NOT NULL CHECK 길이 32, VARCHAR(120) NOT NULL |

ASSERTION은 related_evidence_id·review_decision NULL이며 타입별 사실을 갖는다. REVIEW는 같은 위치·키·kind의 ASSERTION, reviewed_by_user_id, ACCEPT/REJECT와 검토 시각을 요구하고 새 주장 payload를 갖지 않는다. RETRACTION은 원 주장과 사유·수행자를 요구하며 review_decision NULL이다. 대상 근거와 참조 순환은 금지한다. REVIEW/RETRACTION을 다시 REVIEW 대상으로 삼지 않는다. 수용된 현재 수준은 원 ASSERTION과 후속 REVIEW/RETRACTION에서 파생한다.

수동 ASSERTION의 일반 자료는 해당 범위 SECURITY가 직접 제출·수용하여 `MANUAL_ACCEPTED`로 기록할 수 있다. 독립 검토는 일반 자료마다 요구하지 않는다. 할당된 REMEDIATOR는 `MANUAL_SUBMITTED` 자료를 제출하고 해당 범위 SECURITY의 ACCEPT가 있어야 영향 근거로 사용한다. OPERATIONS는 허용된 범위 조사·자료 보완을 수행할 수 있으나 SECURITY의 영향 수용 권한을 얻지 않는다. 수동 수용은 공급자 직접 관측으로 승격되지 않으며 `source_kind`와 원 제출 수준을 보존한다. 권한이 있는 워커의 승인 capability만 `PROVIDER_OBSERVED`를 생성할 수 있다.

ASSERTION의 현재 권한·자산 주장은 유한 `expires_at`을 요구한다. 수용 시각이 원래 관측 시각이나 만료를 연장하지 않는다. 기간 연장은 새 관측·새 ASSERTION을 필요로 한다. 역사적 사건은 유한 TTL의 대상이 아니며 취소·반증 경로로만 교정한다. 수동 근거의 권한, 수용 여부, 효력 만료는 각각 독립적으로 검사한다.

`UQ(scope_id,source_kind,ingestion_key)`를 둔다. 같은 키에 같은 canonical payload와 sourceRefs이면 기존 evidence_id를 반환하고 다른 내용이면 충돌이다. 모든 행은 추가 전용이다. 일반 서비스 역할의 UPDATE·DELETE를 금지하며 수정·거절·철회는 새 행으로만 남긴다.

### 4.4. `secret_severity_assessments`

평가 결과와 승인·적용 이벤트를 같은 추가 전용 원장에 저장한다. 아래 네 `record_kind`는 닫힌 합 타입이며 의미 없는 NULL 조합을 허용하지 않는다. 현황 head는 APPLICATION 행에서 파생하므로 별도 현재 평가 테이블을 만들지 않는다.

| 공통 컬럼 | 타입·제약 |
|---|---|
| `assessment_id` | BIGINT PK |
| `record_kind` | VARCHAR(20) NOT NULL CHECK IN (`AUTO_RESULT`,`OVERRIDE_PROPOSAL`,`REVIEW`,`APPLICATION`) |
| `occurrence_id`, `credential_set_id`, `scope_id` | BIGINT NOT NULL FK. 동일성·소속 관계 검사 |
| `occurrence_sequence` | BIGINT NOT NULL CHECK >0. UQ(occurrence_id,occurrence_sequence), 위치 잠금 아래 할당 |
| `policy_id`, `matched_rule_id` | BIGINT FK policies, VARCHAR(10). AUTO_RESULT의 규칙 매칭에는 복합 FK(policy_id,matched_rule_id)→rules(policy_id,rule_id) |
| `assessment_status`, `severity` | VARCHAR(20), VARCHAR(10). 결과 행에만 1절 조합 CHECK 적용 |
| `input_snapshot`, `evidence_refs`, `reason_codes`, `missing_reason_codes` | JSONB. 결과 행의 immutable canonical 입력과 typed 근거 참조. 결과 외 행은 별도 schema |
| `input_digest` | BYTEA. 결과 행에서 NOT NULL CHECK 길이=32 |
| `evaluation_as_of`, `created_at` | TIMESTAMPTZ. 결과 행의 evaluation_as_of 필수, created_at 공통 NOT NULL |
| `evidence_revision`, `credential_revision`, `scope_revision` | BIGINT. 평가·제안·적용의 비교 기준이며 결과 행에서는 NOT NULL |
| `base_auto_assessment_id`, `target_assessment_id`, `previous_effective_assessment_id`, `previous_application_id` | BIGINT self FK. 각 record_kind별 역할 고정 |
| `decision`, `reason_code` | VARCHAR(25), VARCHAR(80). 승인/적용 타입별 허용 enum |
| `actor_user_id`, `writer_service_id` | 사용자 BIGINT FK, 서비스 VARCHAR(120). 타입별 행위 주체 |
| `expected_evidence_revision`, `expected_current_application_id` | BIGINT. 검토·적용 TOCTOU 검사 기준 |
| `override_expires_at`, `approval_id` | TIMESTAMPTZ, BIGINT self FK. 수동 적용에는 유효한 독립 REVIEW 참조 |
| `request_job_id`, `command_id` | BIGINT FK secret_jobs, secret_command_idempotency. 자동 작업·수동 명령별 참조 |

| record_kind | 필수·금지 조합과 효력 |
|---|---|
| `AUTO_RESULT` | policy_id·입력 snapshot·digest·revision·기준 시각·writer_service_id·matched_rule_id 필수, actor_user_id NULL. base/target/previous/decision/approval/override_expires_at 모두 NULL. METADATA_ONLY 제외 행은 만들지 않음. 계산 사실이며 저장만으로 현재 등급을 바꾸지 않음 |
| `OVERRIDE_PROPOSAL` | 해당 위치 SECURITY actor, base_auto_assessment_id와 당시 effective 참조, 같은 정책·입력 기준, 제안 상태/등급·근거·사유·유한 override_expires_at 필수. ASSESSED는 허용 등급을 요구한다. NOT_APPLICABLE/null은 base_auto가 R00인 정상 보관 교정의 독립 하향 검토에만 허용한다. UNASSESSED를 수동 하향 목표로 사용하지 않는다. matched_rule_id NULL, decision/approval_id NULL. 자동 결과를 수정하거나 자동 규칙 ID를 가장하지 않음. 승인 전 효력 없음 |
| `REVIEW` | OVERRIDE_PROPOSAL의 target_assessment_id, actor_user_id, decision=`APPROVE\|REJECT`, reason_code, expected_evidence_revision·expected_current_application_id 필수. 제안자와 다른 해당 범위 SECURITY. assessment_status·severity·matched_rule_id·input_digest·override_expires_at·approval_id NULL. 제안 전체의 hash·기간·부모 입력을 검토하며 내용 수정은 새 제안으로 처리 |
| `APPLICATION` | AUTO_RESULT 또는 OVERRIDE_PROPOSAL의 target_assessment_id, previous_application_id, 이전 effective 참조, 현재 비교 revision·reason_code·writer_service_id 필수. decision=`APPLY\|HOLD\|EXPIRE_OVERRIDE`. 상태·등급·matched_rule_id·input_digest NULL. 승인된 override APPLY에는 approval_id 필수. 자동 APPLY에는 approval_id NULL. HOLD는 이전 effective를 유지하며 대상 계산을 숨기지 않음 |

최초 APPLICATION은 previous_application_id·previous_effective_assessment_id·expected_current_application_id가 모두 NULL일 수 있다. 이후에는 세 값의 실제 현재 head·동일 위치를 검사한다. target·approval·base_auto는 같은 위치·키이며 승인 대상의 동일한 정책·입력 snapshot을 가리켜야 한다. previous_application_id·previous_effective_assessment_id는 같은 위치·키의 실제 이전 적용을 참조하며 정책 교체 전의 다른 policy_id를 허용한다. 참조 순환을 금지한다. APPLY가 가리키는 proposal과 APPROVE가 검토한 proposal은 같아야 한다. 정책에서 허용하지 않은 UNKNOWN→등급 변환을 수동으로 할 때도 영향 근거와 독립 검토를 생략하지 않는다.

`UQ(occurrence_id,policy_id,input_digest) WHERE record_kind='AUTO_RESULT'`로 자동 중복을 막는다. `UQ(target_assessment_id) WHERE record_kind='REVIEW'`로 하나의 제안은 한 번만 최종 승인/거절한다. 적용 연결은 `UQ(previous_application_id) WHERE record_kind='APPLICATION' AND previous_application_id IS NOT NULL`과 `UQ(occurrence_id) WHERE record_kind='APPLICATION' AND previous_application_id IS NULL`로 분기를 막는다. 반복 HOLD도 기존 head를 부모로 하는 새 행이며 동일 명령/작업 재시도는 멱등 원장으로 기존 결과를 반환한다. 모든 행의 UPDATE·DELETE를 금지한다.

### 4.5. 참조 검증과 색인

typed JSONB 참조는 기존 원장의 참조 방식에 맞추되 적재 시 허용 type/version, 실제 부모, 동일 위치/키, 현재 범위 ACL을 검증하는 지연 constraint trigger를 둔다. assessment snapshot에는 당시 사용한 안전한 값과 부모 ID·version·digest를 함께 고정한다. 부모의 현재 투영이 변해도 과거 입력을 재작성하지 않는다. 보존 중인 근거·평가의 참조 부모를 삭제하지 못하게 한다. 개인의 열람 권한 취소는 자료 삭제가 아니며 모든 GET에서 현재 ACL을 다시 적용한다.

| 조회·검사 | 색인 |
|---|---|
| 정책 선택·규칙 조회 | policies(policy_version) UQ, 단일 ACTIVE 부분 UQ, rules(policy_id,ordinal) UQ |
| 위치별 근거 | risk_evidence(occurrence_id,kind,observed_at DESC,evidence_id DESC), (related_evidence_id,evidence_id DESC) |
| 만료 재평가 | risk_evidence(expires_at,occurrence_id) WHERE record_kind='ASSERTION' AND expires_at IS NOT NULL |
| 키별 허용 범위 근거 | risk_evidence(credential_set_id,scope_id,kind,evidence_id DESC), ACL scope를 선적용 |
| 평가 이력·최신 자동 | assessments(occurrence_id,occurrence_sequence DESC), (occurrence_id,evaluation_as_of DESC,assessment_id DESC) WHERE record_kind='AUTO_RESULT' |
| 현재 적용 head | assessments(occurrence_id,assessment_id DESC) WHERE record_kind='APPLICATION', previous_application_id 부분 UQ |
| 승인 대기·만료 | assessments(scope_id,created_at,assessment_id) WHERE record_kind='OVERRIDE_PROPOSAL', (override_expires_at,occurrence_id) WHERE record_kind='OVERRIDE_PROPOSAL' |

JSONB 전체에 무조건 GIN을 만들거나 보호된 원천 참조·키 신원을 검색용 일반 색인으로 복제하지 않는다. current head와 대기 제안은 위 색인과 typed 참조로 계산한다. 필요하면 기존 조회 스냅샷에 결과 ID를 포함하되 별도 mutable severity 원장을 만들지 않는다.

## 5. 예약·멱등성과 TOCTOU

평가 요청은 기존 작업 큐를 재사용한 `kind=ASSESS`, `purpose=SEVERITY_ASSESSMENT`, `target_occurrence_id`, `expected_evidence_revision`을 갖는다. DB의 확정 사실만 읽으며 수집·탐지·외부 검증을 실행하거나 ScanRun을 만들지 않는다. `requestId=jobId`, `runId=null`이다. FULL_SCAN/REVALIDATE의 수집·탐지·검증 단계와 혼합하지 않는다. ASSESS의 대상·purpose 조합, scope/connection 소속은 작업 적재 시 검사한다.

`evidenceRevision`의 물리 원본은 `secret_occurrences.risk_evidence_revision BIGINT NOT NULL DEFAULT 0 CHECK (risk_evidence_revision>=0)`다. 입력에 영향을 주는 확정 변경 때만 해당 위치를 잠그고 1 증가시키며 원본 사실·재평가 예약과 같은 트랜잭션에서 확정한다. 평가/승인/적용 행을 추가한 사실 자체로 이 counter를 증가시키지 않는다. 평가 이력 순서인 `occurrence_sequence`와 구분한다. 정책·키 공통 근거 등 여러 위치에 영향을 주는 변경은 영향 위치 각각에 새 revision을 부여하며, fan-out이 완료되지 않은 위치에는 policy/source head 불일치 차단을 유지한다. 숫자 revision과 sourceHeadRefs digest를 함께 고정하므로 전파 중인 부모 변경을 옛 revision만으로 통과시키지 못한다.

키 구성·동일성의 물리 revision은 `secret_credential_sets.credential_revision BIGINT NOT NULL DEFAULT 1 CHECK (credential_revision>0)`다. 구성 승격·키 동일성 교정은 이를 증가시키고 관련 위치의 risk_evidence_revision을 갱신한다. scope_revision은 승인된 수집 범위 revision이며 출처 승인·보관 정책의 version은 sourceHeadRefs에 따로 포함한다. 어느 하나를 다른 변경을 대표하는 version으로 대체하지 않는다.

1. 근거·수용/취소·노출 판정·키 동일성·권한/자산/접근 정책·유효성·범위 revision 변경은 원본 사실을 먼저 확정하고 관련 위치의 evidence revision을 증가시킨다. 같은 트랜잭션에서 ASSESS 작업을 멱등 예약한다. 만료 도래와 정책 활성화는 대상별 재평가 작업을 예약하며 실행 예약 실패를 감사·재시도로 보존한다.
2. `(actor,commandKind,idempotencyKey)`와 안전한 요청 digest는 기존 `secret_command_idempotency`를 사용한다. 같은 키·같은 본문은 같은 작업/결과, 다른 본문은 409다. 자동 이벤트는 `(eventType,eventId,occurrenceId,expectedEvidenceRevision,policyId)`를 canonical화한 안전한 멱등 키를 사용한다. ASSESS 활성 작업은 동일 위치·기준 revision·정책에서 하나로 합친다. 별도 scope 전체 수집 lock으로 ASSESS를 막지 않는다.
3. worker는 lease로 선점하고 DB snapshot에서 활성 정책과 모든 부모 head·입력·기준 시각을 고정한다. 외부 호출은 없다. replay 작업은 ACTIVE 선택 대신 명시한 불변 policy_id를 사용한다.
4. 결과 적재 직전 해당 Occurrence 행과 현재 APPLICATION head를 잠그고 정책 활성 버전, 위치·키·범위·evidence revision, sourceHeadRefs digest, 수동 승인 유효 기간·현재 권한을 다시 비교한다. ACL 확인과 상태 변경 사이의 경쟁은 이 트랜잭션 경계에서 처리한다. 자동 적재 서비스 권한도 당시 capability를 재검사한다.
5. 비교가 달라졌으면 기존 snapshot 결과를 현재 head로 적용하지 않는다. 결과를 역사적 AUTO_RESULT로 남기는 경우에도 APPLICATION에는 `HOLD/STALE_BASELINE`을 기록하고 최신 기준 작업을 예약한다. 동일 작업 재시도는 AUTO UQ와 멱등 결과를 사용한다. 동시 worker는 이전 APPLICATION의 유일성 및 CAS 검증 때문에 둘 다 새 head가 될 수 없다.
6. 통과한 결과·허용된 APPLICATION·작업 완료·감사·조회 스냅샷 무효화는 원자 확정한다. 실패한 트랜잭션의 부분 결과를 API 성공으로 반환하지 않는다. 사용자 GET은 외부 API·자동 상태 변경 없이 저장된 결과와 현재 유효성/만료 사유를 읽는다.

요청된 기대 revision이 이미 낡으면 무조건 최신 사실을 대신 평가했다고 성공 처리하지 않는다. 409 또는 안전한 `STALE_BASELINE`과 재요청 정보를 반환한다. 정책 활성화나 근거 추가의 fan-out은 같은 이벤트의 중복 예약을 막고 전체 대상 누락을 추적한다. 큐 임대·재시도·실패 감사는 기존 작업 원장을 사용한다.

## 6. 적용·수동 override·하향 통제

자동 계산은 언제나 저장한다. 기존 effective 등급이 없거나 새 결과가 같은 등급/상향이면 승인된 정책 아래 APPLY할 수 있다. 상향도 근거가 없는 상태를 임의 등급으로 바꾸지 않으며 충돌을 먼저 해결한다. 각 적용에는 이전/새 결과와 policy·rule 차이를 남긴다.

자동 결과가 이전 effective 등급보다 낮거나 `UNASSESSED`/`NOT_APPLICABLE`이면 `APPLICATION/HOLD`로 이전 적용 등급을 유지하고 최신 계산과 원인을 별도 표시한다. `VALID→INVALID`, 검사 실패·미지원·만료, 단순 삭제·예외 승인·체크리스트 완료, 로그 부재, 정책 교체만으로 기존 높은 등급을 하향하지 않는다. 이전 NORMAL에서 새 R99가 된 경우에도 정상 확정으로 계속 표시하지 않고 현재 미확인과 마지막 NORMAL 근거를 구분한다.

수동 override는 해당 위치 SECURITY가 자동 결과·현재 effective·근거 revision에 결합한 제안을 만들고 다른 현재 범위 SECURITY가 승인한다. 상향에는 추가 영향/사고 증거, 하향에는 이전 높은 등급의 근거가 해당 평가 대상·시점에 더 이상 적용되지 않는다는 구체적 반증이 필요하다. 단순 수용 가능한 위험, 처리 편의, 현재 인증 실패는 반증이 아니다. 제안의 대상·상태/등급·근거·사유·만료를 검토 후 수정할 수 없으며 바꾸려면 새 제안이다.

정상 보관으로 교정한 R00이 이전 등급을 낮추는 경우는 R00 AUTO_RESULT를 base_auto_assessment_id로 고정한 NOT_APPLICABLE/null 제안으로 검토한다. API의 basisAssessmentId는 이 자동 평가를 지정하고 targetAssessmentStatus와 nullable targetSeverity는 제안 결과를 나타낸다. 수동 제안만으로 NORMAL 노출 판정을 만들거나 R99 미평가를 정상 보관으로 바꾸지 않는다.

승인 시와 APPLY 직전에 각각 제출자와 다른 검토자, 현재 범위 SECURITY, 원 근거의 수용·기간·동일성, 정책·source heads·기대 head를 확인한다. 승인 후 기준이 변하면 승인을 다른 snapshot에 이식하지 않고 새 제안/검토를 요구한다. 모든 override는 유한 만료를 가지며 자동 결과와 별도로 조회한다. 일반 수동 자료 수용을 override 승인으로 재사용하지 않는다.

만료·철회·범위 권한 변경·상충 근거가 발생하면 override 효력이 끝난 사실과 `EXPIRE_OVERRIDE`를 추가한다. 이 행의 target은 만료/종료된 OVERRIDE_PROPOSAL이고 승인 효력을 다시 만들지 않는다. 같은 트랜잭션에서 현재 유효한 AUTO_RESULT를 대상으로 후속 APPLY 또는 HOLD를 추가해 새 head를 확정한다. 자동 결과가 이전 확정 높은 등급을 낮추는 경우에는 별도 `DOWNGRADE_REVIEW_REQUIRED` hold와 독립 하향 검토를 유지하고, 만료된 override가 계속 승인 상태라고 표시하지 않는다. 만료만으로 낮은 등급이나 정상 보관을 확정하지 않는다. 명시적 승인 없이 override를 연장하지 않는다.

사고 당시 R01 CRITICAL은 현재 키 폐기로 지우지 않는다. `NORMAL+CONFIRMED abuse` 충돌은 R99 최신 계산과 기존 CRITICAL hold를 함께 표시한다. 사건 연결이 잘못됐다는 반증의 독립 검토가 없으면 incident 등급을 낮추지 않는다. 위치가 기존 대응 정책의 해결 조건을 충족해 해결되면 현재 건의 미해결 위치 집합에서 제외되지만 과거 평가·사고 사실은 유지한다. 이 제외는 심각도 재작성과 다르다.

## 7. 정책 활성화·retired·replay

정책 활성화는 규칙 schema·필수 fallback·경로별 요구 근거·명시된 정상/사고 충돌 처리·최신성 숫자·지원 mapping manifest와 승인 hash를 검증한다. 기존 활성 정책의 RETIRED 전이와 새 ACTIVE 전이는 같은 트랜잭션에서 수행하고 기존 감사 원장에 승인자·시각·변경 이유·이전/새 hash를 남긴다. 승인자에게 해당 운영 정책 권한이 필요하며 일반 Secret 화면의 ADMIN 표시만으로 이를 부여하지 않는다.

재평가는 새 정책 아래의 새 AUTO_RESULT다. 이전 정책 ID·rule ID·입력·결과를 덮어쓰지 않는다. 결과가 하향이면 정책이 승인됐어도 위치별 하향 통제를 적용한다. `RETIRED` 정책은 신규 current 계산 대상으로 선택할 수 없지만 과거 평가의 replay에는 사용할 수 있다. retired의 규칙·매핑·normalizer·engine·freshness 선언은 보존한다.

`replay`는 과거 canonical snapshot, 당시 `evaluationAsOf`, 정확한 정책/engine/normalizer/mapping/freshness 버전으로 수행한다. 외부 공급자 재호출이나 최신 DB head 치환 없이 결과·규칙·사유·digest를 비교한다. replay는 current head를 바꾸지 않는다. 필요한 실행물/버전/원본 참조가 보존되지 않았으면 `REPLAY_UNAVAILABLE`을 반환하고 현재 알고리즘으로 같은 결과라고 주장하지 않는다. 현재 근거로 다시 평가하는 명령은 replay가 아니라 신규 평가다.

## 8. 읽기·명령과 대표 심각도

구체 경로와 응답 모양은 [API 계약](api-contract.md)에 따른다. 위험 조회는 안전한 여섯 축·관측 시각·정책/규칙·평가 상태·등급·사유·누락·근거 수용 수준·지원 완전성·최신 자동/현재 적용/마지막 확정을 구분한다. 이력은 결과와 제안·검토·적용의 차이를 드러낸다. 미열람 근거의 보호된 참조·이름·정확한 수를 차감 값으로 유출하지 않는다.

근거 제출은 타입·안전 참조·대상·예상 revision·멱등 키만 받는다. 공급자 관측 수준·임의 유효성·자동 축 값·임의 규칙을 입력할 수 없다. 평가 요청은 현재 DB 사실의 ASSESS 작업을 만들며 요청 접수는 등급 산정 완료가 아니다. 자료 제출·일반 수용·override 검토·자동 평가·현재 적용의 성공과 남은 일을 각각 반환한다. 미지원·근거 부족은 해당 범위 조사·자료 보완 경로를 제공하며 새 역할이나 권한을 자동 부여하지 않는다.

대응 건의 대표 등급은 같은 snapshot의 미해결 위치 중 현재 적용 가능한 확인 등급의 최고값이다. 순서는 `CRITICAL>HIGH>MEDIUM>LOW`다. 최신 자동 결과만 미평가인 위치도 기존 높은 등급 HOLD가 유효하면 대표 등급에 반영하면서 `unassessedOccurrenceCount`와 `heldOccurrenceCount`에 그 상태를 드러낸다. 미평가 수는 최신 계산의 UNASSESSED 위치 수이며 hold 수와 중복될 수 있으므로 둘을 더해 전체 위치 수라고 표시하지 않는다. 모든 미해결 위치가 미평가이고 적용 가능한 hold도 없으면 대표는 UNASSESSED다.

정상 보관·메타데이터·해결 위치는 현재 위험 대표의 미해결 비정상 위치 분모에 넣지 않는다. 해결 위치의 역사적 등급은 이력에서 조회한다. 위치가 없는 건을 LOW로 만들지 않는다. 일부 위치만 열람 가능한 사용자는 visible 범위의 집계와 그 한계를 받으며 숨겨진 위치의 등급·수나 전체 해결 가능성을 추론할 수 없다. 현황 분포는 현재 대응 건 단위로 한 번씩 집계하고 키/위치/과제 수와 합산하지 않는다.

## 9. 결정성과 경계를 확인하는 검증 명세

검증 입력은 합성 식별자·안전한 fixture·승인된 schema만 사용한다. 실제 비밀이나 외부 공급자 호출은 평가기 검증의 필요 조건이 아니다.

| 검증 | 필수 기대 결과 |
|---|---|
| R00~R07·R99 골든 케이스 | 3.2절의 최초 일치 ID·상태·등급을 정확히 재현 |
| R00 예외 | 유효 NORMAL이고 나머지 축 미확인 → R00/NOT_APPLICABLE. 전 축 COMPLETE를 요구하지 않음 |
| R01 예외 | ABNORMAL+연결된 중대한 악용, credential=CANDIDATE, range/impact=UNKNOWN, validity=INVALID/UNCHECKED → R01/CRITICAL |
| 정상·사고 충돌 | NORMAL+동일 위치 연결 CONFIRMED abuse → R99/UNASSESSED, EXPOSURE_INCIDENT_CONFLICT. 자동 ABNORMAL 없음, 기존 CRITICAL hold와 이력 보존 |
| 범위 UNKNOWN | R02~R07의 다른 값이 완전해도 R99. UNKNOWN을 BOUNDED 또는 ANY로 해석하지 않음 |
| 유효성 독립 | 같은 나머지 축에서 VALID/INVALID/UNKNOWN/UNCHECKED만 교체해도 자동 등급·규칙은 같음. 이전 사고 등급 하향 없음 |
| 메타데이터 | METADATA_ONLY GET은 NOT_APPLICABLE, assessmentId/ruleId null. 전용 ASSESS/AUTO_RESULT·CONFIRMED/CANDIDATE 자동 전환·건 생성 없음 |
| 공급자 지원 범위 | 인증만 가능한 GitHub 유형·권한/대상/제한 누락·미지원 권한 → impact UNKNOWN와 정확한 mapping 누락 사유 |
| 두 신원 혼동 | collection identity만 ADMIN, 발견 키 결합 미확인 → UNKNOWN. 다른 키의 성공 검증을 결합 근거로 사용하면 거부 |
| 접근 관측 갱신 | 승인 worker의 ACCESS_SCOPE는 같은 위치·객체·revision에서 현재 range만 갱신. 노출 재판정/자동 ABNORMAL·NORMAL·해결 없음, 과거 access_snapshot 불변. BOUNDED는 완전성 요구, 공개 접근의 양성 BROAD 관측은 보존 |
| 최소 영향 반증 | 테스트 이름·비운영 태그만으로 MINIMAL 금지. 권한/대상/제한/자산 분류의 완전한 근거 필요 |
| 악용 조사 | 로그 없음·조사 미시작·부분 커버리지는 NOT_CONFIRMED+조사 메타데이터. 악용 없음이나 R01 반증으로 변환 금지 |
| 최신성·충돌 | 고정 asOf 경계, expiresAt 직전/동일/직후, 부모 revision 변경, 시간 불확실성, 모순 근거를 재현. 역사적 사건의 일반 TTL 만료 금지 |
| 조건 parser | 알 수 없는 enum/필드·누락 축·빈 IN·UNKNOWN wildcard·임의 SQL/스크립트·R99 뒤 규칙·중복 ordinal을 거부 |
| 정책 수명주기 | 자기 승인·hash 변경·동시 ACTIVE·활성 규칙 수정 거부. RETIRED replay 허용, current 선택 금지 |
| 수동 자료 | SECURITY 직접 MANUAL_ACCEPTED 가능. REMEDIATOR 제출은 수용 전 영향 반영 금지. 수용으로 PROVIDER_OBSERVED 승격·만료 연장 없음 |
| override | 자기 승인·다른 위치 승인 재사용·기준 변경 후 APPLY·기간 만료 적용 거부. 하향은 이전 높은 등급에 대한 반증과 독립 승인 필수 |
| 멱등·경쟁 | 같은 이벤트/본문 재전송은 같은 결과. 다른 본문 같은 키는 충돌. 두 worker/근거 갱신/정책 교체 경쟁에서 head 분기·낡은 APPLY 없음 |
| append-only·부모 | 평가/근거 UPDATE·DELETE, 타 위치 FK/typed ref, 존재하지 않는 부모, 참조 순환·다른 키 검증 참조 거부 |
| replay | 순서만 다른 동등한 근거 배열의 digest·결과 같음. 원 snapshot replay 결과 일치, 최신 입력 재평가는 별도 기록 |
| 대표 등급·ACL | 여러 위치 최고 등급·미평가 수·hold 수·해결 위치 제외를 같은 snapshot에서 검증. 제한 위치의 수·등급 유출 없음 |
| 비밀 비저장 | payload·digest 입력·오류·큐·감사·평가 snapshot에 원문 키·토큰·본문·공급자 전체 응답·자유 URL 없음 |

규칙 fixture는 policy hash와 함께 보존한다. 모든 결과 검증은 등급뿐 아니라 rule ID·상태·누락 사유·typed 근거·최신성·적용 여부를 함께 확인한다. 등급이 우연히 같아도 잘못된 규칙이나 다른 위치의 근거가 선택되면 실패다.

## 10. 사실 적용 시점과 설명 투영

관측한 시각과 사실이 성립한 기간을 구분한다. `applicableFrom`/`applicableUntil`은 둘 다 없거나 유효한 UTC 구간이어야 한다. 수동 기간은 반개구간 [applicableFrom, applicableUntil)이며 from < until, from <= observedAt, until <= validUntil을 적용한다. 같은 위치 원문 관찰 시각을 포함하고 평가 기준시점에도 적용 가능해야 한다. 역사적 ABUSE_LINK는 현재 사실의 적용 기간 필수 조건을 요구하지 않고 기존 사건 시점·동일성·반증 요건을 유지한다. 서버는 원천 관측으로 `sourcePresenceObservedAt`과 `sourceObjectVersion`을 결합한다. 현재 수집기의 비동시 관측은 `OBSERVATION_ONLY`로 보존한다. 같은 run이나 콘텐츠 revision만으로 ACL과 원문의 동시성을 증명하지 않는다. 명시적인 수동 사실 효력 기간의 결합은 `EFFECTIVE_INTERVAL`로 기록한다. 서로 다른 시점의 공개 상태와 키 원문을 조합하지 않는다.

기존 근거에 새 시점·원천 필드가 없으면 추정하거나 수신·승인 시각으로 채우지 않는다. `EVIDENCE_APPLICABILITY_UNCONFIRMED`, `EVIDENCE_APPLICABILITY_MISMATCH`, `SOURCE_REVISION_UNCONFIRMED`로 해당 축의 적용 한계를 반환한다. 과거 평가·근거 원본과 높은 등급 보류는 보존하며 현재 정규화와 구분한다.

NORMAL과 같은 위치·시점의 BROAD가 충돌하면 `ACCESS_POLICY_CONFLICT`로 exposure UNKNOWN을 전달한다. NORMAL과 동일 노출의 사고가 충돌하면 기존 `EXPOSURE_INCIDENT_CONFLICT` 경계를 따른다. BOUNDED는 완전한 관련 접근 경계가 필요하지만 확인된 공개 접근의 양성 BROAD 근거는 일부 명단 누락으로 숨기지 않는다. 권한 없음 자료의 지원 매핑이 없으면 `MAPPING_UNSUPPORTED`이며 임의 MINIMAL을 만들지 않는다.

GET risk의 `riskExperience`는 등급 저장 원장을 바꾸지 않는 읽기 모델이다. 현재 근거 상태·부족 질문·다음 행동·업무 우선순위를 제공하며 [API 계약](api-contract.md)의 명시적 필드를 따른다. `work-priority-v1`은 심각도 정책과 별도이며 같은 업무의 최초 발생 기준에서 영업시간 목표를 계산한다. 공개 원문과 인증 성공/높은 권한 후보가 있어도 영향이 부족하면 등급은 미평가로 두고 우선 조사 이유를 표시한다.

일반 사용 관측과 AI 출력은 ABUSE_LINK로 저장하지 않는다. `incidentObservers`에 등록된 증거 생성기가 없는 배포는 자동 사고 확정을 비활성으로 유지한다. 수동 자료 제출도 필요한 동일 키·동일 노출·비인가 사용·중대한 영향 증거를 생략하지 못한다.

GitHub 파일 수집 뒤의 저장소 공개 메타데이터 조회는 원문과 ACL의 동시 관측을 입증하지 않는다. blob SHA는 파일 콘텐츠 버전이며 ACL 버전이 아니다. 이 관측은 공개 후보 사실로 보존하되 `ACCESS_COEXISTENCE_UNCONFIRMED`와 접근 범위 UNKNOWN을 반환한다. 공개 본문·접근 상태의 실제 동시 관측 또는 원문 관찰과 겹치는 명시적 수동 효력 기간이 확인될 때만 같은 시점의 공개 키 노출 근거로 수용한다. 공개 후보와 높은 권한 후보를 숨기지 않고 우선 확인 질문으로 보여준다.

평가 snapshot에는 `applicabilityVersion=risk-applicability-v1`을 고정한다. 이 적용 시점 해석 버전이 없는 기존 snapshot은 과거 기록으로 보존하되 현재 결과 적격성은 false이며 재평가가 필요하다. `severity-v2` 규칙을 바꾸거나 과거 snapshot을 새 해석으로 덮어쓰지 않는다.

## 11. 현재 설명과 부적격 근거의 구분

현재 인증은 기존 검증 조회의 freshness와 latestAttempt를 재사용한다. CURRENT가 아닌 과거 확정값을 현재 VALID/INVALID로 투영하지 않는다. 원래 결과와 lastConclusiveValidation은 역사로 유지하고 시도 없음은 UNCHECKED, 현재 결론 불가는 UNKNOWN으로 설명한다. 이 표시·업무 신호 처리는 등급의 validity 독립 규칙을 바꾸지 않는다.

정규화의 `excludedEvidence`에는 evidenceId·kind·reasonCode만 남긴다. 이 목록은 과거 근거의 제외 설명이며, 현재 모든 축의 보완 요구가 아니다. `missingFacts`는 현재 UNKNOWN/STALE/CONFLICT 축과 해당 kind에 한정한다. 새 적격 근거로 확인된 축은 이전 만료 근거 때문에 같은 보완 질문을 계속 표시하지 않는다. 현재 평가 자체가 낡았으면 근거 질문과 별도로 재평가 행동을 제공한다.

권한 후보에서 유래한 업무 신호는 조회 연결의 현재 ACL·활성·인증 참조·승인 범위 baseline과 정확한 키 주체 일치를 검증한다. 업무 기산은 현재 등급 전환 또는 현재 유효한 관측 신호의 연속 구간에서 구한다. 원천 공개성 관측과 인증/권한 후보의 겹침은 조사 우선순위 근거이며, 원문과 ACL이 실제 같은 시점에 공개됐다는 평가 근거를 대신하지 않는다.
