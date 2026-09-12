# 작업 기록 — 영수증 프론트엔드 구현 + 품목(품목 배열) 설계 변경

- **일시**: 2026-09-12
- **범위**: kitchen-frontend 영수증 화면 신규 구현, 영수증 설계를 "단일 금액"에서 "품목 배열"로 변경(백엔드+프론트+KES 문서), 그 과정에서 발견한 버그 4건 수정

## 1. 배경 — 설계 변경 계기

기존 영수증 설계(오전 세션에서 구현)는 영수증 1건 = 지출 1건(단일 금액·단일 카테고리)이었다. 사용자가 "영수증 한 장에 품목이 여러 개 섞여 있을 수 있는데 이러면 설계 자체가 잘못된 것 아니냐"고 지적 — 확인해보니 같은 스키마의 `tax_invoice_items`(세금계산서 품목)는 이미 품목 배열 구조였는데 영수증만 단일 구조로 남아있었다. 사용자가 "품목별로 품목명·단가·수량·합금액, 마지막에 전체 합계"를 요구해 그 구조로 다시 설계했다.

## 2. 설계 변경 — 품목 배열

**KES 문서 갱신**: `05-database-schema.md`(receipt_items 테이블 추가), `06-api-design.md`(요청/응답 예시 items[] 구조로), `03-feature-spec.md`(A-1 입력 필드 갱신).

**백엔드(finance-service)**:
- Flyway `V2__add_receipt_items.sql` 추가(V1은 이미 적용돼 있어 손대지 않음) — `receipt_items(receipt_id, name, quantity, unit_price, amount, sort_order)`, `tax_invoice_items`와 같은 패턴
- `Receipt` 엔티티에 `@OneToMany items` 추가, `replaceItems(List<ItemInput>)`로 등록·수정 양쪽에서 품목 전체 교체 + 합계(quantity×unitPrice 합) 재계산. **클라이언트가 보낸 금액은 신뢰하지 않고 항상 서버가 재계산**
- `ReceiptDto`에서 `amount` 필드 제거, `items: List<ItemRequest>` 추가(`@NotEmpty` — 품목 0개면 400)
- `GET /api/receipts`에 `vendorName`(거래처 검색) 필터도 같이 추가 — `03-feature-spec.md` A-2에는 있었는데 처음 구현 때 `06-api-design.md` 초안 경로 표만 보고 누락했던 걸 뒤늦게 보강
- `ReceiptServiceTest` 13개 → 새 구조에 맞게 재작성 + 품목 관련 테스트 추가(품목 합계 계산, 품목 0개 거부)

**프론트엔드(kitchen-frontend)**:
- `receiptSchema`에 `items: z.array({name, quantity, unitPrice}).min(1)` — `amount` 필드 제거
- `ReceiptForm`에 `useFieldArray`로 품목 행 추가/삭제 UI, 실시간 합계 미리보기
- `ReceiptDetailPage`에 품목별 내역 표시, 수정 폼에 기존 품목 프리필

## 3. 신규 기능 — 영수증 슬립(인쇄용 뷰) + PDF 다운로드

상세 화면이 카드 여러 개로 흩어져 있어 가독성이 나쁘다는 피드백 → 실제 영수증처럼 보이는 단일 슬립 UI(`ReceiptSlip`, 모노스페이스 폰트 + 점선 구분선 + 품목 표 + 합계)로 재구성. "인쇄 / PDF 저장" 버튼은 `window.print()`를 호출 — 브라우저 인쇄 다이얼로그의 "PDF로 저장"을 그대로 다운로드로 활용한다(새 라이브러리 없이, 한글 폰트·이미지 캔버스 렌더링 이슈 회피). 슬립 외 네비게이션·버튼은 `print:hidden`으로 인쇄 시 제외 — Playwright로 `emulateMedia({media:'print'})` 렌더링해 슬립만 남는 것을 스크린샷으로 확인.

## 4. 프론트엔드 신규 구현 — 영수증 화면 전체

FSD 구조(`entities/receipt`, `features/receipts`, `pages/receipts`)로 목록(필터+합계+Free 플랜 이용량)/등록/상세(수정·삭제·인쇄) 화면을 처음부터 구현. 목록 필터(기간·거래처·카테고리)는 URL 쿼리 상태로 관리(`useReceiptListParams`) — 새로고침·공유·뒤로가기 자연스럽게 동작.

## 5. 테스트 중 발견해서 고친 버그 4건

### 5-1. (백엔드) `vendorName` 검색 시 500 — `function lower(bytea) does not exist`

프론트에서 처음 영수증 목록을 열자마자 500. 원인: JPQL `lower(:vendorName)`에서 `vendorName`이 null일 때 Postgres가 파라미터 타입을 추론 못 해 `bytea`로 잘못 해석. `cast(:vendorName as string)`으로 명시적 캐스팅해 해결. auth-service `/api/auth/refresh` 500 버그와 같은 클래스(파라미터 타입 추론 실패)의 재발.

### 5-2. (프론트) 타임존 버그 — 이번 달 1일이 8월 31일로 계산됨

