# 코드 서명(Code Signing)
----

**코드 서명**은 앱 바이너리에 개발자의 개인키로 디지털 서명을 붙여, OS가 **"누가 만들었는가"와 "만든 뒤 변조되지 않았는가"** 를 확인하게 하는 장치다.

원리는 HTTPS 인증서와 같은 공개키 암호다. 개발자가 개인키로 서명하고, OS는 공개키(인증서)로 검증한다. 한 바이트라도 바뀌면 서명이 깨진다.

> 기준: Apple Developer · Android Developers 공식 문서 (2026-10-15 확인)

## 1. iOS — 세 조각이 맞아야 설치된다

iOS는 **Apple이 승인한 서명이 없으면 아예 실행하지 않는다.** 그래서 서명이 단순히 "누가 만들었나" 표시가 아니라 **설치 허가증** 역할까지 한다.

```
인증서 (누가)          ── Apple이 발급. 개발자 개인키와 짝
  +
App ID (어떤 앱)       ── 번들 ID. 예: com.petbody.petvatar
  +
(기기 목록 — 개발·Ad Hoc만)
  ↓ 묶어서
프로비저닝 프로필       ── "이 사람이 서명한 이 앱을, 이 기기들에, 이 방식으로 설치해도 된다"
```

| 조각 | 무엇 | 비고 |
| ----------- | ----------- | ----------- |
| **인증서** | Development / Distribution 두 종류 | 개인키가 내 Mac 키체인(또는 EAS)에 있어야 쓸 수 있다 |
| **App ID** | 번들 ID + 사용 기능(푸시·App Groups 등) | 기능을 켜면 프로필을 다시 만들어야 함 |
| **프로비저닝 프로필** | 인증서 + App ID + (기기 목록) + 배포 방식 | 앱 안에 함께 들어간다 |

### 배포 방식별 프로필

| 방식 | 누가 설치하나 | 조건 |
| ----------- | ----------- | ----------- |
| Development | 개발자 기기 | 기기 UDID 등록 |
| **Ad Hoc** | 등록된 테스트 기기 | UDID 등록. **제품군별 연 100대**(멤버십 연도마다 재설정) |
| TestFlight | 초대한 테스터 | App Store Connect에 업로드 + (외부 테스터는) 베타 심사 |
| App Store | 누구나 | 정식 심사 |
| Enterprise | 사내 직원 무제한 | Apple Developer **Enterprise** Program 별도 가입 |

### 계정
* **Apple Developer Program: 연 99 USD** (지역별 현지 통화) — [출처](https://developer.apple.com/programs/enroll/)
* 무료 Apple 계정도 Xcode에서 **Personal Team**으로 내 기기에 설치는 가능. 단 프로필이 **발급 7일 후 만료**돼 다시 빌드·설치해야 하고, App ID 10개·기기 3대·기기당 앱 3개 한도 — [출처](https://developer.apple.com/support/compare-memberships/)
* 기기 100대 한도 출처: [Devices overview](https://developer.apple.com/help/account/devices/devices-overview/). **기기를 비활성화해도 한도가 돌아오지 않는다** — 멤버십 갱신 시점에 삭제해야 복구

### 왜 아이폰은 APK처럼 파일로 못 까나
Android는 "출처를 알 수 없는 앱 허용"을 켜면 **아무 서명이든** 설치된다(서명은 필수지만 자가 서명이면 충분). iOS는 **Apple이 발급한 인증서 + 그 기기가 포함된 프로필**이 있어야만 설치된다. 그래서 테스터에게 IPA 파일만 보내면 안 깔리고, UDID를 먼저 받아 등록한 뒤 Ad Hoc으로 다시 빌드하거나 TestFlight를 써야 한다. 이게 [[EAS Build]] 내부 배포에서 iOS만 UDID 등록이 필요한 이유다.

## 2. Android — 키스토어 하나가 앱의 정체성

Android는 **키스토어(`.jks` / `.keystore`)** 안의 키로 서명한다. Apple 같은 발급 기관이 없고 자가 서명이다. 대신 **같은 키로 서명된 APK만 기존 앱의 업데이트로 인정**한다. 키가 곧 "이 앱이 그 앱이다"라는 증명이다.

### Play App Signing — 키 두 개

```
개발자 ──(업로드 키로 서명)──▶ Google Play ──(앱 서명 키로 다시 서명)──▶ 사용자 기기
         업로드 키: 내가 보관        앱 서명 키: Google이 보관
```

| 키 | 누가 보관 | 잃어버리면 |
| ----------- | ----------- | ----------- |
| **업로드 키** | 개발자 | Play Console에서 **재설정 요청 가능** — 앱 서명 키는 Google에 있으니 업데이트를 계속 낼 수 있다 |
| **앱 서명 키** | Google | (Google이 보관하므로 개발자가 잃을 일 없음). 앱 수명 동안 **절대 바뀌지 않는다** |

* 2021년 8월 이후 만든 앱은 Google Play 배포에 **Play App Signing 필수**. AAB는 Play가 APK를 만들어 서명하는 구조라 설정이 먼저 돼 있어야 한다
* Play App Signing을 안 쓰는 옛 앱이 서명 키를 잃으면: *"Google은 키를 복구해 줄 수 없고, 원래 앱의 업데이트로 새 버전을 낼 수 없다."* 새 패키지명으로 새 앱을 내야 하고 기존 사용자·리뷰는 잃는다

> 출처: [Sign your app](https://developer.android.com/studio/publish/app-signing) (2026-10-15 확인)

## 3. iOS vs Android 비교

| | iOS | Android |
| ----------- | ----------- | ----------- |
| 발급 기관 | Apple | 없음 (자가 서명) |
| 서명의 의미 | 설치 허가 | 앱 정체성(업데이트 연속성) |
| 기기 제한 | 프로필에 등록된 기기만 (Ad Hoc) | 없음 |
| 유료 계정 | 연 99 USD (배포에 필수) | Play Console 등록비 (금액 미검증) |
| 가장 무서운 사고 | 인증서·프로필 만료로 빌드 실패 | **키 분실 → 업데이트 불가** |

## 4. EAS가 대신해 주는 것

[[EAS Build]]는 iOS 배포 인증서·프로비저닝 프로필·푸시 키, Android 키스토어를 **자동 생성해 서버에 보관**한다. 한 번 만들고 나면 팀원이 Apple Developer 팀 권한 없이도 빌드할 수 있다. `eas device:create` 로 UDID를 등록하면 Ad Hoc 프로필도 다시 만들어 준다. 출처: [Managed credentials](https://docs.expo.dev/app-signing/managed-credentials/)

## 5. 함정

* **키를 한 곳에만 두지 마라.** EAS에 맡겼더라도 `eas credentials` 로 Android 키스토어를 받아 별도 안전한 곳(비밀번호 관리자 등)에 백업
* 키스토어 비밀번호를 git에 올리지 말 것. CI에서는 비밀 환경변수로
* iOS 기능(푸시·App Groups 등)을 추가하면 App ID가 바뀌고 **프로필 재발급**이 필요하다. 위젯을 위한 App Group 추가가 대표적 → [[홈 화면 위젯]]
* 개인 계정 인증서로 서명하면 회사 앱이 개인에게 묶인다. 운영 앱은 처음부터 **조직 계정**으로 → [[EAS]] 4절
* 앱 업데이트를 OTA로 보낼 때도 서명 개념은 그대로 적용된다 — 업데이트 서버가 털리면 서명 없는 JS가 모든 기기로 간다 → [[OTA 업데이트]]

## 정리
* 코드 서명 = "누가 만들었고 변조되지 않았다"를 OS가 확인하는 공개키 서명
* iOS: 인증서 + App ID + 프로비저닝 프로필. Apple이 승인한 기기·방식으로만 설치된다
* Android: 키스토어 = 앱의 정체성. Play App Signing으로 업로드 키/앱 서명 키 분리 → 업로드 키는 잃어도 재설정 가능
* EAS가 대부분 자동 관리하지만 **백업은 직접**
