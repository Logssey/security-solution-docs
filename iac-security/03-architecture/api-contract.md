# API 명세

## 공통 규칙

**모든 API는 dashboard-backend의 `/api/v1/iac` 아래. 호출 주체(대시보드, CI, 내부 작업)마다 인증 방식이 다름.** 아래 경로는 기본 경로를 뺀 형태.

- 별도 서비스가 아니라 대시보드 백엔드에 `iac` 모듈로 들어감 (ADR-11). 분석 워커는 API를 거치지 않고 PostgreSQL 대기열(`iac_jobs`)과 결과 테이블을 직접 씀

- 형식·이름·오류는 [대시보드 API 공통 규약](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)을 따름 (ADR-D008)

| 호출 주체 | 인증 | 권한 |
|---|---|---|
| 대시보드 | 통합 대시보드 로그인 세션(서버 세션, HttpOnly 쿠키, ADR-D003). 쓰기 요청은 `X-XSRF-TOKEN` 헤더 필요 | MVP는 기존 `ADMIN` 하나. 조회·재검사·담당 지정·지금 점검 모두 `ADMIN`. 역할 분리(조회 전용, 예외 승인 등)는 예외 관리(FR-D3)와 함께 다음 단계 |
| CI (GitHub Actions) | GitHub OIDC 토큰. `aud = logssey-iac`, `repository`·`event_name`·`sub` 클레임을 허용 목록과 대조 | 자기 PR의 실행만 생성·조회 |
| 내부 작업 (CronJob) | k8s 서비스 계정 토큰 (TokenReview 검증) | 작업 결과 기록만 |

- 형식: JSON, 시각은 ISO 8601 UTC, ID는 숫자(DB의 `BIGINT`)
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
| PUT | `/runs/{runId}/plan` | plan JSON 업로드 (민감값 제거, gzip, 20MB 이하). `iac_plan_payloads`에 임시 저장하고 `iac_jobs`에 `ANALYZE_RUN` 등록 | plan JSON | 202 `{status: "ANALYZING"}` |
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
| GET | `/paths/{pathId}` | 경로(`path`)와 끊을 지점(`cutPoints`: `fromHop`·`toHop`은 `hops` 순번, `action`, `templateKey`) | S-01 C | MVP |
| GET | `/prs` | PR별 최신 run의 판정, `highestSeverity`, 경로 변화 수(`pathsAdded`, `removedRoutes`), State 읽기 주체 증가 여부, 대표 규칙(`reason`). 최근 50개 | S-01 D | MVP |
| GET | `/prs/{number}` | PR 상세 (②①③, 리소스, 다음 할 일, 항목별 `severity`) | S-02 | MVP |
| GET | `/gates` | ② 관문별 상태, 공급망 요약 | S-01 E | MVP |
| GET | `/state/summary` | State 읽기 주체, 버킷 정책, 민감 후보·실제 수 | S-01 F | MVP |
| GET | `/drifts?status=OPEN` | 드리프트 목록 (주소, 속성 이름, 드리프트 높음·중간·낮음, `severity`) | S-01 F | MVP |
| PATCH | `/findings/{id}` | 드리프트 담당·상태 변경 (`assigneeUserId` 또는 `unassign`, `status`는 `OPEN`·`ACCEPTED`, 읽었을 때의 `version`. 다르면 409) | S-01 F | MVP |
| GET | `/remediations/suggested` | 제안 조치 목록 | S-01 G | MVP |
| GET | `/jobs` | 검사별 주기, 마지막 실행, 상태 | S-01 H | MVP |
| POST | `/jobs/{job}/trigger` | 지금 점검. `enrich-graph`·`check-state`는 `iac_jobs`에 `MANUAL`로 등록(같은 종류가 이미 기다리면 그 작업을 돌려줌). `collect-graph`(CronJob을 Job으로 즉시 실행)와 `check-drift`(main plan 필요)는 대시보드 배포 전이라 409. 주기 변경은 API 없음, gitops에서 | S-01 A | MVP |
| POST | `/remediations` | 수정 PR 생성 (`action`, `pathId`) → `{prUrl}` | S-01 C·G | 다음 (FR-D2) |
| POST | `/exceptions` | 예외 요청 (`ruleId`, `target`, `reason`, `expiresAt`) | S-02 | 다음 (FR-D3) |
| PATCH | `/exceptions/{id}` | 예외 승인·반려 | 관리 | 다음 (FR-D3) |

- 쓰기 API는 MVP에서 모두 `ADMIN`. 예외 승인 권한 분리는 다음 단계
- PR 다시 검사 API는 두지 않음. 재실행은 GitHub에서 하고, 대시보드는 PR 링크("PR에서 보기")만 둠. GitHub App에 Actions 쓰기 권한을 주지 않아 대시보드가 CI를 실행할 수 없게 함 (10/21 결정)
- `action` 값: `IMDSV2_REQUIRED`, `SECRETS_READ_SCOPE`, `TFSTATE_BUCKET_POLICY`, `CLOUDTRAIL_ENABLE`. 수정 PR은 정해진 Terraform 템플릿으로만 생성하고, 만든 PR도 ②①③을 거침

