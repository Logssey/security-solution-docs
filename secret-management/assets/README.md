# 설계 이미지 목록

PNG 8장을 용도에 맞는 이름으로 보관한다. 원본 픽셀과 내용은 수정하지 않았다. 파일별 SHA-256과 바이트 수는 [manifest.json](manifest.json)에 기록했다. 이미지는 해당 설계 문서에 배치하고 이 목록은 파일 찾기에 사용한다.

| 원래 파일 | 현재 경로 | 용도 |
|---|---|---|
| 제목 없는 다이어그램.drawio (16).png | [architecture/overall-secret-solution-architecture.png](architecture/overall-secret-solution-architecture.png) | [전체 시스템 구성](../03-architecture/system-overview.md) |
| 제목 없는 다이어그램.drawio (17).png | [architecture/secret-processing-flow.png](architecture/secret-processing-flow.png) | [처리 흐름](../03-architecture/processing-flow.md) |
| 제목 없는 다이어그램.drawio (3).png | [architecture/secret-detection-pipeline.png](architecture/secret-detection-pipeline.png) | 초기 탐지 파이프라인 참고 이미지 |
| image (71).png | [dashboard/01-attack-path-and-response.png](dashboard/01-attack-path-and-response.png) | 상세, 공격 경로, 근거, 대응 절차 |
| image (72).png | [dashboard/02-detection-to-review-summary.png](dashboard/02-detection-to-review-summary.png) | 탐지→분류→우선순위→검토 완료 진행 지표 |
| image (73).png | [dashboard/03-remediation-status-reference.png](dashboard/03-remediation-status-reference.png) | 이전 6단계 조치 막대 참고안 |
| image (74).png | [dashboard/04-severity-validity-matrix.png](dashboard/04-severity-validity-matrix.png) | 심각도×유효성 매트릭스 |
| image (75).png | [dashboard/05-unassigned-findings.png](dashboard/05-unassigned-findings.png) | 담당자 미지정 목록 |

**image 73의 6단계는 최종 기준이 아니다.** PRD에 따라 신규 및 분석 / 조치 중 / 재검증 대기 / 해결 4단계와 별도 예외 표시를 사용한다. 이미지 72의 4개 진행 지표 및 심각도 4등급과 구별한다.

대시보드 이미지의 버튼, 등급, 수치, 규정 태그는 참고이며 구현 완료, 표준 준수, 실제 침해를 나타내지 않는다. 화면별 해석은 [화면 설계](../02-functional-design/dashboard-flow.md), 전체 구성의 해석은 [전체 시스템 구성](../03-architecture/system-overview.md)을 따른다.

## 시크릿 보안 목표 와이어프레임

아래 SVG는 대시보드의 공통 레이아웃과 디자인 토큰에 맞춰 새로 작성한 목표 화면이다. 수치와 시각은 배치 검토용 예시다. 화면별 동작과 현재 구현 상태는 [대시보드 화면 설계](../02-functional-design/dashboard-flow.md)를 따른다.

| 파일 | 화면 |
|---|---|
| [secret-overview-wireframe.svg](dashboard/secret-overview-wireframe.svg) | 통합 현황의 시크릿 요약 |
| [secret-security-wireframe.svg](dashboard/secret-security-wireframe.svg) | 시크릿 현황과 대응 목록 |
| [secret-detail-wireframe.svg](dashboard/secret-detail-wireframe.svg) | 대응 건 상세 패널 |
| [secret-assignment-wireframe.svg](dashboard/secret-assignment-wireframe.svg) | 담당 팀과 개인 지정 |
| [secret-action-record-wireframe.svg](dashboard/secret-action-record-wireframe.svg) | 외부 조치 기록 |
| [secret-exception-wireframe.svg](dashboard/secret-exception-wireframe.svg) | 위험 수용 예외 |
