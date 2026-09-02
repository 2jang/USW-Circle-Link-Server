# 동구라미 API 명세서

> 대상: USW-Circle-Link-Server (수원대 동아리 플랫폼 "동구라미")
> 기준: 이 저장소의 소스 코드 (`src/main/java/com/USWCicrcleLink/server`, `src/main/resources/yaml`)
> 범위: REST 엔드포인트 **82개** (활성 80 · 비활성 2)

---

## 1. 개요

### 1.1 서버 정보

| 항목 | 값 |
|---|---|
| 운영 Base URL | `https://api.donggurami.net` (OpenApiConfig 서버 목록) |
| 로컬 Base URL | `http://localhost:8080` |
| API 문서 | Swagger UI `/` · ReDoc `/docs` |
| 헬스체크 | `GET /health-check` |
| 공통 응답 래퍼 | `ApiResponse{message, data}` |
| 인증 | JWT Bearer (액세스 30분) + refreshToken 쿠키 (Redis, 7일) |

### 1.2 클라이언트 유형과 역할

| 클라이언트 | 역할 | 로그인 | 주요 경로 |
|---|---|---|---|
| 모바일 앱 (학생) | `ROLE_USER` | `POST /users/login` | `/users`, `/mypages`, `/profiles`, `/apply`, `/clubs`, `/my-notices` |
| 웹 (동아리 회장) | `ROLE_LEADER` | `POST /club-leader/login` | `/club-leader/**` |
| 웹 (동아리연합회) | `ROLE_ADMIN` | `POST /admin/login` | `/admin/**`, `/notices` |

로그아웃·토큰 재발급은 3개 역할 공통으로 `/integration/**`을 사용합니다.

### 1.3 컨트롤러별 엔드포인트 수

| 컨트롤러 | 경로 | 개수 |
|---|---|---:|
| UserController | `/users` | 15 |
| IntegrationAuthController | `/integration` | 2 |
| MypageController | `/mypages` | 3 |
| ProfileController | `profiles` | 3 |
| MyNoticeController | `/my-notices` | 2 |
| EventVerificationController | `/users/event` | 2 |
| AplictController | `/apply` | 3 |
| ClubController | `/clubs` | 7 |
| ClubLeaderLoginController | `/club-leader` | 1 |
| ClubLeaderController | `/club-leader` | 24 |
| AdminLoginController | `/admin` | 1 |
| AdminClubController | `/admin/clubs` | 6 |
| AdminClubCategoryController | `/admin/clubs/category` | 3 |
| AdminFloorPhotoController | `/admin/floor/photo` | 3 |
| AdminNoticeController | `/notices` | 5 |
| HealthCheckController | `/health-check` | 1 |
| DocsController | `/docs` | 1 |
| **합계** | | **82** |

---

## 2. 공통 스펙

### A-1. 공통 응답 포맷

#### ApiResponse (성공 응답 래퍼)
`/src/main/java/com/USWCicrcleLink/server/global/response/ApiResponse.java`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 처리 결과 메시지 (한국어) |
| data | T | 응답 데이터. `@JsonInclude(NON_NULL)` — **null이면 JSON에서 아예 생략됨** |

```json
{ "message": "동아리 리스트 조회 성공", "data": { } }
```
```json
{ "message": "동아리 생성 성공" }
```
> `new ApiResponse<>("메시지")` 단일 인자 생성자를 쓴 엔드포인트는 `data` 키가 응답에 존재하지 않습니다.

#### ErrorResponse (에러 응답)
`/src/main/java/com/USWCicrcleLink/server/global/response/ErrorResponse.java`

| 필드 | 타입 | 설명 |
|---|---|---|
| exception | String | 예외 클래스 SimpleName (예: `ClubException`, `AdminException`) |
| code | String | 에러 코드 (`ExceptionType.code` 또는 핸들러 고정 코드) |
| message | String | 에러 메시지 |
| status | Integer | HTTP 상태 코드 숫자 |
| error | String | HTTP 상태 Reason Phrase (예: `Bad Request`) |
| additionalData | Object | 부가 데이터. 검증 실패 시 `{필드명: 메시지}` 맵, 그 외 대부분 null |

```json
{
  "exception": "AdminException",
  "code": "ADM-202",
  "message": "관리자 비밀번호가 일치하지 않습니다.",
  "status": 400,
  "error": "Bad Request",
  "additionalData": null
}
```

Bean Validation 실패 시(`MethodArgumentNotValidException`):
```json
{
  "exception": "MethodArgumentNotValidException",
  "code": "INVALID_ARGUMENT",
  "message": "입력 값 검증에 실패했습니다.",
  "status": 400,
  "error": "Bad Request",
  "additionalData": { "clubName": "동아리명은 최대 10자까지 입력 가능합니다." }
}
```

#### PageResponse
`/src/main/java/com/USWCicrcleLink/server/global/response/PageResponse.java`

| 필드 | 타입 | 설명 |
|---|---|---|
| content | List&lt;T&gt; | 페이지 내용 |
| page | int | 현재 페이지 번호 |
| size | int | 페이지 크기 |
| totalElements | long | 전체 건수 |
| totalPages | int | 전체 페이지 수 |

> **주의**: 관리자 API(동아리 목록·공지 목록)는 `PageResponse`를 쓰지 않고 전용 DTO
> (`AdminClubPageListResponse`, `AdminNoticePageListResponse`)를 사용합니다.
> 필드 구성이 다릅니다: `content / totalPages / totalElements / currentPage` (size 없음, page 대신 currentPage).

---

### A-2. 인증 방식

#### Bearer 헤더
`/src/main/java/com/USWCicrcleLink/server/global/security/jwt/JwtProvider.java`

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9....
```
- 헤더명 상수: `Authorization`, 접두사 `"Bearer "` (공백 포함)
- 서명 알고리즘: HS256 (`jwt.secret.key` 기반 HMAC)
- 로그인 성공 시 서버가 응답 헤더 `Authorization: Bearer <accessToken>` 도 함께 내려줌 (CORS `exposedHeader`에 `Authorization` 포함)

#### accessToken
| 항목 | 값 |
|---|---|
| 유효기간 | `ACCESS_TOKEN_EXPIRATION_TIME = 1800000L` → **30분** |
| subject(sub) | 사용자/관리자/회장의 UUID |
| 추가 claim | `clubUUID` — USER/LEADER인 경우만 포함, ADMIN은 없음 |

#### refreshToken (쿠키)
| 항목 | 값 |
|---|---|
| 쿠키 이름 | `refreshToken` |
| 값 | 랜덤 UUID 문자열 (JWT 아님) |
| 저장소 | Redis 키 `refreshToken:{값}` → value = 사용자 UUID |
| 유효기간 | `REFRESH_TOKEN_EXPIRATION_TIME = 604800000L` → **7일** |
| Path | `/` |
| HttpOnly | 예 |
| Max-Age | 604800 (초) |
| SameSite | prod: `None` / local: `Lax` (프로퍼티 `security.cookie.same-site`, 미설정 시 기본 `Lax`) |
| Secure | prod: `true` / local: `false` (프로퍼티 `security.cookie.secure`, 기본 `true`) |

실제 Set-Cookie 포맷:
```
refreshToken={값}; Path=/; HttpOnly; Max-Age=604800; SameSite=None; Secure
```
로그아웃/삭제 시:
```
refreshToken=; Path=/; HttpOnly; Max-Age=0; Expires=Thu, 01 Jan 1970 00:00:00 GMT; SameSite=None; Secure
```

#### 재발급 흐름
`/src/main/java/com/USWCicrcleLink/server/global/security/Integration/api/IntegrationAuthController.java`

- `POST /integration/refresh-token` (permitAll, 쿠키 필요)
  1. 쿠키에서 `refreshToken` 추출 → 없으면 로그아웃 처리 후 **401** + `{"message":"리프레시 토큰이 유효하지 않습니다. 로그아웃됐습니다."}`
  2. Redis에 `refreshToken:{값}` 존재 여부 검증 → 없으면 로그아웃 처리 후 401
  3. 기존 refreshToken 삭제(로테이션) → 새 accessToken + 새 refreshToken 발급, 쿠키 재설정
  4. 200 + `{"message":"새로운 엑세스 토큰과 리프레시 토큰이 발급됐습니다. 로그인됐습니다.","data":{"accessToken":"...","refreshToken":"..."}}`
- `POST /integration/logout` (permitAll) — Redis refreshToken 삭제, USER면 FCM 토큰 null 처리, 쿠키 만료. `{"message":"로그아웃 성공"}`

#### 토큰 검증 실패 응답 (JwtFilter → CustomAuthenticationEntryPoint)
ApiResponse/ErrorResponse가 아닌 **별도 포맷**입니다.
```json
{ "status": 401, "errorCode": "TOKEN_EXPIRED", "message": "토큰이 만료되었습니다." }
```
| errorCode | 상황 |
|---|---|
| `TOKEN_EXPIRED` | accessToken 만료 |
| `INVALID_TOKEN` | 서명 불일치/변조/형식 오류/토큰 없음 → message는 "인증이 필요합니다." |
| `AUTH_REQUIRED` | 그 외 인증 예외 → "인증이 필요합니다." |

#### 3개 역할
`/src/main/java/com/USWCicrcleLink/server/global/security/jwt/domain/Role.java`

| Role enum | 권한 문자열 | 대상 | 로그인 엔드포인트 |
|---|---|---|---|
| `USER` | `ROLE_USER` | 앱 사용자(학생) | `POST /users/login` |
| `ADMIN` | `ROLE_ADMIN` | 운영팀(웹) | `POST /admin/login` |
| `LEADER` | `ROLE_LEADER` | 동아리 회장(웹) | `POST /club-leader/login` |

권한 문자열은 `"ROLE_" + Role.name()` 으로 생성되며, SecurityConfig의 `hasRole("ADMIN")`은 `ROLE_ADMIN`을 요구합니다.

---

### A-3. 에러 코드 전체 표

`/src/main/java/com/USWCicrcleLink/server/global/exception/ExceptionType.java` 의 모든 enum

#### COM — 공통
| 코드 | HTTP | 메시지 |
|---|---|---|
| COM-501 | 500 | 서버 오류입니다. 관리자에게 문의해주세요. |
| COM-302 | 400 | 잘못된 입력 값입니다. |

#### EMAIL_TOKEN — 이메일 토큰
| 코드 | HTTP | 메시지 |
|---|---|---|
| EMAIL_TOKEN-001 | 400 | 해당 토큰이 존재하지 않습니다. |
| EMAIL_TOKEN-002 | 400 | 토큰이 만료되었습니다. 다시 이메일인증 해주세요 |
| EMAIL_TOKEN-003 | 500 | 이메일 토큰 생성중 오류가 발생했습니다. |
| EMAIL_TOKEN-004 | 500 | 이메일 토큰의 필드 업데이트 후, 저장하는 과정에서 오류가 발생했습니다. |
| EMAIL_TOKEN-005 | 400 | 인증이 완료되지 않은 이메일 토큰입니다. |

#### USR — 사용자
| 코드 | HTTP | 메시지 |
|---|---|---|
| USR-201 | 400 | 사용자가 존재하지 않습니다. |
| USR-202 | 400 | 두 비밀번호가 일치하지 않습니다. |
| USR-203 | 400 | 비밀번호 값이 빈칸입니다 |
| USR-204 | 400 | 현재 비밀번호와 일치하지 않습니다 |
| USR-205 | 500 | 비밀번호 업데이트에 실패했습니다 |
| USR-206 | 409 | 이미 존재하는 회원입니다. |
| USR-207 | 409 | 계정이 중복됩니다. |
| USR-209 | 400 | 올바르지 않은 이메일 혹은 아이디입니다. |
| USR-210 | 400 | 회원의 uuid를 찾을 수 없습니다. |
| USR-211 | 401 | 아이디 혹은 비밀번호가 일치하지 않습니다 |
| USR-214 | 400 | 영문자,숫자,특수문자는 적어도 1개 이상씩 포함되어야합니다 |
| USR-216 | 401 | 비회원 사용자입니다.인증을 완료해주세요 |
| USR-217 | 400 | 현재 비밀번호와 같은 비밀번호로 변경할 수 없습니다. |
| USR-218 | 500 | 회원 생성중 오류 발생 |
| USR-219 | 401 | 요청 받은 SIGNUPUUID가 일치하지 않습니다 |
| USR-220 | 401 | 제3자의 로그인 요청 시도 입니다 |

#### EVT — 이벤트 인증
| 코드 | HTTP | 메시지 |
|---|---|---|
| EVT-101 | 400 | 유효하지 않은 이벤트 코드입니다. |
| EVT-102 | 400 | 이미 인증된 사용자입니다. |

#### CMEM-TEMP / CMEM-ACST — 임시 회원·계정 상태
| 코드 | HTTP | 메시지 |
|---|---|---|
| CMEM-TEMP-301 | 500 | 기존 회원가입 사용자 생성에 실패했습니다 |
| CMEM-TEMP-302 | 400 | CLUBMEMBERTEMP 테이블에 존재하는 이메일 입니다. |
| CMEM-TEMP-303 | 400 | CLUBMEMBERTEMP 에 중복된 프로필이 존재합니다 |
| CMEM-ACST-301 | 500 | AccountStatus 객체 생성 과정중 오류가 발생했습니다 |
| CMEM-ACST-302 | 500 | AccountStatus 객체 저장 과정중 오류가 발생했습니다 |
| CMEM-ACST-303 | 500 | 사용자가 요청한 개수와 실제 요청된 개수가 다릅니다 |
| CMEM-ACST-304 | 500 | 사용자가 요청한 동아리와 실제 요청값이 다르게 생성되었습니다 |

#### TOK — 보안/토큰
| 코드 | HTTP | 메시지 |
|---|---|---|
| TOK-201 | 400 | 유효하지 않은 role입니다. |
| TOK-202 | 401 | 유효하지 않은 토큰입니다. |
| TOK-204 | 401 | 인증되지 않은 사용자입니다. |

#### CLUB — 동아리
| 코드 | HTTP | 메시지 |
|---|---|---|
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. |
| CLUB-202 | 500 | 동아리 조회 중 오류가 발생했습니다. |
| CLUB-203 | 409 | 이미 존재하는 동아리 이름입니다. |
| CLUB-204 | 404 | 동아리 사진이 존재하지 않습니다 |
| CLUB-205 | 409 | 이미 지정된 동아리방입니다. |

#### CTG / CG — 카테고리
| 코드 | HTTP | 메시지 |
|---|---|---|
| CTG-201 | 404 | 존재하지 않는 카테고리입니다. |
| CG-202 | 413 | 카테고리는 최대 3개까지 선택할수 있습니다. |
| CTG-203 | 409 | 이미 존재하는 카테고리입니다. |

#### CINT — 동아리 소개
| 코드 | HTTP | 메시지 |
|---|---|---|
| CINT-201 | 404 | 해당 동아리 소개글이 존재하지 않습니다. |
| CINT-202 | 400 | 구글 폼 URL이 존재하지 않습니다. |
| CINT-303 | 400 | 모집 상태가 올바르지 않습니다. |

#### ADM — 운영팀
| 코드 | HTTP | 메시지 |
|---|---|---|
| ADM-201 | 404 | 해당 계정이 존재하지 않습니다. |
| ADM-202 | 400 | 관리자 비밀번호가 일치하지 않습니다. |

#### NOT — 공지사항
| 코드 | HTTP | 메시지 |
|---|---|---|
| NOT-201 | 404 | 공지사항이 존재하지 않습니다. |
| NOT-202 | 413 | 최대 5개의 사진이 업로드 가능합니다. |
| NOT-204 | 404 | 사진이 존재하지 않습니다. |
| NOT-205 | 400 | 사진 순서는 1에서 5 사이여야 합니다. |
| NOT-206 | 400 | 공지사항 조회 중 에러가 발생했습니다. |

#### PFL — 프로필
| 코드 | HTTP | 메시지 |
|---|---|---|
| PFL-201 | 404 | 프로필이 존재하지 않습니다. |
| PFL-202 | 500 | 프로필 업데이트에 실패했습니다. |
| PFL-203 | 400 | 프로필 입력값은 필수입니다. |
| PFL-204 | 400 | 이미 존재하는 회원입니다. |
| PFL-205 | 400 | 학과 정보는 필수 입력 항목입니다. |
| PFL-206 | 400 | 비회원만 수정할 수 있습니다. |
| PFL-207 | 400 | 프로필이 이미 존재합니다 |
| PFL-208 | 400 | 유효하지 않은 회원 종류입니다. |
| PFL-209 | 400 | 프로필 값이 일치하지 않습니다. |
| PFL-210 | 500 | 프로필 생성중 오류 발생 |

#### CLP — 동아리 사진
| 코드 | HTTP | 메시지 |
|---|---|---|
| CLP-201 | 400 | 범위를 벗어난 사진 순서 값입니다. |
| CLP-202 | 500 | 동아리 ID가 존재하지 않습니다. |

#### CLDR — 동아리 회장
| 코드 | HTTP | 메시지 |
|---|---|---|
| CLDR-101 | 403 | 동아리 접근 권한이 없습니다. |
| CLDR-201 | 400 | 동아리 회장이 존재하지 않습니다. |
| CLDR-202 | 400 | 동아리 회장 비밀번호가 일치하지 않습니다 |
| CLDR-203 | 422 | 이미 존재하는 동아리 회장 계정입니다. |
| CLDR-204 | 400 | 동아리 회장의 이름은 필수 입력 항목입니다. |

#### CMEM / CMEMT — 동아리 회원
| 코드 | HTTP | 메시지 |
|---|---|---|
| CMEM-201 | 404 | 동아리 회원이 존재하지 않습니다. |
| CMEM-202 | 400 | 동아리 회원이 이미 존재합니다. |
| CMEMT-201 | 404 | 회원 가입 요청이 존재하지 않습니다. |

#### APT — 지원서
| 코드 | HTTP | 메시지 |
|---|---|---|
| APT-201 | 404 | 지원서가 존재하지 않습니다. |
| APT-202 | 404 | 유효한 지원자가 존재하지 않습니다. |
| APT-203 | 404 | 유효한 추합 대상자가 존재하지 않습니다. |
| APT-204 | 400 | 선택한 지원자 수와 전체 지원자 수가 일치하지 않습니다. |
| APT-205 | 400 | 이미 지원한 동아리입니다. |
| APT-206 | 400 | 이미 해당 동아리 회원입니다. |
| APT-207 | 400 | 이미 등록된 전화번호입니다. |
| APT-208 | 400 | 이미 등록된 학번입니다. |

#### AC / WT — 인증코드·탈퇴 토큰
| 코드 | HTTP | 메시지 |
|---|---|---|
| AC-101 | 400 | 인증번호가 일치하지 않습니다 |
| AC-102 | 400 | 인증 코드 토큰이 존재하지 않습니다 |
| WT-101 | 400 | 인증번호가 일치하지 않습니다 |
| WT-102 | 400 | 탈퇴 토큰이 존재하지 않습니다 |

#### EML / UUID / ATTEMPT / PHOTO / ENUM — 공통 인프라
| 코드 | HTTP | 메시지 |
|---|---|---|
| EML-501 | 500 | 메일 전송에 실패했습니다. |
| UUID-502 | 400 | 유효하지 않은 UUID 형식입니다. |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 |
| PHOTO-504 | 400 | 사진 파일이 비어있습니다. |
| PHOTO-505 | 404 | 해당 사진이 존재하지 않습니다. |
| ENUM-401 | 400 | 유효하지 않은 Enum 값입니다. |

#### FILE — 파일 I/O
| 코드 | HTTP | 메시지 |
|---|---|---|
| FILE-301 | 400 | 파일 이름 인코딩에 실패했습니다. |
| FILE-302 | 500 | 파일 생성에 실패했습니다. |
| FILE-303 | 400 | 사진 또는 순서 정보가 제공되지 않았습니다. |
| FILE-304 | 400 | 사진의 개수와 순서 정보의 개수가 일치하지 않습니다. |
| FILE-305 | 500 | 파일 저장에 실패했습니다. |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. |
| FILE-307 | 400 | 파일 삭제에 실패했습니다. |
| FILE-308 | 400 | 업로드 가능한 갯수를 초과했습니다. |
| FILE-309 | 400 | 파일 이름이 유효하지 않습니다. |
| FILE-310 | 400 | 파일 확장자가 없습니다. |
| FILE-311 | 400 | 지원하지 않는 파일 확장자입니다. |
| FILE-312 | 400 | 파일 유효성 검사 실패 |

#### GlobalExceptionHandler 고정 코드 (ExceptionType enum이 아님)
`/src/main/java/com/USWCicrcleLink/server/global/exception/GlobalExceptionHandler.java`

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | `@Validated` 검증 실패 (additionalData에 필드별 메시지) |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | 경로변수 타입 변환 실패 (UUID/Enum 모두 해당) |
| INVALID_REQUEST_BODY | 400 | 유효하지 않은 JSON 혹은 ENUM 값입니다. 올바른 값을 입력하세요. (또는 root cause 메시지) | JSON 파싱/Enum 역직렬화 실패 |
| MAX_UPLOAD_SIZE_EXCEEDED | 400 | 업로드 가능한 최대 파일 크기를 초과했습니다. (개별 파일 10MB, 총 파일 크기 50MB) | multipart 용량 초과 |
| RESOURCE_NOT_FOUND | 404 | 요청하신 경로를 찾을 수 없습니다. | 존재하지 않는 경로 |
| DATA_INTEGRITY_VIOLATION | 409 | 데이터 무결성 위반 오류가 발생했습니다. | DB 제약 위반 |
| NULL_POINTER_EXCEPTION | 500 | NullPointerException 발생: ... | NPE |
| SQL_EXCEPTION | 500 | 데이터베이스 오류가 발생했습니다. | SQLException |
| NO_CATCH_ERROR | 500 | (예외 메시지 그대로) | 미처리 예외 |

---

### A-4. 레이트리밋 정책

`/src/main/java/com/USWCicrcleLink/server/global/bucket4j/RateLimitAction.java`
`/src/main/java/com/USWCicrcleLink/server/global/bucket4j/RateLimiterAspect.java`

- 구현: Bucket4j + Redis(Lettuce) 분산 버킷. `Bandwidth.classic(capacity, Refill.intervally(n, duration))`
- 버킷 키: `"{clientId}:{ACTION}"`
- 초과 시: `RateLimitExceededException(TOO_MANY_ATTEMPT)` → **HTTP 400, 코드 `ATTEMPT-503`**, 메시지 "최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요"

| 액션 | 제한 | 리필 방식 | 키 기준 | 적용 위치 |
|---|---|---|---|---|
| `APP_LOGIN` | 5회 / 5분 | intervally(5, 5분) | 요청 DTO의 `getClientId()` (앱 로그인 account) | UserController:198 |
| `WEB_LOGIN` | 5회 / 5분 | intervally(5, 5분) | 요청 DTO의 `getClientId()` = `adminAccount` / `leaderAccount` | AdminLoginService:33, ClubLeaderLoginService:34 |
| `EMAIL_VERIFICATION` | 5회 / 5분 | intervally(5, 5분) | 요청 DTO `getClientId()` | UserService:292 |
| `ID_FOUND_EMAIL` | 5회 / 5분 | intervally(5, 5분) | 요청 DTO `getClientId()` | UserService:309 |
| `PW_FOUND_EMAIL` | 5회 / 5분 | intervally(5, 5분) | 요청 DTO `getClientId()` | UserController:163 |
| `WITHDRAWAL_EMAIL` | 5회 / 5분 | intervally(5, 5분) | **로그인 사용자 UUID** (`userService.getUserByAuth()`) | UserController:207 |
| `VALIDATE_CODE` | 5회 / 5분 | intervally(5, 5분) | 요청 DTO `getClientId()` 또는 메서드 인자 UUID | UserController:176 |
| `WITHDRAWAL_CODE` | 5회 / 5분 | intervally(5, 5분) | **로그인 사용자 UUID** | UserController:220 |
| `LEADER_CHANGE_PW` | 5회 / **1분** | intervally(5, 1분) | — | **코드상 사용처 없음 (enum만 정의됨)** |
| `EVENT_VERIFY` | 5회 / 5분 | intervally(5, 5분) | **로그인 사용자 UUID** | EventVerificationController:47 |

> **IP 기준 레이트리밋은 코드상 존재하지 않습니다.** 키는 account 문자열 또는 UUID입니다.
> `ClientIdentifier` 인터페이스를 구현한 요청 DTO만 clientId를 제공하며, 미구현 시 clientId가 빈 문자열이 되어 전역 공유 버킷이 됩니다.

---

### A-5. CORS 설정

`/src/main/java/com/USWCicrcleLink/server/global/security/config/SecurityConfig.java` + `application-prod.yml`

| 항목 | 값 |
|---|---|
| 허용 Origin (prod) | `https://donggurami.net`, `https://*.donggurami.net` (`cors.allowed-origins`, 콤마 구분, `addAllowedOriginPattern`으로 등록) |
| 허용 Origin (local/test) | 환경변수 `LOCAL_SERVER_ALLOWED_ORIGINS` / `TEST_SERVER_ALLOWED_ORIGINS` |
| 허용 메서드 | `GET, POST, PUT, PATCH, DELETE, OPTIONS` |
| 허용 헤더 | `*` |
| 노출 헤더 | `Authorization` |
| allowCredentials | `true` (쿠키 전송 필수) |
| maxAge | `3600`초 |
| 적용 경로 | `/**` |

기타 보안 설정: CSRF 비활성화, 세션 `STATELESS`.

---

### A-6. 페이징 규약

| 항목 | 값 |
|---|---|
| 쿼리 파라미터 | `page` (0-base), `size` |
| 기본값 | `page=0`, `size=10` |
| 정렬 (동아리 목록) | 컨트롤러에서 `Sort.by("clubId").descending()` 지정. **단, `ClubRepositoryCustomImpl.findAllWithMemberAndLeaderCount`의 JPQL에 `ORDER BY`가 없어 Sort가 실제로는 적용되지 않음** (DB 반환 순서에 의존) |
| 정렬 (공지 목록) | `Sort.by("noticeCreatedAt").descending()` — `noticeRepository.findAll(pageable)`이므로 **실제 적용됨** |
| 응답 필드 | `content`, `totalPages`, `totalElements`, `currentPage` |
| `Pageable` 바인딩 | 자동 바인딩이 아니라 컨트롤러에서 `PageRequest.of(page, size, sort)` 수동 생성. `sort` 쿼리 파라미터는 **지원하지 않음** |

---

### A-7. 파일 업로드 규약 (Presigned URL 방식)

`/src/main/java/com/USWCicrcleLink/server/global/s3File/Service/S3FileUploadService.java`

#### 흐름
1. 클라이언트가 서버에 `multipart/form-data`로 **실제 파일을 전송**한다.
2. 서버는 파일을 S3에 직접 올리지 않고, **확장자 + 파일 시그니처만 검증**한다.
3. 서버가 `{디렉터리}/{랜덤UUID}.{확장자}` 형태의 S3 Key를 만들고 **PUT용 presigned URL을 생성**해 DB에 Key를 저장한 뒤 URL을 응답한다.
4. **클라이언트가 응답받은 presigned URL로 S3에 직접 `PUT` 요청**을 보내 실제 바이너리를 업로드해야 한다. 이 단계를 생략하면 DB에는 레코드만 있고 S3 객체는 없다.
5. 조회 시에는 서버가 저장된 S3 Key로 **GET용 presigned URL**을 생성해 내려준다.

#### 규격
| 항목 | 값 |
|---|---|
| 허용 확장자 | `jpg`, `jpeg`, `png` (`file.allowed-extensions`, 소문자 비교) |
| 시그니처 검증 | `FileSignatureValidator` — jpg/jpeg `FFD8FF`(4바이트 읽음), png `89504E47`(8바이트 읽음). 시작 부분 일치 시 통과 |
| presigned URL 유효기간 | `URL_EXPIRED_TIME = 1000*60*60` → **1시간** |
| S3 리전 / 버킷 | `ap-northeast-2` / `cloud.aws.s3.bucket` |
| 개별 파일 최대 크기 | 10MB (`spring.servlet.multipart.max-file-size`) |
| 요청 전체 최대 크기 | 50MB (`max-request-size`) |
| S3 디렉터리 | 층별 도면 `floorPhoto/`, 공지 사진 `noticePhoto/` |
| 파일명 비어있음 | S3 Key가 빈 문자열이면 presigned URL 생성을 건너뛰고 `""` 반환 |

#### 파일 검증 에러
| 순서 | 조건 | 코드 |
|---|---|---|
| 1 | 파일/원본 파일명 null | FILE-309 |
| 2 | 확장자(`.`) 없음 | FILE-310 |
| 3 | 허용 목록 밖 확장자 | FILE-311 |
| 4 | 시그니처 불일치 | FILE-311 |
| 5 | 스트림 읽기 IOException | FILE-312 |

---

### A-8. API 문서 경로

| 문서 | 경로 | 근거 |
|---|---|---|
| Swagger UI | `/` (루트) | `application-security.yml` → `springdoc.swagger-ui.path: /`, `disable-swagger-default-url: true` |
| OpenAPI JSON | `/v3/api-docs` | `springdoc.api-docs.path`, 버전 `OPENAPI_3_1` |
| ReDoc | `/docs` | `DocsController` → `templates/redoc.html` (내부에서 `/v3/api-docs` 로드) |
| Health Check | `/health-check` | `HealthCheckController` |

OpenAPI 메타(`OpenApiConfig`):
- title `USW Circle Link Server API`, version `v1`
- servers: `https://api.donggurami.net`, `http://localhost:8080`
- securityScheme `bearerAuth` (HTTP bearer / JWT), 전역 SecurityRequirement 적용

위 문서 경로는 모두 `permit-all-paths`에 포함되어 **인증 없이 접근 가능**합니다
(`/swagger-ui.html`, `/swagger-ui/**`, `/v3/api-docs`, `/v3/api-docs/**`, `/docs`, `/`, `/health-check`).

---

---

## 3. 사용자 API — 계정 · 인증

> 검증 근거 파일
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/user/api/UserController.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/security/Integration/api/IntegrationAuthController.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/user/service/{UserService,AuthTokenService,WithdrawalTokenService,PasswordService}.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/email/service/{EmailService,EmailTokenService}.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/security/jwt/JwtProvider.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/security/config/SecurityConfig.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/resources/yaml/application-security.yml`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/exception/ExceptionType.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/bucket4j/{RateLimitAction,RateLimiterAspect}.java`

---

### 0. 공통 사항 (모든 엔드포인트에 적용)

#### 0.1 성공 응답 래퍼 `ApiResponse<T>`
`global/response/ApiResponse.java`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 성공 메시지 (항상 존재) |
| data | T | 페이로드. `@JsonInclude(NON_NULL)` 이므로 **null이면 키 자체가 응답에서 빠짐** |

```json
{ "message": "로그인 성공", "data": { "...": "..." } }
```

#### 0.2 에러 응답 `ErrorResponse` (GlobalExceptionHandler)
`global/response/ErrorResponse.java`, `global/exception/GlobalExceptionHandler.java`

| 필드 | 타입 | 설명 |
|---|---|---|
| exception | String | 예외 클래스 SimpleName (예: `UserException`) |
| code | String | ExceptionType 코드 (예: `USR-211`) |
| message | String | ExceptionType 메시지 |
| status | Integer | HTTP 상태 숫자 |
| error | String | HTTP 상태 reason phrase |
| additionalData | Object | Bean Validation 실패 시 `{필드명: 메시지}` 맵, 그 외 null |

```json
{
  "exception": "UserException",
  "code": "USR-211",
  "message": "아이디 혹은 비밀번호가 일치하지 않습니다",
  "status": 401,
  "error": "Unauthorized",
  "additionalData": null
}
```

전역 공통 에러:

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | Bean Validation 실패. `additionalData`에 필드별 메시지 |
| INVALID_REQUEST_BODY | 400 | 유효하지 않은 JSON 혹은 ENUM 값입니다. 올바른 값을 입력하세요. | JSON 파싱 실패 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | 경로변수/쿼리/헤더 UUID 변환 실패 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 레이트리밋 초과 (`RateLimitExceededException`) |
| NULL_POINTER_EXCEPTION | 500 | NullPointerException 발생: … | NPE |
| NO_CATCH_ERROR | 500 | (예외 메시지) | 미처리 예외 |

인증 실패는 `CustomAuthenticationEntryPoint`가 **다른 형식**으로 응답합니다 (JwtFilter 단계):

```json
{ "status": 401, "errorCode": "TOKEN_EXPIRED", "message": "토큰이 만료되었습니다." }
```
- `errorCode`: `AUTH_REQUIRED`(토큰 없음) / `TOKEN_EXPIRED`(만료) / `INVALID_TOKEN`(변조·형식오류, message는 "인증이 필요합니다.")

#### 0.3 인증 방식
- **액세스 토큰**: `Authorization: Bearer {accessToken}` 요청 헤더. HS256 서명 JWT, `sub`=userUUID, 선택 클레임 `clubUUID`. **유효기간 30분** (`ACCESS_TOKEN_EXPIRATION_TIME = 1800000L`).
- **리프레시 토큰**: JWT가 아닌 **랜덤 UUID 문자열**. Redis 키 `refreshToken:{값}` → userUUID, TTL **7일** (`604800000L`). 클라이언트에는 `refreshToken` **쿠키**로 전달.
- 쿠키 설정 (`JwtProvider.setRefreshTokenCookie`):
  `refreshToken={값}; Path=/; HttpOnly; Max-Age=604800; SameSite={security.cookie.same-site}{; Secure if security.cookie.secure}`
  - prod 프로필: `SameSite=None; Secure` (`application-prod.yml`)
  - local 프로필: `SameSite=Lax`, Secure 없음 (`application-local.yml`)
  - test 프로필: `secure: false`, same-site 미지정 → 코드 기본값 `Lax`
- CORS: `allowCredentials=true`, `Authorization` 헤더 노출(exposedHeader), 허용 메서드 `GET,POST,PUT,PATCH,DELETE,OPTIONS`.

#### 0.4 이메일 값 규칙 (매우 중요)
DB `email` 컬럼과 모든 요청/응답의 `email` 필드는 **수원대 메일 주소의 로컬파트만** 저장·전달합니다.
`EmailService`가 발송 시 `email + "@suwon.ac.kr"` 로 조합합니다. 컬럼 길이 30자.
→ 예시 값: `"honggildong"` (O), `"honggildong@suwon.ac.kr"` (X)

#### 0.5 검증 그룹 동작 (요청 DTO 해석에 필수)
`ValidationSequence = @GroupSequence({Default, NotBlankGroup, SizeGroup, PatternGroup})`

- `@Validated(ValidationSequence.class)` 로 검증되는 DTO는 **NotBlank → Size → Pattern 순서로 단계 실패 시 즉시 중단**됩니다.
- `@Valid` 또는 그룹 미지정 `@Validated` 는 **Default 그룹만** 실행합니다. 따라서 `groups = ...` 가 명시된 제약은 **작동하지 않습니다**. (해당 엔드포인트마다 별도 표기)

#### 0.6 레이트리밋 (`RateLimitAction.java`)
전 액션 동일: `Bandwidth.classic(5, Refill.intervally(5, Duration.ofMinutes(5)))` → **최대 5회, 5분마다 5개 토큰 일괄 충전**. 단 `LEADER_CHANGE_PW`만 5회/1분(본 문서 범위 밖).

버킷 키는 `RateLimiterAspect`가 `{clientId}:{action}` 으로 만들며, clientId 산출 방식이 액션마다 다릅니다(각 엔드포인트에 표기). **`ClientIdentifier`도 `UUID`도 아닌 인자만 있는 경우 clientId가 빈 문자열이 되어 전역 공유 버킷이 됩니다.**

---

### 회원가입 흐름 (1 → 4번 순서)

```
[1] POST /users/temporary/register  {email}
        → EmailToken 생성(5분 만료) + 인증메일 발송
        ← data: { emailToken_uuid, email }
[2] (사용자가 메일 링크 클릭) GET /users/email/verify-token?emailTokenUUID=...
        → isVerified=true, signupUUID 생성  ← HTML 페이지
[3] POST /users/email/verification  {email}
        ← data: { emailTokenUUID, signupUUID }
[4] POST /users/signup
        Header: emailTokenUUID, signupUUID   Body: SignUpRequest
        → 서버가 두 UUID 매칭 확인 후 가입 확정
```
`emailTokenUUID`는 **메일 링크 식별자**(1단계에서 발급), `signupUUID`는 **인증 완료 증표**(2단계 링크 클릭 시점에 서버가 생성). 4단계는 두 값이 동일한 EmailToken 행에 속하는지 대조하여, 링크를 실제로 클릭한 사람만 가입시키는 구조입니다.

---

#### 1. 신규 회원가입 요청 (인증 메일 전송)
`POST /users/temporary/register`

