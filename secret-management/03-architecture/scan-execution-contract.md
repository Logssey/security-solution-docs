# 스캔 진행과 실행 이벤트

## 읽기 계약

`GET /api/v1/secrets/runs/{runId}/progress?afterEventId=0&size=100`은 현재 연결 운영 사실의 열람 권한을 확인하고 한 REPEATABLE_READ 기준에서 실행·단계·이벤트를 읽는다. size는 1~200이며 cursor는 해당 run의 마지막 수신 이벤트 순번이다. 실행 등록은 기존 `POST /api/v1/secrets/runs`의 비동기·멱등 계약을 사용한다.

응답은 `runId`, `status`, `attemptNo`, `maxAttempts`, `capturedAt`, `latestActivityAt`, `pollAfterMs=2500`, `eventsStatus`, `historyComplete`, `truncated`, `suppressedEventCount`, `stages`, `events`, `nextAfterEventId`, `hasMore`를 제공한다. `eventsStatus`는 AVAILABLE·NOT_RECORDED·EXPIRED를 구별한다. 보존 기간 밖의 실행에는 기록 시작·만료 근거를 제공한다.

`validationSummary`는 `reuseRecorded`, `newRequestCount`, `completedRequestCount`, nullable `reusedResultCount`, `uncheckedCount`, `incompleteCount`, `unknownCount`, `reusedResults[]`, `reusedResultsTruncated`를 제공한다. 재사용 항목은 발급자·판정·사유·원래 `providerCheckedAt`·`sourceRunId`·건수로 구성한다. 새 인증 요청과 재사용을 각각 집계하며 과거 실행과 재사용 기록을 지원하지 않는 단건 수동 검증은 `reuseRecorded=false,reusedResultCount=null`이다.

단계 key는 QUEUE·COLLECTION·DETECTION·VALIDATION·RESULT다. 단계는 `status`, nullable `startedAt`, `finishedAt`, `reasonCode`, `processedCount`, `totalCount`, `unit`를 제공한다. 상태는 PENDING·RUNNING·SUCCEEDED·PARTIAL·FAILED·SKIPPED·CANCELLED·UNKNOWN이며 unit은 SCOPE·OBJECT·CANDIDATE·ATTEMPT 또는 null이다. 관측되지 않은 수량은 null이고 관측된 0만 0이다.

이벤트는 `id`, `attemptNo`, `stage`, `operationCode`, `outcome`, nullable `httpStatus`, `reasonCode`, `processedCount`, `totalCount`, `durationMs`, `observedAt`를 제공한다. outcome은 STARTED·COMPLETED·PARTIAL·FAILED·SKIPPED·RETRY_SCHEDULED·CANCELLED다. 명령과 요청 표현은 등록된 operation의 고정 템플릿으로 생성하며 원천 참조·비밀 인수를 포함하지 않는다.

## 실제 처리와 단계

파일·블록 단위로 수집, Gitleaks 탐지, 정규화, 결과 저장과 인증 검증을 반복한다. COLLECTION·DETECTION·VALIDATION은 동시에 실행 중일 수 있다. 실제 첫 관측 이벤트를 단계 시작의 근거로 사용한다. 기존 처리기가 여러 상태를 미리 RUNNING으로 설정한 사실만으로 새로운 실행 이벤트를 만들지 않는다.

QUEUE는 등록·선점·재시도 원장을, RESULT는 최종 presence·저장·재검사 reconciliation이 완료된 트랜잭션을 사용한다. 파일별 저장은 DETECTION 단계의 실제 저장 관측으로 기록한다. 기존 실행의 단계 원장과 실제 관측 시간은 상세 이벤트의 유무와 구별한다.

메타데이터 수집의 탐지·검증 제외는 NOT_APPLICABLE_METADATA로 표시한다. 비활성·미지원·후보 없음·구성 부족·합성 확인에 따른 미실행 사유를 각각 유지한다. 인증 거부 응답은 키의 INVALID 결과와 해당 요청의 완료 사실을 기록하며 통신 장애와 구별한다. 재시도는 attemptNo로 구별하고 이전 시도의 완료를 새 시도의 관측으로 사용하지 않는다.

