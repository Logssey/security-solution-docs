# API 명세

## 공통 규칙

**모든 API는 `/api/v1/iac` 아래. 호출 주체(대시보드, CI, 내부 작업)마다 인증 방식이 다름.** 아래 경로는 기본 경로를 뺀 형태.

- 형식·이름·오류는 [대시보드 API 공통 규약](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)을 따름 (ADR-D008)

| 호출 주체 | 인증 | 권한 |
|---|---|---|
| 대시보드 | 통합 대시보드 로그인 세션(서버 세션, HttpOnly 쿠키, ADR-D003). 쓰기 요청은 `X-XSRF-TOKEN` 헤더 필요 | MVP는 기존 `ADMIN` 하나. 조회·재검사·담당 지정·지금 점검 모두 `ADMIN`. 역할 분리(조회 전용, 예외 승인 등)는 예외 관리(FR-D3)와 함께 다음 단계 |
| CI (GitHub Actions) | GitHub OIDC 토큰. `aud = logssey-iac`, `repository`·`event_name`·`sub` 클레임을 허용 목록과 대조 | 자기 PR의 실행만 생성·조회 |
| 내부 작업 (CronJob) | k8s 서비스 계정 토큰 (TokenReview 검증) | 작업 결과 기록만 |

- 형식: JSON, 시각은 ISO 8601 UTC, ID는 UUID
- 필드 이름은 camelCase, 값 목록(enum)은 대문자 (`BLOCK`, `CRITICAL`). 규칙 ID(`path.tier1.added` 등)는 값 목록이 아니라 식별자라 그대로 씀
- 목록은 `?cursor=&limit=` (기본 20, 최대 100)
- 대시보드 공통 쿼리 `environment`(`ALL`·`PROD`·`DEV`)를 받음. 현재 IaC 검사 대상은 운영 환경 하나
- 오류: `{"code": "NOT_FOUND", "message": "..."}`. 스택 트레이스, 쿼리, 내부 식별자는 넣지 않음

| HTTP | code | 상황 |
|---|---|---|
| 400 | `INVALID_INPUT` | 형식·허용값 위반 (공통) |
| 401 | `UNAUTHENTICATED` | 세션 없음·만료, OIDC 토큰 검증 실패 (공통) |
| 403 | `FORBIDDEN` | CSRF 토큰 누락·불일치, 허용 목록 밖 레포·PR (공통) |
| 404 | `NOT_FOUND` | 없는 실행·경로·PR (IaC 추가) |
| 409 | `CONFLICT` | 이미 새 커밋으로 교체된 실행에 업로드 (IaC 추가) |
| 413 | `PAYLOAD_TOO_LARGE` | plan JSON 20MB 초과 (IaC 추가) |
| 422 | `UNPROCESSABLE` | plan JSON 형식 오류, 민감값이 남아 있음 (IaC 추가) |
| 500 | `INTERNAL_ERROR` | 내부 오류 (공통) |
| 503 | `UNAVAILABLE` | 그래프·분석 워커 일시 사용 불가 (IaC 추가) |

- 어떤 응답에도 시크릿 값은 담지 않음. 주소·속성 이름·개수만

## CI용 API

**러너는 실행 등록과 plan JSON 업로드만 담당.** 검사 결과와 코멘트는 분석이 끝나면 API 쪽 GitHub App이 등록.

| 메서드 | 경로 | 설명 | 요청 | 응답 |
|---|---|---|---|---|
| POST | `/runs` | 실행 등록 + ② 결과 기록 | `repo`, `prNumber`, `headSha`, `gate: {status, findings[]}` | 201 `{runId}`. ② 차단이면 여기서 검사 3개·코멘트까지 등록 |
| PUT | `/runs/{runId}/plan` | plan JSON 업로드 (민감값 제거, gzip, 20MB 이하) | plan JSON | 202 `{status: "ANALYZING"}` |
| GET | `/runs/{runId}` | 실행 상태와 판정 | 없음 | `{status, verdict, highestSeverity, checks[]}` |

- `status` 순서: `QUEUED` → `GATE_FAILED` 또는 `PLANNING` → `ANALYZING` → `DONE` 또는 `ERROR`
- 같은 PR에 새 `headSha`가 등록되면 이전 실행은 `SUPERSEDED`
- `ERROR`는 필수 검사 실패로 등록 (fail-closed, NFR-4)

## 대시보드용 API

**화면 설계서의 영역 하나가 API 하나에 대응.** 화면을 열 때는 조회 API만, 버튼은 쓰기 API 호출.

