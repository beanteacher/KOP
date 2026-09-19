# 작업 기록 — 영수증→매출 문서 재정의 + 오픈뱅킹 입금매칭 + 세금계산서 자동발행 (재시작 체크포인트)

- **일시**: 2026-09-18 (2026-09-17 세션에서 이어짐)
- **상태**: 코드 작업 전부 롤백, 처음부터 순차로 재시작 예정. 이 파일은 `/clear` 이후에도 바로 이어갈 수 있게 하는 체크포인트다.

## 지금까지 있었던 일 (읽고 반복하지 말 것)

1. 사용자가 "영수증(모듈 A)"의 실제 의도를 밝힘: 지출 기록이 아니라 **매출 청구 문서**다. 흐름은 "발행 → 고객 전달 → 입금 → 입금 확인되면 세금계산서 자동 발행". 오픈뱅킹 API로 입금을 확인하고, 세금계산서 발행 방식은 무료=엑셀 수동/유료=홈택스 API 자동처럼 확장 가능하게 설계하기로 함 (아래 "확정된 설계 방향" 참고 — 이 부분은 여전히 유효한 결정이라 재작업 시 그대로 써도 됨).
2. 이 작업을 4개 트랙(A 영수증 재정의 / B 오픈뱅킹 어댑터 스켈레톤 / C 거래처·세금계산서 스켈레톤 / D 프론트 반영)으로 나눠 **git worktree로 격리해 병렬 서브에이전트 4개를 동시에** 돌렸다.
3. **사용자가 Pro 플랜이라 rate limit이 계속 걸려서 병렬 처리가 안 된다고 판단** — 실제로 세션 도중 여러 차례 rate limit(429)과 스톨(10분 무응답)로 에이전트들이 반복해서 중단됐다. 사용자가 "하나씩 차근차근 하자"고 지시.
4. 그 다음 사용자가 **지금까지 트랙에서 만든 코드를 전부 제거하고, 처음부터 순차로(clear 후) 다시 작업하기로** 결정. 확인 결과 4개 트랙 전부 `main`에서 분기된 별도 브랜치였고 `main`은 항상 깨끗한 상태였음 — 그래서 4개 worktree(`KOP-BACKEND-track-{a,b,c}`, `KOP-FRONTEND-track-d`)와 브랜치를 전부 `git worktree remove` + `git branch -D`로 삭제 완료. **현재 `KOP-BACKEND`·`KOP-FRONTEND` 둘 다 `main`, 클린 상태.**

## ⚠️ 다음 세션 진행 원칙

- **서브에이전트 병렬 스폰 금지.** 이 프로젝트는 Pro 플랜이라 동시에 여러 에이전트를 돌리면 rate limit에 걸린다. 작업은 메인 세션이 순차로 하나씩 진행한다 (필요하면 서브에이전트를 쓰더라도 한 번에 하나씩, 끝나고 다음).
- 코드는 전부 사라졌으니 **파일이 존재한다고 가정하지 말고 처음부터 다시 만든다** (아래 "확정된 설계 방향"은 설계 결정이지 이미 구현된 코드가 아니다).
- 작업 순서는 사용자에게 다시 확인하되, 특별한 의견이 없으면 다음 순서를 제안한다: (1) 영수증 재정의(백엔드) → (2) 영수증 프론트 반영 → (3) 오픈뱅킹 어댑터 스켈레톤 → (4) 거래처·세금계산서 스켈레톤 → (5) 매칭 로직 + 자동발행 통합.
- 규모가 크므로 CLAUDE.md의 Loop Engineering(작업 단위 쪼개기 + 검증 게이트)을 그대로 적용한다 — 한 번에 다 하려 하지 말고 작은 단위로 커밋해가며 진행.

## 확정된 설계 방향 (여전히 유효 — 재작업 시 이 계약대로)