동일한 주기·키·위치·검증 설정에서 유효한 관측을 사용하면 원래 시도에 연결한 재사용 사실을 현재 실행·범위·시도에 기록한다. 현재 재수집 근거·승인 문맥·권한·lease를 확인하고 `CREDENTIAL_RESULT_REUSED`로 관측한다. 이 이벤트에는 새 공급자 요청이나 HTTP 응답을 부여하지 않는다. 키 쌍 부족은 INCOMPLETE_PAIR, 임시 키의 세션 토큰 부족은 MISSING_SESSION_TOKEN으로 표시한다.

처리 종료 결과 SUCCEEDED·PARTIAL·SKIPPED의 사용자 표시는 **검사 완료**다. PARTIAL 원장과 미검증·누락·생략 사유는 별도로 보존하고 상세 화면에 제공한다. 검사 완료는 실행 처리가 종료되었다는 의미이며, 키의 유효성은 각 키의 판정으로 확인한다.

## 기록 지점과 저장

GitHub REST·Notion 페이지와 블록 HTTP 호출은 실제 응답 코드와 시간만 관측한다. AWS SDK 응답에서 HTTP 코드가 제공된 경우에만 저장한다. Gitleaks는 실제 실행·exit 결과·도구 소요 시간과 안전한 건수를 기록한다. 인증 검증은 기존 시도 원장의 실제 HTTP·사유·시각을 사용한다. 외부 호출은 DB 트랜잭션 밖에서 수행한다.

이벤트 테이블은 실행·시도·범위의 서버 식별자, 단계·operation·결과 enum, 허용된 상태 코드·수량·시간만 저장한다. 요청 URL·경로·헤더·원천 본문·공급자 응답 원문·예외 메시지·자유 텍스트 로그를 저장하는 칼럼은 두지 않는다. 원천 요청의 결과 기록 실패를 성공 로그로 대체하지 않는다.

기록기는 현재 lease token·시도·generation·DB 시각을 확인하고 run 행 잠금 아래 순번을 배정한다. 등록·선점·재시도·완료의 원장 변경과 해당 이벤트는 같은 트랜잭션에 반영한다. 만료된 실행자가 새 시도에 이벤트를 덧붙일 수 없다. 네트워크 관측 context는 실행·범위마다 명시적으로 전달한다.

실행별 최대 2,000개, 시도별 상세 최대 1,000개를 기록하고 lifecycle용 128개 공간을 확보한다. 제한에 도달하면 생략 수와 내역 제한 사실을 기록하고 최종 단계·결과 요약을 제공한다. 이벤트는 30일 보존하며 정리 후 실행 결과와 기록 만료 사실을 제공한다. 진행·종료 이벤트의 안전한 수량은 감사 가능한 원장 값과 일치해야 한다.

이미 적용된 공유 마이그레이션은 보존하고 후속 변경을 하나의 작업 단위로 배치한다. API 읽기와 worker 쓰기 권한을 배포 grant에 반영한다. 기존 개발 DB의 적용 전 백업과 별도 DB의 전체 적용 검증을 수행한다.

## 갱신과 실패

클라이언트는 화면이 보이고 실행이 진행 중일 때 조회한다. 이벤트 순번을 중복 제거하고 hasMore일 때 순차적으로 다음 페이지를 읽는다. 한 번의 갱신은 최대 20페이지로 제한하며 이후 cursor에서 재개한다. 종료 상태의 미수신 이벤트까지 읽은 후 주기 조회를 끝낸다.

API 오류는 마지막 확인 시각과 재조회 행동으로 표시하고 실행 원장의 실패로 변환하지 않는다. 403·404에서는 자동 조회를 종료한다. 연결을 바꾸거나 컴포넌트가 사라지면 이전 실행의 응답과 폴링을 정리한다. 서버는 매 요청마다 현재 권한을 확인한다.

수용 시험은 실제 상태와 이벤트의 대응, 재시도·동시 선점·만료 fencing, cursor 순서·동시 commit, 한도·보존, 부분 실패·검증 생략, 현재 권한 회수, 원문 비노출, 종료 이벤트 수신과 화면 재접속을 포함한다.
