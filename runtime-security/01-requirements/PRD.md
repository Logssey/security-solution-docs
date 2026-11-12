# PRD

**Re:Used Kubernetes 환경을 첫 대상으로 하는 Logssey 런타임 탐지 기능의 제품 요구사항.** 2026-10-10 AWS·Re:Used 검증 결과를 반영했다. 현재 문서는 구현 기준이며 Falco·Tetragon 설치와 운영 수치는 이후 PoC에서 검증한다.

## 1. 문서 목적

이 문서는 Re:Used Kubernetes 환경을 첫 대상으로 하는 Logssey 런타임 탐지 기능의 목적과 MVP 범위를 정한다.

## 2. 한 줄 설명

> Re:Used 컨테이너에서 발생하는 프로세스·파일·네트워크 행위를 수집하고, 해당 이벤트를 Kubernetes 자산과 연결하여 사용자가 발생 위치와 근거를 확인하고 조사 상태를 관리할 수 있게 한다.

## 3. 현재 상황

### 2026.10.10 기준 확인된 운영 환경

| 구분 | 확인값 | 런타임 설계에 미치는 영향 |
|---|---|---|
| Cloud | AWS 도쿄 리전 | AWS 자산 Context와 연결한다. |
| Kubernetes | Kubespray로 구성한 자체 관리 Cluster, Kubernetes `v1.35.4` | EKS 전용 기능을 전제로 하지 않는다. |
| Node | Control Plane 3대, Worker 3대, 외부 etcd 3대 | MVP 센서는 Worker 3대부터 배포한다. |
| Node OS | Ubuntu `24.04.4 LTS`, Kernel `7.0.0-1013-aws` | 설치 전 Falco·Tetragon의 해당 Kernel 지원을 확인한다. |
| Runtime·CNI | containerd `2.2.3`, Cilium `1.19.3` | Container ID와 Cilium Network Context를 함께 수집한다. |
| eBPF 준비 | `/sys/kernel/btf/vmlinux` 확인 | Modern eBPF 기반 PoC를 진행할 수 있다. |
| Re:Used | `reused` Namespace, `reused-api`·`reused-chat`·`reused-web` 각 2개 Pod | 세 Workload를 첫 Baseline 대상으로 삼는다. |
| Data Store | Private RDS PostgreSQL과 Private Redis EC2 | API·Chat의 승인된 내부 연결 대상으로 등록한다. |
| Pod 보안 설정 | 비root, 권한 상승 금지, Linux Capability 전체 제거, ServiceAccount Token 자동 마운트 해제 | RUN-003은 정상 발생이 거의 없는 고신뢰 신호로 검증한다. |
| Root Filesystem | 세 Workload 모두 쓰기 가능 | RUN-007을 활성 후보로 유지한다. |
| Network | CloudFront·WAF 뒤 Public NLB로 서비스 노출, 모든 Node SG Egress 허용, `reused` Namespace에 NetworkPolicy·CiliumNetworkPolicy 없음 | 외부 노출 Context를 연결하고 RUN-008 Baseline을 우선한다. |
| Node IAM | 모든 EC2가 같은 Role 사용, `AmazonEC2FullAccess`·`AmazonS3FullAccess`와 리전 내 Secrets Manager 조회 권한 보유 | Pod의 IMDS 접근이 Node 권한 노출로 이어질 수 있어 RUN-005를 필수 규칙으로 올린다. |
| IMDS | Token이 선택 사항이며 Hop Limit 2 | Pod에서 IMDSv1·v2 접근 가능성을 안전하게 확인해야 한다. |
| Image | Git Commit SHA Tag, ECR Push Scan 사용, Tag 변경 가능, 최근 20개 보존 | Event에는 Tag와 별도로 실행 중 Image Digest를 저장한다. |
| AWS Audit | 계정에 CloudTrail Trail 없음 | IMDS 접근 뒤 AWS API 사용까지 연결하는 것은 현재 불가능하므로 Context 부족으로 표시한다. |
| Runtime Sensor | Falco·Tetragon 미설치 | 설치 전·설치 중·정상·중단 상태를 구분한다. |
| Dashboard | React·Spring Boot·PostgreSQL, `/runtime` 메뉴 존재, 현재 관리자 계정만 지원 | MVP는 기존 Session·CSRF·ADMIN 권한에 맞춰 구현한다. |