- `receipts.category`(재료비/운반비/인건비/장비비/기타) 폐기 → `payment_status`(`PENDING`/`PAID`/`CANCELED`, 기본 PENDING) + `paid_at` + `matched_transaction_id` 추가. 전이는 `PENDING→PAID` / `PENDING→CANCELED`만 허용, 위반 시 `409` (`RECEIPT_NOT_PENDING`류 에러코드).
- 신규 엔드포인트(관리자 전용): `POST /api/receipts/{id}/mark-paid`(body `{transactionId?}`), `POST /api/receipts/{id}/cancel`. 목록 필터 `?category=` → `?paymentStatus=`. 수정/삭제는 `PENDING`일 때만(+ 기존 24시간 규칙).
- 은행 연동은 인터페이스로 분리: `BankTransactionGateway.fetchTransactions(BankAccount, from, to) -> List<BankTransaction>`, 기본 구현 `MockBankTransactionGateway` (사용자가 오픈뱅킹 개발자센터 아직 미가입 — 가입 전까지는 Mock으로 개발). `bank_accounts` 테이블 필요(company_id UNIQUE, fintech_use_num/access_token/refresh_token은 암호화 저장). 암호화는 auth-service의 `AesEncryptor`(계좌번호 암호화용, 09-security.md 예외조항)를 공용(common) 모듈로 옮겨서 재사용하는 게 맞다(그 클래스 주석에 "다른 서비스 필요해지면 옮긴다"고 이미 적혀 있음). `.env` 변수: `OPENBANKING_BASE_URL`/`OPENBANKING_CLIENT_ID`/`OPENBANKING_CLIENT_SECRET`.
- 입금-영수증 매칭 기준: 금액 완전일치 + 입금자명이 거래처명(`vendorName`)을 포함하면 자동 매칭, 애매하면 관리자 수동(`mark-paid`)으로 폴백.
- 세금계산서(모듈 B, 코드 전무 — 문서만 있음: `03-feature-spec.md`/`05-database-schema.md`/`06-api-design.md`)는 문서 스펙대로 `clients`/`tax_invoices`/`tax_invoice_items` 구현 + `tax_invoices.source_receipt_id`(자동생성 근거)/`issue_method`(`EXCEL`/`HOMETAX_API`) 컬럼 추가. 발행 방식은 `TaxInvoiceIssuanceGateway` 인터페이스로 분리 — `ExcelIssuanceGateway`(지금 유일하게 동작, 홈택스 bulk 엑셀 자동생성)만 구현, `HometaxApiIssuanceGateway`는 스텁(유료 플랜용, 나중에).
- 오픈뱅킹 가입 절차(사용자가 직접 해야 함, 진행 중일 수 있음): developers.openbanking.or.kr 회원가입 → 서비스 신청 → 테스트베드 Client ID/Secret 발급(심사 없이 즉시). 실제 운영 전환(이용기관 등록 심사)은 그다음 단계, 아직 멀었음.

## 재개 시 첫 행동

1. 사용자에게 "어느 부분부터 다시 시작할지" 확인 (위 제안 순서 참고).
2. 병렬 서브에이전트 쓰지 말 것.
3. 작은 단위로 나눠서 진행 + 매 단위마다 커밋.

## 진행 상황 갱신 (2026-09-18 10:07am)

**(1) 영수증 재정의(백엔드) 완료.** `KOP-BACKEND` main에 커밋됨 (`fb0be76`, push는 안 함 — 로컬만).

- `receipts.category`/`ReceiptCategory` enum 완전 제거.
- Flyway V4: `payment_status`(PENDING/PAID/CANCELED, 기본 PENDING) + `paid_at` + `matched_transaction_id` 추가.
- `POST /api/receipts/{id}/mark-paid`, `POST /api/receipts/{id}/cancel` 신규 — 관리자 전용, PENDING 상태에서만 전이 가능(그 외 `409 RECEIPT_NOT_PENDING`, STAFF가 호출 시 `403 FORBIDDEN`).
- update/delete는 PENDING일 때만 허용(+ 기존 STAFF 24시간 규칙 유지) — 위반 시 `409 RECEIPT_NOT_PENDING`.
- 목록 필터 `?category=` → `?paymentStatus=`.
- `ReceiptServiceTest` 26개 전부 GREEN. 로컬 Postgres(`kop-backend-postgres-1`)에 V4 실제 적용 + `finance-service` 부팅해 Hibernate `ddl-auto=validate`로 엔티티-스키마 일치 확인 완료(수동 검증, 자동화된 통합 테스트는 없음 — 이 레포에 애초에 없었음).
- `matched_transaction_id`는 컬럼만 추가, 실제 채우는 로직(오픈뱅킹 매칭)은 다음 트랙.

