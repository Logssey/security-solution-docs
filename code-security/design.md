# 코드 취약점 설계

## 1. 흐름

```
서비스 레포 (code-scan.yml)                          대시보드 (dashboard-backend)
  checkout
  Semgrep ─────────────┐
  SpotBugs·FindSecBugs ─┤ SARIF
                        ├─ 실행 요약 · 아티팩트 (PR 포함 모든 실행)
                        └─ develop push·수동 실행만
                             OIDC 토큰 (aud logssey-code)
                             POST /api/v1/code/scans ──▶  서명·클레임 확인 → SARIF 읽기
                                                          → 레포·도구의 이전 검사를 새 검사로 교체
                                                     대시보드 '코드 취약점' 화면 ◀─ GET /api/v1/code/findings
```

## 2. 업로드 인증

GitHub Actions OIDC 토큰만 받음. 저장해 두는 비밀 값 없음. IaC CI 인증과 같은 방식의 별도 보안 체인(`CodeCiSecurityConfig`).

| 확인 | 값 |
|---|---|
| 서명·발급자 | GitHub JWKS, `https://token.actions.githubusercontent.com` |
| `aud` | `logssey-code` (다른 솔루션 토큰은 못 씀) |
| `repository` | `Logssey/service-backend`, `Logssey/service-frontend` |
| `ref`·`event_name` | `refs/heads/develop`, `push` 또는 `workflow_dispatch` (PR 실행은 올리지 못함) |
| `job_workflow_ref` | `<레포>/.github/workflows/code-scan.yml@refs/heads/develop` (같은 레포의 다른 워크플로 토큰은 거부) |

틀리면 401. 토큰으로는 업로드 하나만 할 수 있고, 대시보드 세션으로는 업로드할 수 없음(403). 설정 키는 `logssey.code.ci.*`이고 기본값이 위 표라서 운영 설정 파일에 따로 넣을 값 없음.

## 3. API

| 메서드·경로 | 호출자 | 내용 |
|---|---|---|
| `POST /api/v1/code/scans?tool=&commit=&workspace=` | CI | 본문은 SARIF 2.1.0 (최대 20MB). `tool`은 `semgrep`·`spotbugs`, `commit`은 40자 SHA, `workspace`는 러너의 작업 폴더(절대 경로를 레포 기준으로 바꿀 때 씀). 201 `{scanId, total, stored}` |
| `GET /api/v1/code/findings` | 대시보드 세션 | `{scans: [레포·도구별 마지막 검사와 High·Medium·Low 건수], findings: [심각도 순 발견]}` |

## 4. 테이블 (V61)

| 테이블 | 열 | 규칙 |
|---|---|---|
| `code_scans` | 레포, 도구, 커밋, 검사 시각, 전체 건수 | 레포·도구마다 1행. 새 검사가 오면 이전 행과 발견을 지움 |
| `code_findings` | 규칙 ID, 심각도(HIGH·MEDIUM·LOW), CWE, 설명, 경로, 줄, 규칙 설명 링크 | 검사당 최대 1,000건. 코드 조각은 저장하지 않음. 링크는 `https://`만 |

## 5. 화면

사이드바 **코드 취약점**(`/code`). 기존 '취약점 관리' 화면과 분리.

- **레포별 마지막 검사**: 레포, 도구, 커밋(GitHub 링크), 검사 시각, High·Medium·Low 건수
- **발견 목록**: 레포·심각도·도구로 거름. 심각도, 규칙(규칙 설명 링크, CWE), 위치(검사한 커밋의 그 줄로 가는 GitHub 링크), 설명, 도구
- 조치 버튼·상태 열 없음 (정책 5절)

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-11-24 |
