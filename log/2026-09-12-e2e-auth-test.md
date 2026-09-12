# E2E 테스트 기록 — Auth 모듈

- **일시**: 2026-09-12
- **대상**: kitchen-frontend + kitchen-backend(auth-service, api-gateway) 연동 Auth 플로우
- **환경**: docker-compose(postgres/redis/kafka) + auth-service(3001) + api-gateway(8080) + frontend dev(5173), 전부 사전 기동 상태에서 진행

## 1. 자동 테스트 — Playwright 회귀 스위트

실행: `~/workspace/typescript_react_frontend_workspace/kitchen-frontend` 에서 `pnpm test:e2e`
대상 파일: `e2e/auth.spec.ts`

| 테스트 | 결과 |
|---|---|
| 업체를 등록하고 그 계정으로 로그인해서 대시보드까지 도달한다 | ✅ PASS |
| 틀린 비밀번호로 로그인하면 에러 메시지가 뜬다 | ✅ PASS |
| 사업자등록번호 형식이 틀리면 제출 전에 클라이언트에서 막는다 | ✅ PASS |
| 로그인 없이 대시보드에 접근하면 로그인 페이지로 리다이렉트된다 | ✅ PASS |

**결과: 4/4 통과** (3.3s)

## 2. 수동 E2E — Playwright MCP (브라우저 직접 조작)

자동 스위트와 별개로 MCP(`mcp__playwright__*`)로 실제 브라우저를 띄워 동일 플로우를 재현·확인.

1. `/register` 이동 → 신규 업체("MCP 수동 테스트 업체", 사업자번호 9081726354) 가입 → `/login`으로 리다이렉트 + "가입이 완료됐어요" 메시지 확인. ✅
2. 방금 가입한 계정으로 로그인 → `/`(대시보드) 이동, "안녕하세요, 수동테스트대표님" / "관리자 · FREE 플랜" 표시 확인. 콘솔 에러 없음. ✅
3. 로그아웃 버튼 클릭 → `/login`으로 정상 리다이렉트. ✅
4. 로그아웃 상태에서 `/` 직접 접근 → `/login`으로 리다이렉트는 정상 동작. 단, 이 과정에서 **버그 발견**(아래 3번 항목). ⚠️

## 3. 발견된 버그 — `/api/auth/refresh` 미인증 시 500 반환

**증상**: refreshToken 쿠키가 없는 상태로 `/api/auth/refresh` 호출 시 401/400이 아닌 **500 Internal Server Error** 반환.

### 재현 방법 1 — API(curl)

```bash
curl -i -X POST http://localhost:8080/api/auth/refresh -H "Content-Type: application/json"
# → HTTP/1.1 500, {"error":{"code":"INTERNAL_ERROR","message":"일시적인 오류가 발생했습니다"}}
```

### 재현 방법 2 — 브라우저에서 직접

사전 조건: kitchen-frontend(`pnpm dev`, 5173) + api-gateway(8080) + auth-service(3001)가 떠 있어야 함.

1. Chrome(또는 아무 브라우저)에서 **시크릿 창**을 새로 연다 — 기존 로그인 쿠키가 없는 상태를 보장하기 위함. (이미 로그인해서 쓰던 창이면 먼저 대시보드 우측 상단 "로그아웃" 버튼을 눌러 로그아웃부터 한다.)
2. `F12` 또는 `Cmd+Option+I`로 개발자 도구를 열고 **Network** 탭으로 이동한다. "Preserve log" 체크박스를 켜두면 리다이렉트되어도 요청이 안 사라져서 보기 편하다.
3. 주소창에 `http://localhost:5173/` 을 입력하고 이동한다 (대시보드 경로 — 로그인 안 된 상태에서 접근).
4. 화면은 정상적으로 `/login`으로 리다이렉트된다 (여기까지는 버그 아님, 정상 동작).
5. Network 탭에서 필터에 `auth`를 입력해 요청 목록을 확인한다. 순서대로 다음 두 요청이 보인다:
   - `GET /api/auth/me` → **401** (정상 — 로그인 안 됐으니 401이 맞음)
   - `POST /api/auth/refresh` → **500** ← 이게 버그. 기대값은 401(또는 400)인데 500이 찍힌다.
