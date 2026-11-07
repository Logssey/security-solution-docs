# 사용 이력과 사고 근거 계약

기준: [API](api-contract.md), [데이터](data-contract.md), [DB 스키마](database-schema.md), [평가 계약](risk-assessment-contract.md). 인증 확인, 실제 사용, 비인가 사용과 중대한 사고는 서로 다른 사실이다.

사용 이력은 중대한 사고 입력을 대체하지 않는다. 다음 사실을 독립 필드로 저장한다.

| 필드 | 값과 의미 |
|---|---|
| observationStatus | OBSERVED / NOT_OBSERVED_IN_SCOPE / UNAVAILABLE / NOT_REQUESTED |
| bindingLevel | EXACT_CREDENTIAL / PRINCIPAL_ONLY / UNRESOLVED |
| coverageStatus | COMPLETE / PARTIAL / UNKNOWN. 특정 기간·계정·리전·이벤트 종류의 범위에 한정 |
| actionResult | SUCCESS / FAILURE / UNKNOWN |
| actorAttribution | OWN_VALIDATION / EXPECTED_WORKLOAD / UNATTRIBUTED. 주장에는 연결 근거 필요 |
| observationWindow | UTC 시작·종료, 수집 가능한 시작, 조회 누락·지연 이유 |
| subject | 보호된 키 참조 또는 주체 참조. 새 키에 이전 키 이력 승계 금지 |

사용 확인과 커버리지는 독립이다. 하나의 이벤트를 확인했어도 전체 조회는 PARTIAL일 수 있다. NOT_OBSERVED_IN_SCOPE는 명시된 범위의 조회를 끝냈을 때만 쓰며, ‘한 번도 사용 안 함’으로 번역하지 않는다.

### 초기 사용 이력 수집 범위

AWS IAM 최근 사용 메타데이터를 재사용한다. 마지막 사용 시각·서비스·리전은 해당 메타데이터로 표시하며 전체 이벤트 조회 결과와 합치지 않는다.

새 AWS 사용 조회는 관리용 읽기 연결에서 CloudTrail LookupEvents를 사용한다. 발견 키에 정확히 결합된 AccessKeyId 필터와 승인 계정·리전만 허용한다. HMAC은 역변환하지 않는다. 승인 원천에서 키 ID를 재취득해 기존 자격 증명과 일치를 검증하거나, 승인된 관리 연결의 IAM 키 목록에서 같은 형식의 키 ID 전용 보호 지문을 대조한다. 키 ID 전용 지문이 없으면 구성 전체의 지문으로 이를 대신하지 않는다. 실제 AccessKeyId는 실행 메모리에서만 사용하고 큐·결과·로그에 저장하지 않는다. 원천 제거·임시 키 ID 부재·동일성 불일치로 정확한 ID를 확보할 수 없으면 KEY_IDENTIFIER_UNAVAILABLE 또는 IDENTITY_BINDING_UNRESOLVED로 종료한다. 같은 IAM 사용자에게 속한 다른 키의 기록을 합치지 않는다.