API는 PostgreSQL, Redis, S3, SES SMTP, Kakao, LLM과 통신한다. Chat은 API와 Redis에 연결하고 Web은 Nginx로 정적 파일을 제공한다. API와 Chat의 비밀값은 External Secrets로 분리 주입된다.

### 구현 전에 남은 확인

- Falco와 Tetragon이 현재 Kernel·Kubernetes 조합을 공식 지원하는지 설치 전 확인
- Sensor DaemonSet에 필요한 Host Mount, Capability, RBAC와 보안 검토
- Kubernetes Audit Log에서 `pods/exec` 행위를 안정적으로 가져올 수 있는지 확인
- 실제 정상 트래픽을 기준으로 Workload별 외부 연결 Baseline 확정
- `attack-lab`을 운영 Cluster 안에 둘지 별도 Test Cluster에 둘지 팀 합의

## 4. 해결하려는 문제

| 문제 | 현재 한계 | 제품이 제공할 내용 |
|---|---|---|
| 컨테이너 내부 행위 확인 | 설정 점검만으로 실제 실행 행위를 알 수 없음 | 프로세스·파일·네트워크 이벤트 수집 |
| 발생 위치 파악 | 센서 경고만으로 상위 Workload를 바로 찾기 어려움 | Event와 Pod·Container·Deployment 연결 |
| 경고 반복 | 같은 행위가 여러 번 발생하면 조사 목록이 늘어남 | 반복 Event를 하나의 Finding으로 묶음 |
| 센서 중복 | Falco와 Tetragon이 같은 행위를 각각 기록할 수 있음 | 원본은 보존하고 하나의 Finding에 연결 |
| 일시적 자산 | 조사할 때 Pod가 이미 삭제될 수 있음 | 자산의 생성·삭제 시점과 과거 상태 보존 |
| 위험도 판단 | Event 하나만으로 최종 심각도를 정하기 어려움 | Context 준비 상태를 표시하고 후속 평가로 분리 |

## 5. 제품 목표

| ID | 목표 | 검증 방법 |
|---|---|---|
| G-01 | 실제 런타임 이벤트를 수집한다. | 테스트 Pod의 셸 실행이 Raw Event로 저장됨 |
| G-02 | 센서별 이벤트를 공통 구조로 변환한다. | Falco·Tetragon Sample이 RuntimeEvent로 변환됨 |
| G-03 | 이벤트를 Re:Used 자산과 연결한다. | Pod UID와 Container ID로 Workload까지 조회됨 |
| G-04 | 반복·중복 이벤트를 조사 단위로 묶는다. | 같은 셸 실행이 하나의 Finding과 여러 Event로 표시됨 |
| G-05 | 사용자가 근거와 조사 상태를 관리한다. | Finding 상세 조회와 상태·메모 변경이 가능함 |
| G-06 | 데이터가 없는 이유를 구분한다. | 정상·미설치·지연·중단·연결 실패가 구분됨 |

## 6. 대상 사용자

| 사용자 | 알고 싶은 것 | 수행할 행동 |
|---|---|---|
| 보안 담당자 | 어떤 행위가 어디에서 발생했는가 | Event 근거 확인, 조사 상태·메모 기록 |
| 인프라 운영자 | 센서와 수집 파이프라인이 정상인가 | Node별 센서 상태와 오류 확인 |
| Re:Used 개발자 | 자신의 Workload에서 무엇이 실행됐는가 | 정상 작업 여부 설명, 코드·Image 수정 |
| SCA 담당자 | 어떤 Image·파일·Process가 실제로 실행됐는가 | Reachability 입력으로 RuntimeEvent 사용 |

## 7. 핵심 개념

### Raw Event

Falco 또는 Tetragon이 생성한 Sensor 고유 형식의 데이터다. Collector에서 민감 필드를 마스킹한 뒤 저장하며, 저장된 Payload는 변경하지 않아 파싱 실패 시 재처리할 수 있게 한다.

### RuntimeEvent

센서별 원본을 Logssey 공통 구조로 변환한 관찰 사실이다.

```text
2026-10-08 10:00:00
attack-lab의 reused-api-lab Pod에서
uid=0 사용자가 /bin/bash를 실행했다.
```

### Finding

관련 RuntimeEvent를 사용자가 조사할 수 있게 묶은 단위다. 하나의 Finding에 반복 Event와 여러 센서의 Event가 연결될 수 있다.

