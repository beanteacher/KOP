# 09 — 보안 설계

> **문서 상태**: 초안 (Draft)
> **작성일**: 2026-09-11
> **버전**: v0.1
> **연관 문서**: `01-prd.md`(비기능 요구사항) · `03-feature-spec.md`(X-2 권한 매트릭스·X-3 인증) · `05-database-schema.md`(auth 스키마) · `06-api-design.md`(Auth API) · `08-deployment.md`(인프라·시크릿 관리)

`03-feature-spec.md`의 X-2(권한 매트릭스)·X-3(인증 규칙)에서 이미 정한 것을 재정의하지 않고, 구현 수준(토큰을 어디에 어떻게 저장하는지, 테넌트 격리를 어느 계층에서 강제하는지, 암호화 범위)까지 구체화한다.

---

## 목차

- [인증 (Authentication)](#authn)
- [인가 (Authorization) — RBAC & 멀티테넌시](#authz)
- [데이터 보호](#data-protection)
- [파일 접근 보안](#file-security)
- [API 보안](#api-security)
- [도면 공유 링크 — 비로그인 접근](#share-link)
- [감사 로그](#audit)
- [보안 헤더](#headers)
- [인시던트 대응](#incident)
- [배포 전 보안 체크리스트](#checklist)

---

<a id="authn"></a>

## 인증 (Authentication)

### 토큰 구조

| 토큰 | 수명 | 저장 위치 |
|---|---|---|
| Access Token (JWT) | 1시간 | 프론트엔드 메모리(Zustand 스토어) — `localStorage`에 저장하지 않는다 |
| Refresh Token | 30일 | `httpOnly` + `Secure` + `SameSite=Strict` 쿠키 (JS에서 읽을 수 없음) |

```jsonc
// Access Token payload
{
  "sub": "employee-uuid",
  "companyId": "company-uuid",
  "role": "ADMIN",          // ADMIN | EMPLOYEE
  "plan": "PRO",             // 플랜 한도 체크에 사용 (X-5)
  "iat": 1757577600,
  "exp": 1757581200
}
```

### Access Token을 메모리에만 두는 이유

- `localStorage`는 XSS 발생 시 모든 스크립트가 읽을 수 있다 — 탈취되면 토큰 만료 전까지(최대 1시간) 공격자가 그대로 사용 가능
- 메모리 보관은 탭을 새로고침하면 사라지므로, 앱 부팅 시 `httpOnly` Refresh Token 쿠키로 자동 재발급받아 복구한다(`POST /api/auth/refresh`가 요청 시 쿠키를 자동 전송)
- Refresh Token은 `httpOnly`라 XSS로도 훔칠 수 없다 — 탈취 경로는 사실상 서버 로그 유출뿐이므로 만료가 길어도(30일) 허용 가능

### SameSite=Strict로 충분한 이유

프론트엔드(`app.kitchensys.com`)와 API(`api.kitchensys.com`)는 `08-deployment.md`의 도메인 구조상 같은 등록 도메인(`kitchensys.com`)의 서브도메인이라 "same-site"로 취급된다 — `SameSite=Strict` 쿠키도 둘 사이 요청에는 정상 전송되면서, 제3자 사이트발 요청(CSRF)은 차단된다. 별도 CSRF 토큰은 두지 않는다.

### 로그인·잠금

- 이메일 + 비밀번호(bcrypt, cost factor 12)
- 5회 연속 실패 시 10분 계정 잠금(`03-feature-spec.md` X-3) — Redis에 `login_fail:{email}` 카운터, TTL 10분(`04-system-architecture.md` Redis 용도 표와 일치)
- 잠금 카운터는 **이메일 기준**이다. 동일 IP에서 여러 계정을 공격하는 크리덴셜 스터핑은 [API 보안](#api-security)의 IP Rate Limit으로 별도 방어한다.
- 비밀번호 정책: 최소 8자, 영문+숫자 조합 필수 — 애플리케이션 레벨(Zod 스키마·서버 검증) 이중 체크

### 비밀번호 재설정

1. `POST /api/auth/password-reset/request` → 이메일로 재설정 링크(1회용 토큰, 유효 1시간)
2. `POST /api/auth/password-reset/confirm` → 새 비밀번호 설정 시 **기존 모든 세션 무효화**: 해당 직원의 Refresh Token을 Redis 블랙리스트에 일괄 등록(`employee_id` 기준으로 발급 이력 조회 → 전체 등록)
3. 재설정 토큰은 1회 사용 후 즉시 폐기(재사용 방지)

### 로그아웃

`POST /api/auth/logout` → Refresh Token을 Redis 블랙리스트에 등록(TTL = 남은 만료 시간까지만, 그 이후엔 자연 만료라 별도 정리 불필요) + Refresh Token 쿠키 삭제.

---

<a id="authz"></a>

## 인가 (Authorization) — RBAC & 멀티테넌시

### 역할 기반 접근 제어

`03-feature-spec.md` X-2 권한 매트릭스를 그대로 따른다(관리자 `ADMIN` / 직원 `EMPLOYEE` 2단). 이 문서에서 다루는 건 "어디서 강제하는가"다.

| 계층 | 강제 방식 |
|---|---|
| API Gateway | 없음(인증 여부만 확인 — JWT 유효성) |
| 각 서비스 진입점 | Spring Security `@PreAuthorize("hasRole('ADMIN')")` — 컨트롤러 메서드 단위로 권한 매트릭스 그대로 매핑 |
| 프론트엔드 | 메뉴 자체를 숨김(`07-ui-wireframe.md` — "없는 메뉴는 숨김") — **UX용일 뿐 보안 경계가 아니다.** 실제 차단은 반드시 서버에서 한다 |

프론트엔드에서 메뉴를 숨기는 것과 서버가 403을 반환하는 것은 다른 목적이다. 메뉴 숨김이 뚫려도(개발자 도구로 직접 API 호출) 서버 쪽 `@PreAuthorize`가 최종 방어선이다.

### 멀티테넌시 격리 — company_id 스코프

이 시스템에서 가장 중요한 보안 경계는 역할(관리자/직원)이 아니라 **업체(테넌트) 간 데이터 격리**다. 한 업체 직원이 다른 업체의 영수증·세금계산서를 보는 사고는 권한 설정 실수보다 훨씬 심각하다.

- 모든 조회·수정 쿼리는 `WHERE company_id = :companyId`를 **예외 없이** 포함한다.
- `companyId`는 클라이언트가 요청 파라미터로 보내는 값을 신뢰하지 않는다 — 항상 **JWT의 `companyId` claim**에서만 가져온다(요청 바디·쿼리스트링에 `companyId`가 오더라도 무시).
- Spring 쪽 구현: 서비스 공통 베이스 클래스(`TenantScopedRepository` 같은 패턴)로 리포지토리 메서드를 감싸 `company_id` 조건을 자동 주입 — 개발자가 실수로 빠뜨릴 수 없게 만든다(휴먼 에러를 코드 리뷰가 아니라 구조로 막는다).
- 05번 DB 스키마에서 스키마 간 FK를 걸지 않기로 한 것(MSA 격리)과 별개로, **스키마 내부 모든 테이블은 `company_id` 인덱스를 필수로 가진다**(`05-database-schema.md` 인덱스 표와 일치).

### 리소스 소유권 체크 (역할과 별개의 규칙)

권한 매트릭스만으로 못 잡는 규칙들 — 서비스 레이어에서 개별 검증:

| 규칙 | 근거 |
|---|---|
| 직원은 영수증을 본인이 등록한 건만 수정 가능, 그것도 등록 후 24시간 이내만 | `03-feature-spec.md` A-3, 권한 매트릭스 |
| 직원은 영수증을 본인이 등록한 건만 조회(전체 조회는 관리자만) | 권한 매트릭스 |
| 초대 링크는 발급한 관리자와 무관하게 토큰 소유자면 누구나 수락 가능하지만, 토큰은 이메일에 바인딩 — 다른 이메일로 수락 시도 시 거부 | `employee_invitations` 스키마의 email 컬럼 |

### 직원 초대 토큰

- 초대 토큰은 난수(UUID v4) 기반, 유효기간 7일
- 초대 이메일 주소와 수락 시 입력한 이메일이 일치해야 함 — 링크를 다른 사람이 열어도 자기 계정으로 가입 불가
- 수락 즉시 토큰 폐기(1회용)

---

<a id="data-protection"></a>

## 데이터 보호

### 저장 시 암호화(At Rest)

`01-prd.md`의 "세금계산서 데이터 암호화 저장" 요구사항을 아래로 구체화한다.

- **RDS 전체를 SSE-KMS로 암호화**한다(`08-deployment.md`) — 세금계산서를 포함한 모든 테이블이 저장소 레벨에서 암호화된다. 별도 컬럼 레벨 암호화(애플리케이션에서 AES 등으로 재암호화)는 **하지 않는다**: 컬럼 암호화는 `WHERE amount = ...`류 검색·인덱싱을 못 쓰게 만들고, 이 시스템 규모(직원 10명 이하 소규모 업체 대상)에서 얻는 보안 이득 대비 구현·운영 비용이 크다.
- **예외 1** — `password_hash`는 저장소 암호화와 무관하게 원래 원문 비밀번호를 저장하지 않는다(bcrypt 단방향 해시, `05-database-schema.md`).
- **예외 2** — `companies.bank_account_number`(계좌번호)는 위 "컬럼 레벨 암호화는 하지 않는다" 원칙의 예외다. RDS 스토리지 암호화는 디스크·백업 탈취만 막고, DB 접근 권한 자체가 뚫린 경우(SQL 인젝션, 내부자, DB 관리자 계정 탈취)엔 평문이 그대로 노출된다 — 계좌번호는 금전 사고로 직결되는 정보라 이 경로까지 방어하기로 결정했다(2026-09-16). `auth-service`의 `AesEncryptor`(Spring Security Crypto `Encryptors.delux`, AES-256-GCM)로 애플리케이션 레벨 암호화 후 저장하고 조회 시 복호화한다. 이 필드는 검색·인덱싱 대상이 아니므로 컬럼 암호화의 단점(등호 검색 불가)이 문제되지 않는다. `representative_name`·`phone` 등 다른 개인정보성 필드는 기존 원칙(RDS 암호화로 충분)을 그대로 따르며 컬럼 암호화하지 않는다 — 확대 적용은 하지 않기로 했다.
- S3(영수증·제품·도면 파일)도 SSE-KMS 기본 암호화(`08-deployment.md` 파일 저장소 절과 동일 키 관리 정책 사용).

### 전송 중 암호화(In Transit)

- 모든 트래픽 HTTPS 강제 — ALB에서 HTTP(80) 요청은 HTTPS(443)로 301 리다이렉트, HSTS 헤더로 브라우저가 이후 자동으로 HTTPS만 사용하도록 강제(`max-age=31536000; includeSubDomains`)
- 서비스 간 내부 통신(ECS Task 간)은 VPC 프라이빗 서브넷 내부이므로 mTLS까지는 도입하지 않는다(MVP 범위 — 트래픽 증가 시 App Mesh 재검토)

### PII 최소화

| 데이터 | 처리 |
|---|---|
| 사업자번호(`business_registration_number`) | 목록·검색 API는 전체 노출하되, 프론트엔드 UI에서 관리자 외 역할에게는 뒤 4자리만 표시(`214-05-XXXXX`, `07-ui-wireframe.md` 세금계산서 상세 참고) |
| `password_hash` | 어떤 API 응답에도 절대 포함하지 않는다(`GET /api/auth/employees` 등은 DTO에서 필드 자체를 제외 — 직렬화 시 실수로 노출되는 사고를 막기 위해 Entity를 직접 반환하지 않고 항상 별도 Response DTO를 통과시킨다) |
| 고객(거래처) 개인정보 | 사업자 간 거래이므로 원칙적으로 개인정보 아님. 단, 거래처 담당자 연락처가 입력되는 경우 관리자만 조회 가능(권한 매트릭스와 동일 원칙 적용) |

---

<a id="file-security"></a>

## 파일 접근 보안

`04-system-architecture.md` 비동기 파일 처리 파이프라인을 보안 관점에서 보강한다.

- **업로드**: S3 Presigned URL, 유효기간 5분(이미 04에 반영) — 업로드 대상 키(`receipts/{company_id}/...`)는 서버가 생성하며 클라이언트가 경로를 지정할 수 없다(다른 업체 경로에 덮어쓰기 방지)
- **업로드 파일 검증**: 확장자 화이트리스트(`.jpg .jpeg .png .webp .pdf`만 허용), Lambda(ImageProcessor) 단계에서 실제 파일 시그니처(magic bytes) 검사 — 확장자만 바꾼 악성 파일 차단
- **읽기**: CloudFront Signed URL, 유효기간 1시간(이미 04에 반영) — S3 버킷은 퍼블릭 접근 완전 차단(Block Public Access 전체 ON), CloudFront를 통해서만 접근 가능
- **도면 공유 링크로 노출되는 파일**은 별도 규칙 — [도면 공유 링크](#share-link) 참고

---

<a id="api-security"></a>

## API 보안

### Rate Limiting (API Gateway)

| 대상 | 제한 |
|---|---|
| 전체 API, IP 기준 | 분당 100 요청 |
| `/api/auth/login`, IP 기준 | 분당 10 요청 (계정 잠금과 별개로 무차별 대입 자체를 차단) |
| `/api/drawings/shared/{token}`(비로그인 접근) | 분당 30 요청 — 토큰 추측 시도 완화 |

### CORS

허용 오리진은 `08-deployment.md` 도메인 구조의 프론트엔드 도메인만 화이트리스트(`app.kitchensys.com`, `staging.kitchensys.com`, `dev.kitchensys.com`, 로컬 개발 시 `localhost:5173`) — 와일드카드(`*`) 금지.

### 입력 검증 및 인젝션 방지

- SQL Injection: Spring Data JPA의 파라미터 바인딩만 사용, 네이티브 쿼리에서 문자열 결합 금지(코드 리뷰 체크 항목)
- XSS: React가 기본적으로 출력을 이스케이프 — `dangerouslySetInnerHTML` 사용 금지(도면 라벨·메모 등 사용자 입력 텍스트를 그대로 HTML로 렌더링하지 않는다)
- 모든 쓰기 API는 요청 스키마 검증(Bean Validation `@Valid`) 후 비즈니스 로직 진입 — 이미 `06-api-design.md`의 `VALIDATION_ERROR` 패턴과 일치

---

<a id="share-link"></a>

## 도면 공유 링크 — 비로그인 접근

이 시스템에서 유일하게 인증 없이 접근 가능한 리소스(`GET /api/drawings/shared/{token}`, `06-api-design.md`)라 별도로 다룬다.

- 토큰은 UUID v4 — 추측 불가능한 난수
- `expiresIn: "7d" | "30d" | "never"` 옵션(`06-api-design.md`) — 관리자가 링크 생성 시 선택
- 비활성화(`DELETE .../share-links/{linkId}`)는 즉시 반영 — 캐시 없이 매 요청 DB 조회로 유효성 확인(공유 링크는 트래픽이 많지 않으므로 캐시로 인한 "취소했는데 아직 열림" 상황을 만들지 않는 것을 우선한다)
- 응답은 **읽기 전용 데이터만**: 도면 콘텐츠(`content`), 이름, 고객명, 발행일 — 거래처 연락처·금액 같은 다른 모듈 데이터는 포함하지 않는다
- 만료·비활성화된 링크는 `410 GONE`으로 응답(`SHARE_LINK_EXPIRED` / `SHARE_LINK_REVOKED`) — 404가 아니라 410을 쓰는 이유는 "존재한 적 없음"과 "있었지만 끝남"을 클라이언트가 구분해 안내 문구를 다르게 보여줄 수 있게 하기 위함

---

<a id="audit"></a>

## 감사 로그

아래 액션은 `company_id` 스키마별 `audit_logs` 테이블에 기록한다(누가/언제/무엇을, 각 서비스가 자기 스키마에 소유 — MSA 격리 원칙과 동일).

| 액션 | 기록 항목 |
|---|---|
| 로그인 실패 | employee_id(있는 경우) 또는 email, IP, 시각 |
| 세금계산서 취소 | 세금계산서 id, 취소 사유, 처리자, 시각 |
| 직원 삭제·역할 변경 | 대상 직원 id, 변경 전/후 role, 처리자 |
| 플랜 변경 | 이전/이후 플랜, 처리자, 시각 |
| 도면 공유 링크 생성·비활성화 | 도면 id, 링크 id, 처리자, 시각 |

감사 로그는 별도 대시보드 없이 DB 조회로 충분한 MVP 범위 — 필요 시 Analytics 서비스가 Kafka로 구독해 조회 화면을 붙이는 것으로 확장 가능하다(`04-system-architecture.md` Kafka 토픽 목록에 감사용 토픽을 추가하는 형태).

---

<a id="headers"></a>

## 보안 헤더

ALB/API Gateway 응답에 공통 적용:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'; img-src 'self' https://*.cloudfront.net; connect-src 'self' https://api.kitchensys.com
Referrer-Policy: strict-origin-when-cross-origin
```

`X-Frame-Options: DENY`는 프론트엔드 전체에 적용하되, 도면 공유 뷰(`/share/drawings/:token`)처럼 외부에 임베드될 가능성이 있는 페이지가 생기면 그 경로만 예외 처리한다(현재 PRD 범위에는 임베드 요구사항이 없으므로 전체 DENY 유지).

---

<a id="incident"></a>

## 인시던트 대응 (요약)

소규모 팀 운영을 전제로 가볍게 정의한다 — 정식 IR 플레이북은 유료 고객 확보 후 재작성.

| 상황 | 즉시 조치 |
|---|---|
| 특정 계정 탈취 의심 | 해당 직원 Refresh Token 전체 블랙리스트 등록(강제 로그아웃) + 비밀번호 재설정 안내 메일 |
| 비정상 대량 조회(스크래핑 의심) | `08-deployment.md`의 CloudWatch 알람(5xx율·API 호출량)으로 탐지 → 해당 IP를 API Gateway 레벨에서 임시 차단 |
| 데이터 유출 의심 | RDS/S3 접근 로그(CloudTrail) 조회로 범위 확인 → 영향받은 업체에 개별 안내(사업자 간 서비스이므로 개인정보보호법상 개인정보 유출 통지 의무보다는 계약상 고지 의무로 접근) |

---

<a id="checklist"></a>

## 배포 전 보안 체크리스트

- [ ] 모든 신규 엔드포인트에 `@PreAuthorize` 권한 매트릭스 매핑 확인
- [ ] 신규 쿼리가 `company_id` 스코프를 빠뜨리지 않았는지 확인(테넌트 격리)
- [ ] 응답 DTO에 `password_hash` 등 민감 필드가 없는지 확인
- [ ] 신규 S3 업로드 경로가 사용자 입력으로 조작 불가능한지 확인
- [ ] Rate Limit·CORS 화이트리스트에 신규 도메인이 필요하면 추가했는지 확인
- [ ] 민감 액션(취소·삭제·역할변경 등)이 감사 로그에 기록되는지 확인
