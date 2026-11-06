# AI 조사 요약 계약

기준: [심각도 평가](risk-assessment-contract.md), [API](api-contract.md), [사용 이력](usage-observation-contract.md), [실행 정책](security-and-runtime.md). `risk-briefing`은 기존 합성 TEST 대응 가이드와 독립된 요청·결과 자원이다. 기존 가이드의 대상 제한을 해제하지 않으며 공급자 호출 예산은 공유한다.

### 역할과 제품 표현

기본 판단 이유·필수 조치·부족 근거는 서버가 항상 생성한다. AI는 여러 사실이 함께 있을 때 보여줄 설명과 허용된 추가 조사 순서를 선택한다. 제품 이름은 ‘AI 조사 요약’이다. AI 위험 점수, AI 인증 확인, AI 악용 확정이라고 표현하지 않는다.

AI는 등급·정책·자산 중요도·권한을 새로 정하지 않는다. 원천을 읽거나 검사 대상을 늘리지 않는다. 정책·근거·대응 상태를 변경하는 도구를 제공하지 않는다. 외부 원문에 포함된 지시가 들어올 통로도 열지 않는다. 코드·페이지·로그 자유 텍스트는 입력 대상에서 제외한다.[OWASP Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)

기본 설명과 비교해 조사 시간·이해도가 개선되지 않으면 AI를 기본 기능으로 홍보하거나 자동화하지 않는다. AI가 선택한 문구도 서버의 검토된 카탈로그로 렌더링한다.

### 활성화와 공급자

새 기능의 모드는 OFF / SYNTHETIC_ONLY / SANITIZED_APPROVED다. 기본은 OFF다. SYNTHETIC_ONLY는 서버가 확인한 합성 fixture에만 허용한다. 실제 키가 시험 페이지에 있다는 이유로 합성으로 취급하지 않는다.

SANITIZED_APPROVED는 조직 운영자가 허용된 필드 목록·공급자 데이터 처리 정책을 확인하여 새 기능 전용 정책을 활성화한 경우만 사용한다. 기존 guide의 합성 제한은 자동 해제하지 않는다. 외부 호출 전에 매번 이 활성 상태와 현재 권한을 검사한다.

