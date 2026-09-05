# 04 — 시스템 아키텍처 (FastAPI 버전)

> **문서 상태**: 참고용 (학습·실험 목적)
> **작성일**: 2026-09-05
> **메인 문서**: `04-system-architecture.md` (Spring Boot — 실제 채택)
> **이 문서의 목적**: FastAPI + Python으로 동일한 MSA + Kafka 구조를 구성할 경우의 설계

---

## 목차

- [Spring Boot 대비 차이점](#diff)
- [백엔드 기술 스택](#backend)
- [Kafka 연동 방식](#kafka)
- [서비스 구조](#services)
- [프로젝트 구조](#project-structure)
- [FastAPI 특이사항](#fastapi-notes)
- [기술 스택 요약](#stack-summary)

---

<a id="diff"></a>

## Spring Boot 대비 주요 차이점

| 항목 | Spring Boot (메인) | FastAPI (이 문서) |
|------|-------------------|-----------------|
| 언어 | Java 21 | Python 3.12 |
| 프레임워크 | Spring Boot 3.x | FastAPI 0.11x |
| ORM | Spring Data JPA | SQLAlchemy 2.0 + Alembic |
| Kafka | Spring Kafka | aiokafka (비동기) |
| 인증 | Spring Security | python-jose + passlib |
| 빌드/패키지 | Gradle | uv (패키지 매니저) |
| 비동기 | 멀티스레드 | asyncio (단일스레드 이벤트 루프) |
| 타입 시스템 | 정적 (Java) | 정적 힌트 (Pydantic v2) |
| API 문서 | Swagger (Springdoc) | Swagger 자동 생성 (/docs) |

---

<a id="backend"></a>

## 백엔드 기술 스택

| 항목 | 선택 | 설명 |
|------|------|------|
| 언어 | Python 3.12 | |
| 프레임워크 | FastAPI 0.11x | 비동기, 자동 OpenAPI 문서 |
| 데이터 검증 | Pydantic v2 | DTO, 요청/응답 스키마 정의 |
| ORM | SQLAlchemy 2.0 (async) | 비동기 DB 접근 |
| DB 마이그레이션 | Alembic | |
| Kafka | aiokafka | Python 비동기 Kafka 클라이언트 |
| 인증 | python-jose (JWT) + passlib (bcrypt) | |
| 파일 업로드 | boto3 (AWS SDK) | S3 업로드 |
| 이메일 | boto3 (SES) | |
| 패키지 매니저 | uv | pip 대비 10~100x 빠름 |
| ASGI 서버 | Uvicorn + Gunicorn | 프로덕션 배포 |

---

<a id="kafka"></a>

## Kafka 연동 방식 (FastAPI + aiokafka)

### Producer — 이벤트 발행

```python
# receipt/infrastructure/kafka/producer.py
from aiokafka import AIOKafkaProducer
import json, uuid
from datetime import datetime, timezone

class KafkaProducer:
    def __init__(self):
        self._producer: AIOKafkaProducer | None = None

    async def start(self, bootstrap_servers: str):
        self._producer = AIOKafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode(),
        )
        await self._producer.start()

    async def emit(self, topic: str, payload: dict):
        event = {
            "eventId": str(uuid.uuid4()),
            "eventType": topic,
            "timestamp": datetime.now(timezone.utc).isoformat(),
            **payload,
        }
        await self._producer.send_and_wait(topic, event)

    async def stop(self):
        await self._producer.stop()
```

```python
# finance/app/application/receipt_service.py
from finance.infrastructure.kafka.producer import KafkaProducer
from finance.infrastructure.db.outbox import OutboxRepository
from shared.common.excel_builder import ExcelBuilder
from shared.common.s3_uploader import S3Uploader
from shared.common.export_job_manager import ExportJobManager

class ReceiptService:
    def __init__(self, repo: ReceiptRepository, kafka: KafkaProducer,
                 outbox: OutboxRepository, export_mgr: ExportJobManager):
        self.repo = repo
        self.kafka = kafka
        self.outbox = outbox
        self.export_mgr = export_mgr

    async def create(self, company_id: str, dto: CreateReceiptDto) -> Receipt:
        # Outbox 패턴: receipt + outbox 이벤트를 트랜잭션으로 함께 저장
        async with self.repo.transaction():
            receipt = await self.repo.save(Receipt(company_id=company_id, **dto.model_dump()))
            await self.outbox.save(event_type="receipt.created", payload={
                "companyId": company_id,
                "payload": {"receiptId": str(receipt.id), "amount": receipt.amount},
            })
        return receipt

    async def request_export(self, company_id: str, filter: ExportFilter) -> dict:
        # 엑셀 내보내기 Job 등록 — 내부 스케줄러가 처리
        job = await self.export_mgr.create(company_id, "RECEIPT_EXCEL")
        return {"jobId": str(job.id), "status": "PENDING"}
```

### Consumer — 이벤트 구독

```python
# analytics/infrastructure/kafka/consumer.py
from aiokafka import AIOKafkaConsumer
import json, asyncio

class KafkaConsumerRunner:
    async def run(self, bootstrap_servers: str, topics: list[str], group_id: str, handler):
        consumer = AIOKafkaConsumer(
            *topics,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            value_deserializer=lambda v: json.loads(v.decode()),
            enable_auto_commit=False,
        )
        await consumer.start()
        try:
            async for msg in consumer:
                await handler(msg.value)
                await consumer.commit()
        finally:
            await consumer.stop()

# analytics/main.py — FastAPI lifespan으로 Consumer 실행
@asynccontextmanager
async def lifespan(app: FastAPI):
    runner = KafkaConsumerRunner()
    task = asyncio.create_task(
        runner.run(
            bootstrap_servers=settings.KAFKA_BROKERS,
            topics=["receipt.created", "receipt.deleted", "tax_invoice.created", "product.viewed"],
            group_id="analytics-consumer",
            handler=analytics_handler,
        )
    )
    # Notification: 내보내기 완료 + 직원 초대 구독
    asyncio.create_task(
        runner.run(
            bootstrap_servers=settings.KAFKA_BROKERS,
            topics=["export.completed", "employee.invited", "tax_invoice.created"],
            group_id="notification-consumer",
            handler=notification_handler,
        )
    )
    yield
    task.cancel()
```

### FastAPI 라우터 (API 엔드포인트)

```python
# receipt/api/router.py
from fastapi import APIRouter, Depends, UploadFile

router = APIRouter(prefix="/receipts", tags=["receipts"])

@router.post("/", response_model=ReceiptResponse, status_code=201)
async def create_receipt(
    dto: CreateReceiptDto,
    service: ReceiptService = Depends(get_receipt_service),
    current_user: CurrentUser = Depends(get_current_user),
):
    return await service.create(current_user.company_id, dto)

@router.post("/{receipt_id}/image")
async def upload_image(
    receipt_id: UUID,
    file: UploadFile,
    service: ReceiptService = Depends(get_receipt_service),
):
    return await service.upload_image(receipt_id, file)
```

---

<a id="services"></a>

## 서비스 구조

Spring Boot 버전과 동일한 **6개 서비스**. 차이점만 기술.

| 서비스 | 포트 | 특이사항 |
|--------|------|----------|
| auth | 8001 | python-jose JWT, passlib bcrypt |
| finance | 8002 | Receipt + Tax Invoice 통합. S3 Presigned URL, 엑셀·PDF 내보내기 인라인 처리 |
| drawing | 8003 | Fabric.js JSON → PostgreSQL JSONB. PDF 내보내기 인라인 처리 |
| product | 8004 | 바코드 코드 조회 (스캔은 프론트) |
| notification | 8005 | boto3 SES. `export.completed` 구독 → 다운로드 링크 이메일 |
| analytics | 8006 | aiokafka Consumer + Redis 캐시. Read Replica 사용 |

### 공통 모듈 (shared/common)

Finance / Drawing 서비스가 공유하는 파일 생성·업로드 유틸리티.

| 모듈 | 역할 |
|------|------|
| `excel_builder.py` | openpyxl 래퍼 — 영수증·세금계산서 엑셀 생성 |
| `pdf_builder.py` | reportlab/fpdf2 래퍼 — 도면 PDF 생성 |
| `s3_uploader.py` | Presigned URL 발급, S3 업로드 공통 처리 |
| `export_job_manager.py` | export_jobs CRUD, 상태 전이 (PENDING → COMPLETED) |

```python
# shared/common/excel_builder.py
import openpyxl
from io import BytesIO

class ExcelBuilder:
    @staticmethod
    def build_receipt_excel(receipts: list[Receipt]) -> bytes:
        wb = openpyxl.Workbook()
        ws = wb.active
        # ... 시트 구성
        buf = BytesIO()
        wb.save(buf)
        return buf.getvalue()

# shared/common/s3_uploader.py
import boto3
from botocore.config import Config

class S3Uploader:
    async def generate_presigned_url(self, key: str, expires_in: int = 300) -> str: ...
    async def upload(self, key: str, data: bytes) -> str: ...
```

---

<a id="project-structure"></a>

## 프로젝트 구조

```
# kitchen-backend (별도 저장소)
kitchen-backend/
├── services/
│   ├── auth/
│   │   ├── app/
│   │   │   ├── api/             ← FastAPI 라우터
│   │   │   ├── application/     ← 서비스 레이어
│   │   │   ├── domain/          ← 엔티티, 도메인 로직
│   │   │   └── infrastructure/  ← DB, Kafka, S3
│   │   ├── alembic/             ← DB 마이그레이션
│   │   ├── main.py
│   │   └── pyproject.toml
│   ├── finance/                 ← Receipt + Tax Invoice 통합
│   ├── drawing/
│   ├── product/
│   ├── notification/
│   └── analytics/
├── shared/
│   ├── kafka_events/            ← 공통 Pydantic 이벤트 스키마
│   └── common/                  ← 파일 생성·업로드 공통 모듈
│       ├── excel_builder.py     ← openpyxl 래퍼
│       ├── pdf_builder.py       ← reportlab/fpdf2 래퍼
│       ├── s3_uploader.py       ← Presigned URL, S3 업로드
│       └── export_job_manager.py
├── lambda/
│   └── image_resizer/           ← AWS Lambda (이미지 리사이징)
└── infra/                       ← Terraform

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

### 공통 이벤트 스키마 (shared/kafka_events)

```python
# shared/kafka_events/receipt.py
from pydantic import BaseModel
from datetime import datetime
from uuid import UUID

class ReceiptCreatedPayload(BaseModel):
    receipt_id: UUID
    amount: int
    category: str

class ReceiptCreatedEvent(BaseModel):
    event_id: UUID
    event_type: str = "receipt.created"
    company_id: UUID
    timestamp: datetime
    payload: ReceiptCreatedPayload
```

---

<a id="fastapi-notes"></a>

## FastAPI 특이사항

### 장점

| 항목 | 내용 |
|------|------|
| **자동 API 문서** | `/docs` (Swagger UI), `/redoc` 자동 생성 — 별도 설정 불필요 |
| **Pydantic 검증** | 요청/응답 스키마가 곧 타입 문서 — DTO 선언만으로 검증·직렬화 완성 |
| **비동기 기본** | `async def` 기반 — aiokafka, asyncpg와 자연스럽게 통합 |
| **경량 이미지** | Python 이미지가 JVM 대비 빠른 콜드 스타트 |

### 주의점

| 항목 | 내용 |
|------|------|
| **GIL** | CPU 집중 작업은 멀티프로세스(Gunicorn workers)로 대응 |
| **Kafka 생태계** | Spring Kafka 대비 aiokafka는 커뮤니티 작음 → 트러블슈팅 자료 적음 |
| **ORM 성숙도** | SQLAlchemy async는 Spring Data JPA 대비 설정 복잡 |
| **엔터프라이즈** | 대규모 팀·복잡한 도메인에서는 Spring Boot보다 구조화 어려울 수 있음 |

---

<a id="stack-summary"></a>

## 홈택스 연동 전략 (FastAPI 구현)

메인 문서의 Strategy Pattern과 동일한 설계. FastAPI에서는 의존성 주입(Depends)으로 전략을 교체한다.

```python
# finance/app/domain/tax_invoice_submitter.py
from abc import ABC, abstractmethod

class TaxInvoiceSubmitter(ABC):
    @abstractmethod
    async def submit(self, invoice: TaxInvoice) -> SubmitResult: ...

# finance/app/infrastructure/submitter/excel_export_strategy.py
class ExcelExportStrategy(TaxInvoiceSubmitter):
    async def submit(self, invoice: TaxInvoice) -> SubmitResult:
        data = ExcelBuilder.build_tax_invoice_excel(invoice)
        key = await self.s3_uploader.upload(f"exports/tax/{invoice.id}.xlsx", data)
        return SubmitResult(type="DOWNLOAD", s3_key=key)

# finance/app/infrastructure/submitter/relay_provider_strategy.py
class RelayProviderStrategy(TaxInvoiceSubmitter):
    async def submit(self, invoice: TaxInvoice) -> SubmitResult:
        return await self.relay_client.issue(invoice)  # 이카운트·더존 등

# finance/app/dependencies.py — 환경변수로 전략 선택
def get_tax_submitter() -> TaxInvoiceSubmitter:
    strategy = settings.TAX_SUBMIT_STRATEGY  # "excel" | "relay" | "nts"
    if strategy == "relay":
        return RelayProviderStrategy(relay_client=get_relay_client())
    return ExcelExportStrategy(s3_uploader=get_s3_uploader())

# finance/app/api/tax_invoice_router.py
@router.post("/{invoice_id}/submit")
async def submit_invoice(
    invoice_id: UUID,
    submitter: TaxInvoiceSubmitter = Depends(get_tax_submitter),
    service: TaxInvoiceService = Depends(get_tax_invoice_service),
):
    invoice = await service.get(invoice_id)
    return await submitter.submit(invoice)
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
| **PDF/엑셀** | jsPDF, openpyxl (서버사이드) |
| **백엔드** | FastAPI 0.11x, Python 3.12 |
| **공통 모듈** | `shared/common/` — excel_builder, pdf_builder, s3_uploader, export_job_manager |
| **데이터 검증** | Pydantic v2 |
| **ORM** | SQLAlchemy 2.0 (async) + Alembic |
| **Kafka** | aiokafka |
| **인증** | python-jose + passlib |
| **장애 격리** | tenacity (Retry) + circuitbreaker 라이브러리 |
| **엑셀 생성** | openpyxl (서버사이드, common 모듈) |
| **PDF 생성** | reportlab or fpdf2 (서버사이드, common 모듈) |
| **패키지 관리** | uv |
| **ASGI 서버** | Uvicorn + Gunicorn |
| **메시지 브로커** | Apache Kafka (AWS MSK) |
| **API Gateway** | AWS API Gateway → Kong |
| **메인 DB** | PostgreSQL (AWS RDS, 스키마 분리) |
| **캐시** | Redis (AWS ElastiCache) |
| **파일 저장** | AWS S3 + CloudFront |
| **컨테이너** | Docker, AWS ECS Fargate |
| **IaC** | Terraform |
| **이메일** | AWS SES (boto3) |
| **결제** | 토스페이먼츠 or 포트원 v2 |
| **CI/CD** | GitHub Actions |
