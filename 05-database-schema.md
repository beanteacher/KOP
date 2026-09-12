# 05 — DB 스키마

> **문서 상태**: 초안 (Draft)
> **작성일**: 2026-09-11
> **버전**: v0.1
> **연관 문서**: `01-prd.md` (PRD v0.3) · `03-feature-spec.md` (기능 명세 v0.1) · `04-system-architecture.md` (아키텍처 v0.3)

---

## 목차

- [공통 규칙](#common)
- [auth 스키마](#auth)
- [finance 스키마](#finance)
- [drawings 스키마](#drawings)
- [products 스키마](#products)
- [인덱스 전략](#index)
- [미해결 사항](#open)

---

<a id="common"></a>

## 공통 규칙

`04-system-architecture.md`의 "단일 PostgreSQL 인스턴스 + 서비스별 스키마 분리" 방침을 그대로 따른다. 아래 규칙은 모든 테이블에 공통 적용되며, 표에는 반복 표기하지 않는다.

| 규칙 | 내용 |
|------|------|
| PK | `id UUID PRIMARY KEY DEFAULT gen_random_uuid()` |
| 생성·수정 시각 | `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()` (수정 시 트리거로 갱신) |
| 소프트 삭제 | 비즈니스 룰상 이력 보존이 필요한 테이블만 `deleted_at TIMESTAMPTZ NULL` 보유 — 표에 명시된 테이블만 해당 |
| 멀티테넌시 | 테넌트 데이터를 갖는 모든 테이블에 `company_id UUID NOT NULL` 포함. 모든 조회는 애플리케이션 레벨에서 `WHERE company_id = :company_id` 강제 |
| 금액 | `BIGINT`, 원 단위 정수 (소수 없음) |
| 스키마 간 참조 | **같은 스키마 내부**는 일반 FK 제약을 건다. **스키마를 넘는 참조**(예: `finance.receipts.created_by` → `auth.employees.id`, `drawings` 콘텐츠 안의 `productId` → `products.products.id`)는 FK를 걸지 않는다 — MSA 서비스 간 DB 직접 조인을 금지하는 `04-system-architecture.md` 격리 원칙 때문. UUID 값만 저장하고 정합성은 API 호출·Kafka 이벤트(`employee.registered` 등)로 보장한다 |
| `export_jobs` / `outbox` | `finance`·`drawings` 스키마에 동일 구조로 존재. 전체 컬럼 정의는 `04-system-architecture.md`의 [Outbox Pattern](./04-system-architecture.md#outbox), [비동기 내보내기](./04-system-architecture.md#async-export) 절에 이미 있으므로 반복하지 않는다 |

---

<a id="auth"></a>

## auth 스키마

### companies — 업체

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| name | VARCHAR(100) | NOT NULL | 업체명 |
| business_registration_number | VARCHAR(10) | NOT NULL, UNIQUE | 사업자번호, 하이픈 없이 10자리 (X-1: 중복 가입 방지) |
| representative_name | VARCHAR(50) | NOT NULL | 대표자명 |
| phone | VARCHAR(20) | NULL | 연락처 |
| plan | VARCHAR(20) | NOT NULL, DEFAULT `'FREE'` | `FREE` / `PRO` / `BUSINESS` (`01-prd.md` §9) |
| plan_updated_at | TIMESTAMPTZ | NULL | 마지막 플랜 변경 시각 |

로그인 계정(이메일·비밀번호)은 회사가 아니라 `employees`에 귀속된다 — 가입 시 회사 1건 + 관리자 직원 1건이 함께 생성된다 (X-1).

---

### employees — 직원 (관리자 / 직원)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL, FK → `companies.id` | 소속 업체 |
| email | VARCHAR(255) | NOT NULL, UNIQUE | 로그인 ID |
| password_hash | VARCHAR(255) | NOT NULL | bcrypt 해시 |
| name | VARCHAR(50) | NOT NULL | |
| role | VARCHAR(20) | NOT NULL | `ADMIN`(관리자) / `STAFF`(직원) — 권한 매트릭스는 `03-feature-spec.md` X-2 |
| status | VARCHAR(20) | NOT NULL, DEFAULT `'ACTIVE'` | `ACTIVE` / `INACTIVE` (X-2 직원 비활성화) |
| email_verified_at | TIMESTAMPTZ | NULL | 이메일 인증 시각 (X-1: 인증 전엔 저장 기능 제한) |
| failed_login_count | SMALLINT | NOT NULL, DEFAULT 0 | 로그인 실패 카운트 |
| locked_until | TIMESTAMPTZ | NULL | 5회 실패 시 10분 잠금 (X-3) |
| deleted_at | TIMESTAMPTZ | NULL | 소프트 삭제 — 영수증·세금계산서·도면의 `created_by` 참조 무결성 유지 목적 |

---

### employee_invitations — 직원 초대

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL, FK → `companies.id` | |
| email | VARCHAR(255) | NOT NULL | 초대 대상 이메일 |
| role | VARCHAR(20) | NOT NULL | 초대 시 지정한 역할 |
| token | VARCHAR(255) | NOT NULL, UNIQUE | 초대 링크 토큰 |
| invited_by | UUID | NOT NULL, FK → `employees.id` | 초대한 관리자 |
| expires_at | TIMESTAMPTZ | NOT NULL | 발급 + 72시간 (X-2) |
| accepted_at | TIMESTAMPTZ | NULL | 수락 시각, NULL이면 미수락 |

---

<a id="finance"></a>

## finance 스키마

### clients — 거래처 (세금계산서 공급받는자 주소록)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL | |
| business_registration_number | VARCHAR(10) | NOT NULL | 사업자번호 10자리 |
| name | VARCHAR(100) | NOT NULL | 상호 |
| representative_name | VARCHAR(50) | NOT NULL | |
| business_type | VARCHAR(100) | NOT NULL | 업태 |
| business_item | VARCHAR(100) | NOT NULL | 종목 |
| address | VARCHAR(255) | NOT NULL | 사업장 주소 |
| email | VARCHAR(255) | NULL | 발행 알림용 |
| contact_name | VARCHAR(50) | NULL | 담당자명 |
| contact_phone | VARCHAR(20) | NULL | |
| status | VARCHAR(20) | NOT NULL, DEFAULT `'ACTIVE'` | `ACTIVE` / `INACTIVE` — 세금계산서 이력이 있으면 삭제 대신 비활성화 (B-1) |

`UNIQUE(company_id, business_registration_number)` — 동일 업체 내 중복 등록 방지.

---

### receipts — 영수증

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL | |
| created_by | UUID | NOT NULL | 등록한 직원 (`auth.employees.id`, FK 없음) |
| receipt_date | DATE | NOT NULL | 발행일 — 미래 날짜 불가 (애플리케이션 검증) |
| amount | BIGINT | NOT NULL, CHECK (amount > 0) | 원 단위 |
| vendor_name | VARCHAR(100) | NOT NULL | 거래처명 — 자유 입력 (항상 저장, 이력 보존용) |
| client_id | UUID | NULL, FK → `clients.id` | 거래처 주소록에서 선택한 경우만 연결 |
| category | VARCHAR(20) | NOT NULL | `MATERIAL`/`TRANSPORT`/`LABOR`/`EQUIPMENT`/`OTHER` (재료비/운반비/인건비/장비비/기타) |
| memo | VARCHAR(200) | NULL | |
| deleted_at | TIMESTAMPTZ | NULL | 소프트 삭제 (A-3) |

---

### receipt_images — 영수증 첨부 이미지

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| receipt_id | UUID | NOT NULL, FK → `receipts.id` | |
| s3_key | VARCHAR(500) | NOT NULL | 원본 경로 (`04-system-architecture.md` 저장 경로 규칙) |
| thumbnail_s3_key | VARCHAR(500) | NULL | Lambda가 생성한 썸네일 |

영수증은 이미지가 선택 사항이라 0~N 행을 가질 수 있다 (A-1: "이미지 없이 등록 가능").

---

### tax_invoices — 세금계산서

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL | 공급자(자사) |
| created_by | UUID | NOT NULL | |
| client_id | UUID | NOT NULL, FK → `clients.id` | 공급받는자 |
| issue_date | DATE | NOT NULL | 작성일자 |
| status | VARCHAR(20) | NOT NULL, DEFAULT `'DRAFT'` | `DRAFT`(임시저장) / `COMPLETED`(작성완료) / `EXCEL_DOWNLOADED`(엑셀다운로드완료) / `REVISED`(수정발행됨) / `CANCELED`(취소) |
| supply_amount | BIGINT | NOT NULL, DEFAULT 0 | 공급가액 합계 — 품목 합산 후 애플리케이션에서 저장 |
| tax_amount | BIGINT | NOT NULL, DEFAULT 0 | 세액 합계 (공급가액 × 10%) |
| total_amount | BIGINT | NOT NULL, DEFAULT 0 | |
| note | VARCHAR(200) | NULL | 비고 |
| revised_from_id | UUID | NULL, FK → `tax_invoices.id` | 수정 발행 시 원본 참조 (B-4) |
| cancel_reason | VARCHAR(100) | NULL | |
| canceled_at | TIMESTAMPTZ | NULL | |
| excel_downloaded_at | TIMESTAMPTZ | NULL | |

세액·합계는 DB 계산 컬럼이 아니라 애플리케이션에서 계산 후 저장한다 (감사 추적을 위해 계산 시점 값을 고정).

---

### tax_invoice_items — 세금계산서 품목

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| tax_invoice_id | UUID | NOT NULL, FK → `tax_invoices.id` | |
| name | VARCHAR(100) | NOT NULL | 품목명 |
| spec | VARCHAR(100) | NULL | 규격 |
| quantity | INT | NOT NULL, CHECK (quantity > 0) | |
| unit_price | BIGINT | NOT NULL, CHECK (unit_price >= 0) | |
| supply_amount | BIGINT | NOT NULL | quantity × unit_price |
| tax_amount | BIGINT | NOT NULL | supply_amount × 10% |
| sort_order | SMALLINT | NOT NULL, DEFAULT 0 | 표시 순서 |

세금계산서 1건당 최대 16개(홈택스 bulk 양식 제한) — DB 제약이 아니라 애플리케이션에서 검증.

---

<a id="drawings"></a>

## drawings 스키마

### drawings — 도면

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL | |
| created_by | UUID | NOT NULL | |
| name | VARCHAR(100) | NOT NULL | 도면명 |
| customer_name | VARCHAR(100) | NULL | 고객명 |
| site_address | VARCHAR(255) | NULL | 현장 주소 |
| canvas_width_cm | NUMERIC(8,1) | NOT NULL, CHECK (>= 100) | 실측 기준 캔버스 폭 (C-1) |
| canvas_height_cm | NUMERIC(8,1) | NOT NULL, CHECK (>= 100) | |
| grid_spacing_cm | NUMERIC(6,1) | NOT NULL, DEFAULT 50 | |
| current_version_id | UUID | NULL, FK → `drawing_versions.id` | 최신 수동 저장 버전 |
| draft_content | JSONB | NULL | 자동 저장 임시본 (버전 번호 미부여, C-2) |
| draft_saved_at | TIMESTAMPTZ | NULL | 마지막 자동 저장 시각 |
| deleted_at | TIMESTAMPTZ | NULL | |

`canvas_width_cm` / `canvas_height_cm`이 실측 mm↔px 환산 기준이 된다. 저장된 `content`(px, fabric 좌표계)를 실제 치수로 바꿀 때 `px_per_cm = 캔버스_렌더링_폭_px / canvas_width_cm`로 계산하므로, 별도의 "축척" 컬럼을 두지 않는다.

---

### drawing_versions — 도면 버전 (수동 저장분만)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| drawing_id | UUID | NOT NULL, FK → `drawings.id` | |
| version_no | INT | NOT NULL | 수동 저장마다 1씩 증가 |
| content | JSONB | NOT NULL | Fabric.js `canvas.toJSON([...])` 확장 결과 — 아래 참고 |
| created_by | UUID | NOT NULL | |

`UNIQUE(drawing_id, version_no)`. 도면당 최대 20개 보관, 초과 시 `version_no`가 가장 작은 행부터 실제 삭제 (C-3, 배치 잡).

#### `content` JSON 구조

편집기는 Fabric.js를 그대로 쓰되, `canvas.toJSON()`에 넘기는 커스텀 프로퍼티 목록을 도메인 요구에 맞게 확장한다(`03-feature-spec.md` C-2). Fabric 표준 필드(`left`/`top`/`width`/`height`/`scaleX`/`scaleY`/`angle` 등)는 그대로 두고, 아래 5개만 우리가 추가한다.

```jsonc
{
  "version": "5.3.0",
  "objects": [
    {
      "type": "rect",
      "left": 80, "top": 40, "width": 60, "height": 60, "angle": 0,
      "scaleX": 1, "scaleY": 1,

      "symbolType": "fridge_commercial",  // 기본 심볼 라이브러리 값 (03-feature-spec C-2 표) — 없으면 productId로 대체
      "productId": "b7c2e6f0-...",        // 특정 SKU를 배치한 경우만 존재 (products.products.id, FK 없음)
      "label": "냉장고 #1",
      "locked": false,

      "heightMm": 1800,        // 벽·집기 높이 — 2D 렌더링엔 안 쓰지만 3D 압출용으로 항상 저장
      "sillHeightMm": null,    // 문/창 전용: 바닥에서 하단까지
      "headHeightMm": null     // 문/창 전용: 바닥에서 상단까지
    }
  ]
}
```

- `symbolType`과 `productId`는 상호 배타적이지 않다 — 카테고리 기본 심볼로 시작했다가 나중에 특정 SKU로 바꿔도 `symbolType`은 남겨둬 필터링·통계에 쓴다.
- `heightMm`/`sillHeightMm`/`headHeightMm`은 지금(2D-only MVP)은 UI에 노출하지 않아도 되지만, 컬럼이 아니라 JSON 안에 있으므로 **마이그레이션 없이** 나중에 3D 렌더러가 그대로 읽을 수 있다.

---

### drawing_share_links — 고객 공유 링크

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| drawing_id | UUID | NOT NULL, FK → `drawings.id` | |
| token | VARCHAR(64) | NOT NULL, UNIQUE | UUID 기반 (C-4) |
| expires_at | TIMESTAMPTZ | NULL | NULL = 만료 없음 |
| revoked_at | TIMESTAMPTZ | NULL | 비활성화 시각 |
| created_by | UUID | NOT NULL | |

---

<a id="products"></a>

## products 스키마

제품(모듈 D)은 "종(품목)"과 "사이즈별 SKU"가 분리된다 — PRD가 "제품 1,000여 종, 사이즈 변형 포함 SKU 3,000~5,000개"로 규모를 추정하고 있어, 바코드·QR·단가는 사이즈(SKU) 단위로 각각 다를 수 있기 때문이다.

### products — 제품(품목)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL | |
| product_code | VARCHAR(20) | NOT NULL | 자동 생성, 예 `PRD-000001` (D-1) |
| name | VARCHAR(200) | NOT NULL | |
| category_large | VARCHAR(50) | NOT NULL | 대분류 — 자유 입력 + 기존 값 자동완성(별도 테이블 없이 DISTINCT 조회) |
| category_medium | VARCHAR(50) | NOT NULL | 중분류 |
| category_small | VARCHAR(50) | NULL | 소분류 |
| material | VARCHAR(100) | NULL | 재질 |
| usage_description | VARCHAR(500) | NULL | 용도 |
| notes | VARCHAR(500) | NULL | 특이사항·주의사항 |
| icon_svg_url | VARCHAR(500) | NULL | 도면 배치용 탑다운 심볼. 미지정 시 카테고리 기본 심볼 사용 |
| deleted_at | TIMESTAMPTZ | NULL | 소프트 삭제 — 삭제 후 스캔 시 "단종/삭제된 제품" 안내 (D-4) |

`UNIQUE(company_id, product_code)`.

---

### product_variants — SKU (사이즈별 판매 단위)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL | 바코드 유일성 조회를 위해 비정규화 |
| product_id | UUID | NOT NULL, FK → `products.id` | |
| sku_code | VARCHAR(30) | NOT NULL | 예 `PRD-000001-M` |
| size_label | VARCHAR(20) | NULL | `S`/`M`/`L`/`XL` 또는 직접 입력(cm). 사이즈 구분 없는 제품은 NULL |
| barcode | VARCHAR(20) | NULL | 제조사 바코드 (EAN-13 등) |
| qr_code | VARCHAR(40) | NULL | 자체 생성 QR 페이로드, 바코드 없을 때만 채움 (`PRD-{product_code}-{size}`) |
| price | BIGINT | NULL | 원 단위 |
| deleted_at | TIMESTAMPTZ | NULL | |

`UNIQUE(company_id, sku_code)`, `UNIQUE(company_id, barcode) WHERE barcode IS NOT NULL AND deleted_at IS NULL` (D-1: 바코드 중복 불가).

---

### product_images — 제품 사진 (품목 단위, 최대 5장)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| product_id | UUID | NOT NULL, FK → `products.id` | |
| s3_key | VARCHAR(500) | NOT NULL | |
| sort_order | SMALLINT | NOT NULL, DEFAULT 0 | |

목록 식별용 정면 사진이며, 도면에 쓰는 `products.icon_svg_url`과는 별개 자산이다.

---

### product_price_history — 단가 변경 이력 (SKU 단위)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| variant_id | UUID | NOT NULL, FK → `product_variants.id` | |
| old_price | BIGINT | NULL | |
| new_price | BIGINT | NOT NULL | |
| changed_by | UUID | NOT NULL | |
| changed_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

관리자만 조회 가능 (D-4) — DB 제약이 아니라 API 권한 체크.

---

### product_scan_logs — 조회·스캔 이력

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| company_id | UUID | NOT NULL | |
| variant_id | UUID | NOT NULL, FK → `product_variants.id` | |
| viewed_by | UUID | NOT NULL | |
| channel | VARCHAR(10) | NOT NULL | `SEARCH`(텍스트 검색) / `SCAN`(바코드·QR) |
| viewed_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

조회수는 이 테이블에만 쌓고 `products`/`product_variants`에 카운터 컬럼을 따로 두지 않는다 — 대시보드 "자주 조회된 제품"(최근 7일 상위 5개)과 `product.viewed` Kafka 이벤트는 Analytics 서비스가 이 로그를 집계해 Redis에 캐싱하는 구조이기 때문 (`04-system-architecture.md` Analytics Service).

---

<a id="index"></a>

## 인덱스 전략

자주 쓰는 조회 패턴 기준으로 우선 추가할 인덱스만 정리한다. 나머지는 실측 쿼리 로그를 보고 추가한다.

| 테이블 | 인덱스 | 용도 |
|--------|--------|------|
| `receipts` | `(company_id, receipt_date DESC)` | 기간 필터 + 최신순 목록 (A-2) |
| `receipts` | `(company_id, created_by)` | 직원 본인 등록건 조회 |
| `tax_invoices` | `(company_id, issue_date DESC)` | 발행 이력 조회 (B-3) |
| `clients` | `(company_id, business_registration_number)` | 사업자번호 검색·중복 체크 |
| `drawing_versions` | `(drawing_id, version_no DESC)` | 버전 목록·최신 버전 조회 |
| `products` | `(company_id, category_large, category_medium)` | 카테고리 필터 |
| `product_variants` | `(company_id, barcode)` | 바코드 스캔 즉시 조회 (D-3, 지연 최소화 필요) |
| `product_variants` | `(company_id, qr_code)` | QR 스캔 조회 |
| `product_scan_logs` | `(company_id, viewed_at DESC)` | 최근 7일 집계 |

`products.name` / `usage_description` 등 텍스트 검색(D-2)은 초기엔 `ILIKE` + 위 카테고리 인덱스로 충분하다고 보고, 조회 성능이 실제로 문제가 되면 `pg_trgm` GIN 인덱스 도입을 검토한다 (지금 만들지 않음 — 과설계 방지).

---

<a id="open"></a>

## 미해결 사항

`00-overview.md` 미결 사항 중 아래 2개는 이 스키마 설계에 직접 영향을 준다. 결정되기 전까지는 유연하게 열어둔 상태다.

- **공급사 엑셀 파일 수령 가능 여부** → 대량 임포트 시 `products`/`product_variants` 컬럼과 공급사 엑셀 컬럼을 어떻게 매핑할지는 실제 샘플 파일을 받은 뒤 별도 매핑 규격 문서로 정리한다.
- **바코드 없는 제품 QR 부착 범위** → `product_variants.qr_code`는 이미 컬럼으로 확보해뒀으므로, 범위가 정해지면 즉시 값 채우기만 하면 된다(스키마 변경 불필요).
