# ERD

## ERD

**결과 DB(`logssey_iac`, PostgreSQL)는 실행(iac_run, job_run), 탐지(finding), 경로(attack_path) 세 축으로 구성.**

- 자산 그래프 자체는 Neo4j에 두고, 여기에는 판정 결과와 이력만 저장

![결과 DB ERD: 테이블 10개](../assets/architecture/database-erd.png)

- 드리프트·② 위반·규칙 점검·State 결과는 모두 `finding` 한 테이블에 넣고 `source`로 구분
- 경로는 PR마다 바뀌는 "차이"와 계속 열려 있는 "상태"가 다르므로 `attack_path`와 `run_path_change`로 분리
- `remediation`·`exception`은 다음 단계(FR-D2·D3) 기능용 테이블. MVP에서는 만들지 않아도 됨

## 테이블 정의

| 테이블 | 주요 컬럼 | 설명 |
|---|---|---|
| `repo` | `id` PK, `full_name` UQ, `default_branch` | 검사 대상 레포 (허용 목록) |
| `iac_run` | `id` PK, `repo_id` FK, `pr_number`, `head_sha`, `trigger`(pr·push·manual), `status`, `verdict`(block·warn·info·pass), `highest_severity`, `graph_as_of`, `superseded_by` FK, `started_at`, `finished_at` | PR 검사 1회. `graph_as_of`는 판정에 쓴 그래프 시점. `status` 값은 API 명세의 실행 상태와 같음. push는 main 병합 후 ② 재검사(S-01 B의 main 기준 위반) |
| `job_run` | `id` PK, `job`(cartography·enrich·drift·state), `status`, `started_at`, `finished_at`, `stats` jsonb | 매일 작업 1회 |
| `rule` | `id` PK(예: `gate.exec_data_source`), `engine`(conftest·git·terraform·hash·kics·path·state·drift), `default_enforcement`, `default_severity`, `template_key`, `version`, `enabled` | 판정 규칙과 문구 템플릿 (규칙 명세와 1:1) |
| `finding` | `id` PK, `run_id` FK null, `job_run_id` FK null, `rule_id` FK, `source`(gate·rule·path·state·drift), `enforcement`, `severity`, `resource_address`, `params` jsonb, `fingerprint` UQ, `status`(open·fixed·accepted), `assignee`, `first_seen`, `last_seen` | 모든 탐지 결과. `params`에는 속성 이름·개수만, 값 없음. `run_id`·`job_run_id` 중 하나는 반드시 있음(CHECK) |
| `data_asset` | `id` PK, `arn` UQ, `kind`(rds·secret·s3·tfstate), `tier`(1·2·3), `owner` | 데이터 등급표 |
| `attack_path` | `id` PK, `fingerprint` UQ(진입+단계+대상의 해시), `entry`, `target_asset_id` FK, `tier`, `severity`, `hops` jsonb, `length`, `status`(open·closed), `first_seen`, `last_seen` | 경로의 현재 상태. S-01 C 영역 |
| `run_path_change` | `run_id` PK FK, `path_id` PK FK, `change`(added·removed·shortened) | PR이 만든 경로 차이. S-02 ① |
| `remediation` | `id` PK, `action`, `path_id` FK null, `finding_id` FK null, `pr_number`, `pr_url`, `status`, `created_by`, `created_at` | 수정 PR 생성 이력. 다음 단계(FR-D2) |
| `exception` | `id` PK, `rule_id` FK, `target`, `reason`, `requested_by`, `approved_by`, `status`, `expires_at` | 예외. 만료되면 판정에서 다시 빠짐. 다음 단계(FR-D3) |
| `audit_log` | `id` PK, `actor`, `action`, `target`, `at` | 예외 승인·재검사·수정 PR 기록 (SR-10). 그림에서는 생략 |

**강도·심각도 컬럼**

| 컬럼 | 값 | 기준 |
|---|---|---|
| `rule.default_enforcement`, `finding.enforcement` | block·warn·info·ignore | 규칙 명세의 강도 |
| `rule.default_severity`, `finding.severity`, `attack_path.severity`, `iac_run.highest_severity` | critical·high·medium·low, 없음은 null | 규칙 명세의 심각도. null은 위험 집계에서 제외 |

- 값 목록은 DB에 소문자로 저장하고, API는 대문자로 반환 (대시보드 API 공통 규약)
- `fingerprint`는 같은 문제가 매일·PR마다 새 행으로 쌓이지 않게 하는 키
    - 같은 지문이면 `last_seen`만 갱신
    - 더 이상 안 보이면 `fixed`·`closed`로 변경

## 그래프 스키마 (Neo4j)

**노드·관계는 대부분 Cartography 것을 그대로 사용하고, 경로 계산에 빠진 관계 4개와 가상 노드 1개만 보강.**

- 보강 관계에는 `_source = 'logssey'`와 `lastupdated`를 붙여 Cartography 관계와 구분하고, 다음 실행에서 확인 안 된 것은 삭제
- 라벨은 공식 스키마 기준(AWS 접두). 10/7 수집에서는 `EC2Instance`·`S3Bucket` 같은 이전 라벨도 함께 붙어 있었음

| 구분 | 이름 | 출처 |
|---|---|---|
| 노드 | `AWSEC2Instance`, `AWSRole`, `AWSInstanceProfile`, `AWSPolicy`, `AWSPolicyStatement`, `AWSS3Bucket`, `AWSRDSInstance`, `AWSLoadBalancerV2`, `AWSEC2SecurityGroup`, `AWSCloudFrontDistribution`, `AWSSecretsManagerSecret` | Cartography AWS 모듈 |
| 노드 | `KubernetesCluster`, `KubernetesNode`, `KubernetesPod` | Cartography Kubernetes 모듈 |
| 노드 | `Internet` (가상 노드 1개, 경로 출발점) | logssey 보강 |
| 관계 | `INSTANCE_PROFILE`, `ASSOCIATED_WITH`, `POLICY`, `STATEMENT`, `EXPOSE`(LB→EC2), `MEMBER_OF_EC2_SECURITY_GROUP`, `RUNS_ON`(파드→노드), `IS_INSTANCE`(노드→EC2, `providerID` 기준) | Cartography |
| 관계 | `CAN_READ`·`CAN_WRITE`(주체→S3·시크릿, 거부 반영) | Cartography 권한 매핑 + 사용자 정의 매핑 파일 |
| 관계 | `ORIGIN`(CloudFront→NLB), `HAS_CREDS_OF`(EC2→역할, IMDS 조건), `CAN_REACH {port}`(네트워크 도달), `IS_INSTANCE` 대체(`providerID`가 비면 내부 IP 매칭) | logssey 보강 |

- 경로 쿼리는 `Internet`에서 출발해 `tier`가 붙은 데이터 노드까지 최대 6단계 탐색, 결과는 `attack_path`에 저장
- PR 분석은 이 그래프의 복사본(또는 메모리 안 부분 그래프)에 plan 변경을 얹어 계산하고 원본은 수정 안 함

| 항목 | 내용 |
|---|---|
| 작성자 | Jay(김종민) |
| 작성일 | 2026-10-12 |
