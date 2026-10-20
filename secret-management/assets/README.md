# 화면 설계 이미지

아래 8장은 [화면 명세](../02-functional-design/dashboard-flow.md)와 함께 사용하는 설계 이미지다. 표시 수치는 합성 예시다. 각 이미지의 상단 표시는 실제 운영·시험 데이터와 문서 예시를 구별하기 위한 것이며 제품의 상태값이 아니다.

| 파일 | 화면 |
|---|---|
| [secret-workbench.svg](dashboard/secret-workbench.svg) | 지금 확인할 일·범위 커버리지·조치 분포 |
| [secret-inventory.svg](dashboard/secret-inventory.svg) | 값 후보·메타데이터와 확인 이유 |
| [secret-key-locations.svg](dashboard/secret-key-locations.svg) | 키별 위치의 존재·조치·확인 불가 비교 |
| [secret-triage-evidence.svg](dashboard/secret-triage-evidence.svg) | 위치 근거·부족한 사실·판정 |
| [secret-response-evidence.svg](dashboard/secret-response-evidence.svg) | 담당 행동·위치별 잔여 작업·해결 증거 |
| [secret-scan-detail.svg](dashboard/secret-scan-detail.svg) | 독립 실행 상세와 성공·누락 범위 |
| [secret-connection-coverage.svg](dashboard/secret-connection-coverage.svg) | 연결 인증과 범위별 실제 검사 결과 |
| [secret-scope-setup.svg](dashboard/secret-scope-setup.svg) | 출처 직접 저장·독립 검토 구분과 보관 정책 잔여 업무 |

예시의 대응 건 4건은 신규1·조치중1·재검사대기1·해결1이다. 현황의 대응 중은 앞의 두 단계 2건이다. 위치 3곳은 발견1·미발견1·확인불가1이며 전체 해결이 아니다. 승인 범위 8개는 성공6·부분1·실패1로, 종료 수와 성공 수가 다르다.

파일 무결성은 [manifest.json](manifest.json)에 기록한다. 업무·권한·집계 규칙은 그림의 예시 숫자가 아닌 본문 명세를 따른다.
