# 시크릿 탐지·검증·대응 설계

승인된 원천에서 자격 증명 후보를 찾고, 동일 키의 발견 위치·현재 인증·노출 심각도·사용 이력·담당 조치·해결 근거를 연결하는 제품의 설계 명세다. 제품 동작과 계약의 기준을 정의하며 개발 현황이나 작업 계획은 포함하지 않는다.

## 문서 구조

| 영역 | 기준 문서 | 내용 |
|---|---|---|
| 요구사항 | [PRD](01-requirements/PRD.md), [수용 기준](01-requirements/acceptance-criteria.md) | 제품 목적·범위·사용자와 기능·보호·수용 조건 |
| 기능·화면 | [기능 설계](02-functional-design/README.md) | 현황·키 위치 시각화·확인·대응·연결·스캔과 상태·문구 |
| 아키텍처 | [아키텍처](03-architecture/README.md) | API·워커·영속 작업·데이터·권한·보안과 기술 결정 |
| 참조 | [자격 증명 유형](04-reference/credential-types.md) | 비밀·식별자·복합 키의 분류와 수집 경계 |
| 시각 자료 | [화면 이미지](assets/README.md) | 본문과 대응하는 화면 설계 8장 |

## 핵심 규칙

- 같은 키를 한 단위로 묶되 위치마다 현재 존재·보관 판단·담당 조치·재검사 결과를 독립적으로 유지한다.
- 형식 탐지, 현재 인증, 보관·공유 적정성, 접근 가능 범위, 권한과 자원 영향, 사용·사고, 작업 진행을 서로 다른 사실로 기록한다. 등급은 기존 `severity-v2`로 결정하고 AI는 조사 순서만 정리한다.
- 유효성의 실제 호출·재취득·최신성은 [검증 계약](03-architecture/credential-validation.md), 위치별 심각도의 근거·룰 저장·평가는 [평가 계약](03-architecture/risk-assessment-contract.md), 사용자 표시는 [검증·심각도 화면](02-functional-design/validation-and-risk-experience.md)을 따른다.
- 보안 담당자의 한 번의 노출 결정으로 위치와 현재 대응 건을 연결한다.
- 해결은 외부 수행·위치 제거·키 수명주기·검사 범위의 근거를 구분해 확인한다.
- 원문 자격 증명은 영속 저장·로그·화면·AI 입력에 남기지 않는다.
- 출처·정책 설정과 위치 판정·대응은 서로 다른 업무다. [사용자 작업과 승인 기준](02-functional-design/task-and-approval-policy.md)에 따라 완료·잔여 이유와 실제 다음 행동을 표시하고, 직접 처리할 작업과 독립 검토가 필요한 변경을 구분한다.
- 보안 효과는 필수 위치의 조치 후 확인과 키 수명주기 근거로 설명한다. 탐지 수 감소·상태 입력·시험 재분류를 위험 감소로 표현하지 않는다.

## 공통 대시보드와의 경계

[대시보드 설계](https://github.com/Logssey/dashboard-design-docs)는 공통 인증·레이아웃·통합 화면 규약을 정의한다. 이 묶음은 Secret 전용 기능·화면·상태·데이터·API를 정의하며 공통 문서에 같은 명세를 중복 작성하지 않는다.

## 판단과 조사

[심각도 정책](02-functional-design/severity-policy.md)은 등급과 여섯 축의 의미를, [사용 이력](03-architecture/usage-observation-contract.md)은 어떤 키의 어떤 기간·범위를 확인했는지를, [AI 조사 요약](03-architecture/risk-briefing-contract.md)은 제한된 사실 전달·출력 검증·장애 처리 경계를 정의한다. 지원 원천은 승인된 GitHub 현재 커밋, AWS 메타데이터·정확한 허용 본문, Notion 승인 페이지·지원 블록이다. Kubernetes·Slack 수집과 GitHub PR·전체 커밋 이력 자동 확장은 이 제품 계약의 적용 범위 밖이다.