| 항목 | 값 |
|---|---|
| 인증 | permitAll (`application-security.yml` permit-all-paths) |
| 레이트리밋 | EMAIL_VERIFICATION 5회/5분 — **버킷 키가 `":EMAIL_VERIFICATION"`(전역 공유)**. `@RateLimite`가 `UserService.sendSignUpMail(EmailToken)` 에 붙어 있는데 `EmailToken`이 `ClientIdentifier`/`UUID`가 아니라 clientId가 빈 문자열로 산출됨 |
| Content-Type | application/json |

**요청**

바디: `EmailDTO` (`@Validated`, Default 그룹)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| email | String | O | `@NotBlank` ("이메일을 입력해주세요") | @suwon.ac.kr **로컬파트만** |

```json
{ "email": "honggildong" }
```

**응답** — `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "인증 메일 전송 완료" |
| data.emailToken_uuid | UUID | 이메일 토큰 UUID. **필드명이 스네이크 표기(`emailToken_uuid`)** — `VerifyEmailResponse` 필드명 그대로 |
| data.email | String | 요청한 이메일 로컬파트 |

```json
{
  "message": "인증 메일 전송 완료",
  "data": {
    "emailToken_uuid": "3f1c9a52-0b1e-4a77-9f0d-7a2b6c4e8d10",
    "email": "honggildong"
  }
}
```

**비즈니스 규칙** (`UserService.checkEmailDuplication` → `sendSignUpMail`)
1. `USER_TABLE`에 동일 email 존재 → `USR-206` 실패.
2. `EMAIL_TOKEN_TABLE`에 동일 email 존재 → 기존 토큰 재사용하고 **만료시간을 현재+5분으로 연장**(`extendExpirationTime`).
3. 둘 다 없으면 새 `EmailToken` 생성 (`emailTokenUUID`=랜덤 UUID, `expirationTime`=현재+5분, `isVerified`=false, `signupUUID`=null).
4. 인증 메일 발송.

**부수효과**
- `EMAIL_TOKEN_TABLE` INSERT 또는 만료시간 UPDATE.
- 메일 발송(`@Async`): 제목 "동구라미 회원가입 인증 메일", 수신자 `{email}@suwon.ac.kr`, 템플릿 `templates/main.html`의 `https://www.naver.com` 링크를 `{email.baseUrl}/users/email/verify-token?emailTokenUUID={UUID}` 로 치환, `static/images/logo.png` 인라인 첨부.
- Redis 레이트리밋 버킷 갱신.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-206 | 409 | 이미 존재하는 회원입니다. | USER_TABLE에 동일 이메일 존재 |
| EMAIL_TOKEN-003 | 500 | 이메일 토큰 생성중 오류가 발생했습니다. | EmailToken 저장 실패 |
| EMAIL_TOKEN-004 | 500 | 이메일 토큰의 필드 업데이트 후, 저장하는 과정에서 오류가 발생했습니다. | 만료시간 연장 저장 실패 |
| EMAIL_TOKEN-001 | 400 | 해당 토큰이 존재하지 않습니다. | 중복 판정 후 재조회 실패(경합) |
| EML-501 | 500 | 메일 전송에 실패했습니다. | MimeMessage 생성 실패 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 레이트리밋 초과 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | email 공백 |

---

#### 2. 이메일 인증 링크 처리 (메일 안 버튼 클릭)
`GET /users/email/verify-token`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | (응답) text/html — **JSON이 아님** |

> 이 엔드포인트는 `ModelAndView`를 반환하는 **브라우저용 HTML 페이지**입니다. 앱/프론트에서 직접 호출하지 않고, 사용자가 메일의 링크를 클릭해 도달합니다.

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 쿼리 | emailTokenUUID | UUID | O | 1번 응답의 `emailToken_uuid` (메일 링크에 포함되어 있음) |

**응답** — `200 OK`, HTML

| 뷰 | 파일 | 조건 |
|---|---|---|
| success | `templates/success.html` | 인증 성공 |
| expired | `templates/expired.html` | `EmailTokenException` — 토큰 만료 |
| failure | `templates/failure.html` | 그 외 모든 예외 (토큰 미존재 `EMAIL_TOKEN-001`, 저장 실패 `EMAIL_TOKEN-004` 등) |

**비즈니스 규칙** (`UserService.verifyEmailToken`)
1. `emailTokenUUID`로 EmailToken 조회 — 없으면 `EmailException(EMAIL_TOKEN-001)` → **failure 뷰**.
2. `isExpired()` (현재 > expirationTime, 발급 후 5분) → `EmailTokenException(EMAIL_TOKEN-002)` → **expired 뷰**.
3. `verifyEmail()` 호출: `isVerified=true`, **`signupUUID`에 새 랜덤 UUID 생성**, 저장 → 실패 시 `EMAIL_TOKEN-004` → failure 뷰.

**부수효과**: `EMAIL_TOKEN_TABLE.is_verified=true`, `signup_uuid` 채워짐.

**에러**
컨트롤러가 예외를 삼키고 뷰로 분기하므로 **JSON 에러 응답은 없습니다.** 단, `emailTokenUUID` 쿼리값이 UUID로 변환되지 않으면 컨트롤러 진입 전에 실패합니다.

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | emailTokenUUID가 UUID 형식이 아님 (JSON 응답) |

---

#### 3. 인증 확인 버튼 (signupUUID 수령)
`POST /users/email/verification`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청**

바디: `EmailDTO` (`@Validated`, Default 그룹)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| email | String | O | `@NotBlank` | 1번에서 인증 요청한 이메일 로컬파트 |

```json
{ "email": "honggildong" }
```

**응답** — `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "인증 확인 버튼 클릭 후, 이메일 인증 완료" |
| data.emailTokenUUID | UUID | 4번 `signUp` 의 `emailTokenUUID` 헤더 값 |
| data.signupUUID | UUID | 4번 `signUp` 의 `signupUUID` 헤더 값 |

```json
{
  "message": "인증 확인 버튼 클릭 후, 이메일 인증 완료",
  "data": {
    "emailTokenUUID": "3f1c9a52-0b1e-4a77-9f0d-7a2b6c4e8d10",
    "signupUUID": "b7d21e04-5c8f-4c9a-8f11-2d3e4a5b6c7d"
  }
}
```

**비즈니스 규칙** (`EmailTokenService.checkEmailIsVerified`)
1. email로 EmailToken 조회 — 없으면 `EMAIL_TOKEN-001`.
2. `isVerified == false` (2번 링크를 아직 클릭하지 않음) → `EMAIL_TOKEN-005`.
3. 통과 시 두 UUID 반환.

**부수효과**: 없음 (조회 전용).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| EMAIL_TOKEN-001 | 400 | 해당 토큰이 존재하지 않습니다. | 해당 email의 EmailToken 없음(1번 미수행/만료 후 삭제) |
| EMAIL_TOKEN-005 | 400 | 인증이 완료되지 않은 이메일 토큰입니다. | 메일 링크 미클릭 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | email 공백 |

---

#### 4. 신규 회원가입 확정
`POST /users/signup`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청 — 헤더**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | emailTokenUUID | UUID | O | 3번 응답 `data.emailTokenUUID` |
| 헤더 | signupUUID | UUID | O | 3번 응답 `data.signupUUID` |

**요청 — 바디** `SignUpRequest` (`@Validated(ValidationSequence.class)` → NotBlank → Size → Pattern 순 단계 검증)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| account | String | O | `@NotBlank` / 5~20자 / `^[a-zA-Z0-9]+$` | 로그인 아이디 |
| password | String | O | `@NotBlank` / 8~20자 / `^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?])(?!.*\s).*$` | 영문+숫자+특수문자 각 1개 이상, 공백 불가 |
| confirmPassword | String | O | `@NotBlank` (패턴 검증 없음) | 비밀번호 확인. 서비스에서 `password`와 동일성 검사 |
| userName | String | O | `@NotBlank` / 2~30자 / `^[a-zA-Z가-힣]+$` | 이름 (한글 또는 영문만, 공백·숫자 불가) |
| telephone | String | O | `@NotBlank` / 정확히 11자 / `^01[0-9]{9}$` | 하이픈 없는 휴대폰 번호 |
| studentNumber | String | O | `@NotBlank` / 정확히 8자 / `^[0-9]{8}$` | 학번 |
| major | String | O | `@NotBlank` / 1~20자 / `^[가-힣a-zA-Z]+$` | 학과 (공백 불가) |

> `password` 정규식의 특수문자 클래스에는 `~` 와 `` ` `` 가 **없습니다**. 반면 서비스단 `PasswordService.specialCharPattern` 에는 `~` `` ` `` 가 포함됩니다. 즉 `~`만 특수문자로 쓰면 **Bean Validation 단계(`INVALID_ARGUMENT`)에서 거부**됩니다.

```json
{
  "account": "gildong01",
  "password": "Abcd1234!",
  "confirmPassword": "Abcd1234!",
  "userName": "홍길동",
  "telephone": "01012345678",
  "studentNumber": "20250001",
  "major": "컴퓨터학부"
}
```

**응답** — `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "회원가입이 정상적으로 완료되어 로그인이 가능합니다." |

`ApiResponse<Void>` 이며 data는 null이라 **직렬화에서 제외**됩니다.

```json
{ "message": "회원가입이 정상적으로 완료되어 로그인이 가능합니다." }
```

**비즈니스 규칙** (검증 순서 그대로)
1. Bean Validation (`ValidationSequence`).
2. `isEmailVerified(emailTokenUUID, signupUUID)`
   - `emailTokenUUID`로 EmailToken 조회 — 없으면 `EMAIL_TOKEN-001`.
   - 저장된 `signupUUID` ≠ 헤더 `signupUUID` → `USR-219` (401).
   - **주의**: 2번 링크를 클릭하지 않아 `signupUUID`가 null인 토큰으로 호출하면 `emailToken.getSignupUUID().equals(...)` 에서 NPE → `500 NULL_POINTER_EXCEPTION`. (코드상 확인됨)
   - 통과 시 EmailToken의 email을 가입 이메일로 사용 (**바디에 email 필드가 없음**).
3. `checkNewSignupCondition`
   - 3-1. 아이디 중복: `USER_TABLE.user_account` 또는 `CLUB_MEMBER_TEMP.profile_temp_account` 에 존재 → `USR-207`.
   - 3-2. 비밀번호 유효성(`PasswordService.validatePassword`): 빈칸(`USR-203`) → 영문/숫자/특수문자 각 1개 이상(`USR-214`) → password==confirmPassword(`USR-202`) 순.
   - 3-3. 프로필 중복: `userName + studentNumber + telephone` 3개 모두 일치하는 Profile 존재 → `PFL-207`.
4. `signUpUser`
   - `User` 생성: 비밀번호 BCrypt 인코딩, `role=USER`, `userUUID` 랜덤 생성(@PrePersist), `userCreatedAt/UpdatedAt` 설정.
   - `Profile` 생성: telephone에서 하이픈 제거 후 저장, `memberType=REGULARMEMBER`(정회원 기본값).
   - 해당 email의 `EmailToken` 행 삭제.

**부수효과**: `USER_TABLE` INSERT, `PROFILE` INSERT, `EMAIL_TOKEN_TABLE` DELETE. 메일/푸시 발송 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | DTO 검증 실패 (additionalData에 필드별 사유) |
| EMAIL_TOKEN-001 | 400 | 해당 토큰이 존재하지 않습니다. | emailTokenUUID에 해당하는 토큰 없음 |
| USR-219 | 401 | 요청 받은 SIGNUPUUID가 일치하지 않습니다 | signupUUID 헤더 불일치 |
| USR-207 | 409 | 계정이 중복됩니다. | account 중복 (User 또는 ClubMemberTemp) |
| USR-203 | 400 | 비밀번호 값이 빈칸입니다 | password/confirmPassword가 공백만 |
| USR-214 | 400 | 영문자,숫자,특수문자는 적어도 1개 이상씩 포함되어야합니다 | 비밀번호 구성 조건 미달 |
| USR-202 | 400 | 두 비밀번호가 일치하지 않습니다. | password ≠ confirmPassword |
| PFL-207 | 400 | 프로필이 이미 존재합니다 | 이름+학번+전화번호 완전 일치 프로필 존재 |
| USR-218 | 500 | 회원 생성중 오류 발생 | User 객체 생성 실패 |
| PFL-210 | 500 | 프로필 생성중 오류 발생 | Profile 객체 생성 실패 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. … | 헤더 UUID 파싱 실패 |

---

#### 5. 아이디 중복 확인
`GET /users/verify-duplicate/{account}`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | account | String | O | 중복 확인할 로그인 아이디. **경로변수에는 Bean Validation이 없음** (형식·길이 미검사) |

`GET /users/verify-duplicate/gildong01`

**응답** — `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "사용 가능한 ID 입니다." |

```json
{ "message": "사용 가능한 ID 입니다." }
```

**비즈니스 규칙** (`UserService.verifyAccountDuplicate`)
1. `USER_TABLE.user_account` 조회.
2. `CLUB_MEMBER_TEMP.profile_temp_account` 조회.
3. **둘 중 하나라도 존재하면** `USR-207`. 즉 기존 동아리원 가입 대기자가 선점한 아이디도 사용 불가.

**부수효과**: 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-207 | 409 | 계정이 중복됩니다. | User 또는 ClubMemberTemp에 동일 account 존재 |

---

#### 6. 이메일 중복 확인 (기존 동아리원 가입용)
`POST /users/check/{email}/duplicate`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | — (요청 바디 없음) |

> `POST` 이지만 **요청 바디를 받지 않습니다.** 값은 경로변수로 전달합니다.

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | email | String | O | @suwon.ac.kr 로컬파트. 경로 세그먼트이므로 `/` 포함 불가 |

`POST /users/check/honggildong/duplicate`

**응답** — `200 OK`

```json
{ "message": "이메일 중복 확인에 성공하였습니다." }
```

**비즈니스 규칙** (`UserService.verifyEmailDuplicate`)
1. `USER_TABLE` 이메일 중복 → `USR-206`.
2. `CLUB_MEMBER_TEMP.profile_temp_email` 중복 → `CMEM-TEMP-302`.
- **1번(temporary/register)과 달리 `EMAIL_TOKEN_TABLE`은 확인하지 않고, 토큰 생성/메일 발송도 하지 않습니다.**

**부수효과**: 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-206 | 409 | 이미 존재하는 회원입니다. | USER_TABLE에 동일 이메일 |
| CMEM-TEMP-302 | 400 | CLUBMEMBERTEMP 테이블에 존재하는 이메일 입니다. | ClubMemberTemp에 동일 이메일 |

---

#### 7. 기존 동아리원 회원가입 — ⚠️ 현재 비활성화됨
`POST /users/existing/register`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

> **본 엔드포인트의 서비스 로직은 전부 주석 처리되어 있습니다** (`UserController.java` 139~147행). 컨트롤러는 **요청 내용과 무관하게 항상 `400 Bad Request`** 를 반환합니다. `ClubMemberTemp` 생성, 동아리 회장 가입신청 전송 등은 **일어나지 않습니다.**
> 단, `@Validated(ValidationSequence.class)`는 살아 있으므로 바디가 검증 규칙을 위반하면 그보다 먼저 `INVALID_ARGUMENT`(400)로 실패합니다.

**요청 — 바디** `ExistingMemberSignUpRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| account | String | O | `@NotBlank` / 5~20자 / `^[a-zA-Z0-9]+$` | 아이디 |
| password | String | O | `@NotBlank` / 8~20자 / `^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?])(?!.*\s).*$` | 비밀번호 |
| confirmPassword | String | O | `@NotBlank` | 비밀번호 확인 |
| userName | String | O | `@NotBlank` / 2~30자 / `^[a-zA-Z가-힣]+$` | 이름 |
| telephone | String | O | `@NotBlank` / 11자 / `^01[0-9]{9}$` | 전화번호 |
| studentNumber | String | O | `@NotBlank` / 8자 / `^[0-9]{8}$` | 학번 |
| major | String | O | `@NotBlank` / 1~20자 / `^[가-힣a-zA-Z]+$` | 학과 |
| email | String | O | `@NotBlank` (groups 미지정 → Default) | 이메일 로컬파트 |
| clubs | List\<ClubDTO\> | O | `@NotEmpty` | 가입 신청할 동아리 목록 |
| clubs[].clubUUID | UUID | O | `@NotNull` (**중첩 `@Valid` 없음 → 실제 검증 미수행**) | 동아리 UUID |

```json
{
  "account": "gildong02",
  "password": "Abcd1234!",
  "confirmPassword": "Abcd1234!",
  "userName": "홍길동",
  "telephone": "01012345678",
  "studentNumber": "20250002",
  "major": "컴퓨터학부",
  "email": "honggildong",
  "clubs": [{ "clubUUID": "9c8b7a65-4321-4def-8abc-1234567890ab" }]
}
```

**응답** — `400 Bad Request` (성공 응답 없음)

```json
{ "message": "현재 기존 동아리원 회원가입이 불가능합니다." }
```

> 응답 본문이 에러 형식(`ErrorResponse`)이 아니라 **`ApiResponse` 형식**입니다. 상태코드만 400.

**비즈니스 규칙**: 없음 (주석 처리됨). 주석 상태로 남아 있는 원래 흐름은 `checkExistingSignupCondition` → `registerClubMemberTemp` → `sendRequest` 였습니다.

**부수효과**: 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| (없음, ApiResponse 형식) | 400 | 현재 기존 동아리원 회원가입이 불가능합니다. | 검증을 통과한 모든 요청 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | 바디 검증 위반 |

---

#### 8. 로그인
`POST /users/login`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | APP_LOGIN 5회/5분 — 버킷 키 `{account}:APP_LOGIN` (`LogInRequest implements ClientIdentifier`, `getClientId()` = account) |
| Content-Type | application/json |

**요청 — 바디** `LogInRequest` (`@Validated(ValidationSequence.class)`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| account | String | O | `@NotBlank` / 5~20자 / `^[a-zA-Z0-9]+$` | 로그인 아이디 |
| password | String | O | `@NotBlank` / 8~20자 / `^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?])(?!.*\s).*$` | 비밀번호 |
| fcmToken | String | X | 없음 | FCM 디바이스 토큰. 값이 있으면 Profile에 저장 |

```json
{
  "account": "gildong01",
  "password": "Abcd1234!",
  "fcmToken": "dGVzdC1mY20tdG9rZW4tdmFsdWU"
}
```

**응답** — `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "로그인 성공" |
| data.accessToken | String | JWT (HS256). `sub`=userUUID, 소속 동아리가 있으면 `clubUUID` 클레임 포함. **30분 유효** |
| data.refreshToken | String | 랜덤 UUID 문자열. Redis TTL **7일**. 바디에도 실려오지만 쿠키로도 내려감 |

응답 헤더
- `Authorization: Bearer {accessToken}`
- `Set-Cookie: refreshToken={refreshToken}; Path=/; HttpOnly; Max-Age=604800; SameSite=None; Secure` (prod 기준. local은 `SameSite=Lax`, Secure 없음)

```json
{
  "message": "로그인 성공",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhYWFhYWFhYS0xMTExLTIyMjItMzMzMy00NDQ0NDQ0NDQ0NDQifQ.signature",
    "refreshToken": "5f7d9c31-2a4b-4c6d-8e0f-1a2b3c4d5e6f"
  }
}
```

**비즈니스 규칙** (`UserService.userLogin`) — **실패 분기별 예외 코드가 모두 다릅니다**
1. `account`로 `USER_TABLE` 조회.
2. **User가 없는 경우**
   - `CLUB_MEMBER_TEMP`에 동일 account가 있고 **비밀번호까지 일치** → `USR-216` (401, "비회원 사용자입니다.인증을 완료해주세요") — 기존 동아리원 가입 대기 상태.
   - 그 외 (임시회원도 없거나, 있어도 비밀번호 불일치) → `USR-220` (401, "제3자의 로그인 요청 시도 입니다").
3. **User가 있고 비밀번호 불일치** → `USR-211` (401, "아이디 혹은 비밀번호가 일치하지 않습니다").
4. userUUID로 Profile 조회 실패 → `PFL-201` (404).
5. 액세스 토큰 생성(응답 헤더 `Authorization` 세팅) → 리프레시 토큰 생성.
6. 리프레시 토큰 생성 시 **해당 UUID의 기존 리프레시 토큰을 Redis에서 먼저 삭제** → 사실상 단일 세션(마지막 로그인만 유효).
7. `fcmToken`이 non-null·non-empty면 Profile에 저장하고 `fcmTokenCertificationTimestamp = 현재 + 60일`.

**부수효과**
- Redis: `refreshToken:{새토큰}` = userUUID, TTL 7일 저장 / 기존 토큰 키 삭제.
- 응답 헤더 `Authorization`, `Set-Cookie` 설정.
- Profile FCM 토큰 UPDATE (fcmToken 전달 시).
- 레이트리밋 버킷 소비.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-216 | 401 | 비회원 사용자입니다.인증을 완료해주세요 | User 없음 + ClubMemberTemp 존재 + 비밀번호 일치 |
| USR-220 | 401 | 제3자의 로그인 요청 시도 입니다 | User 없음 + (ClubMemberTemp 없음 또는 비밀번호 불일치) |
| USR-211 | 401 | 아이디 혹은 비밀번호가 일치하지 않습니다 | User 존재 + 비밀번호 불일치 |
| PFL-201 | 404 | 프로필이 존재하지 않습니다. | User는 있으나 Profile 없음 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 동일 account로 5분 내 6회 시도 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | account/password 형식 위반 |

---

#### 9. 아이디 찾기
`GET /users/find-account/{email}`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | ID_FOUND_EMAIL 5회/5분 — 버킷 키 `{user.email}:ID_FOUND_EMAIL` (`User implements ClientIdentifier`, `getClientId()` = email). `@RateLimite`는 `UserService.sendAccountInfoMail(User)` 에 위치하므로 **User 조회 성공 후에만** 카운트됨 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | email | String | O | @suwon.ac.kr 로컬파트 |

`GET /users/find-account/honggildong`

**응답** — `200 OK`

```json
{ "message": "계정 정보 전송 완료" }
```

> 응답 바디에는 아이디가 담기지 않습니다. **아이디는 메일로만 전달**됩니다.

**비즈니스 규칙**
1. email로 `USER_TABLE` 조회 — 없으면 `USR-201`.
2. 아이디 안내 메일 생성·발송.

**부수효과**
- 메일 발송(`@Async`): 수신 `{email}@suwon.ac.kr`, 제목 "동구라미의 아이디를 찾기 위한 메일입니다.", 본문 "회원님의 아이디는  {userAccount} 입니다." (plain text).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-201 | 400 | 사용자가 존재하지 않습니다. | 해당 이메일의 User 없음 |
| EML-501 | 500 | 메일 전송에 실패했습니다. | 메일 생성 실패 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 동일 이메일로 5분 내 6회 |

---

### 비밀번호 찾기 3단계 흐름 (10 → 11 → 12)

```
[10] POST /users/auth/send-code   {userAccount, email}
        → 4자리 인증코드 메일 발송
        ← data: "<userUUID>"        ← ★ 이 값을 이후 두 단계의 uuid 헤더로 사용
[11] POST /users/auth/verify-token   Header: uuid  Body: {authCode}
        → 코드 일치 확인 후 AuthToken 삭제
[12] PATCH /users/reset-password     Header: uuid  Body: {password, confirmPassword}
```
**단계 간 연결 고리는 오직 `uuid`(=userUUID) 헤더 하나입니다.** 12단계는 11단계 성공 여부를 서버에서 별도로 검사하지 않습니다 (11단계에서 AuthToken이 삭제될 뿐, 12단계는 AuthToken을 조회하지 않음). — 코드상 확인된 동작.

---

#### 10. 비밀번호 찾기 1단계 — 인증 코드 전송
`POST /users/auth/send-code`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | PW_FOUND_EMAIL 5회/5분 — **버킷 키가 `":PW_FOUND_EMAIL"`(전역 공유)**. 인자 `UserInfoDto`가 `ClientIdentifier`/`UUID`가 아니라 clientId가 빈 문자열로 산출됨 |
| Content-Type | application/json |

**요청 — 바디** `UserInfoDto` (`@Valid` → **Default 그룹만 실행**)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| userAccount | String | O(논리상) | 선언은 `@NotBlank`/5~20자/`^[a-zA-Z0-9]+$` 이나 **모두 `groups` 지정 → `@Valid`로는 실행되지 않음** | 로그인 아이디 |
| email | String | O | `@NotBlank`("이메일 필수 입력 값입니다.") — groups 미지정이라 **실제로 실행됨**. `@Size(1~30)`은 SizeGroup 소속이라 미실행 | 이메일 로컬파트 |

```json
{ "userAccount": "gildong01", "email": "honggildong" }
```

**응답** — `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "인증코드가 전송 되었습니다" |
| data | UUID (문자열) | **userUUID**. 11·12단계의 `uuid` 헤더 값 |

```json
{
  "message": "인증코드가 전송 되었습니다",
  "data": "aaaaaaaa-1111-2222-3333-444444444444"
}
```

**비즈니스 규칙**
1. `userAccount + email` **동시 일치**하는 User 조회 — 없으면 `USR-209`. (아이디만 맞거나 이메일만 맞으면 실패)
2. `AuthTokenService.createOrUpdateAuthToken`: 해당 유저의 AuthToken이 있으면 **인증코드만 갱신**, 없으면 새로 생성. 인증코드는 `Random`으로 만든 **4자리 숫자 문자열**(`0000`~`9999`).
3. 인증 코드 메일 발송.
- **AuthToken에는 만료시간 필드가 없습니다** (`AuthToken.java` 확인) → 인증 코드는 시간 만료되지 않고, 재요청 시 덮어써지거나 11단계 성공 시 삭제됩니다.

**부수효과**
- `AUTH_TOKEN_TABLE` INSERT/UPDATE.
- 메일 발송(`@Async`): 수신 `{email}@suwon.ac.kr`, 제목 "비밀번호 찾기 메일 입니다.", 본문 "인증코드는  {4자리} 입니다."

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-209 | 400 | 올바르지 않은 이메일 혹은 아이디입니다. | userAccount+email 조합 불일치 |
| EML-501 | 500 | 메일 전송에 실패했습니다. | 메일 생성 실패 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 레이트리밋 초과(전역 버킷) |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | email 공백 |

---

#### 11. 비밀번호 찾기 2단계 — 인증 코드 검증
`POST /users/auth/verify-token`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | VALIDATE_CODE 5회/5분 — 버킷 키 `{uuid}:VALIDATE_CODE` (메서드 인자 중 `UUID uuid` 헤더값 사용) |
| Content-Type | application/json |

**요청 — 헤더**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | uuid | UUID | O | 10단계 응답 `data` (userUUID) |

**요청 — 바디** `AuthCodeRequest` (`@Valid`, groups 미지정이라 정상 동작)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| authCode | String | O | `@NotBlank` ("인증 코드를 입력해주세요.") | 메일로 받은 4자리 숫자 |

```
uuid: aaaaaaaa-1111-2222-3333-444444444444
```
```json
{ "authCode": "0427" }
```

**응답** — `200 OK`

```json
{ "message": "인증 코드 검증이 완료되었습니다" }
```

**비즈니스 규칙**
1. `uuid`로 AuthToken 조회 — 없으면 `AC-102`.
2. 저장된 authCode와 문자열 `equals` 비교 — 불일치 시 `AC-101`.
3. 검증 성공 시 **AuthToken 즉시 삭제** (`deleteAuthToken`). 동일 코드 재사용 불가.

**부수효과**: `AUTH_TOKEN_TABLE` DELETE.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| AC-102 | 400 | 인증 코드 토큰이 존재하지 않습니다 | 해당 uuid의 AuthToken 없음(10단계 미수행 또는 이미 검증 완료) |
| AC-101 | 400 | 인증번호가 일치하지 않습니다 | authCode 불일치 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 동일 uuid로 5분 내 6회 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | authCode 공백 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. … | uuid 헤더 파싱 실패 |

---

#### 12. 비밀번호 찾기 3단계 — 비밀번호 재설정
`PATCH /users/reset-password`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청 — 헤더**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | uuid | UUID | O | 10단계 응답 `data` (userUUID) |

**요청 — 바디** `PasswordRequest`

> ⚠️ 컨트롤러에 **`@Valid`/`@Validated`가 없습니다** (`@RequestBody PasswordRequest request`). 따라서 DTO의 `@NotBlank`/`@Size(8~20)`/`@Pattern`은 **전혀 실행되지 않으며**, 검증은 전적으로 `PasswordService`가 담당합니다. → **길이 8~20자 제한이 이 엔드포인트에서는 강제되지 않습니다.**

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| password | String | O | (DTO 선언은 8~20자 + `^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?])(?!.*\s).*$` 이나 **미적용**). 실제 적용: 공백 아님 + 영문/숫자/특수문자(`[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?~\`]`) 각 1개 이상 | 새 비밀번호 |
| confirmPassword | String | O | 실제 적용: 공백 아님 + `password`와 완전 일치 | 새 비밀번호 확인 |

```
uuid: aaaaaaaa-1111-2222-3333-444444444444
```
```json
{ "password": "NewPass99!", "confirmPassword": "NewPass99!" }
```

**응답** — `200 OK`

```json
{ "message": "비밀번호가 변경되었습니다." }
```

**비즈니스 규칙** (`UserService.resetPW` — 검증 순서 그대로)
1. `uuid`로 User 조회 — 없으면 `USR-210`.
2. 새 비밀번호가 **현재 비밀번호와 동일**하면 `USR-217` (재사용 금지).
3. `PasswordService.validatePassword`
   - 3-1. `password` 또는 `confirmPassword`가 trim 후 빈 문자열 → `USR-203`.
   - 3-2. 영문/숫자/특수문자 각 1개 이상 아님 → `USR-214`.
   - 3-3. `password ≠ confirmPassword` → `USR-202`.
4. BCrypt 인코딩 후 저장.
- **11단계 인증 성공 여부를 확인하지 않습니다.** 유효한 userUUID만 알면 이 API가 호출 가능한 구조입니다(코드상 확인됨).

**부수효과**: `USER_TABLE.user_pw` UPDATE (`@PreUpdate`로 `user_updated_at` 갱신). 세션/리프레시 토큰 무효화는 **하지 않습니다**.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-210 | 400 | 회원의 uuid를 찾을 수 없습니다. | uuid에 해당하는 User 없음 |
| USR-217 | 400 | 현재 비밀번호와 같은 비밀번호로 변경할 수 없습니다. | 새 비밀번호가 기존과 동일 |
| USR-203 | 400 | 비밀번호 값이 빈칸입니다 | password/confirmPassword 공백 |
| USR-214 | 400 | 영문자,숫자,특수문자는 적어도 1개 이상씩 포함되어야합니다 | 구성 조건 미달 |
| USR-202 | 400 | 두 비밀번호가 일치하지 않습니다. | 두 값 불일치 |
| NULL_POINTER_EXCEPTION | 500 | NullPointerException 발생: … | `password`/`confirmPassword` 키를 아예 누락 (검증 애노테이션 미적용으로 null이 서비스까지 전달됨) |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. … | uuid 헤더 파싱 실패 |

---

#### 13. 비밀번호 변경 (로그인 상태)
`PATCH /users/userpw`

| 항목 | 값 |
|---|---|
| 인증 | **ROLE_USER** (`SecurityConfig`: `PATCH /profiles/change, /users/userpw, /club-leader/fcmtoken → hasRole("USER")`) |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청 — 헤더**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |

**요청 — 바디** `UpdatePwRequest` (`@Validated(ValidationSequence.class)`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| userPw | String | O(논리상) | **검증 애노테이션 없음** | 현재 비밀번호 |
| newPw | String | O | `@NotBlank` / 8~20자 / `^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?])(?!.*\s).*$` | 새 비밀번호 |
| confirmNewPw | String | O | `@NotBlank` | 새 비밀번호 확인 |

```json
{ "userPw": "Abcd1234!", "newPw": "NewPass99!", "confirmNewPw": "NewPass99!" }
```

**응답** — `200 OK`

```json
{ "message": "비밀번호가 성공적으로 업데이트 되었습니다." }
```

**비즈니스 규칙** (`UserService.updateNewPW` — 검증 순서 그대로)
1. SecurityContext에서 로그인 User 획득.
2. `userPw`가 현재 비밀번호와 불일치 → `USR-204`.
3. `newPw`가 현재 비밀번호와 동일 → `USR-217`.
4. `checkPasswordFieldBlank(newPw, newPw)` → `USR-203`. **인자로 `newPw`를 두 번 전달**하므로 `confirmNewPw`의 공백 여부는 이 단계에서 검사되지 않습니다(코드상 확인, `UserService.java` 93행).
5. `checkPasswordCondition(newPw)` → `USR-214`.
6. `checkPasswordMatch(newPw, confirmNewPw)` → `USR-202`.
7. BCrypt 인코딩 후 저장. 저장 실패 시 **`UserException`을 던지지만 코드는 `PFL-202`(PROFILE_UPDATE_FAIL)** 입니다 (코드상 확인, `UserService.java` 106행).

**부수효과**: `USER_TABLE.user_pw` UPDATE. 기존 액세스/리프레시 토큰은 **무효화되지 않습니다**.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-204 | 400 | 현재 비밀번호와 일치하지 않습니다 | userPw 불일치 |
| USR-217 | 400 | 현재 비밀번호와 같은 비밀번호로 변경할 수 없습니다. | newPw == 현재 비밀번호 |
| USR-203 | 400 | 비밀번호 값이 빈칸입니다 | newPw가 공백만 |
| USR-214 | 400 | 영문자,숫자,특수문자는 적어도 1개 이상씩 포함되어야합니다 | 구성 조건 미달 |
| USR-202 | 400 | 두 비밀번호가 일치하지 않습니다. | newPw ≠ confirmNewPw |
| PFL-202 | 500 | 프로필 업데이트에 실패했습니다. | 비밀번호 저장 실패 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | newPw/confirmNewPw 검증 위반 |
| AUTH_REQUIRED / TOKEN_EXPIRED / INVALID_TOKEN | 401 | 인증이 필요합니다. / 토큰이 만료되었습니다. | 액세스 토큰 없음·만료·변조 (EntryPoint 형식) |

---

### 회원 탈퇴 2단계 흐름 (14 → 15)

```
[14] POST   /users/exit/send-code   (ROLE_USER, 바디 없음)
         → WithdrawalToken 생성/갱신 + 4자리 코드 메일 발송
[15] DELETE /users/exit             (ROLE_USER, {authCode})
         → 코드 검증 → 계정·프로필·연관데이터 삭제 → 로그아웃
```

---

#### 14. 회원 탈퇴 1단계 — 인증 메일 전송
`POST /users/exit/send-code`

| 항목 | 값 |
|---|---|
| 인증 | **ROLE_USER** (`SecurityConfig`: `POST /users/exit/send-code → hasRole("USER")`) |
| 레이트리밋 | WITHDRAWAL_EMAIL 5회/5분 — 버킷 키 `{로그인 userUUID}:WITHDRAWAL_EMAIL` (`RateLimiterAspect`가 SecurityContext에서 조회) |
| Content-Type | — (요청 바디 없음) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |

바디 없음. 대상 회원은 액세스 토큰에서 식별합니다.

**응답** — `200 OK`

```json
{ "message": "탈퇴를 위한 인증 메일이 전송 되었습니다" }
```

**비즈니스 규칙**
1. SecurityContext에서 로그인 User 획득.
2. `WithdrawalToken`이 이미 있으면 **코드만 갱신**, 없으면 신규 생성. 코드는 `Random` 4자리 숫자 문자열.
3. 탈퇴 인증 메일 발송.
- **WithdrawalToken에도 만료시간 필드가 없습니다** (`WithdrawalToken.java`).

**부수효과**
- `WITHDRAWAL_TOKEN` INSERT/UPDATE.
- 메일 발송(`@Async`): 수신 `{email}@suwon.ac.kr`, 제목 "회원 탈퇴를 위한 인증 메일 입니다", 본문 "인증 코드는  {4자리} 입니다."

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| EML-501 | 500 | 메일 전송에 실패했습니다. | 메일 생성 실패 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 동일 회원이 5분 내 6회 요청 |
| AUTH_REQUIRED / TOKEN_EXPIRED / INVALID_TOKEN | 401 | 인증이 필요합니다. / 토큰이 만료되었습니다. | 액세스 토큰 문제 (EntryPoint 형식) |

---

#### 15. 회원 탈퇴 2단계 — 코드 확인 및 탈퇴 실행
`DELETE /users/exit`

| 항목 | 값 |
|---|---|
| 인증 | **ROLE_USER** (`SecurityConfig`: `DELETE /users/exit → hasRole("USER")`) |
| 레이트리밋 | WITHDRAWAL_CODE 5회/5분 — 버킷 키 `{로그인 userUUID}:WITHDRAWAL_CODE` |
| Content-Type | application/json |

> **DELETE 메서드에 요청 바디가 필요합니다.** 일부 HTTP 클라이언트는 DELETE 바디를 지원하지 않으니 확인 필요.

**요청 — 헤더/쿠키**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 쿠키 | refreshToken | String | **사실상 O** | 이 값이 없으면 계정이 삭제되지 않고 로그아웃만 수행됨 (아래 규칙 4 참조) |

**요청 — 바디** `AuthCodeRequest` (`@Valid`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| authCode | String | O | `@NotBlank` ("인증 코드를 입력해주세요.") | 14단계 메일의 4자리 코드 |

```json
{ "authCode": "8130" }
```

**응답** — `200 OK`

```json
{ "message": "회원 탈퇴가 완료되었습니다." }
```

응답 헤더: `Set-Cookie: refreshToken=; Path=/; HttpOnly; Max-Age=0; Expires=Thu, 01 Jan 1970 00:00:00 GMT; SameSite=None; Secure` (쿠키 삭제)

**비즈니스 규칙** (컨트롤러 실행 순서 그대로)
1. `WithdrawalTokenService.verifyWithdrawalToken`: 로그인 회원의 WithdrawalToken 조회(없으면 `WT-102`) → 코드 `equals` 비교(불일치 `WT-101`).
2. WithdrawalToken 삭제.
3. 해당 회원의 AuthToken이 남아 있으면 함께 삭제 (`authTokenService.delete`, 없으면 무시).
4. `UserService.cancelMembership`:
   - 쿠키에서 refreshToken 추출. **null이면 `logout()`만 호출하고 즉시 return → 회원 데이터는 삭제되지 않습니다** (이미 1~3단계에서 탈퇴/인증 토큰은 지워진 상태). 코드상 확인된 동작.
   - refreshToken이 있으면 Redis 검증 → userUUID 추출 → Profile의 `fcmToken`을 null로 설정 → `ProfileService.deleteProfileByUserUUID`(Aplict, ClubMembers 연관 데이터 삭제 후 Profile 삭제) → `USER_TABLE` DELETE.
   - Redis 검증 실패(`TokenException`) 시 삭제 없이 무시.
   - `finally`에서 항상 `logout()` 호출.
5. `logout()`: Redis 리프레시 토큰 삭제, `SecurityContextHolder.clearContext()`, refreshToken 쿠키 만료 처리.

**부수효과**
- `WITHDRAWAL_TOKEN` DELETE, `AUTH_TOKEN_TABLE` DELETE(존재 시).
- `APLICT`·`CLUB_MEMBERS`·`PROFILE`·`USER_TABLE` DELETE.
- Profile FCM 토큰 null → 푸시 알림 무효화.
- Redis `refreshToken:*` 키 삭제.
- 응답에 쿠키 삭제 `Set-Cookie` 헤더.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| WT-102 | 400 | 탈퇴 토큰이 존재하지 않습니다 | 14단계 미수행 또는 이미 사용됨 |
| WT-101 | 400 | 인증번호가 일치하지 않습니다 | authCode 불일치 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 동일 회원이 5분 내 6회 시도 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | authCode 공백 |
| USR-201 | 400 | 사용자가 존재하지 않습니다. | Profile 삭제 단계에서 Profile 미존재 |
| AUTH_REQUIRED / TOKEN_EXPIRED / INVALID_TOKEN | 401 | 인증이 필요합니다. / 토큰이 만료되었습니다. | 액세스 토큰 문제 (EntryPoint 형식) |

---

#### 16. 통합 로그아웃
`POST /integration/logout`

| 항목 | 값 |
|---|---|
| 인증 | permitAll (`/integration/**`가 permit-all-paths에 포함 → JwtFilter도 통과). **액세스 토큰 없이 호출 가능** |
| 레이트리밋 | 없음 |
| Content-Type | — (요청 바디 없음) |

> User / Admin / Leader 공통 로그아웃입니다.

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 쿠키 | refreshToken | String | X | 없어도 200 반환. 있으면 Redis에서 해당 세션 제거 |

바디 없음.

**응답** — `200 OK`

```json
{ "message": "로그아웃 성공" }
```

응답 헤더: `Set-Cookie: refreshToken=; Path=/; HttpOnly; Max-Age=0; Expires=Thu, 01 Jan 1970 00:00:00 GMT; SameSite=None; Secure`

**비즈니스 규칙** (`IntegrationAuthService.logout`)
1. 쿠키에서 refreshToken 추출. null이면 3번으로 건너뜀.
2. Redis에 존재하면 → userUUID 추출 → 해당 Profile의 `fcmToken`을 null로(User 계정일 때만 Profile이 존재) → Redis 리프레시 토큰 삭제. 검증 실패(`TokenException`)는 무시.
3. `SecurityContextHolder.clearContext()`.
4. refreshToken 쿠키 만료 처리.
- **어떤 경우에도 실패하지 않고 200을 반환합니다.**
- 액세스 토큰은 서버에서 무효화되지 않으므로(블랙리스트 없음), **클라이언트가 저장된 accessToken을 직접 폐기해야 합니다.**

**부수효과**: Redis 키 삭제, Profile FCM 토큰 null UPDATE, 쿠키 삭제 헤더.

**에러**: 정상 경로에서는 없음.

---

#### 17. 토큰 재발급
`POST /integration/refresh-token`

| 항목 | 값 |
|---|---|
| 인증 | permitAll (`/integration/**`). **만료된 액세스 토큰 상태에서도 호출 가능** |
| 레이트리밋 | 없음 |
| Content-Type | — (요청 바디 없음) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 쿠키 | refreshToken | String | O | 로그인 시 발급된 리프레시 토큰 쿠키. **요청 바디나 헤더로는 받지 않습니다** |

바디 없음.

**응답 — 성공** `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "새로운 엑세스 토큰과 리프레시 토큰이 발급됐습니다. 로그인됐습니다." |
| data.accessToken | String | 새 액세스 토큰(30분) |
| data.refreshToken | String | 새 리프레시 토큰(Redis TTL 7일) |

응답 헤더: `Authorization: Bearer {새 accessToken}`, `Set-Cookie: refreshToken={새 값}; Path=/; HttpOnly; Max-Age=604800; SameSite=None; Secure`

```json
{
  "message": "새로운 엑세스 토큰과 리프레시 토큰이 발급됐습니다. 로그인됐습니다.",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhYWFhYWFhYS0xMTExLTIyMjItMzMzMy00NDQ0NDQ0NDQ0NDQifQ.signature",
    "refreshToken": "c1d2e3f4-a5b6-4789-9012-3456789abcde"
  }
}
```

**응답 — 실패** `401 Unauthorized` (ErrorResponse가 아닌 **ApiResponse 형식**)

```json
{ "message": "리프레시 토큰이 유효하지 않습니다. 로그아웃됐습니다." }
```
`new ApiResponse<>(message, null)` 이므로 `data` 키는 직렬화에서 제외됩니다. 이때 서버는 **로그아웃까지 수행**(Redis 삭제 시도 + 쿠키 만료)하므로 클라이언트는 재로그인 화면으로 보내야 합니다.

**비즈니스 규칙** (`IntegrationAuthService.refreshToken`)
1. 쿠키에서 refreshToken 추출. **null → `logout()` 수행 후 401 반환**.
2. Redis에 `refreshToken:{값}` 존재 확인. 없으면 `TokenException` → `logout()` 수행 후 401 반환.
3. userUUID 추출 → **기존 리프레시 토큰 Redis에서 삭제**(회전, rotation).
4. 새 액세스 토큰 생성(`Authorization` 헤더 세팅) + 새 리프레시 토큰 생성(Redis 저장 + 쿠키 세팅).
- Role 구분 없이 User/Admin/Leader 모두 동일하게 처리되며, `UserDetailsServiceManager`가 UUID로 역할을 판별합니다.

**부수효과**: Redis 기존 키 삭제 + 신규 키 저장(TTL 7일), 응답 `Authorization` 헤더 및 `Set-Cookie` 설정.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| (ApiResponse 형식, code 없음) | 401 | 리프레시 토큰이 유효하지 않습니다. 로그아웃됐습니다. | 쿠키 없음 / Redis에 토큰 없음(만료·이미 회전됨·로그아웃됨) |
| TOK-202 | 401 | 유효하지 않은 토큰입니다. | `getUUIDFromRefreshToken`이 `UserException(INVALID_TOKEN)`을 던지는 경합 상황 (`catch (TokenException)`에 안 걸려 GlobalExceptionHandler로 전파) |
| USR-201 | 400 | 사용자가 존재하지 않습니다. | Redis에는 토큰이 남아 있으나 해당 UUID의 User/Admin/Leader가 이미 삭제됨 |

---

### 부록 A. 인증 요구사항 요약

| # | 엔드포인트 | 인증 | 근거 |
|---|---|---|---|
| 1 | POST /users/temporary/register | permitAll | yml permit-all-paths |
| 2 | GET /users/email/verify-token | permitAll | yml |
| 3 | POST /users/email/verification | permitAll | yml |
| 4 | POST /users/signup | permitAll | yml |
| 5 | GET /users/verify-duplicate/{account} | permitAll | yml |
| 6 | POST /users/check/{email}/duplicate | permitAll | yml |
| 7 | POST /users/existing/register | permitAll | yml |
| 8 | POST /users/login | permitAll | yml |
| 9 | GET /users/find-account/{email} | permitAll | yml |
| 10 | POST /users/auth/send-code | permitAll | yml |
| 11 | POST /users/auth/verify-token | permitAll | yml |
| 12 | PATCH /users/reset-password | permitAll | yml |
| 13 | PATCH /users/userpw | ROLE_USER | SecurityConfig 69행 |
| 14 | POST /users/exit/send-code | ROLE_USER | SecurityConfig 72행 |
| 15 | DELETE /users/exit | ROLE_USER | SecurityConfig 71행 |
| 16 | POST /integration/logout | permitAll | yml `/integration/**` |
| 17 | POST /integration/refresh-token | permitAll | yml `/integration/**` |

### 부록 B. 레이트리밋 요약

| 액션 | 적용 위치 | 제한 | 버킷 키(clientId) |
|---|---|---|---|
| EMAIL_VERIFICATION | `UserService.sendSignUpMail` (엔드포인트 1) | 5회/5분 | **빈 문자열 → 전역 공유** |
| ID_FOUND_EMAIL | `UserService.sendAccountInfoMail` (엔드포인트 9) | 5회/5분 | 조회된 User의 email |
| PW_FOUND_EMAIL | 컨트롤러 `sendAuthCode` (엔드포인트 10) | 5회/5분 | **빈 문자열 → 전역 공유** |
| VALIDATE_CODE | 컨트롤러 `verifyAuthToken` (엔드포인트 11) | 5회/5분 | `uuid` 헤더 값 |
| APP_LOGIN | 컨트롤러 `userLogin` (엔드포인트 8) | 5회/5분 | `account` |
| WITHDRAWAL_EMAIL | 컨트롤러 `sendWithdrawalCode` (엔드포인트 14) | 5회/5분 | 로그인 userUUID |
| WITHDRAWAL_CODE | 컨트롤러 `cancelMembership` (엔드포인트 15) | 5회/5분 | 로그인 userUUID |

모두 `Bandwidth.classic(5, Refill.intervally(5, Duration.ofMinutes(5)))` — 최대 5개 토큰, 5분마다 5개 일괄 충전. 버킷은 Redis(Bucket4j Lettuce)에 저장되며 Redis 만료 전략은 최대 충전 시간 기준 65초입니다.

### 부록 C. 프론트 구현 시 특히 주의할 점

1. **`emailToken_uuid` vs `emailTokenUUID`** — 1번 응답만 스네이크 혼용 표기(`VerifyEmailResponse.emailToken_uuid`), 3번 응답은 `emailTokenUUID`. 필드명을 그대로 매핑해야 합니다.
2. **email은 로컬파트만** — 전 구간에서 `@suwon.ac.kr` 없이 전달합니다(서버가 붙임). 최대 30자.
3. **회원가입 4번은 바디에 email이 없습니다** — 이메일은 `emailTokenUUID` 헤더로 서버가 역추적합니다.
4. **EmailToken 만료는 5분**, AuthToken·WithdrawalToken은 **만료 개념이 없습니다**(필드 부재).
5. **`/users/reset-password`는 Bean Validation이 걸려 있지 않습니다** — 8~20자 제한이 서버에서 강제되지 않으므로 클라이언트에서 반드시 길이를 막아야 합니다.
6. **`/users/auth/send-code`의 `userAccount` 검증도 실제로는 동작하지 않습니다** (`@Valid`+groups 지정 조합). 형식 오류는 `USR-209`로 되돌아옵니다.
7. **로그인 실패 4분기**: `USR-216`(임시회원) / `USR-220`(존재하지 않는 계정) / `USR-211`(비밀번호 불일치) / `PFL-201`(프로필 없음). UX 문구를 분기하려면 이 코드를 사용하세요.
8. **로그인은 단일 세션** — 새 로그인 시 이전 리프레시 토큰이 Redis에서 삭제됩니다.
9. **비밀번호 정규식에 `~`, `` ` `` 가 없습니다** — DTO 패턴과 `PasswordService` 패턴이 다릅니다. 클라이언트 검증은 DTO 패턴 기준으로 맞추세요.
10. **회원 탈퇴(15번)는 refreshToken 쿠키가 반드시 필요합니다** — 쿠키가 없으면 200을 받더라도 계정이 실제로 삭제되지 않습니다.
11. **로그아웃/재발급은 쿠키 기반** — 크로스 오리진에서 호출 시 `credentials: 'include'`(fetch) / `withCredentials: true`(axios)가 필수이며, prod는 `SameSite=None; Secure`이므로 HTTPS에서만 동작합니다.
12. **인증 실패 응답 형식이 두 가지** — JwtFilter 단계는 `{status, errorCode, message}`, 서비스 단계 예외는 `{exception, code, message, status, error, additionalData}`. 파서를 양쪽 모두 대응시켜야 합니다.

---

## 4. 사용자 API — 마이페이지 · 프로필 · 지원 · 알림 · 이벤트

### 공통 사항

**인증 방식**
- 헤더: `Authorization: Bearer {accessToken}` (`JwtProvider.resolveAccessToken` — `Bearer ` 접두사 필수)
- `SecurityConfig`는 `application-security.yml`의 `permit-all-paths`를 **가장 먼저** 등록하므로, 이후의 `hasRole(...)` 규칙과 겹치면 **permitAll이 우선** 적용된다. `JwtFilter`도 동일 목록을 `AntPathMatcher`로 검사해 토큰 검증 자체를 건너뛴다.
- `security` 프로파일은 `local/prod/test` 프로파일 그룹에 모두 포함되어 항상 활성화된다 (`/home/jovyan/work/USW-Circle-Link-Server/src/main/resources/application.yml`).

**성공 응답 래퍼** (`/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/response/ApiResponse.java`)
```json
{ "message": "...", "data": { } }
```
`data`에는 `@JsonInclude(NON_NULL)`이 걸려 있어 **null이면 키 자체가 생략**된다.

**에러 응답 본문** (`ErrorResponse`, `GlobalExceptionHandler`)
```json
{ "exception": "ProfileException", "code": "PFL-201", "message": "프로필이 존재하지 않습니다.", "status": 404, "error": "Not Found", "additionalData": null }
```

**공통 에러 (모든 인증 필요 엔드포인트)**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| `AUTH_REQUIRED` | 401 | 인증이 필요합니다. | 토큰 없음/누락 |
| `TOKEN_EXPIRED` | 401 | 토큰이 만료되었습니다. | 액세스 토큰 만료 |
| `INVALID_TOKEN` | 401 | 인증이 필요합니다. | 변조·형식오류 토큰 |
| (본문 없음) | 403 | — | 토큰은 유효하나 역할 불일치 |

※ 401 응답만 `{ "status", "errorCode", "message" }` 형태로 `CustomAuthenticationEntryPoint`가 직접 작성하며, 위의 `ErrorResponse` 구조와 다르다.

**공통 검증 에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| `INVALID_ARGUMENT` | 400 | 입력 값 검증에 실패했습니다. | `@Validated`/`@Valid` 실패. `additionalData`에 `{필드명: 메시지}` 맵 |
| `INVALID_UUID_FORMAT` | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | 경로변수 UUID 파싱 실패 |
| `INVALID_REQUEST_BODY` | 400 | 유효하지 않은 JSON 혹은 ENUM 값입니다. 올바른 값을 입력하세요. | JSON 파싱 실패 |

**검증 순서** — `ValidationSequence` = `Default → NotBlankGroup → SizeGroup → PatternGroup`. 앞 그룹에서 하나라도 실패하면 뒤 그룹은 실행되지 않는다(그룹 단위 단축 평가).

**Presigned URL** — 모든 S3 사진 URL은 `S3FileUploadService.generatePresignedGetUrl`로 생성되며 **유효기간 1시간**(`URL_EXPIRED_TIME = 1000*60*60`). S3 키가 null/빈 문자열이면 `""`(빈 문자열)을 반환한다.

---

#### 1. 소속된 동아리 조회
`GET /mypages/my-clubs`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: GET `/mypages/my-clubs` → `hasRole("USER")`) |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청**
- 경로변수/쿼리/헤더: 없음 (`Authorization` 헤더만 필요)

**응답** — `ApiResponse<List<MyClubResponse>>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"소속된 동아리 목록 조회 성공"` |
| data[].clubUUID | UUID | 동아리 UUID |
| data[].mainPhotoPath | String \| null | 동아리 메인 사진 presigned URL. 사진 레코드가 없으면 `null` |
| data[].clubName | String | 동아리명 |
| data[].leaderName | String | 회장 이름 |
| data[].leaderHp | String | 회장 전화번호 |
| data[].clubInsta | String | 동아리 인스타그램 |
| data[].clubRoomNumber | String | 동아리방 호수 |

```json
{
  "message": "소속된 동아리 목록 조회 성공",
  "data": [
    {
      "clubUUID": "3f2a9c1e-0000-4a11-9c31-1a2b3c4d5e6f",
      "mainPhotoPath": "https://bucket.s3.ap-northeast-2.amazonaws.com/club/main/xxx.jpg?X-Amz-Expires=3600&...",
      "clubName": "동구라미",
      "leaderName": "홍길동",
      "leaderHp": "01012345678",
      "clubInsta": "@donggurami",
      "clubRoomNumber": "B101"
    }
  ]
}
```
HTTP 200

**비즈니스 규칙**
1. `SecurityContextHolder`에서 `CustomUserDetails → User` 추출.
2. `profileRepository.findByUserUserId(userId)` — 없으면 `PROFILE_NOT_EXISTS`.
3. `clubMembersRepository.findByProfileProfileId(profileId)` — 결과가 없으면 **빈 배열 반환**(예외 아님).
4. 각 동아리마다 `ClubMainPhoto` 조회 후 presigned URL 생성. 사진 엔티티가 없으면 `mainPhotoPath = null`.
- 부수효과: 없음(S3 presigned URL 생성만).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| PFL-201 | 404 | 프로필이 존재하지 않습니다. | 로그인 사용자의 프로필 미생성 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | S3 presigned URL 생성 중 SDK 오류 |

---

#### 2. 지원한 동아리(지원내역) 조회
`GET /mypages/aplict-clubs`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: GET `/mypages/aplict-clubs`) |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청** — 없음

**응답** — `ApiResponse<List<MyAplictResponse>>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | `"지원한 동아리 목록 조회 성공"` |
| data[].clubUUID | UUID | 동아리 UUID |
| data[].mainPhotoPath | String \| null | 메인 사진 presigned URL(1시간) |
| data[].clubName | String | 동아리명 |
| data[].leaderName | String | 회장 이름 |
| data[].leaderHp | String | 회장 전화번호 |
| data[].clubInsta | String | 인스타그램 |
| data[].clubRoomNumber | String | 동아리방 호수. **DTO 필드명은 `ClubRoomNumber`(대문자 C)지만 Lombok 게터 `getClubRoomNumber()` 기준으로 직렬화되므로 JSON 키는 `clubRoomNumber`** |
| data[].aplictStatus | Enum | `WAIT` \| `PASS` \| `FAIL` |

```json
{
  "message": "지원한 동아리 목록 조회 성공",
  "data": [
    {
      "clubUUID": "3f2a9c1e-0000-4a11-9c31-1a2b3c4d5e6f",
      "mainPhotoPath": "https://bucket.s3.ap-northeast-2.amazonaws.com/club/main/xxx.jpg?X-Amz-Expires=3600&...",
      "clubName": "동구라미",
      "leaderName": "홍길동",
      "leaderHp": "01012345678",
      "clubInsta": "@donggurami",
      "clubRoomNumber": "B101",
      "aplictStatus": "WAIT"
    }
  ]
}
```
HTTP 200

**비즈니스 규칙 — 상태(WAIT/PASS/FAIL)와 checked/deleteDate**
1. 프로필 조회 → 없으면 `PROFILE_NOT_EXISTS`.
2. `aplictRepository.findByProfileProfileId(profileId)`로 **모든 지원서**를 조회한다. `checked`/`deleteDate` 값에 따른 필터링은 **하지 않는다**.
3. 지원서 ID로 동아리를 다시 조회(`findClubByAplictId`) — 없으면 `CLUB_NOT_EXISTS`.
4. 응답에는 `aplictStatus`만 노출된다. **`checked`와 `deleteDate`는 응답 DTO(`MyAplictResponse`)에 포함되지 않는다.**
5. 상태 전이(동아리 회장 측 로직, `ClubLeaderService` 741~747행): 합/불 통보 시 `updateAplictStatus(PASS|FAIL, checked=true, deleteDate = now + 4일)`. 즉 **PASS/FAIL이 되는 순간 `checked=true`, `deleteDate`가 4일 뒤로 설정**된다.
6. `SchedulerConfig.deleteOldApplications()`(매일 00시 cron `0 0 0 * * ?`)가 `deleteDate < now`인 지원서를 삭제한다. → **합/불 통보 4일 후 해당 지원내역은 이 목록에서 사라진다.**
7. `Aplict`는 `uk_aplict_active (club_id, profile_id, aplict_checked)` 유니크 제약이 있어, `checked=false`인 미통보 지원서는 동아리당 1건만 존재할 수 있다.
- 부수효과: 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| PFL-201 | 404 | 프로필이 존재하지 않습니다. | 프로필 미존재 |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 지원서에 연결된 동아리 조회 실패 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 오류 |

---

#### 3. 동아리방 층별 도면(사진) 조회
`GET /mypages/clubs/{floor}/photo`

| 항목 | 값 |
|---|---|
| 인증 | **permitAll** (`application-security.yml`의 `/mypages/clubs/{floor}/photo`) — 토큰 불필요 |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | floor | String(Enum) | O | `FloorPhotoEnum` 값: `B1`, `F1`, `F2` (대소문자 구분, `valueOf` 사용) |

**응답** — `ApiResponse<ClubFloorPhotoResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | `"동아리방 층별 사진 조회 성공"` |
| data.roomFloor | Enum | `B1` \| `F1` \| `F2` |
| data.floorPhotoPath | String | 층별 도면 presigned URL(1시간) |

```json
{
  "message": "동아리방 층별 사진 조회 성공",
  "data": {
    "roomFloor": "B1",
    "floorPhotoPath": "https://bucket.s3.ap-northeast-2.amazonaws.com/floor/b1.png?X-Amz-Expires=3600&..."
  }
}
```
HTTP 200

**비즈니스 규칙**
1. `FloorPhotoEnum.valueOf(floor)` 시도 → `IllegalArgumentException` 시 `INVALID_ENUM_VALUE`.
2. `floorPhotoRepository.findByFloor(enum)` → 없으면 `PHOTO_NOT_FOUND`.
3. `floorPhotoS3key`로 presigned URL 생성.
- 부수효과: 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| ENUM-401 | 400 | 유효하지 않은 Enum 값입니다. | `floor`가 B1/F1/F2 외의 값 |
| PHOTO-505 | 404 | 해당 사진이 존재하지 않습니다. | 해당 층 도면 미등록 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 오류 |

---

#### 4. 내 프로필 조회
`GET /profiles/me`

> **경로 주의**: `ProfileController`의 `@RequestMapping("profiles")`에는 선행 슬래시가 **누락되어 있다**. 다만 Spring MVC(`PathPatternsRequestCondition`/`PatternsRequestCondition`)가 매핑 등록 시 선행 슬래시를 자동으로 붙이므로 **실제 호출 경로는 `/profiles/me`** 이다. (`SecurityConfig`·`application-security.yml`도 `/profiles/...` 기준으로 작성되어 있어 정상 동작한다.)

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: GET `/profiles/me`) |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청** — 없음

**응답** — `ApiResponse<ProfileResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | `"프로필 조회 성공"` |
| data.userName | String | 이름 |
| data.studentNumber | String | 학번(8자리) |
| data.userHp | String | 전화번호(11자리, 하이픈 없음) |
| data.major | String | 학과 |

```json
{
  "message": "프로필 조회 성공",
  "data": {
    "userName": "김수원",
    "studentNumber": "20231234",
    "userHp": "01012345678",
    "major": "컴퓨터학부"
  }
}
```
HTTP 200

**비즈니스 규칙**
1. SecurityContext의 `User.userId`로 프로필 조회 → 없으면 `PROFILE_NOT_EXISTS`.
- 부수효과: 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| PFL-201 | 404 | 프로필이 존재하지 않습니다. | 프로필 미존재 |

---

#### 5. 프로필 수정
`PATCH /profiles/change`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: PATCH `/profiles/change`) |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청 바디** — `ProfileRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| userPw | String | O | `@NotBlank` (그룹 미지정 → Default 그룹, 시퀀스상 **가장 먼저** 검증). 메시지는 Bean Validation 기본 메시지 | **현재** 비밀번호(본인 확인용) |
| userName | String | O | `@NotBlank` "이름은 필수 입력 값입니다." / `@Size(min=2,max=30)` "이름은 2~30자 이내여야 합니다." / `@Pattern(^[a-zA-Z가-힣]+$)` "이름은 영어 또는 한글만 입력 가능합니다." | 이름 |
| studentNumber | String | O | `@NotBlank` / `@Size(min=8,max=8)` "학번은 8자리 숫자여야 합니다." / `@Pattern(^[0-9]{8}$)` | 학번 |
| userHp | String | O | `@NotBlank` / `@Size(min=11,max=11)` "전화번호는 11자여야 합니다." / `@Pattern(^01[0-9]{9}$)` "올바른 전화번호를 입력하세요." | 전화번호(하이픈 없음) |
| major | String | O | `@NotBlank` "학과는 필수 입력 값입니다." / `@Size(min=1,max=20)` / `@Pattern(^[가-힣a-zA-Z]+$)` "학과는 한글 또는 영어만 입력 가능합니다." | 학과 |

```json
{
  "userPw": "Test1234!@",
  "userName": "김수원",
  "studentNumber": "20231234",
  "userHp": "01012345678",
  "major": "컴퓨터학부"
}
```

**응답** — `ApiResponse<ProfileResponse>` (필드는 4번과 동일)
```json
{
  "message": "프로필 수정 성공",
  "data": {
    "userName": "김수원",
    "studentNumber": "20231234",
    "userHp": "01098765432",
    "major": "정보보호학과"
  }
}
```
HTTP 200

**비즈니스 규칙 (실행 순서)**
1. Bean Validation(`@Validated(ValidationSequence.class)`) → 실패 시 400 `INVALID_ARGUMENT`.
2. `confirmPW`: `passwordEncoder.matches(userPw, 로그인 사용자의 userPw)` → 불일치 시 `USER_PASSWORD_NOT_MATCH`.
3. `validateProfileRequest`: userName/studentNumber/userHp/major 중 null 또는 공백이면 `PROFILE_NOT_INPUT` (Bean Validation과 중복되는 방어 로직).
4. 프로필 조회 → 없으면 `PROFILE_NOT_EXISTS`.
5. `profile.updateProfile(userName, studentNumber, major, userHp)` 후 `save`. 저장 결과가 null이면 `PROFILE_UPDATE_FAIL` (JPA `save`는 null을 반환하지 않으므로 실질적으로 도달 불가).
6. **수정 시에는 이름/학번/전화번호 중복 검사를 하지 않는다** (`checkProfileDuplicated`는 이 흐름에서 호출되지 않음).
- 부수효과: DB 프로필 UPDATE. 메일/푸시/S3/Redis/쿠키 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-204 | 400 | 현재 비밀번호와 일치하지 않습니다 | `userPw` 불일치 |
| PFL-203 | 400 | 프로필 입력값은 필수입니다. | 4개 필드 중 공백/null |
| PFL-201 | 404 | 프로필이 존재하지 않습니다. | 프로필 미존재 |
| PFL-202 | 500 | 프로필 업데이트에 실패했습니다. | 저장 결과 null (사실상 미발생) |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | 검증 애노테이션 위반 |

---

#### 6. 프로필 중복 확인
`POST /profiles/duplication-check`

| 항목 | 값 |
|---|---|
| 인증 | **permitAll** (`application-security.yml`의 `/profiles/duplication-check`) — 토큰 불필요 |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청 바디** — `ProfileDuplicationCheckRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| userName | String | O | `@NotBlank` / `@Size(min=2,max=30)` / `@Pattern(^[a-zA-Z가-힣]+$)` | 이름 |
| studentNumber | String | O | `@NotBlank` / `@Size(min=8,max=8)` / `@Pattern(^[0-9]{8}$)` | 학번 |
| userHp | String | O | `@NotBlank` / `@Size(min=11,max=11)` / `@Pattern(^01[0-9]{9}$)` | 전화번호 |
| clubUUID | UUID | X | 검증 애노테이션 없음(null 허용) | 비교 대상 동아리 UUID. null이면 `inTargetClub`은 항상 false |

```json
{
  "userName": "김수원",
  "studentNumber": "20231234",
  "userHp": "01012345678",
  "clubUUID": "3f2a9c1e-0000-4a11-9c31-1a2b3c4d5e6f"
}
```

**응답** — `ApiResponse<ProfileDuplicationCheckResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"프로필 중복 확인 결과"` (중복 여부와 무관) |
| data.exists | boolean | 이름+학번+전화번호가 모두 일치하는 프로필 존재 여부 |
| data.classification | String | `NOT_FOUND` / `NO_CLUB` / `SAME_CLUB` / `OTHER_CLUB` |
| data.inTargetClub | boolean | 요청한 `clubUUID`의 회원인지 여부 |
| data.clubUUIDs | UUID[] | 해당 프로필이 소속된 모든 동아리 UUID 목록 (미존재 시 `[]`) |
| data.targetClubUUID | UUID \| null | 요청에 들어온 `clubUUID`를 그대로 echo |
| data.profileId | Long \| null | 매칭된 프로필 PK. 미존재 시 `null`(→ `@JsonInclude`는 `ApiResponse.data`에만 적용되므로 이 필드는 `null`로 출력) |

**classification 분류 규칙 (`ProfileService.checkProfileDuplication`)**

| 값 | 조건 | exists | inTargetClub | clubUUIDs |
|---|---|---|---|---|
| `NOT_FOUND` | 이름+학번+전화번호 일치 프로필 **없음** | false | false | `[]` |
| `SAME_CLUB` | 프로필 존재 && `clubUUID != null` && 소속 목록에 `clubUUID` 포함 | true | true | 소속 목록 |
| `NO_CLUB` | 프로필 존재 && 타깃 동아리 아님 && 소속 동아리 **하나도 없음** | true | false | `[]` |
| `OTHER_CLUB` | 프로필 존재 && 타깃 동아리 아님 && 다른 동아리에 소속됨 | true | false | 소속 목록 |

```json
{
  "message": "프로필 중복 확인 결과",
  "data": {
    "exists": true,
    "classification": "OTHER_CLUB",
    "inTargetClub": false,
    "clubUUIDs": ["11111111-2222-3333-4444-555555555555"],
    "targetClubUUID": "3f2a9c1e-0000-4a11-9c31-1a2b3c4d5e6f",
    "profileId": 42
  }
}
```
HTTP 200 (중복이어도 200, 예외 아님)

**비즈니스 규칙**
1. Bean Validation(`ValidationSequence`) 통과 후 `profileRepository.findByUserNameAndStudentNumberAndUserHp(3개 완전 일치)` 조회.
2. 미존재 → `NOT_FOUND` 응답(200).
3. 존재 → `clubMembersRepository.findClubUUIDsByProfileId`로 소속 동아리 UUID 목록 조회 후 위 표대로 분류.
4. 이 API는 **예외를 던지지 않는다**. (별도 메서드 `checkProfileDuplicated`가 던지는 `PROFILE_ALREADY_EXISTS`(PFL-207)는 이 엔드포인트에서 호출되지 않는다.)
5. 인증이 없으므로 `profileId` 등 내부 식별자가 비인증 클라이언트에 노출된다.
- 부수효과: 없음(조회 전용).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | 이름/학번/전화번호 검증 위반 |
| INVALID_REQUEST_BODY | 400 | 유효하지 않은 JSON 혹은 ENUM 값입니다… | `clubUUID`가 UUID 형식이 아닐 때 등 본문 파싱 실패 |

---

#### 7. 공지사항 목록 조회
`GET /my-notices`

| 항목 | 값 |
|---|---|
| 인증 | **permitAll** — `application-security.yml`에 `/my-notices/**`가 등록되어 있고 permitAll이 먼저 등록되므로, `SecurityConfig`의 `GET /my-notices → hasRole("USER")` 규칙보다 우선한다. (AntPathMatcher에서 `/my-notices/**`는 `/my-notices` 자체도 매칭) |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청** — 없음 (페이징/정렬 파라미터 없음)

**응답** — `ApiResponse<List<MyNoticeResponse>>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | `"공지사항 조회 성공"` |
| data[].noticeUUID | UUID | 공지 UUID (상세 조회 키) |
| data[].noticeTitle | String | 제목 |
| data[].adminName | String | 작성 관리자 이름 |
| data[].noticeCreatedAt | LocalDateTime | 작성일시 (`2026-08-15T10:30:00` 형식) |

```json
{
  "message": "공지사항 조회 성공",
  "data": [
    {
      "noticeUUID": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "noticeTitle": "2026학년도 동아리 모집 안내",
      "adminName": "관리자",
      "noticeCreatedAt": "2026-08-15T10:30:00"
    }
  ]
}
```
HTTP 200 (`ResponseEntity.ok`)

**비즈니스 규칙**
1. `noticeRepository.findAll()` — **전체 조회, 페이징·정렬 없음**(DB 반환 순서에 의존).
2. 각 공지의 `admin.adminName`을 함께 매핑.
3. 목록에는 본문/사진이 포함되지 않는다.
- 부수효과: 없음.

**에러** — 서비스 로직상 던지는 도메인 예외 없음.

---

#### 8. 공지사항 상세 조회
`GET /my-notices/{noticeUUID}/details`

| 항목 | 값 |
|---|---|
| 인증 | **permitAll** (`/my-notices/**`가 permitAll로 먼저 등록됨. `SecurityConfig`의 `hasRole("USER")` 규칙은 도달하지 않음) |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | noticeUUID | UUID | O | 목록 응답의 `noticeUUID` |

**응답** — `ApiResponse<NoticeDetailResponse>` (`AdminNoticeService.getNoticeByUUID` 재사용)

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | `"공지사항 세부 조회 성공"` |
| data.noticeUUID | UUID | 공지 UUID |
| data.noticeTitle | String | 제목 |
| data.noticeContent | String | 본문 |
| data.noticePhotos | String[] | 사진 presigned URL 배열. **`NoticePhoto.order` 오름차순 정렬**, 유효기간 1시간. 사진이 없으면 `[]` |
| data.noticeCreatedAt | LocalDateTime | 작성일시 |
| data.adminName | String | 작성 관리자 이름 |

```json
{
  "message": "공지사항 세부 조회 성공",
  "data": {
    "noticeUUID": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "noticeTitle": "2026학년도 동아리 모집 안내",
    "noticeContent": "모집 기간은 ...",
    "noticePhotos": [
      "https://bucket.s3.ap-northeast-2.amazonaws.com/notice/1.png?X-Amz-Expires=3600&...",
      "https://bucket.s3.ap-northeast-2.amazonaws.com/notice/2.png?X-Amz-Expires=3600&..."
    ],
    "noticeCreatedAt": "2026-08-15T10:30:00",
    "adminName": "관리자"
  }
}
```
HTTP 200

**비즈니스 규칙**
1. `noticeRepository.findByNoticeUUID` → 없으면 `NOTICE_NOT_EXISTS`.
2. 사진을 `order` 기준 오름차순 정렬 후 각각 presigned URL 생성.
- 부수효과: 없음(조회수 증가 등 없음).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| NOT-201 | 404 | 공지사항이 존재하지 않습니다. | 해당 UUID 공지 없음 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다… | 경로변수가 UUID 형식이 아님 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 오류 |

---

#### 9. 이벤트 인증 상태 조회
`GET /users/event/status`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: GET `/users/event/**`) |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청** — 없음. (컨트롤러에 `HttpServletRequest`로 액세스 토큰의 `clubUUID`를 읽던 코드가 있으나 **주석 처리되어 비활성**이며, 현재는 서버가 DB에서 동아리를 조회한다.)

**응답** — `ApiResponse<EventStatusResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | `"이벤트 인증 상태 조회"` |
| data.verified | boolean | 해당 사용자+동아리 조합의 인증 완료 여부 |

```json
{ "message": "이벤트 인증 상태 조회", "data": { "verified": false } }
```
HTTP 200

**비즈니스 규칙**
1. `@AuthenticationPrincipal CustomUserDetails`에서 `User` 획득.
2. `profileRepository.findByUser_UserUUID` → 없으면 `INVALID_INPUT`.
3. `clubMembersRepository.findClubUUIDsByProfileId` → **빈 목록이면 `INVALID_INPUT`**(소속 동아리가 있어야 상태 조회 가능).
4. **여러 동아리에 소속돼 있어도 `clubUUIDs.get(0)`(쿼리 반환 순서의 첫 번째)만 사용**한다. 정렬 지정이 없어 어떤 동아리가 선택될지 보장되지 않는다.
5. `eventVerificationRepository.existsByUserUUIDAndClubUUID(userUUID, clubUUID)` 결과를 반환.
- 부수효과: 없음(`@Transactional(readOnly = true)`).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| COM-302 | 400 | 잘못된 입력 값입니다. | 프로필 없음 또는 소속 동아리 없음 |

---

#### 10. 이벤트 코드 검증
`POST /users/event/verify`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: POST `/users/event/**`) |
| 레이트리밋 | `EVENT_VERIFY` **5회 / 5분** (`Bandwidth.classic(5, Refill.intervally(5, Duration.ofMinutes(5)))`), 버킷 키 = `{userUUID}:EVENT_VERIFY`, Redis(Lettuce) 기반 |
| Content-Type | application/json |

**요청 바디** — `EventVerifyRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| code | String | O | `@NotBlank` (그룹 없음, `@Valid` 적용) | 이벤트 인증 코드 |

```json
{ "code": "1115" }
```

**응답** — `ApiResponse<EventVerifyResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 성공 시 `"이벤트 인증 완료"`. (`isFirstVerify=false` 분기의 `"이미 인증된 사용자입니다"` 메시지는 서비스가 재인증 시 예외를 던지므로 **현재 코드상 도달 불가**) |
| data.clubUUID | UUID | 인증에 사용된 동아리 UUID(서버가 결정) |
| data.isFirstVerify | boolean | `@JsonProperty("isFirstVerify")`. 성공 응답에서는 항상 `true` |
| data.verified_at | LocalDateTime | `@JsonProperty("verified_at")`. 인증 저장 시각 |

```json
{
  "message": "이벤트 인증 완료",
  "data": {
    "clubUUID": "3f2a9c1e-0000-4a11-9c31-1a2b3c4d5e6f",
    "isFirstVerify": true,
    "verified_at": "2026-08-15T14:05:12.345"
  }
}
```
HTTP 200

**비즈니스 규칙 (실행 순서)**
1. `@Valid` 검증(`code` 공백 불가).
2. 레이트리밋 AOP(`RateLimiterAspect`)가 먼저 동작: `EVENT_VERIFY`는 로그인 이후 액션으로 분류되어 `UserService.getUserByAuth()`의 **userUUID를 클라이언트 키**로 사용. 토큰 소진 시 `TOO_MANY_ATTEMPT`.
3. `profileRepository.findByUser_UserUUID` → 없으면 `INVALID_INPUT`.
4. `findClubUUIDsByProfileId` → 빈 목록이면 `INVALID_INPUT`. **`clubUUIDs.get(0)`만 사용**.
5. **1회 제한**: `existsByUserUUIDAndClubUUID(userUUID, clubUUID)`가 true면 `EVENT_ALREADY_VERIFIED`를 던진다. 즉 (userUUID, clubUUID) 조합당 인증은 **단 1회**만 가능하며, 재시도는 200이 아니라 400 에러가 된다. DB에도 `UK_event_user_club (user_uuid, club_uuid)` 유니크 제약이 있다.
6. **코드 검증 방식**: `code.equals(expectedEventCode)` 단순 문자열 완전 일치. 기대값은 `@Value("${event.code:1115}")` — `resources/` 하위 yml에 `event.code` 설정이 없으므로 **기본값 `"1115"`가 사용**된다(환경변수/외부 설정으로 주입 시 그 값). 불일치 시 `INVALID_EVENT_CODE`. 코드 검증은 **이미 인증된 경우 검사 이후**에 수행된다.
7. 성공 시 `EventVerification.create(userUUID, clubUUID, userId, profileId, userAccount, email)`로 `verified=true`, `verifiedAt=now()`를 **MySQL에 저장**.
- 부수효과: `event_verification_table` INSERT(사용자 계정·이메일 스냅샷 포함), Redis 레이트리밋 버킷 갱신. 메일/FCM/S3 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| EVT-102 | 400 | 이미 인증된 사용자입니다. | 동일 (user, club) 조합으로 이미 인증됨 |
| EVT-101 | 400 | 유효하지 않은 이벤트 코드입니다. | `code` 불일치 |
| COM-302 | 400 | 잘못된 입력 값입니다. | 프로필 없음 또는 소속 동아리 없음 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 5분 내 6회째 요청 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | `code` 공백/누락 |

---

#### 11. 동아리 지원 가능 여부 확인
`GET /apply/can-apply/{clubUUID}`

| 항목 | 값 |
|---|---|
| 인증 | **ROLE_USER** (`SecurityConfig`: GET `/apply/**` → `hasRole("USER")`). 컨트롤러 주석의 "(ANYONE)"과 달리 토큰이 필요하다 |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | clubUUID | UUID | O | 지원하려는 동아리 UUID |

**응답** — `ApiResponse<Boolean>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"지원 가능"` (결과가 false여도 동일) |
| data | boolean | 지원 가능 여부 |

```json
{ "message": "지원 가능", "data": true }
```
HTTP 200

**비즈니스 규칙 — 3단계 중 1단계(불리언 판정)**
`AplictService.canApply`는 아래 4가지를 순서대로 검사하며, 하나라도 걸리면 **예외 없이 `false`** 를 반환한다(제출 API와 검사 항목은 같지만 사유를 구분해 주지 않는다).
1. 인증 프로필 조회(`findByUser_UserUUID`) → **없으면 `USER_NOT_EXISTS` 예외**(이 단계만 예외).
2. 이미 지원함: `existsByProfileAndClubUUID` — **`checked = false`인 지원서만 카운트**. 즉 합/불 통보가 끝난(`checked=true`) 지원서는 재지원을 막지 않는다.
3. 이미 해당 동아리 회원: `clubMembersRepository.existsByProfileAndClubUUID`.
4. 동아리 회원 중 **전화번호**가 동일한 사람이 있음.
5. 동아리 회원 중 **학번**이 동일한 사람이 있음.
6. 모두 통과하면 `true`.
- **동아리 존재 여부는 확인하지 않는다.** 존재하지 않는 `clubUUID`를 넣으면 위 검사가 모두 무해하게 통과해 `true`가 반환된다(실제 제출 시 `CLUB_NOT_EXISTS`로 실패).
- 부수효과: 없음(`readOnly`).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-201 | 400 | 사용자가 존재하지 않습니다. | 로그인 사용자의 프로필 미존재 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다… | 경로변수 UUID 파싱 실패 |

---

#### 12. 지원서(구글 폼) URL 조회
`GET /apply/{clubUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: GET `/apply/**`) |
| 레이트리밋 | 없음 |
| Content-Type | 요청 본문 없음 |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | clubUUID | UUID | O | 동아리 UUID |

**응답** — `ApiResponse<String>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | `"구글 폼 URL 조회 성공"` |
| data | String | 동아리 소개(`ClubIntro`)에 등록된 구글 폼 URL 문자열 |

```json
{ "message": "구글 폼 URL 조회 성공", "data": "https://forms.gle/exampleFormId" }
```
HTTP 200

**비즈니스 규칙 — 3단계 중 2단계**
1. `clubIntroRepository.findByClubUUID` → 없으면 `CLUB_INTRO_NOT_EXISTS`.
2. `googleFormUrl`이 null이거나 빈 문자열이면 `GOOGLE_FORM_URL_NOT_EXISTS`.
3. **이 단계에서는 지원 자격(중복 지원·회원 여부 등)을 검사하지 않는다.**
- 부수효과: 없음(`readOnly`).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CINT-201 | 404 | 해당 동아리 소개글이 존재하지 않습니다. | ClubIntro 미존재 |
| CINT-202 | 400 | 구글 폼 URL이 존재하지 않습니다. | URL이 null/빈 문자열 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다… | UUID 파싱 실패 |

---

#### 13. 동아리 지원서 제출
`POST /apply/{clubUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_USER (`SecurityConfig`: POST `/apply/**`) |
| 레이트리밋 | 없음 |
| Content-Type | **요청 바디 없음** (경로변수만 사용) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| path | clubUUID | UUID | O | 지원할 동아리 UUID |

바디 없음. (구글 폼 응답 내용은 서버로 전송되지 않으며, 서버는 "지원했다"는 사실만 기록한다.)

**응답** — `ApiResponse<Void>` — `data`는 null이므로 `@JsonInclude(NON_NULL)`에 의해 **키가 생략**된다.
```json
{ "message": "지원서 제출 성공" }
```
HTTP 200

**비즈니스 규칙 — 3단계 중 3단계(사유별 예외)**
1. 인증 프로필 조회 → 없으면 `USER_NOT_EXISTS`.
2. `checkIfCanApply(clubUUID)` — 11번과 **동일한 검사 항목이지만 실패 사유별로 서로 다른 예외**를 던진다.
   1. `checked=false`인 기존 지원서 존재 → `ALREADY_APPLIED`(APT-205)
   2. 이미 해당 동아리 회원 → `ALREADY_MEMBER`(APT-206)
   3. 동아리 회원 중 전화번호 중복 → `PHONE_NUMBER_ALREADY_REGISTERED`(APT-207)
   4. 동아리 회원 중 학번 중복 → `STUDENT_NUMBER_ALREADY_REGISTERED`(APT-208)
3. `clubRepository.findByClubUUID` → 없으면 `CLUB_NOT_EXISTS`(11번과 달리 **여기서는 동아리 존재를 검증**).
4. `Aplict` 생성: `submittedAt = now()`, `aplictStatus = WAIT`, `checked = false`(기본값), `deleteDate = null`, `aplictUUID`는 랜덤 생성.
5. 저장 중 `DataIntegrityViolationException`(유니크 제약 `uk_aplict_active` 위반, 동시 중복 제출) → `ALREADY_APPLIED`로 변환.
6. 제출 직후 상태는 `WAIT`이며, 이후 동아리 회장이 합/불을 통보하면 `PASS`/`FAIL` + `checked=true` + `deleteDate=통보시각+4일`로 갱신되고, 매일 자정 스케줄러가 `deleteDate`가 지난 지원서를 삭제한다(2번 엔드포인트 설명 참조).
- 부수효과: `APLICT_TABLE` INSERT. 제출 시점에는 메일/FCM 발송 없음(FCM은 회장의 합·불 통보 단계에서 발송).

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| APT-205 | 400 | 이미 지원한 동아리입니다. | 미통보(`checked=false`) 지원서 존재 / 유니크 제약 충돌 |
| APT-206 | 400 | 이미 해당 동아리 회원입니다. | 이미 동아리 멤버 |
| APT-207 | 400 | 이미 등록된 전화번호입니다. | 동아리 회원 중 동일 전화번호 존재 |
| APT-208 | 400 | 이미 등록된 학번입니다. | 동아리 회원 중 동일 학번 존재 |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 해당 UUID 동아리 없음 |
| USR-201 | 400 | 사용자가 존재하지 않습니다. | 프로필 미존재 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다… | UUID 파싱 실패 |

---

### 검토 시 참고한 파일 (절대경로)

- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/user/api/MypageController.java`, `.../user/service/MypageService.java`, `.../user/dto/MyClubResponse.java`, `MyAplictResponse.java`, `ClubFloorPhotoResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/profile/api/ProfileController.java`, `.../profile/service/ProfileService.java`, `.../profile/dto/ProfileRequest.java`, `ProfileResponse.java`, `ProfileDuplicationCheckRequest.java`, `ProfileDuplicationCheckResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/user/api/MyNoticeController.java`, `.../user/service/MyNoticeService.java`, `.../user/dto/MyNoticeResponse.java`, `.../admin/notice/dto/NoticeDetailResponse.java`, `.../admin/notice/service/AdminNoticeService.java`(73~92행)
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/user/api/EventVerificationController.java`, `.../user/service/EventVerificationService.java`, `.../user/domain/EventVerification.java`, `.../user/dto/EventVerifyRequest.java`, `EventStatusResponse.java`, `EventVerifyResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/aplict/api/AplictController.java`, `.../aplict/service/AplictService.java`, `.../aplict/domain/Aplict.java`, `AplictStatus.java`, `.../aplict/repository/AplictRepository.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/exception/ExceptionType.java`, `.../global/exception/GlobalExceptionHandler.java`, `.../global/response/ApiResponse.java`, `ErrorResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/security/config/SecurityConfig.java`, `.../global/security/jwt/filter/JwtFilter.java`, `.../global/security/exception/CustomAuthenticationEntryPoint.java`, `/home/jovyan/work/USW-Circle-Link-Server/src/main/resources/yaml/application-security.yml`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/bucket4j/RateLimitAction.java`, `RateLimiterAspect.java`, `APIRateLimiter.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/s3File/Service/S3FileUploadService.java`(38행 `URL_EXPIRED_TIME`), `.../global/config/SchedulerConfig.java`(50~65행), `.../clubLeader/service/ClubLeaderService.java`(725~755행), `.../global/validation/ValidationSequence.java`

### 명세 작성 중 확인된 코드상 특이사항

1. `ProfileController`의 `@RequestMapping("profiles")`는 선행 슬래시가 없으나 Spring이 매핑 등록 시 자동으로 `/`를 붙여 `/profiles/...`로 동작한다(보안 설정도 `/profiles/...` 기준이라 일치).
2. `/my-notices/**`가 `permit-all-paths`에 있어 `SecurityConfig`의 `GET /my-notices`, `GET /my-notices/{noticeUUID}/details` → `hasRole("USER")` 규칙이 **무력화**된다(설정 충돌, 실제로는 비인증 접근 가능).
3. `EventVerificationController`의 JWT `clubUUID` 추출 로직(`jwtProvider.getClubUUIDFromAccessToken`)과 `HttpServletRequest` 파라미터는 두 엔드포인트 모두 **주석 처리되어 비활성**이며, 현재는 서비스가 `clubMembersRepository.findClubUUIDsByProfileId(...).get(0)`로 동아리를 결정한다(정렬 미지정).
4. `EventVerifyResponse.isFirstVerify=false` 분기의 메시지 `"이미 인증된 사용자입니다"`는 서비스가 재인증 시 `EVENT_ALREADY_VERIFIED` 예외를 던지므로 도달할 수 없다.
5. `AplictController.canApply`의 "(ANYONE)" 주석과 달리 `SecurityConfig`상 `GET /apply/**`는 `ROLE_USER` 필요.
6. `existsByProfileAndClubUUID`가 `checked = false`만 세므로, 합/불 통보 완료된 지원서는 재지원을 차단하지 않는다.
7. `ProfileService.updateProfile`의 `PROFILE_UPDATE_FAIL`(PFL-202)은 `JpaRepository.save`가 null을 반환하지 않으므로 실질적으로 도달 불가한 방어 코드다.

---

## 5. 사용자 API — 동아리 공개 조회

**Base Path**: `/clubs`
**컨트롤러**: `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/api/ClubController.java`
**서비스**: `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/service/ClubService.java`

### 공통 사항

#### 인증
`application-security.yml`의 `permit-all-paths`에 `/clubs/**` 및 `/clubs/filter/**`가 등록되어 있습니다. `SecurityConfig`가 이 목록을 `permitAll()`로 등록하고, `JwtFilter`도 `AntPathMatcher`로 동일 목록을 매칭해 토큰 검증을 건너뜁니다. Ant/PathPattern 규칙상 `/clubs/**`는 `/clubs` 자기 자신도 매칭하므로 **7개 엔드포인트 전부 permitAll(토큰 불필요)** 입니다.

#### 레이트리밋
`ClubController`에는 `@RateLimite` 애노테이션이 **하나도 없습니다** → 전부 "없음".

#### 성공 응답 래퍼 (`ApiResponse`)
```json
{ "message": "...", "data": ... }
```
`data`는 `@JsonInclude(NON_NULL)` → null이면 필드 자체가 생략됩니다. (단, 본 7개 API는 항상 리스트/객체를 반환하므로 실제로 생략되는 경우는 없습니다.)

#### 공통 에러 응답 (`ErrorResponse`)
```json
{
  "exception": "BaseException",
  "code": "CTG-201",
  "message": "존재하지 않는 카테고리입니다.",
  "status": 404,
  "error": "Not Found",
  "additionalData": null
}
```

#### 공통 특성
- **페이징 없음**: 7개 API 모두 `Page`/`Pageable`을 사용하지 않고 전체 `List`를 한 번에 반환합니다.
- **정렬 보장 없음**: `clubRepository.findAll()`, `findByClubIds`, `findClubsByCategoryIds` 등 모든 조회 쿼리에 `ORDER BY`가 없습니다. 유일하게 정렬이 명시된 곳은 동아리 상세의 소개 사진(`ORDER BY cip.order`)뿐입니다. → 목록 순서는 DB 반환 순서에 의존하며 **코드상 정렬 기준 미지정**.
- **사진 presigned URL**: `S3FileUploadService.generatePresignedGetUrl(s3Key)`로 매 요청마다 새로 생성. `URL_EXPIRED_TIME = 1000 * 60 * 60` → **유효기간 1시간**. S3 키가 null/빈 문자열이면 `""`(빈 문자열) 반환, 사진 레코드 자체가 없으면 `null`.
- **`departmentName` 직렬화**: `Department` enum에 `@JsonValue`로 한글 값("학술" 등)이 지정되어 있으나, **`ClubListResponse.departmentName`은 `String` 타입이고 서비스에서 `club.getDepartment().name()`을 넣습니다.** → Jackson의 `@JsonValue`가 적용되지 않고 **영문 enum 상수명(`ACADEMIC`, `RELIGION`, `ART`, `SPORT`, `SHOW`, `VOLUNTEER`)이 그대로 응답됩니다.** (한글 아님)

---

#### 1. 전체 동아리 조회 (모바일)
`GET /clubs`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | 요청 바디 없음 / 응답 `application/json` |

**요청**
- 경로변수·쿼리·헤더·바디 **없음**

**응답**

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"전체 동아리 조회 완료"` |
| data | Array&lt;ClubListResponse&gt; | 동아리 목록 (빈 배열 가능) |
| data[].clubUUID | UUID(String) | 동아리 식별자 |
| data[].clubName | String | 동아리명 (최대 10자) |
| data[].mainPhoto | String \| null | 메인 사진 presigned GET URL(1시간). 사진 미등록 시 `null` |
| data[].departmentName | String | 분과 **영문 enum명** (`ACADEMIC`/`RELIGION`/`ART`/`SPORT`/`SHOW`/`VOLUNTEER`) |
| data[].clubHashtags | Array&lt;String&gt; | 해시태그(각 최대 10자). 없으면 `[]` |

```json
{
  "message": "전체 동아리 조회 완료",
  "data": [
    {
      "clubUUID": "11111111-2222-3333-4444-555555555555",
      "clubName": "동구라미",
      "mainPhoto": "https://example-bucket.s3.amazonaws.com/mainPhoto/abc.jpg?X-Amz-Signature=...",
      "departmentName": "ACADEMIC",
      "clubHashtags": ["코딩", "스터디"]
    },
    {
      "clubUUID": "66666666-7777-8888-9999-000000000000",
      "clubName": "수원밴드",
      "mainPhoto": null,
      "departmentName": "SHOW",
      "clubHashtags": []
    }
  ]
}
```
- HTTP 200 OK

**비즈니스 규칙**
1. `clubRepository.findAll()` — 모집 상태와 무관하게 전체 동아리 조회. 필터/정렬/페이징 없음.
2. 조회된 clubId 목록으로 메인사진(`findByClubIds`)과 해시태그(`findByClubIds`)를 **각 1회 벌크 조회**(N+1 회피).
3. 메인사진은 `Collectors.toMap(clubId, presignedUrl)`으로 매핑 → 한 동아리에 메인사진 행이 2건 이상이면 `IllegalStateException`(500)이 발생할 수 있음(엔티티는 `@OneToOne`이라 정상 상황에서는 미발생).
4. 부수효과: **S3 presigned GET URL 생성**(동아리 수만큼 호출). 메일/FCM/Redis/쿠키 부수효과 없음. `@Transactional(readOnly = true)`.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 중 `AmazonS3Exception`/`SdkClientException` 발생 시(`FileException`) |
| NO_CATCH_ERROR | 500 | (예외 메시지) | 그 외 처리되지 않은 예외 |

---

#### 2. 동아리 리스트 조회 (기존 회원가입용 경량 목록)
`GET /clubs/list`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | 요청 바디 없음 / 응답 `application/json` |

**요청**
- 파라미터 **없음**

**응답**

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"동아리 리스트 조회 성공"` |
| data | Array&lt;ClubInfoListResponse&gt; | 동아리 목록 |
| data[].clubUUID | UUID(String) | 동아리 식별자 |
| data[].clubName | String | 동아리명 |
| data[].mainPhoto | String \| null | 메인 사진 presigned GET URL(1시간), 없으면 `null` |

```json
{
  "message": "동아리 리스트 조회 성공",
  "data": [
    {
      "clubUUID": "11111111-2222-3333-4444-555555555555",
      "clubName": "동구라미",
      "mainPhoto": "https://example-bucket.s3.amazonaws.com/mainPhoto/abc.jpg?X-Amz-Signature=..."
    }
  ]
}
```
- HTTP 200 OK

**비즈니스 규칙**
1. `clubRepository.findAll()` — 전체 동아리(모집 상태 무관).
2. **1번 API와의 차이**: `departmentName`, `clubHashtags` 필드가 **없음**. 회원가입 시 소속 동아리 선택용 경량 목록.
3. 사진 조회를 `clubMainPhotoRepository.findByClub(club)`로 **동아리마다 개별 호출**(N+1 쿼리). 1번 API의 벌크 조회와 구현 방식이 다름.
4. 부수효과: S3 presigned GET URL 생성. `@Transactional(readOnly = true)`.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 실패 |
| NO_CATCH_ERROR | 500 | (예외 메시지) | 그 외 미처리 예외 |

---

#### 3. 모집 중인 전체 동아리 조회
`GET /clubs/open`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | 요청 바디 없음 / 응답 `application/json` |

**요청**
- 파라미터 **없음**

**응답**
- 필드 구조는 **1번(`GET /clubs`)과 완전히 동일**(`ClubListResponse`).
- `message`만 `"모집 중인 동아리 조회 완료"`.

```json
{
  "message": "모집 중인 동아리 조회 완료",
  "data": [
    {
      "clubUUID": "11111111-2222-3333-4444-555555555555",
      "clubName": "동구라미",
      "mainPhoto": "https://example-bucket.s3.amazonaws.com/mainPhoto/abc.jpg?X-Amz-Signature=...",
      "departmentName": "ACADEMIC",
      "clubHashtags": ["코딩", "스터디"]
    }
  ]
}
```
- HTTP 200 OK

**비즈니스 규칙**
1. `clubIntroRepository.findOpenClubIds()` → `SELECT ci.club.clubId FROM ClubIntro ci WHERE ci.recruitmentStatus = 'OPEN'`. **모집 상태는 `ClubIntro.recruitmentStatus`(기본값 `CLOSE`)로 판정**하며, 소개글(`ClubIntro`) 레코드가 없는 동아리는 결과에서 제외됨.
2. 해당 clubId들로 `clubRepository.findByClubIds()` 조회 → 메인사진/해시태그 벌크 조회 후 매핑.
3. 정렬·페이징 없음. 부수효과: S3 presigned GET URL 생성. `@Transactional(readOnly = true)`.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 실패 |
| NO_CATCH_ERROR | 500 | (예외 메시지) | 그 외 미처리 예외 |

---

#### 4. 카테고리 리스트 조회
`GET /clubs/categories`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | 요청 바디 없음 / 응답 `application/json` |

**요청**
- 파라미터 **없음**

**응답**

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"카테고리 조회 완료"` |
| data | Array&lt;ClubCategoryResponse&gt; | 카테고리 전체 목록 |
| data[].clubCategoryUUID | UUID(String) | 카테고리 식별자. **5·6번 API의 `clubCategoryUUIDs` 파라미터에 사용** |
| data[].clubCategoryName | String | 카테고리명 (최대 20자) |

```json
{
  "message": "카테고리 조회 완료",
  "data": [
    { "clubCategoryUUID": "aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb", "clubCategoryName": "IT" },
    { "clubCategoryUUID": "cccccccc-4444-5555-6666-dddddddddddd", "clubCategoryName": "음악" }
  ]
}
```
- HTTP 200 OK

**비즈니스 규칙**
1. `clubCategoryRepository.findAll()` → `ClubCategoryMapper.toDtoList()` 변환. 정렬·페이징·필터 없음.
2. 부수효과 없음. `@Transactional(readOnly = true)`.

**에러**
- 코드상 명시적으로 던지는 예외 없음. 미처리 예외 시 `NO_CATCH_ERROR` / 500.

---

#### 5. 카테고리별 전체 동아리 조회
`GET /clubs/filter`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | 요청 바디 없음 / 응답 `application/json` |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 쿼리 | clubCategoryUUIDs | List&lt;UUID&gt; | 형식상 선택(`defaultValue = ""`), **실질적으로 필수** | 카테고리 UUID 목록. **최대 3개**. 생략 시 빈 리스트가 되어 `CATEGORY_NOT_FOUND`(404) 발생 |

- 파라미터 형식 (둘 다 Spring 기본 바인딩으로 동작)
  - 콤마 구분: `?clubCategoryUUIDs=aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb,cccccccc-4444-5555-6666-dddddddddddd`
  - 반복 파라미터: `?clubCategoryUUIDs=aaaa...&clubCategoryUUIDs=cccc...`
- 요청 예시
```
GET /clubs/filter?clubCategoryUUIDs=aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb,cccccccc-4444-5555-6666-dddddddddddd
```

**응답**

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"카테고리별 전체 동아리 조회 완료"` |
| data | Array&lt;ClubListByClubCategoryResponse&gt; | 카테고리 단위 그룹 배열 |
| data[].clubCategoryUUID | UUID(String) | 카테고리 식별자 |
| data[].clubCategoryName | String | 카테고리명 |
| data[].clubs | Array&lt;ClubListResponse&gt; | 동아리 목록 (필드는 1번 API와 동일: clubUUID/clubName/mainPhoto/departmentName/clubHashtags) |

```json
{
  "message": "카테고리별 전체 동아리 조회 완료",
  "data": [
    {
      "clubCategoryUUID": "aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb",
      "clubCategoryName": "IT",
      "clubs": [
        {
          "clubUUID": "11111111-2222-3333-4444-555555555555",
          "clubName": "동구라미",
          "mainPhoto": "https://example-bucket.s3.amazonaws.com/mainPhoto/abc.jpg?X-Amz-Signature=...",
          "departmentName": "ACADEMIC",
          "clubHashtags": ["코딩"]
        }
      ]
    },
    {
      "clubCategoryUUID": "cccccccc-4444-5555-6666-dddddddddddd",
      "clubCategoryName": "음악",
      "clubs": [ ]
    }
  ]
}
```
- HTTP 200 OK

**비즈니스 규칙 (서비스 실행 순서 그대로)**
1. `validateCategoryLimit()` — `clubCategoryUUIDs`(null이면 빈 리스트로 간주)의 **size > 3이면 즉시 `INVALID_CATEGORY_COUNT`**. 개수 검증이 존재 검증보다 **먼저** 수행됨.
2. `findClubCategoryIdsByUUIDs(uuids)` 로 내부 PK 조회. 결과가 비면 `CATEGORY_NOT_FOUND`.
   - **부분 매칭 허용**: 3개 중 1개만 존재해도 에러 없이 진행하며, 존재하지 않는 UUID는 **조용히 무시**됩니다.
   - **파라미터 미전달 시**: `defaultValue = ""` → 빈 리스트 → 조회 결과 빈 리스트 → `CATEGORY_NOT_FOUND`(404).
3. `clubCategoryMappingRepository.findClubsByCategoryIds(ids)` 로 요청 카테고리들에 매핑된 동아리 조회(**`DISTINCT` 없음** → 한 동아리가 요청한 카테고리 2개 이상에 속하면 결과에 **중복 등장**).
4. 메인사진/해시태그 벌크 조회 후 presigned URL 생성.
5. **응답 그룹핑 주의 (코드상 실제 동작)**: 각 카테고리 그룹의 `clubs`에는 해당 카테고리 소속 동아리만 담기는 것이 아니라, **3단계에서 조회한 전체 동아리 목록(요청한 카테고리들의 합집합)이 모든 그룹에 동일하게 복제되어** 들어갑니다. 즉 2개 카테고리를 요청하면 두 그룹의 `clubs` 내용이 서로 같습니다. 클라이언트는 이 점을 전제로 처리해야 합니다.
6. 그룹 배열의 순서는 2단계 쿼리 결과 순서(`ORDER BY` 없음)이며 **요청 UUID 순서와 일치한다는 보장이 없습니다.**
7. 부수효과: S3 presigned GET URL 생성. `@Transactional(readOnly = true)`.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CG-202 | 413 Payload Too Large | 카테고리는 최대 3개까지 선택할수 있습니다. | `clubCategoryUUIDs` 개수가 4개 이상 |
| CTG-201 | 404 Not Found | 존재하지 않는 카테고리입니다. | 파라미터 미전달/빈 값이거나, 전달한 UUID가 **전부** 존재하지 않을 때. (그룹 생성 중 `findById` 실패 시에도 동일 예외) |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | UUID 파싱 실패(`MethodArgumentTypeMismatchException`) |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 실패 |
| NO_CATCH_ERROR | 500 | (예외 메시지) | 그 외 미처리 예외 |

---

#### 6. 카테고리별 모집 중인 동아리 조회
`GET /clubs/open/filter`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | 요청 바디 없음 / 응답 `application/json` |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 쿼리 | clubCategoryUUIDs | List&lt;UUID&gt; | 형식상 선택(`defaultValue = ""`), 실질적으로 필수 | 카테고리 UUID 목록, **최대 3개** |

```
GET /clubs/open/filter?clubCategoryUUIDs=aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb
```

**응답**
- 구조는 5번과 동일(`ClubListByClubCategoryResponse`), `message`만 `"카테고리별 모집 중인 동아리 조회 완료"`.

```json
{
  "message": "카테고리별 모집 중인 동아리 조회 완료",
  "data": [
    {
      "clubCategoryUUID": "aaaaaaaa-1111-2222-3333-bbbbbbbbbbbb",
      "clubCategoryName": "IT",
      "clubs": [
        {
          "clubUUID": "11111111-2222-3333-4444-555555555555",
          "clubName": "동구라미",
          "mainPhoto": "https://example-bucket.s3.amazonaws.com/mainPhoto/abc.jpg?X-Amz-Signature=...",
          "departmentName": "ACADEMIC",
          "clubHashtags": ["코딩"]
        }
      ]
    }
  ]
}
```
- HTTP 200 OK

**비즈니스 규칙**
1. `validateCategoryLimit()` (최대 3개) → 초과 시 `INVALID_CATEGORY_COUNT`.
2. `findClubCategoryIdsByUUIDs()` 결과가 비면 `CATEGORY_NOT_FOUND`. (부분 매칭은 5번과 동일하게 허용)
3. `clubIntroRepository.findOpenClubIds()` 로 모집중(`ClubIntro.recruitmentStatus = OPEN`) 동아리 ID 조회.
4. `findOpenClubsByCategoryIds(categoryIds, openClubIds)` — **`SELECT DISTINCT`** 사용 → 5번과 달리 동아리 중복 없음.
5. 메인사진/해시태그 벌크 조회 및 presigned URL 생성.
6. **5번과 동일한 그룹핑 특성**: 각 카테고리 그룹의 `clubs`에 요청 카테고리 합집합의 동일한 동아리 목록이 복제되어 들어갑니다.
7. `openClubIds`가 비어 있으면 JPQL의 `IN ()`가 되어 DB/드라이버에 따라 결과가 비거나 오류가 날 수 있음 — **코드상 빈 리스트 방어 로직 미확인**.
8. 부수효과: S3 presigned GET URL 생성. `@Transactional(readOnly = true)`.

**에러**
- 5번 표와 동일 (CG-202 / 413, CTG-201 / 404, INVALID_UUID_FORMAT / 400, FILE-306 / 400, NO_CATCH_ERROR / 500).

---

#### 7. 동아리 소개글(상세) 조회
`GET /clubs/intro/{clubUUID}`

| 항목 | 값 |
|---|---|
| 인증 | permitAll |
| 레이트리밋 | 없음 |
| Content-Type | 요청 바디 없음 / 응답 `application/json` |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID(String) | 필수 | 조회할 동아리 UUID |

```
GET /clubs/intro/11111111-2222-3333-4444-555555555555
```

**응답** (`AdminClubIntroResponse` — 운영팀 웹/모바일 공용 DTO)

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정값 `"동아리 소개글 조회 성공"` |
| data.clubUUID | UUID(String) | 동아리 식별자 |
| data.mainPhoto | String \| null | 메인 사진 presigned GET URL(1시간). 사진 없으면 `null` |
| data.introPhotos | Array&lt;String&gt; | 소개 사진 presigned GET URL 배열. **`order`(photo_order) 오름차순 정렬** (쿼리 `ORDER BY cip.order` + Java `Comparator.comparingInt` 이중 정렬). 없으면 `[]` |
| data.clubName | String | 동아리명 (최대 10자) |
| data.leaderName | String | 회장 이름 (최대 30자) |
| data.leaderHp | String | 회장 연락처 (최대 11자, 하이픈 없는 저장 길이) |
| data.clubInsta | String \| null | 동아리 인스타그램 |
| data.clubIntro | String \| null | 동아리 소개글 (최대 3000자) |
| data.recruitmentStatus | String(enum) | 모집 상태. `@JsonValue` 없는 enum이라 **`"OPEN"` / `"CLOSE"`** 로 직렬화 |
| data.googleFormUrl | String \| null | 지원용 구글폼 URL |
| data.clubHashtags | Array&lt;String&gt; | 해시태그(각 최대 10자). 없으면 `[]`. **정렬 기준 없음** |
| data.clubCategoryNames | Array&lt;String&gt; | 카테고리 **이름** 배열(UUID 아님). 없으면 `[]` |
| data.clubRoomNumber | String | 동아리방 호수 (최대 4자) |
| data.clubRecruitment | String \| null | 모집글 (최대 3000자) |

```json
{
  "message": "동아리 소개글 조회 성공",
  "data": {
    "clubUUID": "11111111-2222-3333-4444-555555555555",
    "mainPhoto": "https://example-bucket.s3.amazonaws.com/mainPhoto/abc.jpg?X-Amz-Signature=...",
    "introPhotos": [
      "https://example-bucket.s3.amazonaws.com/introPhoto/p1.jpg?X-Amz-Signature=...",
      "https://example-bucket.s3.amazonaws.com/introPhoto/p2.jpg?X-Amz-Signature=..."
    ],
    "clubName": "동구라미",
    "leaderName": "홍길동",
    "leaderHp": "01000000000",
    "clubInsta": "https://instagram.com/example_club",
    "clubIntro": "저희는 수원대 개발 동아리입니다.",
    "recruitmentStatus": "OPEN",
    "googleFormUrl": "https://forms.gle/example",
    "clubHashtags": ["코딩", "스터디"],
    "clubCategoryNames": ["IT", "학술"],
    "clubRoomNumber": "A101",
    "clubRecruitment": "2026학년도 1학기 신입 부원을 모집합니다."
  }
}
```
- HTTP 200 OK

**비즈니스 규칙 (실행 순서)**
1. `clubRepository.findByClubUUID(clubUUID)` → 없으면 `ClubException(CLUB_NOT_EXISTS)`.
2. `clubIntroRepository.findByClubClubId(clubId)` → 없으면 `ClubException(CLUB_INTRO_NOT_EXISTS)`. **소개글 레코드가 없는 동아리는 상세 조회 자체가 404**.
3. 메인 사진: `findByClubClubId` → 있으면 presigned GET URL, 없으면 `null`.
4. 소개 사진: `findByClubIntroClubId(clubId)`(쿼리에서 `ORDER BY cip.order`) → Java에서 `order` 기준 재정렬 → 각각 presigned GET URL 생성. **URL 배열만 반환되며 order 값 자체는 응답에 포함되지 않고, 배열 인덱스 순서가 곧 order 순서입니다.**
5. 해시태그: `clubHashtagRepository.findByClubClubId(clubId)` → 문자열 리스트.
6. 카테고리: `clubCategoryMappingRepository.findByClubClubId(clubId)`(`JOIN FETCH clubCategory`) → 카테고리 **이름** 리스트.
7. 모집 상태/구글폼/소개글/모집글은 `ClubIntro`에서, 회장정보(`leaderName`, `leaderHp`)·인스타·동아리방(`clubRoomNumber`)·동아리명은 `Club` 엔티티에서 가져옵니다.
8. 부수효과: **S3 presigned GET URL 생성**(메인 1건 + 소개사진 N건). 메일/FCM/Redis/쿠키 부수효과 없음. `@Transactional(readOnly = true)`.
9. 보안 참고: 이 엔드포인트는 permitAll인데 **회장 이름·연락처(`leaderName`, `leaderHp`)가 인증 없이 노출**됩니다. (코드상 마스킹 처리 없음)

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLUB-201 | 404 Not Found | 존재하지않는 동아리 입니다. | `clubUUID`에 해당하는 동아리 없음 |
| CINT-201 | 404 Not Found | 해당 동아리 소개글이 존재하지 않습니다. | 동아리는 있으나 `ClubIntro` 레코드 없음 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | 경로변수가 UUID 형식이 아님 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 실패 |
| NO_CATCH_ERROR | 500 | (예외 메시지) | 그 외 미처리 예외 |

---

### 목록 API 필드/특성 비교표

| API | 응답 DTO | 반환 필드 | 대상 범위 | 그룹핑 | 정렬 | 페이징 |
|---|---|---|---|---|---|---|
| `GET /clubs` | `ClubListResponse[]` | clubUUID, clubName, mainPhoto, departmentName, clubHashtags | 전체 | 없음 | 미지정 | 없음 |
| `GET /clubs/list` | `ClubInfoListResponse[]` | clubUUID, clubName, mainPhoto | 전체 | 없음 | 미지정 | 없음 |
| `GET /clubs/open` | `ClubListResponse[]` | 위와 동일(5필드) | `ClubIntro.recruitmentStatus = OPEN` | 없음 | 미지정 | 없음 |
| `GET /clubs/categories` | `ClubCategoryResponse[]` | clubCategoryUUID, clubCategoryName | 전체 카테고리 | 없음 | 미지정 | 없음 |
| `GET /clubs/filter` | `ClubListByClubCategoryResponse[]` | 카테고리(UUID/이름) + clubs(5필드) | 요청 카테고리 매핑 동아리(DISTINCT 없음 → 중복 가능) | 카테고리별(단, 모든 그룹에 동일 목록 복제) | 미지정 | 없음 |
| `GET /clubs/open/filter` | `ClubListByClubCategoryResponse[]` | 카테고리(UUID/이름) + clubs(5필드) | 요청 카테고리 ∩ 모집중(DISTINCT 적용) | 카테고리별(동일 목록 복제) | 미지정 | 없음 |
| `GET /clubs/intro/{clubUUID}` | `AdminClubIntroResponse` | 14필드(상세) | 단건 | - | introPhotos만 order 오름차순 | 없음 |

### 프론트엔드 구현 시 유의점 (코드 확인 결과)
1. **`departmentName`은 영문 코드값**(`ACADEMIC` 등)으로 내려옵니다. 한글 표기가 필요하면 클라이언트에서 매핑해야 합니다. (`Department` enum의 한글 `@JsonValue`는 String 필드로 변환되면서 적용되지 않음)
2. **필터 API의 `clubs`는 카테고리별로 분리되지 않습니다** — 모든 그룹이 동일한 합집합 목록을 갖습니다.
3. **`clubCategoryUUIDs` 미전달 시 200이 아니라 404(CTG-201)** 입니다. 전체 조회가 필요하면 `/clubs` 또는 `/clubs/open`을 사용하세요.
4. **카테고리 4개 이상 요청 시 HTTP 413**(일반적인 400이 아님)입니다.
5. 사진 URL은 **1시간 만료 presigned URL**이므로 캐싱/영구 저장하면 안 됩니다.
6. 주석 처리되어 비활성화된 엔드포인트는 `ClubController` 내에 **없습니다**(7개 모두 활성).

### 참고 파일 경로
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/api/ClubController.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/service/ClubService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/dto/ClubListResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/dto/ClubInfoListResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/dto/ClubListByClubCategoryResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/dto/ClubCategoryResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/dto/AdminClubIntroResponse.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/club/domain/Department.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/clubIntro/domain/ClubIntro.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/club/clubIntro/domain/ClubIntroPhoto.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/s3File/Service/S3FileUploadService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/exception/ExceptionType.java` (84, 94, 95, 101, 204행)
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/exception/GlobalExceptionHandler.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/resources/yaml/application-security.yml` (30, 33행)

---

## 6. 동아리 회장 API — 계정 · 동아리 운영

> 조사 대상 파일
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/clubLeader/api/ClubLeaderLoginController.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/clubLeader/api/ClubLeaderController.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/clubLeader/service/ClubLeaderLoginService.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/clubLeader/service/ClubLeaderService.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/clubLeader/dto/**`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/validation/ClubRoomNumberValidator.java`
> - `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/s3File/Service/S3FileUploadService.java`

### 공통 사항

**응답 래퍼** — `ApiResponse<T>` (`/global/response/ApiResponse.java`)
```json
{ "message": "...", "data": { } }
```
`data`는 `@JsonInclude(NON_NULL)` — null이면 JSON에서 아예 생략됨. 즉 `new ApiResponse<>("메시지")`만 쓰는 엔드포인트는 `{"message":"..."}`만 반환.

**에러 응답** — `ErrorResponse` (`/global/response/ErrorResponse.java`)
```json
{
  "exception": "ClubLeaderException",
  "code": "CLDR-101",
  "message": "동아리 접근 권한이 없습니다.",
  "status": 403,
  "error": "Forbidden",
  "additionalData": null
}
```

**인증 실패(필터 단계)** — `CustomAuthenticationEntryPoint`는 위 포맷이 아니라 별도 포맷 반환 (401):
```json
{ "status": 401, "errorCode": "TOKEN_EXPIRED", "message": "토큰이 만료되었습니다." }
```
`errorCode`: `AUTH_REQUIRED` / `TOKEN_EXPIRED` / `INVALID_TOKEN`

**인증 방식** — `Authorization: Bearer {accessToken}` 헤더. 액세스 토큰 유효기간 30분, 리프레시 토큰 7일(Redis + HttpOnly 쿠키 `refreshToken`). 액세스 토큰 payload에는 `sub`(leaderUUID)와 `clubUUID` 클레임이 담김.

**`validateLeaderAccess(clubUUID)` (모든 `{clubUUID}` 경로 공통 선행 검증)** — `ClubLeaderService:95`
1. `SecurityContextHolder`의 principal을 `CustomLeaderDetails`로 캐스팅 (LEADER가 아닌 principal이면 `ClassCastException` → 500 `NO_CATCH_ERROR`)
2. 경로변수 `clubUUID` != JWT 발급 시점의 `leaderDetails.getClubUUID()` → `CLDR-101 / 403`
3. `clubRepository.findByClubUUID` 실패 → `CLUB-201 / 404`

**공통 에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| AUTH_REQUIRED / INVALID_TOKEN | 401 | 인증이 필요합니다. | 토큰 미첨부·변조 |
| TOKEN_EXPIRED | 401 | 토큰이 만료되었습니다. | 액세스 토큰 30분 만료 |
| (Spring Security) | 403 | - | ROLE_LEADER 아님 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | 경로변수 UUID 파싱 실패 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | Bean Validation 실패, `additionalData`에 `{필드:메시지}` 맵 |
| CLDR-101 | 403 | 동아리 접근 권한이 없습니다. | JWT clubUUID ≠ 경로 clubUUID |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | clubUUID로 동아리 조회 실패 |
| COM-501 / NO_CATCH_ERROR | 500 | 서버 오류 | 미처리 예외 |

**검증 그룹 순서** — `ValidationSequence` = `{Default, NotBlankGroup, SizeGroup, PatternGroup}` (`@GroupSequence`). 앞 그룹이 실패하면 뒤 그룹은 평가되지 않음. 즉 `@NotBlank` → `@Size` → `@Pattern` 순으로 하나씩만 에러가 나옴.

**비활성화된 엔드포인트** — `ClubLeaderController.java:108~112`의 `GET /club-leader/v1/members` (`getClubMembers(LeaderToken token)`)는 **전체 주석 처리되어 있어 호출 불가**. 마찬가지로 `ClubLeaderService.java:504~522`의 구 `findClubMembers(LeaderToken)`도 주석 처리됨.

---

#### 1. 동아리 회장 로그인
`POST /club-leader/login`

| 항목 | 값 |
|---|---|
| 인증 | permitAll (`application-security.yml`의 `permit-all-paths`에 `/club-leader/login` 등재) |
| 레이트리밋 | `WEB_LOGIN` 5회/5분 (버킷 키 = `{leaderAccount}:WEB_LOGIN`, `Refill.intervally(5, 5분)` — 5분마다 5개 일괄 충전) |
| Content-Type | application/json |

**요청**

바디 — `LeaderLoginRequest` (`@Validated(ValidationSequence.class)`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| leaderAccount | String | O | `@NotBlank` "아이디는 필수 입력 값입니다." | 회장 계정 아이디. 레이트리밋 버킷 식별자로도 사용 |
| leaderPw | String | O | `@NotBlank` "비밀번호는 필수 입력 값입니다." | 평문 비밀번호 |

```json
{
  "leaderAccount": "club_leader_demo",
  "leaderPw": "Example1234!"
}
```

**응답** — 200 OK

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"동아리 회장 로그인 성공"` |
| data.accessToken | String | JWT 액세스 토큰 (30분). `Authorization` 응답 헤더에도 `Bearer {token}`으로 세팅됨 |
| data.refreshToken | String | 리프레시 토큰(UUID 문자열). 동시에 HttpOnly 쿠키로도 내려감 |
| data.role | String(enum) | 항상 `"LEADER"` (`Role` = USER/ADMIN/LEADER) |
| data.clubUUID | UUID | 회장이 담당하는 동아리 UUID. **이후 모든 `/club-leader/{clubUUID}/**` 호출에 그대로 사용** |
| data.isAgreedTerms | Boolean | 약관 동의 여부. `false`면 클라이언트가 약관 동의 화면으로 유도 후 2번 API 호출 필요 |

```json
{
  "message": "동아리 회장 로그인 성공",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMTExMTExMS0xMTExLTExMTEtMTExMS0xMTExMTExMTExMTEiLCJjbHViVVVJRCI6ImFhYWFhYWFhLWJiYmItY2NjYy1kZGRkLWVlZWVlZWVlZWVlZSJ9.signature",
    "refreshToken": "9f1c0d2e-3a4b-5c6d-7e8f-0a1b2c3d4e5f",
    "role": "LEADER",
    "clubUUID": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "isAgreedTerms": false
  }
}
```

**비즈니스 규칙**
1. `leaderAccount`로 Leader 조회 → 없으면 `USR-211`
2. `passwordEncoder.matches(leaderPw, 저장된 해시)` 불일치 → `USR-211` (계정 없음/비번 틀림을 동일 코드로 응답)
3. `leaderRepository.findClubUUIDByLeaderUUID` 로 담당 동아리 UUID 조회 → 없으면 `USR-201`
4. 액세스 토큰 생성: subject=leaderUUID, `clubUUID` 클레임 포함, 만료 30분
5. 리프레시 토큰 생성: **기존 리프레시 토큰을 Redis에서 먼저 삭제**한 뒤 새 UUID 발급 (동일 계정 단일 세션)

**부수효과**
- 응답 헤더 `Authorization: Bearer {accessToken}` (CORS `exposedHeader`에 `Authorization` 등록되어 브라우저에서 읽기 가능)
- 응답 헤더 `Set-Cookie: refreshToken={...}; Path=/; HttpOnly; Max-Age=604800; SameSite={security.cookie.same-site, 기본 Lax}[; Secure]`
- Redis: `refreshToken:{refreshToken}` = leaderUUID, TTL 7일
- Redis(bucket4j): `{leaderAccount}:WEB_LOGIN` 버킷 갱신

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-211 | 401 | 아이디 혹은 비밀번호가 일치하지 않습니다 | 계정 없음 또는 비밀번호 불일치 |
| USR-201 | 400 | 사용자가 존재하지 않습니다. | Leader에 연결된 동아리(clubUUID)가 없음 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 동일 계정 5회/5분 초과 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | leaderAccount/leaderPw 공백 |

---

#### 2. 약관 동의 완료
`PATCH /club-leader/terms/agreement`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER (`SecurityConfig`: `PATCH /club-leader/**` → `hasRole("LEADER")`) |
| 레이트리밋 | 없음 |
| Content-Type | 해당 없음 (바디 없음) |

**요청**
- 경로변수·쿼리·바디 **모두 없음**. 대상 회장은 JWT의 principal에서 결정됨.

**응답** — 200 OK

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"약관 동의 완료"` |

`data`는 null이므로 직렬화에서 생략:
```json
{ "message": "약관 동의 완료" }
```

**비즈니스 규칙**
1. `SecurityContextHolder` principal이 `CustomLeaderDetails`가 아니면 `USR-201`
2. `leader.setAgreeTerms(true)` 후 `leaderRepository.save(leader)`
3. `validateLeaderAccess`를 호출하지 않음 → 동아리 대조 없음

**부수효과** — `LEADER_TABLE.is_agreed_terms = true` 갱신

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| USR-201 | 400 | 사용자가 존재하지 않습니다. | principal이 LEADER 타입이 아님 |

---

#### 3. 동아리 기본 정보 조회
`GET /club-leader/{clubUUID}/info`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | 해당 없음 |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 로그인 응답의 `clubUUID`. JWT 값과 일치해야 함 |

**응답** — 200 OK, `ApiResponse<ClubInfoResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"동아리 기본 정보 조회 완료"` |
| data.mainPhotoUrl | String \| null | 대표 사진 **presigned GET URL (유효 1시간)**. 사진 미등록 시 `null` |
| data.clubName | String | 동아리명 (이 API로 변경 불가) |
| data.leaderName | String | 회장 이름 |
| data.leaderHp | String | 회장 전화번호 (하이픈 없는 11자리) |
| data.clubInsta | String \| null | 인스타그램 링크 |
| data.clubRoomNumber | String | 동아리방 호수 |
| data.clubHashtag | String[] | 해시태그 목록 |
| data.clubCategoryName | String[] | 카테고리명 목록 |
| data.department | String(enum) | 학부. `Department`에 `@JsonValue` 적용 → **한글 값으로 직렬화**: `학술`/`종교`/`예술`/`체육`/`공연`/`봉사` |

```json
{
  "message": "동아리 기본 정보 조회 완료",
  "data": {
    "mainPhotoUrl": "https://example-bucket.s3.ap-northeast-2.amazonaws.com/mainPhoto/0f1e2d3c-4b5a-6789-abcd-ef0123456789.jpg?X-Amz-Signature=...",
    "clubName": "동구라미",
    "leaderName": "홍길동",
    "leaderHp": "01012345678",
    "clubInsta": "https://instagram.com/example_club",
    "clubRoomNumber": "B101",
    "clubHashtag": ["코딩", "스터디"],
    "clubCategoryName": ["개발", "학술"],
    "department": "학술"
  }
}
```

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. 대표 사진 → 있으면 S3 presigned GET URL 생성, 없으면 null
3. 해시태그·카테고리 매핑 조회 (읽기 전용 트랜잭션)

**에러** — 공통 에러(`CLDR-101 403`, `CLUB-201 404`) 외 추가 없음.

---

#### 4. 카테고리 목록 조회
`GET /club-leader/category`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER (`GET /club-leader/**` → `hasRole("LEADER")`) |
| 레이트리밋 | 없음 |
| Content-Type | 해당 없음 |

**요청** — 파라미터 없음. `AdminClubCategoryService.getAllClubCategories()`로 **전체 카테고리 목록**을 조회하며 동아리별 필터링·`validateLeaderAccess` 없음.

**응답** — 200 OK, `ApiResponse<List<ClubCategoryResponse>>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"카테고리 리스트 조회 성공"` |
| data[].clubCategoryUUID | UUID | 카테고리 UUID |
| data[].clubCategoryName | String | 카테고리명. **5번 API의 `clubCategoryName` 배열에는 이 이름 문자열을 그대로 보내야 함** (UUID 아님) |

```json
{
  "message": "카테고리 리스트 조회 성공",
  "data": [
    { "clubCategoryUUID": "11111111-2222-3333-4444-555555555555", "clubCategoryName": "개발" },
    { "clubCategoryUUID": "66666666-7777-8888-9999-000000000000", "clubCategoryName": "학술" }
  ]
}
```

**비즈니스 규칙** — 카테고리는 관리자가 등록할 때 `toLowerCase()`로 정규화되어 저장됨(`AdminClubCategoryService.addClubCategory`). 목록은 정렬 없이 `findAll()` 순서.

**에러** — 공통 인증 에러만.

---

#### 5. 동아리 기본 정보 수정 (+ 회장 비밀번호 변경)
`PUT /club-leader/{clubUUID}/info`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER (`PUT /club-leader/**` → `hasRole("LEADER")`) |
| 레이트리밋 | 없음 (`RateLimitAction.LEADER_CHANGE_PW`는 enum에 정의만 되어 있고 **어디에도 적용되지 않음** — 코드 전역 grep 결과 사용처 0건) |
| Content-Type | multipart/form-data (3개 파트) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 대상 동아리 |
| 파트 | mainPhoto | file | X (`required=false`) | 대표 사진. jpg/jpeg/png만 허용, 개별 10MB / 요청 전체 50MB 제한 |
| 파트 | clubInfoRequest | JSON | X (선언상) / **사실상 O** | 아래 DTO. 생략 시 서비스에서 `clubInfoRequest.getLeaderName()` NPE → **500 `NULL_POINTER_EXCEPTION`** |
| 파트 | leaderUpdatePwRequest | JSON | X | 있으면 비밀번호도 함께 변경. 없으면 비밀번호 로직 미수행 |

각 JSON 파트는 `Content-Type: application/json`을 지정해야 함.

**파트 `clubInfoRequest`** — `ClubInfoRequest`, `@Validated(ValidationSequence.class)`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| leaderName | String | O | `@NotBlank` / `@Size(min=2, max=30)` "회장 이름은 2~30자 이내여야 합니다." / `@Pattern(^[a-zA-Z가-힣]+$)` "회장 이름은 영어 또는 한글만 입력 가능합니다." (공백·숫자 불가) | 회장 이름. **기존 값과 다르면 약관 동의가 false로 초기화됨** |
| leaderHp | String | O | `@NotBlank` / `@Size(min=11, max=11)` / `@Pattern(^01[0-9]{9}$)` | 하이픈 없는 11자리 |
| clubInsta | String | X | `@Pattern(^(https?://)?(www\.)?instagram\.com/.+$\|^$)` (그룹 미지정 = Default 그룹). null 허용, 빈 문자열 허용 | **미전송/`null`이면 기존 값이 null로 덮어써짐** (`Club.updateClubInfo`가 무조건 대입) |
| clubRoomNumber | String | O | `@NotBlank`(Default 그룹) + `@ValidClubRoomNumber`(PatternGroup) "유효하지 않은 동아리방입니다." | 아래 정규식+화이트리스트 |
| clubHashtag | List\<String\> | X | 리스트 `@Size(max=2)` "해시태그는 2개까지 입력 가능합니다." / 각 원소 `@Size(min=1, max=6)` "각 해시태그는 최대 6자까지 입력 가능합니다." | `null`이거나 빈 배열이면 **기존 해시태그 유지(변경 없음)** — 전체 삭제 불가 |
| clubCategoryName | List\<String\> | X | 리스트 `@Size(max=3)` "카테고리는 3개까지 입력 가능합니다." / 각 원소 `@Size(min=1, max=20)` | 4번 API의 `clubCategoryName` 문자열. `null`/빈 배열이면 **기존 카테고리 유지** |

**`clubRoomNumber` 검증 (`ClubRoomNumberValidator`)**
1. null 또는 공백 → 실패
2. 형식 정규식: `^(B\d{3}|\d{3})$` (B + 숫자 3자리, 또는 숫자 3자리)
3. 화이트리스트 포함 여부 (총 35개, 주석: "B109, 111, 204 제외")

| 층 | 허용 호수 |
|---|---|
| 지하 | `B101` `B102` `B103` `B104` `B105` `B106` `B107` `B108` `B110` `B111` `B112` `B113` `B114` `B115` `B116` `B117` `B118` `B119` `B120` `B121` `B122` `B123` |
| 1층 | `102` `103` `104` `105` `106` `107` `108` `109` `110` `112` |
| 2층 | `203` `205` `206` `207` `208` `209` `210` |

**파트 `leaderUpdatePwRequest`** — `LeaderUpdatePwRequest`, `@Validated(ValidationSequence.class)`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| leaderPw | String | O | `@NotBlank` "현재 비밀번호는 필수 입력 값입니다." | 현재 비밀번호 |
| newPw | String | O | `@NotBlank` / `@Size(min=8, max=20)` "비밀번호는 8~20자 이내여야 합니다." / `@Pattern(^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\\|,.<>/?])(?!.*\s).*$)` "비밀번호는 영문, 숫자, 특수문자를 모두 포함해야 하며 공백을 포함할 수 없습니다." | 새 비밀번호 |
| confirmNewPw | String | O | `@NotBlank` "새 비밀번호 확인은 필수 입력 값입니다." | 새 비밀번호 확인 |

서버 측 2차 검증(`PasswordService.validatePassword`)의 특수문자 집합은 DTO 정규식과 미세하게 다름: `[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>/?~\`]` (`~`, 백틱 추가 포함).

**요청 예시 (multipart)**

```
PUT /club-leader/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee/info
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: multipart/form-data; boundary=----X

------X
Content-Disposition: form-data; name="mainPhoto"; filename="main.jpg"
Content-Type: image/jpeg

<binary>
------X
Content-Disposition: form-data; name="clubInfoRequest"
Content-Type: application/json

{
  "leaderName": "홍길동",
  "leaderHp": "01012345678",
  "clubInsta": "https://instagram.com/example_club",
  "clubRoomNumber": "B101",
  "clubHashtag": ["코딩", "스터디"],
  "clubCategoryName": ["개발", "학술"]
}
------X
Content-Disposition: form-data; name="leaderUpdatePwRequest"
Content-Type: application/json

{
  "leaderPw": "OldPass1234!",
  "newPw": "NewPass5678!",
  "confirmNewPw": "NewPass5678!"
}
------X--
```

**응답** — 200 OK, `ApiResponse<UpdateClubInfoResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"동아리 기본 정보 변경 완료"` |
| data.presignedUrl | String \| null | **mainPhoto를 보낸 경우: 새 파일용 presigned PUT URL (유효 1시간)** / mainPhoto를 보내지 않은 경우: 기존 사진의 **S3 key 문자열**(URL 아님), 기존 사진도 없으면 `null` |

```json
{
  "message": "동아리 기본 정보 변경 완료",
  "data": {
    "presignedUrl": "https://example-bucket.s3.ap-northeast-2.amazonaws.com/mainPhoto/0f1e2d3c-4b5a-6789-abcd-ef0123456789.jpg?X-Amz-Algorithm=...&X-Amz-Signature=..."
  }
}
```

> **중요 (업로드 2단계 구조)** — `S3FileUploadService.uploadFile()`은 파일 바이트를 S3에 직접 올리지 않고, 확장자·파일 시그니처 검증 후 **presigned PUT URL만 생성**하고 DB 메타데이터를 갱신함. 따라서 클라이언트는 응답의 `presignedUrl`로 **직접 `PUT` 요청을 보내 실제 바이너리를 업로드해야** 사진이 반영됨. 비밀번호 변경만 하고 사진을 안 보낸 경우 `presignedUrl`은 URL이 아니라 S3 key이므로 그대로 PUT 하면 안 됨.

**비즈니스 규칙** (`ClubLeaderService.updateClubInfo` 실행 순서)
1. `validateLeaderAccess(clubUUID)`
2. **회장 이름 변경 감지** — `club.getLeaderName() != clubInfoRequest.leaderName`이면 해당 동아리의 Leader를 조회(`CLDR-201`)해 `setAgreeTerms(false)` 저장 → **약관 재동의 필요**(2번 API 재호출). 이름을 바꾸지 않으려면 기존 이름을 그대로 전송해야 함
3. **해시태그 갱신** — null/빈 배열이면 스킵. 아니면 새 집합(`Set`, 중복 자동 제거)에 없는 기존 해시태그를 일괄 삭제하고, 기존에 없던 것만 삽입
4. **카테고리 갱신** — null/빈 배열이면 스킵. 새 집합에 없는 매핑 삭제 후, 신규 카테고리명으로 `ClubCategory` 조회(없으면 `CTG-201`) 후 매핑 삽입
5. **대표 사진 갱신** — `mainPhoto`가 null/빈 파일이면 기존 S3 key 반환. 아니면 기존 S3 객체 삭제 → 확장자·시그니처 검증 → presigned PUT URL 생성 → 기존 `ClubMainPhoto` 행 삭제+flush 후 새 행 저장
6. `club.updateClubInfo(leaderName, leaderHp, clubInsta, clubRoomNumber)` — **4개 값 무조건 덮어쓰기**
7. **`leaderUpdatePwRequest != null`이면** `updatePassword()` 실행 (컨트롤러 레벨):
   - principal이 LEADER가 아니면 `USR-201`
   - 현재 비밀번호 불일치 → `USR-204`
   - 새 비밀번호가 현재 비밀번호와 동일 → `USR-217`
   - `PasswordService.validatePassword(newPw, confirmNewPw)`: 공백 → `USR-203`, 영문/숫자/특수문자 미충족 → `USR-214`, 두 값 불일치 → `USR-202`
   - 인코딩 후 저장
   - **`jwtProvider.deleteRefreshToken(leaderUUID)` + `deleteRefreshTokenCookie(response)` → 리프레시 토큰 무효화·쿠키 삭제 → 강제 재로그인 필요** (단, 발급된 액세스 토큰은 만료(최대 30분) 전까지 계속 유효)
   - `updatePassword`의 반환값 `ApiResponse("비밀번호가 변경되었습니다.")`는 **컨트롤러에서 버려짐** — 응답 본문에는 동아리 정보 변경 메시지만 나옴

**부수효과**
- S3: 기존 대표 사진 객체 삭제 (`DeleteObject`), 신규 파일용 presigned PUT URL 발급
- DB: `CLUB_TABLE`, `CLUB_MAIN_PHOTO`, 해시태그/카테고리 매핑, `LEADER_TABLE`(약관·비밀번호)
- Redis: (비밀번호 변경 시) `refreshToken:*` 키 삭제
- 쿠키: (비밀번호 변경 시) `Set-Cookie: refreshToken=; Max-Age=0; Expires=Thu, 01 Jan 1970 ...`

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLDR-101 | 403 | 동아리 접근 권한이 없습니다. | JWT clubUUID ≠ 경로 clubUUID |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 동아리 조회 실패 |
| CLDR-201 | 400 | 동아리 회장이 존재하지 않습니다. | 이름 변경 시 해당 동아리의 Leader 미존재 |
| CTG-201 | 404 | 존재하지 않는 카테고리입니다. | `clubCategoryName`에 미등록 카테고리 포함 |
| CLP-202 | 500 | 동아리 ID가 존재하지 않습니다. | clubId가 null |
| CLUB-204 | 404 | 동아리 사진이 존재하지 않습니다 | `saveClubMainPhoto`에 빈 파일 전달 |
| FILE-309 | 400 | 파일 이름이 유효하지 않습니다. | 원본 파일명 없음 |
| FILE-310 | 400 | 파일 확장자가 없습니다. | 파일명에 `.` 없음 |
| FILE-311 | 400 | 지원하지 않는 파일 확장자입니다. | jpg/jpeg/png 외, 또는 파일 시그니처 불일치 |
| FILE-312 | 400 | 파일 유효성 검사 실패 | 시그니처 검사 중 IOException |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | S3 presigned URL 생성 실패 |
| FILE-307 | 400 | 파일 삭제에 실패했습니다. | S3 객체 삭제 실패 |
| MAX_UPLOAD_SIZE_EXCEEDED | 400 | 업로드 가능한 최대 파일 크기를 초과했습니다. (개별 파일 10MB, 총 파일 크기 50MB) | 용량 초과 |
| USR-201 | 400 | 사용자가 존재하지 않습니다. | 비밀번호 변경 시 principal 타입 불일치 |
| USR-204 | 400 | 현재 비밀번호와 일치하지 않습니다 | `leaderPw` 불일치 |
| USR-217 | 400 | 현재 비밀번호와 같은 비밀번호로 변경할 수 없습니다. | 새 비밀번호 재사용 |
| USR-203 | 400 | 비밀번호 값이 빈칸입니다 | newPw/confirmNewPw 공백 |
| USR-214 | 400 | 영문자,숫자,특수문자는 적어도 1개 이상씩 포함되어야합니다 | 서버 2차 조건 미충족 |
| USR-202 | 400 | 두 비밀번호가 일치하지 않습니다. | newPw ≠ confirmNewPw |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | DTO 검증 실패(호수/전화번호/해시태그 개수 등) |
| NULL_POINTER_EXCEPTION | 500 | NullPointerException 발생: ... | `clubInfoRequest` 파트 누락 |

---

#### 6. 동아리 요약 조회
`GET /club-leader/{clubUUID}/summary`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | 해당 없음 |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 대상 동아리 |

**응답** — 200 OK, `ApiResponse<ClubSummaryResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"동아리 요약 조회 완료"` |
| data.clubUUID | UUID | 동아리 UUID |
| data.clubName | String | 동아리명 |
| data.leaderName | String | 회장 이름 |
| data.leaderHp | String | 회장 전화번호 |
| data.clubInsta | String \| null | 인스타 링크 |
| data.clubRoomNumber | String | 동아리방 호수 |
| data.clubHashtag | String[] | 해시태그 |
| data.clubCategories | String[] | 카테고리명 (3번 API의 `clubCategoryName`과 **필드명이 다름**) |
| data.clubIntro | String \| null | 소개글 |
| data.clubRecruitment | String \| null | 모집글 |
| data.recruitmentStatus | String(enum) | `OPEN` / `CLOSE` |
| data.googleFormUrl | String \| null | 구글폼 링크 |
| data.mainPhoto | String \| null | 대표 사진 presigned GET URL (1시간), 없으면 null |
| data.introPhotos | String[] | 소개 사진 presigned GET URL 목록. **`order` 오름차순 정렬**. 사진이 하나도 없으면 빈 배열. 삭제된(빈) 슬롯은 S3 key가 `""`이므로 **빈 문자열 `""`이 그대로 배열에 포함됨** |

```json
{
  "message": "동아리 요약 조회 완료",
  "data": {
    "clubUUID": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "clubName": "동구라미",
    "leaderName": "홍길동",
    "leaderHp": "01012345678",
    "clubInsta": "https://instagram.com/example_club",
    "clubRoomNumber": "B101",
    "clubHashtag": ["코딩", "스터디"],
    "clubCategories": ["개발", "학술"],
    "clubIntro": "저희는 매주 스터디를 진행합니다.",
    "clubRecruitment": "2026-1학기 신입 부원 모집",
    "recruitmentStatus": "OPEN",
    "googleFormUrl": "https://forms.gle/exampleForm",
    "mainPhoto": "https://example-bucket.s3.ap-northeast-2.amazonaws.com/mainPhoto/xxx.jpg?X-Amz-Signature=...",
    "introPhotos": [
      "https://example-bucket.s3.ap-northeast-2.amazonaws.com/introPhoto/aaa.jpg?X-Amz-Signature=...",
      "",
      "https://example-bucket.s3.ap-northeast-2.amazonaws.com/introPhoto/ccc.png?X-Amz-Signature=..."
    ]
  }
}
```

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `ClubIntro` 조회 실패 → `CINT-201`
3. 해시태그/카테고리/대표사진/소개사진 조회 후 presigned GET URL 생성 (읽기 전용 트랜잭션)

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLDR-101 | 403 | 동아리 접근 권한이 없습니다. | 접근 통제 실패 |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 동아리 미존재 |
| CINT-201 | 404 | 해당 동아리 소개글이 존재하지 않습니다. | ClubIntro 행 없음 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 중 S3/SDK 오류 |

---

#### 7. 동아리 소개 조회
`GET /club-leader/{clubUUID}/intro`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | 해당 없음 |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 대상 동아리 |

**응답** — 200 OK, `ApiResponse<LeaderClubIntroResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"동아리 소개 조회 완료"` |
| data.clubUUID | UUID | 동아리 UUID |
| data.clubIntro | String \| null | 소개글 (최대 3000자) |
| data.clubRecruitment | String \| null | 모집글 (최대 3000자) |
| data.recruitmentStatus | String(enum) | `OPEN` / `CLOSE` |
| data.googleFormUrl | String \| null | 지원 구글폼 링크 |
| data.introPhotos | String[] | 소개 사진 presigned GET URL, `order` 오름차순. 빈 슬롯은 `""` |

```json
{
  "message": "동아리 소개 조회 완료",
  "data": {
    "clubUUID": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "clubIntro": "저희는 매주 스터디를 진행합니다.",
    "clubRecruitment": "2026-1학기 신입 부원 모집",
    "recruitmentStatus": "OPEN",
    "googleFormUrl": "https://forms.gle/exampleForm",
    "introPhotos": [
      "https://example-bucket.s3.ap-northeast-2.amazonaws.com/introPhoto/aaa.jpg?X-Amz-Signature=..."
    ]
  }
}
```

**비즈니스 규칙** — 6번과 동일한 접근 통제 + `ClubIntro` 필수. 사진 URL 만료 1시간.

**에러** — 6번과 동일 (`CLDR-101 403`, `CLUB-201 404`, `CINT-201 404`).

---

#### 8. 동아리 소개 수정 (사진 5슬롯)
`PUT /club-leader/{clubUUID}/intro`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | multipart/form-data (2개 파트) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 대상 동아리 |
| 파트 | clubIntroRequest | JSON | X (선언상) / **사실상 O** | 생략 시 `clubIntroRequest.getRecruitmentStatus()` NPE → 500 |
| 파트 | introPhotos | file[] | X | 동일 파트명으로 여러 파일 전송. jpg/jpeg/png, 최대 5장 |

**파트 `clubIntroRequest`** — `ClubIntroRequest`, `@Valid` (**`ValidationSequence` 미적용 → Default 그룹만 평가, 여러 필드 에러가 동시에 반환될 수 있음**)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| clubIntro | String | X | `@Size(max=3000)` "소개글은 최대 3000자까지 입력 가능합니다." | **미전송 시 null로 덮어써짐** |
| recruitmentStatus | enum | O | `@NotNull` "모집 상태를 설정해주세요." | `OPEN` / `CLOSE`. **DTO상 필수지만 서비스에서 엔티티에 반영되지 않음** (아래 규칙 참조) |
| clubRecruitment | String | X | `@Size(max=3000)` "모집글은 최대 3000자까지 입력 가능합니다." | **미전송 시 null로 덮어써짐** |
| googleFormUrl | String | X | `@Pattern(^(https://[a-zA-Z0-9._-]+(?:\.[a-zA-Z]{2,})+.*)?$)` "유효한 HTTPS 링크를 입력해주세요." (http 불가, 빈 문자열 허용) | **미전송 시 null로 덮어써짐** |
| orders | List\<Integer\> | X | 애노테이션 없음, 서비스에서 검증 | `introPhotos[i]`가 들어갈 **슬롯 번호(1~5)**. 인덱스 순서로 1:1 매핑 |
| deletedOrders | List\<Integer\> | X | 애노테이션 없음, 서비스에서 검증 | 비울 슬롯 번호(1~5) 목록 |

**요청 예시 (multipart)**

```
PUT /club-leader/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee/intro
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: multipart/form-data; boundary=----Y

------Y
Content-Disposition: form-data; name="clubIntroRequest"
Content-Type: application/json

{
  "clubIntro": "저희는 매주 스터디를 진행합니다.",
  "recruitmentStatus": "OPEN",
  "clubRecruitment": "2026-1학기 신입 부원 모집",
  "googleFormUrl": "https://forms.gle/exampleForm",
  "orders": [1, 3],
  "deletedOrders": [5]
}
------Y
Content-Disposition: form-data; name="introPhotos"; filename="photo1.jpg"
Content-Type: image/jpeg

<binary>
------Y
Content-Disposition: form-data; name="introPhotos"; filename="photo3.png"
Content-Type: image/png

<binary>
------Y--
```
→ `photo1.jpg`는 1번 슬롯, `photo3.png`는 3번 슬롯에 배치되고 5번 슬롯은 비워짐.

**응답** — 200 OK, `ApiResponse<UpdateClubIntroResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"동아리 소개 변경 완료"` |
| data.presignedUrls | String[] | **새로 업로드할 사진용 presigned PUT URL 배열 (유효 1시간)**. 실제 업로드된 파일 순서대로만 담김(빈 파일로 스킵된 항목은 미포함). 사진을 안 보냈으면 빈 배열 `[]` |

```json
{
  "message": "동아리 소개 변경 완료",
  "data": {
    "presignedUrls": [
      "https://example-bucket.s3.ap-northeast-2.amazonaws.com/introPhoto/1a2b3c.jpg?X-Amz-Signature=...",
      "https://example-bucket.s3.ap-northeast-2.amazonaws.com/introPhoto/4d5e6f.png?X-Amz-Signature=..."
    ]
  }
}
```

> **중요** — 5번과 동일하게 서버는 바이너리를 S3로 전송하지 않는다. 클라이언트는 반환된 각 `presignedUrls[i]`에 **직접 `PUT`으로 해당 파일 바이트를 업로드**해야 사진이 실제로 반영된다.

**비즈니스 규칙** (`ClubLeaderService.updateClubIntro` 실행 순서)
1. `validateLeaderAccess(clubUUID)`
2. `ClubIntro` 조회 실패 → `CINT-201`
3. `recruitmentStatus == null` → `CINT-303` (단, `@NotNull` 때문에 보통 그 전에 400 `INVALID_ARGUMENT`로 막힘)
4. **삭제 처리** — `deletedOrders`가 null/빈 배열이 아니면:
   - `validateOrderValues`: 리스트 크기 1~5 아님 또는 원소가 1~5 범위 밖 → `CLP-201`
   - 각 order에 해당하는 `ClubIntroPhoto` 슬롯 조회, 없으면 `CLP-201`
   - S3 객체 삭제 후 **행은 유지한 채** `updateClubIntroPhoto("", "", order)` — 슬롯 비우기 (행 삭제 아님)
5. **업로드 처리** — `introPhotos`와 `orders`가 **둘 다 non-null & non-empty일 때만** 수행 (하나라도 비면 사진 관련 로직 전체 스킵):
   - `validateOrderValues(orders)` (1~5개, 값 1~5)
   - `introPhotos.size() > 5` → `FILE-308`
   - `for i in 0..introPhotos.size()-1`: `introPhotos[i]` ↔ `orders[i]` 인덱스 매핑
     - 빈 파일이면 `continue` (해당 순서 스킵)
     - `orders[i]` 슬롯 행 조회, 없으면 `CLP-201`
     - 기존 슬롯에 name/s3key가 모두 비어있지 않으면 기존 S3 객체 삭제
     - 확장자(jpg/jpeg/png)·파일 시그니처 검증 후 presigned PUT URL 생성, 슬롯 메타데이터(name/s3key/order) 갱신
     - presignedUrls에 추가
   - ⚠️ `orders.size() < introPhotos.size()`이면 `orders.get(i)`에서 `IndexOutOfBoundsException` → 500. 두 배열 길이를 반드시 일치시킬 것
   - ⚠️ 슬롯(`ClubIntroPhoto` 행)은 **사전에 존재해야** 한다. 존재하지 않는 order를 지정하면 `CLP-201`
   - ⚠️ 같은 `orders` 값을 중복 지정하면 나중 파일이 앞 파일을 덮어씀(검증 없음)
6. `clubIntro.updateClubIntro(clubIntro, clubRecruitment, googleFormUrl)` 저장 — **세 필드 무조건 덮어쓰기**
7. **`recruitmentStatus`는 요청에 필수지만 엔티티에 반영되지 않는다.** 모집 상태 변경은 9번(토글) API로만 가능

**부수효과**
- S3: 삭제 대상/교체 대상 객체 `DeleteObject`, 신규 파일용 presigned PUT URL 발급
- DB: `CLUB_INTRO_TABLE`(소개글/모집글/구글폼), `CLUB_INTRO_PHOTO_TABLE`(슬롯 메타데이터)

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLDR-101 | 403 | 동아리 접근 권한이 없습니다. | 접근 통제 실패 |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 동아리 미존재 |
| CINT-201 | 404 | 해당 동아리 소개글이 존재하지 않습니다. | ClubIntro 행 없음 |
| CINT-303 | 400 | 모집 상태가 올바르지 않습니다. | `recruitmentStatus`가 null인 채 서비스 진입 |
| CLP-201 | 400 | 범위를 벗어난 사진 순서 값입니다. | orders/deletedOrders 개수 0 또는 6 이상, 값이 1~5 밖, 또는 해당 order 슬롯 미존재 |
| FILE-308 | 400 | 업로드 가능한 갯수를 초과했습니다. | `introPhotos` 6장 이상 |
| FILE-309 / FILE-310 / FILE-311 / FILE-312 | 400 | 파일명·확장자·시그니처 오류 | jpg/jpeg/png 외 또는 위조 파일 |
| FILE-306 / FILE-307 | 400 | 파일 업로드/삭제 실패 | S3 오류 |
| MAX_UPLOAD_SIZE_EXCEEDED | 400 | 업로드 가능한 최대 파일 크기를 초과했습니다. (개별 파일 10MB, 총 파일 크기 50MB) | 용량 초과 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | `recruitmentStatus` 누락, 3000자 초과, googleFormUrl 형식 오류 |
| INVALID_REQUEST_BODY | 400 | 유효하지 않은 JSON 혹은 ENUM 값입니다. 올바른 값을 입력하세요. | `recruitmentStatus`에 OPEN/CLOSE 외 값 |
| NULL_POINTER_EXCEPTION | 500 | NullPointerException 발생: ... | `clubIntroRequest` 파트 누락 |

---

#### 9. 동아리 모집 상태 토글
`PATCH /club-leader/{clubUUID}/recruitment`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | 해당 없음 (바디 없음) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 대상 동아리 |

바디·쿼리 없음. **원하는 상태를 지정하는 것이 아니라 현재 상태의 반전만 가능** (`OPEN ↔ CLOSE`).

**응답** — 200 OK, `ApiResponse<RecruitmentStatus>`

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | 고정 `"동아리 모집 상태 변경 완료"` |
| data | String(enum) | **토글 후**의 상태: `"OPEN"` 또는 `"CLOSE"` |

```json
{
  "message": "동아리 모집 상태 변경 완료",
  "data": "OPEN"
}
```

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `ClubIntro` 조회 실패 → `CINT-201`
3. `clubIntro.toggleRecruitmentStatus()` — `RecruitmentStatus.toggle()` (`OPEN`이면 `CLOSE`, 아니면 `OPEN`). 엔티티 기본값은 `CLOSE`
4. 변경 상태는 JPA 더티체킹으로 반영됨 (코드상 명시적으로 저장하는 대상은 `clubRepository.save(club)`이고 `clubIntroRepository.save`는 호출하지 않음)

**부수효과** — `CLUB_INTRO_TABLE.club_intro_recruitment_status` 갱신. 메일/FCM/S3 등 외부 연동 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLDR-101 | 403 | 동아리 접근 권한이 없습니다. | JWT clubUUID ≠ 경로 clubUUID |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 동아리 미존재 |
| CINT-201 | 404 | 해당 동아리 소개글이 존재하지 않습니다. | ClubIntro 행 없음 |

---

### 프론트엔드 구현 시 주의점 요약

1. **로그인 후 반드시 `clubUUID`를 저장**해 모든 `{clubUUID}` 경로에 사용. 다른 값을 넣으면 403 `CLDR-101`.
2. **`isAgreedTerms == false`면 약관 동의 화면 → 2번 API 호출.** 5번 API로 회장 이름을 변경하면 서버가 자동으로 `false`로 되돌리므로, 이름 변경 직후 다시 약관 동의 플로우를 태워야 함.
3. **비밀번호 변경 시 리프레시 토큰이 삭제되고 쿠키가 만료**되므로, 응답 수신 후 클라이언트에서 로그아웃 처리 + 재로그인 유도 권장.
4. **사진 업로드는 2단계** — API 응답의 presigned URL로 클라이언트가 직접 `PUT` 해야 실제 반영. 유효기간 1시간.
5. 5번의 `data.presignedUrl`은 mainPhoto를 보내지 않은 경우 **URL이 아니라 S3 key**이므로 그대로 사용하면 안 됨.
6. **부분 수정 불가** — 5번의 `clubInsta`, 8번의 `clubIntro`/`clubRecruitment`/`googleFormUrl`은 미전송 시 `null`로 덮어써짐. 항상 현재 값을 함께 전송할 것.
7. **해시태그/카테고리는 빈 배열로 전체 삭제 불가** (null·빈 배열이면 갱신 로직 자체를 스킵).
8. 8번의 `recruitmentStatus`는 필수 전송이지만 저장되지 않음 — 모집 상태 변경은 9번 토글 API로만.

---

## 7. 동아리 회장 API — 지원자 · 부원 관리

**대상 컨트롤러**: `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/clubLeader/api/ClubLeaderController.java`
**서비스**: `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/clubLeader/service/ClubLeaderService.java`
**클래스 레벨 매핑**: `@RequestMapping("/club-leader")`

### 공통 사항

#### 인증
- `application-security.yml`의 `permit-all-paths`에 `/club-leader/login`만 등록되어 있음 → 본 문서의 모든 엔드포인트는 인증 필요.
- `SecurityConfig`(78~82행): `/club-leader/**`의 GET/POST/PUT/PATCH/DELETE = `hasRole("LEADER")`.
- **예외**: `SecurityConfig` 69행이 그보다 먼저 등록되어 있어 `PATCH /club-leader/fcmtoken`만 `hasRole("USER")`.
- 토큰 전달: `Authorization: Bearer {accessToken}` (`JwtProvider.AUTHORIZATION_HEADER`, `BEARER_PREFIX`).

#### 레이트리밋
`ClubLeaderController` / `ClubLeaderService` / `FcmServiceImpl` 어디에도 `@RateLimite`가 없음 → **본 문서의 16개 엔드포인트 전부 "없음"**.

#### 공통 접근 검증 `validateLeaderAccess(clubUUID)`
모든 `{clubUUID}` 엔드포인트가 서비스 첫 줄에서 호출.
1. SecurityContext의 principal(`CustomLeaderDetails`)의 `clubUUID`와 경로변수 `clubUUID`가 다르면 → `CLDR-101` (403)
2. `clubRepository.findByClubUUID` 실패 → `CLUB-201` (404)

#### 공통 응답 래퍼 `ApiResponse<T>`
```json
{ "message": "문자열", "data": {} }
```
`data`는 `@JsonInclude(NON_NULL)` → null이면 필드 자체가 생략됨.

#### 공통 에러 응답 `ErrorResponse`
```json
{
  "exception": "ClubMemberException",
  "code": "CMEM-201",
  "message": "동아리 회원이 존재하지 않습니다.",
  "status": 404,
  "error": "Not Found",
  "additionalData": null
}
```

#### 공통 에러 표

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLDR-101 | 403 | 동아리 접근 권한이 없습니다. | 토큰의 clubUUID ≠ 경로 clubUUID |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | clubUUID로 동아리 조회 실패 |
| TOK-204 | 401 | 인증되지 않은 사용자입니다. | 토큰 없음/인증 실패 |
| TOK-202 | 401 | 유효하지 않은 토큰입니다. | 토큰 만료·위조 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | 경로변수 UUID 파싱 실패 (`MethodArgumentTypeMismatchException` 핸들러) |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | Bean Validation 실패. `additionalData`에 `{필드명: 메시지}` 맵 |
| INVALID_REQUEST_BODY | 400 | 유효하지 않은 JSON 혹은 ENUM 값입니다. 올바른 값을 입력하세요. | JSON 파싱 실패, enum 값 불일치 |

#### 공통 검증 그룹 순서 `ValidationSequence`
`@GroupSequence({Default, NotBlankGroup, SizeGroup, PatternGroup})` → **NotBlank → Size → Pattern 순서로 단계 검증되며, 앞 단계 실패 시 뒷 단계는 실행되지 않음**(그룹별 첫 실패에서 시퀀스 중단).

---

### 부원 관리

#### 1. 소속 동아리 회원 조회
`GET /club-leader/{clubUUID}/members`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | - |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |
| 쿼리 | sort | String | X | 기본값 `default`. 허용값: `default`, `regular-member`, `non-member` (컨트롤러에서 `sort.toLowerCase()` 후 분기 → 대소문자 무관) |

**응답** — `ApiResponse<List<ClubMembersResponse>>`

sort 값에 따라 `message`가 달라짐:
- `default` → `"소속 동아리 회원 가나다순 조회 완료"`
- `regular-member` → `"소속 동아리 정회원 조회 완료"`
- `non-member` → `"소속 동아리 비회원 조회 완료"`

| 필드 | 타입 | 설명 |
|---|---|---|
| data[].clubMemberUUID | UUID | 동아리 회원 UUID (퇴출·수정 시 사용) |
| data[].userName | String | 이름 |
| data[].major | String | 학과 |
| data[].studentNumber | String | 학번(8자리) |
| data[].userHp | String | 전화번호(하이픈 없는 11자리) |
| data[].memberType | Enum | `REGULARMEMBER` / `NONMEMBER` |

```json
{
  "message": "소속 동아리 회원 가나다순 조회 완료",
  "data": [
    {
      "clubMemberUUID": "11111111-1111-1111-1111-111111111111",
      "userName": "홍길동",
      "major": "컴퓨터학부",
      "studentNumber": "20250001",
      "userHp": "01012345678",
      "memberType": "REGULARMEMBER"
    }
  ]
}
```
HTTP 200

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `default`: `findAllWithProfileByName(clubId)` — JPQL `ORDER BY p.userName ASC` (이름 오름차순)
3. `regular-member` / `non-member`: `findAllWithProfileByMemberType(clubId, memberType)` — **정렬 절 없음**(DB 반환 순서)
4. 그 외 문자열 → `ProfileException(INVALID_MEMBER_TYPE)`
5. 부수효과 없음 (`@Transactional(readOnly = true)`)

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| PFL-208 | 400 | 유효하지 않은 회원 종류입니다. | sort가 허용 3값이 아님 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

#### 2. 동아리 회원 퇴출(일괄 삭제)
`DELETE /club-leader/{clubUUID}/members`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | application/json (DELETE + 요청 바디) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

바디: `List<ClubMembersDeleteRequest>` (배열 자체가 최상위)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| [].clubMemberUUID | UUID | O | `@NotNull("삭제할 동아리 회원을 선택해주세요.")` — **단, 컨트롤러에 `@Valid`/`@Validated`가 없어 Bean Validation은 실행되지 않음** | 퇴출 대상 동아리 회원 UUID |

```json
[
  { "clubMemberUUID": "11111111-1111-1111-1111-111111111111" },
  { "clubMemberUUID": "22222222-2222-2222-2222-222222222222" }
]
```

**응답** — `ApiResponse<List<ClubMembersDeleteRequest>>` (요청 바디를 그대로 에코)

```json
{
  "message": "동아리 회원 삭제 완료",
  "data": [
    { "clubMemberUUID": "11111111-1111-1111-1111-111111111111" },
    { "clubMemberUUID": "22222222-2222-2222-2222-222222222222" }
  ]
}
```
HTTP 200

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. 요청 UUID 목록 추출 → `findByClubClubIdAndClubMemberUUIDIn(clubId, uuids)`
3. **all-or-nothing**: 조회 건수 ≠ 요청 건수이면 `CLUB_MEMBER_NOT_EXISTS`(404)로 전체 롤백. 하나라도 타 동아리 소속/존재하지 않는 UUID면 **아무도 삭제되지 않음**. (요청 배열에 같은 UUID가 중복되면 조회 건수 < 요청 건수가 되어 동일 예외 발생)
4. `clubMembersRepository.deleteAll(membersToDelete)`
5. **고아 프로필 정리**: 삭제 대상 중 `memberType == NONMEMBER`인 프로필 ID를 모아 `findByProfileProfileIdsWithoutClub`(`NOT EXISTS`로 어떤 ClubMembers에도 연결되지 않은 프로필 조회) 실행 → 결과가 있으면 `profileRepository.deleteAllByIdInBatch(...)`로 프로필 자체를 삭제
   - 즉 **비회원(엑셀로 추가된 회원)이 마지막 동아리에서 퇴출되면 프로필도 함께 영구 삭제**됨
   - `REGULARMEMBER` 프로필은 어떤 경우에도 삭제되지 않음

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CMEM-201 | 404 | 동아리 회원이 존재하지 않습니다. | 요청 UUID 중 하나라도 해당 동아리 회원이 아님 / UUID 중복 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

#### 3. 동아리 회원 엑셀 내보내기
`GET /club-leader/{clubUUID}/members/export`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type(응답) | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

바디 없음.

**응답**

응답 헤더 (서비스에서 `HttpServletResponse`에 직접 세팅):

| 헤더 | 값 |
|---|---|
| Content-Type | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| Content-Disposition | `attachment; filename={encodedFileName}` |

- 원본 파일명: `{동아리명}_회원_명단.xlsx`
- 인코딩: `URLEncoder.encode(fileName, "UTF-8")` — **`filename*=UTF-8''` 형식이 아니고 따옴표도 없음**. `URLEncoder`는 공백을 `+`로, 한글을 `%XX`로 변환하므로 클라이언트에서 `decodeURIComponent` + `+`→공백 치환이 필요할 수 있음.

엑셀 내용:
- 시트 이름 = 동아리명
- 0행 = 헤더, 셀 스타일: 배경 RGB(74,119,202) 단색 채움, 가로/세로 가운데 정렬, 상하좌우 THIN 테두리
- 컬럼(0~3): **`학과` | `학번` | `이름` | `전화번호`** (각각 `profile.major`, `studentNumber`, `userName`, `userHp`)
- 1행부터 회원 데이터. 조회는 `findAllWithProfileByClubClubId` — **ORDER BY 없음**
- 형식은 XSSF(.xlsx)

HTTP 상태코드: 200

**코드상 특이사항 (프론트 주의)**
서비스가 워크북 바이트를 `response.getOutputStream()`에 쓰고 `flushBuffer()`까지 호출한 뒤, 컨트롤러가 다시 `ResponseEntity<ApiResponse>(new ApiResponse<>("동아리 회원 엑셀 파일 내보내기 완료"), HttpStatus.OK)`를 반환함. 응답은 이미 커밋된 상태이므로 **엑셀 바이너리 뒤에 JSON이 추가로 이어붙을 수 있음**. 정상 케이스에서 순수 xlsx만 오는지는 서블릿 컨테이너 동작에 의존 — **코드상 단정 불가**.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| FILE-301 | 400 | 파일 이름 인코딩에 실패했습니다. | `URLEncoder.encode` 중 IOException |
| FILE-302 | 500 | 파일 생성에 실패했습니다. | 워크북 생성/스트림 기록 중 IOException |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

#### 4. 비회원 프로필 수정
`PATCH /club-leader/{clubUUID}/members/{clubMemberUUID}/non-member`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |
| 경로변수 | clubMemberUUID | UUID | O | 수정 대상 동아리 회원 UUID |

바디: `ClubNonMemberUpdateRequest` (`@Validated(ValidationSequence.class)`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| userName | String | O | `@NotBlank`, `@Size(min=2, max=30)`, `@Pattern(^[a-zA-Z가-힣]+$)` | 이름. 영문/한글만, 공백·숫자 불가 |
| studentNumber | String | O | `@NotBlank`, `@Size(min=8, max=8)`, `@Pattern(^[0-9]{8}$)` | 학번 8자리 숫자 |
| userHp | String | O | `@NotBlank`, `@Size(min=11, max=11)`, `@Pattern(^01[0-9]{9}$)` | 전화번호 11자리, 하이픈 없이 |
| major | String | O | `@NotBlank`, `@Size(min=1, max=20)`, `@Pattern(^[가-힣a-zA-Z]+$)` | 학과. 한글/영문만 (공백 불가) |

```json
{
  "userName": "홍길동",
  "studentNumber": "20250001",
  "userHp": "01012345678",
  "major": "컴퓨터학부"
}
```

**응답** — `ApiResponse<ClubNonMemberUpdateRequest>` (요청 에코)

```json
{
  "message": "비회원 프로필 업데이트 완료",
  "data": {
    "userName": "홍길동",
    "studentNumber": "20250001",
    "userHp": "01012345678",
    "major": "컴퓨터학부"
  }
}
```
HTTP 200

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `findByClubClubIdAndClubMemberUUID` → 없으면 `CMEM-201`(404)
3. **`profile.memberType != NONMEMBER`이면 `PFL-206`(400)** — 정회원 프로필은 회장이 수정할 수 없음
4. `profile.updateProfile(userName, studentNumber, major, userHp)` — 각 필드가 null이 아닐 때만 반영하고, 내부 `validateProfileInput`이 null/공백이면 `COM-302` 발생. `profileUpdatedAt = now()`로 갱신
5. `profileRepository.save(profile)`

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CMEM-201 | 404 | 동아리 회원이 존재하지 않습니다. | clubMemberUUID가 해당 동아리 회원이 아님 |
| PFL-206 | 400 | 비회원만 수정할 수 있습니다. | 대상 프로필이 `REGULARMEMBER` |
| COM-302 | 400 | 잘못된 입력 값입니다. | `Profile.validateProfileInput` 실패(공백) |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | DTO 검증 실패 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

### 지원자 관리

#### 5. 최초 지원자 조회
`GET /club-leader/{clubUUID}/applicants`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | - |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

**응답** — `ApiResponse<List<ApplicantsResponse>>`

| 필드 | 타입 | 설명 |
|---|---|---|
| data[].aplictUUID | UUID | 지원서 UUID (합/불 처리 시 사용) |
| data[].userName | String | 지원자 이름 |
| data[].major | String | 학과 |
| data[].studentNumber | String | 학번 |
| data[].userHp | String | 전화번호 |

```json
{
  "message": "최초 동아리 지원자 조회 완료",
  "data": [
    {
      "aplictUUID": "aaaaaaaa-1111-2222-3333-444444444444",
      "userName": "김수원",
      "major": "정보통신학부",
      "studentNumber": "20250002",
      "userHp": "01098765432"
    }
  ]
}
```
HTTP 200

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `findAllWithProfileByClubId(clubId, checked=false)` — **아직 합/불 처리(checked)되지 않은 지원서만** 조회. 정렬 절 없음
3. `@Transactional(readOnly = true)`, 부수효과 없음

**에러**: 공통(CLDR-101 / CLUB-201)만.

---

#### 6. 최초 합격/불합격 결과 처리 및 알림
`POST /club-leader/{clubUUID}/applicants/notifications`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

바디: `List<ApplicantResultsRequest>` (**배열이 최상위**, `@Validated(ValidationSequence.class)`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| [].aplictUUID | UUID | O | `@NotNull("지원서는 필수 입력값입니다.")` | 지원서 UUID |
| [].aplictStatus | Enum | O | `@NotNull("지원 상태는 필수 입력값입니다.")` / 서비스에서 **`WAIT`·null 거부** | enum `AplictStatus` = `WAIT`\|`PASS`\|`FAIL` 중 **`PASS` 또는 `FAIL`만 허용** |

> 참고: 파라미터 타입이 `List`라 `@Validated`가 요소까지 캐스케이드되는지는 코드상 단정 불가하나, 서비스가 `aplictStatus == null || == WAIT`를 직접 검사하므로 실질 방어는 서비스에서 수행됨.

```json
[
  { "aplictUUID": "aaaaaaaa-1111-2222-3333-444444444444", "aplictStatus": "PASS" },
  { "aplictUUID": "bbbbbbbb-1111-2222-3333-555555555555", "aplictStatus": "FAIL" }
]
```

**응답** — `ApiResponse<String>` (data 없음, `@JsonInclude(NON_NULL)`로 생략)

```json
{ "message": "지원 결과 처리 완료" }
```
HTTP 200

**비즈니스 규칙 (실행 순서)**
1. `validateLeaderAccess(clubUUID)`
2. `applicants = findByClub_ClubIdAndChecked(clubId, false)` — 미처리 지원서 전체 조회
3. `validateTotalApplicants(applicants, results)`
   - `results`가 null이거나 비어 있으면 → `COM-302`(400)
   - **부분집합 검증**: 요청 `aplictUUID` 집합 ⊆ 미처리 지원서 UUID 집합이어야 함. 아니면 `APT-204`(400).
   - **전원을 넣을 필요는 없음.** 미처리 지원자 일부만 골라 보내는 것이 허용됨(메시지는 "선택한 지원자 수와 전체 지원자 수가 일치하지 않습니다"지만 실제 로직은 `containsAll(requested)`, 즉 요청이 전체의 부분집합인지만 확인).
4. 요청 배열을 순회하며 각 항목에 대해:
   1. `aplictStatus`가 null 또는 `WAIT` → `COM-302`(400)
   2. `findByClub_ClubIdAndAplictUUIDAndChecked(clubId, uuid, false)` → 없으면 `APT-202`(404)
   3. `checkDuplicateClubMember(profileId, clubId)` — 이미 해당 동아리 회원이면 `CMEM-202`(400)
   4. **PASS인 경우**
      - `ClubMembers(club, profile)` 신규 생성·저장 (`clubMemberUUID`는 `UUID.randomUUID()` 기본값)
      - `aplict.updateAplictStatus(PASS, checked=true, deleteDate = LocalDateTime.now().plusDays(4))`
   5. **FAIL인 경우**
      - `ClubMembers` 생성 없음
      - `aplict.updateAplictStatus(FAIL, checked=true, deleteDate = LocalDateTime.now().plusDays(4))`
   6. `aplictRepository.save(applicant)`
   7. `fcmService.sendMessageTo(applicant, aplictResult)` 호출
5. 같은 `aplictUUID`를 한 요청에 두 번 넣으면 두 번째 조회 시 이미 `checked=true`이므로 4-2에서 `APT-202` 발생 → 전체 롤백(코드 흐름상 도출).

**부수효과**
- **DB**: PASS 지원자마다 `CLUB_MEMBERS_TABLE` 신규 행 생성. PASS/FAIL 모두 `APLICT_TABLE`의 `aplict_status`, `aplict_checked=true`, `aplict_delete_date = now+4일` 갱신
- **스케줄러 연동**: `SchedulerConfig.deleteOldApplications()`가 매일 00:00(`cron = "0 0 0 * * ?"`)에 `deleteDate < now`인 지원서를 물리 삭제 → **통보 4일 뒤 지원서 자동 삭제**
- **FCM 푸시** (`FcmServiceImpl`)
  - 대상: `aplict.profile.fcmToken`. 토큰이 null/공백이면 로그만 남기고 **전송 생략(0 반환), 예외 없음**
  - 엔드포인트: `https://fcm.googleapis.com/v1/projects/usw-circle-link/messages:send`
  - 제목(title): `"동아리 지원 결과"` (고정)
  - 본문(body): PASS → `"{동아리명}에 합격했습니다."` / FAIL → `"{동아리명}에 불합격했습니다."` / 그 외 → `"관리자에게 문의 해주세요."`
  - FCM이 401 또는 400을 반환하면 해당 프로필의 `fcmToken`을 null로 무효화 저장
  - 전송 실패(IOException)는 로그만 남기고 0 반환 → **API는 여전히 200 성공**
- 트랜잭션: 서비스 클래스 `@Transactional` → 루프 중 예외 시 DB 변경 전체 롤백. **단, 이미 전송된 FCM 푸시는 롤백되지 않음**

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| COM-302 | 400 | 잘못된 입력 값입니다. | results가 null/빈 배열, 또는 `aplictStatus`가 null/`WAIT` |
| APT-204 | 400 | 선택한 지원자 수와 전체 지원자 수가 일치하지 않습니다. | 요청 UUID가 미처리 지원서 집합의 부분집합이 아님 |
| APT-202 | 404 | 유효한 지원자가 존재하지 않습니다. | 해당 동아리의 미처리(checked=false) 지원서로 조회 실패 |
| CMEM-202 | 400 | 동아리 회원이 이미 존재합니다. | 지원자 프로필이 이미 해당 동아리 회원 |
| INVALID_REQUEST_BODY | 400 | 유효하지 않은 JSON 혹은 ENUM 값입니다. | `aplictStatus`에 `PASS/FAIL/WAIT` 외 문자열 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

#### 7. 불합격자 조회 (추가합격 후보 목록)
`GET /club-leader/{clubUUID}/failed-applicants`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | - |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

**응답** — `ApiResponse<List<ApplicantsResponse>>` (필드 구조는 5번과 동일)

```json
{
  "message": "불합격자 조회 완료",
  "data": [
    {
      "aplictUUID": "bbbbbbbb-1111-2222-3333-555555555555",
      "userName": "이화성",
      "major": "경영학부",
      "studentNumber": "20250003",
      "userHp": "01055556666"
    }
  ]
}
```
HTTP 200

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `findAllWithProfileByClubIdAndFailed(clubId, checked=true, status=FAIL)` — **`checked=true` AND `aplictStatus=FAIL`**인 지원서만
3. 6번에서 FAIL 처리된 지원서는 `deleteDate = 처리시각+4일`이 걸려 있으므로 **4일이 지나면 스케줄러가 삭제해 이 목록에서 사라짐** → 추가합격 가능 기간은 사실상 최초 통보 후 4일
4. `@Transactional(readOnly = true)`

**에러**: 공통(CLDR-101 / CLUB-201)만.

---

#### 8. 추가 합격 처리 및 알림
`POST /club-leader/{clubUUID}/failed-applicants/notifications`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

바디: `List<ApplicantResultsRequest>` (6번과 동일 DTO)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| [].aplictUUID | UUID | O | `@NotNull` | 지원서 UUID |
| [].aplictStatus | Enum | O | 서비스에서 **`PASS`가 아니면 즉시 거부** | **`PASS`만 허용** (FAIL·WAIT·null 모두 `COM-302`) |

```json
[
  { "aplictUUID": "bbbbbbbb-1111-2222-3333-555555555555", "aplictStatus": "PASS" }
]
```

**응답**

```json
{ "message": "추합 결과 처리 완료" }
```
HTTP 200

**비즈니스 규칙 (실행 순서)**
1. `validateLeaderAccess(clubUUID)`
2. **6번과 달리 `validateTotalApplicants` 호출이 없음** → 빈 배열 `[]`을 보내면 아무 처리 없이 200 성공 반환
3. 요청 배열 순회:
   1. `aplictStatus != PASS` → `COM-302`(400)
   2. `findByClub_ClubIdAndAplictUUIDAndCheckedAndAplictStatus(clubId, uuid, checked=true, status=FAIL)` → 없으면 `APT-203`(404)
      - **대상 조건: 해당 동아리 + `checked=true` + `aplictStatus=FAIL`.** 즉 최초 심사에서 불합 처리된 지원서만 추가합격 가능. 미처리(WAIT/checked=false) 지원자는 이 API로 처리 불가
   3. `checkDuplicateClubMember(profileId, clubId)` → 중복이면 `CMEM-202`(400)
   4. `ClubMembers(club, profile)` 생성·저장
   5. `applicant.updateFailedAplictStatus(PASS)` — **`aplictStatus`만 PASS로 변경. `checked`는 true 유지, `deleteDate`는 최초 통보 때 설정된 값 그대로 유지**(재연장 없음) → 해당 지원서는 예정대로 스케줄러에 의해 삭제됨
   6. `aplictRepository.save(applicant)`
   7. `fcmService.sendMessageTo(applicant, PASS)`

**부수효과**
- `CLUB_MEMBERS_TABLE` 신규 행 생성
- `APLICT_TABLE.aplict_status`만 `PASS`로 갱신 (`checked`, `deleteDate` 불변)
- **FCM 푸시**: 제목 `"동아리 지원 결과"`, 본문 `"{동아리명}에 합격했습니다."`. 토큰 없으면 전송 생략
- 트랜잭션 롤백 정책은 6번과 동일

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| COM-302 | 400 | 잘못된 입력 값입니다. | `aplictStatus != PASS` |
| APT-203 | 404 | 유효한 추합 대상자가 존재하지 않습니다. | `checked=true` + `FAIL` 조건의 지원서를 찾지 못함 |
| CMEM-202 | 400 | 동아리 회원이 이미 존재합니다. | 이미 해당 동아리 회원 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

### 엑셀 기반 기존 회원 등록 (2단계 흐름)

> **전체 흐름**
> 1단계 `POST /{clubUUID}/members/import` (엑셀 업로드) → 서버가 **신규(addClubMembers)** / **중복(duplicateClubMembers)** 으로 분류만 해서 반환 (**저장 없음**)
> 2단계-A `POST /{clubUUID}/members` : 신규 목록에 **학과(major)를 프론트에서 입력받아** 채운 뒤 전송 → Profile + ClubMembers 생성
> 2단계-B `POST /{clubUUID}/members/duplicate-profiles` : 중복 목록 중 "동일인이 맞다"고 확인한 건을 **1건씩** 전송 → 기존 Profile을 재사용해 ClubMembers만 생성
> 엑셀에는 학과 컬럼이 없어(이름/학번/전화번호 3열만 읽음) 1단계 응답에도 `major`가 없다.

#### 9. 기존 동아리 회원 엑셀 업로드(1단계: 분류)
`POST /club-leader/{clubUUID}/members/import`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | `multipart/form-data` (`consumes` 명시) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |
| multipart part | clubMembersFile | File | O (`required = true`) | `.xls` 또는 `.xlsx` 엑셀 파일 |

엑셀 파일 형식 (첫 번째 시트만 사용):

| 열 인덱스 | 내용 |
|---|---|
| 0 | 이름 |
| 1 | 학번 |
| 2 | 전화번호 |

- 0행은 헤더로 간주해 건너뜀 (`i = 1`부터 읽음)
- 모든 셀이 BLANK인 행은 무시
- 이름·학번: 모든 공백문자 제거(`replaceAll("\\s+","")`)
- 전화번호: `-` 제거 후 공백 제거
- 셀 타입이 STRING이면 문자열, NUMERIC이면 `(long)` 캐스팅 후 문자열, **그 외(수식·불리언 등)는 빈 문자열 `""`**

**응답** — `ApiResponse<ClubMembersImportExcelResponse>`

| 필드 | 타입 | 설명 |
|---|---|---|
| data.addClubMembers | List\<ExcelProfileMemberResponse\> | DB에 동일 프로필이 없는 **신규** 대상 |
| data.duplicateClubMembers | List\<ExcelProfileMemberResponse\> | DB에 이미 프로필이 존재하는 **중복** 대상 |
| (공통 요소) .userName | String | 이름 |
| (공통 요소) .studentNumber | String | 학번 |
| (공통 요소) .userHp | String | 전화번호 |

```json
{
  "message": "기존 동아리 회원 엑셀로 가져오기 완료",
  "data": {
    "addClubMembers": [
      { "userName": "홍길동", "studentNumber": "20250001", "userHp": "01012345678" }
    ],
    "duplicateClubMembers": [
      { "userName": "김수원", "studentNumber": "20250002", "userHp": "01098765432" }
    ]
  }
}
```
HTTP 200

**비즈니스 규칙 (실행 순서)**
1. `validateLeaderAccess(clubUUID)`
2. 파일 null 또는 empty → `FILE-308`(400)
3. **확장자 검증**: `FilenameUtils.getExtension(originalFilename)`이 `xls` 또는 `xlsx`가 아니면 `FILE-311`(400)
4. **파일 시그니처(매직넘버) 검증** (`FileSignatureValidator`) — 앞 4바이트를 읽어 16진수 대문자 문자열로 변환 후 `startsWith` 비교
   - `xls` → `D0CF11E0` (OLE2 복합 문서)
   - `xlsx` → `504B0304` (ZIP)
   - 불일치 → `FILE-311`(400) / 읽기 중 IOException → `FILE-312`(400)
   - **확장자만 바꾼 위장 파일 차단**
5. `xls` → `HSSFWorkbook`, `xlsx` → `XSSFWorkbook`으로 열고 `getSheetAt(0)` 사용
6. 행별로 `이름_학번_전화번호` 키로 `Map`에 저장 → **파일 내 완전 동일 행은 자동으로 1건으로 합쳐짐**
7. `profileRepository.findByUserNameInAndStudentNumberInAndUserHpIn(names, studentNumbers, hps)`로 **한 번에** 중복 후보 조회
   - 이 쿼리는 3개 `IN` 조건의 AND 조합(교차곱)이므로 "이름은 A행, 학번은 B행"인 프로필도 후보로 딸려올 수 있으나, 이후 `이름_학번_전화번호` 정확 키 매칭에 성공한 것만 `duplicateClubMembers`에 담김
8. 키 매칭된 항목은 Map에서 제거 → 남은 항목 전체가 `addClubMembers`
9. **`@Transactional(readOnly = true)` — 이 단계에서는 아무것도 저장하지 않음**
10. 파일 업로드 크기 제한: `application-file.yml`의 `max-file-size: 10MB`, `max-request-size: 50MB`

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| FILE-308 | 400 | 업로드 가능한 갯수를 초과했습니다. | 파일이 null이거나 비어 있음 (메시지와 실제 조건이 다름에 유의) |
| FILE-311 | 400 | 지원하지 않는 파일 확장자입니다. | 확장자가 xls/xlsx가 아님, 또는 파일 시그니처 불일치 |
| FILE-312 | 400 | 파일 유효성 검사 실패 | 시그니처 확인 중 IOException |
| MAX_UPLOAD_SIZE_EXCEEDED | 400 | 업로드 가능한 최대 파일 크기를 초과했습니다. (개별 파일 10MB, 총 파일 크기 50MB) | 용량 초과 |
| NO_CATCH_ERROR | 500 | (예외 메시지) | 파일명이 null이라 `getExtension`이 null을 반환해 NPE가 나는 경우 등 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

#### 10. 엑셀 신규 회원 저장(2단계-A)
`POST /club-leader/{clubUUID}/members`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

바디: `ClubMembersAddFromExcelRequestList` (`@Validated(ValidationSequence.class)`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| clubMembersAddFromExcelRequestList | List\<ClubMembersAddFromExcelRequest\> | O | `@NotEmpty("회원 목록은 비어있을 수 없습니다.")`, `@Valid` | 저장할 회원 배열 |
| └ userName | String | O | `@NotBlank`, `@Size(min=2, max=30)`, `@Pattern(^[a-zA-Z가-힣]+$)` | 이름 |
| └ major | String | O | `@NotBlank`, `@Size(min=1, max=20)` — **`@Pattern` 없음**(4번과 달리 한글/영문 제한이 없어 숫자·기호도 통과) | 학과. **엑셀에 없는 값이라 프론트가 입력받아 채워야 함** |
| └ studentNumber | String | O | `@NotBlank`, `@Size(min=8, max=8)`, `@Pattern(^[0-9]{8}$)` | 학번 8자리 |
| └ userHp | String | O | `@NotBlank`, `@Size(min=11, max=11)`, `@Pattern(^01[0-9]{9}$)` | 전화번호 11자리 |

```json
{
  "clubMembersAddFromExcelRequestList": [
    {
      "userName": "홍길동",
      "major": "컴퓨터학부",
      "studentNumber": "20250001",
      "userHp": "01012345678"
    }
  ]
}
```

**응답**

```json
{ "message": "엑셀로 추가된 기존 동아리 회원 저장 완료" }
```
HTTP 200 (data 없음)

**비즈니스 규칙 (실행 순서)**
1. `validateLeaderAccess(clubUUID)`
2. 각 요청 항목의 `major`가 null 또는 trim 후 빈 문자열이면 → `PFL-205`(400) (`@NotBlank` 통과분에 대한 서비스 레벨 재검증)
3. `이름_학번_전화번호_학과` 키로 Map 구성 — **완전 동일한 항목이 배열에 중복되면 1건으로 합쳐짐**
4. `findByUserNameInAndStudentNumberInAndUserHpInAndMajorIn(...)`으로 DB 중복 조회 → 정확 키(4필드 전부 일치) 매칭 건을 `duplicateUsers`에 수집
5. **all-or-nothing**: `duplicateUsers`가 하나라도 있으면 `ProfileException(DUPLICATE_PROFILE, duplicateUsers)` → **한 명도 저장되지 않음**
   - 에러 응답 `additionalData`에 중복자 목록이 실려옴 (한글 키 사용):
     ```json
     {
       "exception": "ProfileException",
       "code": "PFL-204",
       "message": "이미 존재하는 회원입니다.",
       "status": 400,
       "error": "Bad Request",
       "additionalData": [
         { "이름": "홍길동", "학번": "20250001", "전화번호": "01012345678", "전공": "컴퓨터학부" }
       ]
     }
     ```
6. 중복이 없으면 각 항목마다
   - `Profile` 생성: `userName`, `studentNumber`, `userHp`, `major`, `profileCreatedAt = now()`, `profileUpdatedAt = now()`, **`memberType = NONMEMBER`**, `user = null` → 저장
   - `ClubMembers(club, profile)` 생성 → 저장 (`clubMemberUUID` 자동 생성)

**부수효과**: `PROFILE_TABLE`·`CLUB_MEMBERS_TABLE` 신규 행 생성. 메일/푸시/S3 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| PFL-205 | 400 | 학과 정보는 필수 입력 항목입니다. | `major`가 null/공백 |
| PFL-204 | 400 | 이미 존재하는 회원입니다. | 4필드가 모두 일치하는 프로필이 DB에 이미 존재. `additionalData`에 중복 목록 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | 배열이 비었거나 요소 검증 실패 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

#### 11. 프로필 중복 회원 동아리에 추가(2단계-B)
`POST /club-leader/{clubUUID}/members/duplicate-profiles`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

바디: `DuplicateProfileMemberRequest` (`@Validated(ValidationSequence.class)`) — **단건만 처리(배열 아님)**

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| userName | String | O | `@NotBlank`, `@Size(min=2, max=30)`, `@Pattern(^[a-zA-Z가-힣]+$)` | 이름 |
| studentNumber | String | O | `@NotBlank`, `@Size(min=8, max=8)`, `@Pattern(^[0-9]{8}$)` | 학번 |
| userHp | String | O | `@NotBlank`, `@Size(min=11, max=11)`, `@Pattern(^01[0-9]{9}$)` | 전화번호 |

> `major`는 요청에 없음 — 기존 프로필의 학과를 그대로 사용.

```json
{
  "userName": "김수원",
  "studentNumber": "20250002",
  "userHp": "01098765432"
}
```

**응답** — `ApiResponse<DuplicateProfileMemberRequest>` (요청 에코)

```json
{
  "message": "프로필 중복 동아리 회원 추가 완료",
  "data": {
    "userName": "김수원",
    "studentNumber": "20250002",
    "userHp": "01098765432"
  }
}
```
HTTP 200

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `findByUserNameAndStudentNumberAndUserHp(userName, studentNumber, userHp)` — 3필드 완전 일치 프로필 조회. 없으면 `PFL-201`(404)
3. `checkDuplicateClubMember(profileId, clubId)` → 이미 해당 동아리 회원이면 `CMEM-202`(400)
4. **기존 Profile을 그대로 재사용**해 `ClubMembers(club, profile)`만 신규 생성·저장 (프로필 신규 생성 없음, `memberType` 변경 없음)

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| PFL-201 | 404 | 프로필이 존재하지 않습니다. | 3필드 일치 프로필 없음 |
| CMEM-202 | 400 | 동아리 회원이 이미 존재합니다. | 이미 해당 동아리 회원 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | DTO 검증 실패 |
| NO_CATCH_ERROR | 500 | (예외 메시지) | `findByUserNameAndStudentNumberAndUserHp`가 `Optional` 단건 조회라 동일 3필드 프로필이 2건 이상이면 `NonUniqueResultException` 발생 가능(코드상 도출) |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

### 기존 회원 가입 요청 처리

#### 12. 기존 동아리 회원 가입 요청 조회
`GET /club-leader/{clubUUID}/members/sign-up`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | - |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

**응답** — `ApiResponse<List<SignUpRequestResponse>>`

| 필드 | 타입 | 설명 |
|---|---|---|
| data[].clubMemberAccountStatusUUID | UUID | 가입 요청 UUID (수락/거절 시 사용) |
| data[].profileTempName | String | 신청자가 입력한 이름 |
| data[].profileTempStudentNumber | String | 신청자가 입력한 학번 |
| data[].profileTempMajor | String | 신청자가 입력한 학과 |
| data[].profileTempHp | String | 신청자가 입력한 전화번호 |

```json
{
  "message": "기존 동아리 회원 가입 요청 조회 완료",
  "data": [
    {
      "clubMemberAccountStatusUUID": "cccccccc-1111-2222-3333-666666666666",
      "profileTempName": "김수원",
      "profileTempStudentNumber": "20250002",
      "profileTempMajor": "정보통신학부",
      "profileTempHp": "01098765432"
    }
  ]
}
```
HTTP 200

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `findAllWithClubMemberTemp(clubId)` — `ClubMemberAccountStatus` + `ClubMemberTemp` fetch join. 정렬 절 없음
3. `@Transactional(readOnly = true)`

**에러**: 공통(CLDR-101 / CLUB-201)만.

---

#### 13. 가입 요청 거절(삭제)
`DELETE /club-leader/{clubUUID}/members/sign-up/{clubMemberAccountStatusUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | - |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |
| 경로변수 | clubMemberAccountStatusUUID | UUID | O | 거절할 가입 요청 UUID (12번 응답의 `clubMemberAccountStatusUUID`) |

바디 없음.

**응답**

```json
{ "message": "기존 동아리 회원 가입 요청 거절 완료" }
```
HTTP 200 (data 없음)

**비즈니스 규칙**
1. `validateLeaderAccess(clubUUID)`
2. `findByClubMemberAccountStatusUUIDAndClub_ClubUUID(uuid, club.clubUUID)` → 없으면 `CMEMT-201`(404)
3. `clubMemberAccountStatusRepository.delete(...)`

**부수효과**
- `CLUB_MEMBER_ACCOUNTSTATUS_TABLE` 행 삭제만 수행
- **`ClubMemberTemp`(임시 회원)는 삭제하지 않고 `clubRequestCount`도 증가시키지 않음** → 해당 신청자는 `totalClubRequest == clubRequestCount` 조건을 영영 충족할 수 없어 계정이 생성되지 않으며, `SchedulerConfig`의 만료 임시회원 정리 대상이 됨
- 메일/푸시 없음

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CMEMT-201 | 404 | 회원 가입 요청이 존재하지 않습니다. | 해당 동아리의 가입 요청이 아님/존재하지 않음 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

#### 14. 가입 요청 수락
`POST /club-leader/{clubUUID}/members/sign-up`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 경로변수 | clubUUID | UUID | O | 동아리 UUID |

바디: `ClubMembersAcceptSignUpRequest` (`@Validated(ValidationSequence.class)`)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| signUpProfileRequest | ClubMemberProfileRequest | O | `@NotNull`, `@Valid` | **가입 요청 측** 정보 |
| clubNonMemberProfileRequest | ClubMemberProfileRequest | O | `@NotNull`, `@Valid` | **동아리에 이미 등록된 비회원** 측 정보 |

`ClubMemberProfileRequest` 공통 필드:

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| uuid | UUID | O | `@NotNull("대상을 선택해주세요.")` | `signUpProfileRequest`에서는 **`clubMemberAccountStatusUUID`**, `clubNonMemberProfileRequest`에서는 **`clubMemberUUID`** (DTO 주석에 명시) |
| userName | String | O | `@NotBlank`, `@Size(min=2, max=30)`, `@Pattern(^[a-zA-Z가-힣]+$)` | 이름 |
| studentNumber | String | O | `@NotBlank`, `@Size(min=8, max=8)`, `@Pattern(^[0-9]{8}$)` | 학번 8자리 |
| userHp | String | O | `@NotBlank`, `@Size(min=11, max=11)`, `@Pattern(^01[0-9]{9}$)` | 전화번호 11자리 |
| major | String | O | `@NotBlank`, `@Size(min=1, max=20)`, `@Pattern(^[가-힣a-zA-Z]+$)` | 학과 |

```json
{
  "signUpProfileRequest": {
    "uuid": "cccccccc-1111-2222-3333-666666666666",
    "userName": "김수원",
    "studentNumber": "20250002",
    "userHp": "01098765432",
    "major": "정보통신학부"
  },
  "clubNonMemberProfileRequest": {
    "uuid": "11111111-1111-1111-1111-111111111111",
    "userName": "김수원",
    "studentNumber": "20250002",
    "userHp": "01098765432",
    "major": "정보통신학부"
  }
}
```

**응답** — 두 가지 분기

(a) 아직 다른 동아리 승인이 남은 경우 (`totalClubRequest != clubRequestCount`)
```json
{ "message": "기존 동아리 회원 가입 요청 수락 완료" }
```

(b) 신청한 모든 동아리가 수락 완료된 경우 → 계정 생성
```json
{
  "message": "기존 동아리 회원 가입 요청 수락 후 계정 생성 완료",
  "data": "김수원"
}
```
`data`는 `String` (생성된 계정 주인의 이름 = `clubNonMember.getUserName()`). 프론트에서 "계정이 생성되었습니다" 팝업 용도.

HTTP 200

**비즈니스 규칙 — 4필드 3중 대조 (실행 순서)**
1. `validateLeaderAccess(clubUUID)`
2. **대조 ①: `signUpProfileRequest` vs `ClubMemberTemp`(DB)** — `validateSignUpProfile`
   - `findByClubMemberAccountStatusUUIDAndClub_ClubId(uuid, clubId)` → 없으면 `CMEMT-201`(404)
   - `userName`↔`profileTempName`, `studentNumber`↔`profileTempStudentNumber`, `major`↔`profileTempMajor`, `userHp`↔`profileTempHp` **4필드 전부 `Objects.equals`로 비교**, 하나라도 다르면 동일하게 `CMEMT-201`(404)
3. **대조 ②: `clubNonMemberProfileRequest` vs `Profile`(DB)** — `validateClubNonMemberProfile`
   - `findByClubClubIdAndClubMemberUUID(clubId, uuid)` → `.map(ClubMembers::getProfile)` 실패 시 `CMEM-201`(404)
   - `userName`, `studentNumber`, `major`, `userHp` 4필드 전부 비교, 하나라도 다르면 `CMEM-201`(404)
4. **대조 ③: 두 요청 DTO 상호 비교** — `compareProfile`
   - 불일치 필드명을 순서대로 리스트에 수집: `"이름"`(userName) → `"학번"`(studentNumber) → `"학과"`(major) → `"전화번호"`(userHp)
   - 리스트가 비어있지 않으면 `ProfileException(PROFILE_VALUE_MISMATCH, message)` → **`additionalData`에 불일치 항목 한글 이름 배열 반환**
     ```json
     {
       "exception": "ProfileException",
       "code": "PFL-209",
       "message": "프로필 값이 일치하지 않습니다.",
       "status": 400,
       "error": "Bad Request",
       "additionalData": ["학과", "전화번호"]
     }
     ```
     프론트는 이 배열로 "어느 항목이 다른지" 하이라이트 가능
5. 세 대조를 모두 통과하면:
   - `clubMemberTemp.updateClubRequestCount()` → `clubRequestCount += 1`, 저장
   - `clubMemberAccountStatusRepository.delete(clubMemberAccountStatus)` — 이 동아리의 가입 요청 행 삭제
6. **계정 생성 조건**: `clubMemberTemp.getTotalClubRequest() == clubMemberTemp.getClubRequestCount()`
   (신청자가 기재한 소속 동아리 수만큼 모든 회장이 수락해야 함)
   - `User` 생성: `userAccount = profileTempAccount`, `userPw = profileTempPw`, `email = profileTempEmail`, `userCreatedAt/UpdatedAt = now()`, **`role = Role.USER`** → 저장
   - 대조 ②에서 찾은 비회원 `Profile`을 `memberType = REGULARMEMBER`로 변경
   - `profile.updateUser(user)` — 엑셀로 등록됐던 프로필에 실제 계정 연결
   - `profileRepository.save(profile)`
   - `clubMemberTempRepository.delete(clubMemberTemp)` — 임시 회원 정보 삭제

**부수효과 요약**
- `CLUB_MEMBERTEMP_TABLE.clubRequestCount` 증가
- `CLUB_MEMBER_ACCOUNTSTATUS_TABLE` 해당 행 삭제
- (조건 충족 시) `USER_TABLE` 신규 계정 생성, `PROFILE_TABLE`의 `member_type`→`REGULARMEMBER` + `user_id` 연결, `CLUB_MEMBERTEMP_TABLE` 행 삭제
- **메일/FCM 푸시/S3/Redis/쿠키 부수효과 없음**
- 트랜잭션: 클래스 `@Transactional` → 예외 시 전부 롤백

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CMEMT-201 | 404 | 회원 가입 요청이 존재하지 않습니다. | 가입 요청 UUID 조회 실패, 또는 요청값 4필드가 `ClubMemberTemp`와 불일치 |
| CMEM-201 | 404 | 동아리 회원이 존재하지 않습니다. | 비회원 UUID 조회 실패, 또는 요청값 4필드가 `Profile`과 불일치 |
| PFL-209 | 400 | 프로필 값이 일치하지 않습니다. | 두 요청 DTO의 4필드 중 불일치 존재. `additionalData` = `["이름","학번","학과","전화번호"]` 중 해당 항목 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | DTO 검증 실패 |
| CLDR-101 / CLUB-201 | 403 / 404 | (공통) | |

---

### 기타

#### 15. FCM 토큰 갱신
`PATCH /club-leader/fcmtoken`

| 항목 | 값 |
|---|---|
| 인증 | **ROLE_USER** (경로는 `/club-leader`지만 권한은 USER) |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

> **권한 근거**: `SecurityConfig` 69행 `auth.requestMatchers(HttpMethod.PATCH, "/profiles/change", "/users/userpw", "/club-leader/fcmtoken").hasRole("USER");`가 80행의 `PATCH /club-leader/**` → `hasRole("LEADER")`보다 **먼저 등록**되어 있어 더 구체적인 규칙이 우선 적용됨. 서비스(`FcmServiceImpl.refreshFcmToken`)도 principal을 `CustomUserDetails`로 캐스팅하므로 회장 토큰으로 호출하면 `ClassCastException` → 500(`NO_CATCH_ERROR`).

**요청**

바디: `FcmTokenRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| fcmToken | String | - | **검증 애노테이션 없음** (null도 통과) | 앱에서 발급받은 FCM 등록 토큰 |

```json
{ "fcmToken": "fcm-registration-token-sample-value" }
```

**응답**

```json
{ "message": "fcm token 갱신 완료" }
```
HTTP 200 (data 없음)

**비즈니스 규칙**
1. SecurityContext principal → `CustomUserDetails` → `User`
2. `profileRepository.findByUserUserId(user.getUserId())`
3. 프로필이 **있으면**: `profile.updateFcmTokenTime(fcmToken, LocalDateTime.now().plusDays(60))`
   - `FCM_TOKEN_CERTIFICATION_TIME = 60` → 만료 시각 = **현재 + 60일** (`SchedulerConfig`의 주석은 "7일"이라고 되어 있으나 실제 상수는 60일)
   - 저장
4. 프로필이 **없으면**: 경고 로그만 남기고 **예외 없이 그대로 200 성공 반환** (조용한 실패)
5. 만료 처리: `SchedulerConfig.deleteExpiredFcmTokens()`가 매일 00:00에 `fcmTokenCertificationTimestamp < now`인 프로필의 `fcmToken`을 null로 설정

**부수효과**: `PROFILE_TABLE.fcm_token`, `fcm_token_updated_at` 갱신. 외부 호출 없음.

**에러**

| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| TOK-204 | 401 | 인증되지 않은 사용자입니다. | 토큰 없음 |
| (Spring Security) | 403 | - | ROLE_USER가 아닌 토큰(예: LEADER)으로 호출 |
| NO_CATCH_ERROR | 500 | (예외 메시지) | principal이 `CustomUserDetails`가 아닐 때 `ClassCastException` |

---

#### 16. (비활성) 소속 동아리원 조회 구버전
`GET /club-leader/v1/members` — **주석 처리되어 있어 호출 불가**

`ClubLeaderController` 108~112행이 통째로 주석 처리됨:
```java
//    @GetMapping("/v1/members")
//    public ResponseEntity<ApiResponse> getClubMembers(LeaderToken token) {
//        // 원래는 GET 요청임 토큰때문
//        return new ResponseEntity<>(clubLeaderService.findClubMembers(token), HttpStatus.OK);
//    }
```
- 대응 서비스 메서드 `ClubLeaderService.findClubMembers(LeaderToken)`(503~522행)도 함께 주석 처리됨
- 주석 상 "성능 비교용" 구버전 구현이며, 응답 DTO는 `clubMemberId`(Long)를 사용했음 (현행 1번 API는 `clubMemberUUID` 사용)
- **호출 시 `RESOURCE_NOT_FOUND`(404, "요청하신 경로를 찾을 수 없습니다.") 반환**
- 대체 엔드포인트: **1번 `GET /club-leader/{clubUUID}/members`**

---

## 8. 관리자 API

### B-0. 인가 규칙 요약 (SecurityConfig)

```
GET  /admin/clubs, /admin/clubs/{clubUUID}   → hasAnyRole(ADMIN, LEADER)
GET  /notices, /notices/{noticeUUID}         → hasAnyRole(ADMIN, LEADER)
GET|POST|PUT|DELETE /admin/**                → hasRole(ADMIN)
POST|DELETE /notices/**                      → hasRole(ADMIN)
그 외                                        → authenticated()
```
> **중요**: `GET /admin/clubs/category`는 AntPathMatcher상 `/admin/clubs/{clubUUID}` 패턴에 매칭되므로 **ADMIN + LEADER 모두 접근 가능**합니다.
> `PUT /notices/{noticeUUID}`는 PUT 전용 `/notices/**` 규칙이 없어 `anyRequest().authenticated()`로 떨어집니다 → **인증만 되면 USER/LEADER도 호출 가능** (코드상 사실).

---

#### 1. 운영팀 로그인
`POST /admin/login`

| 항목 | 값 |
|---|---|
| 인증 | permitAll (`application-security.yml` permit-all-paths에 `/admin/login` 포함) |
| 레이트리밋 | `WEB_LOGIN` 5회/5분 (키: `{adminAccount}:WEB_LOGIN`) |
| Content-Type | application/json |

**요청 바디** — `AdminLoginRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| adminAccount | String | O | `@NotBlank` "아이디는 필수 입력 값입니다." | 운영팀 계정 아이디. `getClientId()` 반환값 = 레이트리밋 키 |
| adminPw | String | O | `@NotBlank` "비밀번호는 필수 입력 값입니다." | 운영팀 비밀번호 |

```json
{ "adminAccount": "admin_test", "adminPw": "Test1234!@" }
```

**응답** — `ApiResponse<AdminLoginResponse>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| message | String | "운영팀 로그인 성공" |
| data.accessToken | String | JWT (유효 30분) |
| data.refreshToken | String | 랜덤 UUID 문자열 (쿠키로도 동시 전달) |
| data.role | String | `ADMIN` |

```json
{
  "message": "운영팀 로그인 성공",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIwMDAwMDAwMC0wMDAwLTAwMDAtMDAwMC0wMDAwMDAwMDAwMDEifQ.xxxxx",
    "refreshToken": "11111111-2222-3333-4444-555555555555",
    "role": "ADMIN"
  }
}
```

**비즈니스 규칙**
1. `@Validated(ValidationSequence.class)` — Default → NotBlank → Size → Pattern 순서로 그룹 검증(앞 그룹 실패 시 뒤 그룹 미실행)
2. 레이트리밋 AOP가 먼저 동작(`AdminLoginService.adminLogin`에 `@RateLimite`)
3. `adminAccount`로 Admin 조회 → 없으면 인증 실패 예외(계정 존재 여부를 노출하지 않기 위해 동일 예외 사용)
4. BCrypt 비밀번호 비교 실패 시 동일 예외
5. accessToken 발급 → 응답 헤더 `Authorization: Bearer {token}` 설정
6. 기존 refreshToken을 Redis에서 삭제(1계정 1세션) 후 새 refreshToken 생성

**부수효과**
- Redis: `refreshToken:{UUID}` → adminUUID, TTL 7일 저장
- 응답 헤더: `Authorization`, `Set-Cookie: refreshToken=...; HttpOnly; ...`
- Redis: Bucket4j 버킷 카운트 소모

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | adminAccount/adminPw 공백 |
| ATTEMPT-503 | 400 | 최대 시도 횟수를 초과했습니다. 5분 후  다시 시도 하세요 | 동일 계정 5분 내 6회 시도 |
| USR-211 | 401 | 아이디 혹은 비밀번호가 일치하지 않습니다 | 계정 미존재 또는 비밀번호 불일치 |

---

#### 2. 동아리 목록 조회
`GET /admin/clubs`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN 또는 ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 쿼리 | page | int | X | 페이지 번호(0-base), 기본값 `0` |
| 쿼리 | size | int | X | 페이지 크기, 기본값 `10` |

```
GET /admin/clubs?page=0&size=10
```

**응답** — `ApiResponse<AdminClubPageListResponse>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data.content[].clubUUID | UUID | 동아리 UUID |
| data.content[].department | String | 학부. `Department` enum의 **한글 value로 직렬화**(`@JsonValue`) — 학술/종교/예술/체육/공연/봉사 |
| data.content[].clubName | String | 동아리명 |
| data.content[].leaderName | String | 회장 이름(생성 직후엔 빈 문자열) |
| data.content[].numberOfClubMembers | long | 동아리원 수 = `COUNT(DISTINCT clubMemberId) + (회장 존재 시 1)` |
| data.totalPages | int | 전체 페이지 수 |
| data.totalElements | long | 전체 동아리 수 |
| data.currentPage | int | 현재 페이지 번호 |

```json
{
  "message": "동아리 리스트 조회 성공",
  "data": {
    "content": [
      { "clubUUID": "aaaaaaaa-1111-2222-3333-444444444444", "department": "학술", "clubName": "코드동", "leaderName": "홍길동", "numberOfClubMembers": 24 }
    ],
    "totalPages": 3,
    "totalElements": 25,
    "currentPage": 0
  }
}
```

**비즈니스 규칙**
1. JPQL로 Club LEFT JOIN ClubMembers / Leader, `GROUP BY clubId, clubUUID, department, clubName, leaderName`
2. 전체 건수는 `SELECT COUNT(c) FROM Club c`로 별도 조회
3. `Sort.by("clubId").descending()`을 넘기지만 JPQL에 ORDER BY가 없어 **정렬 미보장**
4. 부수효과 없음(읽기 전용 트랜잭션)

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| TOKEN_EXPIRED / INVALID_TOKEN / AUTH_REQUIRED | 401 | (EntryPoint 포맷) | 토큰 만료/변조/미첨부 |
| — | 403 | Spring Security 기본 | ROLE_USER로 호출 |
| NO_CATCH_ERROR | 500 | — | `page < 0` 또는 `size < 1` 시 `PageRequest.of`가 IllegalArgumentException 발생 (전용 핸들러 없음) |

---

#### 3. 동아리 소개/모집글 상세 조회
`GET /admin/clubs/{clubUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN 또는 ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | clubUUID | UUID | O | 동아리 UUID |

**응답** — `ApiResponse<AdminClubIntroResponse>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| clubUUID | UUID | 동아리 UUID |
| mainPhoto | String | 메인 사진 GET presigned URL (없으면 null, S3 Key가 빈 문자열이면 `""`) |
| introPhotos | List&lt;String&gt; | 소개 사진 GET presigned URL 목록, `order` 오름차순 (생성 시 5개 슬롯 기본 생성) |
| clubName | String | 동아리명 |
| leaderName | String | 회장 이름 |
| leaderHp | String | 회장 연락처 |
| clubInsta | String | 인스타그램 |
| clubIntro | String | 동아리 소개글 |
| recruitmentStatus | String | `OPEN` / `CLOSE` |
| googleFormUrl | String | 구글폼 URL |
| clubHashtags | List&lt;String&gt; | 해시태그 목록 |
| clubCategoryNames | List&lt;String&gt; | 카테고리명 목록 |
| clubRoomNumber | String | 동아리방 호수 |
| clubRecruitment | String | 모집글 |

```json
{
  "message": "동아리 소개/모집글 페이지 조회 성공",
  "data": {
    "clubUUID": "aaaaaaaa-1111-2222-3333-444444444444",
    "mainPhoto": "https://bucket.s3.ap-northeast-2.amazonaws.com/mainPhoto/xxxx.png?X-Amz-Signature=...",
    "introPhotos": ["https://.../introPhoto/aaa.png?X-Amz-Signature=..."],
    "clubName": "코드동",
    "leaderName": "홍길동",
    "leaderHp": "010-0000-0000",
    "clubInsta": "code_dong",
    "clubIntro": "코딩 동아리입니다.",
    "recruitmentStatus": "CLOSE",
    "googleFormUrl": "https://forms.gle/example",
    "clubHashtags": ["코딩", "스터디"],
    "clubCategoryNames": ["학술"],
    "clubRoomNumber": "B101",
    "clubRecruitment": "3월 모집합니다."
  }
}
```

**비즈니스 규칙 / 부수효과**
1. `clubUUID`로 Club 조회 → 없으면 CLUB-201
2. clubId로 ClubIntro 조회 → 없으면 CINT-201
3. 메인 사진 / 소개 사진 각각 S3 **GET presigned URL(1시간)** 생성 → 응답 시점마다 새 URL 발급
4. 해시태그·카테고리 매핑 조회

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | clubUUID가 UUID 형식이 아님 |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 동아리 미존재 |
| CINT-201 | 404 | 해당 동아리 소개글이 존재하지 않습니다. | ClubIntro 미존재 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 중 S3/SDK 오류 |

---

#### 4. 동아리 생성
`POST /admin/clubs`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청 바디** — `AdminClubCreationRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| leaderAccount | String | O | `@NotBlank`; `@Size(min=5,max=20)` "아이디는 5~20자 이내여야 합니다."; `@Pattern(^[a-zA-Z0-9]+$)` "아이디는 영문 대/소문자, 숫자만 포함할 수 있으며 공백을 포함할 수 없습니다." | 동아리 회장 로그인 아이디 |
| leaderPw | String | O | `@NotBlank`; `@Size(min=8,max=20)`; `@Pattern(^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\\|,.<>/?])(?!.*\s).*$)` "비밀번호는 영문, 숫자, 특수문자를 모두 포함해야 하며 공백을 포함할 수 없습니다." | 회장 비밀번호 (영문+숫자+특수문자 필수, 공백 불가) |
| leaderPwConfirm | String | O | `@NotBlank` "비밀번호 확인은 필수 입력 값입니다." | 비밀번호 확인 (정규식 검증 없음, 서비스에서 동등 비교) |
| clubName | String | O | `@NotBlank`; `@Size(min=1,max=10)` "동아리명은 최대 10자까지 입력 가능합니다."; `@Pattern(^[가-힣a-zA-Z0-9]+$)` "동아리명에는 한글, 영문 대소문자, 숫자만 포함할 수 있으며 공백 또는 특수문자를 포함할 수 없습니다." | 동아리명 |
| department | String(enum) | O | `@NotNull` "학부는 필수 입력 값입니다." | `ACADEMIC/RELIGION/ART/SPORT/SHOW/VOLUNTEER` 또는 한글 값 `학술/종교/예술/체육/공연/봉사` 모두 허용(`@JsonCreator`, 대소문자 무시) |
| adminPw | String | O | `@NotBlank` "운영자 비밀번호는 필수 입력 값입니다." | 요청자(로그인된 ADMIN) 본인 비밀번호 재확인 |
| clubRoomNumber | String | O | `@NotBlank`; `@ValidClubRoomNumber` "유효하지 않은 동아리방입니다." | 아래 허용 목록 참조 |

**`clubRoomNumber` 허용 값** (`ClubRoomNumberValidator`)
- 형식 정규식: `^(B\d{3}|\d{3})$`
- 지하: `B101 B102 B103 B104 B105 B106 B107 B108 B110 B111 B112 B113 B114 B115 B116 B117 B118 B119 B120 B121 B122 B123` (**B109 제외**)
- 1층: `102 103 104 105 106 107 108 109 110 112` (**111 제외**)
- 2층: `203 205 206 207 208 209 210` (**204 제외**)

```json
{
  "leaderAccount": "leader001",
  "leaderPw": "Leader123!",
  "leaderPwConfirm": "Leader123!",
  "clubName": "코드동",
  "department": "학술",
  "adminPw": "Test1234!@",
  "clubRoomNumber": "B101"
}
```

**응답** — `ApiResponse<String>`, HTTP 200. **`data` 필드 없음**
```json
{ "message": "동아리 생성 성공" }
```

**비즈니스 규칙 (검증 순서 그대로)**
1. SecurityContext에서 인증된 ADMIN(`CustomAdminDetails.admin()`) 획득
2. `leaderPw != leaderPwConfirm` → CLDR-202
3. `leaderAccount` 중복 확인 → 존재 시 CLDR-203
4. `clubName` 중복 확인 → 존재 시 CLUB-203
5. `adminPw`를 로그인 ADMIN의 해시와 BCrypt 비교 → 불일치 시 ADM-202
6. Club 생성 — `leaderName=""`, `leaderHp=""`, `clubInsta=""`, `clubRoomNumber`=요청값
7. Leader 생성 — 비밀번호 BCrypt 해싱, `leaderUUID` 랜덤 생성, `role=LEADER`
8. 기본 데이터 생성: ClubMainPhoto 1건(이름/Key 빈 문자열), ClubIntro 1건(`clubIntro=""`, `googleFormUrl=""`, `recruitmentStatus=CLOSE`), ClubIntroPhoto **5건**(order 1~5, 이름/Key 빈 문자열)

**부수효과**
- DB: CLUB / LEADER / CLUB_MAIN_PHOTO / CLUB_INTRO / CLUB_INTRO_PHOTO(5) 삽입
- S3·메일·FCM 없음
- **문자열 필드는 `SanitizationBinder`(@ControllerAdvice)의 대상이나, 해당 바인더는 `WebDataBinder` 기반이므로 `@RequestBody` JSON 역직렬화에는 적용되지 않음** (쿼리/폼 파라미터에만 적용)

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | NotBlank/Size/Pattern/ValidClubRoomNumber 위반 |
| INVALID_REQUEST_BODY | 400 | 지원하지 않는 학부 값입니다. / 학부 값은 필수입니다. | `department`에 정의되지 않은 값 |
| CLDR-202 | 400 | 동아리 회장 비밀번호가 일치하지 않습니다 | leaderPw ≠ leaderPwConfirm |
| CLDR-203 | 422 | 이미 존재하는 동아리 회장 계정입니다. | leaderAccount 중복 |
| CLUB-203 | 409 | 이미 존재하는 동아리 이름입니다. | clubName 중복 |
| ADM-202 | 400 | 관리자 비밀번호가 일치하지 않습니다. | adminPw 불일치 |
| DATA_INTEGRITY_VIOLATION | 409 | 데이터 무결성 위반 오류가 발생했습니다. | DB 유니크 제약 위반(동시 요청 등) |

---

#### 5. 동아리 삭제
`DELETE /admin/clubs/{clubUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | application/json (**DELETE이지만 요청 바디 필수**) |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | clubUUID | UUID | O | 삭제할 동아리 UUID |

**요청 바디** — `AdminPwRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| adminPw | String | O | `@NotBlank` "운영자 비밀번호는 필수 입력 값입니다." | 로그인된 ADMIN 본인 비밀번호 |

```json
{ "adminPw": "Test1234!@" }
```

**응답** — `ApiResponse<Long>`, HTTP 200. 컨트롤러가 `new ApiResponse<>("동아리 삭제 성공")` 단일 인자 생성자를 사용하므로 **`data` 없음**
```json
{ "message": "동아리 삭제 성공" }
```

**비즈니스 규칙**
1. 인증된 ADMIN 획득
2. `adminPw` BCrypt 비교 → 불일치 시 ADM-202 (**동아리 존재 확인보다 먼저**)
3. `clubUUID` → clubId 조회, 없으면 CLUB-201
4. `deleteClubAndDependencies(clubId)` 순서대로 삭제:
   ClubMemberAccountStatus → ClubHashtag → ClubCategoryMapping → ClubMembers → Aplict → ClubIntroPhoto → ClubMainPhoto → ClubIntro → Leader → (S3 Key 조회) → S3 일괄 삭제 → Club
5. 예외 발생 시 `BaseException(SERVER_ERROR)`로 감싸 500 반환

**부수효과**
- S3: 동아리 소개 사진 + 메인 사진 객체 일괄 삭제(`deleteObjects`)
- **주의(코드상 사실)**: S3 Key 수집 쿼리가 해당 엔티티 DELETE **이후**에 실행되므로 실제로는 빈 목록이 되어 S3 객체가 남을 수 있음

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | adminPw 공백 |
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | clubUUID 형식 오류 |
| ADM-202 | 400 | 관리자 비밀번호가 일치하지 않습니다. | adminPw 불일치 |
| CLUB-201 | 404 | 존재하지않는 동아리 입니다. | 동아리 미존재 |
| FILE-307 | 400 | 파일 삭제에 실패했습니다. | S3 삭제 실패 |
| COM-501 | 500 | 서버 오류입니다. 관리자에게 문의해주세요. | 삭제 과정 중 그 외 예외 |

---

#### 6. 동아리 회장 아이디 중복 확인
`GET /admin/clubs/leader/check`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 쿼리 | leaderAccount | String | O | 확인할 회장 아이디. **누락 시 400** (`@RequestParam` 기본 required=true) |

```
GET /admin/clubs/leader/check?leaderAccount=leader001
```

**응답** — `ApiResponse<String>`, HTTP 200. **`data` 없음**
```json
{ "message": "사용 가능한 동아리 회장 아이디입니다." }
```

**비즈니스 규칙**
- `leaderRepository.existsByLeaderAccount(leaderAccount)` 가 true면 예외
- 이 엔드포인트 자체에는 길이/패턴 검증이 없음 (동아리 생성 시점에만 적용)
- 사용 가능 시에도 예약(선점)되지 않으므로 생성 요청에서 재검증됨

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLDR-203 | 422 | 이미 존재하는 동아리 회장 계정입니다. | 이미 사용 중인 아이디 |
| NO_CATCH_ERROR | 500 | Required request parameter ... | `leaderAccount` 파라미터 누락 (전용 핸들러 없음) |

---

#### 7. 동아리 이름 중복 확인
`GET /admin/clubs/name/check`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 쿼리 | clubName | String | O | 확인할 동아리명 |

```
GET /admin/clubs/name/check?clubName=코드동
```

**응답** — `ApiResponse<String>`, HTTP 200. **`data` 없음**
```json
{ "message": "사용 가능한 동아리 이름입니다." }
```

**비즈니스 규칙**
- `clubRepository.existsByClubName(clubName)` true면 예외
- 쿼리 파라미터는 `SanitizationBinder`의 `WebDataBinder` 대상이므로 HTML 태그가 Jsoup Safelist로 정제된 뒤 비교됨

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| CLUB-203 | 409 | 이미 존재하는 동아리 이름입니다. | 이미 사용 중인 동아리명 |
| NO_CATCH_ERROR | 500 | Required request parameter ... | `clubName` 파라미터 누락 |

---

#### 8. 동아리 카테고리 목록 조회
`GET /admin/clubs/category`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN 또는 ROLE_LEADER (경로가 `/admin/clubs/{clubUUID}` 패턴에 매칭됨) |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |

**응답** — `ApiResponse<List<ClubCategoryResponse>>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data[].clubCategoryUUID | UUID | 카테고리 UUID (삭제 시 사용) |
| data[].clubCategoryName | String | 카테고리명 (**소문자로 저장됨**) |

```json
{
  "message": "카테고리 리스트 조회 성공",
  "data": [
    { "clubCategoryUUID": "bbbbbbbb-1111-2222-3333-444444444444", "clubCategoryName": "학술" },
    { "clubCategoryUUID": "cccccccc-1111-2222-3333-444444444444", "clubCategoryName": "sport" }
  ]
}
```

**비즈니스 규칙 / 부수효과**
- `clubCategoryRepository.findAll()` 전체 반환, 페이징·정렬 없음, 읽기 전용

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| TOKEN_EXPIRED / INVALID_TOKEN | 401 | (EntryPoint 포맷) | 인증 실패 |
| — | 403 | Spring Security 기본 | ROLE_USER로 호출 |

---

#### 9. 동아리 카테고리 추가
`POST /admin/clubs/category`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | application/json |

**요청 바디** — `AdminClubCategoryCreationRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| clubCategoryName | String | O | `@NotBlank` "카테고리 이름은 필수 입력 값입니다."; `@Size(min=1,max=20)` "카테고리는 최대 20자까지 입력 가능합니다." | 카테고리명. 서버에서 `toLowerCase()` 처리 후 저장 |

```json
{ "clubCategoryName": "학술" }
```

**응답** — `ApiResponse<ClubCategoryResponse>`, HTTP 200

```json
{
  "message": "카테고리 추가 성공",
  "data": { "clubCategoryUUID": "bbbbbbbb-1111-2222-3333-444444444444", "clubCategoryName": "학술" }
}
```

**비즈니스 규칙**
1. 입력값을 `toLowerCase()`로 정규화
2. 정규화된 이름으로 `existsByClubCategoryName` → 존재 시 CTG-203
3. 저장 후 UUID/이름 반환 (**저장·응답 모두 소문자 값**)

**부수효과** — DB CLUB_CATEGORY 삽입만. 외부 연동 없음

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | 공백 또는 21자 이상 |
| CTG-203 | 409 | 이미 존재하는 카테고리입니다. | 소문자 정규화 후 중복 |

---

#### 10. 동아리 카테고리 삭제
`DELETE /admin/clubs/category/{clubCategoryUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | clubCategoryUUID | UUID | O | 삭제할 카테고리 UUID |

**응답** — `ApiResponse<ClubCategoryResponse>`, HTTP 200 (삭제된 카테고리 정보 반환)

```json
{
  "message": "카테고리 삭제 성공",
  "data": { "clubCategoryUUID": "bbbbbbbb-1111-2222-3333-444444444444", "clubCategoryName": "학술" }
}
```

**비즈니스 규칙**
1. UUID로 카테고리 조회 → 없으면 CTG-201
2. `ClubCategoryMapping`에서 해당 카테고리 매핑 전부 삭제(동아리↔카테고리 연결 해제)
3. 카테고리 자체 삭제
4. 2~3 과정 예외는 `BaseException(SERVER_ERROR)`로 래핑

**부수효과** — DB CLUB_CATEGORY_MAPPING 다건 삭제 + CLUB_CATEGORY 1건 삭제. S3/메일/FCM 없음

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | UUID 형식 오류 |
| CTG-201 | 404 | 존재하지 않는 카테고리입니다. | 카테고리 미존재 |
| COM-501 | 500 | 서버 오류입니다. 관리자에게 문의해주세요. | 삭제 중 예외 |

---

#### 11. 층별 도면 사진 업로드
`PUT /admin/floor/photo/{floor}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | multipart/form-data |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | floor | Enum | O | `FloorPhotoEnum`: `B1`(지하 1층), `F1`(1층), `F2`(2층) |
| 파트 | photo | File | O | 도면 이미지. jpg/jpeg/png, 개별 10MB 이하 |

```
PUT /admin/floor/photo/B1
Content-Type: multipart/form-data; boundary=----X

------X
Content-Disposition: form-data; name="photo"; filename="b1_floor.png"
Content-Type: image/png

<binary>
------X--
```

**응답** — `ApiResponse<AdminFloorPhotoCreationResponse>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data.floor | String | `B1` / `F1` / `F2` |
| data.presignedUrl | String | **PUT용** presigned URL (1시간). 클라이언트가 이 URL로 실제 파일을 S3에 직접 업로드해야 함 |

```json
{
  "message": "해당 층 사진 업로드 성공",
  "data": {
    "floor": "B1",
    "presignedUrl": "https://bucket.s3.ap-northeast-2.amazonaws.com/floorPhoto/9f1c....png?X-Amz-Algorithm=...&X-Amz-Signature=..."
  }
}
```

**비즈니스 규칙 (순서)**
1. `photo == null || photo.isEmpty()` → PHOTO-504
2. 해당 층에 기존 사진이 있으면 **S3 객체 삭제 + DB 레코드 삭제** (층별 1장 정책)
3. 확장자 검증(jpg/jpeg/png) → 파일 시그니처 검증
4. S3 Key `floorPhoto/{랜덤UUID}.{ext}` 생성, PUT presigned URL 발급
5. `FloorPhoto` 저장 — `floorPhotoName` = 원본 파일명, `floorPhotoS3key` = 생성 Key
6. **서버는 S3에 바이너리를 직접 올리지 않음.** 클라이언트가 presignedUrl로 PUT하지 않으면 DB 레코드만 존재

**부수효과** — 기존 S3 객체 삭제, DB FLOOR_PHOTO 갱신, presigned PUT URL 발급

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | `{floor}`가 B1/F1/F2가 아님 (Enum 변환 실패 → MethodArgumentTypeMismatch 핸들러, 메시지가 UUID 문구인 점 주의) |
| PHOTO-504 | 400 | 사진 파일이 비어있습니다. | photo 파트 비어있음 |
| FILE-309 | 400 | 파일 이름이 유효하지 않습니다. | 원본 파일명 null |
| FILE-310 | 400 | 파일 확장자가 없습니다. | 파일명에 `.` 없음 |
| FILE-311 | 400 | 지원하지 않는 파일 확장자입니다. | jpg/jpeg/png 외 또는 시그니처 불일치 |
| FILE-312 | 400 | 파일 유효성 검사 실패 | 파일 스트림 읽기 실패 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 중 S3/SDK 오류 |
| FILE-307 | 400 | 파일 삭제에 실패했습니다. | 기존 사진 S3 삭제 실패 |
| MAX_UPLOAD_SIZE_EXCEEDED | 400 | 업로드 가능한 최대 파일 크기를 초과했습니다. (개별 파일 10MB, 총 파일 크기 50MB) | 용량 초과 |

---

#### 12. 층별 도면 사진 조회
`GET /admin/floor/photo/{floor}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN (GET `/admin/**` 규칙, `/admin/clubs*` 예외에 해당하지 않음) |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | floor | Enum | O | `B1` / `F1` / `F2` |

**응답** — `ApiResponse<AdminFloorPhotoCreationResponse>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data.floor | String | 요청한 층 |
| data.presignedUrl | String | **GET용** presigned URL (1시간). 이미지 표시에 바로 사용 |

```json
{
  "message": "해당 층 사진 조회 성공",
  "data": { "floor": "B1", "presignedUrl": "https://bucket.s3.ap-northeast-2.amazonaws.com/floorPhoto/9f1c....png?X-Amz-Signature=..." }
}
```

**비즈니스 규칙 / 부수효과**
- 해당 층 FloorPhoto 조회 → 없으면 PHOTO-505
- 요청 시마다 새 GET presigned URL 생성(1시간 유효). 읽기 전용 트랜잭션

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | `{floor}` Enum 변환 실패 |
| PHOTO-505 | 404 | 해당 사진이 존재하지 않습니다. | 해당 층 사진 미등록 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 오류 |

---

#### 13. 층별 도면 사진 삭제
`DELETE /admin/floor/photo/{floor}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | floor | Enum | O | `B1` / `F1` / `F2` |

**응답** — `ApiResponse<String>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data | String | `"Floor: " + floor.name()` 형식 문자열 |

```json
{ "message": "해당 층 사진 삭제 성공", "data": "Floor: B1" }
```

**비즈니스 규칙 / 부수효과**
1. 해당 층 FloorPhoto 조회 → 없으면 PHOTO-505
2. S3 객체 삭제 → DB 레코드 삭제

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | `{floor}` Enum 변환 실패 |
| PHOTO-505 | 404 | 해당 사진이 존재하지 않습니다. | 해당 층 사진 미등록 |
| FILE-307 | 400 | 파일 삭제에 실패했습니다. | S3 삭제 실패 |

---

#### 14. 공지사항 목록 조회
`GET /notices`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN 또는 ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 쿼리 | page | int | X | 페이지 번호(0-base), 기본 `0` |
| 쿼리 | size | int | X | 페이지 크기, 기본 `10` |

```
GET /notices?page=0&size=10
```

**응답** — `ApiResponse<AdminNoticePageListResponse>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data.content[].noticeUUID | UUID | 공지 UUID |
| data.content[].noticeTitle | String | 제목 |
| data.content[].adminName | String | 작성 운영자 이름 |
| data.content[].noticeCreatedAt | LocalDateTime | 작성 일시 (`2026-03-02T13:45:12`) |
| data.totalPages | int | 전체 페이지 수 |
| data.totalElements | long | 전체 공지 수 |
| data.currentPage | int | 현재 페이지 번호 |

```json
{
  "message": "공지사항 리스트 조회 성공",
  "data": {
    "content": [
      { "noticeUUID": "dddddddd-1111-2222-3333-444444444444", "noticeTitle": "2026학년도 1학기 동아리 등록 안내", "adminName": "운영자", "noticeCreatedAt": "2026-03-02T13:45:12" }
    ],
    "totalPages": 2,
    "totalElements": 14,
    "currentPage": 0
  }
}
```

**비즈니스 규칙 / 부수효과**
- `noticeCreatedAt DESC` 정렬 적용 (`findAll(pageable)`)
- 목록에는 사진 URL을 포함하지 않음 → presigned URL 생성 비용 없음
- 읽기 전용

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| TOKEN_EXPIRED / INVALID_TOKEN | 401 | (EntryPoint 포맷) | 인증 실패 |
| — | 403 | Spring Security 기본 | ROLE_USER로 호출 |
| NO_CATCH_ERROR | 500 | — | `page<0` 또는 `size<1` |

---

#### 15. 공지사항 상세 조회
`GET /notices/{noticeUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN 또는 ROLE_LEADER |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | noticeUUID | UUID | O | 공지 UUID |

**응답** — `ApiResponse<NoticeDetailResponse>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| noticeUUID | UUID | 공지 UUID |
| noticeTitle | String | 제목 |
| noticeContent | String | 내용 |
| noticePhotos | List&lt;String&gt; | 사진 GET presigned URL 목록, `order` 오름차순 |
| noticeCreatedAt | LocalDateTime | 작성 일시 |
| adminName | String | 작성 운영자 이름 |

```json
{
  "message": "공지사항 조회 성공",
  "data": {
    "noticeUUID": "dddddddd-1111-2222-3333-444444444444",
    "noticeTitle": "2026학년도 1학기 동아리 등록 안내",
    "noticeContent": "3월 2일부터 등록을 시작합니다.",
    "noticePhotos": [
      "https://bucket.s3.ap-northeast-2.amazonaws.com/noticePhoto/aaaa.png?X-Amz-Signature=...",
      "https://bucket.s3.ap-northeast-2.amazonaws.com/noticePhoto/bbbb.jpg?X-Amz-Signature=..."
    ],
    "noticeCreatedAt": "2026-03-02T13:45:12",
    "adminName": "운영자"
  }
}
```

**비즈니스 규칙 / 부수효과**
1. UUID로 공지 조회 → 없으면 NOT-201
2. `NoticePhoto`를 `order` 오름차순 정렬 후 각각 GET presigned URL(1시간) 생성

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | UUID 형식 오류 |
| NOT-201 | 404 | 공지사항이 존재하지 않습니다. | 공지 미존재 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 오류 |

---

#### 16. 공지사항 작성
`POST /notices`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN |
| 레이트리밋 | 없음 |
| Content-Type | multipart/form-data |

**요청 파트**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 파트 | request | JSON (`application/json`) | 형식상 `required=false`, **실질 필수** | `AdminNoticeCreationRequest`. 누락 시 서비스에서 NPE → 500 |
| 파트 | photos | File[] | X | 최대 5장, jpg/jpeg/png |

**request 파트 필드** — `AdminNoticeCreationRequest`

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| noticeTitle | String | O | `@NotBlank` "공지사항 제목은 비워둘 수 없습니다."; `@Size(min=1,max=200)` "공지사항 제목은 최대 200자까지 입력 가능합니다." | 제목 |
| noticeContent | String | O | `@NotBlank` "공지사항 내용은 비워둘 수 없습니다."; `@Size(min=1,max=3000)` "공지사항 내용은 최대 3000자까지 입력 가능합니다." | 내용 |
| photoOrders | List&lt;Integer&gt; | photos 있을 때 필수 | `@Size(max=5)` "사진은 최대 5장까지 업로드 가능합니다."; 각 원소 `@Min(1) @Max(5)` | 사진 순서 배열. **photos 배열과 인덱스가 1:1 대응** |

```
POST /notices
Content-Type: multipart/form-data; boundary=----X

------X
Content-Disposition: form-data; name="request"
Content-Type: application/json

{"noticeTitle":"2026학년도 1학기 동아리 등록 안내","noticeContent":"3월 2일부터 등록을 시작합니다.","photoOrders":[1,2]}
------X
Content-Disposition: form-data; name="photos"; filename="poster1.png"
Content-Type: image/png

<binary>
------X
Content-Disposition: form-data; name="photos"; filename="poster2.jpg"
Content-Type: image/jpeg

<binary>
------X--
```

**응답** — `ApiResponse<List<String>>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data | List&lt;String&gt; | 업로드된 사진 수만큼의 **PUT용 presigned URL 목록**. 사진 미첨부 시 `[]` |

```json
{
  "message": "공지사항 생성 성공",
  "data": [
    "https://bucket.s3.ap-northeast-2.amazonaws.com/noticePhoto/aaaa.png?X-Amz-Signature=...",
    "https://bucket.s3.ap-northeast-2.amazonaws.com/noticePhoto/bbbb.jpg?X-Amz-Signature=..."
  ]
}
```

**비즈니스 규칙 (실행 순서 그대로)**
1. SecurityContext에서 인증된 ADMIN 획득 → 작성자로 기록
2. **Notice 먼저 저장** (`noticeCreatedAt = LocalDateTime.now()`)
3. 사진 검증 — photos가 비어있지 않을 때만 수행
   - `photoOrders == null` 또는 `photos.size() != photoOrders.size()` → FILE-304
   - `photos.size() > 5` → NOT-202
   - **photos가 없으면 photoOrders 값은 검증/사용되지 않음**
4. 사진 처리 — i번째 photos ↔ i번째 photoOrders 매칭
   - 개별 파일이 null/empty면 **경고 로그만 남기고 건너뜀**(예외 아님) → 해당 순서는 저장되지 않음
   - 확장자·시그니처 검증 후 S3 Key `noticePhoto/{UUID}.{ext}` 생성, PUT presigned URL 발급
   - `NoticePhoto` 일괄 저장 (`noticePhotoName` = 원본 파일명)
5. 검증 실패 시 트랜잭션 롤백되어 2번에서 저장한 Notice도 취소됨

**부수효과**
- DB: NOTICE 1건 + NOTICE_PHOTO N건 삽입
- S3: 객체 직접 업로드 없음. **클라이언트가 반환된 PUT presigned URL(1시간)로 직접 업로드해야 실제 이미지가 생성됨**
- 메일/FCM 발송 없음

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | 제목/내용 공백·길이 초과, photoOrders 원소가 1~5 밖 또는 6개 이상 |
| FILE-304 | 400 | 사진의 개수와 순서 정보의 개수가 일치하지 않습니다. | photos는 있는데 photoOrders 누락 또는 개수 불일치 |
| NOT-202 | 413 | 최대 5개의 사진이 업로드 가능합니다. | photos 6장 이상 |
| FILE-309 / FILE-310 / FILE-311 / FILE-312 | 400 | (파일 검증 메시지) | 파일명/확장자/시그니처 오류 |
| FILE-306 | 400 | 파일 업로드에 실패했습니다. | presigned URL 생성 오류 |
| MAX_UPLOAD_SIZE_EXCEEDED | 400 | 업로드 가능한 최대 파일 크기를 초과했습니다. (개별 파일 10MB, 총 파일 크기 50MB) | 용량 초과 |
| NULL_POINTER_EXCEPTION | 500 | NullPointerException 발생: ... | `request` 파트 자체를 보내지 않은 경우 |

---

#### 17. 공지사항 수정
`PUT /notices/{noticeUUID}`

| 항목 | 값 |
|---|---|
| 인증 | **authenticated()** — SecurityConfig에 PUT `/notices/**` 전용 규칙이 없어 `anyRequest().authenticated()`가 적용됨. 즉 ROLE_USER/LEADER도 호출 가능 (코드상 사실, 의도치 않은 설정으로 보임) |
| 레이트리밋 | 없음 |
| Content-Type | multipart/form-data |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | noticeUUID | UUID | O | 수정할 공지 UUID |
| 파트 | request | JSON | 형식상 `required=false`, **실질 필수** | `AdminNoticeUpdateRequest` |
| 파트 | photos | File[] | X | 새로 등록할 사진 (최대 5장) |

**request 파트 필드** — `AdminNoticeUpdateRequest` (`AdminNoticeCreationRequest`와 필드/검증 동일)

| 필드 | 타입 | 필수 | 검증 규칙 | 설명 |
|---|---|---|---|---|
| noticeTitle | String | O | `@NotBlank`; `@Size(min=1,max=200)` | 제목 |
| noticeContent | String | O | `@NotBlank`; `@Size(min=1,max=3000)` | 내용 |
| photoOrders | List&lt;Integer&gt; | photos 있을 때 필수 | `@Size(max=5)`; 원소 `@Min(1) @Max(5)` | 사진 순서 배열 |

```
PUT /notices/dddddddd-1111-2222-3333-444444444444
Content-Type: multipart/form-data; boundary=----X

------X
Content-Disposition: form-data; name="request"
Content-Type: application/json

{"noticeTitle":"[수정] 동아리 등록 안내","noticeContent":"등록 기간이 연장되었습니다.","photoOrders":[1]}
------X
Content-Disposition: form-data; name="photos"; filename="poster_new.png"
Content-Type: image/png

<binary>
------X--
```

**응답** — `ApiResponse<List<String>>`, HTTP 200

```json
{
  "message": "공지사항 수정 성공",
  "data": ["https://bucket.s3.ap-northeast-2.amazonaws.com/noticePhoto/cccc.png?X-Amz-Signature=..."]
}
```

**비즈니스 규칙 (실행 순서 그대로)**
1. UUID로 공지 조회 → 없으면 NOT-201
2. 제목·내용 갱신 (`updateTitle`, `updateContent`)
3. **기존 사진 전량 삭제** — S3 일괄 삭제 + `NOTICE_PHOTO` 배치 삭제
   → **photos를 보내지 않으면 기존 사진이 모두 사라짐**(전체 교체 방식, 부분 수정 불가)
4. 사진 검증 (작성과 동일: 개수 불일치 FILE-304, 5장 초과 NOT-202)
5. 새 사진 업로드 처리 → PUT presigned URL 목록 반환
6. 작성자(`admin`)는 변경되지 않음. `noticeCreatedAt`도 변경되지 않음(`updatable=false`)

**부수효과**
- S3: 기존 공지 사진 객체 일괄 삭제
- DB: NOTICE 갱신, NOTICE_PHOTO 전량 삭제 후 재삽입
- 클라이언트가 반환된 PUT presigned URL로 직접 업로드 필요

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | noticeUUID 형식 오류 |
| INVALID_ARGUMENT | 400 | 입력 값 검증에 실패했습니다. | 제목/내용/photoOrders 검증 실패 |
| NOT-201 | 404 | 공지사항이 존재하지 않습니다. | 공지 미존재 |
| FILE-304 | 400 | 사진의 개수와 순서 정보의 개수가 일치하지 않습니다. | photos/photoOrders 개수 불일치 |
| NOT-202 | 413 | 최대 5개의 사진이 업로드 가능합니다. | photos 6장 이상 |
| FILE-307 | 400 | 파일 삭제에 실패했습니다. | 기존 사진 S3 삭제 실패 |
| FILE-309~312 / FILE-306 | 400 | (파일 검증·업로드 메시지) | 파일 검증/URL 생성 오류 |
| MAX_UPLOAD_SIZE_EXCEEDED | 400 | 업로드 가능한 최대 파일 크기를 초과했습니다. (개별 파일 10MB, 총 파일 크기 50MB) | 용량 초과 |
| NULL_POINTER_EXCEPTION | 500 | NullPointerException 발생: ... | `request` 파트 미전송 |

---

#### 18. 공지사항 삭제
`DELETE /notices/{noticeUUID}`

| 항목 | 값 |
|---|---|
| 인증 | ROLE_ADMIN (`DELETE /notices/**`) |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청**

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| 헤더 | Authorization | String | O | `Bearer {accessToken}` |
| 경로 | noticeUUID | UUID | O | 삭제할 공지 UUID |

**응답** — `ApiResponse<UUID>`, HTTP 200

| 필드 | 타입 | 설명 |
|---|---|---|
| data | UUID | 삭제된 공지의 UUID |

```json
{ "message": "공지사항 삭제 성공", "data": "dddddddd-1111-2222-3333-444444444444" }
```

**비즈니스 규칙 / 부수효과**
1. UUID로 공지 조회 → 없으면 NOT-201
2. 연결된 `NoticePhoto`의 S3 객체 일괄 삭제 + DB 배치 삭제
3. Notice 삭제

**에러**
| 코드 | HTTP | 메시지 | 발생 조건 |
|---|---|---|---|
| INVALID_UUID_FORMAT | 400 | 유효하지 않은 UUID 형식입니다. 올바른 UUID를 입력하세요. | UUID 형식 오류 |
| NOT-201 | 404 | 공지사항이 존재하지 않습니다. | 공지 미존재 |
| FILE-307 | 400 | 파일 삭제에 실패했습니다. | S3 삭제 실패 |

---

#### 19. 헬스 체크
`GET /health-check`

| 항목 | 값 |
|---|---|
| 인증 | permitAll (`permit-all-paths`에 `/health-check` 포함) |
| 레이트리밋 | 없음 |
| Content-Type | — |

**요청** — 파라미터·헤더·바디 없음

**응답** — `ResponseEntity<String>`, HTTP 200. **`ApiResponse` 래퍼를 사용하지 않는 유일한 엔드포인트**

```
OK
```
(Content-Type은 Spring 기본 협상 결과, 일반적으로 `text/plain;charset=UTF-8`)

**비즈니스 규칙 / 부수효과** — 없음. DB/Redis/S3 접근 없이 상수 문자열만 반환하므로 의존 서비스 장애를 감지하지 못함

**에러** — 없음

---

### 부록. 관리자 API 전체 요약표

| # | 기능 | 메서드 | 경로 | 인증 | 레이트리밋 |
|---|---|---|---|---|---|
| 1 | 운영팀 로그인 | POST | `/admin/login` | permitAll | WEB_LOGIN 5회/5분 |
| 2 | 동아리 목록 | GET | `/admin/clubs` | ADMIN, LEADER | 없음 |
| 3 | 동아리 상세 | GET | `/admin/clubs/{clubUUID}` | ADMIN, LEADER | 없음 |
| 4 | 동아리 생성 | POST | `/admin/clubs` | ADMIN | 없음 |
| 5 | 동아리 삭제 | DELETE | `/admin/clubs/{clubUUID}` | ADMIN | 없음 |
| 6 | 회장 아이디 중복확인 | GET | `/admin/clubs/leader/check` | ADMIN | 없음 |
| 7 | 동아리명 중복확인 | GET | `/admin/clubs/name/check` | ADMIN | 없음 |
| 8 | 카테고리 목록 | GET | `/admin/clubs/category` | ADMIN, LEADER | 없음 |
| 9 | 카테고리 추가 | POST | `/admin/clubs/category` | ADMIN | 없음 |
| 10 | 카테고리 삭제 | DELETE | `/admin/clubs/category/{clubCategoryUUID}` | ADMIN | 없음 |
| 11 | 층별 도면 업로드 | PUT | `/admin/floor/photo/{floor}` | ADMIN | 없음 |
| 12 | 층별 도면 조회 | GET | `/admin/floor/photo/{floor}` | ADMIN | 없음 |
| 13 | 층별 도면 삭제 | DELETE | `/admin/floor/photo/{floor}` | ADMIN | 없음 |
| 14 | 공지 목록 | GET | `/notices` | ADMIN, LEADER | 없음 |
| 15 | 공지 상세 | GET | `/notices/{noticeUUID}` | ADMIN, LEADER | 없음 |
| 16 | 공지 작성 | POST | `/notices` | ADMIN | 없음 |
| 17 | 공지 수정 | PUT | `/notices/{noticeUUID}` | authenticated (규칙 누락) | 없음 |
| 18 | 공지 삭제 | DELETE | `/notices/{noticeUUID}` | ADMIN | 없음 |
| — | 헬스 체크 | GET | `/health-check` | permitAll | 없음 |

> 담당 범위 컨트롤러 5개(`AdminLoginController`, `AdminClubController`, `AdminClubCategoryController`, `AdminFloorPhotoController`, `AdminNoticeController`)와 `HealthCheckController`에는 **주석 처리되어 비활성화된 엔드포인트가 없습니다.**

#### 상위 종합 단계 참고 사항 (코드에서 확인된 이슈)
- `GET /admin/clubs`의 `Sort`가 JPQL에 반영되지 않아 정렬이 보장되지 않음 (`ClubRepositoryCustomImpl:27-33`)
- `PUT /notices/{noticeUUID}`에 ADMIN 전용 인가 규칙 누락 (`SecurityConfig:66-67`에 POST/DELETE만 존재)
- `deleteClubAndDependencies`에서 S3 Key 수집이 엔티티 삭제 이후에 실행됨 (`ClubRepositoryCustomImpl:71-99`)
- 공지 작성/수정의 `request` 파트가 `required=false`라 미전송 시 400이 아닌 500(NPE) 반환
- `RateLimitAction.LEADER_CHANGE_PW`는 정의만 되어 있고 사용처 없음

**참조 파일 (모두 절대 경로)**
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/api/AdminLoginController.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/api/AdminClubController.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/api/AdminClubCategoryController.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/api/AdminFloorPhotoController.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/notice/api/AdminNoticeController.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/service/AdminClubService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/service/AdminClubCategoryService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/service/AdminFloorPhotoService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/admin/service/AdminLoginService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/admin/notice/service/AdminNoticeService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/exception/ExceptionType.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/exception/GlobalExceptionHandler.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/security/config/SecurityConfig.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/security/jwt/JwtProvider.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/bucket4j/RateLimitAction.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/s3File/Service/S3FileUploadService.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/validation/ClubRoomNumberValidator.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/java/com/USWCicrcleLink/server/global/docs/OpenApiConfig.java`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/resources/yaml/application-security.yml`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/resources/yaml/application-prod.yml`
- `/home/jovyan/work/USW-Circle-Link-Server/src/main/resources/yaml/application-file.yml`

---

## 9. 부록: 주요 흐름 시퀀스

### 9.1 로그인 ~ 토큰 재발급 ~ 로그아웃

```
◄── 200 …
   │
   │ ③ accessToken 만료 → 401 { status:401, errorCode:"TOKEN_EXPIRED", message:"토큰이 만료되었습니다." }
   │
   │ ④ POST /integration/refresh-token      (바디·헤더 없음, refreshToken 쿠키만 전송)
   ├──────────────────────────────────────────────────►│
   │                                                   │ 쿠키 없음/Redis 미존재 → logout() 후 401
   │                                                   │ 성공: 기존 refreshToken 삭제(rotation) → 신규 access+refresh 발급
   │ ◄── 200  Header: Authorization: Bearer {new access}
   │          Set-Cookie: refreshToken={new}; …
   │          Body:  { accessToken, refreshToken }
   │
   │ ◄── (실패 시) 401 { message: "리프레시 토큰이 유효하지 않습니다. 로그아웃됐습니다." }
   │              → 재로그인 화면으로 이동
   │
   │ ⑤ POST /integration/logout             (refreshToken 쿠키 전송, 액세스 토큰 없어도 됨)
   ├──────────────────────────────────────────────────►│
   │                                                   │ Redis refreshToken 삭제, USER면 Profile.fcmToken=null
   │                                                   │ SecurityContext clear, 쿠키 만료
   │ ◄── 200 { message: "로그아웃 성공" }  Set-Cookie: refreshToken=; Max-Age=0
   │       → 클라이언트가 저장된 accessToken을 직접 폐기해야 함(서버 블랙리스트 없음)
```
```
**핵심 포인트**
- ADMIN(`POST /admin/login`) / LEADER(`POST /club-leader/login`)도 ④⑤ 단계는 **동일한 `/integration/**` 엔드포인트**를 사용합니다.
- **단일 세션**: ①에서 기존 refreshToken을 삭제하므로, 다른 기기에서 로그인하면 이전 기기의 재발급이 실패합니다.
- **rotation**: ④가 성공하면 이전 refreshToken은 즉시 무효화됩니다. 동시 요청 시 뒤늦은 요청은 401을 받습니다.
- 쿠키 기반이므로 크로스 오리진에서는 `credentials: 'include'` / `withCredentials: true`가 필수이며, prod는 `SameSite=None; Secure`라 HTTPS에서만 동작합니다.
- 비밀번호 변경(U-13, U-12)은 토큰을 무효화하지 **않지만**, 회장 비밀번호 변경(L-05)은 refreshToken을 삭제하고 쿠키를 만료시킵니다.

### 9.2 동아리 지원 ~ 합격/불합격 통보

```
[사용자 앱]                    [서버]                     [회장 웹]                [FCM/스케줄러]
   │
   │ ① GET /clubs/open  또는  /clubs/open/filter?clubCategoryUUIDs=…
   ├──────────────────►│  (permitAll, 모집중 동아리 목록)
   │ ◄── 200 [ … ]
   │
   │ ② GET /clubs/intro/{clubUUID}      (상세 — 소개글·모집글·구글폼 URL 노출)
   ├──────────────────►│
   │ ◄── 200 { … recruitmentStatus:"OPEN", googleFormUrl … }
   │
   │ ③ GET /apply/can-apply/{clubUUID}  (ROLE_USER, true/false만 반환)
   ├──────────────────►│
   │ ◄── 200 { data: true }
   │
   │ ④ GET /apply/{clubUUID}            → 구글 폼 URL 수령 → 외부 폼 작성
   │ ◄── 200 { data: "https://forms.gle/…" }
   │
   │ ⑤ POST /apply/{clubUUID}           (바디 없음)
   ├──────────────────►│  APT-205/206/207/208 사유별 거절, CLUB-201
   │                   │  Aplict INSERT: status=WAIT, checked=false, deleteDate=null
   │ ◄── 200 { message: "지원서 제출 성공" }
   │
   │ ⑥ GET /mypages/aplict-clubs        → aplictStatus: "WAIT"
   │
   │                                        │ ⑦ GET /club-leader/{clubUUID}/applicants
   │                                        ├───────────────►│ (checked=false 지원자 목록)
   │                                        │
   │                                        │ ⑧ POST /club-leader/{clubUUID}/applicants/notifications
   │                                        │    [ {aplictUUID, "PASS"}, {aplictUUID, "FAIL"} ]
   │                                        ├───────────────►│
   │                                        │                │ PASS → ClubMembers INSERT
   │                                        │                │ 공통 → status=PASS/FAIL, checked=true,
   │                                        │                │        deleteDate = now + 4일
   │                                        │                ├──── FCM 푸시 ────►  "동아리 지원 결과"
   │                                        │                │                     "{동아리명}에 합격/불합격했습니다."
   │                                        │ ◄── 200 { message: "지원 결과 처리 완료" }
   │
   │ ⑨ (푸시 수신) GET /mypages/aplict-clubs → aplictStatus: "PASS" 또는 "FAIL"
   │    합격이면 GET /mypages/my-clubs 목록에 동아리가 추가됨
   │
   │                                        │ ⑩ (추가합격) GET /club-leader/{clubUUID}/failed-applicants
   │                                        ├───────────────►│ (checked=true AND status=FAIL)
   │                                        │ ⑪ POST /club-leader/{clubUUID}/failed-applicants/notifications
   │                                        │    [ {aplictUUID, "PASS"} ]
   │                                        ├───────────────►│ ClubMembers INSERT + status=PASS
   │                                        │                │ (checked·deleteDate는 그대로 유지)
   │                                        │                ├──── FCM 푸시(합격) ────►
   │
   │                                                          ⑫ 매일 00:00 스케줄러
   │                                                             deleteDate < now → Aplict 물리 삭제
   │                                                             → 통보 4일 후 지원내역/추합 후보 목록에서 소멸
```

**핵심 포인트**
- ③과 ⑤는 검사 항목이 같지만, ③은 사유 없이 `false`만 반환하고 ⑤는 `APT-205~208`로 사유를 구분합니다. ③은 **동아리 존재 여부를 확인하지 않으므로** 잘못된 `clubUUID`에도 `true`가 나올 수 있습니다.
- **추가합격 가능 기간은 사실상 최초 통보 후 4일**입니다(⑫ 스케줄러가 FAIL 지원서를 삭제하면 ⑩ 목록에서 사라짐). ⑪은 `deleteDate`를 연장하지 않습니다.
- 합/불 통보로 `checked=true`가 되면 유니크 제약(`uk_aplict_active`)이 풀려 **같은 동아리에 재지원이 가능**해집니다.
- FCM 토큰이 없거나(로그인 시 미전송, 60일 미갱신으로 스케줄러가 null 처리) FCM 전송이 실패해도 **API는 200**이며, 사용자는 ⑨의 폴링으로만 결과를 알 수 있습니다.
- ⑧은 DB 롤백이 발생해도 **이미 전송된 푸시는 취소되지 않습니다.**

---

---

## 10. 문서 이력 및 범위

- **작성 기준**: 본 저장소의 현재 소스 코드 (`src/main/java/com/USWCicrcleLink/server`, `src/main/resources/yaml`)
- **수록 범위**: 활성 REST 엔드포인트 81개 + 비활성 1개 = **82개**
- **비활성/주의 엔드포인트**
  - ⚠️ `POST /users/existing/register` (U-07) — 서비스 로직 전체 주석 처리, 항상 400 반환
  - ⚠️ `GET /club-leader/v1/members` (L-25) — 매핑 주석 처리, 404(`RESOURCE_NOT_FOUND`)
- **설정 충돌로 의도와 다르게 동작하는 지점** (모두 코드상 사실)
  - `GET /my-notices`, `GET /my-notices/{noticeUUID}/details` → permitAll (USER 권한 규칙 무력화)
  - `PATCH /club-leader/fcmtoken` → ROLE_USER (LEADER 아님)
  - `GET /admin/clubs/category` → ADMIN + LEADER
  - `PUT /notices/{noticeUUID}` → authenticated (ADMIN 전용 규칙 누락)
- **코드상 미확인 항목**
  - `GET /club-leader/{clubUUID}/members/export`에서 엑셀 바이너리 뒤 JSON이 실제로 이어붙는지 (서블릿 컨테이너 동작 의존)
  - `GET /clubs/open/filter`에서 모집중 동아리가 0건일 때 JPQL `IN ()`의 동작 (빈 리스트 방어 로직 없음)
  - `POST /club-leader/{clubUUID}/applicants/notifications`에서 `List` 파라미터에 대한 `@Validated` 요소 캐스케이드 여부