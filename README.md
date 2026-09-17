      # Kotlin Backend V1 API 분석

분석 대상: [team-native/Kotlin-Backend-V1](https://github.com/team-native/Kotlin-Backend-V1) · 기준 커밋 [`81345d5`](https://github.com/team-native/Kotlin-Backend-V1/commit/81345d5)

## 1. API마다 메서드, path, 요청/응답 Body, 헤더값

### 공통 헤더

| 구분 | 헤더 | 필수 여부 | 설명 |
| --- | --- | --- | --- |
| 요청 | `Accept: application/json` | 선택 | JSON 응답을 요청한다. |
| 요청 | `X-Trace-Id: <문자열>` | 선택 | 보내면 서버가 같은 값을 추적 ID로 사용하며, 없으면 UUID를 생성한다. |
| 요청 | `Content-Type: application/json` | 로그인만 필수 | 로그인 요청 Body의 JSON 형식을 지정한다. |
| 요청 | `Authorization: Basic <base64(username:password)>` | private API만 필수 | HTTP Basic 인증 정보다. |
| 응답 | `X-Trace-Id` | 정상 처리 및 Controller 예외 응답 | 요청 추적 ID다. |
| 응답 | `Content-Type: application/json` | JSON 응답 | 응답 Body 형식이다. |
| 응답 | `WWW-Authenticate: Basic realm=\"Realm\", charset=\"UTF-8\"` | 401 응답 | Basic 인증이 필요함을 알린다. |

Spring Security가 공통으로 추가하는 응답 헤더:

```http
X-Content-Type-Options: nosniff
X-XSS-Protection: 0
X-Frame-Options: DENY
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
```

인증되지 않은 private 요청은 Security 계층에서 먼저 종료되므로 실제 검증에서 응답 Body와 `X-Trace-Id`가 없었다.

### 1-1. 서버 상태 확인

- Method: `GET`
- Path: `/` 또는 `/health`
- 인증: 없음
- 요청 Body: 없음
- 요청 헤더: 공통 헤더만 사용

성공 응답 Body:

```json
{
  \"status\": \"ok\"
}
```

### 1-2. 공개 Ping

- Method: `GET`
- Path: `/api/v1/public/ping`
- 인증: 없음
- 요청 Body: 없음
- 요청 헤더: 공통 헤더만 사용

성공 응답 Body:

```json
{
  \"message\": \"public pong\"
}
```

### 1-3. 비공개 Ping

- Method: `GET`
- Path: `/api/v1/private/ping`
- 인증: HTTP Basic
- 요청 Body: 없음
- 필수 요청 헤더:

```http
Authorization: Basic <base64(username:password)>
```

인증 성공 시 코드상 응답 Body:

```json
{
  \"message\": \"private pong\"
}
```

인증 누락 또는 실패 시 Body는 없다.

> 현재 템플릿은 자동 생성 사용자의 평문 비밀번호와 `BCryptPasswordEncoder`가 충돌한다. 실제 실행에서는 생성된 비밀번호를 사용해도 `Encoded password does not look like BCrypt` 경고와 함께 401이 반환되었다. 실제 인증 성공을 구현하려면 `UserDetailsService`에 BCrypt로 인코딩한 비밀번호를 저장하거나 인증 Provider를 명시해야 한다.

### 1-4. 로그인

- Method: `POST`
- Path: `/api/v1/auth/login`
- 인증: 없음
- 필수 요청 헤더:

```http
Content-Type: application/json
```

요청 Body:

```json
{
  \"username\": \"user1\",
  \"password\": \"password123\"
}
```

성공 응답 Body:

```json
{
  \"accessToken\": \"replace-with-jwt-token\",
  \"tokenType\": \"Bearer\",
  \"username\": \"user1\"
}
```

`accessToken`은 실제 JWT가 아니라 고정된 예시 문자열이다. 현재 코드는 비밀번호 일치 여부를 검사하지 않고, 두 필드가 비어 있지 않으면 성공 응답을 반환한다.

## 2. API 케이스별 status와 예시 응답 Body

### 2-1. 서버 상태 확인: `GET /`, `GET /health`

| 케이스 | Status | 응답 Body |
| --- | --- | --- |
| 정상 GET | 200 OK | `{\"status\":\"ok\"}` |
| 지원하지 않는 HTTP 메서드 | 현재 구현상 500 Internal Server Error | 공통 `INTERNAL_SERVER_ERROR` Body |

### 2-2. 공개 Ping: `GET /api/v1/public/ping`

| 케이스 | Status | 응답 Body |
| --- | --- | --- |
| 정상 GET | 200 OK | `{\"message\":\"public pong\"}` |
| 지원하지 않는 HTTP 메서드 | 현재 구현상 500 Internal Server Error | 공통 `INTERNAL_SERVER_ERROR` Body |

### 2-3. 비공개 Ping: `GET /api/v1/private/ping`

| 케이스 | Status | 응답 Body |
| --- | --- | --- |
| Authorization 헤더 없음 | 401 Unauthorized | 없음 |
| 잘못된 Basic 인증 정보 | 401 Unauthorized | 없음 |
| 인증 성공 | 코드상 200 OK | `{\"message\":\"private pong\"}` |
| 템플릿의 자동 생성 계정 사용 | 실제 검증 결과 401 Unauthorized | 없음 |

401 주요 헤더:

```http
WWW-Authenticate: Basic realm=\"Realm\", charset=\"UTF-8\"
```

### 2-4. 로그인: `POST /api/v1/auth/login`

| 케이스 | Status | 응답 Body |
| --- | --- | --- |
| username/password 모두 non-blank | 200 OK | `LoginResponse` |
| username가 빈 문자열 또는 공백 | 400 Bad Request | `VALIDATION_ERROR` / `Username is required.` |
| password가 빈 문자열 또는 공백 | 400 Bad Request | `VALIDATION_ERROR` / `Password is required.` |
| 두 필드가 모두 잘못됨 | 400 Bad Request | 첫 번째 field error 한 건만 반환 |
| JSON 문법 오류 | 현재 구현상 500 Internal Server Error | 공통 `INTERNAL_SERVER_ERROR` Body |
| `Content-Type`이 JSON이 아님 | 현재 구현상 500 Internal Server Error | 공통 `INTERNAL_SERVER_ERROR` Body |
| GET 등 잘못된 HTTP 메서드 | 현재 구현상 500 Internal Server Error | 공통 `INTERNAL_SERVER_ERROR` Body |

성공 예시:

```json
{
  \"accessToken\": \"replace-with-jwt-token\",
  \"tokenType\": \"Bearer\",
  \"username\": \"user1\"
}
```

username 검증 실패 예시:

```json
{
  \"code\": \"VALIDATION_ERROR\",
  \"message\": \"Username is required.\",
  \"timestamp\": \"<ISO-8601 UTC 시각>\"
}
```

password 검증 실패 예시:

```json
{
  \"code\": \"VALIDATION_ERROR\",
  \"message\": \"Password is required.\",
  \"timestamp\": \"<ISO-8601 UTC 시각>\"
}
```

JSON 문법 오류, 잘못된 Content-Type, 잘못된 HTTP 메서드의 현재 응답 예시:

```json
{
  \"code\": \"INTERNAL_SERVER_ERROR\",
  \"message\": \"Unexpected server error.\",
  \"timestamp\": \"<ISO-8601 UTC 시각>\"
}
```

일반적으로 JSON 파싱 오류는 400, 미지원 Content-Type은 415, 미지원 메서드는 405가 적절하다. 하지만 현재 `GlobalExceptionHandler`가 검증 오류 이외의 모든 `Exception`을 500으로 처리한다.

## 3. API 진행 흐름

### 3-1. 전체 요청 흐름

```text
클라이언트 요청
→ Spring Security가 공개/인증 필요 경로 판단
→ HTTP 로깅 필터가 요청·응답을 caching wrapper로 감싸고 Trace ID 준비
→ DispatcherServlet이 path와 HTTP method에 맞는 Controller 탐색
→ Jackson이 JSON Body를 Kotlin DTO로 변환
→ @Valid가 Bean Validation 수행
→ Controller 메서드 실행
→ Kotlin 객체를 Jackson이 JSON으로 변환
→ 로깅 필터가 status·처리 시간·헤더·Body를 기록하고 응답 반환
```

인증이 필요한 요청에서 인증에 실패하면 Security 계층에서 401을 반환하며 Controller는 실행되지 않는다.

### 3-2. 상태 확인 API

```text
GET / 또는 GET /health
→ SecurityConfig의 permitAll
→ HealthController.health()
→ mapOf(\"status\" to \"ok\")
→ Jackson JSON 직렬화
→ 200 OK
```

### 3-3. 공개 Ping API

```text
GET /api/v1/public/ping
→ GET /api/v1/public/** permitAll
→ SampleController.publicPing()
→ mapOf(\"message\" to \"public pong\")
→ 200 OK
```

### 3-4. 비공개 Ping API

```text
GET /api/v1/private/ping
→ anyRequest().authenticated()
→ Authorization Basic 검증
ㄴ> 실패: Controller를 실행하지 않고 401
ㄴ> 성공: SampleController.privatePing()
          → {\"message\":\"private pong\"}
          → 200 OK
```

### 3-5. 로그인 API

```text
POST /api/v1/auth/login
→ /api/v1/auth/** permitAll
→ JSON을 LoginRequest(username, password)로 역직렬화
→ @Valid와 @NotBlank 검증
ㄴ> 실패: MethodArgumentNotValidException
|        → GlobalExceptionHandler
|        → 400 VALIDATION_ERROR
ㄴ> 성공: AuthController.login()
          → 고정 accessToken과 username으로 LoginResponse 생성
          → 200 OK
```

로그 로직은 요청의 `password`와 응답의 `accessToken`, `tokenType`처럼 key에 민감 단어가 포함된 값을 `***`로 마스킹한다.

## 4. 템플릿 내 추가 지원 기능

| 기능 | 지원 내용 |
| --- | --- |
| Spring Security | Stateless 세션, CSRF 비활성화, HTTP Basic, 공개/인증 경로 분리 |
| 비밀번호 인코딩 | `BCryptPasswordEncoder` Bean 제공 |
| 요청 검증 | `@Valid`, `@NotBlank`로 로그인 필드 검증 |
| 공통 예외 처리 | `@RestControllerAdvice`와 `ErrorResponse(code, message, timestamp)` |
| Trace ID | 요청의 `X-Trace-Id` 재사용 또는 UUID 생성 후 응답에 포함 |
| HTTP 로깅 | method, path, query, status, duration, client IP, 일부 헤더와 Body 기록 |
| 민감정보 보호 | password/token/secret/cookie/authorization 관련 값 마스킹 |
| 로그 크기 제한 | 요청·응답 Body를 최대 2,000자까지 기록 |
| 서버 수명주기 로그 | 서버 시작/종료 시 애플리케이션명, 포트, profile 기록 |
| Swagger/OpenAPI | `/swagger-ui.html`, `/swagger-ui/index.html`, `/v3/api-docs` |
| Actuator | `/actuator/health`와 `/actuator/info` 노출 설정 |
| 테스트 기본 구조 | Context 로딩, public 200, private 401 MockMvc 테스트 |
| CI | PR 및 main push 시 Java 21 환경에서 `./gradlew test` 실행 |
| Dependabot | Gradle과 GitHub Actions 의존성을 매주 확인 |
| 협업 템플릿 | PR 체크리스트, Bug/Feature Issue 템플릿 |
| GitHub Template | 새 프로젝트에서 템플릿 저장소로 복제 가능 |

### 지원 엔드포인트

| Method | Path | 인증/동작 | Status |
| --- | --- | --- | --- |
| GET | `/actuator/health` | 인증 없이 서버 상태 반환 | 200 |
| GET | `/actuator/info` | 노출은 되어 있지만 Security상 인증 필요 | 인증 없으면 401 |
| GET | `/swagger-ui.html` | Swagger UI로 이동 | 302 |
| GET | `/swagger-ui/index.html` | Swagger UI 화면 | 200 |
| GET | `/v3/api-docs` | OpenAPI JSON 반환 | 200 |

Swagger 설정에는 HTTP Basic 보안 스키마가 전역으로 등록되어 있어 문서상 public API도 인증이 필요한 것처럼 보일 수 있지만, 실제 접근 제어 기준은 `SecurityConfig`이다.
