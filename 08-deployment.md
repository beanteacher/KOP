# 08 — 배포 전략·인프라·CI/CD

> **문서 상태**: 초안 (Draft)
> **작성일**: 2026-09-11
> **버전**: v0.1
> **연관 문서**: `04-system-architecture.md`(인프라·CI/CD 개요) · `01-prd.md`(비기능 요구사항)

이 문서는 `04-system-architecture.md`의 [인프라](04-system-architecture.md#infra)·[CI/CD](04-system-architecture.md#cicd) 절에서 이미 정한 것(AWS 서울 리전, ECS Fargate, GitHub Actions, 환경 3단 구성)을 실행 가능한 수준까지 구체화한다. 스택 자체를 재논의하지 않는다.

---

## 목차

- [배포 목표](#goals)
- [환경 구성](#environments)
- [도메인 구조](#domains)
- [네트워크](#network)
- [컨테이너 배포 (ECS Fargate)](#ecs)
- [무중단 배포 전략](#zero-downtime)
- [DB 마이그레이션](#migration)
- [CI/CD 파이프라인](#pipeline)
- [시크릿 관리](#secrets)
- [모니터링·로깅·알림](#observability)
- [백업·재해복구](#backup)
- [비용 개요](#cost)
- [배포 전 체크리스트](#checklist)

---

<a id="goals"></a>

## 배포 목표

`01-prd.md` 비기능 요구사항을 인프라 목표로 변환한다.

| PRD 요구사항 | 인프라 목표 |
|---|---|
| 가용성 월 99% 이상, 세금계산서 마감일 전후 중단 없음 | prod 무중단 배포(Rolling) 필수, 마감기간(매월 초 며칠)은 배포 동결(freeze) 권장 |
| 영수증 1,000건 3초 이내 로딩 | Analytics Read Replica·Redis 캐시로 조회 경로 분리(04번 기 반영), ALB 응답시간 알람 임계치 근거로 사용 |
| 영수증·도면 5년 보관 | S3 Lifecycle로 Glacier 이전 자동화, RDS 백업 보관주기와는 별도 관리 |
| HTTPS 필수·데이터 암호화 | ACM 인증서 강제(HTTP→HTTPS 리다이렉트), RDS·S3 저장 시 암호화(SSE-KMS) |

---

<a id="environments"></a>

## 환경 구성

`04-system-architecture.md`의 표를 인스턴스 스펙까지 확정한다.

| 항목 | dev | staging | prod |
|---|---|---|---|
| ECS Task 수 (서비스당) | 1 | 1 | 2 (오토스케일링 2~6) |
| Task CPU/Memory | 0.25 vCPU / 512MB | 0.25 vCPU / 512MB | 0.5 vCPU / 1024MB |
| RDS | db.t3.micro, Single-AZ | db.t3.micro, Single-AZ | db.r6g.large, **Multi-AZ** |
| MSK 브로커 | 1 (kafka.t3.small) | 2 (kafka.t3.small) | 3 (kafka.m5.large) |
| ElastiCache | 없음(로컬 in-memory 대체) | cache.t3.micro ×1 | cache.t3.micro ×2 (Primary+Replica) |
| 배포 트리거 | PR 브랜치 수동 | `develop` 머지 시 자동 | `main` 머지 + 수동 승인 |
| 접근 | 팀 VPN/사내망만 | 팀 + QA 초대 링크 | 공개 |
| 데이터 | 시드 데이터 | 익명화된 prod 스냅샷 또는 시드 | 실 데이터 |

**dev는 상시 기동하지 않는다** — 사용하지 않을 때는 ECS 서비스 desired count 0으로 내려 비용을 아낀다(스케줄러로 평일 09~22시만 기동).

---

<a id="domains"></a>

## 도메인 구조

Route 53 + ACM. 서비스 도메인은 `kitchensys.com`(가칭)을 기준으로 한다.

| 환경 | 프론트엔드 | API |
|---|---|---|
| prod | `app.kitchensys.com` | `api.kitchensys.com` |
| staging | `staging.kitchensys.com` | `api-staging.kitchensys.com` |
| dev | `dev.kitchensys.com` | `api-dev.kitchensys.com` |
| 도면 공유 링크 | `share.kitchensys.com/{token}` | — (Drawing Service 프록시, 로그인 불필요 경로이므로 별도 서브도메인으로 분리해 캐시·Rate Limit 정책을 다르게 적용) |

프론트엔드(React SPA)는 S3 정적 호스팅 + CloudFront, API는 API Gateway 뒷단 ALB → ECS.

---

<a id="network"></a>

## 네트워크

`04-system-architecture.md`의 VPC 개요(퍼블릭/프라이빗 서브넷)를 AZ 이중화까지 확정한다.

```
VPC (10.0.0.0/16) — ap-northeast-2

Public Subnet  (10.0.0.0/24, 10.0.1.0/24)   ← AZ-a, AZ-c
  ALB, NAT Gateway ×2(AZ별)

Private Subnet — App (10.0.10.0/24, 10.0.11.0/24)
  ECS Fargate Tasks

Private Subnet — Data (10.0.20.0/24, 10.0.21.0/24)
  RDS, ElastiCache, MSK
```

- NAT Gateway는 AZ별로 하나씩 둔다(단일 NAT는 AZ 장애 시 전체 아웃바운드 차단 — SPOF).
- Security Group은 계층별 화이트리스트: `alb-sg`(0.0.0.0/0:443 인바운드) → `ecs-sg`(alb-sg에서만 인바운드) → `data-sg`(ecs-sg에서만 인바운드). 데이터 계층 SG는 인터넷 인바운드를 절대 허용하지 않는다.
- dev/staging은 비용 절감을 위해 AZ 이중화를 생략하고 단일 AZ로 운영해도 무방하다. prod만 이중화를 강제한다.

---

<a id="ecs"></a>

## 컨테이너 배포 (ECS Fargate)

### Task 정의 전략

서비스별로 독립된 Task Definition·ECR 리포지토리를 가진다(04번 마이크로서비스 목록과 1:1 대응). 공통 사항:

- 베이스 이미지: `eclipse-temurin:21-jre-alpine` (Java 21, Spring Boot 3.x와 일치)
- 헬스체크: 모든 서비스가 `/actuator/health` 노출(Spring Boot Actuator), ALB Target Group과 ECS Task 헬스체크가 동일 엔드포인트 사용
- 로그: `awslogs` 드라이버 → CloudWatch Logs (`/ecs/{service}/{env}`)
- 환경변수: Task Definition에는 비민감 설정만, 민감정보는 [시크릿 관리](#secrets) 참고

### 오토스케일링 (prod)

| 지표 | 임계치 | 동작 |
|---|---|---|
| ALB Target 평균 CPU | 60% 초과 5분 지속 | Task +1 (최대 6) |
| ALB Target 평균 CPU | 30% 미만 10분 지속 | Task -1 (최소 2) |
| Finance Service 전용 | 매월 말일~익월 5일은 최소 Task 수를 3으로 상향(세금계산서 마감 트래픽 대비, `01-prd.md` §6 가용성 요구사항 근거) |

---

<a id="zero-downtime"></a>

## 무중단 배포 전략

세금계산서 마감기간 중단 금지(`01-prd.md`)가 핵심 제약이므로 prod는 아래를 강제한다.

- **ECS Rolling Update**: `minimumHealthyPercent: 100`, `maximumPercent: 200` — 새 Task가 헬스체크를 통과한 뒤에만 이전 Task를 내린다.
- **ALB Deregistration Delay**: 30초 — 진행 중인 요청이 끝날 시간을 준다.
- **배포 동결 기간**: 매월 말일~익월 5일(세금계산서 마감)은 긴급 장애 수정 외 prod 배포를 하지 않는다. CI에서 해당 기간엔 `main` 배포 워크플로에 수동 승인 게이트를 하나 더 추가한다.
- **DB 마이그레이션은 항상 하위 호환**: 컬럼 삭제·타입 변경처럼 이전 버전 애플리케이션 코드가 깨지는 변경은 "확장 → 배포 → 정리(expand-deploy-contract)" 3단계로 나눈다. 한 번의 배포에서 스키마 파괴적 변경과 코드 배포를 동시에 하지 않는다.

---

<a id="migration"></a>

## DB 마이그레이션

### Flyway 채택 이유

- Spring Boot 공식 통합(`spring-boot-starter-data-jpa`와 함께 자동 구성) — Liquibase 대비 설정 오버헤드가 적다
- SQL 파일 기반이라 리뷰가 쉽고, PostgreSQL 스키마 분리(auth/finance/drawings/products/analytics) 구조와 잘 맞는다 — 서비스별 `db/migration` 디렉터리를 독립적으로 관리

### 서비스별 적용 방식

```
services/finance/src/main/resources/db/migration/
  V1__init_schema.sql
  V2__add_receipt_category.sql
  ...
```

- 각 서비스는 자기 스키마의 마이그레이션만 소유한다(서비스 간 마이그레이션 의존 금지 — MSA 격리 원칙 위반).
- **마이그레이션은 애플리케이션 기동 시 자동 실행하지 않는다.** CI/CD 파이프라인의 별도 Job(`migrate`)에서 배포 직전에 실행 — 여러 Task가 동시에 뜨며 각자 마이그레이션을 시도하는 경쟁 상태를 피하기 위함.

---

<a id="pipeline"></a>

## CI/CD 파이프라인

`04-system-architecture.md`의 개요를 단계별로 구체화한다. 저장소는 `kitchen-backend`(모노레포, 서비스별 path filter) / `kitchen-frontend` 2개.

```
[백엔드: kitchen-backend]

on: push
  ├── changed-services 감지 (dorny/paths-filter)
  │
  ├── [Test]  변경된 서비스만 병렬 실행
  │     ├── ./gradlew :services:{svc}:test
  │     └── Testcontainers로 PostgreSQL·Kafka 통합 테스트
  │
  ├── [Build]  테스트 통과한 서비스만
  │     ├── Docker build → ECR push (태그: git sha)
  │     └── Trivy로 이미지 취약점 스캔 (HIGH 이상 발견 시 실패)
  │
  └── [Deploy]
        develop 브랜치 → staging
          ├── migrate job (Flyway, staging DB)
          └── ECS service update (새 태스크 정의 revision)
        main 브랜치 → prod
          ├── 수동 승인 (GitHub Environments — 최소 1인 리뷰)
          ├── migrate job (Flyway, prod DB) — 실패 시 배포 중단
          └── ECS Rolling deploy
```

### 브랜치 전략

| 브랜치 | 용도 |
|---|---|
| `feature/*` | 기능 개발, PR → `develop` |
| `develop` | staging 자동 배포. 통합 확인용 |
| `main` | prod 배포 소스. `develop`에서만 머지(hotfix 예외) |
| `hotfix/*` | 긴급 수정. `main`에서 분기 → `main`·`develop` 양쪽에 머지 |

### 롤백

- **애플리케이션**: ECS는 이전 Task Definition revision으로 즉시 재배포(GitHub Actions에 `rollback` 워크플로 별도 트리거, 직전 성공 sha 기록해둔 것 사용)
- **DB**: Flyway는 자동 다운그레이드를 지원하지 않는다 — 위 [무중단 배포 전략](#zero-downtime)의 하위 호환 원칙을 지켰다면 애플리케이션만 롤백해도 깨지지 않는다. 파괴적 변경이 이미 나간 경우에만 수동 복구(스냅샷 복원)로 대응한다.

---

<a id="secrets"></a>

## 시크릿 관리

| 항목 | 저장 위치 |
|---|---|
| DB 접속정보, JWT 서명 키, 결제 API 키, SES 키 | **AWS Secrets Manager** — ECS Task Definition에서 `secrets` 필드로 참조(컨테이너 환경변수로 주입, 이미지·로그에 평문 노출 없음) |
| Non-secret 설정(피처 플래그, 타임아웃 값 등) | AWS Systems Manager Parameter Store (SecureString 미사용, 비용 절감) |
| GitHub Actions → AWS 인증 | OIDC (Access Key 장기 발급 금지) — 리포지토리별 IAM Role 분리 |

시크릿 로테이션: DB 비밀번호는 Secrets Manager 자동 로테이션(90일) 활성화. JWT 서명 키 로테이션은 `09-security.md`에서 다룬다.

---

<a id="observability"></a>

## 모니터링·로깅·알림

초기 규모(직원 10명 이하 타깃 SaaS)에 맞춰 별도 관측 도구(Datadog 등) 없이 **AWS 네이티브로 시작**하고, 트래픽 증가 시 재검토한다.

| 계층 | 도구 |
|---|---|
| 로그 | CloudWatch Logs (서비스별 로그 그룹), 90일 보관 후 S3 아카이브 |
| 메트릭 | CloudWatch Metrics (ECS/RDS/ALB/MSK 기본 제공 지표) |
| 대시보드 | CloudWatch Dashboard 1개 — 서비스별 CPU·5xx율·응답시간·큐 적체 |
| 알림 채널 | CloudWatch Alarm → SNS → Slack Webhook |
| 분산 추적 | 미도입 (MVP 범위 밖 — 서비스 6개 규모에서는 로그 상관관계 ID로 대체: 모든 요청에 `X-Request-Id` 헤더 전파) |

### 핵심 알람

| 알람 | 조건 | 심각도 |
|---|---|---|
| API 5xx 비율 | 5분간 5% 초과 | Critical |
| ALB 응답시간 p95 | 3초 초과 5분 지속 (PRD 성능 요구사항 근거) | Warning |
| RDS CPU | 80% 초과 10분 지속 | Warning |
| RDS 여유 스토리지 | 20% 미만 | Critical |
| Outbox 미발행 이벤트 적체 | `status = PENDING` 100건 초과 5분 지속 | Warning (Kafka 발행 장애 신호) |
| Export Job 실패율 | 10분간 20% 초과 | Warning |
| ECS Task 재시작 반복 | 5분 내 3회 이상 | Critical |

---

<a id="backup"></a>

## 백업·재해복구

| 대상 | 방식 | 목표 |
|---|---|---|
| RDS (prod) | 자동 스냅샷 매일 + PITR(Point-in-Time Recovery) 7일 | RPO ≤ 5분, RTO ≤ 1시간(Multi-AZ 자동 장애조치) |
| RDS (dev/staging) | 자동 스냅샷 매일, 보관 3일 | RPO ≤ 24시간 |
| S3 (영수증·도면·제품 이미지) | 버저닝 활성화 + 5년 후 Glacier 이전(`01-prd.md` 세법 보관 요구사항) | 오삭제 복구 가능 |
| MSK | 토픽 보관주기 7일 (이벤트는 Outbox+DB가 원본, Kafka는 전달 매개체 — 장기 보관 불필요) | — |
| 설정·인프라 정의 | Terraform state를 S3(버저닝) + DynamoDB(락)로 원격 관리 | 인프라 자체를 코드로 재현 가능 |

**재해복구 시나리오**: 리전 전체 장애는 이번 버전 범위 밖으로 둔다(단일 리전 운영). AZ 장애는 Multi-AZ RDS + 다중 AZ ECS/NAT 구성으로 자동 대응된다.

---

<a id="cost"></a>

## 비용 개요 (대략, 월 기준 — 확정 견적 아님)

| 환경 | 대략 월 비용 | 비고 |
|---|---|---|
| dev | ~$80 | 평일 주간만 기동, RDS/MSK 최소 사양 |
| staging | ~$250 | 상시 기동, 최소 사양 |
| prod (초기, 저트래픽) | ~$700~900 | RDS Multi-AZ·MSK 3브로커가 비중 큼 |

RDS Multi-AZ와 MSK가 prod 비용의 큰 비중을 차지한다 — 고객사 10곳 미만인 극초기 단계에서는 MSK를 단일 브로커로 시작하고 유료 고객 확보 후 3브로커로 올리는 단계적 접근도 검토 가능(단, `04-system-architecture.md`의 MSA·Kafka 채택 이유와는 별개로 순수 비용 최적화 옵션이므로 실제 결정은 초기 고객 확보 속도를 보고 판단한다).

---

<a id="checklist"></a>

## 배포 전 체크리스트 (prod)

- [ ] 마감기간(매월 말일~익월 5일) 배포 동결 기간이 아닌지 확인
- [ ] Flyway 마이그레이션이 하위 호환(expand-deploy-contract)인지 확인
- [ ] `staging`에서 최소 24시간 정상 동작 확인
- [ ] CloudWatch 핵심 알람이 모두 정상(OK) 상태인지 확인
- [ ] 롤백 대상 Task Definition revision·직전 성공 커밋 sha 기록
- [ ] 결제·홈택스 연동 등 외부 API 관련 변경이면 Circuit Breaker fallback 동작 재확인