6. `refresh` 요청을 클릭 → **Response** 탭을 보면 `{"error":{"code":"INTERNAL_ERROR","message":"일시적인 오류가 발생했습니다"}}` 가 보인다. INTERNAL_ERROR는 원래 예상 못 한 서버 장애용 코드라, "로그인 안 한 정상 상황"에서 뜨면 안 되는 코드다.
7. 같은 시점에 개발자 도구 **Console** 탭에도 아래 두 줄이 에러로 찍혀 있는 걸 확인할 수 있다:
   ```
   Failed to load resource: the server responded with a status of 401 () — /api/auth/me
   Failed to load resource: the server responded with a status of 500 () — /api/auth/refresh
   ```

기대 동작(수정 후 확인 기준): 5번 단계의 `refresh` 요청 상태 코드가 500이 아니라 401(또는 400)로 바뀌고, `error.code`도 `INTERNAL_ERROR`가 아닌 인증 관련 코드(예: `UNAUTHORIZED`)로 바뀌어야 한다.

**원인 (코드 확인 완료)**:
- `auth-service/src/main/java/com/kitchensys/auth/controller/AuthController.java:51-58`의 `refresh()`가 `@CookieValue(REFRESH_COOKIE) String refreshToken`을 **필수(required=true, 기본값)**로 선언 → 쿠키가 없으면 스프링이 `MissingRequestCookieException`을 던짐.
- `common/.../GlobalExceptionHandler.java`에는 `BusinessException`, `MethodArgumentNotValidException`만 개별 처리되어 있고 `MissingRequestCookieException`에 대한 핸들러가 없어 catch-all `Exception` 핸들러(29-33행)로 떨어져 500으로 응답됨.

**영향**: 프론트엔드 최종 사용자 경험 자체는 깨지지 않음(어차피 `/login`으로 리다이렉트됨)이나, 정상적인 "토큰 없음" 상황이 서버 로그·모니터링에는 500(장애)으로 잡혀 알람 오탐/노이즈를 유발할 수 있음. 401 계열로 명확히 구분하는 게 맞음.

### 적용한 수정

`auth-service/src/main/java/com/kitchensys/auth/controller/AuthController.java:53`

```diff
- @CookieValue(REFRESH_COOKIE) String refreshToken, HttpServletResponse response
+ @CookieValue(value = REFRESH_COOKIE, required = false) String refreshToken, HttpServletResponse response
```

`logout` 엔드포인트(62행)에 이미 쓰이던 `required = false` 패턴과 동일하게 맞췄다. 쿠키가 없으면 `refreshToken`이 `null`로 컨트롤러까지 들어오고, `AuthService.refresh(null)` → `RefreshTokenService.resolve(null)`이 Redis에서 못 찾아 `Optional.empty()`를 반환 → 기존에 이미 있던 "세션 만료" 처리 경로(`AuthService.java:112`, `BusinessException(UNAUTHORIZED, "UNAUTHORIZED", "세션이 만료되었습니다...")`)를 그대로 타서 401로 응답한다. `GlobalExceptionHandler`나 다른 파일은 건드리지 않았다 — 기존 정상 흐름에 자연스럽게 합류시키는 최소 변경.

### 수정 검증

- auth-service 재기동 후 `curl -i -X POST http://localhost:8080/api/auth/refresh` → **401**, `{"error":{"code":"UNAUTHORIZED","message":"세션이 만료되었습니다. 다시 로그인해주세요"}}` 로 정상화 확인.
- `pnpm test:e2e` 재실행 → 4/4 통과 (회귀 없음).
- Playwright MCP로 로그아웃 상태에서 `/` 재접근 → Network 탭 `POST /api/auth/refresh`가 500이 아닌 **401**로 확인, 화면은 여전히 `/login`으로 정상 리다이렉트.

## 요약

- 회원가입 → 로그인 → 대시보드 → 로그아웃 → 인증가드 리다이렉트, 핵심 플로우 전부 정상 동작 확인 (자동 4/4 통과 + 수동 재확인).
- `/api/auth/refresh`의 예외 처리 버그(401 대신 500)를 발견해 수정 완료. 수정 후 자동/수동 테스트 모두 재검증함.
