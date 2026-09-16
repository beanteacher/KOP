# 06 — API 설계

> **문서 상태**: 초안 (Draft)
> **작성일**: 2026-09-11
> **버전**: v0.1
> **연관 문서**: `02-user-stories.md` · `03-feature-spec.md` · `04-system-architecture.md` (라우팅) · `05-database-schema.md`

---

## 목차

- [공통 규약](#common)
- [Auth API](#auth)
- [Finance API — 거래처](#clients)
- [Finance API — 영수증](#receipts)
- [Finance API — 세금계산서](#tax-invoices)
- [Drawing API](#drawings)
- [Product API](#products)
- [Dashboard API](#dashboard)

---

<a id="common"></a>

## 공통 규약

### 버전·라우팅
URL 버저닝은 두지 않는다(v1 고정). 서비스 라우팅은 `04-system-architecture.md`의 API Gateway 표를 그대로 따른다 — 아래 경로는 전부 Gateway 뒷단 실제 서비스 경로다.

### 인증
- `Authorization: Bearer <accessToken>` — Access Token 유효기간 1시간 (X-3)
- Refresh Token은 응답 JSON 바디에 담지 않는다 — 로그인·재발급 성공 시 `Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Strict`로만 내려간다(XSS로 탈취 불가하게 하기 위함, `09-security.md` 참고). `/api/auth/refresh`·`/api/auth/logout`은 이 쿠키를 읽어 동작하므로 별도 요청 바디가 필요 없다.
- 인증 불필요 엔드포인트: 회원가입·로그인·비밀번호 재설정·초대 수락·도면 공유 링크 뷰어(`GET /api/drawings/shared/{token}`)
- 그 외 전부 로그인 필요. 관리자 전용 엔드포인트는 각 표의 "권한" 컬럼에 `관리자`로 표기

### 응답 포맷

성공(단건):
```json
{ "data": { "...": "..." } }
```

성공(목록 — 페이지네이션 공통):
```json
{
  "data": [ { "...": "..." } ],
  "meta": { "page": 1, "size": 20, "totalElements": 134, "totalPages": 7 }
}
```

실패:
```json
{ "error": { "code": "RECEIPT_PLAN_LIMIT_EXCEEDED", "message": "이번 달 무료 등록 한도(20건)를 초과했습니다." } }
```

### 목록 API 공통 쿼리 파라미터
| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `page` | 1 | |
| `size` | 20 | 최대 100 |
| `from` / `to` | 모듈별 상이 | 날짜 필터 (`YYYY-MM-DD`) |

기본 정렬은 모든 목록에서 최신순(날짜 내림차순)이다.

### 공통 에러 코드
| 코드 | HTTP | 설명 |
|------|------|------|
| `VALIDATION_ERROR` | 400 | 입력값 검증 실패 |
| `UNAUTHORIZED` | 401 | 토큰 없음/만료 |
| `FORBIDDEN` | 403 | 역할 권한 없음, 본인 소유 아님 |
| `NOT_FOUND` | 404 | 리소스 없음 |
| `CONFLICT` | 409 | 중복, 상태 충돌 |
| `PLAN_LIMIT_EXCEEDED` | 403 | 플랜 한도 초과 (기능별 상세 코드는 각 절 참고) |

모듈별 세부 에러는 `03-feature-spec.md`의 "예외 처리" 표와 1:1 대응한다 — 여기서는 코드명만 부여하고 문구를 반복하지 않는다.

---

<a id="auth"></a>

## Auth API — `/api/auth`

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| POST | `/api/auth/register` | 없음 | 업체 + 관리자 계정 동시 생성 (X-1) |
| POST | `/api/auth/verify-email` | 없음 | 이메일 인증 토큰 확인 |
| POST | `/api/auth/login` | 없음 | 로그인 |
| POST | `/api/auth/refresh` | 없음(Refresh Token) | Access Token 재발급 |
| POST | `/api/auth/logout` | 로그인 | Refresh Token 블랙리스트 등록 |
| POST | `/api/auth/password-reset/request` | 없음 | 재설정 메일 발송 (X-3) |
| POST | `/api/auth/password-reset/confirm` | 없음(Reset Token) | 새 비밀번호 설정, 기존 세션 전체 무효화 |
| GET | `/api/auth/me` | 로그인 | 내 정보 조회 — 인쇄용 영수증 헤더에 쓰는 사업자정보(사업자번호·대표자명·전화·주소) 포함 |
| GET | `/api/auth/company` | 관리자 | 사업자 프로필 조회 — 계좌정보 포함(`/me`에는 없음) |
| PATCH | `/api/auth/company` | 관리자 | 사업자 프로필 수정 — 전화·팩스·주소·계좌정보만(상호·사업자번호·대표자명은 불변) |
| GET | `/api/auth/employees` | 관리자 | 직원 목록 (X-2) |
| PATCH | `/api/auth/employees/{id}` | 관리자 | 역할·활성 상태 변경 |
| DELETE | `/api/auth/employees/{id}` | 관리자 | 직원 삭제 (soft) |
| POST | `/api/auth/invitations` | 관리자 | 직원 초대 (X-2) |
| GET | `/api/auth/invitations` | 관리자 | 초대 목록·상태 |
| POST | `/api/auth/invitations/{token}/accept` | 없음 | 초대 수락 + 비밀번호 설정 |

#### `POST /api/auth/register`
```jsonc
// request
{
  "companyName": "삼진주방설비",
  "businessRegistrationNumber": "1234567890",
  "representativeName": "김대표",
  "email": "admin@samjin.co.kr",
  "password": "samjin1234",
  "phone": "010-1234-5678"
}
```
```jsonc
// 201 response
{ "data": { "companyId": "8f0e...", "employeeId": "c1a2...", "email": "admin@samjin.co.kr", "plan": "FREE" } }
```
에러: `VALIDATION_ERROR`, `CONFLICT`(`BUSINESS_NUMBER_DUPLICATE`), `CONFLICT`(`EMAIL_DUPLICATE`)

요청에는 선택 필드로 `address`·`fax`·`bankName`·`bankAccountHolder`·`bankAccountNumber`도 받는다(전부 생략 가능) — 인쇄용 영수증·사업자 프로필에 쓰인다. 가입 후에도 `PATCH /api/auth/company`로 계속 수정할 수 있다.

#### `POST /api/auth/login`
```jsonc
// request
{ "email": "admin@samjin.co.kr", "password": "samjin1234" }
```
```jsonc
// 200 response — Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Strict 헤더 동반, 바디에는 없음
{
  "data": {
    "accessToken": "eyJ...",
    "employee": { "id": "c1a2...", "name": "김대표", "role": "ADMIN", "companyId": "8f0e...", "plan": "FREE" }
  }
}
```
에러: `UNAUTHORIZED`(`INVALID_CREDENTIALS`), `FORBIDDEN`(`ACCOUNT_LOCKED`, 5회 실패 시 10분)

#### `PATCH /api/auth/company`
`phone`·`fax`·`address`·`bankName`·`bankAccountHolder`·`bankAccountNumber` 전부 선택 — 요청에 없는(= `null`) 필드는 변경하지 않는다(`EmployeeDto.UpdateRequest`와 동일한 부분 수정 규칙). `name`·`businessRegistrationNumber`·`representativeName`은 이 엔드포인트로 바꿀 수 없다.
```jsonc
// request — 주소만 바꾸는 예시
{ "address": "서울시 강남구 테헤란로 123" }
```
```jsonc
// 200 response — bankAccountNumber는 복호화된 평문으로 내려간다(계좌정보는 관리자만 조회 가능)
{
  "data": {
    "id": "8f0e...", "name": "삼진주방설비", "businessRegistrationNumber": "1234567890",
    "representativeName": "김대표", "phone": "010-1234-5678", "fax": null,
    "address": "서울시 강남구 테헤란로 123", "bankName": "국민은행",
    "bankAccountHolder": "김대표", "bankAccountNumber": "123-456-789012"
  }
}
```

---

<a id="clients"></a>

## Finance API — 거래처 `/api/clients`

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| GET | `/api/clients?query=&status=` | 관리자 | 목록·검색 (B-1) |
| POST | `/api/clients` | 관리자 | 등록 |
| GET | `/api/clients/{id}` | 관리자 | 상세 |
| PATCH | `/api/clients/{id}` | 관리자 | 수정 |
| DELETE | `/api/clients/{id}` | 관리자 | 삭제 — 세금계산서 이력 있으면 `409 CONFLICT`(`CLIENT_HAS_INVOICES`), 대신 `status: "INACTIVE"`로 `PATCH` 권장 |

---

<a id="receipts"></a>

## Finance API — 영수증 `/api/receipts`

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| POST | `/api/receipts/presigned-url` | 로그인 | S3 업로드용 Presigned URL 발급 (A-1) |
| POST | `/api/receipts` | 로그인 | 등록 |
| GET | `/api/receipts?from=&to=&clientId=&category=&createdBy=` | 로그인 | 목록·필터 (A-2). 직원은 본인 등록건만 |
| GET | `/api/receipts/{id}` | 로그인 | 상세 |
| PATCH | `/api/receipts/{id}` | 로그인 | 수정 — 직원은 본인 등록 후 24시간 이내만 (A-3) |
| DELETE | `/api/receipts/{id}` | 로그인 | 삭제 (soft) |
| POST | `/api/receipts/export` | 관리자(Pro+) | PDF/엑셀 비동기 내보내기 요청 (A-4) → `{ jobId }` |
| GET | `/api/exports/{jobId}` | 로그인 | 내보내기 Job 상태 조회 (영수증·세금계산서·도면 공용) |

#### `POST /api/receipts`

`amount`는 요청 바디에 없다 — `items[]`(품목명·수량·단가)의 합계를 서버가 계산해 저장한다(품목 1개 이상 필수).

```jsonc
// request
{
  "receiptDate": "2026-09-10", "vendorName": "한성식자재",
  "clientId": null, "category": "MATERIAL", "memo": "9월 재료비",
  "items": [
    { "name": "돼지고기 앞다리살", "quantity": 10, "unitPrice": 8000 },
    { "name": "배추", "quantity": 5, "unitPrice": 1000 }
  ],
  "imageS3Key": "receipts/8f0e.../2026/09/original_img01.jpg"
}
```
```jsonc
// 201 response
{
  "data": {
    "id": "r-001", "receiptDate": "2026-09-10", "amount": 85000, "category": "MATERIAL",
    "items": [
      { "id": "ri-001", "name": "돼지고기 앞다리살", "quantity": 10, "unitPrice": 8000, "amount": 80000 },
      { "id": "ri-002", "name": "배추", "quantity": 5, "unitPrice": 1000, "amount": 5000 }
    ],
    "createdAt": "2026-09-10T09:12:00Z"
  }
}
```
에러: `VALIDATION_ERROR`(미래 날짜, 품목 0개, `quantity <= 0`, `unitPrice <= 0`), `PLAN_LIMIT_EXCEEDED`(`RECEIPT_FREE_LIMIT`, Free 월 20건)

#### `GET /api/receipts` 응답 — 필터 합계 포함
```jsonc
{
  "data": [ { "id": "r-001", "receiptDate": "2026-09-10", "amount": 85000, "vendorName": "한성식자재", "category": "MATERIAL", "hasImage": true } ],
  "meta": { "page": 1, "size": 20, "totalElements": 42, "totalPages": 3, "totalAmount": 3120000 }
}
```

---

<a id="tax-invoices"></a>

## Finance API — 세금계산서 `/api/tax-invoices`

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| POST | `/api/tax-invoices` | 관리자(Pro+) | 작성 — `status: "DRAFT"` 또는 `"COMPLETED"` (B-2) |
| GET | `/api/tax-invoices?from=&to=&clientId=&status=` | 관리자 | 이력 조회 (B-3) |
| GET | `/api/tax-invoices/{id}` | 관리자 | 상세 |
| GET | `/api/tax-invoices/{id}/excel` | 관리자 | 홈택스 bulk 업로드용 엑셀 다운로드 → 성공 시 `status`를 `EXCEL_DOWNLOADED`로 갱신 |
| POST | `/api/tax-invoices/{id}/revise` | 관리자 | 수정 발행 — 새 건 생성, 원본은 `REVISED` (B-4) |
| POST | `/api/tax-invoices/{id}/cancel` | 관리자 | 취소 — `{ reason }` 필수 |

#### `POST /api/tax-invoices`
```jsonc
// request
{
  "clientId": "cl-014", "issueDate": "2026-09-11", "status": "COMPLETED",
  "items": [
    { "name": "업소용 냉장고 900L", "spec": "2도어", "quantity": 1, "unitPrice": 1800000 },
    { "name": "설치비", "spec": null, "quantity": 1, "unitPrice": 150000 }
  ],
  "note": "9월 정기 납품"
}
```
```jsonc
// 201 response — 공급가액·세액은 서버가 계산해 저장
{
  "data": {
    "id": "ti-201", "status": "COMPLETED", "issueDate": "2026-09-11",
    "items": [
      { "name": "업소용 냉장고 900L", "quantity": 1, "unitPrice": 1800000, "supplyAmount": 1800000, "taxAmount": 180000 },
      { "name": "설치비", "quantity": 1, "unitPrice": 150000, "supplyAmount": 150000, "taxAmount": 15000 }
    ],
    "supplyAmount": 1950000, "taxAmount": 195000, "totalAmount": 2145000
  }
}
```
에러: `VALIDATION_ERROR`(품목 0개 또는 17개 이상, 수량·단가 음수, 거래처 미선택)

---

<a id="drawings"></a>

## Drawing API — `/api/drawings`

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| POST | `/api/drawings` | 관리자(Pro+) | 도면 생성 (C-1) |
| GET | `/api/drawings` | 관리자 | 목록 |
| GET | `/api/drawings/{id}` | 로그인 | 상세 + 최신 버전 `content` 포함 (직원은 조회만, X-2) |
| PATCH | `/api/drawings/{id}` | 관리자 | 메타 정보(이름·고객명·현장주소) 수정 |
| DELETE | `/api/drawings/{id}` | 관리자 | 삭제 |
| PUT | `/api/drawings/{id}/draft` | 관리자 | 자동 저장 — `draft_content` 덮어쓰기 (30초 디바운스는 프론트 책임) |
| POST | `/api/drawings/{id}/versions` | 관리자 | 수동 저장 → 새 버전 생성 (C-2/C-3) |
| GET | `/api/drawings/{id}/versions` | 관리자 | 버전 목록 |
| GET | `/api/drawings/{id}/versions/{versionNo}` | 관리자 | 특정 버전 미리보기 |
| POST | `/api/drawings/{id}/versions/{versionNo}/restore` | 관리자 | 복원 — 현재 상태를 새 버전으로 저장 후 덮어씀 |
| POST | `/api/drawings/{id}/export` | 관리자 | PDF 내보내기 요청 (C-4) → `{ jobId }` |
| POST | `/api/drawings/{id}/share-links` | 관리자 | 공유 링크 생성 — `{ expiresIn: "7d" \| "30d" \| "never" }` |
| DELETE | `/api/drawings/{id}/share-links/{linkId}` | 관리자 | 링크 비활성화 |
| GET | `/api/drawings/shared/{token}` | 없음 | 고객용 읽기 전용 뷰어 (C-4) |

#### `POST /api/drawings/{id}/versions`
`content`는 `05-database-schema.md`에 정의한 Fabric.js 확장 스키마를 그대로 받는다.
```jsonc
// request
{
  "content": {
    "version": "5.3.0",
    "objects": [
      {
        "type": "rect", "left": 80, "top": 40, "width": 60, "height": 60, "angle": 0,
        "symbolType": "fridge_commercial", "productId": null, "label": "냉장고 #1", "locked": false,
        "heightMm": 1800, "sillHeightMm": null, "headHeightMm": null
      }
    ]
  }
}
```
```jsonc
// 201 response
{ "data": { "id": "dv-089", "drawingId": "dw-014", "versionNo": 4, "createdAt": "2026-09-11T10:02:00Z" } }
```
에러: `PLAN_LIMIT_EXCEEDED`(`DRAWING_PRO_LIMIT`, Pro 5개), 버전 20개 초과 시 가장 오래된 버전 자동 삭제(에러 아님)

#### `GET /api/drawings/shared/{token}`
```jsonc
// 200 response — 편집 불가, 읽기 전용
{ "data": { "name": "OO식당 주방 배치도", "customerName": "OO식당", "issuedAt": "2026-09-11", "content": { "...": "..." } } }
```
에러: `410 GONE`(`SHARE_LINK_EXPIRED`), `410 GONE`(`SHARE_LINK_REVOKED`)

---

<a id="products"></a>

## Product API — `/api/products`

제품(품목)과 SKU(사이즈 변형)가 분리되어 있으므로(`05-database-schema.md`), 등록·수정은 품목 기준, 스캔·단가는 SKU 기준으로 나뉜다.

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| POST | `/api/products` | 관리자 | 제품(품목) 등록 (D-1) |
| GET | `/api/products?query=&categoryLarge=&size=` | 로그인 | 텍스트 검색·필터 (D-2) |
| GET | `/api/products/{id}` | 로그인 | 상세 — 하위 SKU 목록 포함 |
| PATCH | `/api/products/{id}` | 관리자 | 수정 |
| DELETE | `/api/products/{id}` | 관리자 | 삭제 (soft) — 이후 스캔 시 `PRODUCT_DISCONTINUED` 안내 (D-4) |
| POST | `/api/products/{id}/variants` | 관리자 | SKU 추가 |
| PATCH | `/api/products/variants/{variantId}` | 관리자 | SKU 수정 — 단가 변경 시 `product_price_history` 자동 기록 |
| DELETE | `/api/products/variants/{variantId}` | 관리자 | SKU 삭제 |
| GET | `/api/products/variants/{variantId}/price-history` | 관리자 | 단가 변경 이력 (D-4) |
| GET | `/api/products/variants/scan?code={barcodeOrQr}` | 로그인 | 바코드·QR 스캔 조회 (D-3) |

#### `GET /api/products/variants/scan?code=8801234567890`
```jsonc
// 200 response
{
  "data": {
    "productId": "p-330", "variantId": "v-330-2",
    "name": "업소용 스텐 냄비", "sizeLabel": "L", "price": 42000,
    "material": "스테인리스", "usageDescription": "대량 조리용",
    "photos": ["https://cdn.../thumb_01.jpg"], "notes": "직화 전용, 인덕션 사용 불가"
  }
}
```
에러: `NOT_FOUND`(`PRODUCT_NOT_REGISTERED`, 매칭 실패 — 관리자에게는 등록 바로가기 링크 포함), `410 GONE`(`PRODUCT_DISCONTINUED`)

---

<a id="dashboard"></a>

## Dashboard API — `/api/dashboard`

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| GET | `/api/dashboard` | 로그인 | 위젯 통합 조회 (X-4) — Analytics 서비스가 Redis 캐시에서 응답 |

```jsonc
// 200 response
{
  "data": {
    "taxInvoices": { "locked": false, "count": 12, "supplyAmountSum": 24500000 },
    "receipts": { "count": 38, "amountSum": 3120000 },
    "recentDrawings": [ { "id": "dw-014", "name": "OO식당 주방 배치도", "updatedAt": "2026-09-11T10:02:00Z" } ],
    "topProducts": [ { "variantId": "v-330-2", "name": "업소용 스텐 냄비 (L)", "viewCount7d": 26 } ],
    "usage": { "plan": "FREE", "receiptUsedThisMonth": 18, "receiptLimit": 20 }
  }
}
```
`taxInvoices.locked: true`면 Free 플랜이라 값 대신 잠금 표시만 내려준다 (X-4).

---

## 다음 단계

1. `07-ui-wireframe.md` — 위 엔드포인트를 소비하는 화면·네비게이션 흐름 정의
2. `08-deployment.md` — 배포 전략·인프라·CI/CD
3. `09-security.md` — 인증·인가·암호화·접근 제어 상세