## 내부 작업용 API와 결과 예시

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/jobs/collect-graph/runs` | 매일 작업(Cartography)이 수집 시작·종료와 통계를 기록. 종료되면 `ENRICH_GRAPH`, `CHECK_DRIFT`, `CHECK_STATE`를 `iac_jobs`에 등록 |

- 보강·드리프트·State 점검 결과는 워커가 결과 테이블에 직접 적재하고, 끝나면 열린 경로를 다시 계산
- `job` 경로 값은 `iac_jobs.kind`의 소문자·하이픈 형태 (`collect-graph`, `enrich-graph`, `check-drift`, `check-state`)

`GET /prs/58` 응답 예시 (S-02 패널에 그대로 사용, 10/21 로컬 데모 결과에서 경로 1개만 남기고 계정 ID를 가림):

```json
{
  "number": 58,
  "title": null,
  "runId": 11,
  "headSha": "602a5dca0bf3a213bfbe43d3a06f14407670595e",
  "status": "DONE",
  "verdict": "BLOCK",
  "highestSeverity": "CRITICAL",
  "checkedAt": "2026-10-21T05:00:10Z",
  "gate": { "status": "PASS", "findings": [] },
  "paths": [
    {
      "change": "ADDED",
      "tier": 1,
      "ruleId": "path.tier1.added",
      "severity": "CRITICAL",
      "hops": [
        { "kind": "Internet", "id": "internet", "name": "internet" },
        { "kind": "AWSCloudFrontDistribution", "id": "arn:aws:cloudfront::<account>:distribution/E3NA9R8NTNSVRH", "name": "arn:aws:cloudfront::<account>:distribution/E3NA9R8NTNSVRH" },
        { "kind": "AWSLoadBalancerV2", "id": "arn:aws:elasticloadbalancing:ap-northeast-1:<account>:loadbalancer/net/logssey-prod-nlb-public/a37c39e077808af2", "name": "logssey-prod-nlb-public" },
        { "kind": "AWSEC2Instance", "id": "arn:aws:ec2:ap-northeast-1:<account>:instance/i-030605b5b05e1eb94", "name": "i-030605b5b05e1eb94" },
        { "kind": "AWSRole", "id": "arn:aws:iam::<account>:role/logssey-prod-role-node", "name": "logssey-prod-role-node" },
        { "kind": "AWSS3Bucket", "id": "arn:aws:s3:::logssey-prod-s3-tfstate", "name": "logssey-prod-s3-tfstate" }
      ],
      "cause": { "address": "module.iam.aws_iam_role_policy.deny_tfstate", "action": "DELETE" },
      "evidence": { "simulator": { "before": "explicitDeny", "after": "allowed" } }
    }
  ],
  "resourceFindings": [],
  "resources": { "add": 0, "change": 0, "delete": 1 },
  "state": { "ruleId": "state.reader_increase", "severity": "HIGH", "readersBefore": 5, "readersAfter": 6, "secretsAdded": 0 }
}
```

- `hops`는 경로 단계 배열. 단계마다 `kind`(Cartography 라벨, 인터넷은 `Internet`), `id`(ARN), `name`(표시 이름). 열린 경로(`/paths`)도 같은 모양
- 문구는 화면과 코멘트가 `ruleId`·`templateKey`로 각자 생성. API는 데이터만 반환 (다음 할 일도 `ruleId`로 화면이 만듦)
- `cause.action`은 plan의 변경 종류(`CREATE`, `UPDATE`, `DELETE`, `REPLACE`). 저장은 `iac_run_path_changes.evidence.causeAction`
- `evidence.simulator` 값은 IAM 시뮬레이터 응답의 `EvalDecision` 값을 그대로 씀
- `resourceFindings`는 경로 없이 리소스 하나에 걸리는 ① 발견 건 (`path.imds_v1_added`). 항목은 `ruleId`, `enforcement`, `severity`, `address`, `action`(`cause.action`과 같은 값). 경로 발견 건은 `paths`에만 나옴 (11/04)
- `resources`는 plan의 추가·변경·삭제 수만. plan 원문과 리소스 값은 내려주지 않음
- `title`은 아직 받지 않아 `null` (CI 요청에 PR 제목이 없음). 화면은 제목 없이 번호·커밋만 표시
- ②에서 막힌 run은 `gate.status`가 `BLOCK`이고 `paths`·`state`·`resources`가 비어 있음 (계산 안 함)

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |
| 수정일 | 2026-11-04 |
