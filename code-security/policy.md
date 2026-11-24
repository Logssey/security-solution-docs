# 코드 취약점 검사 정책

## 1. 검사 대상과 시점

| 시점 | 검사 | 결과 |
|---|---|---|
| PR (develop·main 대상) | 전체 검사 | 실행 요약(도구별 High·Medium·Low 건수)과 SARIF 아티팩트(14일) |
| main push | 전체 검사 | PR과 같음 (배포되는 코드의 기록. 대시보드에는 올리지 않음) |
| develop push, 수동 실행 | 전체 검사 | PR과 같고, 대시보드로 올림 |

대상 레포는 service-backend(Java API + 채팅 서버 TS), service-frontend(React·TS). 빌드 산출물(`build`, `dist`)과 `node_modules`는 제외.

## 2. 도구와 규칙 묶음

| 레포 | 도구 | 규칙 |
|---|---|---|
| service-backend | Semgrep `1.185.0` (Docker 이미지 고정) | `p/default`, `p/java`, `p/typescript`, `p/owasp-top-ten` |
| service-backend | SpotBugs `6.5.13`(Gradle 플러그인) + FindSecBugs `1.14.0` | SpotBugs `SECURITY` 분류만 (코드 품질 규칙은 범위 밖) |
| service-frontend | Semgrep `1.185.0` | `p/default`, `p/typescript`, `p/react`, `p/owasp-top-ten` |

- 도구 버전은 고정함. 올릴 때는 PR로 바꾸고, 결과 건수 변화를 실행 요약으로 확인
- Semgrep은 `--metrics off`로 실행 정보를 밖으로 보내지 않음. 규칙 묶음만 레지스트리에서 받음
- **프론트엔드에 ESLint 보안 플러그인을 더하지 않는 이유**: `eslint-plugin-security`의 규칙은 Semgrep JS·TS 규칙과 대부분 겹치고 오탐이 많음(`detect-object-injection` 등). 서비스 레포의 ESLint 설정과 의존성을 바꾸지 않고 Semgrep 하나로 보안 검사를 맡김. ESLint는 지금처럼 코드 품질을 맡음

## 3. 심각도와 운영 단계

SARIF `level`을 그대로 씀 (결과에 없으면 규칙의 기본값).

| SARIF level | 대시보드 | 뜻 |
|---|---|---|
| `error` | High | 악용 경로가 뚜렷한 패턴 (예: 워크플로 쉘 인젝션) |
| `warning` | Medium | 입력 검증 여부를 사람이 봐야 하는 패턴 (예: 문자열로 만든 SQL) |
| `note`, 그 밖 | Low | 참고 |

| 단계 | 동작 | 넘어가는 기준 |
|---|---|---|
| 1. 리포트 전용 | 검사는 실패하지 않음. 머지를 막지 않음 | 2주 동안 결과를 보고 오탐 규칙을 정리 |
| 2. 새 High 차단 | PR에서 develop 대비 새로 생긴 High만 실패 (Semgrep `--baseline-commit`) | 서비스 팀과 합의 후 required check로 지정 |
| 3. High 전체 0건 유지 | 기존 High도 정리하고 0건을 기준선으로 유지 | — |

## 4. 오탐과 예외

- **코드 한 줄**: Semgrep은 `// nosemgrep: <규칙 ID>`, SpotBugs는 `@SuppressFBWarnings(value = "<패턴>", justification = "...")`. 이유를 반드시 함께 적음
- **규칙 전체**: 규칙 묶음에서 빼는 것은 PR로만 하고, PR 설명에 근거를 남김
- 대시보드에는 예외 관리 기능이 없음. 예외는 코드와 PR 이력에 남는 것을 기준으로 함

## 5. 하지 않는 것

- 대시보드에서의 담당 지정·상태 변경·조치 추적
- 의존성 취약점(SCA): 도달 가능성 분석 파트가 맡음
- 코드 조각 저장: 대시보드에는 규칙·위치·설명만 저장하고, 코드는 GitHub 링크(검사한 커밋의 그 줄)로 봄

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-11-24 |
