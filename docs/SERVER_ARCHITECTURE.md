# 동구라미 서버 구조도

> 팀 Notion 자료 **"동구라미 자료" (2025-07-29 기준)** 의 서버 구조 기술을 실제 저장소 코드와 대조 검증하여 종합한 문서입니다.
> 저장소에서 확인 불가능한 항목(서버 로컬 설정, 인스턴스 스펙 등)은 **미확인** 또는 **자료 기준**으로 명시했습니다. 코드와 자료가 어긋나는 지점은 [9절](#9-️-notion-자료와-코드의-불일치--주의사항)에 정리했습니다.

---

## 1. 한눈에 보기

```
┌──────────────────────────────────────────────┐        ┌──────────────────────────┐
│                   Clients                    │        │      GitHub Actions      │
│   - Mobile App (USER)  <--(FCM push)         │        │   (PR merged --> main)   │
│   - Web Dashboard (LEADER / ADMIN)           │        └────────────┬─────────────┘
│   origin: https://donggurami.net             │                     │ SSH / SCP :22
│           https://*.donggurami.net           │                     │ (deploy.zip)
└──────┬────────────────────────────┬──────────┘                     │
       │ HTTPS :443 / HTTP :80      │ presigned PUT/GET              │
       │                            └---(HTTPS :443)---> [AWS S3]    │
       ↓                                                             ↓
┌────────────────────────────────────────────────────────────────────────────────┐
│                   Lightsail Web Instance  (<masked-ip1>)                       │
│                                                                                │
│    ┌────────────────┐  upstream: /etc/nginx/conf.d/service-url.inc             │
│    │     Nginx      │  (switch.sh: overwrite + nginx reload)                   │
│    │   :80 / :443   │                                                          │
│    └───────┬────────┘                                                          │
│            │ HTTP 127.0.0.1:{8081|8082}                                        │
│            ↓                                                                   │
│    ┌───────────────────┐        ┌───────────────────┐                          │
│    │ Spring Boot BLUE  │        │ Spring Boot GREEN │   one active at a time   │
│    │ :8081             │        │ :8082             │   (nohup, profile=prod,  │
│    └─────────┬─────────┘        └─────────┬─────────┘    Pinpoint agent 3.0.1) │
│              └────────────┬───────────────┘                                    │
│                           │ TCP 127.0.0.1:6379                                 │
│                    ┌──────┴──────┐                                             │
│                    │    Redis    │  refreshToken:* (TTL 7d), bucket4j bucket   │
│                    └─────────────┘                                             │
└─────────┬─────────────────────────────────────────────────────┬────────────────┘
          │ JDBC (MySQL protocol)                               │ outbound
          │ TCP :3306                                           │ HTTPS :443 / SMTPS :465
          ↓                                                     ↓
┌───────────────────────────────┐      ┌───────────────────────────────────────────────┐
│   Lightsail DB Instance       │      │              External Services                │
│   (<masked-ip2>)              │      │  AWS S3 ap-northeast-2 ..... HTTPS :443       │
│   MySQL :3306                 │      │  Firebase FCM (HTTP v1) .... HTTPS :443       │
│   (Notion diagram: MariaDB)   │      │  Google OAuth2 token ....... HTTPS :443       │
└───────────────────────────────┘      │  Naver SMTP ................ SMTPS :465       │
                                       │  Pinpoint Collector ........ addr unknown     │
                                       └───────────────────────────────────────────────┘
```

읽는 법:

- **블루/그린은 항상 한쪽만 활성.** Nginx가 `service-url.inc` 파일 내용(8081 또는 8082)으로 업스트림을 정하고, 배포 시 파일 덮어쓰기 + reload로 전환한다 (`scripts/switch.sh`).
- **Redis는 웹 인스턴스 로컬(127.0.0.1:6379)** 에 상주하며, 캐시가 아니라 ① 리프레시 토큰 저장소(TTL 7일) ② bucket4j 분산 레이트리밋 버킷의 두 가지 역할을 한다.
- 클라이언트의 **이미지 업로드/조회는 서버를 거치지 않고 presigned URL로 S3에 직행**한다(서버는 URL만 발급, 유효 1시간).
- FCM 푸시는 서버 → FCM → 모바일 앱 방향(동아리 지원 합격/불합격 통보).
- Pinpoint Collector 주소는 서버 로컬 파일(`pinpoint-root.config`)에 있어 저장소에서 **미확인** — 점선 취급.
- DB 종류는 Notion 다이어그램상 MariaDB 로고이나 **코드 기준 MySQL** (9절 참조).

---

## 2. 인프라 토폴로지

### 2-1. 인스턴스 구성

| 인스턴스 | 주소 | 탑재 소프트웨어 | 검증 상태 |
|---|---|---|---|
| **웹 인스턴스** (Lightsail) | <masked-ip1> | Nginx(:80/:443), Spring Boot WAS ×2 (8081/8082, `nohup` 직접 실행 — systemd 아님), Redis(:6379), Pinpoint Agent 3.0.1 (`/home/ec2-user/pinpoint/pinpoint-agent-3.0.1/`) | 스크립트/설정에서 코드로 확인 (`scripts/run_new_was.sh`, `scripts/switch.sh`) |
| **DB 인스턴스** (Lightsail) | <masked-ip2> | MySQL(코드 기준; 자료는 MariaDB 표기) :3306 | 드라이버/커넥터는 코드로 확인, DB 서버 실체는 **미확인** |
| 인스턴스 스펙(RAM 등), swap 설정 | — | — | Notion 자료 영역 — 저장소에서 **검증 불가(미확인)**. 10절의 리소스 관찰은 자료의 1GB RAM 기술을 전제로 함 |

### 2-2. 네트워크 경로 / 포트

| 구간 | 프로토콜 / 포트 | 비고 |
|---|---|---|
| 클라이언트 → Nginx | HTTPS 443 / HTTP 80 | CORS origin: `https://donggurami.net`, `https://*.donggurami.net` |
| Nginx → WAS | HTTP 127.0.0.1:8081 또는 :8082 | `service-url.inc` 로 전환. **8080 리스너는 프로덕션에 존재하지 않음** (BASIC_PORT=8080은 `-Dserver.port`로 오버라이드됨) |
| WAS → Redis | TCP 127.0.0.1:6379 | 동일 인스턴스 로컬 |
| WAS → DB 인스턴스 | TCP 3306 (JDBC/MySQL 프로토콜) | 인스턴스 간 통신 |
| WAS → 외부 | HTTPS 443 (S3/FCM/OAuth2), SMTPS 465 (smtp.naver.com) | 아웃바운드 |
| WAS(Pinpoint agent) → Collector | **미확인** | 주소는 서버 로컬 config |
| GitHub Actions → 웹 인스턴스 | SSH/SCP 22 | 배포 전용 제어 평면 (데이터 평면과 구분) |

### 2-3. OS / 런타임 세팅

| 항목 | 내용 | 검증 상태 |
|---|---|---|
| Java | JDK 17. CI 빌드는 temurin, 서버 런타임은 amazon-corretto (배포판만 상이) | `.github/workflows/deploy.yml` 확인 |
| 타임존 | JDBC URL에 `serverTimezone=Asia/Seoul` 명시 | `src/main/resources/yaml/application-prod.yml` 확인. OS 레벨 타임존 설정은 **미확인** |
| swap | Notion 자료 영역 | 저장소에서 **미확인** |
| WAS 프로세스 관리 | `nohup ... &` + 로그는 `nohup.out` 리다이렉트. systemd 유닛 없음 → **인스턴스 재부팅 시 자동 기동 없음** | `scripts/run_new_was.sh` 확인 |

---

## 3. 무중단 배포 파이프라인

트리거: **main 브랜치 대상 PR이 머지될 때만** 실행 (`pull_request: types [closed]` + `merged == true`). push 트리거 아님. CodeDeploy 미사용 — 단계명만 CodeDeploy 훅 이름(AfterInstall 등)을 차용한 **SSH/scp 직접 배포**.

```
GitHub Actions       scripts @ web instance           Nginx       WAS(new)        WAS(old)
    │                           │                       │             │               │
    │ (1) checkout, JDK 17      │                       │             │               │
    │ (2) .env <- DOTENV        │                       │             │               │
    │ (3) gradlew build         │                       │             │               │
    │ (4) zip jar+.env+scripts  │                       │             │               │
    │  (5) scp deploy.zip, ssh  │                       │             │               │
    │──────────────────────────>│                       │             │               │
    │                           │ unzip -> ~/app        │             │               │
    │                           │                       │             │               │
    │  (6) ssh: run_new_was.sh  │                       │             │               │
    │──────────────────────────>│                       │             │               │
    │                           │ read service-url.inc  │             │               │
    │                           │ TARGET = other port   │             │               │
    │                           │ (no inc -> 8081)      │             │               │
    │                           │ kill zombie on TARGET │             │               │
    │                           │   nohup java -javaagent:pinpoint    │               │
    │                           │     -Dserver.port=TARGET, prod      │               │
    │                           │────────────────────────────────────>│               │
    │                           │                       │             │               │
    │    (7) ssh: health.sh     │                       │             │               │
    │──────────────────────────>│                       │             │               │
    │                           │          GET /health-check          │               │
    │                           │           (10s x max 10)            │               │
    │                           │────────────────────────────────────>│               │
    │                           │                       │             │               │
    │                           │               200 OK  │             │               │
    │                           │<------------------------------------│               │
    │                           │                       │             │               │
    │    (8) ssh: switch.sh     │                       │             │               │
    │──────────────────────────>│                       │             │               │
    │                           │   rewrite inc file    │             │               │
    │                           │    + nginx reload     │             │               │
    │                           │──────────────────────>│             │               │
    │                           │               kill old WAS (SIGTERM,│               │
    │                           │                 right after reload) │               │
    │                           │────────────────────────────────────────────────────>│
    │                           │                       │             │               │
    │ (9) rm ~/deploy.zip       │                       │             │               │
```

### 스크립트별 역할

| 스크립트 | 역할 |
|---|---|
| `scripts/run_new_was.sh` | `service-url.inc`에서 현재 포트를 `grep -Po '[0-9]+' \| tail -1`로 추출 → 반대 포트를 TARGET으로 결정, `/home/ec2-user/app/target-port`에 기록. TARGET 포트 점유 잔여 프로세스를 `lsof`+`kill`로 정리 후, `.env`(`set -a; source`) 로드 → `nohup java -javaagent:(Pinpoint 3.0.1, agentId=circle-link-server, app=app01) -Dserver.port=${TARGET_PORT} -Dspring.profiles.active=prod -jar` 기동. JAR는 `ls -tr *.jar \| tail -n 1`(최신 수정본) 선택 |
| `scripts/health.sh` | TARGET의 `http://127.0.0.1:{port}/health-check`를 **10초 간격 최대 10회** curl. 200이면 exit 0. 폴백 체인: `service-url.inc` → `target-port` 파일 → 8081 |
| `scripts/switch.sh` | `set $service_url http://127.0.0.1:${TARGET_PORT};`를 `/etc/nginx/conf.d/service-url.inc`에 덮어쓰기 → `sudo service nginx reload` → **reload 직후** 구 포트 PID를 `lsof`로 찾아 `sudo kill` (드레인 대기 없음, graceful shutdown 별도 설정 없음). CURRENT가 8081/8082 외 값이면 exit 1 (콜드스타트 폴백 없음) |

### 콜드스타트 / 자가복구(Self-Heal) 분기

```
[콜드스타트]  service-url.inc 없음
   ├─ run_new_was.sh: TARGET=8081 로 기본 기동
   └─ deploy.yml: inc 파일을 http://127.0.0.1:8082 로 생성 + nginx reload
        (→ 이후 실행이 8081을 타겟팅하도록 베이스라인 확보)

[자가복구]  health.sh 실패 (10회 모두 실패) 시 deploy.yml이:
   ① 8081/8082 양쪽 포트의 프로세스 전부 kill
   ② service-url.inc 를 8082 로 강제 리셋
   ③ run_new_was.sh 재실행  → 8081 기동
   ④ health.sh 재검사       → 실패 시 배포 실패 처리
```

- 헬스체크 대상은 Actuator가 아니라 커스텀 컨트롤러: `src/main/java/com/USWCicrcleLink/server/global/api/HealthCheckController.java` — `GET /health-check` → `200 "OK"` (permit-all).
- `.env`는 GitHub Secrets(`DOTENV`) 전체를 파일로 생성해 아티팩트(`deploy.zip`)에 포함.

---

## 4. 애플리케이션 내부 구조 (요청 처리 파이프라인)

```
클라이언트 (모바일 앱: Authorization: Bearer / 웹: Bearer + refreshToken HttpOnly 쿠키)
  │  HTTPS :443
  ↓
Nginx (:80/:443) ──HTTP──> 127.0.0.1:{8081|8082}   (forward-headers-strategy: framework)
  │
  ↓
[1] CorsFilter ──────────── preflight/OPTIONS 처리. cors.allowed-origins 프로퍼티 기반,
  │                         allowCredentials=true, Authorization 헤더 노출, maxAge 3600s
  ↓
[2] JwtFilter (OncePerRequestFilter)
  │     ├─ permit-all 경로(AntPathMatcher, application-security.yml 약 34개) → 검사 없이 통과
  │     ├─ Bearer 토큰 추출 → JJWT HS256 검증 → VALID | EXPIRED | INVALID
  │     ├─ VALID: sub 클레임 UUID → UserDetailsServiceManager 순회 조회
  │     │         (Admin → Leader → User, @Order 없음/빈 이름순 — 최대 3회 DB 조회)
  │     │         → SecurityContext 설정 + MDC(userType, userUUID)
  │     └─ EXPIRED/INVALID → CustomAuthenticationEntryPoint → 401 JSON
  ↓                          (prod에서 변조 JWT는 [SECURITY ALERT] 로그)
[3] LoggingFilter ────────── /admin/**, /notices/**, /club-leader/** 의
  │                          POST/DELETE/PUT/PATCH 만 MDC 기반 감사 로깅
  ↓
[4] 인가 (authorizeHttpRequests) ── /admin/**=ADMIN, /club-leader/**=LEADER,
  │                                 /apply/**·/mypages/** 등=USER, anyRequest=authenticated
  ↓
[5] DispatcherServlet → @RestController (17개)
  │     └─ [AOP] RateLimiterAspect @Around(@RateLimite 10곳)
  │              → bucket4j 버킷(Redis, 만료 65초) 초과 시 TOO_MANY_ATTEMPT
  ↓              (대부분 5회/5분, LEADER_CHANGE_PW만 5회/1분)
[6] Service (@Transactional)
  │     ├──> Redis: refreshToken:{...} 저장/검증 (RedisTemplate)
  │     ├──> S3: presigned URL 발급 (S3FileUploadService)
  │     └──> FCM(HTTP v1 REST) / 네이버 SMTP(@Async)
  ↓
[7] Repository: Spring Data JPA + 커스텀 4개 (EntityManager/JPQL — QueryDSL 아님)
  │     └─ [AOP] RepositoryExceptionAspect @AfterThrowing → SQLException 번역/로깅
  ↓
[8] MySQL (mysql-connector-j, jdbc:mysql://...:3306, HikariCP max 30 / min idle 10)

응답: ApiResponse<T> {message, data}
예외: GlobalExceptionHandler(@RestControllerAdvice) → ErrorResponse
      {exception, code, message, status, error, additionalData} — ExceptionType 약 103개 코드
```

핵심 상세:

- **필터 순서 확정 근거**: `global/security/config/SecurityConfig.java` — `addFilterBefore(jwtAuthFilter(), UsernamePasswordAuthenticationFilter.class)` + `addFilterAfter(loggingFilter(), JwtFilter.class)`. 세션 정책 `STATELESS`, CSRF disable.
- **인증 판별 방식**: 토큰에 role 클레임이 없음(`clubUUID`만 클레임 포함). UUID로 `CustomAdminDetailsService` / `CustomLeaderDetailsService` / `CustomUserDetailsService`를 순회하며 첫 성공을 채택 → 매 인증 요청마다 최대 3회 DB 조회.
- **토큰 체계**: Access 30분(HS256, `jwt.secret.key`). Refresh는 JWT가 아닌 **랜덤 UUID를 Redis에 `refreshToken:{uuid}` 키로 저장**(TTL 7일), HttpOnly 쿠키(prod: Secure + SameSite=None)로 전달. 로그아웃/재발급 시 `keys("refreshToken:*")` 스캔 삭제 로직 존재.
- **예외 처리**: `BaseException`(도메인 예외 22종의 부모) 외 Validation 400, UUID 타입 미스매치 400, 업로드 초과 400(개별 10MB/총 50MB), 무결성 위반 409, 최후순위 500(NO_CATCH_ERROR). **prod 프로필에서 4xx 로그 억제**.
- **알려진 결함**: `global/exception/RepositoryExceptionAspect.java` 포인트컷 문자열의 `||` 연결 오타(L28 끝 누락 / L31 후행 잔존) + 패키지 경로가 실제 구조(`club.club.repository`)와 불일치 → club 계열 리포지토리에 어드바이스 미적용 가능성 높음.

---

## 5. 논리 계층 / 도메인 구조

### 5-1. 3개 사용자 유형별 진입점

`Role` enum = `USER, ADMIN, LEADER` (`global/security/jwt/domain/Role.java`)

| 유형 | 인증 엔티티 | 로그인 진입점 | 주 사용 컨트롤러군 |
|---|---|---|---|
| 일반 사용자 (USER) — 모바일 앱 | `User` (+Profile, AuthToken) | `POST /users/login` | `/users/**`, `/mypages/**`, `/my-notices/**`, `/profiles/**`, `/apply/**`, `/users/event/**`, `/clubs/**`(조회 permit-all) |
| 동아리 회장 (LEADER) — 웹 | `Leader` | `POST /club-leader/login` | `/club-leader/**` (예외: `PATCH /club-leader/fcmtoken`은 USER 권한 — 회장 계정으로 앱 로그인한 사용자용) |
| 관리자/동연회 (ADMIN) — 웹 | `Admin` | `POST /admin/login` | `/admin/**`, `/notices/**` 쓰기 (일부 GET은 ADMIN+LEADER 공용) |
| (3역할 공통) | RefreshToken(Redis) | — | `POST /integration/logout`, `/integration/refresh-token` |

웹 클라이언트가 단일 SPA인지 별개 앱인지는 서버 코드만으로 단정 불가하나, CORS가 apex + 와일드카드 서브도메인 2개 패턴인 점은 웹 오리진이 복수임을 시사.

### 5-2. 패키지 트리 (총 241개 .java)

```
com.USWCicrcleLink.server            ← 베이스 패키지 철자 오탈자("Cicrcle") 실제 코드 그대로
├─ ServerApplication.java            (@EnableScheduling)
├─ admin/
│  ├─ admin/       api 4 · domain 1 · dto 9 · mapper 1 · repository 1 · service 4
│  └─ notice/      api 1 · domain 2 · dto 5 · repository 2 · service 1
├─ aplict/         api 1 · domain 2 · dto 2 · repository 3 · service 1   ← "Applicant" 오기
├─ club/
│  ├─ club/        api 1 · domain 10 · dto 4 · repository 11 · service 1
│  └─ clubIntro/   domain 2 · repository 2  (api/service 없음 — Club/ClubLeaderService가 사용)
├─ clubLeader/     api 2 · domain 1 · dto 5(+club 7, clubMembers 12) · repository 1
│                  · service 4(FCM, 엑셀 POI) · util 1
├─ email/          config 2 · domain 1 · repository 1 · service 2
├─ profile/        api 1 · domain 2 · dto 5 · repository 1 · service 1
├─ user/           api 4 · domain 4(+ExistingMember 2) · dto 18 · repository 8 · service 8
└─ global/
   ├─ api/         HealthCheckController, DocsController(ReDoc)
   ├─ bucket4j/    APIRateLimiter, RateLimiterAspect, @RateLimite (Redis 분산 레이트리밋)
   ├─ config/      S3Config, SchedulerConfig
   ├─ data/        DummyData, SeedData
   ├─ docs/        OpenApiConfig (springdoc)
   ├─ exception/   GlobalExceptionHandler, RepositoryExceptionAspect + errortype 23종
   ├─ redis/       RedisConfig
   ├─ response/    ApiResponse, ErrorResponse, PageResponse
   ├─ s3File/      S3FileUploadService
   ├─ security/    SecurityConfig, JwtProvider, JwtFilter/LoggingFilter,
   │               details(역할별 3종 + Manager), Integration(공통 로그아웃/토큰갱신)
   └─ validation/  Enum·파일시그니처·동아리방번호 검증, @Sanitize(jsoup XSS)
```

공통 패턴: `api(Controller) → service → repository` + `domain`, `dto`. 컨트롤러 17개 — User(`/users`), Mypage(`/mypages`), MyNotice(`/my-notices`), EventVerification(`/users/event`), Profile(`profiles` — 원문에 선행 `/` 누락), Aplict(`/apply`), Club(`/clubs`), ClubLeaderLogin·ClubLeader(`/club-leader`), AdminLogin(`/admin`), AdminClub(`/admin/clubs`), AdminClubCategory(`/admin/clubs/category`), AdminFloorPhoto(`/admin/floor/photo`), AdminNotice(`/notices`), IntegrationAuth(`/integration`), HealthCheck(`/health-check`), Docs.

---

## 6. 데이터 저장소

세 저장소의 역할 분담: **MySQL = 영속 데이터의 단일 원천 / Redis = 휘발성 인증·제어 상태 / S3 = 이미지 바이너리 (DB에는 key만 저장)**.

### 6-1. MySQL — 엔티티 22개 (`ddl-auto: none`)

```
User 1─1 Profile N─(ClubMembers)─N Club 1─1 ClubIntro 1─N ClubIntroPhoto
User 1─1 AuthToken / WithdrawalToken        Club 1─1 ClubMainPhoto, 1─N ClubHashtag
Profile N─(Aplict)─N Club                   Club N─(ClubCategoryMapping)─N ClubCategory
Leader 1─1 Club                             Admin 1─N Notice 1─N NoticePhoto
ClubMemberTemp 1─N ClubMemberAccountStatus N─1 Club
독립 테이블: EmailToken(5분 만료), EventVerification, FloorPhoto
RefreshToken은 JPA 엔티티가 아님 → Redis 저장
```

- 특이점: `Profile.user_id`는 **nullable** (계정 없는 기존부원 프로필 허용). `Aplict`는 club+profile+checked 유니크 제약. `ClubCategory`의 테이블명은 `ClUB_CATEGORY_TABLE` — **오탈자가 DB에 그대로 존재**.
- 커스텀 리포지토리 4개(`ClubRepositoryCustomImpl`, `ClubMembersRepositoryImpl`, `AplictRepositoryCustomImpl`, `ClubMemberAccountStatusRepositoryImpl`)는 전부 **EntityManager + JPQL** (QueryDSL 의존성 자체가 없음).
- HikariCP: max 30 / min idle 10 / conn-timeout 30s / idle 600s / max-lifetime 1800s. p6spy 쿼리 로깅.

### 6-2. Redis — 키 용도 / TTL

| 키 패턴 | 용도 | TTL | 근거 파일 |
|---|---|---|---|
| `refreshToken:{랜덤 UUID}` → 사용자 UUID | 리프레시 토큰 저장소 (로그아웃/재발급 시 `keys()` 스캔 삭제) | **7일** (604800000ms) | `global/security/jwt/JwtProvider.java` |
| bucket4j 버킷 (clientId+action 조합 bucketId) | API 레이트리밋 (`LettuceBasedProxyManager`) | **65초** 만료 전략 | `global/bucket4j/APIRateLimiter.java` |

주의: `global/redis/RedisConfig.java`의 `new LettuceConnectionFactory()`(인자 없음)는 프로퍼티를 무시하고 **localhost:6379 고정** — bucket4j용 `RedisClient`만 `REDIS_HOST`/`REDIS_PORT`를 실제 사용 (9절 참조).

### 6-3. S3 — 키 prefix 구조 (버킷 `${SERVER_AWS_S3_BUCKET}`, ap-northeast-2)

| prefix | 용도 | 쓰는 서비스 |
|---|---|---|
| `mainPhoto/` | 동아리 대표 사진 | ClubLeaderService |
| `introPhoto/` | 동아리 소개 사진 | ClubLeaderService |
| `floorPhoto/` | 동아리방 층별 지도 | AdminFloorPhotoService |
| `noticePhoto/` | 공지 사진 | AdminNoticeService |

업로드 = presigned **PUT** URL 발급(클라이언트 → S3 직접), 조회 = presigned **GET**, 유효 **1시간**. 확장자(jpg/jpeg/png) + 파일 시그니처 검증. 삭제는 서버가 직접 호출. (`global/s3File/Service/S3FileUploadService.java`, `global/config/S3Config.java` — prod 자격증명은 `default` 체인=.env 액세스키, test만 `instance-profile`)

---

## 7. 외부 서비스 연동

| 주체 → 대상 | 프로토콜 / 포트 | 용도 | 설정 위치 |
|---|---|---|---|
| 클라이언트 → Nginx | HTTPS 443 / HTTP 80 | API 진입점 | 서버측 Nginx(저장소 외), CORS는 `yaml/application-prod.yml` |
| Nginx → Spring Boot | HTTP 127.0.0.1:8081\|8082 | 리버스 프록시 (블루/그린 전환) | `scripts/switch.sh`, `/etc/nginx/conf.d/service-url.inc` |
| Spring Boot → MySQL | JDBC(TCP) 3306 | 주 데이터 저장 (JPA/Hikari) | `yaml/application-prod.yml` |
| Spring Boot(RedisTemplate) → Redis | TCP 6379 (**localhost 고정**) | 리프레시 토큰 | `global/redis/RedisConfig.java`, `JwtProvider.java` |
| Spring Boot(bucket4j Lettuce) → Redis | TCP `${REDIS_HOST}:${REDIS_PORT}` | 레이트리밋 버킷 | `RedisConfig.java`, `global/bucket4j/APIRateLimiter.java` |
| Spring Boot → AWS S3 | HTTPS 443 | presigned URL 발급(1h) · 객체 삭제 | `S3Config.java`, `S3FileUploadService.java` |
| 클라이언트 → AWS S3 (직접) | HTTPS 443 (presigned PUT/GET) | 이미지 업로드/조회 | 동일 |
| Spring Boot → FCM (`fcm.googleapis.com`) | HTTPS 443, **HTTP v1 REST** (Admin SDK Messaging 미사용) | 지원 합격/불합격 푸시 → 모바일 앱. 401/400 시 DB의 FCM 토큰 무효화 | `clubLeader/service/FcmServiceImpl.java` |
| Spring Boot → Google OAuth2 | HTTPS 443 | 서비스계정 JSON → Bearer 토큰 | `FcmServiceImpl.getAccessToken()`, `${firebase.config-path}` |
| Spring Boot → `smtp.naver.com` | **SMTPS 465 (SSL)** | 가입 인증 링크·인증코드·아이디 찾기·탈퇴코드 메일 (`@Async`, 수신자 `@suwon.ac.kr` 강제) | `email/config/MailConfig.java`(호스트 하드코딩), `yaml/application-secret.yml` |
| Pinpoint Agent(JVM 내) → Collector | **미확인** (서버 로컬 config) | APM 트레이싱 | `scripts/run_new_was.sh` (javaagent만, 앱 코드 흔적 없음) |
| GitHub Actions → 웹 인스턴스 | SSH/SCP 22 | 무중단 배포 | `.github/workflows/deploy.yml` |

---

## 8. 스케줄 작업

`global/config/SchedulerConfig.java` (`@EnableScheduling` — `ServerApplication.java`), 4개 전부 `@Transactional`:

| 메서드 | cron | 대상 |
|---|---|---|
| `deleteExpiredTokens` | `0 0 * * * *` (매시 정각) | 만료 1시간 경과 EmailToken 삭제 (미인증 회원 정리) |
| `deleteOldApplications` | `0 0 0 * * ?` (매일 자정) | 최초 합격 통보 후 4일 경과(deleteDate 도래) 지원서(Aplict) 삭제 |
| `deleteExpiredFcmTokens` | `0 0 0 * * ?` (매일 자정) | 인증 타임스탬프 만료된 Profile의 FCM 토큰 null 처리 |
| `deleteExpiredClubMemberTemp` | `0 0 0 * * ?` (매일 자정) | 만료 임시회원(ClubMemberTemp) + 연관 AccountStatus 삭제 |

---

## 9. ⚠️ Notion 자료와 코드의 불일치 / 주의사항

| # | 자료 기술 | 실제 코드 | 판정 |
|---|---|---|---|
| 1 | DB 다이어그램에 **MariaDB** 로고 | `com.mysql:mysql-connector-j`(build.gradle), `com.mysql.cj.jdbc.Driver` + `jdbc:mysql://`(application-prod.yml), 로컬 도커 `mysql:8.0.35` | **불일치** — 구조도는 MySQL로 표기. (드라이버 호환으로 서버 실체가 MariaDB일 가능성은 남음 — 미확인) |
| 2 | `.env`의 `PROD_DB_URL` (완성형 JDBC URL 1개) | `PROD_MYSQL_HOST/PORT/NAME/USER/PASS` **5개 분리 변수**로 URL 조립 | **불일치 — 자료의 .env 목록이 낡음** |
| 3 | `REDIS_HOST=127.0.0.1:6379` (합쳐진 값) | `REDIS_HOST` + `REDIS_PORT` **별도 변수**. 합쳐 쓰면 기동 실패 | **형식 불일치** (Redis가 로컬이라는 사실 자체는 코드와 모순 없음) |
| 4 | `BASIC_PORT=8080` | `run_new_was.sh`의 `-Dserver.port=8081\|8082`가 오버라이드 → **프로덕션에 8080 리스너 없음** | **불일치** — 8080은 스크립트 없이 직접 실행할 때의 폴백일 뿐 |
| 5 | `PROD_SERVER_URL=http://<masked-ip1>:8080` | 이메일 인증 링크 baseUrl(`email.config.baseUrl`)로 사용 중. 코드상 Nginx 80/443을 가리켜야 정합 | **정합성 의문** — 실제 서버 .env 값 **미확인** |
| 6 | "QueryDSL 커스텀 리포지토리" | 커스텀 리포지토리 4개는 전부 **EntityManager + JPQL**. QueryDSL 의존성/임포트 전무 | **불일치** |
| 7 | UserDetails 조회 "User→Leader→Admin 순차" | `@Order` 없음 → 빈 등록 순서(통상 이름 알파벳순): **Admin→Leader→User** | **불일치(뉘앙스)** — 순서는 명시 제어 아님 |
| 8 | Redis 용도 불명시(캐시 뉘앙스) | ① 리프레시 토큰 저장소(TTL 7일) ② bucket4j 레이트리밋 버킷 | **보완 필요** — 단순 캐시로 그리면 부정확 |
| 9 | 다이어그램에 S3/FCM/SMTP/Pinpoint/모바일앱/GitHub Actions 없음 | 전부 실사용 확인 (S3Config, FcmServiceImpl, MailConfig, run_new_was.sh javaagent, deploy.yml) | **자료 단순화** — 본 문서 1절 다이어그램에 반영 |
| 10 | "jojoldu 방식 변형" 배포 | 서술 자체는 정확하나 CodeDeploy/S3 배포가 아닌 **GitHub Actions SSH/scp 직접 배포** + 자가복구 단계 추가 | **부분 일치** |
| 11 | CORS `https://donggurami.net`만 언급 | `https://*.donggurami.net` 와일드카드도 허용 | **보완** |
| 12 | (자료 무관, 코드 결함 기록) | ⓐ `RedisConfig`의 `LettuceConnectionFactory()`가 프로퍼티 무시하고 localhost 고정 — Redis 분리 시 리프레시 토큰 저장 파손 ⓑ `RepositoryExceptionAspect` 포인트컷 `\|\|` 오타 + 패키지 경로 불일치 ⓒ `MailConfig` 커스텀 빈이 secret.yml의 starttls 설정을 사실상 무시(SSL 사용) | **주의사항** |

---

## 10. 구조적 관찰

아키텍처 관점의 코멘트 (검증된 사실에 근거):

1. **단일 인스턴스 블루/그린의 본질적 한계** — 이 구성은 "무중단 배포"이지 "고가용성"이 아니다. 웹 인스턴스 1대가 Nginx·WAS 2개·Redis를 모두 담으므로 인스턴스 장애 = 전체 서비스 중단. DB 인스턴스 역시 단일이며, 결과적으로 **웹 인스턴스와 DB 인스턴스 각각이 SPOF**다.
2. **리소스 제약** — 전환 순간에는 JVM 2개(각각 Pinpoint javaagent 부착) + Redis + Nginx가 동시 상주한다. 자료 기준 1GB RAM 인스턴스(스펙은 저장소에서 미확인)라면 전환 구간의 메모리 압박이 상당하며, swap 의존이 클 것으로 추정된다. HikariCP max 30 × 2 JVM이면 전환 순간 DB 커넥션도 최대 60개까지 열릴 수 있다.
3. **드레인 없는 구 WAS 종료** — `switch.sh`가 Nginx reload **직후** 구 WAS를 kill한다. reload는 기존 커넥션을 유지하지만, 구 WAS에서 처리 중이던 장기 요청은 SIGTERM 시점에 유실될 수 있다 (Spring graceful shutdown 별도 설정 없음).
4. **nohup 기반 프로세스 관리** — systemd가 아니므로 인스턴스 재부팅·프로세스 비정상 종료 시 자동 복구가 없다. 복구 수단이 "다음 배포의 Self-Heal 로직"뿐이며, Self-Heal 자체도 양쪽 포트를 전부 kill하므로 **그 경로는 다운타임을 수반**한다.
5. **Redis 결합도** — `LettuceConnectionFactory()` localhost 고정 때문에 Redis를 별도 인스턴스로 분리하는 순간 리프레시 토큰 기능이 조용히 깨진다. 또한 `keys("refreshToken:*")` 스캔은 O(N) 블로킹 연산으로 키가 늘면 성능 리스크.
6. **인증 경로의 DB 부하** — 토큰에 role 클레임이 없어 매 인증 요청마다 최대 3개 테이블을 순차 조회한다. role 클레임 추가 또는 조회 결과 캐싱으로 줄일 수 있는 구조적 비용.
7. **배포 보안** — `.env` 전체가 GitHub Secrets(`DOTENV`) → 빌드 아티팩트(`deploy.zip`)에 포함되어 전송된다. 아티팩트 취급 범위가 곧 시크릿 노출 범위.
8. **이메일 링크 경로의 불일치 잠재** — 이메일 인증 링크 baseUrl이 `PROD_SERVER_URL`(자료상 `:8080`)인데 프로덕션에 8080 리스너가 없다. 실제 .env가 Nginx(80/443) 주소가 아니라면 인증 링크가 동작하지 않는 구조 — 서버 .env 확인 필요(미확인).

---

### 참조 파일

- 배포: `scripts/run_new_was.sh`, `scripts/health.sh`, `scripts/switch.sh`, `.github/workflows/deploy.yml`
- 설정: `src/main/resources/application.yml`, `src/main/resources/yaml/application-{prod,local,test,secret,file,security}.yml`, `build.gradle`, `docker-compose.yml`
- 보안/인증: `src/main/java/com/USWCicrcleLink/server/global/security/config/SecurityConfig.java`, `global/security/jwt/JwtProvider.java`, `global/security/jwt/filter/JwtFilter.java`
- 인프라 연동: `global/redis/RedisConfig.java`, `global/bucket4j/APIRateLimiter.java`, `global/config/S3Config.java`, `global/s3File/Service/S3FileUploadService.java`, `clubLeader/service/FcmServiceImpl.java`, `email/config/MailConfig.java`
- 기타: `global/api/HealthCheckController.java`, `global/config/SchedulerConfig.java`, `global/exception/GlobalExceptionHandler.java`, `global/exception/RepositoryExceptionAspect.java`
