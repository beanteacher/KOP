# 04 — 시스템 아키텍처 (NestJS 버전)

> **문서 상태**: 참고용 (학습·실험 목적)
> **작성일**: 2026-09-05
> **메인 문서**: `04-system-architecture.md` (Spring Boot — 실제 채택)
> **이 문서의 목적**: NestJS + TypeScript 풀스택으로 동일한 MSA + Kafka 구조를 구성할 경우의 설계

---

## 목차

- [Spring Boot 대비 차이점](#diff)
- [백엔드 기술 스택](#backend)
- [Kafka 연동 방식](#kafka)
- [서비스 구조](#services)
- [모노레포 구조](#monorepo)
- [기술 스택 요약](#stack-summary)

---

<a id="diff"></a>

## Spring Boot 대비 주요 차이점

| 항목 | Spring Boot (메인) | NestJS (이 문서) |
|------|-------------------|-----------------|
| 언어 | Java 21 | TypeScript (Node.js 20) |
| 프레임워크 | Spring Boot 3.x | NestJS 10.x |
| ORM | Spring Data JPA | Prisma |
| Kafka | Spring Kafka | @nestjs/microservices (KafkaJS) |
| 인증 | Spring Security | Passport.js + JWT |
| 빌드 | Gradle | pnpm workspaces (모노레포) |
| 구동 방식 | JVM (멀티스레드) | Event Loop (단일스레드, 비동기) |
| 컨테이너 | Docker (JVM 이미지) | Docker (Node.js 이미지, 경량) |

---

<a id="backend"></a>

## 백엔드 기술 스택

| 항목 | 선택 | 설명 |
|------|------|------|
| 런타임 | Node.js 20 LTS | |
| 프레임워크 | NestJS 10 | DI, 데코레이터, 모듈 구조 |
| MSA 지원 | @nestjs/microservices | Kafka Transport 내장 |
| ORM | Prisma | TypeScript 친화적, 마이그레이션 관리 |
| 인증 | @nestjs/passport + JWT | Passport 전략 기반 |
| Kafka | KafkaJS (via @nestjs/microservices) | |
| 파일 업로드 | Multer + AWS SDK v3 | |
| 이메일 | AWS SES SDK | |
| 유효성 검증 | class-validator + class-transformer | |
| 빌드 도구 | pnpm workspaces + Turborepo | 모노레포 빌드 최적화 |

---

<a id="kafka"></a>

## Kafka 연동 방식 (NestJS)

### Producer — 이벤트 발행

```typescript
// finance/src/receipt/receipt.service.ts
import { Inject } from '@nestjs/common';
import { ClientKafka } from '@nestjs/microservices';

@Injectable()
export class ReceiptService {
  constructor(
    @Inject('KAFKA_CLIENT') private readonly kafkaClient: ClientKafka,
    private readonly prisma: PrismaService,
  ) {}

  async create(companyId: string, dto: CreateReceiptDto) {
    // Outbox 패턴: receipt + outbox 이벤트를 트랜잭션으로 함께 저장
    const [receipt] = await this.prisma.$transaction([
      this.prisma.receipt.create({ data: { ...dto, companyId } }),
      this.prisma.outbox.create({
        data: {
          eventType: 'receipt.created',
          payload: JSON.stringify({ companyId, amount: dto.amount }),
          status: 'PENDING',
        },
      }),
    ]);

    return receipt;
  }

  // 엑셀 내보내기 — common 모듈 사용, 비동기 인라인 처리
  async requestExport(companyId: string, filter: ExportFilter) {
    const job = await this.prisma.exportJob.create({
      data: { companyId, type: 'RECEIPT_EXCEL', status: 'PENDING' },
    });
    // @Cron 스케줄러가 PENDING 잡 처리 → ExcelBuilder → S3Uploader → export.completed 발행
    return { jobId: job.id, status: 'PENDING' };
  }
}
```

### Consumer — 이벤트 구독

```typescript
// analytics.controller.ts
@Controller()
export class AnalyticsController {
  constructor(private readonly analyticsService: AnalyticsService) {}

  @EventPattern('receipt.created')
  async handleReceiptCreated(@Payload() data: ReceiptCreatedEvent, @Ctx() context: KafkaContext) {
    await this.analyticsService.updateMonthlySum(data.companyId, data.payload.amount);
    context.getMessage().ack();
  }

  @EventPattern('export.completed')
  async handleExportCompleted(@Payload() data: ExportCompletedEvent, @Ctx() context: KafkaContext) {
    // Notification Service에서 구독 → 다운로드 링크 이메일 발송
    await this.notificationService.sendExportReadyEmail(data.companyId, data.payload.downloadUrl);
    context.getMessage().ack();
  }
}
```

### 서비스 등록 (main.ts)

```typescript
// analytics/src/main.ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AnalyticsModule,
  {
    transport: Transport.KAFKA,
    options: {
      client: { brokers: ['kafka:9092'] },
      consumer: { groupId: 'analytics-consumer' },
    },
  },
);
await app.listen();
```

---

<a id="services"></a>

## 서비스 구조

Spring Boot 버전과 동일한 **6개 서비스** 구성. 차이점만 기술.

| 서비스 | 포트 | 특이사항 |
|--------|------|----------|
| auth | 3001 | Passport.js LocalStrategy + JwtStrategy |
| finance | 3002 | Receipt + Tax Invoice 통합. S3 Presigned URL, 엑셀·PDF 내보내기 인라인 처리 |
| drawing | 3003 | Fabric.js JSON 저장 (Prisma Json 타입). PDF 내보내기 인라인 처리 |
| product | 3004 | ZXing-js 스캔 결과는 프론트에서 처리, 서버는 코드 조회만 |
| notification | 3005 | AWS SES SDK v3. `export.completed` 구독 → 다운로드 링크 이메일 |
| analytics | 3006 | Kafka Consumer 전용. Redis 캐시 갱신. Read Replica 사용 |

### 공통 모듈 (packages/common)

Finance / Drawing 서비스가 공유하는 파일 생성·업로드 유틸리티.

| 모듈 | 역할 |
|------|------|
| `ExcelBuilder` | SheetJS 래퍼 — 영수증·세금계산서 엑셀 생성 |
| `PdfBuilder` | jsPDF 래퍼 — 도면 PDF 생성 |
| `S3Uploader` | Presigned URL 발급, S3 업로드 공통 처리 |
| `ExportJobManager` | export_jobs CRUD, 상태 전이 (PENDING → COMPLETED) |

```typescript
// packages/common/src/export/excel-builder.ts
export class ExcelBuilder {
  static buildReceiptExcel(receipts: Receipt[]): Buffer { ... }
  static buildTaxInvoiceExcel(invoices: TaxInvoice[]): Buffer { ... }
}

// packages/common/src/storage/s3-uploader.ts
export class S3Uploader {
  async generatePresignedUrl(key: string, expiresIn = 300): Promise<string> { ... }
  async upload(key: string, buffer: Buffer): Promise<string> { ... }
}
```

---

<a id="monorepo"></a>

## 모노레포 구조

```
# kitchen-backend (별도 저장소) — pnpm workspaces + Turborepo
kitchen-backend/
├── apps/
│   ├── auth/                  ← NestJS 앱
│   │   ├── src/
│   │   ├── prisma/schema.prisma
│   │   └── package.json
│   ├── finance/               ← Receipt + Tax Invoice 통합
│   ├── drawing/
│   ├── product/
│   ├── notification/
│   └── analytics/
├── packages/
│   ├── shared-types/          ← 공통 DTO, 이벤트 타입 정의
│   ├── kafka-events/          ← Kafka 이벤트 스키마
│   └── common/                ← 공통 유틸 (ExcelBuilder, PdfBuilder, S3Uploader, ExportJobManager)
├── lambda/
│   └── image-resizer/         ← AWS Lambda (이미지 리사이징)
├── infra/                     ← Terraform
├── pnpm-workspace.yaml
└── turbo.json                 ← Turborepo 빌드 파이프라인

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

### 공통 이벤트 타입 (packages/kafka-events)

```typescript
// packages/kafka-events/src/receipt.events.ts
export interface ReceiptCreatedEvent {
  eventId: string;
  eventType: 'receipt.created';
  companyId: string;
  timestamp: string;
  payload: {
    receiptId: string;
    amount: number;
    category: string;
  };
}
```

---

<a id="stack-summary"></a>

## 홈택스 연동 전략 (NestJS 구현)

메인 문서의 Strategy Pattern과 동일한 설계. NestJS에서는 커스텀 프로바이더로 전략을 주입한다.

```typescript
// finance/src/tax-invoice/submitter/tax-invoice-submitter.interface.ts
export interface TaxInvoiceSubmitter {
  submit(invoice: TaxInvoice): Promise<SubmitResult>;
}

// finance/src/tax-invoice/submitter/excel-export.strategy.ts
@Injectable()
export class ExcelExportStrategy implements TaxInvoiceSubmitter {
  async submit(invoice: TaxInvoice): Promise<SubmitResult> {
    const buffer = ExcelBuilder.buildTaxInvoiceExcel(invoice);
    const key = await this.s3Uploader.upload(`exports/tax/${invoice.id}.xlsx`, buffer);
    return { type: 'DOWNLOAD', s3Key: key };
  }
}

// finance/src/tax-invoice/tax-invoice.module.ts
const submitterProvider = {
  provide: 'TAX_INVOICE_SUBMITTER',
  useClass: configService.get('TAX_SUBMIT_STRATEGY') === 'relay'
    ? RelayProviderStrategy
    : ExcelExportStrategy,
};
```

전략 전환은 환경변수 `TAX_SUBMIT_STRATEGY=relay` 변경만으로 가능.

---

## 기술 스택 요약

| 영역 | 기술 |
|------|------|
| **프론트엔드** | React 18, TypeScript, Vite, Tailwind CSS |
| **상태 관리** | TanStack Query, Zustand |
| **도면 편집** | Fabric.js |
| **바코드 스캔** | ZXing-js |
| **PDF/엑셀** | jsPDF, SheetJS |
| **백엔드** | NestJS 10, TypeScript, Node.js 20 |
| **공통 모듈** | `packages/common/` — ExcelBuilder, PdfBuilder, S3Uploader, ExportJobManager |
| **MSA** | @nestjs/microservices |
| **ORM** | Prisma |
| **Kafka** | KafkaJS (via @nestjs/microservices) |
| **인증** | Passport.js + JWT |
| **장애 격리** | @nestjs/throttler + 커스텀 Circuit Breaker 인터셉터 |
| **빌드** | pnpm workspaces + Turborepo |
| **메시지 브로커** | Apache Kafka (AWS MSK) |
| **API Gateway** | AWS API Gateway → Kong |
| **메인 DB** | PostgreSQL (AWS RDS, 스키마 분리) |
| **캐시** | Redis (AWS ElastiCache) |
| **파일 저장** | AWS S3 + CloudFront |
| **컨테이너** | Docker, AWS ECS Fargate |
| **IaC** | Terraform |
| **이메일** | AWS SES |
| **결제** | 토스페이먼츠 or 포트원 v2 |
| **CI/CD** | GitHub Actions |