초기 공급자는 Groq의 openai/gpt-oss-20b와 strict JSON schema다.[Groq structured outputs](https://console.groq.com/docs/structured-outputs) 배포 설정에 모델 ID·schema·prompt·카탈로그 버전을 함께 고정한다. 모델이 중단되거나 변경되면 기본 설명으로 전환하며 다른 모델·유료 공급자로 자동 대체하지 않는다. Claude adapter는 이 계약을 구현하고 같은 수용 시험을 통과한 별도 배포에서만 선택할 수 있다.

Groq의 ZDR 설정을 외부 전송 활성화 점검에 포함한다. 수집한 설정 확인은 운영 증거로 보존하고 공급자 보관 정책이 변경되면 재검토한다. 요청 수준의 임의 플래그로 ZDR을 보장한다고 주장하지 않는다. 배치·파일 업로드·학습·검색 기능은 사용하지 않는다.[Groq 데이터 처리](https://console.groq.com/docs/your-data)

전송 endpoint는 https://api.groq.com/openai/v1/chat/completions로 고정하고 TLS 검증을 유지한다. redirect는 따라가지 않는다. DB나 브라우저에서 임의 endpoint를 지정하지 못하게 하며 API 키는 운영 비밀 설정에서만 읽는다. 신규 설정은 SECRET_RISK_BRIEFING_MODE=OFF, SECRET_RISK_BRIEFING_PROVIDER=groq, SECRET_RISK_BRIEFING_MODEL=openai/gpt-oss-20b를 기본으로 한다. 새 사용 이력 자동 조회의 SECRET_USAGE_CLOUDTRAIL_ENABLED도 기본 false이며 승인된 관리 연결·계정·리전 등록 후 활성화한다. 활성화는 기능 배포와 별도이며 설정·정책·권한 조건을 모두 통과해야 한다.

### 외부 입력

서버는 허용된 enum·정수·boolean만으로 새 객체를 만든다. DB 객체에서 민감 필드 몇 개를 제거하는 방식은 쓰지 않는다. F01 같은 근거 참조는 요청 내부 임시 별칭이며 실제 DB ID는 외부로 보내지 않는다. 입력 생성 시 모든 사실의 내부 원천·조직·키·위치 결합과 요청자의 읽기 권한을 검사하고 외부 전송 직전에 다시 확인한다. 요청 위치에 사용할 수 없거나 읽을 수 없는 자료를 요구하는 입력이면 외부 호출 없이 CONTEXT_NOT_AUTHORIZED로 종료한다. 이 사유에 숨겨진 자료의 이름·존재·개수를 넣지 않는다.

```json
{
  "schema_version": "risk-briefing-input-v1",
  "policy_version": "severity-v2",
  "assessment": {"status": "UNASSESSED", "severity": null, "rule": "R99"},
  "credential_type": "AWS_ACCESS_KEY_PAIR",
  "facts": [
    {"ref": "F01", "code": "PUBLIC_ACCESS_OBSERVED", "coverage": "COMPLETE", "freshness": "CURRENT", "acceptance": "ACCEPTED"},
    {"ref": "F02", "code": "ADMIN_PERMISSION_CANDIDATE", "coverage": "PARTIAL", "freshness": "CURRENT", "acceptance": "CANDIDATE"}
  ],
  "mandatory_reason_codes": ["EFFECTIVE_PERMISSION_UNCONFIRMED"],
  "allowed_investigation_codes": ["CONFIRM_PERMISSION_RESTRICTIONS", "CONFIRM_ASSET_CLASSIFICATION"]
}
```

외부 전송에서 원문·마스킹 키·지문·원천 이름·경로·ARN·계정/사용자/사건 ID·IP·이메일·자유 메모·URL·정확한 업무 시각을 제외한다. 필요한 최신성은 CURRENT/STALE 같은 범주로 전달한다. 같은 형식의 새 field 추가도 전송 정책 버전과 검토를 필요로 한다.

사실 최대 32개, 입력 직렬화 최대 16KiB다. 초과하면 사실을 조용히 잘라 보내지 않고 CONTEXT_LIMIT과 기본 설명을 반환한다. 사용자 임의 prompt는 받지 않는다.

### 외부 출력과 검증

```json
{
  "schema_version": "risk-briefing-output-v1",
  "reason_order": [
    {"code": "EFFECTIVE_PERMISSION_UNCONFIRMED", "fact_refs": ["F02"]}
  ],
  "investigation_order": ["CONFIRM_PERMISSION_RESTRICTIONS", "CONFIRM_ASSET_CLASSIFICATION"]
}
```

출력에는 자유 설명문, 점수, severity, 상태 변경 명령, 링크가 없다. schema의 모든 속성을 필수로 선언하고 additionalProperties=false를 적용한다. 이유 최대 5개, 조사 최대 5개, reasoning_effort=low, completion 최대 512 tokens, 모델 content 최대 8KiB, HTTP 응답 body 최대 64KiB다.

서버는 schema뿐 아니라 코드의 허용 여부, 사실 참조의 존재·현재 권한·연결 predicate, 중복, 후보/확정 구분, 필수 조치 순서까지 확인한다. 필수 위험 사실과 부족 근거는 AI가 누락해도 서버가 항상 표시한다. 허용되지 않는 항목이 하나라도 있으면 전체 AI 출력을 폐기한다. 형식 준수는 의미 정확성을 보장하지 않으므로 이 의미 검사를 생략하지 않는다.[Groq structured outputs](https://console.groq.com/docs/structured-outputs)

초기 설명 카탈로그는 PUBLIC_ACCESS_OBSERVED, ORG_WIDE_ACCESS_OBSERVED, STORAGE_POLICY_VIOLATION, STORAGE_POLICY_REQUIRED, EFFECTIVE_PERMISSION_UNCONFIRMED, ASSET_CLASSIFICATION_REQUIRED, AUTHENTICATION_CONFIRMED, AUTHENTICATION_REJECTED, USAGE_COVERAGE_PARTIAL, IDENTITY_BINDING_UNRESOLVED, EVIDENCE_STALE, EVIDENCE_CONFLICT를 사용한다. 조사 코드는 CONFIRM_STORAGE_POLICY, CONFIRM_ACCESS_BOUNDARY, CONFIRM_PERMISSION_RESTRICTIONS, CONFIRM_ASSET_CLASSIFICATION, REVIEW_USAGE_WINDOW, VERIFY_CREDENTIAL_BINDING, RECHECK_AUTHENTICATION다. 각 코드의 사용 조건은 같은 이름의 서버 사실·누락 predicate에 결합하며 모델이 조건을 새로 만들지 않는다.

### 요청 수명과 장애 처리

요청 상태는 QUEUED → RUNNING → SUCCEEDED / FALLBACK / SUPERSEDED / CANCELLED다. 매 요청의 실제 전송 시도는 별도 attempt다. 최대 lease 30초, fencing generation으로 중복 결과 적용을 차단한다. attempt는 RESERVED → DISPATCHING → 종료 결과로 진행한다. 외부 호출 직전에 DISPATCHING 전환과 예산 차감을 원자 커밋한다. RESERVED에서 외부 전송을 하지 않았다는 사실이 확실한 취소만 예약을 반환한다. DISPATCHING 이후 워커 종료·통신 결과 불명은 차감을 유지하고 자동 재전송하지 않는다. lease 만료로 같은 attempt를 다시 보내지 않는다. fencing은 로컬 결과의 중복 적용을 막으며 이미 전송된 원격 요청을 취소하거나 원격 중복을 보장하는 장치가 아니다.

헤더·본문 전체 timeout 5초, 자동 재시도 0회다. 실패 후 사용자가 새 요청을 만들려면 최소 60초를 기다린다. 429 Retry-After가 더 길면 그 시각까지 차단하고, 24시간 초과이면 운영 확인 전 재시도를 차단한다. 401/403은 설정 오류로 표시하여 운영 설정이 바뀔 때까지 공급자 호출을 중단한다. 5xx·timeout·거부·잘린 응답·의미 검증 실패는 각각 reasonCode로 기록하고 기본 설명을 유지한다.

초기 배포 전체 동시 외부 호출 1건, 분당 6회, 하루 100회다. 성공 여부와 무관하게 DISPATCHING에 진입한 요청은 예산에서 차감하며 전송 결과 불명도 포함한다. DB 트랜잭션과 공통 카운터로 여러 인스턴스·기존 guide 호출을 합산하고, 프로세스 재시작으로 한도가 초기화되지 않게 한다. 무료 한도나 가격을 보장하지 않으며 예산 소진 후 유료 전환하지 않는다.

캐시 최대 24시간이며 동일한 occurrence·평가 ID·전체 contextDigest·ACL 범위·전송 정책·모델·prompt·카탈로그 버전에서만 재사용한다. 로컬 입력 snapshot과 contextDigest에는 위험 근거 revision뿐 아니라 인증 시도와 현재 적격성 revision, 사용 조회/관측 revision, 각 사실의 최신성·수용 상태를 포함한다. v2 등급이나 평가 ID가 같아도 인증·사용 사실이 바뀌면 digest가 달라진다. 현재 결과의 유효기한은 캐시 상한과 포함된 모든 사실의 유효기한 중 가장 이른 시각이다.

요청 시, 실행 시, 결과 조회 시 현재 digest와 권한을 재확인한다. 결과 채택 트랜잭션은 contextDigest·평가 ID·근거/인증/사용/ACL/전송 정책 revision·lease generation을 현재 값과 조건부 비교한다. 하나라도 다르면 current 결과 포인터를 갱신하지 않고 SUPERSEDED 또는 권한 철회에 따른 CANCELLED로 종료한다. 이미 성공한 과거 요청의 성공 기록은 지우지 않고 현재 결과 적격성만 STALE로 표시한다. GET·캐시 적중은 공급자 호출이 아니다.

화면은 source=DEFAULT 또는 AI, providerCalled, cached, reasonCode, 생성 시각, 기준 평가를 구분한다. providerCalled는 통신 시도 의미이며 공급자의 실제 처리 완료나 과금 확정을 뜻하지 않는다. 모델 응답 원문은 로그·DB에 저장하지 않고 검증된 코드·순서와 사용량·상태 메타데이터만 남긴다.

## 실행과 보존

AI는 수집·평가 작업과 분리된 영속 큐를 사용한다. 기본 2초 polling, 비어 있으면 10초까지 완화한다. 수동 요청 wake-up은 선택 최적화이며 DB가 요청 원본이다. 요청·시도·예산 모델과 보존 제약은 [DB 스키마](database-schema.md)를 따른다. AI 요청이나 성공을 ScanRun·인증 성공·위험 평가 횟수에 합산하지 않는다.

기본 설명은 AI 요청 없이 항상 조회할 수 있다. 근거 평가가 없으면 `ASSESSMENT_REQUIRED`, 메타데이터 전용이면 대상 제외를 반환한다. 현재 결과의 적격성 만료는 과거 성공 기록 삭제나 대응 상태 변경을 만들지 않는다.
