# 코드 취약점(SAST) 설계문서

**서비스 코드(Re:Used의 service-backend·service-frontend)를 잘 알려진 정적 분석 도구로 CI에서 검사하고, 지금 어떤 발견이 있는지를 통합 대시보드의 '코드 취약점' 화면에 보여 줌.** 별도 분석 엔진을 만들지 않고 검사 정책과 도구 구성을 정하는 것이 이 파트의 범위.

## 솔루션 목표

| 대상 | 도구 | 무엇을 찾나 |
|---|---|---|
| Java API (service-backend) | Semgrep + SpotBugs·FindSecBugs | 인젝션(SQL·명령·경로), 역직렬화, 약한 암호·난수, ReDoS, Spring 설정 실수 |
| 채팅 서버·웹 (TypeScript, React) | Semgrep | XSS(`dangerouslySetInnerHTML` 등), `eval`류, 안전하지 않은 URL·리다이렉트, 하드코딩된 비밀 값 |
| 워크플로 (.github/workflows) | Semgrep | 쉘 인젝션, 신뢰하지 않는 코드 체크아웃, 고정하지 않은 액션 |

- **도구는 업계 표준으로**: Semgrep(오픈소스, 언어 공통 규칙)과 SpotBugs·FindSecBugs(Java 바이트코드 보안 규칙)는 무료이고 결과를 SARIF로 냄. 레포가 비공개여도 추가 비용 없음
- **결과만 남기고 막지 않는다(리포트 전용)**: 오탐을 먼저 보고 기준을 맞춘 뒤 차단으로 넘어감 ([정책](policy.md) 3단계)
- **보여 주기까지만**: 대시보드는 발견 목록만 보여 줌. 담당·상태·예외 같은 조치 관리는 이 파트 범위 밖
- **범위 경계**: 의존성(라이브러리) 취약점은 [취약점 도달 가능성 분석](../reachability-analysis/README.md), 코드 속 시크릿의 검증·대응은 [시크릿](../secret-management/README.md)이 맡음

## 문서

| 문서 | 담는 내용 |
|---|---|
| [정책](policy.md) | 검사 대상, 도구·규칙 묶음, 심각도 기준, 운영 단계(리포트 전용 → 차단), 오탐 처리, 프론트엔드 정책 |
| [설계](design.md) | CI 흐름, 업로드 인증, API, 테이블, 화면 |

## 구현 위치

| 구성 | 위치 |
|---|---|
| 검사 | 서비스 레포의 `.github/workflows/code-scan.yml`. service-backend는 SpotBugs를 `.github/code-scan/spotbugs.init.gradle`로 이 검사에서만 붙임(`build.gradle` 변경 없음) |
| API | [dashboard-backend](https://github.com/Logssey/dashboard-backend)의 `code` 모듈, `/api/v1/code` |
| 결과 DB | 대시보드 PostgreSQL의 `code_scans`·`code_findings` (V61) |
| 화면 | [dashboard-frontend](https://github.com/Logssey/dashboard-frontend)의 `src/code`, 사이드바 '코드 취약점'(`/code`) |

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-11-24 |
