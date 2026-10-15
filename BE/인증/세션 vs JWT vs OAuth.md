# 세션 vs JWT vs OAuth
----

로그인을 설계할 때 이 셋을 나란히 놓고 "뭐가 제일 좋냐"고 묻는 글이 많은데, **셋은 같은 층위가 아니다.**

```
[1단계] 인증 — 이 사용자가 누구인지 확인한다
        → Email + Password 직접 확인  또는  OAuth/OIDC로 위임 (Google, Apple, Kakao)

[2단계] 세션 유지 — 확인된 신원을 요청마다 어떻게 이어갈 것인가
        → Session (서버 상태)  또는  JWT (무상태 토큰)
```

- **OAuth 2.0** 은 원래 *인가(authorization)* 프로토콜. "이 앱이 당신의 리소스에 접근해도 되나요"를 다룬다.
- **OIDC (OpenID Connect)** 는 그 위에 *인증* 을 얹은 표준. `id_token`(JWT)으로 "이 사람이 누구인지"를 알려준다. 소셜 로그인에서 실제로 쓰는 건 대부분 이쪽.
- **Session / JWT** 는 1단계 결과를 어떻게 들고 다닐지에 대한 답.

즉 진짜 질문은 이렇게 바뀐다.
> 소셜 로그인으로 신원을 받은 뒤, **우리 서비스 세션을 세션으로 굴릴 것인가 JWT로 굴릴 것인가**

## 1. Session 기반 (서버 상태 + 쿠키)

로그인 시 서버가 세션 레코드를 만들고(Redis/DB), 세션 ID만 httpOnly 쿠키로 내려준다. 쿠키에는 의미 있는 정보가 없다.

### 장점
- **즉시 무효화 가능** — 강제 로그아웃, 계정 정지, 권한 강등이 그 순간 반영됨
- **XSS에 강함** — `httpOnly` 쿠키는 JS로 읽을 수 없음
- 사용자 정보 변경이 즉시 반영됨
- 구현이 단순. 만료/갱신 로직을 직접 설계할 일이 거의 없음
- 접속 중인 기기 목록 조회, 개별 세션 종료가 자연스럽게 나옴

### 단점
- **상태 저장소가 필요** — Redis가 죽으면 전원 로그아웃 (SPOF)
- 수평 확장 시 세션 저장소 공유 필요
- **CSRF 방어 필수** — 쿠키는 자동 전송되므로 `SameSite` + CSRF 토큰
- **모바일 앱·서드파티에서 불편** — 쿠키 자동관리가 없는 환경에서는 직접 다뤄야 함

## 2. JWT 기반 (무상태 토큰)

서명된 토큰 안에 사용자 ID·권한·만료가 들어있고, 서버는 서명만 검증한다. 저장소 조회가 없다.

### 장점
- **무상태** — 인스턴스 확장에 유리
- **클라이언트 종류를 가리지 않음** — `Authorization: Bearer` 하나로 웹·앱·서드파티 모두 동일
- 크로스 도메인/마이크로서비스에 유연
- 토큰에 권한을 담아 조회를 줄일 수 있음

### 단점
- **취소가 어렵다** — 발급된 JWT는 만료 전까지 유효. 블랙리스트를 두면 결국 상태가 생겨 무상태의 이점이 반쯤 사라짐
- **탈취 시 피해가 큼** — localStorage에 두면 XSS 한 방에 털림
- **권한 변경이 지연됨** — 강등해도 기존 토큰 만료까지 옛 권한 유지
- **Refresh Token 설계가 진짜 난이도** — 회전, 재사용 탐지, 동시 갱신 경합. 사고가 가장 많이 나는 지점
- `alg: none`, HS/RS 혼용 같은 고전적 취약점 주의

## 3. OAuth 2.0 / OIDC (소셜 로그인)

### 장점
- **비밀번호를 저장하지 않음** — 유출 리스크와 규제 부담이 크게 감소
- 가입 전환율이 높음 (국내에서 카카오 로그인은 사실상 기본값)
- 이메일 인증·비밀번호 재설정·무차별 대입 방어를 제공자가 대신 처리
- 제공자 측 2FA를 그대로 활용

### 단점
- **외부 의존성** — 제공자 장애 = 해당 경로 로그인 불가
- 제공자별 구현 편차가 큼
- **계정 병합 문제** — Google로 가입했다가 Kakao로 로그인하면? 정책을 안 정하면 중복 계정이 쌓임
- **이것만으로는 세션이 완성되지 않음** — 그 다음은 어차피 Session이나 JWT

## 4. 비교표