### Sensor Priority와 Final Severity

- `sourcePriority`: Falco·Tetragon 규칙이 제공한 기본 중요도
- `finalSeverity`: 설정·권한·노출 Context를 결합해 Logssey가 판단할 값
- MVP에서는 `finalSeverity=null`, `evaluationStatus=PENDING_CONTEXT`를 허용한다.

## 8. 대표 사용 시나리오

### S-01. Re:Used API 형태의 테스트 Pod에서 셸 실행

1. `attack-lab`에 Re:Used API 형태의 테스트 Pod를 배포한다.
2. `kubectl exec`로 `/bin/bash`를 실행한다.
3. Falco가 셸 실행을 탐지한다.
4. Logssey가 원본 Event를 저장하고 RuntimeEvent로 변환한다.
5. Asset Resolver가 Pod·Container·Deployment를 연결한다.
6. 사용자가 화면에서 Process와 발생 위치를 확인한다.
7. 사용자는 `정상 작업` 또는 `조사 필요`로 상태를 기록한다.

**완료 조건:** Event 발생부터 화면 조회와 상태 변경까지 하나의 흐름으로 시연할 수 있다.

### S-02. 서비스 Process가 셸을 실행

1. 테스트 애플리케이션 Process가 `sh`를 자식 Process로 실행한다.
2. Falco와 Tetragon이 각각 Event를 생성한다.
3. 두 Event를 하나의 Finding에 연결한다.
4. 상세 화면에서 부모·자식 Process와 두 원본 Event를 확인한다.

**완료 조건:** 운영자의 직접 셸 실행과 서비스 Process의 셸 실행을 서로 다른 근거로 설명할 수 있다.

### S-03. 삭제된 Pod의 과거 Event 조사

1. 테스트 Pod에서 Event를 발생시킨다.
2. Pod를 삭제한다.
3. 사용자가 Finding 상세를 조회한다.
4. 화면은 Event 발생 당시 Pod·Image·Node·ServiceAccount를 표시한다.

**완료 조건:** 현재 삭제된 자산이라는 사실과 발생 당시 정보를 함께 확인할 수 있다.

## 9. MVP 범위

### 이벤트 수집

- Falco JSON 또는 HTTP Event 수집
- Tetragon `process_exec`·`process_exit` Event 수집
- Raw Event 원본 저장
- 파싱 실패 Event 격리와 재처리 상태 저장
- Worker Node 3대의 Sensor Heartbeat·Drop·Queue 상태 수집

### 이벤트 처리

- Falco Adapter와 Tetragon Adapter
- 공통 RuntimeEvent 변환
- Event와 Asset 연결
- 반복 Event와 센서 중복 묶음
- Finding 최초·마지막 발생 시각과 반복 횟수 갱신

### 자산 수집

- Kubernetes LIST·WATCH
- Cluster, Namespace, Node, Deployment, ReplicaSet, Pod, Container, ServiceAccount
- UID 기반 식별
- 삭제 자산의 과거 상태 보존
- Image Tag와 실제 실행 Digest를 함께 Snapshot으로 저장

### 사용자 기능

- 런타임 개요
- Finding 목록·상세
- 원본 Event 조회
- 자산 Context 조회
- 센서 상태 조회
- 조사 상태·담당자·메모 변경

## 10. MVP 제외 범위

- Final Severity 자동 계산
- Pod·Process·Network 자동 차단
- 운영 Cluster에서 실제 공격 수행
- 모든 Falco 기본 Rule 제공
- 모든 Tetragon TracingPolicy 제공
- 멀티 Cluster 통합 운영 검증
- 장기 Event Archive와 별도 Data Lake

## 11. 기능 요구사항

