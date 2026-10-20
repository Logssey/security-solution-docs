# ADR-S006 — 원천별 수집 경계와 Kubernetes 권한

- 상태: **Accepted**
- 결정일: 2026-10-09
- 기준: [PRD](../../01-requirements/PRD.md), FR-01·FR-03, SEC-01

## 배경

하나의 플랫폼은 코드·CI/CD·배포 환경·키 발급자 역할을 함께 할 수 있다. 연결 이름과 서비스 배치만으로 실제 읽을 수 있는 데이터가 정해지지 않는다. AWS/EKS 관리 API와 Kubernetes API는 인증·권한·네트워크가 다르다.

## 결정

AWS·GitHub·Notion·K8s는 독립 승인 범위와 읽기 전용 권한으로 수집한다. K8s 내부 Secret은 Kubernetes API로 읽으며 AWS/EKS 관리 API 결과로 대체하지 않는다. 연결마다 위치·콘텐츠·이력·접근 실패·미지원 영역을 기록하고 대상 존재와 비정상 노출을 구분한다.

| 연결 | 읽기 경계 | 별도 승인·표시 |
|---|---|---|
| AWS | 승인된 계정·리전·자원·설정·권한·지원 사용 메타데이터 | 값·객체·로그는 메타데이터와 독립 범위 |
| GitHub | 승인된 저장소·브랜치의 코드와 CI/CD 설정 | 현재 파일·커밋 이력·Actions 로그·아티팩트·저장 비밀 값을 구분 |
| Notion | 연결에 공유된 페이지·지원 블록 | 하위 페이지·첨부·OCR·비공유 영역 |
| K8s | 승인 클러스터·네임스페이스의 Secret 객체 | projected 토큰·외부 저장소·노드·워크로드 참조 |

## 실행·배치

수집기는 Java 21 모듈로 구성해 API와 별도 워커에서 실행한다. AWS는 AWS SDK for Java 2.x·읽기 전용 IAM 역할, GitHub는 저장소·읽기 권한이 제한된 App 설치 토큰, Notion은 공유 페이지의 연결 토큰, K8s는 공식 Java 클라이언트·네임스페이스별 최소 RBAC를 사용한다. GitHub·Notion HTTP 호출은 승인 호스트만 허용한다.

워커가 승인 클러스터 API에 접근 가능한 환경에서는 외부 워커가 K8s 수집기를 호출한다. 네트워크 경계상 접근이 불가능하면 동일 수집기를 클러스터 내부 별도 워커에 배치하고 ServiceAccount를 사용한다. 어느 배치든 객체·네임스페이스·실패·누락·감사 계약은 동일하다. 구체적 배치 위치는 환경 네트워크·권한 설정에 따른다.

Kubernetes Java 클라이언트는 승인 클러스터 버전과 공식 호환표에 맞춰 버전 고정한다. TLS 검증을 해제하거나 미지원 API를 임의 호출하지 않는다. Pod exec·노드 파일·etcd 직접 접근은 기본 수집 권한에 포함하지 않으며 kubeconfig의 exec 명령을 실행하지 않는다.

## 근거와 결과

수집용 연결 자격 증명과 발견한 비밀은 분리한다. 원천별 허용·거부·부분 누락을 공통 형식으로 기록하고 범위 밖을 정상 0건으로 보고하지 않는다. 수집 계약과 작업·저장 모델은 공통이며 어댑터가 별도 스케줄러나 DB 계약을 만들지 않는다. 정기 수집·검증은 [ADR-S002](ADR-S002.md), 콘텐츠 세부 경계는 [수집원과 검증기 계약](../source-and-validator-contract.md)을 따른다.

참고: [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/), [공식 Java 클라이언트](https://github.com/kubernetes-client/java), [호환표](https://github.com/kubernetes-client/java/wiki/2.-Versioning-and-Compatibility), [AWS SDK for Java](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/), [GitHub App 권한](https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/best-practices-for-creating-a-github-app).