| 항목 | Session | JWT | OAuth/OIDC |
| ----------- | ----------- | ----------- | ----------- |
| 계층 | 세션 유지 | 세션 유지 | 신원 확인 |
| 서버 상태 | 필요 (Redis) | 불필요 | 해당 없음 |
| 즉시 로그아웃 | 쉬움 | 어려움 (블랙리스트 필요) | — |
| 웹([[싱글 페이지 애플리케이션(SPA)]]) 적합도 | 매우 좋음 | 저장 위치 고민 | — |
| 모바일 앱 적합도 | 쿠키 관리 번거로움 | 매우 좋음 | 좋음 |
| 외부 연동 | 부적합 | 적합 | 적합 |
| XSS 내성 | 높음 (httpOnly) | 낮음 (저장 방식에 따라) | — |
| CSRF 방어 | 필수 | 헤더 사용 시 불필요 | — |
| 권한 변경 반영 | 즉시 | 토큰 만료까지 지연 | — |
| 구현 난이도 | 낮음 | **높음** (refresh 설계) | 중간 |

## 5. 결론: 하이브리드

**소셜 제공자의 토큰을 그대로 API 인증에 쓰면 안 된다.** 반드시 자체 자격증명으로 교환한다.

```
Google / Apple / Kakao (OIDC)  ─┐
                                ├─→  [자체 인증 서버]  ─→  자체 세션/토큰 발급
Email + Password  ──────────────┘         (신원 통합)
```

이렇게 해야 제공자를 추가/교체해도 API 레이어가 흔들리지 않는다.

| 클라이언트 | Access | Refresh |
| ----------- | ----------- | ----------- |
| 웹 ([[싱글 페이지 애플리케이션(SPA)]]) | httpOnly 쿠키 | httpOnly 쿠키 + CSRF 토큰 |
| 모바일 앱 | 메모리 내 Bearer | OS 보안 저장소 (Keychain / Keystore) |
| 외부 연동 | Client Credentials 토큰 | — |

### 토큰 정책 권장값
- **Access Token** — JWT, 15분 내외. 짧게 유지해 탈취 피해 시간을 줄임
- **Refresh Token** — **불투명(opaque) 랜덤 문자열을 DB에 저장.** JWT로 만들지 말 것. DB에 있어야 취소가 된다
	- **회전(Rotation)** — 사용할 때마다 새로 발급하고 이전 것은 폐기
	- **재사용 탐지** — 이미 쓴 refresh가 다시 오면 탈취 정황 → 해당 사용자 **토큰 패밀리 전체 무효화**
	- 유효기간 14~30일 + 절대 만료 별도 설정
- **권한 변경 시** — `token_version` 컬럼을 JWT에 담아, 강등/퇴사 시 값을 올려 기존 토큰 일괄 무효화. **무상태 JWT의 최대 약점을 저렴하게 메우는 방법**

### 토큰 패밀리를 스키마로 표현하면

```
Identity_RefreshTokens
  Id
  UserId
  TokenHash          ← 원문이 아니라 해시를 저장 (unique)
  FamilyId           ← 회전 체인을 묶는 키. 재사용 탐지 시 이 단위로 폐기
  ClientType         ← web / app / external — 클라이언트별 정책 분기
  ExpiresOn
  RevokedOn
  RevokedReason      ← 'rotated' / 'reuse_detected' / 'logout'
  ReplacedByTokenId  ← 회전 체인 추적
  UserAgent, IpAddress
```

`FamilyId` + `ReplacedByTokenId` 조합이 회전과 재사용 탐지의 핵심이다. 토큰 하나가 재사용되면 같은 `FamilyId` 전부를 죽인다.

## 6. 제공자별 함정

### Apple

> 2026-09-01 확인 — [client_secret 생성](https://developer.apple.com/documentation/accountorganizationaldatasharing/creating-a-client-secret) · [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) · [Sign in with Apple 도메인 변경 공지](https://developer.apple.com/news/?id=1ptvdtcm)

- **client_secret이 문자열이 아니라 ES256으로 서명한 JWT.** `exp` 최댓값이 **15777000초(≈6개월)** 로 못박혀 있어 **주기적 재발급 자동화가 필요**. 수동으로 두면 반년 뒤 조용히 로그인이 깨진다
- 사용자가 **이메일 가리기(Private Relay)** 를 선택할 수 있음 → 릴레이 주소가 온다. **이메일을 계정 병합의 유일한 키로 쓰면 안 되는 이유**
	- ⚠️ **도메인이 하나가 아니다.** 기존 `@privaterelay.appleid.com` 에 더해 신규 Sign in with Apple 주소는 **`@private.icloud.com`** 으로 발급된다(2026년 중 전환, 기존 주소는 계속 동작). 애플이 명시적으로 "허용목록에 새 도메인을 추가하라"고 공지했다
	- 그래서 **도메인 문자열로 릴레이 여부를 판별하는 코드는 애초에 잘못된 설계다.** 도메인은 애플 사정으로 바뀐다. `id_token`의 **`is_private_email` 클레임**을 보는 게 맞다
- **이름(fullName)은 최초 인가 응답 1회만 전달.** 그때 저장하지 않으면 사용자가 Apple ID 설정에서 앱 연결을 지우고 재인가하기 전엔 다시 못 받는다
	- 이메일은 사정이 다르다 — **이후 로그인에서도 `id_token`의 `email` 클레임으로 계속 온다.** "이름·이메일 둘 다 1회"로 알려진 경우가 많은데, **1회짜리는 이름 쪽**이다
- ⚠️ **가이드라인 4.8은 더 이상 Sign in with Apple을 콕 집어 요구하지 않는다.** 현행 조문(4.8 Login Services)은 서드파티/소셜 로그인으로 주 계정을 만든다면 **아래 3조건을 만족하는 로그인 수단을 동등한 선택지로 하나 더** 제공하라고만 한다
	1. 수집 데이터를 **이름과 이메일로 제한**
	2. 가입 과정에서 **이메일을 비공개로 유지**할 수 있게 허용
	3. 동의 없이 **광고 목적으로** 앱 내 상호작용을 수집하지 않음
	- Sign in with Apple은 이 조건을 만족하는 **한 가지 선택지**일 뿐, 유일한 답이 아니다. 게다가 **자사 계정 시스템만 쓰는 앱, 교육·기업용 앱** 등은 아예 면제 대상
	- 실무 판단 — 그래도 iOS 비중이 크면 Sign in with Apple을 넣는 게 심사 논쟁을 피하는 가장 싼 길이다. 다만 **"규정이라서 강제"가 아니라 "가장 안전한 선택지라서"** 라는 걸 알고 넣어야 한다

### Kakao

> 2026-09-01 확인 — [카카오 로그인 문서](https://developers.kakao.com/docs/latest/ko/kakaologin/common)

- OIDC를 **지원**. `id_token`을 받아 다른 제공자와 같은 파이프라인에 태울 수 있음. 단 **콘솔에서 OpenID Connect를 켜야** 발급된다
- **이메일이 필수 제공 항목이 아니다.** `account_email` 동의항목은 **비즈니스 앱 전환**을 해야 설정할 수 있고, 그 뒤에도 "선택 동의"로 둘 수 있어 **끝내 못 받는 계정이 존재한다.** 이메일 없는 계정을 허용하는 스키마가 필요
	- 받더라도 카카오 문서가 **인증 여부를 직접 확인하라**고 명시한다. 미인증 이메일은 발송이 도달하지 않을 수 있다
- `@fastify/oauth2`에 **Kakao 프리셋이 없다.** `authorizeHost`/`tokenHost`를 직접 설정해야 함 (v8.3.0 패키지 소스에서 확인 — 아래 표 참고)

### Google

> 2026-09-01 확인 — [OpenID Connect 문서](https://developers.google.com/identity/openid-connect/openid-connect)

- 가장 표준적. 문서가 좋음
- `email_verified` 클레임을 **반드시 확인.** 미검증 이메일로 계정 병합을 하면 계정 탈취 경로가 된다
- 더 근본적으로, 구글 문서는 **"`email`을 사용자 고유 식별자로 쓰지 말고 항상 `sub`을 쓰라"**고 못박는다. 이메일 값은 이 계정에 유일하지 않을 수 있고 시간이 지나면 바뀔 수 있기 때문. 아래 `auth_identities(provider, provider_user_id)` 의 `provider_user_id` 에 들어가야 하는 게 바로 이 **`sub`** 이다

### 공통 — 계정 병합 정책을 먼저 정할 것
1. 검증된 이메일이 일치하면 자동 병합할 것인가, 명시적 연결 절차를 요구할 것인가 → **보안상 후자가 안전**
2. Apple Private Relay처럼 이메일을 못 믿는 경우의 처리
3. 기존 계정에 새 소셜을 추가 연결하는 UI 흐름

4개 로그인 경로가 **하나의 내부 사용자 레코드로 수렴**해야 한다.
→ `users` 1 : N `auth_identities(provider, provider_user_id)` 구조

## 7. 보안 체크리스트

- [ ] Refresh Token은 DB 저장 + 회전 + 재사용 탐지
- [ ] Access Token은 짧게(15분 내외), 페이로드에 민감정보 금지
- [ ] JWT 알고리즘 명시적 고정 (`alg` 검증, `none` 차단)
- [ ] 쿠키는 `httpOnly` + `Secure` + 적절한 `SameSite`
- [ ] 쿠키 세션 사용 시 CSRF 토큰
- [ ] OAuth 요청에 **state 파라미터 + PKCE**
- [ ] `redirect_uri` 화이트리스트 고정 (오픈 리다이렉트 방지). 인가 코드는 [[쿼리스트링(Query String)]]으로 오므로 로그에 남는다는 점도 유의
- [ ] 로그인/재설정 엔드포인트 rate limit
- [ ] 비밀번호는 **argon2id** — OWASP 1순위이며 최소 `m=19456(19MiB), t=2, p=1`. bcrypt는 OWASP가 *레거시 시스템용*으로 분류하며 최소 work factor는 **10**이다(여기서는 여유를 둬 12를 쓴다). bcrypt를 쓴다면 **입력 72바이트 제한을 애플리케이션에서 강제**할 것 — 넘는 부분이 조용히 잘린다
- [ ] 소셜 `id_token`은 서명·`iss`·`aud`·`exp` 모두 검증
- [ ] Apple client_secret 자동 재발급 스케줄 등록

## 8. Fastify 플러그인 매핑

> Fastify v5.12.1 기준. 아래 버전은 2026-09-01 npm 레지스트리에서 확인했다. 7개 전부 실재하고 활발히 유지된다(모두 2026-07~08 사이 릴리스).

| 용도 | 패키지 | 메이저 |
| ----------- | ----------- | ----------- |
| 쿠키 처리 | `@fastify/cookie` | 11 |
| 서버 세션 | `@fastify/session` + 외부 스토어 | 11 |
| JWT | `@fastify/jwt` | 10 |
| 소셜 로그인 | `@fastify/oauth2` | 8 |
| 인증 훅 조합 | `@fastify/auth` | 5 |
| CSRF | `@fastify/csrf-protection` | 8 |
| 요청 제한 | `@fastify/rate-limit` | 11 |

### 표에 안 드러나는 함정 두 가지

**`@fastify/session`의 기본 스토어는 프로덕션 금지.** 공식 README가 "The default store should not be used in a production environment because it will leak memory" 라고 **메모리 누수까지 명시**한다. express-session 호환 스토어를 붙이면 된다.

- 그런데 **`connect-redis` v10을 그냥 붙이면 걸린다.** peerDependencies에 `express-session >=1` 이 **optional이 아닌 필수**로 선언돼 있어서, Fastify만 쓰는데도 `express-session`을 설치해야 peer 경고가 사라진다. `redis >=5` 와 **Node 22 이상**도 요구한다
- 교훈 — 세션 스토어는 붙이기 전에 **peerDependencies부터 볼 것.** express 생태계 패키지를 Fastify에 끌어오면 이런 게 딸려온다

**`@fastify/oauth2` v8.3.0** — 패키지 소스에서 직접 확인한 사실

- PKCE 지원한다. `pkce: 'S256' | 'plain'` 두 값만 받고 그 외는 **플러그인 등록 시점에 에러**로 막는다. 제공자가 지원하면 항상 `S256`
- 내장 프리셋에 **`GOOGLE_CONFIGURATION` 과 `APPLE_CONFIGURATION` 이 둘 다 있다.** 전체 목록 — Apple / Discord / Epic Games / Facebook / GitHub / GitLab / LinkedIn / Microsoft / Spotify / Twitch / VATSIM / VKontakte / X / Yandex
- **국내 제공자는 목록에 하나도 없다.** Kakao·Naver는 호스트를 직접 써야 한다

## 9. 한 줄 정리

| 질문 | 답 |
| ----------- | ----------- |
| 세션 vs JWT 중 뭐가 좋나? | 클라이언트가 웹뿐이면 **세션**, 앱·외부연동이 있으면 **하이브리드** |
| OAuth는 대안인가? | 아님. **다른 층위**이며 소셜 로그인을 쓰는 이상 필수 |
| 가장 위험한 지점은? | **Refresh Token 설계**와 **계정 병합 정책** |
| 처음부터 다 만들어야 하나? | 아니오. 세션으로 시작해 앱 출시 시점에 토큰 레이어를 얹어도 됨 |

사용자가 테넌트에 종속될 때 생기는 별개의 문제는 [[멀티테넌시 데이터 모델]]에 정리했다. 스키마 구현은 [[TypeORM 엔티티 설계]].

#펫바디