| ID | 요구사항 | 우선순위 | 검증 방법 |
|---|---|---|---|
| FR-RT-001 | 시스템은 Falco 원본 Event를 수집해야 한다. | Must | 셸 실행 Event 수신 |
| FR-RT-002 | 시스템은 Tetragon 원본 Event를 수집할 수 있어야 한다. | Should | `process_exec` Event 수신 |
| FR-RT-003 | 시스템은 민감 필드를 마스킹한 Raw Event를 불변으로 보존해야 한다. | Must | Sanitized Sample과 저장 Payload 비교 |
| FR-RT-004 | 시스템은 Raw Event를 RuntimeEvent로 변환해야 한다. | Must | Sample 기반 Mapping Test |
| FR-RT-005 | 시스템은 Kubernetes Asset을 LIST·WATCH로 수집해야 한다. | Must | Pod 생성·수정·삭제 반영 |
| FR-RT-006 | 시스템은 RuntimeEvent를 Asset과 연결해야 한다. | Must | Pod UID·Container ID 연결 |
| FR-RT-007 | 연결 실패 Event를 보존하고 재시도할 수 있어야 한다. | Must | Asset 선후 도착 시 재연결 |
| FR-RT-008 | 반복·중복 Event를 하나의 Finding으로 묶어야 한다. | Must | 동일 시나리오 반복 검증 |
| FR-RT-009 | 사용자는 Finding 목록과 상세를 조회할 수 있어야 한다. | Must | UI-210·UI-220 검증 |
| FR-RT-010 | 사용자는 조사 상태·담당자·메모를 변경할 수 있어야 한다. | Must | 상태 변경 후 재조회 |
| FR-RT-011 | 사용자는 센서와 수집 상태를 확인할 수 있어야 한다. | Must | Sensor 중단 시 상태 표시 |
| FR-RT-012 | 시스템은 Sensor Priority와 Final Severity를 구분해야 한다. | Must | `finalSeverity=null` 허용 |
| FR-RT-013 | 사용자는 같은 자산의 관련 보안 결과를 조회할 수 있어야 한다. | Could | 공통 Asset 계약 이후 검증 |
| FR-RT-014 | 사용자는 규칙별 Event·Finding·정상 행위 판정 수를 조회할 수 있어야 한다. | Must | UI-250 운영 지표 검증 |
| FR-RT-015 | 시스템은 Event 발생 당시 Image Digest와 Pod Security Context를 저장해야 한다. | Must | live Pod의 Image ID·Security Context와 Snapshot 비교 |
| FR-RT-016 | 시스템은 Workload별 정상 외부 연결 Baseline을 관리해야 한다. | Must | API·Chat·Web의 승인 목적지와 Test 목적지 구분 |

## 12. 비기능 요구사항

| ID | 구분 | 요구사항 |
|---|---|---|
| NFR-RT-001 | 지연 | 정상 상태에서 Event 발생 후 10초 안에 조회 가능해야 한다. |
| NFR-RT-002 | 중복 | 동일 Source Event 재전송은 중복 저장하지 않아야 한다. |
| NFR-RT-003 | 유실 | 파싱 실패와 처리 실패를 성공 Event와 구분해 재처리할 수 있어야 한다. |
| NFR-RT-004 | 보존 | MVP 기본값으로 Raw Event 30일, RuntimeEvent 90일, Finding·감사 이력 365일을 보존하며 환경변수로 조정할 수 있어야 한다. |
| NFR-RT-005 | 보안 | Sensor·Collector 간 인증과 전송 구간 보호를 적용해야 한다. |
| NFR-RT-006 | 마스킹 | Command Line·환경변수·원본 Event의 민감 값을 마스킹해야 한다. |
| NFR-RT-007 | 권한 | Asset Collector는 필요한 Kubernetes 조회 권한만 가져야 한다. |
| NFR-RT-008 | 설명 가능성 | 상태·묶음·평가 결과에 근거 필드를 제공해야 한다. |
| NFR-RT-009 | 이벤트량 | 규칙·Sensor·Workload별 이벤트량과 Drop 수를 측정하고 임계치 초과 시 Finding 생성보다 수집 상태 경고를 우선해야 한다. |

## 13. 보안 요구사항

- 운영 Credential과 실제 사용자 데이터를 테스트에 사용하지 않는다.
- 모든 재현은 `attack-lab` 또는 별도 테스트 Cluster에서 수행한다.
- Kubernetes Secret 값은 수집하지 않고 Metadata와 참조 관계만 저장한다.
- Raw Event의 환경변수·Command Line에 포함된 Secret을 저장 전에 마스킹한다.
- 운영 Event 수집 Endpoint는 Cluster별 Client Certificate를 사용한 mTLS로 인증한다.
- 조사 상태 변경과 Raw Event 조회는 Audit Log를 남긴다.
- Sensor Image와 Rule Version을 기록한다.
- 운영 Sensor는 Worker Node에만 먼저 배포하고 Control Plane·etcd·Redis Host 적용은 별도 검토한다.
- AWS 계정 번호, Secret ARN 전체값, Credential은 화면과 문서 예시에 노출하지 않는다.

## 14. 성공 지표

| 지표 | 1차 목표 |
|---|---:|
| 우선 시나리오 탐지 성공 | 5개 중 4개 이상 |
| Raw Event 저장 성공 | 발생시킨 Test Event의 100% |
| RuntimeEvent 변환 성공 | 지원 Event의 99% 이상 |
| Asset 연결률 | Kubernetes Metadata가 있는 Event의 95% 이상 |
| 화면 표시 지연 | 10초 이하 |
| Sensor 상태 감지 | 중단 후 2분 이내 |
| 중복 묶음 정확도 | Test Case 기준 95% 이상 |
| 정상 행위 Baseline | 24시간 이상 수집하고 규칙별 예외 후보를 기록 |
| 규칙 과다 발생 감지 | 1분 안에 Event 급증 상태 표시 |
| 운영 영향 | Re:Used 요청 오류율 증가 없음, Worker CPU 평균 증가 5%p 이하를 1차 기준으로 측정 |

목표값은 PoC 결과를 확인한 뒤 조정한다.

## 15. 완료 기준

- [ ] Re:Used 형태 테스트 Pod에서 셸 Event를 재현했다.
- [ ] Falco Raw Event와 RuntimeEvent Mapping Sample을 확보했다.
- [ ] Tetragon `process_exec` Mapping Sample을 확보했다.
- [ ] Kubernetes Asset의 생성·수정·삭제가 Inventory에 반영된다.
- [ ] Event가 Pod·Container·Deployment와 연결된다.
- [ ] 삭제된 Pod의 과거 정보를 조회할 수 있다.
- [ ] 반복 Event가 하나의 Finding으로 표시된다.
- [ ] Finding 상태와 메모를 변경할 수 있다.
- [ ] Sensor 중단을 정상 상태와 구분해 표시한다.
- [ ] 악성 행위 Test와 정상 행위 Baseline의 Event 차이를 비교했다.
- [ ] 규칙별 Event 수, Finding 수, 정상 행위 판정 수를 확인할 수 있다.

## 16. 의존성과 역할 경계

| 파트 | 런타임에서 제공 | 런타임에서 필요 |
|---|---|---|
| SCA | Process·Image Digest·Loader Event 후보 | CVE·취약 파일·SBOM |
| IaC | 실제 Asset과 런타임 상태 | 기대 구성과 변경 예정 Asset |
| IAM·RBAC | Event의 Pod·ServiceAccount | 권한 경로와 관리자 권한 여부 |
| Secret | 민감 경로 접근 Event | Secret 종류·소비 관계 |
| 대시보드 공통 | Runtime API와 화면 요구사항 | 기존 BIGINT Asset ID, Session·CSRF, 공통 Finding Schema |

## 17. 위험과 대응

| 위험 | 영향 | 대응 |
|---|---|---|
| Kernel·BTF 미지원 | 센서 설치 실패 | 설치 전 Node 점검, 지원 Driver 검토 |
| Event 과다 발생 | 저장·조회 부하 | Export Filter, Rule 선택, Grouping |
| Kubernetes Metadata 누락 | Asset 연결 실패 | UID 외 보조 Key, 재연결 Job |
| 정상 운영 행위 오탐 | 경보 피로 | Actor·Action·Target 기반 Exception |
| 센서 간 중복 | Finding 증가 | 원본 보존 후 Fingerprint 묶음 |
| 일시적 Pod 삭제 | 조사 Context 유실 | 유효 기간과 Tombstone 보존 |
| IMDSv1 허용과 공유 Node Role | Pod 침해 시 AWS 권한 노출 가능 | RUN-005 우선 검증, Metadata Option 개선은 IaC 개선안으로 전달 |
| `reused` Egress 무제한 | 외부 유출 경로를 사전에 제한하지 못함 | RUN-008 Baseline 후 NetworkPolicy 제안과 연결 |
| ECR Mutable Tag | 같은 Tag가 다른 Image를 가리킬 수 있음 | Pod Status의 Image Digest를 Event Snapshot에 저장 |
| 공용 Node Role의 넓은 권한 | Worker 침해 영향 확대 | Runtime Finding에 Node Role Context 연결, IAM 개선은 별도 파트에 전달 |
| 운영 Cluster에 Sensor 미설치 | 초기에는 실제 Event가 없음 | 화면에서 `미설치`를 정상 0건과 구분 |

## 18. 운영 전개 방식

### 1단계. Test Cluster 검증

- `attack-lab`에서 악성 행위와 정상 행위를 모두 재현한다.
- 규칙별 탐지 Event, 누락 필드, Event 수, 정상 행위 판정 수를 비교한다.
- RUN-001 셸 실행으로 수집부터 화면까지 전체 흐름을 먼저 완성한다.

### 2단계. 관찰 운영

- 운영 연결 후 2~3개월은 탐지만 수행한다.
- 담당자에게 Finding과 근거를 제공하고 `정상 행위`, `조사 중`, `조치 필요`를 기록하게 한다.
- 이 기간의 결과로 규칙 조건, 예외, Event Rate Limit, Finding 묶음 시간을 조정한다.

### 3단계. 후속 대응 검토

- 자동 차단은 관찰 운영 결과와 승인·복구 절차가 준비된 규칙만 별도 ADR로 검토한다.
- Tetragon Enforcement나 Pod 격리는 MVP 구현에 포함하지 않는다.

## 19. 구현 기준값과 미확정값

| 항목 | 현재 기준 | 상태 |
|---|---|---|
| 운영 Namespace | `reused` | 확인 |
| 대상 Workload | `reused-api`, `reused-chat`, `reused-web` | 확인 |
| 정상 Main Process | Java 21, Node.js 24, Nginx unprivileged | 코드 확인 |
| Worker | 3대 | 확인 |
| Kernel·BTF | `7.0.0-1013-aws`, BTF 있음 | 확인 |
| Kubernetes·Runtime | `v1.35.4`, containerd `2.2.3` | 확인 |
| CNI | Cilium `1.19.3` | 확인 |
| 앱 ServiceAccount | Workload별 전용 SA, Token 자동 마운트 해제 | 확인 |
| Sensor | 미설치 | 확인 |
| Test 위치 | `attack-lab` 또는 별도 Cluster | 팀 결정 필요 |
| Audit Log | `pods/exec` 연계 가능 여부 | 검증 필요 |
| Sensor Version·Driver | 현재 환경 지원표 확인 후 고정 | 검증 필요 |

## 20. 개발 스프린트

| 스프린트 | 구현 범위 | 완료 산출물 |
|---|---|---|
| 개발 1주차 | `attack-lab`, Falco, Cluster Collector, RUN-001 | 셸 실행 Raw Event 저장과 재현 Script |
| 개발 2주차 | RuntimeEvent Adapter, Kubernetes LIST·WATCH, Asset Resolution | Pod·Container·Deployment가 연결된 Event |
| 개발 3주차 | Finding Grouping, 목록·상세 API, UI-210·UI-220 | Event 발생부터 조사 상태 저장까지 end-to-end Demo |
| 개발 4주차 | Sensor 상태, 규칙 지표, Tetragon `process_exec`, 중복 처리 | Falco·Tetragon Event가 연결된 Finding과 운영 지표 |
| 관찰 운영 | RUN-002~RUN-008, 정상 행위 Baseline, 예외 조정 | 규칙별 Event량·정상 행위 판정 결과와 Rule Version |

각 주차는 앞 단계의 완료 산출물을 실제로 확인한 뒤 다음 단계로 넘어간다. Tetragon을 먼저 추가해 Falco end-to-end 완성을 늦추지 않는다.

## 21. 검증 근거

- GitHub: `Logssey/infrastructure`, `Logssey/gitops`, `Logssey/service-backend`, `Logssey/service-frontend`
- Dashboard: `Logssey/dashboard-backend`, `Logssey/dashboard-frontend`
- AWS CLI 읽기 전용 조회: EC2, SSM, IAM, ECR, RDS, ELBv2, VPC, CloudFront, WAF, Secrets Manager Metadata
- Cluster 읽기 전용 조회: Node Version, Deployment·Pod·ServiceAccount, Security Context, NetworkPolicy, DaemonSet
- 확인 시각: 2026.10.10

## 참고 자료

- [Falco Documentation](https://falco.org/docs/)
- [Tetragon Events](https://tetragon.io/docs/concepts/events/)
- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [OCSF](https://ocsf.io/)

| 항목 | 내용 |
|---|---|
| 작성자 | 김진호 |
| 작성일 | 2026-10-10 |
| 수정일 | 2026-10-14 |
