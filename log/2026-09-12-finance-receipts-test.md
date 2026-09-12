# 테스트 기록 — finance-service 영수증(receipts) API

- **일시**: 2026-09-12
- **대상**: kitchen-backend finance-service `/api/receipts` (등록/목록/상세/수정/삭제)
- **환경**: docker-compose(postgres/redis/kafka) + auth-service(3001) + api-gateway(8080) + finance-service(3002), 전부 기동 상태에서 진행. 프론트엔드 화면은 아직 없어 브라우저(Playwright) E2E는 이번 라운드 대상 아님 — 전부 백엔드 자동/수동 테스트.

## 1. 자동 테스트 — ReceiptServiceTest (Mockito 단위 테스트)

실행: `~/workspace/java_backend_workspace/kitchen-backend`에서 `./gradlew :finance-service:test`

| 테스트 | 결과 |
|---|---|
| register_정상적으로_등록된다 | ✅ PASS |
| register_FREE_플랜_월20건_초과시_한도초과_예외를_던진다 | ✅ PASS |
| register_지원하지_않는_카테고리면_VALIDATION_ERROR를_던진다 | ✅ PASS |
| list_직원은_본인_등록건만_조회한다 | ✅ PASS |
| list_관리자는_등록자_필터를_지정하지_않으면_전체를_조회한다 | ✅ PASS |
| update_직원이_본인_등록건을_24시간_이내_수정한다 | ✅ PASS |
| update_직원이_24시간_초과건_수정시_FORBIDDEN을_던진다 | ✅ PASS |
| update_직원이_타인_등록건_수정시_FORBIDDEN을_던진다 | ✅ PASS |
| update_관리자는_기간_제한없이_수정한다 | ✅ PASS |
| delete_소프트_삭제한다 | ✅ PASS |
| getDetail_존재하지_않으면_NOT_FOUND를_던진다 | ✅ PASS |
| getDetail_직원이_타인_등록건_조회시_FORBIDDEN을_던진다 | ✅ PASS |

**결과: 12/12 통과.** 이어서 `./gradlew test`(전체 모듈)로 회귀도 확인 — 기존 auth-service 포함 전부 통과, 깨진 곳 없음.

## 2. 수동 테스트 — curl (실제 auth-service·api-gateway·finance-service 연동)

ADMIN 계정 1개를 새로 가입시켜(FREE 플랜) 발급받은 accessToken으로 gateway(`:8080`)를 통해 진행.

| 시나리오 | 기대 | 결과 |
|---|---|---|
| 정상 등록(`POST /api/receipts`) | 201 + `createdAt` 값 포함 | ✅ |
| 미래 날짜로 등록 | 400 VALIDATION_ERROR | ✅ |
| `amount<=0`으로 등록 | 400 VALIDATION_ERROR | ✅ |
| 존재하지 않는 카테고리 값 | 400 VALIDATION_ERROR (500 아님) | ✅ |
| 토큰 없이 목록 조회 | 401 UNAUTHORIZED | ✅ |
| 목록 조회(`GET /api/receipts?from=&to=`) | 등록한 건 전부 표시, `meta.totalAmount`에 합계 | ✅ |
| 상세 조회 | 등록 내용 그대로 반환 | ✅ |
| 수정(`PATCH`) | 필드 반영된 응답 | ✅ |
| 삭제(`DELETE`, soft delete) | 200, 이후 상세조회 404 | ✅ |
| 삭제 후 목록·합계 | 삭제된 건 제외, `totalAmount` 재계산됨 | ✅ |
| FREE 플랜 20건째 등록 | 성공 | ✅ (누적 20건까지 정상 등록) |
| FREE 플랜 21건째 등록 | 403 `RECEIPT_PLAN_LIMIT_EXCEEDED` | ✅ |

## 3. 테스트 중 발견해서 즉시 고친 버그 2건

### 3-1. `createdAt`이 등록 응답에서 `null`로 나감

**증상**: `POST /api/receipts` 201 응답의 `createdAt` 필드가 `null`.

**원인**: `@CreationTimestamp`(Hibernate)는 실제 INSERT가 나가는 시점(flush)에 값을 채운다. `receiptRepository.save(receipt)`만 호출하고 트랜잭션이 아직 커밋 전인 상태에서 그 반환값을 바로 응답 DTO로 변환했기 때문에, flush가 안 일어나 `createdAt`이 비어 있었다. (`id`는 `GenerationType.UUID`라 flush 없이도 즉시 채워져서 대조적으로 정상으로 보였다.)

**수정**: `ReceiptService.register()`에서 `save()` → `saveAndFlush()`로 변경. `ReceiptServiceTest`도 `saveAndFlush` 모킹으로 갱신하고 `createdAt` not-null 단언 추가.

**어떻게 잡았나**: 유닛 테스트(Mockito)는 리포지토리를 통째로 목킹해서 이 타이밍 문제를 못 잡는다 — 실제 DB에 붙여서 curl로 응답 바디를 직접 확인했을 때만 드러났다. "유닛 테스트 통과 = 완료"가 아니라는 근거 사례.

### 3-2. (사전 예방) enum 요청 필드가 500을 낼 뻔한 설계 함정

`category` 요청 필드를 자바 enum 타입으로 바로 받으면, 잘못된 문자열이 왔을 때 Jackson이 `HttpMessageNotReadableException`을 던지는데 `GlobalExceptionHandler`가 이를 개별 처리하지 않아 catch-all(500)로 떨어진다 — 이전에 고친 `auth-service` `/api/auth/refresh` 500 버그와 같은 종류. 설계 단계에서 미리 인지하고 `category`를 `String`으로 받아 Service에서 `Enum.valueOf()`를 검증 후 `VALIDATION_ERROR`(400)로 던지도록 처음부터 그렇게 구현해서, 이번엔 실제로 500이 발생하기 전에 막았다. curl로 잘못된 카테고리 값을 보내 400이 나오는 것도 확인 완료(위 표).

## 요약

- 영수증 등록/목록(필터+합계)/상세/수정/삭제 핵심 플로우와 Free 플랜 한도·24시간 수정제한까지 전부 자동(12개 단위테스트) + 수동(curl) 이중 검증 완료.
- 실제 DB 연동에서만 드러나는 버그(`createdAt` null)를 하나 잡아 고쳤고, 같은 클래스의 버그(enum→500)는 설계 단계에서 예방.
- **안 한 것**: STAFF 계정으로 실제 HTTP 요청 테스트(본인 것만 조회/24시간 제한은 Mockito 단위 테스트로만 검증), 브라우저(Playwright) E2E(프론트 화면이 아직 없음).