## 진행 상황 갱신 (2026-09-18 10:22am)

**(2) 영수증 프론트 반영 완료.** `KOP-FRONTEND` main에 커밋됨 (`add9aed`, push는 안 함 — 로컬만).

- `category`/`ReceiptCategory` 타입·필터·칩·폼필드 전부 제거 → `paymentStatus`(PENDING/PAID/CANCELED) 기반으로 교체.
- `ReceiptPaymentStatusChip`(구 `ReceiptCategoryChip`) 상태별 색상(PENDING 회색/PAID 초록/CANCELED 빨강), `ReceiptDetailPage`에 입금확인·영수증취소 버튼(관리자+PENDING일 때만) 추가, PENDING 아니면 수정/삭제 숨김+안내문구.
- `useMarkReceiptPaid`/`useCancelReceipt` 훅, `receiptApi`에 엔드포인트 추가.
- 검증: `tsc`/`vite build`/vitest(75개) 전부 통과 + 기존 e2e `receipts.spec.ts` 5개 전부 통과. 추가로 로컬 스택(auth-service/api-gateway/finance-service 전부 현재 코드로 재기동, postgres/redis/kafka는 기존 docker-compose 그대로) 띄워 Playwright로 직접 로그인→등록→입금확인→취소 전체 플로우를 실제 브라우저로 확인함(스크린샷 2장, 상태 배지·필터 서버 왕복까지 정상).
- **주의**: 기존에 떠 있던 finance-service(3002)/api-gateway(8080) 프로세스가 어제 `com.kitchensys`→`com.kop` 네임스페이스 마이그레이션 이전의 스테일 빌드였음(계속 떠 있었는데 재기동이 안 됐던 것으로 보임) — 이번에 kill 후 현재 코드로 재기동함. 지금 로컬에 auth-service(3001)/api-gateway(8080)/finance-service(3002)/frontend(5173)가 전부 떠 있는 상태.

## 진행 상황 갱신 (2026-09-18 10:47am)

**(3) 오픈뱅킹 어댑터 스켈레톤 완료.** `KOP-BACKEND` main에 커밋됨 (`a7409d8`, push는 안 함 — 로컬만).

- `AesEncryptor`를 auth-service → common 모듈로 이동(그대로 재사용, auth-service 동작/테스트 불변). common에 첫 유닛테스트 추가.
- Flyway V5: `finance.bank_accounts`(company_id UNIQUE, fintech_use_num/access_token/refresh_token은 AesEncryptor로 암호화 저장).
- `BankAccount` 엔티티 + `BankAccountRepository`, `BankTransactionGateway` 포트 + `BankTransaction` 값 객체 + `MockBankTransactionGateway`(항상 빈 목록 반환 — 가짜 데이터로 착시 만들지 않음).
- `application.yml`에 `openbanking.*`(base-url/client-id/client-secret/redirect-uri)와 `app.encryption.*` 자리 마련 — 셸 환경변수로 주입(이 레포는 `.env` 파일 안 씀). 콜백 URL 기본값은 사용자가 실제 등록한 `http://localhost:5173/settings/bank-account/callback`.
- 검증: `common`/`auth-service`/`finance-service` 전부 테스트 GREEN + 로컬 Postgres에 V5 실제 적용해 Hibernate `ddl-auto=validate`로 엔티티-스키마 일치 확인.
- **사용자가 진행 중인 외부 절차**: 금융결제원 오픈뱅킹 개발자센터(developers.openbanking.or.kr) 가입 → 테스트베드 Client ID/Secret 발급 → Callback URL `http://localhost:5173/settings/bank-account/callback` 등록.
- **명시적 비범위(다음 트랙으로 미룸)**: 실제 HTTP OAuth 클라이언트 구현체(발급받은 Client ID/Secret으로 실제 API 호출), `/settings/bank-account/callback` 콜백 처리 엔드포인트/프론트 페이지, 계좌 등록 플로우(`BankAccountService`) — 전부 아직 코드 없음. 지금은 인터페이스+스키마+Mock만 있는 "스켈레톤" 상태.

