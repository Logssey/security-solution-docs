# ADR-S006 — 소스별 수집 경계와 Kubernetes 초기 포함

- 상태: **Accepted — 소스별 범위 표시와 K8s 내부 Secret 수집**, 배포 방식은 Proposed
- 결정일: 2026-10-09
- 기준: [PRD v0.3](../../01-requirements/PRD.md), FR-01, FR-03, SEC-01, [전체 시스템 구성](../system-overview.md)

## 이슈

하나의 플랫폼은 코드, CI/CD, 배포 환경 또는 외부 서비스 역할을 함께 할 수 있다. 연결 이름만으로 실제 읽을 수 있는 데이터가 정해지지 않는다. 특히 서비스가 K8s에 실행된다는 사실과 솔루션이 Kubernetes Secret을 읽을 수 있다는 사실은 다르다. AWS API와 Kubernetes API도 인증, 권한, 네트워크 범위가 다르다.

## 최종 결정

AWS, GitHub, Notion, K8s 연결을 각각의 승인 범위와 권한으로 수집한다. K8s 내부 Secret 수집은 초기 범위에 포함하며 AWS 또는 EKS 관리 API 수집으로 대체하지 않는다. 모든 연결에서 실제 수집 위치, 콘텐츠 유형, 이력 범위, 접근 실패와 미지원 영역을 기록한다.

## 수집 계약

| 연결 | 초기 수집 경계 | 별도 표시할 범위 |
|---|---|---|
| AWS | 승인된 리소스, 설정, 권한과 지원되는 사용 메타데이터 | 비밀 저장소 원문, 객체 내용, 로그 등 추가 읽기 권한이 필요한 데이터 |
| GitHub | 승인된 저장소와 수집 계약에 포함된 코드 및 CI/CD 설정 | 브랜치와 이력 범위, Actions 실행 로그, 아티팩트, 읽을 수 없는 저장 비밀 값 |
| Notion | 연결에 공유된 페이지와 지원되는 블록 콘텐츠 | 하위 페이지, 첨부 파일, OCR, 공유되지 않은 영역 |
| K8s | 승인된 네임스페이스의 내부 Secret | 워크로드 참조, projected 토큰, 외부 비밀 저장소와 노드 데이터 |

수집 대상의 존재와 비정상 노출 판정은 분리한다. AWS와 K8s 어댑터는 별도 권한 경계를 사용하고 K8s Secret의 접근 권한을 최소화한다.

| K8s 방식 | 검토 항목 |
|---|---|
| 외부 워커→API Server | 네트워크 접근, 인증, RBAC, kubeconfig 처리 |
| 클러스터 내부 수집기 | 배포, 업데이트, ServiceAccount, 송신 경로, 객체 범위 |
| 기존 인프라 API가 결과 제공 | 제공 정보의 완전성, 신뢰성, 인증, 누락, 최신성 계약 |

## 영향, 검증

Secret 목록만으로 projected ServiceAccount 토큰이나 외부 비밀 저장소의 모든 값을 수집했다고 할 수 없다. Pod exec, 노드 파일, etcd 직접 접근은 초기 기본 권한으로 가정하지 않는다. kubeconfig는 분석 데이터이며 포함된 exec 명령을 자동 실행하지 않는다.

AWS/EKS 관리 권한과 K8s Secret 읽기 권한을 따로 시험하고 허용, 거부, 일부 누락을 기록한다. GitHub와 Notion도 연결 범위 밖의 자료를 정상 0건으로 보고하지 않는다. [전체 구성 이미지](../system-overview.md)는 K8s 아이콘을 생략했지만 이 결정의 수집 범위는 그대로 적용한다.

근거: [K8s 조사](../../05-research/key-types-and-collection-scope.md), [공식 Secrets 문서](https://kubernetes.io/docs/concepts/configuration/secret/).