| 메서드 | 경로 | 설명 | 화면 | 단계 |
|---|---|---|---|---|
| GET | `/summary` | 지표 4개, 열린 발견 건의 심각도별 건수(`severityCounts`), 마지막 전체 점검 시각, 그래프 기준 시각 | S-01 A·B, 통합 현황 | MVP |
| GET | `/paths?status=OPEN&tier=1` | 열린 경로 목록 (경로별 `severity`) | S-01 C | MVP |
| GET | `/paths/{pathId}` | 경로 단계, 끊을 지점, 끊긴 관계 | S-01 C | MVP |
| GET | `/prs?state=OPEN` | PR별 최신 판정과 `highestSeverity` | S-01 D | MVP |
| GET | `/prs/{number}` | PR 상세 (②①③, 리소스, 다음 할 일, 항목별 `severity`) | S-02 | MVP |
| POST | `/prs/{number}/rerun` | 워크플로 재실행 (GitHub API) | S-02 | MVP |
| GET | `/gates` | ② 관문별 상태, 공급망 요약 | S-01 E | MVP |
| GET | `/state/summary` | State 읽기 주체, 버킷 정책, 민감 후보·실제 수 | S-01 F | MVP |
| GET | `/drifts?status=OPEN` | 드리프트 목록 (주소, 속성 이름, 드리프트 높음·중간·낮음, `severity`) | S-01 F | MVP |
| PATCH | `/findings/{id}` | 드리프트·조치 담당·상태 변경 (`assignee`, `status`) | S-01 F·G | MVP |
| GET | `/remediations/suggested` | 제안 조치 목록 | S-01 G | MVP |
| GET | `/jobs` | 검사별 주기, 마지막 실행, 상태 | S-01 H | MVP |
| POST | `/jobs/{job}/trigger` | 지금 점검 (해당 CronJob을 Job으로 즉시 실행). 주기 변경은 API 없음, gitops에서 | S-01 A | MVP |
| POST | `/remediations` | 수정 PR 생성 (`action`, `pathId`) → `{prUrl}` | S-01 C·G | 다음 (FR-D2) |
| POST | `/exceptions` | 예외 요청 (`ruleId`, `target`, `reason`, `expiresAt`) | S-02 | 다음 (FR-D3) |
| PATCH | `/exceptions/{id}` | 예외 승인·반려 | 관리 | 다음 (FR-D3) |

- 쓰기 API는 MVP에서 모두 `ADMIN`. 예외 승인 권한 분리는 다음 단계
- `POST /prs/{number}/rerun`은 GitHub App에 Actions 쓰기 권한이 필요 (PRD SR-8)
- `action` 값: `IMDSV2_REQUIRED`, `SECRETS_READ_SCOPE`, `TFSTATE_BUCKET_POLICY`, `CLOUDTRAIL_ENABLE`. 수정 PR은 정해진 Terraform 템플릿으로만 생성하고, 만든 PR도 ②①③을 거침

## 내부 작업용 API와 결과 예시

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/jobs/{job}/runs` | 작업 시작 기록. `job` = `cartography`, `enrich`, `drift`, `state` |
| POST | `/jobs/runs/{jobRunId}/results` | 작업 결과 기록 (드리프트 목록, State 요약, 수집 통계). 끝나면 열린 경로 재계산 |

`GET /prs/58` 응답 예시 (S-02 패널에 그대로 사용):

```json
{
  "number": 58,
  "title": "IAM 모듈 정리: 중복된 S3 정책 제거",
  "headSha": "a3f9c21",
  "verdict": "BLOCK",
  "highestSeverity": "CRITICAL",
  "gate": { "status": "PASS", "findings": [] },
  "paths": [
    {
      "change": "ADDED",
      "tier": 1,
      "ruleId": "path.tier1.added",
      "severity": "CRITICAL",
      "hops": ["internet", "nlb", "pod", "imds_v1", "role/logssey-prod-role-node", "s3/logssey-prod-s3-tfstate"],
      "cause": { "address": "module.iam.aws_iam_role_policy.deny_tfstate", "action": "DELETE" },
      "evidence": { "simulator": { "before": "explicitDeny", "after": "allowed" } }
    }
  ],
  "state": { "ruleId": "state.reader_increase", "severity": "HIGH", "readersBefore": 5, "readersAfter": 6, "secretsAdded": 0 },
  "resources": { "add": 0, "change": 0, "delete": 1 },
  "next": { "templateKey": "keep_deny_or_bucket_policy" }
}
```

- 문구는 화면과 코멘트가 `ruleId`·`templateKey`로 각자 생성. API는 데이터만 반환
- 예시의 역할·버킷 이름은 `infrastructure` 레포의 실제 이름(`logssey-prod-role-node`, `logssey-prod-s3-tfstate`)
- `evidence.simulator` 값은 IAM 시뮬레이터 응답의 `EvalDecision` 값을 그대로 씀

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |
