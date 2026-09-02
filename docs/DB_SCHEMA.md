# 동구라미 DB 구조

이 문서는 동구라미(USW-Circle-Link) 서버 저장소의 JPA 엔티티에 대응하는 **22개 테이블**의 스키마 명세다. 모든 DDL·행수·데이터 통계는 프로덕션 덤프 `docs/동구라미DB_CircleServer.sql` 에서 추출했다.

---

## 1. 개요

| 항목 | 값 |
|---|---|
| 테이블 수 | **22개** (JPA 엔티티와 1:1 대응) |
| 총 행수 | **1,961행** |
| 엔진 | 전 테이블 `InnoDB` |
| 문자셋 | 21개 테이블 `utf8mb4 / utf8mb4_0900_ai_ci`, `event_verification_table` 만 `utf8mb4_unicode_ci` |
| PK 설계 | 22/22 테이블 `bigint AUTO_INCREMENT` 서러게이트 키. 복합 PK·자연키 0개 |
| 외부 식별자 | `binary(16)` UUID 13개 컬럼 (내부 조인은 `bigint`, 외부 API는 UUID) |
| FK | 18개, **전부 `ON DELETE` 미지정 = RESTRICT** (캐스케이드 0개) |
| UNIQUE 제약 | 24개 |

### 3개 계정 체계

동구라미의 인증 주체는 **물리적으로 분리된 3개 테이블**이다. 공통 상위 계정 테이블도, 상속 매핑도 없다.

| 테이블 | 주체 | 행수 | 인증 구현 |
|---|---|---|---|
| `user_table` | 앱 사용자(학생) | 508 | `CustomUserDetails` |
| `leader_table` | 동아리 회장 | 31 | `CustomLeaderDetails` |
| `admin_table` | 운영자(동아리연합회) | 2 | `CustomAdminDetails` |

세 테이블 모두 각자의 PK·UUID·비밀번호 컬럼을 갖고, 동일한 `role enum('ADMIN','LEADER','USER')` 정의(공용 `Role` enum)를 독립적으로 공유한다. 비밀번호는 모두 `BCryptPasswordEncoder`(`PasswordEncoderConfig`)로 해시되어 `varchar(255)`에 저장되며 평문 저장 경로는 없다.

### 도메인 영역 구분

| 영역 | 테이블 수 | 내용 |
|---|---|---|
| **계정 · 인증** | 6 | 사용자 계정·프로필·각종 인증 토큰·이벤트 인증 이력 |
| **동아리** | 9 | 동아리 본체와 소개/사진/카테고리/해시태그/회장/동아리원 위성 테이블 |
| **지원 · 공지 · 기타** | 7 | 지원서, 기존 동아리원 일괄 가입 요청, 공지·첨부, 층별 도면, 관리자 계정 |

구조상 허브는 **`club_table`**(자식 6개가 `club_id` 참조)과 **`user_table`**(자식 3개) 두 곳뿐이고, `profile_table`이 계정 영역과 동아리/지원 영역을 잇는 유일한 다리다. 순환 참조는 없다.

---

## 2. ⚠️ 개인정보 주의

> **`docs/동구라미DB_CircleServer.sql` 은 프로덕션 덤프이며 실사용자 개인정보를 그대로 포함한다.**
>
> - 사용자 계정 **508건** (로그인 ID·학교 이메일), 프로필 **505건** (실명·학번·휴대폰 번호·학과)
> - BCrypt 비밀번호 해시 **541건** (user 508 + leader 31 + admin 2)
> - FCM 푸시 토큰 **13건**, 이벤트 인증 스냅샷 **210건**(계정·이메일 사본 포함)
> - 동아리 소속 관계 **353건**, 지원서 **9건** — 개인 단위로 식별 가능
>
> 이 파일은 저장소 외부 공유·재배포·로그 출력 금지 대상이다. 분석 시에는 건수·분포 등 **통계만** 인용하고 실제 값은 인용하지 않는다. 로컬 개발용 시드가 필요하면 별도의 마스킹 데이터를 사용해야 한다.

---

## 3. ERD

```
범례 :  ──▶  FK 제약 있음        ‥‥▶  FK 없음 (애플리케이션이 무결성 책임)
        [1] 계정·인증·운영   [2] 동아리   [3] 지원·가입요청
        박스 = 테이블명 / 엔티티명,  화살표는 자식(FK 보유) → 부모 방향

[1] ACCOUNT / AUTH / ADMIN
┌──────────────────┐        ┌──────────────────┐                        ┌─────────────┐
│ auth_token_table │        │ withdrawal_token │                        │ admin_table │
│ AuthToken        │        │ WithdrawalToken  │                        │ Admin       │
└─────────┬────────┘        └─────────┬────────┘                        └──────▲──────┘
          │ user_uuid -> uuid         │ user_uuid -> uuid                      │ admin_id
          │                           │                                ┌───────┴──────┐
          └──────────┬────────────────┘                                │ notice_table │
                     │                                                 │ Notice       │
              ┌──────▼─────┐        ┌───────────────────┐              └───────▲──────┘
              │ user_table │◀‥‥‥‥‥‥ │ email_token_table │                      │ notice_id
              │ User       │ email  │ EmailToken        │           ┌──────────┴─────────┐
              └──────▲─────┘        └───────────────────┘           │ notice_photo_table │
                     │ user_id                                      │ NoticePhoto        │
             ┌───────┴───────┐                                      └────────────────────┘
             │ profile_table │
             │ Profile       │
             └───────────────┘

[2] CLUB
┌───────────────────┐                                                                         ┌────────────┐
│ floor_photo_table │                                                               ┌────────▶│ club_table │
│ FloorPhoto        │  standalone : no FK in / out                                  │         │ Club       │
└───────────────────┘                                                               │         └────────────┘
                                    ┌──────────────┐                                │
                                    │ leader_table │────────────────────────────────┤
                                    │ Leader       │ club_id  (UNIQUE, 1:1)         │
                                    └──────────────┘                                │
                                                                                    │
                                    ┌───────────────────────┐                       │
                                    │ club_main_photo_table │───────────────────────┤
                                    │ ClubMainPhoto         │ club_id  (UNIQUE, 1:1)│
                                    └───────────────────────┘                       │
                                                                                    │
                                    ┌────────────────────┐                          │
    [1] profile_table.profile_id ◀──│ club_members_table │──────────────────────────┤
                                    │ ClubMembers        │ club_id  (N:1)           │
                                    └────────────────────┘                          │
                                                                                    │
                                    ┌────────────────────┐                          │
                                    │ club_hashtag_table │──────────────────────────┤
                                    │ ClubHashtag        │ club_id  (N:1)           │
                                    └────────────────────┘                          │
                        club_category_id                                            │
┌─────────────────────┐             ┌─────────────────────────────┐                 │
│ club_category_table │◀────────────│ club_category_mapping_table │─────────────────┤
│ ClubCategory        │             │ ClubCategoryMapping         │ club_id  (N:1)  │
└─────────────────────┘             └─────────────────────────────┘                 │
                         club_info_id                                               │
┌───────────────────────┐           ┌─────────────────┐                             │
│ club_info_photo_table │──────────▶│ club_info_table │─────────────────────────────┘
│ ClubIntroPhoto        │           │ ClubIntro       │ club_id  (UNIQUE, 1:1)
└───────────────────────┘           └─────────────────┘

[3] APPLICATION / JOIN-REQUEST
┌──────────────┐
│ aplict_table │──▶ [1] profile_table.profile_id      (FK)
│ Aplict       │──▶ [2] club_table.club_id            (FK)
└──────────────┘

┌───────────────────────┐                         ┌─────────────────────────────────┐
│ club_membertemp_table │◀────────────────────────│ club_member_accountstatus_table │──▶ [2] club_table.club_id  (FK)
│ ClubMemberTemp        │  clubmembertemp_id      │ ClubMemberAccountStatus         │
└───────────────────────┘                         └─────────────────────────────────┘

┌──────────────────────────┐
│ event_verification_table │‥‥▶ [1] user_table.uuid               (no FK)
│ EventVerification        │‥‥▶ [1] profile_table.profile_id      (no FK)
│                          │‥‥▶ [2] club_table.club_uuid          (no FK)
└──────────────────────────┘
```

---

## 4. 테이블 카탈로그

> 표기: `club_info_table` = 엔티티 **`ClubIntro`**, `club_info_photo_table` = 엔티티 **`ClubIntroPhoto`**. 이하 DB 실제 이름을 기준으로 쓰고 엔티티명을 병기한다.

### 4-1. 계정 · 인증 (6개 / 1,226행)

| 테이블 | 대응 엔티티 | 역할 | 행수 |
|---|---|---|---:|
| `user_table` | `User` | 앱 사용자 계정 — 로그인 ID/이메일/BCrypt 비밀번호/역할/외부 UUID | 508 |
| `profile_table` | `Profile` | 사용자 신원정보(실명·학번·학과·연락처)와 FCM 토큰. user와 1:1(계정 없이도 존재 가능) | 505 |
| `auth_token_table` | `AuthToken` | 비밀번호 찾기 4자리 인증코드, 사용자당 1건 | 3 |
| `withdrawal_token` | `WithdrawalToken` | 회원 탈퇴 확인 4자리 인증코드, 사용자당 1건 | 0 |
| `email_token_table` | `EmailToken` | 가입 전 이메일 인증 토큰(5분 만료) + 가입용 UUID 발급 | 0 |
| `event_verification_table` | `EventVerification` | 오프라인 이벤트(동밤) 참여 인증 이력. 사용자 정보 스냅샷 보관 | 210 |

### 4-2. 동아리 (9개 / 675행)

| 테이블 | 대응 엔티티 | 역할 | 행수 |
|---|---|---|---:|
| `club_table` | `Club` | 동아리 마스터 — 이름/분과/동아리방/회장 표시 연락처 | 31 |
| `club_info_table` | `ClubIntro` | 동아리 소개글·모집글·구글폼 URL·모집상태. club과 1:1 | 31 |
| `club_info_photo_table` | `ClubIntroPhoto` | 소개 사진 메타데이터(S3 키 + 슬롯 1~5) | 155 |
| `club_main_photo_table` | `ClubMainPhoto` | 대표 사진 1장. club과 1:1 | 31 |
| `club_category_table` | `ClubCategory` | 관리자가 관리하는 카테고리 마스터 | 33 |
| `club_category_mapping_table` | `ClubCategoryMapping` | 동아리↔카테고리 N:M 조인 | 40 |
| `club_hashtag_table` | `ClubHashtag` | 동아리 해시태그(마스터 없는 비정규화 1:N) | 10 |
| `leader_table` | `Leader` | 동아리 회장 로그인 계정. club과 1:1 | 31 |
| `club_members_table` | `ClubMembers` | 동아리 소속 회원(동아리↔프로필 N:M) | 353 |

### 4-3. 지원 · 공지 · 기타 (7개 / 20행)

| 테이블 | 대응 엔티티 | 역할 | 행수 |
|---|---|---|---:|
| `aplict_table` | `Aplict` | 동아리 지원서 — 상태(WAIT/PASS/FAIL)·통보 여부·자동삭제 예정일 | 9 |
| `club_membertemp_table` | `ClubMemberTemp` | 기존 동아리원 일괄 가입 시 회원정보 임시 보관(7일 만료) | 0 |
| `club_member_accountstatus_table` | `ClubMemberAccountStatus` | 임시회원 → 동아리 회장에게 보낸 승인 요청 큐 | 0 |
| `notice_table` | `Notice` | 동아리연합회 공지사항 | 6 |
| `notice_photo_table` | `NoticePhoto` | 공지 첨부 이미지(S3 키 + 순서) | 0 |
| `floor_photo_table` | `FloorPhoto` | 학생회관 층별(B1/F1/F2) 동아리방 배치도 | 3 |
| `admin_table` | `Admin` | 관리자(동아리연합회) 계정 | 2 |

---

## 5. 계정 · 인증 테이블 상세

### 5-1. `user_table` — 일반 사용자 계정 (`User`, 508행, AI=5677)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `user_id` | bigint AI | NO | - | PK. 엔티티 `userId`, `GenerationType.IDENTITY` |
| `uuid` | binary(16) | NO | - | 엔티티 `userUUID`. 외부(API/JWT) 노출 식별자. `@PrePersist`에서 `UUID.randomUUID()` |
| `user_account` | varchar(20) | NO | - | 로그인 아이디 |
| `email` | varchar(30) | NO | - | 학교 이메일. 레이트리밋 `getClientId()` 반환값으로도 사용 |
| `user_pw` | varchar(255) | NO | - | BCrypt 해시. `updateUserPw()`로만 변경 |
| `role` | enum('ADMIN','LEADER','USER') | NO | - | `@Enumerated(STRING)` + 공용 `Role` |
| `user_created_at` | datetime(6) | NO | - | 가입 시각. `@PrePersist` |
| `user_updated_at` | datetime(6) | NO | - | 최종 수정 시각. `@PrePersist`/`@PreUpdate` |

- **UUID 컬럼명만 접두사가 없다.** 다른 테이블은 `club_uuid`·`leader_uuid`·`admin_uuid`인데 이 테이블만 `uuid`(엔티티 필드는 `userUUID`, `@Column(name="uuid")`).
- **토큰 테이블이 PK가 아닌 `uuid`를 참조한다.** `auth_token_table`/`withdrawal_token`이 `@JoinColumn(referencedColumnName="uuid")`로 UNIQUE 컬럼을 FK 대상으로 삼는다. 서비스가 JWT에서 뽑은 UUID만으로 토큰을 조회(`findByUserUserUUID`)하기 때문이다.
- 소프트 딜리트 플래그·`deleted_at`이 없고 탈퇴는 물리 삭제다. 인바운드 FK 3개가 모두 RESTRICT이므로 `UserService.cancelMembership()`은 프로필·토큰을 먼저 지운 뒤 사용자를 지우는 순서 의존이 코드에 박혀 있다.

### 5-2. `profile_table` — 사용자 프로필 (`Profile`, 505행, AI=7763)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `profile_id` | bigint AI | NO | - | PK |
| `user_id` | bigint | **YES** | NULL | `user_table` 참조. `@OneToOne @JoinColumn(nullable = true)` |
| `user_name` | varchar(30) | NO | - | 실명 |
| `student_number` | varchar(8) | NO | - | 학번 |
| `user_hp` | varchar(11) | NO | - | 휴대폰 번호(하이픈 없는 11자리) |
| `major` | varchar(20) | NO | - | 학과/전공명 문자열 |
| `member_type` | enum('NONMEMBER','REGULARMEMBER') | NO | - | 정회원/비회원. `MemberType`, `@Enumerated(STRING)` |
| `fcm_token` | varchar(255) | YES | NULL | 푸시 알림 FCM 등록 토큰 |
| `fcm_token_updated_at` | datetime(6) | YES | NULL | 토큰 만료 판정 기준 시각. 엔티티 필드 `fcmTokenCertificationTimestamp` |
| `profile_created_at` | datetime(6) | NO | - | 생성 시각 |
| `profile_updated_at` | datetime(6) | NO | - | 수정 시각 |

- **`user_id` nullable이 이 테이블의 핵심 설계.** 회장이 엑셀로 동아리원을 일괄 등록할 때 계정 없이 프로필만 생성(`memberType = NONMEMBER`)한다. 즉 프로필은 "앱 계정을 가진 학생"과 "명단에만 있는 학생"을 모두 표현한다. MySQL UNIQUE는 NULL 중복을 허용하므로 계정 미연결 프로필은 개수 제한 없이 공존하며, UNIQUE는 "한 계정이 두 프로필을 가질 수 없다"만 보장한다.
- **`major`가 코드 참조가 아닌 자유 입력 `varchar(20)`** 이라 엑셀 등록 시 표기 흔들림이 그대로 저장될 수 있다.
- FCM 토큰은 프로필당 1개(멀티 디바이스 미지원). `SchedulerConfig.deleteExpiredFcmTokens()`가 매일 자정 만료분의 `fcm_token`을 NULL로 만든다 — 행 삭제가 아닌 컬럼 무효화.

### 5-3. `auth_token_table` — 비밀번호 찾기 인증코드 (`AuthToken`, 3행, AI=50)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `auth_token_id` | bigint AI | NO | - | PK |
| `user_uuid` | binary(16) | YES | NULL | 대상 사용자. `@OneToOne(LAZY) @JoinColumn(referencedColumnName="uuid", unique=true)` |
| `auth_code` | varchar(255) | NO | - | 4자리 숫자 인증코드. `isAuthCodeValid()`로 문자열 비교 |

- **만료 시각 컬럼이 없다.** 수명은 상태 전이로 관리된다: 재발송 시 `auth_code` 덮어쓰기(`createOrUpdateAuthToken`), 검증 성공 시 행 삭제, 탈퇴 시 잔여 행 정리. 스케줄러 정리 대상도 아니라 **검증을 완료하지 않고 이탈한 요청만 무기한 잔존**한다(현재 3행).
- **인증코드는 해시 없이 평문 저장**된다(비밀번호와 달리 `PasswordEncoder`를 거치지 않음). `varchar(255)`에 항상 4자가 들어가는 것은 엔티티에 `length` 미지정이라 JPA 기본값이 적용된 결과.

### 5-4. `withdrawal_token` — 탈퇴 인증코드 (`WithdrawalToken`, 0행, AI=53)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `withdrawal_id` | bigint AI | NO | - | PK. `@Column(name = "WITHDRAWAL_ID")` |
| `user_uuid` | binary(16) | YES | NULL | 탈퇴 요청 사용자. `referencedColumnName = "uuid"` |
| `withdrawal_code` | varchar(255) | NO | - | 4자리 숫자 인증코드 |

- `auth_token_table`과 **컬럼 구성·제약·수명주기가 사실상 동일한 쌍둥이 테이블**이다. 만료 컬럼 없음, 사용자당 1건 UNIQUE, 재요청 시 덮어쓰기, 검증 후 삭제까지 같고 `generateRandomAuthCode()`도 각각 복제되어 있다.
- 항상 비워지는 방향으로 동작해 0행이며(탈퇴 완료 시 토큰·사용자가 함께 사라짐), AI=53이 누적 발급 시도 규모를 보여준다.
- **22개 중 유일하게 `_table` 접미사가 없는 테이블명**이다(`@Table(name = "WITHDRAWAL_TOKEN")`).

### 5-5. `email_token_table` — 회원가입 이메일 인증 (`EmailToken`, 0행, AI=1055)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `email_token_id` | bigint AI | NO | - | PK |
| `email_token_uuid` | binary(16) | NO | - | 인증 메일 링크에 실리는 토큰 식별자 |
| `signup_uuid` | binary(16) | YES | NULL | 이메일 인증 **완료 후** 발급되는 가입용 UUID (`verifyEmail()`) |
| `email` | varchar(30) | NO | - | 인증 요청 대상 이메일 |
| `expiration_time` | datetime(6) | NO | - | 만료 시각. 생성 시 `now()+5분`, `extendExpirationTime()`으로 연장 |
| `is_verified` | bit(1) | NO | - | 인증 완료 여부. `@Builder.Default false` (DB DEFAULT 절 없음) |

- **UUID 2단계 설계**: `email_token_uuid`는 "메일 링크를 클릭한 사람"을, 인증 후 발급되는 `signup_uuid`는 "가입 폼 제출 권한"을 증명한다(`UserService.isEmailVerified()`).
- **유일하게 `expiration_time`을 컬럼으로 가진 토큰 테이블**이며 `SchedulerConfig.deleteExpiredTokens()`가 매시 정각 만료 1시간 경과분을 물리 삭제한다(5분 만료 + 1시간 유예).
- `email` UNIQUE 때문에 같은 주소로 두 건의 인증 요청이 공존할 수 없다. 재요청은 새 행이 아니라 기존 행의 만료 연장으로 처리된다.
- FK가 없다 — 가입 이전 단계라 대응하는 `user_table` 행이 아직 없기 때문이며, `email` 문자열이 유일한 논리적 연결고리다.

### 5-6. `event_verification_table` — 이벤트 인증 이력 (`EventVerification`, 210행, AI=211)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `event_verification_id` | bigint AI | NO | - | PK. 엔티티 `id` |
| `user_uuid` | binary(16) | NO | - | 인증한 사용자 UUID. `columnDefinition = "BINARY(16)"` 명시 |
| `club_uuid` | binary(16) | NO | - | 인증 시점 소속 동아리 UUID |
| `user_id` | bigint | YES | NULL | 인증 시점 `user_table.user_id` 스냅샷 |
| `profile_id` | bigint | YES | NULL | 인증 시점 `profile_table.profile_id` 스냅샷 |
| `user_account` | varchar(20) `utf8mb4_unicode_ci` | YES | NULL | 인증 시점 로그인 아이디 스냅샷 |
| `email` | varchar(30) `utf8mb4_unicode_ci` | YES | NULL | 인증 시점 이메일 스냅샷 |
| `verified` | bit(1) | NO | - | 인증 성공 플래그 |
| `verified_at` | datetime(6) | NO | - | 인증 완료 시각 |

- **FK가 하나도 없고 참조 대신 값을 복사하는 스냅샷/이력 테이블**이다. 원본 사용자가 사라져도 참여 기록이 남고, 사용자 삭제도 이 테이블 때문에 막히지 않는다.
- **UNIQUE 입도가 엔티티보다 좁다.** 엔티티는 `@UniqueConstraint(columnNames = {"user_uuid","club_uuid"})`를 선언하지만 DB의 `UK_event_user_club`은 **`user_uuid` 단일 컬럼**이다. 따라서 DB가 강제하는 규칙은 "사용자당 이벤트 인증 평생 1건"이며, 애플리케이션(`existsByUserUUIDAndClubUUID`)의 (사용자, 동아리) 단위 가정과 어긋난다.
- **NULL 허용 범위도 엔티티보다 넓다.** 엔티티는 `user_id`·`user_account`·`email`을 `nullable = false`로 선언하나 DB는 전부 NULL 허용이다.
- `verified`는 사실상 상수 `true`다(`create()`가 항상 `.verified(true)`, 실패는 예외로 처리되어 행이 생기지 않음). `club_uuid`는 `findClubUUIDsByProfileId(...).get(0)`으로 정해지므로 복수 소속자는 조회 순서에 의존한다.
- 인증 코드 자체는 DB가 아니라 `@Value("${event.code:1115}")` 설정값이다.

---

## 6. 동아리 테이블 상세

동아리 도메인 9개 테이블에는 **생성/수정 시각 컬럼이 하나도 없다**(계정·공지 영역과 대조적). 감사 이력을 남기지 않는다. 1:1은 자식 쪽 `club_id`에 UNIQUE를 걸어, N:M은 별도 조인 테이블 + 대리키로 구현한다.

### 6-1. `club_table` — 동아리 본체 (`Club`, 31행, AI=74)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_id` | bigint AI | NO | - | PK |
| `club_uuid` | binary(16) | NO | - | 외부 노출 식별자. `UUID.randomUUID()` 기본 + `@PrePersist` 보정, `updatable=false` |
| `club_name` | varchar(10) | NO | - | 동아리명 |
| `department` | enum('ACADEMIC','ART','RELIGION','SHOW','SPORT','VOLUNTEER') | NO | - | 소속 분과. `Department` enum(학술/예술/종교/공연/체육/봉사를 `@JsonValue`로 매핑) |
| `club_room_number` | varchar(4) | NO | - | 동아리방 호수 |
| `leader_name` | varchar(30) | NO | - | 회장 이름(표시용) |
| `leader_hp` | varchar(11) | NO | - | 회장 연락처 |
| `club_insta` | varchar(255) | YES | NULL | 인스타그램 계정 |

- **분과(`department`)와 카테고리(`club_category_table`)는 별개 축이다.** 전자는 코드에 하드코딩된 6값 enum, 후자는 관리자가 추가/삭제하는 데이터 주도 마스터. 동아리 하나가 분과 1개 + 카테고리 N개를 갖는다.
- **회장 표시 연락처(`leader_name`/`leader_hp`)와 로그인 주체(`leader_table`)가 분리**되어 있고 두 값의 동기화는 보장되지 않는다.
- 동아리 생성 시 `leaderName`/`leaderHp`/`clubInsta`는 빈 문자열 `""`로 채워진다 — NOT NULL 컬럼이라 `""`가 "미입력"을 뜻한다.

### 6-2. `club_info_table` — 소개/모집 정보 (`ClubIntro`, 31행, AI=73)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_info_id` | bigint AI | NO | - | PK. 엔티티 `clubIntroId` |
| `club_id` | bigint | NO | - | 소유 동아리 FK. `@OneToOne(LAZY)` |
| `club_info` | varchar(3000) | YES | NULL | 동아리 소개글. 엔티티 `clubIntro` |
| `club_recruitment` | varchar(3000) | YES | NULL | 모집 안내글 |
| `google_form_url` | varchar(255) | YES | NULL | 외부 지원 구글폼 링크 |
| `club_info_recruitment_status` | enum('CLOSE','OPEN') | NO | - | 모집 상태. `RecruitmentStatus`, `@Enumerated(STRING)` |

- 기본값은 DB DEFAULT가 아니라 **엔티티 필드 초기화**(`= RecruitmentStatus.CLOSE`)로만 존재한다. JPA를 우회한 INSERT는 값을 명시해야 한다. 토글은 `toggleRecruitmentStatus()`(OPEN ⇄ CLOSE).
- `ClubIntroRepository.findOpenClubIds()`가 `recruitmentStatus='OPEN'` 전체 스캔을 하는데 해당 컬럼에 인덱스가 없다(31행이라 실질 영향 없음).
- 본문이 `TEXT`가 아닌 `varchar(3000)`이라 3000자가 하드 리밋이다.

### 6-3. `club_info_photo_table` — 소개 사진 (`ClubIntroPhoto`, 155행, AI=361)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_info_photo_id` | bigint AI | NO | - | PK |
| `club_info_id` | bigint | NO | - | 소속 소개정보 FK. `@ManyToOne(LAZY)` |
| `photo_order` | int | NO | - | 사진 슬롯 번호(1~5). 엔티티 필드 `order` |
| `club_info_photo_name` | varchar(255) | YES | NULL | 업로드 원본 파일명 |
| `club_info_photo_s3key` | varchar(255) | YES | NULL | S3 오브젝트 키 |

- **행을 미리 만들어 두는 고정 슬롯 방식**이다. 동아리 생성 시 `AdminClubService.createClubIntroPhotos()`가 `order = 1..5`인 빈 행 5개를 즉시 INSERT하고, 이후 업로드는 INSERT가 아니라 해당 슬롯 행의 **UPDATE**다. 사진 삭제도 행 삭제가 아니라 값 비우기 — 즉 **행수는 사진 개수가 아니라 슬롯 개수**를 뜻한다.
- 슬롯 상한 5는 애플리케이션 상수(`ClubLeaderService.PHOTO_LIMIT = 5`)와 `validateOrderValues()`로만 강제된다. DB에는 `(club_info_id, photo_order)` UNIQUE도 CHECK도 없어 슬롯 중복/범위 초과를 막지 못하며, 슬롯 조회가 `Optional` 단건이라 중복 행이 생기면 조회 자체가 깨진다.

### 6-4. `club_main_photo_table` — 대표 사진 (`ClubMainPhoto`, 31행, AI=166)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_main_photo_id` | bigint AI | NO | - | PK |
| `club_id` | bigint | NO | - | 소유 동아리 FK. `@OneToOne(LAZY)` |
| `club_main_photo_name` | varchar(255) | YES | NULL | 업로드 원본 파일명 |
| `club_main_photo_s3key` | varchar(255) | YES | NULL | S3 오브젝트 키 |

- 슬롯 개념 없이 1:1 단일 행(소개 사진의 5슬롯과 대비). 동아리 생성 시 빈 값(`""`)으로 선생성되고 이후 UPDATE로 채워지므로 **행 존재 ≠ 사진 존재**다.

### 6-5. `club_hashtag_table` — 해시태그 (`ClubHashtag`, 10행, AI=11)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_hashtag_id` | bigint AI | NO | - | PK |
| `club_id` | bigint | NO | - | 소유 동아리 FK. `@ManyToOne(LAZY)` |
| `club_hashtag` | varchar(10) | NO | - | 해시태그 문자열 |

- **태그 마스터 테이블이 없다.** 카테고리와 달리 문자열을 동아리별로 중복 저장하는 비정규화 1:N 구조라, 태그 기준 역검색은 문자열 비교가 된다.
- `(club_id, club_hashtag)` UNIQUE가 없어 같은 동아리에 같은 태그가 중복 등록되는 것을 DB가 막지 않는다. 갱신은 `deleteAllByClub_ClubIdAndClubHashtagNotIn()` 후 추가 방식.

### 6-6. `club_category_table` — 카테고리 마스터 (`ClubCategory`, 33행, AI=61)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_category_id` | bigint AI | NO | - | PK |
| `club_category_uuid` | binary(16) | NO | - | 외부 노출 식별자. `updatable=false` |
| `club_category_name` | varchar(20) | NO | - | 카테고리명 |

- **`club_category_name`에 UNIQUE가 없다.** 그런데 매핑 정리 로직(`deleteAllByClub_ClubIdAndClubCategory_ClubCategoryNameNotIn`)이 **이름 문자열 기준**으로 동작하므로, 이름 중복은 매핑 갱신의 정확도에 직접 영향을 준다.

### 6-7. `club_category_mapping_table` — 동아리↔카테고리 N:M (`ClubCategoryMapping`, 40행, AI=54)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_category_mapping_id` | bigint AI | NO | - | PK(대리키) |
| `club_id` | bigint | NO | - | 동아리 FK. `@ManyToOne(LAZY)` |
| `club_category_id` | bigint | NO | - | 카테고리 FK. `@ManyToOne(LAZY)` |

- `@ManyToMany` 대신 명시적 조인 엔티티지만 추가 속성이 없어 사실상 순수 조인 테이블이며, 복합 PK가 아닌 대리키를 쓴다. **`(club_id, club_category_id)` UNIQUE가 없어** 같은 쌍이 중복 저장될 수 있다.
- 양방향 조회(`findClubsByCategoryIds` / `findByClubClubId`)가 각각 FK 인덱스로 커버된다.

### 6-8. `leader_table` — 동아리 회장 계정 (`Leader`, 31행, AI=73)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `leader_id` | bigint AI | NO | - | PK |
| `leader_uuid` | binary(16) | NO | - | 외부 노출 식별자. `updatable=false` |
| `leader_account` | varchar(255) | NO | - | 로그인 아이디 |
| `leader_pw` | varchar(255) | NO | - | BCrypt 해시 비밀번호 |
| `club_id` | bigint | **YES** | NULL | 담당 동아리 FK. `@OneToOne(LAZY)` |
| `role` | enum('ADMIN','LEADER','USER') | NO | - | 공용 `Role` enum, 생성 시 항상 `LEADER` |
| `is_agreed_terms` | bit(1) | NO | - | 약관 동의 여부. DB DEFAULT 없음(엔티티에서만 `false` 초기화) |

- **`club_id` NULL 허용 + UNIQUE**: "동아리당 회장 최대 1명"은 보장하되, 무소속 회장 계정은 여러 개 공존할 수 있다(NULL 중복 허용).
- **`leader_uuid`에 UNIQUE도 인덱스도 없다** — 다른 모든 노출용 UUID(`club_uuid`, `club_category_uuid`, `club_member_uuid`, `admin_uuid`, `user_table.uuid` 등)가 UNIQUE인 것과 대비되는 유일한 예외다. 엔티티에도 `unique = true`가 없다.
- `@PrePersist`가 조건 없이 `leaderUUID`를 덮어쓴다(다른 엔티티는 `null`일 때만 생성).
- `leader_account`가 `varchar(255)` — 엔티티에 `length` 미지정이라 JPA 기본값이 반영된 결과이며, `user_account`/`admin_account`(각 20)와 유일하게 다르다.

### 6-9. `club_members_table` — 동아리원 (`ClubMembers`, 353행, AI=7729)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_member_id` | bigint AI | NO | - | PK |
| `club_member_uuid` | binary(16) | NO | - | 외부 노출 식별자(회장의 동아리원 관리 API 대상 지정). `updatable=false` |
| `club_id` | bigint | NO | - | 소속 동아리 FK. `@ManyToOne(LAZY)` |
| `profile_id` | bigint | NO | - | 회원 프로필 FK. `@ManyToOne(LAZY)` |

- **회원 유형(`member_type`)은 이 테이블이 아니라 `profile_table`에 있다.** 즉 회원 구분은 동아리별 속성이 아니라 프로필 전역 속성이며, 이 테이블은 UUID 외에 아무 속성도 없는 순수 조인 엔티티다.
- **`(club_id, profile_id)` UNIQUE가 없다** → 같은 회원의 같은 동아리 중복 등록을 DB가 막지 못하고, 애플리케이션이 `existsByProfileAndClubUUID()`로 사전 확인한다(같은 도메인의 `aplict_table`이 `uk_aplict_active`로 DB 레벨 차단을 하는 것과 대조적).
- 조인 테이블에 UUID를 부여해 **조인 레코드 자체가 API 자원으로 노출**되는 설계다. 동아리원 삭제 시 남은 소속이 없는 프로필은 `deleteAllByIdInBatch()`로 함께 정리되어, 조인 행 삭제가 프로필 생명주기에 영향을 준다.

---

## 7. 지원 · 공지 · 기타 테이블 상세

### 7-1. `aplict_table` — 동아리 지원서 (`Aplict`, 9행, AI=442)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `aplict_id` | bigint AI | NO | - | PK |
| `aplict_uuid` | binary(16) | NO | - | 외부 노출 식별자. `@PrePersist` 생성, `updatable = false` |
| `club_id` | bigint | NO | - | 지원 대상 동아리. `@ManyToOne(LAZY)` |
| `profile_id` | bigint | NO | - | 지원자 프로필. `@ManyToOne(LAZY)` |
| `aplict_submitted_at` | datetime(6) | NO | - | 제출 시각 |
| `aplict_status` | enum('FAIL','PASS','WAIT') | NO | - | 심사 상태. `AplictStatus`, 기본 `WAIT` |
| `aplict_checked` | bit(1) | NO | `b'0'` | 회장의 최초 합/불 통보 완료 여부. 엔티티 `boolean checked` |
| `aplict_delete_date` | datetime(6) | YES | NULL | 자동 삭제 예정 시각. 통보 시 `now + 4일`, 미처리는 NULL |
| `active_flag` | tinyint (STORED 생성 컬럼) | YES | 계산값 | 미처리만 1, 처리 완료는 NULL. 유니크 제약 전용 파생 컬럼 |

```sql
`active_flag` tinyint GENERATED ALWAYS AS ((case when (`aplict_checked` = 0x00) then 1 else NULL end)) STORED
```

- **`uk_aplict_active(club_id, profile_id, active_flag)`는 부분 유니크 인덱스 에뮬레이션**이다. MySQL에 조건부 UNIQUE가 없어 생성 컬럼으로 흉내내며, NULL 중복 허용 특성 덕분에 **"(동아리, 지원자)당 미처리 지원서 1건"만 강제하고 처리 완료된 과거 지원 이력은 몇 건이든 공존**한다(재모집 재지원 허용).
- 이 제약이 필요한 이유는 `AplictService.submitAplict()`가 **선(先)조회-후(後)삽입** 구조이기 때문이다. SELECT와 INSERT 사이에 락이 없어 동시 요청 시 양쪽 모두 통과할 수 있고, 두 번째 INSERT를 DB가 거부하면 서비스가 `DataIntegrityViolationException`을 `ALREADY_APPLIED`로 변환한다 — 애플리케이션 검증의 race를 DB 제약이 최종 방어하는 구조다.
- 수명주기: `checked=false, delete_date=NULL`(대기) → 통보 시 `checked=true` + `delete_date=+4일`이 함께 세팅되어 유니크 제약에서 빠짐 → `SchedulerConfig.deleteOldApplications()`가 매일 자정 물리 삭제. 추가 합격(`FAIL→PASS`)은 `delete_date`를 건드리지 않아 원래 삭제 일정을 그대로 따른다.

### 7-2. `club_membertemp_table` — 기존 동아리원 가입 임시 데이터 (`ClubMemberTemp`, 0행, AI=55020)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_membertemp_id` | bigint AI | NO | - | PK |
| `profile_temp_account` | varchar(20) | NO | - | 가입 예정 계정 ID. 확정 시 `user_table.user_account`로 이관 |
| `profile_temp_pw` | varchar(255) | NO | - | 인코딩된 비밀번호(`passwordEncoder.encode`) |
| `profile_temp_name` | varchar(30) | NO | - | 이름 |
| `profile_temp_student_number` | varchar(8) | NO | - | 학번 |
| `profile_temp_hp` | varchar(11) | NO | - | 휴대폰 번호(하이픈 제거 후 저장) |
| `profile_temp_major` | varchar(20) | NO | - | 학과 |
| `profile_temp_email` | varchar(30) | NO | - | 이메일 |
| `total_club_request` | int | NO | - | 가입 요청을 보낸 총 동아리 수 |
| `club_request_count` | int | NO | - | 회장이 수락한 누적 횟수(기본 0) |
| `club_member_temp_expiry_date` | datetime(6) | NO | - | 요청 마감 시각. 생성 시 `now + 7일` |

- 엔티티에 `@Column(name=...)`이 없어 Hibernate 기본 네이밍 전략의 snake_case가 그대로 컬럼명이 된 유일한 테이블이다.
- **UNIQUE·FK·부가 인덱스가 하나도 없다.** `findByProfileTempAccount`/`findByProfileTempEmail` 단건 조회와 만료 스캔이 모두 풀스캔이다.
- `total_club_request == club_request_count`가 되는 순간에만 `user_table` 계정이 생성되고 이 행은 삭제된다. 미완료 행은 자정 스케줄러가 만료분을 정리한다.

### 7-3. `club_member_accountstatus_table` — 가입 승인 요청 (`ClubMemberAccountStatus`, 0행, AI=55022)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `club_member_accountstatus_id` | bigint AI | NO | - | PK |
| `clubmember_account_status_uuid` | binary(16) | NO | - | 외부 노출 식별자. `@PrePersist` 생성, `updatable = false` |
| `club_id` | bigint | NO | - | 승인 주체 동아리. `@ManyToOne(LAZY)` |
| `clubmembertemp_id` | bigint | NO | - | 요청자 임시 데이터. `@ManyToOne(LAZY)` |

**두 테이블이 구현하는 "N개 동아리 동시 승인" 플로우**

1. `UserService.registerClubMemberTemp()` — `ClubMemberTemp` 1행 생성(`total_club_request` = 선택 동아리 수, 만료 7일).
2. `UserService.sendRequest()` — 동아리마다 `ClubMemberAccountStatus` 1행씩 분배(temp 1 : status N).
3. `checkRequest()` — 저장 건수와 요청 UUID 집합을 대조해 누락/오배송 검출.
4. 회장이 수락하면 `club_request_count`를 1 올리고 해당 요청 행을 삭제. 거절도 행 삭제만 한다 — **상태 컬럼이 없고 행의 존재 자체가 "미처리"를 뜻한다**(테이블명은 accountstatus지만 status 컬럼은 없다).
5. 전원 승인 시 계정이 생성되고 프로필이 `NONMEMBER → REGULARMEMBER`로 승격되며 temp 행이 삭제된다.
6. 7일 내 미완료 시 스케줄러가 자식(status) → 부모(temp) 순서로 삭제한다. CASCADE가 없어 이 순서를 애플리케이션이 보장해야 한다.

- `(club_id, clubmembertemp_id)` UNIQUE가 없어 중복 요청 행이 가능하고, 그 경우 `club_request_count`가 `total_club_request`를 초과해 `==` 비교 기반 계정 생성 조건이 영영 성립하지 않을 수 있다.

### 7-4. `notice_table` — 공지사항 (`Notice`, 6행, AI=24)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `notice_id` | bigint AI | NO | - | PK |
| `notice_uuid` | binary(16) | NO | - | 외부 노출 식별자. `@PrePersist` 생성, `updatable = false` |
| `notice_title` | varchar(200) | NO | - | 제목. `updateTitle()` |
| `notice_content` | varchar(3000) | NO | - | 본문. `updateContent()` |
| `notice_created_at` | datetime(6) | NO | - | 작성 시각. `@PrePersist`, `updatable = false` |
| `admin_id` | bigint | NO | - | 작성자 관리자. `@ManyToOne(LAZY)` |

- `updated_at`이 없다 — 수정 API는 있지만 수정 시각은 기록되지 않는다. 목록 정렬용 `notice_created_at` 인덱스도 없다(현재 6행).
- `admin_id`가 NOT NULL + RESTRICT라 공지를 남긴 관리자 계정은 물리 삭제될 수 없다.

### 7-5. `notice_photo_table` — 공지 첨부 사진 (`NoticePhoto`, 0행, AI=15)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `notice_photo_id` | bigint AI | NO | - | PK |
| `notice_id` | bigint | NO | - | 소속 공지. `@ManyToOne(LAZY)` |
| `notice_photo_name` | varchar(255) | YES | NULL | 업로드 원본 파일명 |
| `notice_photo_s3key` | varchar(255) | YES | NULL | S3 오브젝트 키(`noticePhoto/` 접두) |
| `photo_order` | int | NO | - | 노출 순서. 엔티티 필드명 `order`가 SQL 예약어라 `@Column(name="photo_order")` 명시 |

- `(notice_id, photo_order)` UNIQUE가 없어 같은 공지 안에서 순서 값이 중복될 수 있다. 순서·개수 검증(최대 5장)은 애플리케이션에서만 한다.
- 사진 갱신이 전량 삭제 후 재삽입 방식이라 AUTO_INCREMENT만 증가하고 잔존 행은 0이다. 파일명/S3 키는 엔티티에 `nullable=false`가 없어 NULL 허용이지만 실제 경로에서는 항상 채워진다.

### 7-6. `floor_photo_table` — 동아리방 층별 도면 (`FloorPhoto`, 3행, AI=10)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `floor_photo_id` | bigint AI | NO | - | PK |
| `floor_photo_name` | varchar(255) | NO | - | 업로드 원본 파일명 |
| `floor_photo_s3_key` | varchar(255) | NO | - | S3 오브젝트 키(`floorPhoto/` 접두). 엔티티 `floorPhotoS3key` |
| `floor_photo_floor` | enum('B1','F1','F2') | NO | - | 층 구분. `FloorPhotoEnum`, `@Enumerated(STRING)` |

- **FK도 UNIQUE도 없는 완전 독립 테이블**이다. `findByFloor`가 `Optional`을 반환해 층당 1행을 전제하지만, 유일성은 `AdminFloorPhotoService.uploadPhoto()`의 "조회 → S3 삭제 → 행 삭제 → 신규 저장" 규약에만 의존한다. 동시 업로드 시 2행이 생기면 이후 조회가 `NonUniqueResultException`으로 실패한다.
- S3 키 컬럼명만 `_s3_key`(언더스코어 포함)로 다른 사진 테이블의 `_s3key` 표기와 다르다. `photo_order`가 없는 유일한 사진 테이블이기도 하다(층당 1장).

### 7-7. `admin_table` — 관리자 계정 (`Admin`, 2행, AI=3)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|---|---|---|---|---|
| `admin_id` | bigint AI | NO | - | PK |
| `admin_uuid` | binary(16) | NO | - | 외부 노출 식별자. `@PrePersist` 생성, `updatable = false` |
| `admin_account` | varchar(20) | NO | - | 로그인 계정 ID. `findByAdminAccount`로 인증 |
| `admin_name` | varchar(30) | NO | - | 관리자 표시명(UNIQUE 아님) |
| `admin_pw` | varchar(255) | NO | - | 인코딩된 비밀번호. 동아리 삭제 등 민감 작업 재확인에도 `matches()` 사용 |
| `role` | enum('ADMIN','LEADER','USER') | NO | - | 공용 `Role` enum. 이 테이블에는 ADMIN만 저장 |

- `SeedData`가 `prod` 프로파일 기동 시 2개 계정을 시딩하며 이후 추가되지 않았다. 생성/수정 시각 컬럼이 없다.

---

## 8. 제약조건과 인덱스

### 8-1. FK 전수 (18개) — **ON DELETE/ON UPDATE 절을 가진 FK는 0개**

| # | 자식 테이블.컬럼 | → 부모 테이블.컬럼 | 제약명 | ON DELETE |
|---|---|---|---|---|
| 1 | `aplict_table.club_id` | `club_table.club_id` | `FKpgsmuq2edvc3f77991md7v532` | RESTRICT(기본) |
| 2 | `aplict_table.profile_id` | `profile_table.profile_id` | `FK6wi76h2y2qsmy2656tceudjns` | RESTRICT(기본) |
| 3 | `auth_token_table.user_uuid` | **`user_table.uuid`** | `FKjp5ba1wed90a80moimojs560w` | RESTRICT(기본) |
| 4 | `withdrawal_token.user_uuid` | **`user_table.uuid`** | `FK6c2k0dy0gqfcplx7ilb4tal0h` | RESTRICT(기본) |
| 5 | `profile_table.user_id` | `user_table.user_id` | `FKcetpmkcyc3a92gsj8nhoh081x` | RESTRICT(기본) |
| 6 | `notice_table.admin_id` | `admin_table.admin_id` | `FKjalbbeoh84yuwx2arvtvj6fqq` | RESTRICT(기본) |
| 7 | `notice_photo_table.notice_id` | `notice_table.notice_id` | `FK6tviuae77mqwi2rs51ltt08fc` | RESTRICT(기본) |
| 8 | `leader_table.club_id` | `club_table.club_id` | `FK52pi0c71l94mhd5m69whg7hh6` | RESTRICT(기본) |
| 9 | `club_info_table.club_id` | `club_table.club_id` | `FKlup1d8ory7f9m1eox3i9kou5h` | RESTRICT(기본) |
| 10 | `club_info_photo_table.club_info_id` | `club_info_table.club_info_id` | `FK3ji4aeww6b04mhoaific7x6u` | RESTRICT(기본) |
| 11 | `club_main_photo_table.club_id` | `club_table.club_id` | `FK27yaifeis35eh8re9ws2awans` | RESTRICT(기본) |
| 12 | `club_members_table.club_id` | `club_table.club_id` | `FKpajae55weujg4wrl6j5504hf9` | RESTRICT(기본) |
| 13 | `club_members_table.profile_id` | `profile_table.profile_id` | `FK8i2w2ypaa8qqy88pqs2hjmguo` | RESTRICT(기본) |
| 14 | `club_hashtag_table.club_id` | `club_table.club_id` | `FKkpyay1m38g7vh4tu71lm0c7kc` | RESTRICT(기본) |
| 15 | `club_category_mapping_table.club_id` | `club_table.club_id` | `FKqnsk78w64baquvhwmbm7y6gw7` | RESTRICT(기본) |
| 16 | `club_category_mapping_table.club_category_id` | `club_category_table.club_category_id` | `FKgw16m5b2elnbrki3aly6mea3r` | RESTRICT(기본) |
| 17 | `club_member_accountstatus_table.club_id` | `club_table.club_id` | `FKqyx12i0emv9w8gx9lschjp3n6` | RESTRICT(기본) |
| 18 | `club_member_accountstatus_table.clubmembertemp_id` | `club_membertemp_table.club_membertemp_id` | `FKn1342r1u74pcqjrtgs4ym6sfq` | RESTRICT(기본) |

- 제약명 16개는 Hibernate 자동 생성명(`FK`+해시)이고, 사람이 이름을 붙인 제약은 `uk_aplict_active`와 `UK_event_user_club` 둘뿐이다.
- **3·4번은 PK가 아닌 UNIQUE 컬럼(`user_table.uuid`)을 참조**한다. MySQL에서 허용되지만 참조 키가 `binary(16)`이라 `user_id` 참조보다 조인 비용·인덱스 크기가 크다.
- CASCADE가 하나도 없어 삭제 순서는 전적으로 애플리케이션 몫이다. `ClubRepositoryCustomImpl.deleteClubAndDependencies()`가 `ClubMemberAccountStatus → ClubHashtag → ClubCategoryMapping → ClubMembers → Aplict → ClubIntroPhoto → ClubMainPhoto → ClubIntro → Leader → Club` 순으로 JPQL 벌크 DELETE를 수행한다. 공지(`deleteExistingPhotos` → notice), 임시회원(`deleteAccountStatus` → temp)도 같은 패턴이다.

### 8-2. UNIQUE 제약 전수 (24개)

| 테이블 | 제약명 | 컬럼 구성 | 의미 |
|---|---|---|---|
| `admin_table` | `UKhr5m0vcflcbm7t8gmav9f39b2` | (`admin_uuid`) | 외부 노출 식별자 |
| `admin_table` | `UK953lj6fco5mjkoj7inhapd80o` | (`admin_account`) | 로그인 ID |
| `aplict_table` | `UKrnxw4j85csarbk4fo24wtdqyv` | (`aplict_uuid`) | 외부 노출 식별자 |
| `aplict_table` | **`uk_aplict_active`** | (`club_id`,`profile_id`,`active_flag`) | 미처리 지원서 1건 (조건부 UNIQUE 에뮬레이션) |
| `auth_token_table` | `UK75x948cs6xsvbu45mhm7e9hu7` | (`user_uuid`) | 사용자당 인증코드 1건 |
| `club_category_table` | `UKnum3sfv6v7iufm1o0wyjsrhwf` | (`club_category_uuid`) | 외부 노출 식별자 |
| `club_info_table` | `UKddj7v7unhymy51jd0hluh81mi` | (`club_id`) | 동아리당 소개 1건 (1:1) |
| `club_main_photo_table` | `UK76tpqa9bsriusk0cvfwkx72qt` | (`club_id`) | 동아리당 대표사진 1건 (1:1) |
| `club_member_accountstatus_table` | `UKp3lo9mttehwwrrplv4x6m1maw` | (`clubmember_account_status_uuid`) | 외부 노출 식별자 |
| `club_members_table` | `UKjtad2vh7uwh4df1y4tim24kdh` | (`club_member_uuid`) | 외부 노출 식별자 |
| `club_table` | `UKn6w23t3vxgnkkqi7a4ftsxu7y` | (`club_name`) | 동아리명 중복 금지 |
| `club_table` | `UKipk908s91xa42jeq5x8a7grur` | (`club_uuid`) | 외부 노출 식별자 |
| `email_token_table` | `UK3n8hu1ege6ydvgit741l1pru4` | (`email_token_uuid`) | 인증 링크 토큰 |
| `email_token_table` | `UKgs4dyhheavdtkq5hxoi3eoqbo` | (`email`) | 이메일당 진행 중 토큰 1건 |
| `email_token_table` | `UK2jffq9j36txl98jljkhksi8lx` | (`signup_uuid`) | 인증 완료 후 가입 토큰 |
| `event_verification_table` | **`UK_event_user_club`** | (`user_uuid`) ← **단일 컬럼** | 사용자당 이벤트 인증 1건 |
| `leader_table` | `UK8sh967f3er1ryycpg6who33xu` | (`leader_account`) | 회장 로그인 ID |
| `leader_table` | `UKr5ah158u22btiq3w6dfqj1asw` | (`club_id`) | 동아리당 회장 1명 (1:1) |
| `notice_table` | `UKgruu4uahfmrw8n8pk2jjvwe6f` | (`notice_uuid`) | 외부 노출 식별자 |
| `profile_table` | `UK3cjengfu1esrsno7l4jnwxu16` | (`user_id`) | user↔profile 1:1 |
| `user_table` | `UKas85vd3tv1f8ljew12glptgpc` | (`uuid`) | 외부 노출 식별자 (FK 참조 대상) |
| `user_table` | `UKtdttpwysmji8eiskjjyfu9ai9` | (`user_account`) | 로그인 ID |
| `user_table` | `UKeamk4l51hm6yqb8xw37i23kb5` | (`email`) | 이메일 중복 가입 금지 |
| `withdrawal_token` | `UKjtnyylehrufmjyombys88b8lc` | (`user_uuid`) | 사용자당 탈퇴코드 1건 |

**UNIQUE가 없는데 애플리케이션은 유일성을 가정하는 곳**

| 대상 | 애플리케이션 가정 | 위험 |
|---|---|---|
| `club_members_table(club_id, profile_id)` | `findByProfileProfileIdAndClubClubId` → `Optional` | 동시 등록 시 중복 소속 |
| `club_category_mapping_table(club_id, club_category_id)` | 중복 매핑 없음 | 동일 쌍 중복 저장 |
| `floor_photo_table(floor_photo_floor)` | `findByFloor` → `Optional` | 동시 업로드 시 `NonUniqueResultException` |
| `club_category_table(club_category_name)` | `existsByClubCategoryName` 사전 검사 | 이름 중복 시 매핑 정리 로직 오작동 |
| `club_info_photo_table(club_info_id, photo_order)` | 슬롯 단건 조회 | 슬롯 중복 시 조회 실패 |
| `notice_photo_table(notice_id, photo_order)` | 순서 유일 | 순서 중복 |
| `club_member_accountstatus_table(club_id, clubmembertemp_id)` | 요청 1건 | 카운터 초과로 계정 생성 불가 |
| `leader_table(leader_uuid)` | 토큰 인증 시 단건 조회 | UNIQUE도 인덱스도 없는 유일한 노출용 UUID |

### 8-3. 인덱스 목록 (PK 22 + UNIQUE 24 + 일반 20)

| 테이블 | PK | 일반 인덱스(비유니크) |
|---|---|---|
| `admin_table` | `admin_id` | — |
| `aplict_table` | `aplict_id` | `FKpgsmuq…`(club_id), `FK6wi76…`(profile_id), `idx_aplict_club_checked`(club_id, aplict_checked), `idx_aplict_club_uuid_checked`(club_id, aplict_uuid, aplict_checked), `idx_aplict_club_checked_status`(club_id, aplict_checked, aplict_status), `idx_aplict_delete_date`(aplict_delete_date), `idx_aplict_wait`(aplict_status, active_flag, club_id, profile_id, aplict_submitted_at) |
| `auth_token_table` | `auth_token_id` | — |
| `club_category_mapping_table` | `club_category_mapping_id` | `FKqnsk78…`(club_id), `FKgw16m5…`(club_category_id) |
| `club_category_table` | `club_category_id` | — |
| `club_hashtag_table` | `club_hashtag_id` | `FKkpyay1…`(club_id) |
| `club_info_photo_table` | `club_info_photo_id` | `FK3ji4ae…`(club_info_id) |
| `club_info_table` | `club_info_id` | — |
| `club_main_photo_table` | `club_main_photo_id` | — |
| `club_member_accountstatus_table` | `club_member_accountstatus_id` | `FKqyx12i…`(club_id), `FKn1342r…`(clubmembertemp_id) |
| `club_members_table` | `club_member_id` | `FKpajae5…`(club_id), `FK8i2w2y…`(profile_id) |
| `club_membertemp_table` | `club_membertemp_id` | **없음** |
| `club_table` | `club_id` | — |
| `email_token_table` | `email_token_id` | — |
| `event_verification_table` | `event_verification_id` | `idx_user_uuid`(user_uuid), `idx_user_id`(user_id), `idx_profile_id`(profile_id) |
| `floor_photo_table` | `floor_photo_id` | **없음** |
| `leader_table` | `leader_id` | — |
| `notice_photo_table` | `notice_photo_id` | `FK6tviua…`(notice_id) |
| `notice_table` | `notice_id` | `FKjalbbe…`(admin_id) |
| `profile_table` | `profile_id` | — |
| `user_table` | `user_id` | — |
| `withdrawal_token` | `withdrawal_id` | — |

전략 요약: **`aplict_table` 한 곳만 의도적으로 튜닝**되어 있고(수동 명명 인덱스 5개, 커버링 목적 5컬럼 복합 인덱스 포함), 나머지 21개의 인덱스는 사실상 Hibernate가 FK/UNIQUE 때문에 자동 생성한 것뿐이다.

### 8-4. FK 없이 애플리케이션이 책임지는 참조 관계

| 참조하는 쪽 | 컬럼 | 논리적 대상 | 연결 방식 |
|---|---|---|---|
| `event_verification_table` | `user_uuid` (NOT NULL) | `user_table.uuid` | 인증 시점 식별자 복사 |
| `event_verification_table` | `user_id`, `profile_id` (NULL 허용) | `user_table`, `profile_table` | 비정규화 사본 |
| `event_verification_table` | `club_uuid` (NOT NULL) | `club_table.club_uuid` | `existsByUserUUIDAndClubUUID`로만 조회 |
| `event_verification_table` | `user_account`, `email` | `user_table` 동명 컬럼 | 값 스냅샷(원본 변경 시 비동기화) |
| `email_token_table` | `email` (UNIQUE) | `user_table.email` | 가입 전 단계라 user 행 없음 → 이메일 문자열이 유일한 고리 |
| `email_token_table` | `signup_uuid` (UNIQUE) | 가입 절차상의 사용자 | 어떤 테이블도 참조하지 않음 |
| `club_membertemp_table` | `profile_temp_*` 7개 | `profile_table`, `user_table` | 이름/학번/전화/이메일 문자열 매칭 |
| `leader_table` | `leader_account` | `user_table.user_account` | 스키마상 무관계(데이터상 상당수 중복 존재) |
| `admin_table` | `admin_account` | `user_table.user_account` | 동일 |

### 8-5. 인덱스 누락 지점 (리포지토리 쿼리 기준)

| 테이블 | 인덱스 없이 실행되는 쿼리 | 제안 |
|---|---|---|
| `leader_table` | `findByLeaderUUID`, `findClubUUIDByLeaderUUID` (회장 요청마다 호출) | `UNIQUE(leader_uuid)` |
| `club_membertemp_table` | `findByProfileTempAccount/Email/Name…Hp`, `findAllByClubMemberTempExpiryDateBefore` | `(profile_temp_account)`, `(profile_temp_email)`, `(club_member_temp_expiry_date)` |
| `profile_table` | `findByUserNameAndStudentNumberAndUserHp`, `findByUserNameIn…UserHpIn(…MajorIn)` | `(student_number, user_name, user_hp)` |
| `profile_table` | `findAllByFcmTokenCertificationTimestampBefore` (FCM 만료 스케줄러) | `(fcm_token_updated_at)` |
| `email_token_table` | `findAllByExpirationTimeBefore` (만료 정리 스케줄러) | `(expiration_time)` |
| `club_members_table` | `findByProfileProfileIdAndClubClubId`, `existsByProfileAndClubUUID` | `UNIQUE(club_id, profile_id)` — 성능·유일성 동시 해결 |
| `club_category_table` | `existsByClubCategoryName`, `findByClubCategoryName` | `UNIQUE(club_category_name)` |
| `club_table` | `existsByClubRoomNumber` | `(club_room_number)` 또는 UNIQUE |
| `club_info_photo_table` | `findByClubIntro_ClubIntroIdAndOrder`, `ORDER BY order` | `(club_info_id, photo_order)` |
| `notice_photo_table` | `findByNotice` + 순서 출력 | `(notice_id, photo_order)` |
| `event_verification_table` | 동아리 단위 집계/조회 | `(club_uuid)` — 어떤 인덱스에도 없음 |
| `floor_photo_table` | `findByFloor` (`Optional`) | `UNIQUE(floor_photo_floor)` (정합성 목적) |

**중복·과잉 인덱스**

- `event_verification_table.idx_user_uuid` 는 `UNIQUE UK_event_user_club(user_uuid)` 와 컬럼 구성이 완전히 같은 순수 중복이다(삭제 가능).
- `aplict_table.FKpgsmuq…(club_id)` 는 여러 복합 인덱스의 선두 프리픽스와 겹쳐 사실상 잉여다(반면 `profile_id` 쪽 FK 인덱스는 어떤 인덱스에서도 선두가 아니라 필요).
- `idx_aplict_club_uuid_checked` 는 이미 UNIQUE인 `aplict_uuid` 를 두 번째 컬럼에 둔 형태로 9행 규모에서 실익이 없다. `aplict_table` 은 9행에 세컨더리 인덱스 7개 — 테이블보다 인덱스가 크다.

---

## 9. 타입 · 네이밍 관례

| 항목 | 관례 | 상세 |
|---|---|---|
| **PK** | 예외 없이 `bigint AUTO_INCREMENT` 단일 서러게이트 키 | 22/22 테이블. 복합 PK·자연키 0개 |
| **UUID 타입** | **`binary(16)`** (varchar(36) 미사용) | `user_table.uuid`, `admin_uuid`, `club_uuid`, `club_category_uuid`, `aplict_uuid`, `club_member_uuid`, `notice_uuid`, `leader_uuid`, `email_token_uuid`, `signup_uuid`, `clubmember_account_status_uuid`, `event_verification_table.user_uuid`/`club_uuid` 등 13개 컬럼 전부. 엔티티가 `java.util.UUID`를 그대로 쓰고 Hibernate 6가 16바이트로 매핑한 결과이며, `EventVerification`만 `columnDefinition = "BINARY(16)"`으로 명시 |
| **UUID 용도** | 내부 조인은 `bigint` PK, 외부 API는 UUID (이중 식별자) | 거의 모든 테이블이 `~_id`(내부) + `~_uuid`(외부)를 함께 갖는다. UUID가 없는 예외는 `profile_table`, `club_info_table`, `club_info_photo_table`, `club_main_photo_table`, `club_hashtag_table`, `club_category_mapping_table`, `floor_photo_table`, `notice_photo_table`, `club_membertemp_table`, `auth_token_table`, `withdrawal_token`. 반대로 `auth_token_table`/`withdrawal_token`의 FK는 UUID 컬럼을 직접 참조한다 |
| **네이밍** | 테이블은 `~_table` 접미사, 컬럼은 snake_case + 테이블 접두 | 예외 3종: `withdrawal_token`(접미사 없음), `user_table.uuid`(접두 없음), `floor_photo_s3_key`(다른 테이블은 `~_s3key`). `club_info_table`/`club_info_photo_table`은 엔티티명(`ClubIntro`/`ClubIntroPhoto`)과 이름이 다르다 |
| **문자열 길이** | 4 / 8 / 10 / 11 / 20 / 30 / 200 / 255 / 3000 의 9종만 사용 | 4=동아리방, 8=학번, 10=동아리명·해시태그, 11=휴대폰, 20=로그인 계정·학과·카테고리명, 30=이름·이메일, 200=공지 제목, 255=BCrypt 해시·S3 키·파일명·URL·FCM 토큰, 3000=소개/모집글/공지 본문 |
| **길이 예외** | `leader_table.leader_account varchar(255)` | `user_account`·`admin_account`는 20. 엔티티에 `length` 미지정이라 JPA 기본값이 들어간 결과이며, 계정 ID 3종 중 유일한 불일치 |
| **긴 텍스트** | `TEXT` 미사용, `varchar(3000)` | 소개/모집글/공지 본문 모두 3000자가 하드 리밋 |
| **불리언** | **`bit(1)`** | `aplict_checked`(유일하게 `DEFAULT b'0'`), `is_verified`, `is_agreed_terms`, `verified`. `tinyint`는 생성 컬럼 `active_flag` 한 곳뿐 |
| **enum** | **MySQL 네이티브 `enum` + `@Enumerated(STRING)`** | `role`(3개 테이블), `aplict_status('FAIL','PASS','WAIT')`, `department`(6값), `club_info_recruitment_status('CLOSE','OPEN')`, `member_type('NONMEMBER','REGULARMEMBER')`, `floor_photo_floor('B1','F1','F2')` — 총 8개 컬럼. DDL 값 목록은 알파벳순이라 자바 선언 순서와 다르지만 문자열 저장이라 무관하다. 대신 값 추가에는 `ALTER TABLE`이 필요하다 |
| **시각** | **`datetime(6)`** (마이크로초 6자리), 타임존 없음 | `user_created_at/updated_at`, `profile_created_at/updated_at`, `fcm_token_updated_at`, `aplict_submitted_at`, `aplict_delete_date`, `expiration_time`, `club_member_temp_expiry_date`, `notice_created_at`, `verified_at`. `timestamp` 타입과 `DEFAULT CURRENT_TIMESTAMP`는 0건 — 시각은 전부 `@PrePersist`/`@PreUpdate`가 채운다 |
| **감사 컬럼 편차** | 영역별로 크게 다름 | `user_table`·`profile_table`은 created/updated 둘 다, `notice_table`은 created만, 동아리 9개 테이블과 `admin_table`은 아예 없음 |
| **DEFAULT** | 거의 없음 | 컬럼 기본값은 `aplict_checked DEFAULT b'0'` 하나뿐. 나머지는 전부 엔티티 `@Builder.Default`(`WAIT`, `CLOSE`, `REGULARMEMBER` 등)라 JPA 우회 INSERT는 값을 명시해야 한다 |
| **생성 컬럼** | 1개 | `aplict_table.active_flag`(STORED) — 조건부 UNIQUE 에뮬레이션 전용 |
| **엔진/문자셋** | 22개 전부 `InnoDB`, 21개 `utf8mb4_0900_ai_ci` | `event_verification_table`만 테이블 collation이 `utf8mb4_unicode_ci`이고 `user_account`·`email`에는 컬럼 단위로 또 명시되어 있다. 다른 테이블의 동일 성격 컬럼과 직접 문자열 조인 시 `Illegal mix of collations` 가 발생할 수 있는 구성이다(현재는 FK도 조인도 없어 드러나지 않음) |
| **NULL 성향** | 대체로 엄격 | NULL 허용은 사진 파일명/S3 키, 소개/모집글, `club_insta`, `google_form_url`, `aplict_delete_date`, `fcm_token(_updated_at)`, `leader_table.club_id`, `profile_table.user_id`, 토큰 2종의 `user_uuid`, `event_verification_table` 사본 4개뿐. 미입력은 NULL 대신 빈 문자열 `""`로 채우는 경로도 있다(동아리 생성 시) |

---

## 10. 데이터 프로파일

### 10-1. 행수 (총 1,961행)

| 테이블 | 행수 | AUTO_INCREMENT | 해석 |
|---|---:|---:|---|
| `user_table` | 508 | 5677 | 물리 삭제로 PK 대량 소비 |
| `profile_table` | 505 | 7763 | user보다 3행 적음 |
| `club_members_table` | 353 | 7729 | 최다 관계 테이블 |
| `event_verification_table` | 210 | 211 | append-only, 삭제 이력 없음 |
| `club_info_photo_table` | 155 | 361 | 31 × 5 슬롯 |
| `club_category_mapping_table` | 40 | 54 | |
| `club_category_table` | 33 | 61 | |
| `club_table` | 31 | 74 | |
| `club_info_table` | 31 | 73 | club과 1:1 완전 충족 |
| `club_main_photo_table` | 31 | 166 | club과 1:1 완전 충족 |
| `leader_table` | 31 | 73 | club과 1:1 완전 충족 |
| `club_hashtag_table` | 10 | 11 | |
| `aplict_table` | 9 | 442 | 4일 만료 삭제 정책의 결과 |
| `notice_table` | 6 | 24 | |
| `floor_photo_table` | 3 | 10 | B1/F1/F2 각 1 (갱신이 delete-then-insert) |
| `auth_token_table` | 3 | 50 | 미완료 요청 잔여물 |
| `admin_table` | 2 | 3 | 시딩 2건 그대로 |
| `club_member_accountstatus_table` | **0** | 55022 | |
| `club_membertemp_table` | **0** | 55020 | |
| `email_token_table` | **0** | 1055 | |
| `notice_photo_table` | **0** | 15 | |
| `withdrawal_token` | **0** | 53 | |

### 10-2. 빈 테이블 5개의 의미

미사용 기능이 아니라 대부분 **수명이 짧은(TTL) 테이블**이며, AUTO_INCREMENT가 과거 처리량을 보여준다.

- `email_token_table`(AI=1055): 5분 만료 + 1시간 유예 후 스케줄러 정리. 누적 1,054건 발급. **정상적으로 비어 있는 상태.**
- `withdrawal_token`(AI=53): 검증 후 삭제되는 1회성. 누적 52건.
- `club_membertemp_table`(AI=55020) / `club_member_accountstatus_table`(AI=55022): 두 값이 거의 같아 **항상 1:1에 가깝게 함께 생성**되었음을 보여준다. 5만 건 이상 처리 후 만료 기준으로 전량 정리됨.
- `notice_photo_table`(AI=15): 공지 6건은 남아 있으나 첨부는 0건 — 현재는 텍스트 전용 공지만 운영 중.
- (참고) `auth_token_table` 3행은 검증을 마치지 않고 이탈한 요청의 잔여물이며, 정리 주체가 없어 계속 남는다.

### 10-3. 상태값 분포

| 대상 | 분포 |
|---|---|
| `aplict_table.aplict_status` (9행) | `WAIT` 8 (88.9%) · `FAIL` 1 (11.1%) · `PASS` **0** |
| `aplict_table.aplict_checked` (9행) | `false` 9 (100%) — 전 행이 `active_flag=1`이라 `uk_aplict_active`가 9행 전부에 적용 중 |
| `aplict_table.aplict_delete_date` | 9행 전부 NULL (삭제 예약 없음) |
| `profile_table.member_type` (505행) | `REGULARMEMBER` 502 (99.4%) · `NONMEMBER` 3 (0.6%) |
| `club_info_table.club_info_recruitment_status` (31행) | `OPEN` 20 (64.5%) · `CLOSE` 11 (35.5%) |
| `user_table.role` (508행) | `USER` 476 (93.7%) · `LEADER` 30 (5.9%) · `ADMIN` 2 (0.4%) |
| `leader_table.role` (31행) | `LEADER` 31 (100%) |
| `admin_table.role` (2행) | `ADMIN` 2 (100%) |
| `club_table.department` (31행) | `SPORT` 7 · `ACADEMIC` 6 · `ART` 6 · `SHOW` 6 · `RELIGION` 3 · `VOLUNTEER` 3 |
| `floor_photo_table.floor_photo_floor` (3행) | `B1` 1 · `F1` 1 · `F2` 1 |
| `event_verification_table.verified` (210행) | `true` 210 (100%) — `false`가 저장되는 경로 없음 |
| `leader_table.is_agreed_terms` (31행) | `true` 28 · `false` 3 |

### 10-4. 관계 밀도 · 컬럼 충전율

- **user ↔ profile**: 508 : 505. 프로필 없는 3명은 관리자 2 + 회장 1(role 기준). `profile_table.user_id`는 NULL 허용이지만 실제 NULL 0건.
- **club 1:1 3종**: 31개 동아리 전부가 `club_info_table`·`club_main_photo_table`·`leader_table` 행을 정확히 1개씩 보유(누락 0). `leader_table.club_id` 실제 NULL 0건.
- **club_members**: 353행. 회원이 있는 동아리 21개 / **회원 0명인 동아리 10개**. 동아리당 최소 1 · 중앙값 19 · 최대 43 · 평균 16.8명. 소속 프로필 341명 중 11명이 2개 이상(최대 3개) 소속. 어떤 동아리에도 속하지 않은 프로필 164명(32.5%).
- **club_info_photo**: 155 = 31 × 5로 `photo_order` 1~5가 각각 31행씩 완벽히 균일 — 5슬롯 선생성 규칙이 데이터에 그대로 드러난다.
- **카테고리**: 매핑 40행 / 카테고리를 가진 동아리 20개(평균 2.0), **카테고리가 하나도 없는 동아리 11개**. 마스터 33개 중 **10개는 한 번도 매핑된 적 없음**. 모집중(OPEN) 20개 중 8개가 카테고리 미매핑이라 카테고리 필터 검색에서 노출되지 않는다.
- **해시태그**: 10행이 6개 동아리에만 존재(나머지 25개는 0개).
- **FCM 토큰** (505행 기준): `fcm_token` NOT NULL은 13행(2.6%)뿐. 조합별로 (토큰 NULL, 갱신시각 NOT NULL) 459 · (둘 다 NULL) 33 · (둘 다 NOT NULL) 13 — 만료 정리가 토큰만 비우고 시각은 남기는 구조를 그대로 보여준다.
- **선택 컬럼 충전율**: `club_insta` 31/31, `club_info` 31/31, `google_form_url` 30/31, `club_recruitment` 25/31. S3 키는 `club_info_photo` 155/155, `club_main_photo` 31/31 모두 NULL 0건.
- **공지**: 6건이 관리자 2명 중 한 명(5건)에 집중.
- **이벤트 인증**: 210행이 17개 동아리에 분포(동아리당 1~29건, 상위 2개가 각 29건). 사용자당 1건 제약을 감안하면 508명 중 약 41%가 인증을 마쳤다.
- **길이 여유**: `club_recruitment` 최대 약 2,979자 · `club_info` 약 2,445자로 **`varchar(3000)` 한계에 근접**. `notice_content` 약 1,941자. `google_form_url` 약 184자, `club_insta` 약 70자, `fcm_token` 약 142자(모두 255 이내).

### 10-5. 정합성 검증 결과

**FK가 걸린 18개 관계 — 고아 레코드 0건.** (`aplict→club/profile`, `club_members→club/profile`, `profile→user`, `club_info→club`, `club_info_photo→club_info`, `club_main_photo→club`, `club_category_mapping→club/club_category`, `club_hashtag→club`, `leader→club`, `notice→admin`, `auth_token→user.uuid` 전수 검사)

**FK가 없는 참조 — 고아 2건 발견.** `event_verification_table` 210행 중

| 검사 | 결과 |
|---|---|
| `user_uuid` → `user_table.uuid` | **1행 고아** |
| `user_id` → `user_table.user_id` | **1행 고아** (위와 동일 행) |
| `profile_id` → `profile_table.profile_id` | **1행 고아** (위와 동일 행) |
| `club_uuid` → `club_table.club_uuid` | **1행 고아** (위와 다른 행) |

즉 사용자가 사라진 행 1건, 동아리가 사라진 행 1건이다. FK가 있었다면 RESTRICT로 원본 삭제 자체가 막혔을 상황이며, FK가 없어 삭제가 그대로 통과한 결과다. 반면 **비정규화 사본은 100% 정합**하다: `user_id`↔`user_uuid` 불일치 0건, `user_account`/`email` 원본과 불일치 0건, `profile_id`가 가리키는 프로필의 `user_id` 불일치 0건. 210행 중 206행은 (club, profile) 조합이 `club_members_table`에도 실재한다.

**UNIQUE가 없는 곳의 실제 중복 — 전부 0건.** `club_members_table(club_id, profile_id)` 353행 전부 고유, `event_verification_table(user_uuid, club_uuid)` 210행 전부 고유, `club_category_table.club_category_name` 33행 전부 고유, `leader_table.leader_uuid` 31행 전부 고유, `floor_photo_table.floor_photo_floor` 3행 = 3개 층, `aplict_table(club_id, profile_id)` 중복 0건(9행이 동아리 7개·프로필 6명에 분산). 애플리케이션 레벨 검사가 현재까지는 잘 동작하고 있으나 DB가 보장하지 않으므로 동시 요청 경로는 방어되지 않는다.

**계정 3종의 중복 저장.** `user_table`/`leader_table`/`admin_table`은 스키마상 아무 관계도 없지만 데이터는 겹친다. 관리자 2건은 **2건 모두** 동일 계정 문자열이 `user_table`에 `role=ADMIN`으로 존재하고, 회장 31건 중 **30건**이 `user_table`에 `role=LEADER`로 존재한다(`user_table.role=LEADER` 행 수도 정확히 30). 이를 잇는 FK도 UNIQUE도 없어 한쪽만 변경/삭제되면 조용히 어긋나며, 실제로 회장 1명은 이미 `user_table` 쪽이 없는 상태다.

---

## 11. 설계 관찰

### 강점

1. **키 설계의 일관성** — 22/22 테이블이 `bigint` 서러게이트 PK + `binary(16)` 외부 UUID라는 이중 식별자 관례를 지킨다. 내부 조인은 좁은 정수 키로, 외부 API는 추측 불가능한 UUID로 처리되어 순차 ID 노출 위험이 없다. 복합 PK·자연키가 없어 리팩터링 여지도 넓다.
2. **관계 형태가 스키마에 정확히 드러난다** — 1:1은 자식의 `club_id`/`user_id`에 UNIQUE로, N:M은 별도 조인 테이블로 표현되어 있고, 순환 참조 없이 허브 2개(`club_table`, `user_table`) 중심의 얕은 트리다. FK가 걸린 18개 관계에 고아 레코드가 0건인 것도 이 구조가 실제로 지켜지고 있음을 보여준다.
3. **`uk_aplict_active`는 이 스키마에서 가장 정교한 부분** — MySQL에 없는 조건부 UNIQUE를 STORED 생성 컬럼으로 구현해, "미처리 지원서는 1건, 과거 이력은 다건"이라는 도메인 규칙을 DB 레벨에서 정확히 표현한다. 애플리케이션의 선조회-후삽입 race를 DB가 최종 방어하는 유일한 지점이기도 하다.
4. **TTL 기반 수명 관리** — 소프트 딜리트 없이 만료 시각 컬럼(`aplict_delete_date` 4일, `club_member_temp_expiry_date` 7일, `expiration_time` 5분) + 스케줄러 물리 삭제 조합으로 임시 데이터가 누적되지 않는다. 5개 테이블이 0행인 것은 이 정책이 정상 동작한 결과다.
5. **인증 코드·비밀번호의 저장 분리** — 비밀번호는 3개 계정 테이블 모두 BCrypt 해시 `varchar(255)`이며 평문 저장 경로가 없다.

### 약점 (심각도 순)

1. **`event_verification_table`이 구조적 공백의 집합소** — FK 4개가 전부 없고, 유일한 UNIQUE는 이름(`UK_event_user_club`)과 달리 `user_uuid` 단일 컬럼이라 **애플리케이션이 가정하는 (사용자, 동아리) 입도보다 DB 제약이 좁다**. 사용자가 다른 동아리로 두 번째 인증을 시도하면 `Duplicate entry`로 실패한다. FK 부재로 **이미 고아 2건이 발생**했고, 테이블 collation만 `utf8mb4_unicode_ci`라 다른 테이블과의 문자열 비교에 잠재적 충돌이 있다. 엔티티가 `nullable=false`로 선언한 컬럼도 DB에서는 NULL 허용이다.
2. **3개 계정 테이블이 스키마상 완전히 단절되어 있다** — 상속 매핑도 공통 계정 테이블도 없이 `user`/`leader`/`admin`이 각자 자격증명을 보관하는데, 실제로는 동일 인물이 두 테이블에 동시에 존재한다(관리자 2/2, 회장 30/31). 이를 잇는 FK도 UNIQUE도 없어 한쪽만 변경·삭제되면 조용히 어긋나며 이미 1건이 어긋나 있다. `leader_account varchar(255)` vs `user_account varchar(20)`의 길이 불일치까지 겹친다.
3. **유일성 보장이 애플리케이션에 위임된 지점이 많다** — `club_members_table(club_id, profile_id)`, `club_category_mapping_table(club_id, club_category_id)`, `floor_photo_table(floor_photo_floor)`, `club_info_photo_table(club_info_id, photo_order)`, `club_category_table(club_category_name)`, `leader_table(leader_uuid)`. 특히 `findByFloor`·슬롯 단건 조회처럼 `Optional`을 반환하는 쿼리는 중복이 생기는 순간 조회 자체가 예외로 실패한다. 현재 데이터에 중복은 0건이지만 동시 요청 경로는 무방비다.
4. **인덱스 투자가 한 테이블에 몰려 있다** — 9행짜리 `aplict_table`에 세컨더리 인덱스 7개(그중 2개는 중복·저효용)가 있는 반면, 매 요청마다 조회되는 `leader_uuid`, 스케줄러가 매일/매시 스캔하는 `fcm_token_updated_at`·`expiration_time`·`club_member_temp_expiry_date`, 일괄 등록 시 문자열 매칭에 쓰이는 `profile_table` 3컬럼은 전부 무인덱스다. 인덱스가 0개인 테이블이 2개(`club_membertemp_table`, `floor_photo_table`) 있다.
5. **삭제 안전성이 전적으로 코드 순서에 달려 있다** — FK 18개 전부 RESTRICT이고 CASCADE가 0개라, `deleteClubAndDependencies()`의 9단계 삭제 순서나 탈퇴 플로우의 토큰→프로필→사용자 순서가 한 곳이라도 어긋나면 런타임 제약 위반이 된다. 새 자식 테이블을 추가할 때마다 삭제 코드를 함께 고쳐야 하는 결합이 있다.
6. **기본값과 감사 이력이 스키마가 아닌 코드에 있다** — DB DEFAULT는 `aplict_checked` 하나뿐이고 나머지 기본값(`WAIT`, `CLOSE`, `REGULARMEMBER`, `false`)은 전부 엔티티 `@Builder.Default`다. 생성/수정 시각도 `@PrePersist`가 채우며, 동아리 도메인 9개 테이블과 `admin_table`에는 시각 컬럼 자체가 없다. JPA를 우회한 INSERT/UPDATE는 기본값도 이력도 얻지 못한다.
7. **본문 컬럼이 `varchar(3000)` 한계에 근접** — 실제 데이터에 2,979자·2,445자 본문이 있어 여유가 20자 남짓인 동아리가 존재한다. `TEXT`가 아니라 길이 확장에 `ALTER TABLE`이 필요하며, MySQL enum 컬럼 8개도 값 추가 시 동일하게 DDL 변경을 요구한다.
8. **인증코드 평문 저장과 만료 컬럼 부재** — `auth_token_table`/`withdrawal_token`의 4자리 코드는 해시 없이 저장되고 만료 시각 컬럼이 없다. 수명이 "검증 후 삭제"라는 상태 전이에만 의존해, 이탈한 요청은 정리 주체 없이 무기한 잔존한다(현재 3행). 두 테이블은 컬럼·제약·로직까지 사실상 복제된 쌍둥이다.
9. **비정규화 흔적** — `club_table.leader_name`/`leader_hp`가 `leader_table`과 동기화 보장 없이 공존하고, 해시태그는 마스터 없이 문자열을 중복 저장하며, `profile_table.major`는 코드 참조가 아닌 자유 입력이라 표기 흔들림을 막을 수단이 없다. 사진 테이블 2종은 "행 선생성 후 UPDATE" 방식이라 **행 존재가 사진 존재를 의미하지 않는다**(155행 = 155장이 아님).