## 진행 상황 갱신 (2026-09-18 11:10am)

**BankAccountService(오픈뱅킹 계좌 연결 OAuth) 완료.** `KOP-BACKEND` main에 커밋됨 (`63e162a`, push는 안 함 — 로컬만).

- `OpenBankingClient`: 인증 URL 생성(`GET /oauth/2.0/authorize`), 토큰 교환(`POST /oauth/2.0/token`), 계좌 조회(`GET /v2.0/user/me`) — 필드명은 금융결제원 공식 API 명세서 PDF + 독립 예제 2개로 교차검증.
- `OpenBankingStateSigner`: OAuth state를 companyId+만료시각 HMAC 서명으로 자기검증(서버 저장 없이 CSRF 방지, TTL 10분).
- `BankAccountService`(관리자 전용): 등록 시 토큰/핀테크이용번호를 AesEncryptor로 암호화 저장, 이미 연결된 회사는 재연결로 갱신. `GET/POST /api/bank-accounts`, `GET /api/bank-accounts/authorize-url`.
- **api-gateway에 `/api/bank-accounts/**` 라우트 추가** — 원래 빠져있던 걸 실제 브라우저 호출로 발견(라우트 없으면 전부 404).
- 검증: 유닛테스트 14개 GREEN + 실제 `.env`에 채워진 Client ID로 finance-service/api-gateway 재기동 후 로그인된 브라우저 세션에서 `/api/bank-accounts/authorize-url`·`/api/bank-accounts` 실제 호출해 200 확인(authorize-url에 진짜 client_id/state 포함 확인). **실제 은행 로그인(계좌 등록 완료)은 사용자 본인 인증수단이 필요해 진행 안 함** — `POST /api/bank-accounts`(토큰 교환)는 유닛테스트로만 검증됨. 사용자가 브라우저에서 직접 "계좌 연결" 플로우를 끝까지 타보고 실제 등록되는지 확인 필요.
- **아직 없음(비범위)**: 프론트 `/settings/bank-account` 페이지(연결 버튼 + 콜백 처리), 토큰 refresh 로직(만료 시 자동 갱신), 실제 거래내역조회(`BankTransactionGateway`의 진짜 구현체 — 지금도 Mock).

## 진행 상황 갱신 (2026-09-18 4:29pm)

**프론트 `/settings/bank-account` 페이지 + 콜백 처리 완료.** `KOP-FRONTEND` 커밋 `bfd0720`. `KOP-BACKEND`에 버그 수정 1개 추가 커밋 `ddf704a`(로컬만, push 안 함).