기본 기간은 최근 7일, 수동 확장은 최근 90일 이내다. 50개 단위 페이지, 계정·리전당 초당 1요청, 요청당 최대 100페이지·5,000건을 처리한다. 상한, 접근 실패, 누락 리전, 조회 도중 변경은 PARTIAL 사유다. 관리 이벤트 조회이므로 S3 GetObject 같은 데이터 이벤트와 모든 사용을 확인했다고 표시하지 않는다.[CloudTrail LookupEvents](https://docs.aws.amazon.com/awscloudtrail/latest/APIReference/API_LookupEvents.html)

조회가 활성화된 연결만 정기 6시간에 최근 7일을 다시 확인하며 공급자 이벤트 ID로 중복을 제거한다. 수동 요청은 해당 키와 허용 범위에 한정한다. 원시 CloudTrailEvent는 메모리에서 허용 필드만 추출하고 저장·로그 출력하지 않는다. 감사 수집을 위해 trail·버킷·유료 Lake query를 자동 생성하지 않는다.

GitHub와 Notion은 인증 endpoint에서 사용 이력을 추론하지 않는다. 초기 자동 사용 이력 adapter는 미지원 상태로 고정하고, 승인된 관리 자료의 구조화된 수동 기록을 허용한다. 앱·사용자 수준 자료는 PRINCIPAL_ONLY로 표시한다. 향후 공급자 adapter 등록도 이 계약을 따라야 하며 현재 화면에서 실행 가능한 조회처럼 표시하지 않는다.

AWS 영구 키는 승인 IAM 목록의 ID를 키 ID 전용 보호 지문과 대조하고, 메타데이터 전용 키는 기존 보호 참조를 대조한다. 이전 수집에 ID 전용 지문이 없으면 원천 재수집이 필요하다. 영구 키의 보호 지문은 있지만 조회에 필요한 IAM 주체를 확인하지 못하면 FRESH_IDENTITY_VALIDATION_REQUIRED로 현재 인증 확인을 안내한다. 임시 자격 증명의 ID를 재취득할 경로가 없거나 IAM에서 삭제된 ID를 확보하지 못하면 KEY_IDENTIFIER_UNAVAILABLE로 남긴다. 이 경우 인증 미지원이나 전체 사용 이력 0건으로 표시하지 않는다. 조회에 쓰는 자격 증명은 별도 관리 연결이며 발견한 키로 CloudTrail을 조회하지 않는다.

수동 사용 자료는 sourceKind=MANUAL로 제출하며 job_id나 공급자 이벤트 ID를 요구하지 않는다. 허용된 내부 자료 참조, 해당 키/주체 결합 근거, 적용 기간·행위 종류·결과·자원 참조를 구조화하여 받는다. SECURITY 또는 해당 업무의 REMEDIATOR가 제출할 수 있고 SECURITY가 수용·거절한다. 제출자가 보호 지문·공급자 관측 출처를 직접 입력하거나 EXACT_CREDENTIAL을 주장하는 것만으로 결합을 확정하지 않는다. 서버가 동일성을 검증하지 못하면 PRINCIPAL_ONLY/UNRESOLVED를 유지한다. 수용·거절·철회는 별도 추가 전용 결정 이력이며 공급자 자동 관측으로 승격하지 않는다. 수동 사용 자료는 자동 사고 입력으로 사용하지 않는다.

자체 검증은 실제 실행·공급자 request reference·키 결합 근거로 구분한다. IP·시간이 비슷하다는 이유만으로 자체 검사로 제외하지 않는다. 다른 사용의 정상 여부도 업무 담당자의 근거 없이 자동 확정하지 않는다.

키의 비인가 사용은 확인됐지만 어느 위치에서 유출됐는지 모르면 ‘키 비인가 사용 확인 · 유출 경로 미확인’을 기록한다. 특정 위치의 R01에 임의로 연결하지 않는다. 중대한 사고 observer가 등록되지 않은 배포에서는 자동 abuse 확정 대신 독립 조사 상태와 우선순위로 전달한다.

## 실행·재시도·보존

`SECRET_USAGE_CLOUDTRAIL_ENABLED=false`가 기본이다. 별도로 승인된 관리 연결·계정·리전이 있어야 활성화한다. 사용 요청과 기존 `secret_jobs`의 `ENRICH/USAGE_OBSERVATION`을 같은 트랜잭션에 등록한다. 범위·기간·키 결합·커버리지·0건 요약은 전용 요청에 보존하고 기존 lease/fencing을 적용한다. 스캔 이력을 생성하지 않는다.

페이지별 일시 오류는 최대 2회, 1초·3초와 jitter로 재시도한다. 긴 Retry-After는 lease를 반환하고 재예약하며 인증·권한 거절은 재시도하지 않는다. 읽지 못한 페이지 뒤를 COMPLETE로 처리하지 않는다. 반환된 공급자 본문은 저장하지 않고 등록된 행위·결과·안전한 참조만 추출한다.

정기·수동 등록은 같은 위치의 활성 작업을 중복 생성하지 않는다. 사용 조회와 권한 조회는 각 목적의 lease만 회수한다. 계정·리전별 호출 슬롯은 DB에서 공유하며 SDK의 숨은 재시도를 추가하지 않는다. 재예약 후 조회가 실패해도 먼저 확인한 이벤트와 부분 조회 사실을 유지한다. 자동 행위자 분류는 정확한 검증 request 참조를 입증하기 전 UNATTRIBUTED로 유지한다.

조회 예산은 각 CloudTrail SDK 호출 전에 lease 소유권과 함께 원자 예약하며 강제 종료·복구 후에도 차감 상태를 유지한다. 재시도도 예약에 포함해 요청당 100회 호출 시도와 5,000건 처리 상한을 보수적으로 적용한다. 완료된 페이지 수와 예약된 호출 수는 별도로 보존한다. 승인 불일치로 실행할 수 없는 정기 대상은 다음 재검토 시각으로 이동해 뒤의 정상 대상이 계속 예약될 수 있게 하며 승인·활성 상태를 임의 변경하지 않는다.

활성화에는 API·워커의 `logssey.secret.usage.cloudtrail-enabled`, 승인된 AWS 읽기 연결, 계정·리전별 서버 등록 대상과 보호 지문 설정이 필요하다. 사용 조회만 켠 워커도 읽기 자격 증명과 필요한 보호 지문을 독립 구성할 수 있다. 정기 6시간 조회는 대상의 schedule_enabled를 별도로 승인한다. 워커는 승인 대상을 새로 만들거나 승인 범위를 변경할 권한을 받지 않는다. 서버 등록에는 백엔드의 `deploy/register-usage-target.sql`, 읽기 권한 검토에는 `deploy/usage-read-policy.example.json`을 사용하며 실제 계정·리전은 운영 승인에 따라 지정한다.

요청 요약·안전한 관측·결정은 초기 365일 보존한다. 사고·미해결 대응·평가가 참조한 기록은 참조가 해소되기 전 자동 삭제하지 않는다. 일반 API에서 삭제를 제공하지 않으며 정리는 운영 유지보수 권한으로 수행한다.
