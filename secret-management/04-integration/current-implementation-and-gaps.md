# 현재 구현과 Secret 기능 연동 차이

확인일: 2026-10-13. 정적 확인 기준은 프론트 `cafca5c`, 백엔드 공유 main `14b1812`, 대시보드 문서 분리 전 main `23b8e2d`다. 로컬 `feat/secret-scheduler`의 미커밋·미푸시 작업은 공유 코드와 구분한다. 이번 문서 분리에서 빌드, 서버 재시작, 외부 수집, 마이그레이션 적용과 DB 이전은 수행하지 않았다.

## 1. 문서와 구현의 경계

[대시보드 설계 문서](https://github.com/Logssey/dashboard-design-docs)는 공통 기술 스택, 로그인, 공통 화면, 통합 현황과 공통 API를 관리한다. Secret의 제품 요구사항, 수집·탐지·검증, 키 모델, 평가·대응, 상세 화면과 전용 API는 이 문서 묶음에서 관리한다. 이 문서는 현재 코드와 정식 설계 사이의 차이를 기록하며 PRD와 ADR을 대신하지 않는다.

대시보드 문서의 단일 키 테이블, 임의 등급 판정, Admin PAT Mock 상세는 정식 설계로 옮기지 않았다. 원문 처리·분리 모델·상태·집계처럼 이미 정의된 원칙은 [데이터 계약](../03-architecture/data-contract.md), [DB 스키마](../03-architecture/database-schema.md), [실행 정책](../03-architecture/security-and-runtime.md)을 참조한다.

## 2. 현재 코드와 남은 연동

| 영역 | 현재 코드에서 확인한 내용 | 정식 설계 연결에 필요한 작업 |
|---|---|---|
| 화면 | `/overview`와 기본 `/secrets` 목록·수동 스캔 화면이 있다 | 정식 요약·대응 목록·상세·지정·체크리스트·실행 상태 조회 연결 |
| 판정 표시 | 문서에서 발견한 키는 미평가, 환경 미확인은 UNKNOWN으로 표시한다. 발급처 목록에는 위험 등급을 부여하지 않는다 | 유형별 완전성·유효성·노출·영향 근거와 평가 이력 연결 |
| 수집·탐지 | API가 Notion 페이지 읽기와 현재 Java 형식 탐지를 직접 수행한다 | Notion 페이지 읽기를 별도 워커 수집기로 정리하고 선정한 탐지 도구·정규화 파이프라인 연결 |
| 시작 동작 | local 샘플 적재기는 설정이 있으면 Notion 스캔을 호출한다 | 외부 스캔 호출을 제거하고 작업 등록·워커 실행으로만 연결 |
| 일반 이벤트 | 10/13 코드에서 Secret 자동 detection_events 생성과 등급 부여를 제거했다 | 정식 Secret 전용 기록과 새 통합 조회 연결 |
| 저장 | 공유 V2의 단일 credentials가 있으며 같은 키 재발견 시 storage_location을 갱신한다 | 고유 키와 여러 발견 위치를 분리하고 검사·평가·대응 이력을 연결 |
| 샘플 적재 | 전체 행 삭제 대신 조건부 정리 후 샘플을 재적재한다. 샘플 외에도 기존 credential_id 연결 이벤트를 삭제한다 | 이전할 기존 기록과 정리 조건을 대조하고 신규 FK·기록이 재적재로 손상되지 않는지 확인 |
| 예약 | 로컬 Java 워커에는 DB 기반 작업 예약만 있다. 작업은 PENDING이고 수집 실행기는 없다 | 작업 선점·리스 복구·한도 내 재시도·수집 소비와 실행 이력 구현 |
| 인증 | 기존 세션·CSRF와 ADMIN을 사용한다 | Secret 역할과 연결 범위 권한, 수행자·변경 감사 연결 |
| 상태·담당자 | 기존 Overview의 ownerTeam과 상태 계약을 유지한다 | 정식 모델의 개인 담당자·변경 이력·재검증 대기·예외와 해결 근거 구현 |
| 통합 현황 | 기존 100점 카드와 일반 탐지 조회가 남아 있다 | Secret 대응 건 합성 계약과 새 버전을 검증한 뒤 클라이언트 전환 |

소스 확인 위치: 프론트 `src/pages/SecretsPage.tsx`, `src/api/credentials.ts`, `src/api/types.ts`; 백엔드 `CredentialListController.java`, `CredentialListService.java`, `NotionLeakScanService.java`, `NotionPageReader.java`, `LeakCredentialRepository.java`, `LocalSampleDataLoader.java`, `V2__create_credentials.sql`. 코드 존재와 운영 지원·실제 연결 성공은 구분한다.

## 3. 전환 기간의 기존 API

아래는 현재 사용처를 이전하기 위한 구현 기록이다. 정식 API와 모델은 [API 계약](../03-architecture/api-contract.md)에 정의하며 기존 필드를 목표 계약으로 복사하지 않는다.

| 현재 경로 | 현재 계약과 전환 시 주의사항 |
|---|---|
| GET `/api/v1/credentials` | 필터는 issuer, status, foundOnly다. 요약은 totalCount, foundCount, candidateCount, inventoryCount이고 전체 기준이다. 응답에는 source, location, environment, componentState, severity, missingReason이 있다. source의 API/SCAN/MANUAL은 현재 구현 식별자이며 정식 수집 소스·발급자 구분으로 재해석하지 않는다 |
| POST `/api/v1/credentials/scan` | 세션·CSRF를 확인하고 API 프로세스에서 직접 Notion 스캔을 실행한다. 응답은 COMPLETED/PARTIAL/NOT_CONFIGURED/FAILED와 scannedPages, unreadablePages, foundKeys, newKeys, message, scannedAt이다. 현재는 공급자 실패도 HTTP 200의 status로 전달한다 |

현재 화면은 위 API를 사용한다. 예전 riskOnly, risks, atRiskCount, expiringSoonCount와 newEvents는 10/13 현재 계약에 없다. 형식 탐지의 CONFIRMED 표시는 현재 구성요소 판정이며 인증 성공이나 위험 등급을 뜻하지 않는다. 임시 AWS 키의 세션 토큰 등 유형별 필수 구성요소는 정식 계약에 맞춰 보완해야 한다.

현재 foundKeys는 스캔 결과 목록 길이이며 여러 페이지의 같은 키를 고유 키 한 건으로 센다고 보장하지 않는다. 전체 키·발견 위치·Finding 집계는 정식 모델에서 각각 정의한다. 기존 API의 직접 실행·오류 전달 방식은 전환 대상이며 비동기 정식 API의 동작 기준이 아니다.

대시보드 공통 인증·Overview 계약은 [공통 API 명세](https://github.com/Logssey/dashboard-design-docs/blob/main/05-api/api-spec.md)를 참조한다.

## 4. Notion 수집기 재사용 범위

현재 페이지 읽기 코드에는 검색 목록의 커서 페이지네이션, Markdown 본문 읽기와 제한된 호출 한도 재시도 처리가 있다. 본문 읽기는 현재 단일 응답이며 블록·본문 전체의 페이지네이션이 구현됐다고 표시하지 않는다. 404로 읽지 못한 페이지는 unreadablePages와 PARTIAL로 구분한다. 이 기능은 재사용 가능하지만 아직 공통 연결의 승인 범위·객체 버전·실행 ID·분리 모델 저장에 연결되지 않았다.

토큰이 접근할 수 있는 페이지 목록 전체를 승인 범위와 동일하게 취급하지 않는다. 명시적으로 승인한 페이지·지원 콘텐츠만 처리하도록 연결한다. 담당 기능과 공통 결과·테스트 경계는 [수집원 계약](../03-architecture/source-and-validator-contract.md)을 따른다. Admin PAT 목록은 이번 작업에서 제외한다.

## 5. 로컬 스케줄러와 DB 전환 상태

로컬 스케줄러는 Java 21 별도 모듈에 연결, 승인 범위, 작업, 검사 실행, 실행 범위 테이블 5개와 예약 로직을 구현했다. 6시간 기본 일정, 도래 작업 예약과 중복 방지까지 있으며 작업 소비·파일 읽기·탐지는 없다. 공유 main에 반영된 것으로 표시하지 않는다.

공유 V2와 미공유 스케줄러 V2가 같은 번호를 사용한다. 스케줄러를 V3로 옮기고 별도 개발 DB에서 전체 적용·필요 데이터 이전을 검증하는 절차는 [DB 스키마의 마이그레이션 절](../03-architecture/database-schema.md#6-보안-보존과-마이그레이션)에 둔다. 원래 개발 DB와 적용 이력은 보존한다. 현재 코드·DB 전환은 미완료이며 이번 문서 정리에서 번호나 데이터를 바꾸지 않았다.

## 6. 다음 구현과 협업 검토

1. 공통 DB 담당자가 마이그레이션 번호, 승인 범위 표현, 실행 결과와 데이터 이전 계약을 조율한다. 구체적인 담당자 이름은 아직 지정되지 않았다.
2. 워커 실행기는 도래 작업을 선점하고 공통 수집기를 호출한다. Notion 담당자는 페이지 수집기와 관련 테스트를 공통 계약에 맞춰 정리한다.
3. 정식 `/api/v1/secrets`의 수동 등록·실행 조회 세부 계약을 함께 검토하고 기존 화면 사용처를 전환한다.
4. [PRD](../01-requirements/PRD.md)의 단계별 GitHub·AWS·Notion·K8s 지원, 대응 기능과 영향 평가를 구현하고 각 단계의 실패·누락·원문 비노출을 확인한다. 이번 Notion 수집기 정리가 전체 지원 완료나 스프린트 일정 변경을 뜻하지 않는다.
5. 공통 DB·API·ADR 변경은 구현 전에 합의하고 공동 저장소 변경은 PR로 검토한다.

저장·새로고침·실패 복구 후에도 근거·상태·집계가 일치하는지 검증하며 제품 수용 기준은 코드가 존재한다는 사실만으로 충족되지 않는다.