`new Date(year, month, 1).toISOString().slice(0,10)`이 KST(UTC+9)에서 로컬 자정을 UTC로 변환하며 하루 밀림(로컬 09.01 00:00 → UTC 08.31 15:00). 목록 기본 기간, 영수증 폼의 "미래 날짜 불가" 검증 두 곳에서 같은 패턴의 버그를 발견 — `shared/lib/date.ts`에 `toLocalDateString`/`todayLocalDateString`을 만들어 한 곳으로 모으고 세 곳 모두(목록 필터, 폼 스키마, 폼 UI) 교체.

### 5-3. (프론트) 토큰 재발급 후 강제 재시도가 막혀 새로고침하면 로그인으로 튕김

`ky` 인스턴스의 `retry: 0` 설정이 401→refresh→`ky.retry()` 강제 재시도까지 0으로 묶어버려서, 재발급 자체는 성공해도 원 요청은 401로 남았다. 지금까지 E2E가 "로그인 직후 한 세션" 시나리오만 테스트해서 한 번도 걸린 적 없던 경로 — 영수증 페이지를 새로고침해서 세션 복구를 실제로 타보고서야 발견. `retry: { limit: 1, methods: [] }`로 수정.

### 5-4. (프론트) 위 버그를 고치자 드러난 2차 버그 — RequireAuth 레이스 컨디션

재시도가 실제로 성공하게 되자, `useMe()` 성공 직후 `authStore.isAuthenticated` 갱신(useEffect, 한 틱 늦음)보다 먼저 `RequireAuth`가 리다이렉트 분기를 평가해버리는 레이스가 드러남. `isAuthenticated` 대신 쿼리 자체의 `isLoading`/`isError`로 분기하도록 수정.

## 6. 검증

- 백엔드: `ReceiptServiceTest` 13개, 전체 백엔드 `./gradlew test` 회귀 — 전부 통과. curl로 품목 등록/조회/수정(교체) 직접 확인
- 프론트: 유닛 테스트 41개(`receiptSchema` 14개 포함) 통과, `pnpm exec tsc --noEmit` 무오류, 기존 Playwright `auth.spec.ts` 4/4 유지
- Playwright MCP로 브라우저 직접 조작 — 회원가입→로그인→영수증 등록(품목 2개)→목록(합계)→상세(품목 내역)→수정(품목 삭제 후 합계 재계산)→인쇄 미리보기(print 미디어 에뮬레이션)까지 전부 확인

## 7. 추가 검증 — 처음엔 빠져 있던 부분들

사용자가 "테스트 모두 완료했어?"라고 재차 확인해서 점검해보니 아래가 비어 있었다. 전부 이어서 채웠다.

- **거래처 검색 필터 실동작**: curl로 `vendorName=한성` → 1건, 존재하지 않는 이름 → 0건 확인(500 안 나는 것만 확인하고 실제 필터링 결과는 확인 안 했었음)
- **품목 있는 영수증 삭제**: curl로 DELETE(200) → 이후 GET(404) 확인
- **영수증 Playwright E2E 스펙 신규 작성** (`e2e/receipts.spec.ts`, 8개) — 지금까지 영수증은 MCP 수동 조작만 했고 `auth.spec.ts`처럼 회귀로 남는 자동화 테스트가 없었다. 등록(품목 2개)→목록·상세 합계 확인→품목 삭제 후 합계 재계산→삭제→클라이언트 검증 필수값 시나리오까지, 실행 결과 8/8 통과
- 이 스펙을 작성하는 과정에서 실제 버그를 하나 더 발견: 품목 수량·단가 입력의 HTML 네이티브 `min={1}`이 zod 유효성 검증보다 먼저 폼 제출을 막아버려서, 사용자에게 우리가 만든 한글 에러 메시지 대신 브라우저 기본 툴팁만 보이고 있었다. `min` 속성을 지우고 zod 메시지로 통일
- **백엔드 테스트 로그 가독성**: 사용자 요청으로 루트 `build.gradle`의 공용 `test` 블록에 `testLogging`을 추가 — 이제 `./gradlew test` 콘솔에 `ClassName > testName() PASSED/FAILED`가 케이스별로 바로 찍힌다(이전엔 "BUILD SUCCESSFUL"만 보이고 개별 결과는 HTML 리포트를 열어야 보였음). 모든 서비스에 공통 적용됨(auth-service로 재확인)

**여전히 안 한 것**: STAFF 계정으로 실제 HTTP/브라우저 테스트(여전히 Mockito 단위 테스트만).

## 요약

- 영수증 설계를 "단일 금액"에서 "품목 배열 합계" 구조로 다시 잡았다(백엔드·프론트·KES 문서 전부 갱신)
- 영수증 목록/등록/상세/수정/삭제 프론트 화면을 처음 구현했고, 상세 화면은 실제 영수증처럼 보이는 인쇄/PDF 다운로드 가능한 슬립으로 만들었다
- 이 과정에서 실제 브라우저 동작(새로고침 세션 복구, 타임존, null 파라미터 캐스팅, 네이티브 폼 검증 충돌)에서만 드러나는 버그 5건을 잡았다 — 전부 유닛 테스트만으로는 못 잡고 curl/브라우저 직접 조작으로 발견
- 영수증 E2E 자동화 테스트(8개)를 추가해 앞으로는 회귀로 남는다. 백엔드 테스트도 케이스별 로그가 콘솔에 바로 보이도록 설정 변경