- **실제로 발견하고 고친 버그**: state 파라미터가 "32-byte fixed random string"이어야 하는데 HMAC 자기서명 방식이라 ~107자였음 → 오픈뱅킹 테스트베드가 매번 `O0001`/`3000103`으로 거부. `OpenBankingStateSigner`를 32자 랜덤 문자열 + 서버 메모리(ConcurrentHashMap) 매핑 방식으로 재설계(1회용, TTL 10분). 한계: 인스턴스 로컬 메모리라 재시작 시 미완료 요청 무효화, 다중 인스턴스 미지원(나중에 Redis로 옮길 것 — auth-service는 이미 Redis 씀, finance-service는 아직 없음).
- **api-gateway `/api/bank-accounts/**` 라우트 누락도 이전 턴에 발견해서 고쳐둠** (63e162a에 포함).
- 프론트: `/settings/bank-account`(연결 상태 + 연결 버튼, 관리자 전용) + `/settings/bank-account/callback`(code/state 파싱 → 등록 → 리다이렉트, 에러 처리 포함) + 사이드바 메뉴.
- **실제 브라우저로 전체 플로우 확인**: 연결 버튼 클릭 → 실제 오픈뱅킹 본인인증 방식 선택 화면(금융인증서/공동인증서/휴대전화 인증)까지 정상 도달. 콜백 페이지의 "코드 없음"/"state 무효" 에러 처리도 확인. **실제 은행 로그인(계좌 등록 완료)은 사용자 본인의 인증수단이 필요해 여기서 진행 안 함 — 사용자가 직접 `/settings/bank-account`에서 "오픈뱅킹으로 계좌 연결" 버튼을 눌러 끝까지 테스트해봐야 함.**

**다음 후보**: (a) 사용자가 직접 계좌 연결을 실제로 완료해보고 결과 확인(제일 먼저 해볼 만함 — 방금 고친 버그가 진짜 고쳐졌는지 최종 확인). (b) (4) 거래처·세금계산서 스켈레톤. (c) 실제 `BankTransactionGateway` 구현체 + 토큰 refresh(지금도 거래내역은 Mock).

## 진행 상황 갱신 (2026-09-18 밤 ~ 2026-09-19)

사용자가 계좌 연결을 실제로 끝까지 테스트 완료(성공 확인). 그 다음 "다음 단계 진행하자" + "거래내역 조회 api도 진행하고" 지시로 두 트랙을 순차 진행:

**실제 거래내역조회 API 완료.** `KOP-BACKEND` 커밋 `4495f08`(로컬만).
- `OpenBankingClient.fetchTransactionHistory()`(`GET /v2.0/account/transaction_list/fin_num`, 입금만) + `refreshToken()`. 공식 API 명세서 원문을 다시 확인해 `BankTransaction` 설계를 고쳤다 — 이 API는 거래 단위 고유번호를 안 준다(이전 설계는 실재하지 않는 필드를 가정하고 있었음), 입금자명은 `print_content`(통장인자내용)를 쓴다.
- `RealBankTransactionGateway`(`openbanking.gateway-provider=real`일 때만 활성화, 기본은 여전히 Mock) — 토큰 만료 5분 전이면 자동 refresh.
- `OPENBANKING_INSTITUTION_CODE` 필요(Client ID와 다른 값, 거래고유번호 생성용) — 아직 사용자가 안 채움, real로 전환 시 채워야 함.

**(4) 거래처·세금계산서 스켈레톤 완료** (스켈레톤 수준을 넘어 B-1~B-4 전부 동작). `KOP-BACKEND` 커밋 `135f2b3`, `a834543`(로컬만).
- `clients`(거래처) CRUD 전부: 등록/목록(검색+상태)/상세/수정/삭제(세금계산서 이력 있으면 409, PATCH로 INACTIVE 전환이 유일한 비활성화 경로). `receipts.client_id`에 이제 실제 FK 추가(V1부터 미구현이라 없었던 것).
- `tax_invoices`/`tax_invoice_items` 스키마 + 엔티티 — `issue_method`(EXCEL/HOMETAX_API) 컬럼 추가(문서 초안엔 없던 확장, 체크포인트 설계 반영).
- `TaxInvoiceIssuanceGateway` 포트 + `ExcelIssuanceGateway`(Apache POI로 홈택스 bulk 업로드 xlsx 생성 — 컬럼 구성은 스펙 문서 표 그대로, **홈택스 공식 템플릿과 바이트 단위로 검증된 건 아님**, 실제 업로드해보고 조정 필요할 수 있음). `HometaxApiIssuanceGateway`(유료 플랜용)는 아직 클래스도 없음 — enum 값만 자리 있음.
- `TaxInvoiceService`: 작성(품목 최대 16개, DRAFT/COMPLETED만)/이력조회(공급가액·세액 합계)/엑셀발행(→EXCEL_DOWNLOADED)/수정발행(새 건+원본 REVISED)/취소(사유 필수, CANCELED 건은 추가 조작 불가).
- `common.PageMeta`에 `taxAmount` 필드 추가(기존 `totalAmount`와 같은 패턴, 다른 소비자 영향 없음 확인).
- **검증**: 유닛테스트 20개(ClientServiceTest 9 + TaxInvoiceServiceTest 11) + ExcelIssuanceGatewayTest(생성된 xlsx를 다시 읽어 바이트 단위 검증) 전부 GREEN. 로컬 스택 실제 호출로 거래처 등록→세금계산서 작성(06-api-design.md 예제와 금액 정확히 일치)→엑셀 다운로드(실제 200+xlsx)→이력조회→거래처 삭제 차단(409)→취소까지 풀 플로우 확인.
- **프론트는 전혀 없음** — 거래처·세금계산서 둘 다 백엔드 API만 있고 화면이 없다. `AppShell` 사이드바엔 아직 "준비중"으로 표시돼 있을 것.

