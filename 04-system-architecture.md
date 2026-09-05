# 04 — 시스템 아키텍처

> **문서 상태**: 초안 (Draft)
> **작성일**: 2026-09-05
> **버전**: v0.3
> **연관 문서**: `01-prd.md` (PRD v0.3)

---

## 목차

- [아키텍처 방향](#arch-direction)
- [전체 구조 개요](#overview)
- [마이크로서비스 목록](#services)
- [Kafka 이벤트 설계](#kafka)
- [Outbox Pattern](#outbox)
- [비동기 파일 처리 파이프라인](#async-file)
- [비동기 내보내기 (엑셀/PDF)](#async-export)
- [Circuit Breaker](#circuit-breaker)
- [API Gateway](#api-gateway)
- [프론트엔드](#frontend)
- [데이터베이스 전략](#database)
- [파일 저장소](#storage)
- [인프라 (AWS)](#infra)
- [외부 연동](#external)
- [CI/CD](#cicd)
- [기술 스택 요약](#stack-summary)

---

<a id="arch-direction"></a>

## 아키텍처 방향

### MSA 채택 이유

- 기능 확장 시 특정 서비스만 독립 배포·스케일 가능
- 트래픽 급증 구간(세금계산서 마감기간, 제품 조회 피크)에 해당 서비스만 스케일아웃
- 팀이 커질 때 서비스 단위로 소유권 분리 가능
- 초기부터 MSA로 설계해야 나중에 분리 비용 발생하지 않음

### Kafka 채택 이유

- 서비스 간 직접 HTTP 호출 의존성 제거 (느슨한 결합)
- 영수증 업로드·제품 조회 등 쓰기 집중 구간의 비동기 처리
- 이벤트 로그 보존으로 장애 시 재처리 가능
- 향후 분석·통계 파이프라인으로 확장 용이

### Spring Boot 선택 이유

- Spring Kafka (`spring-kafka`)로 Kafka Producer/Consumer 완전 지원
- Spring Cloud를 통한 MSA 생태계 성숙 (Service Discovery, Config Server 등)
- 개발자 숙련도 최대화 → 생산성·안정성 확보
- Spring Security로 인증·인가 체계 견고하게 구성 가능

### Outbox Pattern 채택 이유

- DB 저장과 Kafka 발행이 별도 작업이므로 중간 장애 시 이벤트 유실 가능
- Outbox 테이블을 동일 트랜잭션으로 함께 저장 → 정합성 보장
- Poller가 미발행 이벤트를 재처리하므로 at-least-once 보장

### S3 Presigned URL 채택 이유

- 파일을 백엔드 경유로 올리면 백엔드가 파일 트래픽·메모리 전부 부담
- Presigned URL로 클라이언트가 S3에 직접 업로드 → 백엔드 부하 제거
- 영수증·제품 이미지·도면처럼 파일이 많은 서비스에서 필수

### Circuit Breaker 채택 이유

- 외부 결제 API, 서비스 간 HTTP 호출 실패 시 장애가 전파되는 것을 차단
- Resilience4j로 실패율 임계치 초과 시 즉시 fallback 응답
- 커넥션 풀 고갈 방지 → 전체 서비스 안정성 확보

---

<a id="overview"></a>

## 전체 구조 개요

```
┌─────────────────────────────────────────────────────────────┐
│                       클라이언트                             │
│         브라우저 (React SPA)      모바일 웹                  │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS
               ┌─────────▼──────────┐
               │   API Gateway      │  ← Kong / AWS API GW
               │  인증·라우팅·제한   │
               └──┬──┬──┬──┬────────┘
                  │  │  │  │
     ┌────────────┘  │  │  └──────────────────┐
     │    ┌──────────┘  └───────────┐          │
     ▼    ▼                         ▼           ▼
  [Auth] [Finance]             [Drawing]  [Product]
  서비스  서비스                  서비스      서비스
         (Receipt+TaxInvoice)
     │    │                         │           │
     └────┴─────────────────────────┴───────────┘
          │  (각 서비스: Outbox 테이블 → Poller)
          │  (Finance·Drawing: 내부 Export 스케줄러)
          ▼
┌─────────────────────┐
│    Apache Kafka     │  ← AWS MSK
│  (이벤트 브로커)    │
└──────────┬──────────┘
           │
  ┌────────┴──────────────────────────┐
  ▼                                   ▼
[Notification 서비스]          [Analytics 서비스]
이메일·알림                    대시보드 집계 (Read Replica)
                               ← Redis 캐시
           │
           ▼
┌─────────────────────┐
│   AWS S3            │  ← Presigned URL 직접 업로드
│  (파일 저장소)      │
└──────────┬──────────┘
           │ S3 Event
           ▼
┌─────────────────────┐
│  SQS → Lambda       │  ← 이미지 리사이징 / 썸네일
│  (이미지 처리)      │
└─────────────────────┘

[공통 모듈 — common/]
  S3Uploader · ExcelBuilder · PdfBuilder · ExportJobManager
  ← Finance / Drawing 서비스가 의존성으로 사용
```

---

<a id="services"></a>

## 마이크로서비스 목록

### 1. Auth Service

| 항목 | 내용 |
|------|------|
| 역할 | 로그인, 회원가입, JWT 발급·검증, 비밀번호 재설정 |
| DB | PostgreSQL (auth 스키마): `companies`, `employees` |
| 포트 | 3001 |
| Kafka 발행 | `employee.invited`, `employee.registered` |
| Kafka 구독 | — |

---

### 2. Finance Service

| 항목 | 내용 |
|------|------|
| 역할 | 영수증 CRUD, 세금계산서 CRUD, 거래처 관리, S3 Presigned URL 발급, 엑셀·PDF 내보내기 (비동기 인라인 처리) |
| DB | PostgreSQL (finance 스키마): `receipts`, `receipt_images`, `clients`, `tax_invoices`, `tax_invoice_items`, `export_jobs`, `outbox` |
| 포트 | 3002 |
| Kafka 발행 | `receipt.created`, `receipt.deleted`, `tax_invoice.created`, `tax_invoice.cancelled`, `export.completed` |
| Kafka 구독 | — |
| 공통 모듈 | `common/ExcelBuilder`, `common/PdfBuilder`, `common/S3Uploader`, `common/ExportJobManager` |

---

### 3. Drawing Service

| 항목 | 내용 |
|------|------|
| 역할 | 도면 CRUD, 버전 관리, 공유 링크 생성, PDF 내보내기 (비동기 인라인 처리) |
| DB | PostgreSQL (drawings 스키마): `drawings`, `drawing_versions`, `drawing_share_links`, `export_jobs`, `outbox` |
| 포트 | 3003 |
| Kafka 발행 | `drawing.created`, `drawing.shared`, `export.completed` |
| Kafka 구독 | — |
| 공통 모듈 | `common/PdfBuilder`, `common/S3Uploader`, `common/ExportJobManager` |

---

### 4. Product Service

| 항목 | 내용 |
|------|------|
| 역할 | 제품 카탈로그 CRUD, 바코드·QR 조회, 이미지 관리 |
| DB | PostgreSQL (products 스키마): `products`, `product_images`, `product_price_history`, `product_scan_logs` |
| 포트 | 3004 |
| Kafka 발행 | `product.viewed`, `product.created` |
| Kafka 구독 | — |

---

### 5. Notification Service

| 항목 | 내용 |
|------|------|
| 역할 | 이메일 발송 (초대, 비밀번호 재설정, 발행 알림, 내보내기 완료 알림) |
| DB | 없음 (Stateless) |
| 포트 | 3005 (내부 전용) |
| Kafka 발행 | — |
| Kafka 구독 | `employee.invited`, `tax_invoice.created`, `export.completed` |

---

### 6. Analytics Service

| 항목 | 내용 |
|------|------|
| 역할 | 대시보드 집계 데이터 생성·캐싱 |
| DB | PostgreSQL (analytics 스키마, **Read Replica 사용**) + Redis (집계 캐시) |
| 포트 | 3006 (내부 전용) |
| Kafka 발행 | — |
| Kafka 구독 | `receipt.created`, `receipt.deleted`, `tax_invoice.created`, `product.viewed` |

---

<a id="kafka"></a>

## Kafka 이벤트 설계

### 토픽 목록

| 토픽 | 발행자 | 구독자 | 설명 |
|------|--------|--------|------|
| `receipt.created` | Finance | Analytics | 영수증 등록 → 월 집계 갱신 |
| `receipt.deleted` | Finance | Analytics | 영수증 삭제 → 집계 보정 |
| `tax_invoice.created` | Finance | Analytics, Notification | 세금계산서 발행 → 집계 + 알림 |
| `tax_invoice.cancelled` | Finance | Analytics | 취소 → 집계 보정 |
| `product.viewed` | Product | Analytics | 조회 → 자주 본 제품 집계 |
| `product.created` | Product | Analytics | 제품 등록 → 카탈로그 통계 |
| `employee.invited` | Auth | Notification | 직원 초대 → 이메일 발송 |
| `employee.registered` | Auth | Analytics | 직원 가입 → 업체 현황 집계 |
| `drawing.created` | Drawing | Analytics | 도면 생성 → 대시보드 최근 도면 |
| `drawing.shared` | Drawing | Notification | 공유 링크 생성 → (필요 시 알림) |
| `export.completed` | Finance / Drawing | Notification | 파일 생성 완료 → 다운로드 링크 이메일 |
| `image.uploaded` | S3 Event (SQS → Lambda) | — | 파일 업로드 완료 → 썸네일 처리 트리거 |

### 이벤트 메시지 구조 (공통)

```json
{
  "eventId": "uuid-v4",
  "eventType": "receipt.created",
  "companyId": "company-uuid",
  "timestamp": "2026-09-05T10:00:00Z",
  "payload": {
    // 이벤트별 데이터
  }
}
```

### 이벤트 흐름 예시 — 영수증 등록

```
직원 (모바일)
  │ POST /api/receipts
  ▼
API Gateway → Receipt Service
  │ DB 저장 완료
  │ Kafka produce: receipt.created
  ▼
Kafka (receipt.created 토픽)
  │
  ├── Analytics Service: 이번 달 영수증 합계 Redis 캐시 갱신
  └── (향후) Billing Service: Free 플랜 한도 카운트 갱신
```

### 이벤트 흐름 예시 — 세금계산서 발행

```
관리자
  │ POST /api/tax-invoices
  ▼
API Gateway → Tax Invoice Service
  │ DB 저장 + 엑셀 생성
  │ Kafka produce: tax_invoice.created
  ▼
Kafka (tax_invoice.created 토픽)
  │
  ├── Analytics Service: 이번 달 발행 건수·금액 캐시 갱신
  └── Notification Service: 발행 완료 이메일 (설정 시)
```

---

<a id="outbox"></a>

## Outbox Pattern

### 문제

DB 저장 성공 후 Kafka `produce()` 호출 전에 애플리케이션이 죽으면 이벤트가 유실된다.

```
// 위험한 패턴
db.save(receipt);        // 성공
kafka.produce(event);    // 실패 → DB에는 저장됐지만 이벤트 없음
```

### 해결: Outbox 테이블

각 서비스 스키마에 `outbox` 테이블을 추가하고, **같은 트랜잭션**으로 저장한다.

```sql
-- receipts 스키마 예시
CREATE TABLE outbox (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type  VARCHAR(100) NOT NULL,     -- 'receipt.created'
  payload     JSONB        NOT NULL,
  status      VARCHAR(20)  NOT NULL DEFAULT 'PENDING',  -- PENDING / SENT
  created_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

```java
// Receipt Service - 트랜잭션 1개로 묶음
@Transactional
public Receipt createReceipt(CreateReceiptRequest req) {
    Receipt receipt = receiptRepository.save(...);

    OutboxEvent event = OutboxEvent.of("receipt.created", receipt);
    outboxRepository.save(event);  // 같은 트랜잭션

    return receipt;
}
```

### Outbox Poller

별도 스케줄러(`@Scheduled` 또는 Debezium CDC)가 `status = PENDING` 레코드를 읽어 Kafka에 발행 후 `SENT`로 변경.

```
Outbox Poller (5초 주기)
  → SELECT * FROM outbox WHERE status = 'PENDING' LIMIT 100
  → kafka.produce(event)
  → UPDATE outbox SET status = 'SENT'
```

**결과**: DB 저장과 Kafka 발행이 항상 함께 성공하거나 함께 실패 → at-least-once 보장

---

<a id="async-file"></a>

## 비동기 파일 처리 파이프라인

### S3 Presigned URL — 직접 업로드

백엔드를 경유하지 않고 클라이언트가 S3에 직접 업로드한다.

```
[기존 방식]
클라이언트 → 백엔드(파일 수신) → S3  (백엔드 메모리·대역폭 소비)

[개선 방식]
1. 클라이언트 → 백엔드: "Presigned URL 발급 요청"
2. 백엔드 → 클라이언트: S3 Presigned URL (PUT, 유효 5분) 반환
3. 클라이언트 → S3: 파일 직접 PUT
4. 클라이언트 → 백엔드: "업로드 완료" 콜백 (파일 경로 저장)
```

```java
// Receipt Service
public PresignedUrlResponse generateUploadUrl(String companyId, String filename) {
    String key = "receipts/%s/%s/%s".formatted(companyId, YearMonth.now(), filename);
    PresignedPutObjectRequest request = s3Presigner.presignPutObject(r -> r
        .signatureDuration(Duration.ofMinutes(5))
        .putObjectRequest(p -> p.bucket(bucket).key(key))
    );
    return new PresignedUrlResponse(request.url().toString(), key);
}
```

### 이미지 리사이징 파이프라인

S3 업로드 완료 이벤트를 받아 썸네일을 자동 생성한다.

```
클라이언트 → S3 직접 PUT (원본 이미지)
  │
  └── S3 Event Notification → SQS (image-processing-queue)
        │
        └── Lambda (ImageProcessor)
              ├── 원본 이미지 리사이즈 (300x300 썸네일)
              ├── WebP 변환 (용량 최적화)
              └── S3 저장: receipts/.../thumb_{filename}
```

| 저장 경로 | 내용 |
|-----------|------|
| `receipts/{company_id}/{year}/{month}/{receipt_id}/original_{filename}` | 원본 |
| `receipts/{company_id}/{year}/{month}/{receipt_id}/thumb_{filename}` | 썸네일 (300px) |

---

<a id="async-export"></a>

## 비동기 내보내기 (엑셀/PDF)

세금계산서 엑셀, 도면 PDF처럼 생성 시간이 긴 작업을 **각 서비스 내부에서 비동기로 처리**한다.
별도 Export 서비스 없이 `common` 공통 모듈을 의존성으로 사용한다.

### 흐름

```
1. 클라이언트 → POST /api/finance/tax-invoices/export
        ↓ (즉시 반환)
2. Finance Service → export_jobs 테이블에 PENDING 기록
                   → { "jobId": "uuid", "status": "PENDING" } 반환
        ↓
3. Finance Service 내부 스케줄러 (@Scheduled)
        ├── common/ExcelBuilder 로 엑셀 생성
        ├── common/S3Uploader 로 S3 저장
        ├── export_jobs 상태 → COMPLETED
        └── Kafka produce: export.completed (다운로드 URL 포함)
        ↓
4. Notification Service: 다운로드 링크 이메일 발송
        ↓
5. 클라이언트 → GET /api/finance/exports/{jobId}
        → { "status": "COMPLETED", "downloadUrl": "..." }
```

### Export Job 상태 테이블 (각 서비스 스키마에 포함)

```sql
-- finance 스키마, drawings 스키마 각각에 동일하게 존재
CREATE TABLE export_jobs (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id   UUID        NOT NULL,
  type         VARCHAR(50) NOT NULL,  -- 'TAX_INVOICE_EXCEL', 'RECEIPT_EXCEL', 'DRAWING_PDF'
  status       VARCHAR(20) NOT NULL DEFAULT 'PENDING',  -- PENDING / PROCESSING / COMPLETED / FAILED
  s3_key       VARCHAR(500),
  error_msg    TEXT,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ
);
```

### common 공통 모듈

| 클래스 | 역할 |
|--------|------|
| `ExcelBuilder` | SheetJS 기반 엑셀 생성 유틸리티 |
| `PdfBuilder` | jsPDF 기반 PDF 생성 유틸리티 |
| `S3Uploader` | Presigned URL 발급, S3 업로드 공통 처리 |
| `ExportJobManager` | export_jobs CRUD, 상태 전이 |

---

<a id="circuit-breaker"></a>

## Circuit Breaker (Resilience4j)

외부 API 호출 및 서비스 간 HTTP 호출 실패 시 장애 전파를 차단한다.

### 적용 대상

| 호출 지점 | 대상 | 설명 |
|-----------|------|------|
| Tax Invoice Service | 결제 API (토스페이먼츠/포트원) | 결제 실패 시 fallback 응답 |
| Drawing Service | 외부 CDN / PDF 렌더러 | 타임아웃 시 기본 응답 |
| Export Worker | S3 업로드 | 실패 시 재시도 후 Job 실패 처리 |

### 설정 예시

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        slidingWindowSize: 10          # 최근 10회 호출 기준
        failureRateThreshold: 50       # 50% 실패 시 OPEN
        waitDurationInOpenState: 30s   # 30초 후 HALF_OPEN 시도
        permittedNumberOfCallsInHalfOpenState: 3
  retry:
    instances:
      payment:
        maxAttempts: 3
        waitDuration: 1s
```

```java
// Tax Invoice Service
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
@Retry(name = "payment")
public PaymentResult charge(PaymentRequest req) {
    return paymentClient.charge(req);
}

public PaymentResult paymentFallback(PaymentRequest req, Exception e) {
    // 결제 실패 시 → 대기 상태로 저장, 관리자 알림
    return PaymentResult.pending("결제 서비스 일시 장애, 잠시 후 재시도합니다.");
}
```

### Circuit Breaker 상태 전이

```
CLOSED (정상)
  → 실패율 50% 초과
OPEN (차단) — fallback 즉시 반환
  → 30초 후
HALF_OPEN (탐색) — 3회 테스트 호출
  → 성공 시 CLOSED / 실패 시 OPEN
```

---

<a id="api-gateway"></a>

## API Gateway

### 역할

| 기능 | 설명 |
|------|------|
| 라우팅 | URL prefix 기반으로 각 마이크로서비스로 프록시 |
| 인증 검증 | JWT 유효성 검사 (Auth Service 위임 또는 Gateway 자체 검증) |
| 속도 제한 | IP·계정별 Rate Limiting |
| 로드밸런싱 | 서비스별 인스턴스 분산 |

### 라우팅 규칙

| 경로 | 서비스 |
|------|--------|
| `/api/auth/*` | Auth Service :3001 |
| `/api/receipts/*`, `/api/tax-invoices/*`, `/api/clients/*`, `/api/finance/*` | Finance Service :3002 |
| `/api/drawings/*` | Drawing Service :3003 |
| `/api/products/*` | Product Service :3004 |
| `/api/dashboard/*` | Analytics Service :3006 |

### 선택 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Kong Gateway** | 오픈소스, 플러그인 풍부, 자체 호스팅 | 운영 부담 |
| **AWS API Gateway** | 완전 관리형, AWS 통합 용이 | 비용, 커스터마이징 제한 |

→ **초기: AWS API Gateway** (운영 부담 최소화) → 트래픽 증가 시 Kong 전환 검토

---

<a id="frontend"></a>

## 프론트엔드

### 기술 스택

| 항목 | 선택 |
|------|------|
| 프레임워크 | React 18 + TypeScript |
| 빌드 도구 | Vite |
| 스타일링 | Tailwind CSS |
| 서버 상태 | TanStack Query (React Query) |
| 클라이언트 상태 | Zustand |
| 폼 | React Hook Form + Zod |
| 도면 편집기 | Fabric.js |
| 바코드·QR 스캔 | ZXing-js |
| PDF 생성 | jsPDF |
| 엑셀 생성 | SheetJS (xlsx) |
| 라우팅 | React Router v6 |
| 아이콘 | Lucide React |

### 반응형 전략

| 화면 | 대상 기능 |
|------|-----------|
| PC (1280px+) | 세금계산서, 도면 편집, 관리 전체 |
| 태블릿 (768px+) | 관리자 전체 기능 |
| 모바일 (360px+) | 직원: 영수증 등록, 제품 스캔·검색 |

---

<a id="database"></a>

## 데이터베이스 전략

### 방식: 단일 PostgreSQL 인스턴스 + 스키마 분리

MSA에서 DB를 서비스별로 완전히 분리하는 것이 이상적이나, 초기에는 운영 비용과 복잡도를 줄이기 위해 **단일 RDS 인스턴스에 스키마(schema)를 서비스별로 분리**한다.

```
PostgreSQL (단일 RDS 인스턴스)
├── schema: auth        ← Auth Service 전용
├── schema: finance     ← Finance Service 전용 (receipts + tax + export_jobs)
├── schema: drawings    ← Drawing Service 전용 (+ export_jobs)
├── schema: products    ← Product Service 전용
└── schema: analytics   ← Analytics Service 전용
```

**장점:** 서비스 간 DB 직접 조인 불가(격리 유지), 운영 인스턴스 최소화
**향후:** 트래픽·규모 증가 시 스키마 단위로 별도 RDS 인스턴스로 분리 가능

### RDS Read Replica

Analytics Service의 집계 쿼리를 Primary와 분리해 Write 성능을 보호한다.

```
RDS Primary  ← 모든 서비스의 Write (INSERT / UPDATE / DELETE)
     │ 복제
RDS Replica  ← Analytics Service 전용 Read
```

Spring의 `AbstractRoutingDataSource`로 트랜잭션 읽기 전용 여부에 따라 자동 라우팅:

```java
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? "replica" : "primary";
    }
}
```

- Analytics Service: `@Transactional(readOnly = true)` → Replica 자동 사용
- 나머지 서비스: Primary 사용

### Redis (ElastiCache)

| 용도 | 설명 |
|------|------|
| 대시보드 집계 캐시 | Analytics Service가 Kafka 이벤트 소비 후 갱신. TTL 없음(이벤트 기반 무효화) |
| 로그인 실패 카운트 | TTL 10분 |
| Refresh Token 블랙리스트 | TTL = 토큰 만료 시간 |
| 제품 목록 캐시 | TTL 5분 |
| Export Job 상태 캐시 | TTL 1시간 (폴링 응답 최적화) |

---

<a id="storage"></a>

## 파일 저장소 (AWS S3)

```
s3://kitchensys-{env}/
├── receipts/{company_id}/{year}/{month}/{receipt_id}/
│   ├── original_{filename}     ← 원본 이미지
│   └── thumb_{filename}        ← 썸네일 (Lambda 자동 생성)
├── products/{company_id}/{product_id}/
│   ├── original_{filename}
│   └── thumb_{filename}
├── drawings/{company_id}/{drawing_id}/exports/
│   └── {drawing_id}.pdf
└── exports/{company_id}/{job_id}/
    └── result.xlsx             ← Export Worker 비동기 생성 결과
```

- **업로드**: S3 Presigned URL — 클라이언트가 S3에 직접 PUT (백엔드 경유 없음)
- **읽기**: CloudFront Signed URL (만료 1시간)
- **이미지 처리**: 업로드 완료 → S3 Event → SQS → Lambda (썸네일 자동 생성)
- **보관**: 영수증 이미지 5년 후 S3 Glacier 이전

---

<a id="infra"></a>

## 인프라 (AWS — ap-northeast-2 서울 리전)

### 서비스 구성

| AWS 서비스 | 용도 |
|------------|------|
| **ECS Fargate** | 각 마이크로서비스 컨테이너 실행 |
| **ECR** | 서비스별 Docker 이미지 저장소 |
| **AWS API Gateway** | API 라우팅·인증·Rate Limiting |
| **MSK (Managed Kafka)** | Kafka 클러스터 관리형 서비스 |
| **SQS** | S3 이미지 업로드 이벤트 큐 (Lambda 트리거) |
| **Lambda** | 이미지 리사이징·썸네일 생성 (서버리스) |
| **RDS PostgreSQL** | 메인 DB — Primary Write + Replica Read 분리 |
| **ElastiCache Redis** | 캐시·세션·Export Job 상태 |
| **S3 + CloudFront** | 파일 저장 (Presigned URL 직접 업로드) + CDN |
| **Route 53** | DNS |
| **ACM** | SSL/TLS |
| **SES** | 이메일 발송 |
| **ALB** | 서비스별 내부 로드밸런서 |

### 환경 구성

| 환경 | 구성 |
|------|------|
| **dev** | 단일 컨테이너, RDS t3.micro, MSK 단일 브로커 |
| **staging** | dev 규모, Kafka 2 브로커 |
| **prod** | ECS 오토스케일링, RDS Multi-AZ, MSK 3 브로커 |

### 네트워크

```
VPC (10.0.0.0/16)
├── Public Subnet    → API Gateway, ALB, NAT Gateway
└── Private Subnet   → ECS Tasks, RDS, ElastiCache, MSK
```

---

<a id="external"></a>

## 외부 연동

| 연동 대상 | 방식 | 용도 |
|-----------|------|------|
| 홈택스 | **Strategy Pattern** (아래 상세 참고) | 세금계산서 제출 방식을 교체 가능하게 설계 |
| 바코드·QR 스캔 | ZXing-js (브라우저 카메라 API) | 앱 설치 없이 웹에서 스캔 |
| 결제 | 토스페이먼츠 or 포트원 v2 | 플랜 업그레이드 결제 |
| 이메일 | AWS SES | 직원 초대, 비밀번호 재설정 |

### 홈택스 연동 전략 (Strategy Pattern)

세금계산서 제출 방식이 향후 변경될 수 있으므로 **전략 패턴**으로 교체 가능하게 설계한다.

#### 지원 전략 3가지

| 전략 | 시점 | 방식 | 조건 |
|------|------|------|------|
| **ExcelExportStrategy** | 현재 (Phase 1) | SheetJS로 홈택스 엑셀 생성 → 사용자가 직접 업로드 | 외부 계약 없이 즉시 사용 가능 |
| **RelayProviderStrategy** | Phase 2 | 중계사업자 API (이카운트·더존·스마트빌 등) 연동 | 중계사업자 계약 후 전환 |
| **NtsDirectStrategy** | Phase 3 | 국세청 전자세금계산서 API 직접 연동 | 공인인증서·ASP 자격 취득 후 전환 |

#### 인터페이스 설계

```java
// TaxInvoiceSubmitter.java
public interface TaxInvoiceSubmitter {
    SubmitResult submit(TaxInvoice invoice);
}

// Phase 1 — 엑셀 생성
@Component
@ConditionalOnProperty(name = "tax.submit.strategy", havingValue = "excel")
public class ExcelExportStrategy implements TaxInvoiceSubmitter {
    public SubmitResult submit(TaxInvoice invoice) {
        byte[] excel = excelBuilder.buildHometaxExcel(invoice);
        String s3Key = s3Uploader.upload("exports/tax/" + invoice.getId() + ".xlsx", excel);
        return SubmitResult.downloadReady(s3Key);
    }
}

// Phase 2 — 중계사업자 API
@Component
@ConditionalOnProperty(name = "tax.submit.strategy", havingValue = "relay")
public class RelayProviderStrategy implements TaxInvoiceSubmitter {
    public SubmitResult submit(TaxInvoice invoice) {
        return relayClient.issue(invoice);  // 이카운트·더존 등 API 호출
    }
}
```

#### 전환 방법

`application.yml` 값 하나만 바꾸면 전략이 교체된다. 코드 변경 없음.

```yaml
# Phase 1 (현재)
tax:
  submit:
    strategy: excel

# Phase 2로 전환 시
tax:
  submit:
    strategy: relay
    relay-provider: smartbill   # 이카운트 / 더존 / 스마트빌
    api-key: ${TAX_RELAY_API_KEY}
```

#### Circuit Breaker 적용 대상

Phase 2 이후 외부 API 호출이 생기므로 `RelayProviderStrategy`, `NtsDirectStrategy`에 Circuit Breaker 필수 적용.

```java
@CircuitBreaker(name = "taxRelay", fallbackMethod = "fallbackToExcel")
public SubmitResult submit(TaxInvoice invoice) { ... }

public SubmitResult fallbackToExcel(TaxInvoice invoice, Exception e) {
    // 외부 API 장애 시 엑셀 다운로드로 자동 폴백
    return excelExportStrategy.submit(invoice);
}
```

---

<a id="cicd"></a>

## CI/CD

### 파이프라인 (GitHub Actions)

```
Push to branch
    │
    ├── [Test]   유닛·통합 테스트 (서비스별 병렬 실행)
    │
    ├── [Build]  서비스별 Docker 이미지 빌드 → ECR 푸시
    │            (변경된 서비스만 빌드 — path filter 적용)
    │
    └── [Deploy]
            ├── develop → staging 자동 배포
            └── main    → prod 수동 승인 후 배포
```

### 모노레포 구조

```
# kitchen-backend (별도 저장소)
kitchen-backend/
├── services/
│   ├── common/         ← 공통 모듈 (ExcelBuilder, PdfBuilder, S3Uploader, ExportJobManager)
│   ├── auth/
│   ├── finance/        ← Receipt + Tax Invoice 통합
│   ├── drawing/
│   ├── product/
│   ├── notification/
│   └── analytics/
├── lambda/
│   └── image-resizer/  ← AWS Lambda (이미지 리사이징)
└── infra/              ← Terraform (AWS 리소스 정의)

# kitchen-frontend (별도 저장소)
kitchen-frontend/
├── src/
│   ├── pages/          ← 화면 단위 컴포넌트
│   ├── components/     ← 공통 UI 컴포넌트
│   ├── hooks/          ← 커스텀 훅 (TanStack Query)
│   ├── stores/         ← Zustand 전역 상태
│   ├── api/            ← API 클라이언트 (axios)
│   └── types/          ← 공통 타입 정의
├── public/
├── index.html
├── vite.config.ts
└── package.json
```

---

<a id="stack-summary"></a>

## 기술 스택 요약

| 영역 | 기술 |
|------|------|
| **프론트엔드** | React 18, TypeScript, Vite, Tailwind CSS |
| **상태 관리** | TanStack Query, Zustand |
| **도면 편집** | Fabric.js |
| **바코드 스캔** | ZXing-js |
| **PDF/엑셀** | jsPDF, SheetJS |
| **백엔드** | Spring Boot 3.x, Java 21, Spring Cloud |
| **공통 모듈** | `common/` — ExcelBuilder, PdfBuilder, S3Uploader, ExportJobManager |
| **ORM** | Spring Data JPA + Hibernate |
| **Kafka** | Spring Kafka |
| **인증** | Spring Security + JWT |
| **빌드** | Gradle |
| **장애 격리** | Resilience4j (Circuit Breaker, Retry) |
| **메시지 브로커** | Apache Kafka (AWS MSK) |
| **이벤트 큐** | AWS SQS (이미지 처리 트리거) |
| **서버리스** | AWS Lambda (이미지 리사이징) |
| **API Gateway** | AWS API Gateway (초기) → Kong (확장 시) |
| **메인 DB** | PostgreSQL (AWS RDS, Primary + Read Replica) |
| **캐시** | Redis (AWS ElastiCache) |
| **파일 저장** | AWS S3 (Presigned URL 직접 업로드) + CloudFront |
| **컨테이너** | Docker, AWS ECS Fargate |
| **IaC** | Terraform |
| **이메일** | AWS SES |
| **결제** | 토스페이먼츠 or 포트원 v2 |
| **CI/CD** | GitHub Actions |
| **DNS/SSL** | AWS Route 53, ACM |