**다음 후보**: (a) 프론트 — 거래처 관리 화면 + 세금계산서 작성/이력조회 화면(지금 API만 있고 테스트할 화면이 없음). (b) (5) 입금-영수증 자동 매칭 + 세금계산서 자동발행 통합(마지막 단계, `tax_invoices.source_receipt_id` 컬럼도 이때 추가 예정 — 아직 없음). (c) 실제 `BankTransactionGateway`를 real로 전환해서 진짜 거래내역 받아보기(institution-code 필요).

## 진행 상황 갱신 (2026-09-19)

**거래처·세금계산서 프론트 완료.** (a)를 골라 진행 — (b)/(c)는 아직 안 함. `KOP-FRONTEND` 커밋 `8ef5087`(거래처), `1631c36`(세금계산서).

- **거래처**: 목록(검색+상태필터+URL동기화)/등록/상세/수정/활성화·비활성화 토글/삭제. 삭제는 백엔드 409(CLIENT_HAS_INVOICES)를 그대로 토스트로 보여준다.
- **세금계산서**: 이력조회(기간·거래처·상태 필터 + 공급가액·세액 합계 요약)/작성(거래처 선택+품목 최대 16개)/상세(품목 테이블+금액 요약)/**엑셀 다운로드**(ky blob→Object URL 방식, 실제 파일 저장 확인)/수정발행(새 건 생성+원본 REVISED 전이)/취소(`window.prompt`로 사유 입력 — 다이얼로그 컴포넌트 없어서 최소구현).
- `shared/api/types.ts PageMeta`에 `taxAmount` 필드 추가(백엔드 확장과 동기화).
- `AppShell`에 "거래처"·"세금계산서" 메뉴를 관리자에게 실제 링크로 노출(STAFF는 계속 "준비중").
- **실제로 발견하고 고친 버그**: 수정발행 후 새 레코드 URL로 navigate해도 React Router가 컴포넌트를 리마운트하지 않아서(`:id` 파라미터만 바뀜) `isRevising` 상태가 안 풀려서 새 글도 계속 편집 폼으로 보이던 문제 — `navigate` 직전에 `setIsRevising(false)` 추가해서 해결.
- **검증**: tsc/vitest(83개)/build 통과 + 실제 브라우저로 거래처 등록→목록→상세→수정→비활성화→삭제차단(409)→재활성화, 세금계산서 작성→상세→엑셀다운로드(실제 파일)→상태전이 확인→수정발행(금액변경 반영+원본 REVISED 확인)→취소(사유·시각 표시 확인)까지 전체 플로우 확인.

**다음 후보**: (a) (5) 입금-영수증 자동 매칭 + 세금계산서 자동발행 통합(원래 로드맵 마지막 단계 — `tax_invoices.source_receipt_id` 컬럼 추가 필요). (b) 실제 `BankTransactionGateway`를 real로 전환해서 진짜 거래내역 받아보기(institution-code 필요, 사용자가 아직 안 채움).
