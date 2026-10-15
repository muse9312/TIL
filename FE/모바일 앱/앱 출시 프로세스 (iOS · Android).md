# 앱 출시 프로세스 (iOS · Android)
----

웹은 빌드해서 우리 서버에 올리면 끝이다. 앱은 **서명된 설치 파일**을 만들어 **애플·구글 심사**를 통과해야 사용자에게 간다. 이 노트는 계정 준비부터 업데이트까지의 **흐름**을 정리한 가이드이고, 각 단계의 개념은 개별 노트로 넘긴다.

> 정책·수치는 Apple Developer · App Store Connect Help · Play Console Help · Expo 공식 문서 기준 (2026-10-15 확인). 출처는 각 절 아래에 둔다. 스토어 정책은 자주 바뀌므로 실제 신청 직전에 다시 볼 것

## 1. 전체 흐름

```
계정 준비 ─▶ 개발 ─▶ 빌드 + 서명 ─▶ 내부 테스트 ─▶ 스토어 등록 정보 ─▶ 심사 ─▶ 출시 ─▶ 업데이트
(가장 오래 걸림)        (.ipa/.aab)    (TestFlight·        (스크린샷·         (애플·   (단계적)   (재심사 or OTA)
                                       테스트 트랙)         개인정보)          구글)
```

### 웹 vs 앱

| | 웹 | 앱 |
| ----------- | ----------- | ----------- |
| 결과물 | 정적 파일 · Docker 이미지 | iOS `.ipa` / Android `.aab`(스토어용) · `.apk`(직접 설치) |
| 서명 | 없음 | **필수** — [[코드 서명(Code Signing)]] |
| 배포처 | 우리 서버 | App Store · Google Play |
| 반영 | 즉시 | **심사 후**. JS만 바뀌면 [[OTA 업데이트]]로 심사 없이 |
| 롤백 | 이전 이미지로 되돌리면 끝 | 스토어에 "이전 버전으로 되돌리기"가 없다 → **수정한 새 버전을 다시 심사** |

롤백이 안 된다는 점이 앱 배포를 보수적으로 만든다. 그래서 **단계적 출시**로 일부 사용자에게 먼저 내보내고, 문제가 보이면 중단한 뒤 고친 빌드를 낸다. Google 공식 문서도 단계적 출시 중 문제가 생기면 *"수정된 번들로 새 릴리스를 만들어 내보내라"* 고 안내한다.

## 2. 계정 준비 — 일정의 최앞단

개발보다 먼저 시작해야 한다. 회사 명의 가입은 서류 검증 때문에 **수일~수주** 걸린다.

| | Apple Developer Program | Google Play Console |
| ----------- | ----------- | ----------- |
| 비용 | **연 99 USD** (지역별 현지 통화) | **1회 25 USD** |
| 회사 명의 | [[D-U-N-S 번호]] **필수**(정부기관 제외). 법인이어야 하고 DBA·상호·지점 불가. 조직 도메인의 공개 웹사이트·업무 이메일 필요 | [[D-U-N-S 번호]] **필수** — *"없으면 조직 계정을 만들 수 없다"*. Google Payments 프로필의 법인명·주소가 D&B 정보와 일치해야 함 |
| D-U-N-S 소요 | D&B 발급 최대 영업일 5일 + Apple 반영 최대 영업일 2일 ≈ **5~7 영업일**. 2주 넘으면 D&B 문의 | Google 문서는 **최대 30일**까지 걸릴 수 있다고 안내 (발급 자체는 무료) |
| 개인 명의 | 스토어 판매자명이 **본인 법적 이름**으로 노출 | 신분증·신용카드 본인 명의 확인 |
| 신규 개인 계정 제약 | 없음 | 2023-11-13 이후 만든 **개인** 계정은 프로덕션 전에 **비공개 테스트 12명 이상 · 14일 연속 옵트인** 필요 |

* D-U-N-S는 한 번 받으면 애플·구글 둘 다에 쓴다. **회사가 이미 번호를 갖고 있을 수 있으니** 새로 신청하기 전에 조회부터
* 12명/14일 요건은 공식 문서 제목부터 "신규 **개인** 개발자 계정" 대상이다. 조직 계정이 면제라는 건 여러 외부 자료가 말하지만 **공식 문서에서 면제 문구를 직접 찾지는 못했다(부분 검증)**. 어쨌든 회사 명의로 가면 이 문제를 피할 가능성이 크다
* **번들 ID(패키지명)는 한 번 출시하면 못 바꾼다.** 바꾸면 스토어 입장에선 다른 앱이다. 계정과 함께 일정 최앞단에서 확정한다

> 출처: [Apple — Enrollment](https://developer.apple.com/programs/enroll/) · [Apple — D-U-N-S](https://developer.apple.com/help/account/membership/D-U-N-S/) · [Play — 계정 등록](https://support.google.com/googleplay/android-developer/answer/6112435) · [Play — 조직 계정 D-U-N-S](https://support.google.com/googleplay/android-developer/answer/13628312) · [Play — 신규 개인 계정 테스트 요건](https://support.google.com/googleplay/android-developer/answer/14151465) (2026-10-15 확인)

## 3. iOS 8단계

| # | 단계 | 내용 |
| ----------- | ----------- | ----------- |
| 1 | **App ID 등록** | 번들 ID + 쓸 기능(capability) — 푸시, Sign in with Apple, App Groups(위젯과 데이터 공유, [[홈 화면 위젯]]) |
| 2 | **서명 준비** | 인증서(개발용/배포용) = 누가 만들었나, 프로비저닝 프로필 = 어떤 앱을 어떤 기기에 어떤 방식으로. 상세는 [[코드 서명(Code Signing)]] |
| 3 | **빌드** | **macOS + Xcode에서만** 가능. 결과물 `.ipa` |
| 4 | **업로드** | [[App Store Connect]]로. Xcode Organizer 또는 Transporter |
| 5 | **테스트** | 아래 표 |
| 6 | **스토어 정보** | 설명·스크린샷·개인정보처리방침 URL·App Privacy(수집 데이터 신고)·연령 등급 |
| 7 | **심사** | 반려되면 사유 확인 → 수정 → 재제출 |
| 8 | **출시** | 승인 즉시 / 예약 / 단계적(Phased Release) |

### 테스트 배포 방식

| 방식 | 인원 | 심사 | 비고 |
| ----------- | ----------- | ----------- | ----------- |
| [[TestFlight]] 내부 | **최대 100명** | **없음** | App Store Connect 팀 멤버(Admin·Developer 등 역할 보유자)만 |
| [[TestFlight]] 외부 | **최대 10,000명** | 그룹에 넣은 **첫 빌드는 베타 심사** | 이메일 초대 또는 공개 링크. 이후 빌드는 전체 심사가 생략될 수 있음 |
| Ad Hoc | 기기 UDID 등록 | 없음 | 제품군별 **연 100대**, 멤버십 연도마다 재설정 |

* TestFlight 빌드는 **업로드 후 90일**까지만 테스트 가능하다. 오래된 빌드로 테스트하던 테스터가 갑자기 못 연다면 이것
* 내부 테스터는 "앱스토어 커넥트 계정이 있는 우리 팀"이다. 외부 협력사·고객사에게 보여 주려면 외부 테스트로 가야 하고, 그 순간 베타 심사가 붙는다

### 심사 · 출시

* 심사 소요: Apple 공식 수치로 **50% 이상이 24시간 내, 90%가 48시간 내**. 메타데이터가 빠지면 더 늦어진다. 치명적 버그 수정·이벤트 일정이면 **긴급 심사(expedited review)** 요청 가능
* 단계적 출시(Phased Release)는 **7일간 1% → 2% → 5% → 10% → 20% → 50% → 100%**. 단 **자동 업데이트 사용자에게만** 적용되고, App Store에서 직접 업데이트 누르는 사람은 첫날부터 받는다. 일시정지는 합계 30일까지

> 출처: [TestFlight](https://developer.apple.com/testflight/) · [TestFlight overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview) · [App Review](https://developer.apple.com/distribute/app-review/) · [Phased release](https://developer.apple.com/help/app-store-connect/update-your-app/release-a-version-update-in-phases) · [Devices overview](https://developer.apple.com/help/account/devices/devices-overview/) (2026-10-15 확인)

## 4. Android 7단계

| # | 단계 | 내용 |
| ----------- | ----------- | ----------- |
| 1 | **앱 생성** | [[Google Play Console]]에서. 패키지명 = iOS 번들 ID에 해당 |
| 2 | **서명** | 업로드 키(`.jks`, 우리가 보관) + 앱 서명 키(구글이 보관) = [[Play App Signing]]. 업로드 키를 잃어도 재설정 요청 가능. **2021-08 이후 신규 앱은 필수** |
| 3 | **빌드** | macOS·Windows·Linux 어디서든 (JDK + Android SDK). 스토어용 `.aab`, 직접 설치용 `.apk` |
| 4 | **테스트** | APK 직접 공유, 또는 Play 테스트 트랙(아래 표) |
| 5 | **스토어 정보** | 설명·스크린샷·개인정보처리방침·**데이터 보안(Data safety)**·콘텐츠 등급·타깃 연령 |
| 6 | **심사** | 보통 수 시간~며칠. 공식 문서는 *"일부 계정은 **최대 7일, 예외적으로 그 이상**"* — 신규 계정·신규 앱이 여기 걸리기 쉽다 |
| 7 | **출시** | 단계적 출시(예: 10% → 50% → 100%) |

### 테스트 트랙

| 트랙 | 인원 | 심사 |
| ----------- | ----------- | ----------- |
| 내부 테스트 | **앱당 최대 100명** | 표준 정책·보안 심사를 **거치지 않을 수 있음** → 가장 빠름 |
| 비공개 테스트 | 이메일 목록 단위(목록당 최대 2,000명, 트랙당 목록 50개) | 표준 심사 |
| 공개 테스트 | 기본 무제한 (제한 시 최소 1,000명) | 표준 심사 |

* 신규 **개인** 계정의 12명/14일 요건은 **비공개 테스트** 트랙이어야 한다. 내부 테스트로는 채워지지 않는다
* **단계적 출시는 업데이트에만 된다.** 첫 출시는 100%로 나간다. 비율은 자동으로 안 오르니 직접 올려야 하고, 중단(halt)하면 새 사용자에게는 안 가지만 이미 받은 사람은 그대로다

> 출처: [Sign your app](https://developer.android.com/studio/publish/app-signing) · [테스트 트랙](https://support.google.com/googleplay/android-developer/answer/9845334) · [심사 기간](https://support.google.com/googleplay/android-developer/answer/9859751) · [단계적 출시](https://support.google.com/googleplay/android-developer/answer/6346149) (2026-10-15 확인)

## 5. 심사 전에 챙길 것 (펫바타)

반려 사유 상위권은 기능 버그가 아니라 **정책 누락**이다. [[앱 심사 가이드라인]]에서 펫바타에 걸리는 항목.

| 항목 | 요구 | 펫바타 상태 |
| ----------- | ----------- | ----------- |
| **계정 삭제** | Apple 5.1.1(v): 계정 생성을 지원하면 **앱 안에서 삭제** 제공. Google: **앱 내 삭제 경로 + 웹 삭제 요청 링크 둘 다** (Data safety 양식에도 신고) | 피그마에 삭제 화면이 있는지 확인 필요 |
| **로그인 옵션** | Apple 4.8: 제3자·소셜 로그인을 주 계정으로 쓰면 **동등한 다른 로그인 옵션**도 제공. 그 옵션은 이름·이메일만 수집, 이메일 비공개 허용, 동의 없는 광고 추적 금지를 만족해야 함 (= 사실상 Sign in with Apple이 가장 쉬운 답) | 카카오·구글 로그인을 쓰므로 해당 |
| **권한 사유 문구** | Apple 5.1.1(ii): purpose string이 데이터 사용을 **명확하고 완전하게** 설명 | `app.config.ts`에 카메라·마이크·사진 문구 있음. **위치 권한 문구는 아직 없음** |
| **심사용 테스트 계정** | Apple 2.1: 로그인이 있으면 **데모 계정 정보 제공 + 백엔드 켜 두기** | 심사 기간 동안 살아 있는 서버와 계정 필요 |

* 4.8은 예전엔 "Sign in with Apple 필수"로 읽혔지만 현재 문구는 **조건을 만족하는 동등 옵션**이다. 자체 계정 시스템만 쓰는 앱, 기존 기업 계정으로만 로그인하는 업무용 앱 등은 예외
* Google 계정 삭제 정책은 웹 링크만으로 되는 게 아니다. **앱 안 경로도 필요**하고, 웹 링크는 앱을 지운 사용자를 위한 추가 경로다
* 심사관은 반려동물이 없다. 핵심 기능이 펫 등록·AI 분석이면 **데이터가 미리 들어 있는 데모 계정**을 줘야 기능을 보고 통과시킨다

> 출처: [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) · [Play — 계정 삭제 요구사항](https://support.google.com/googleplay/android-developer/answer/13327111) (2026-10-15 확인)

## 6. 버전 번호는 두 개다

| | 사용자 버전 | 빌드 번호 |
| ----------- | ----------- | ----------- |
| 예 | `1.0.0` | `1`, `2`, `3` … |
| 필드 | `version` | iOS `buildNumber` / Android `versionCode` |
| 단위 | 기능·출시 단위 ([[시맨틱 버저닝(SemVer)]]) | **빌드할 때마다** |
| 규칙 | 같은 값으로 여러 빌드 가능 | **무조건 증가, 재사용 불가.** 스토어가 같은 번호 업로드를 거부 |

같은 1.0.0으로 TestFlight 빌드를 다섯 번 올리려면 빌드 번호가 다섯 번 달라야 한다. 손으로 올리다 보면 반드시 충돌하므로 자동화한다 → [[빌드 번호와 버전]]

**펫바디:** `eas.json`의 `cli.appVersionSource: "remote"` 로 빌드 번호를 **EAS 서버가 관리**하고, `production` 프로필에 `autoIncrement: true` 가 걸려 있다. 첫 `eas build` 때 versionCode가 1 → 2로 올라간 것이 이 때문이다. 로컬 `app.config.ts` 의 값이 아니라 EAS 쪽 값이 기준이라는 걸 기억할 것.

## 7. EAS vs 내부 빌드·배포

**결론: 내부에서도 전부 가능하다.** 애플·구글 계정과 심사는 어느 쪽이든 똑같다. 달라지는 건 **빌드 기계 · 서명 관리 · 업로드 · OTA 를 누가 맡느냐**다. [[EAS]]는 이걸 대신해 주는 서비스일 뿐, [[Expo]] 앱이 EAS에 묶이는 건 아니다.

| 항목 | EAS | 내부 |
| ----------- | ----------- | ----------- |
| iOS 빌드 | EAS macOS 서버 ([[EAS Build]]). **Mac 불필요** | 사내 Mac 필수. Mac mini를 빌드 전용으로 |
| Android 빌드 | EAS Linux 서버 | 사내 Linux. 기존 Docker·[[쿠버네티스(Kubernetes)]] 활용 |
| 서명 관리 | 자동 생성·보관 | 직접. [[fastlane]] match, CI 비밀값 |
| 테스트 배포 | 설치 링크·QR 자동 | Android는 사내 nginx에 APK, iOS는 TestFlight 직접 |
| 스토어 제출 | `eas submit` ([[EAS Submit]]) | Xcode·Transporter·Play Console 수동, 또는 fastlane |
| OTA | `eas update` ([[EAS Update]]) | expo-updates 프로토콜 서버 직접 운영 |
| 자동화 | EAS Workflows — 자동 트리거는 **GitHub 이벤트**·스케줄뿐. 사내 Gitea에선 `eas workflow:run` 을 호출하는 식 (미검증) | 사내 Gitea Actions · Jenkins ([[지속적 통합(CI)]]) |
| 비용 | 무료 월 iOS 15 + Android 15회, 이후 유료 | Mac mini 구매 + 구축·유지 인력 |
| 소스 보안 | **코드가 Expo 서버로 업로드** | 사내에만 |
| 초기 난이도 | 낮음 | 높음. 특히 iOS 서명 |

### 중간 선택지 — `eas build --local`
EAS 빌드 절차를 **내 맥에서** 돌린다.
* 클라우드 빌드 횟수를 안 쓰고, 소스가 밖으로 안 나간다
* 한 번에 한 플랫폼만, 캐시·비밀 환경변수 미지원, Node·fastlane·CocoaPods·Android SDK를 로컬에 직접 갖춰야 함
* **Expo 로그인은 필요하다** — `eas login` 또는 `EXPO_TOKEN`. EAS 서버와의 통신은 프로젝트 존재 확인과 (EAS 관리 서명을 쓸 때) 자격 증명 다운로드뿐
* 서명 파일을 EAS에 둘지 로컬 `credentials.json` 으로 둘지는 따로 정해야 한다

> 출처: [Local builds](https://docs.expo.dev/build-reference/local-builds/) · [expo.dev/pricing](https://expo.dev/pricing) (2026-10-15 확인)

## 8. 내부에서 하는 방법

### 공통
`npx expo prebuild --clean` 으로 `ios/`·`android/` 를 만들면 그다음은 **일반 네이티브 프로젝트**다. Expo의 CNG(설정에서 네이티브 프로젝트를 생성) 덕분에 이 폴더는 커밋하지 않고 빌드 때마다 새로 만든다.

### Android
```bash
npx expo prebuild --clean --platform android
cd android
./gradlew assembleRelease   # APK — 사내 테스트 배포용
./gradlew bundleRelease     # AAB — 스토어 업로드용
```
* JDK 17 + Android SDK + Node가 든 Docker 이미지를 만들어 기존 `make build` 흐름에 붙인다 ([[Docker 멀티스테이지 빌드]]와 같은 방식)
* 업로드 키스토어와 비밀번호는 k8s Secret / CI 비밀값으로. **레포에 커밋 금지** — 업로드 키는 재설정이라도 되지만 유출은 다른 문제다
* 테스트 배포: 사내 nginx에 APK 다운로드 페이지 (사내망 한정)
* 스토어 업로드: Play Console 수동, 또는 fastlane supply (구글 서비스 계정 JSON 키 필요)

### iOS
* Mac mini + Xcode + Gitea Actions 러너(또는 Jenkins 에이전트)
* **fastlane match**: 인증서·프로필을 **암호화해 사내 git에 보관**하고 팀원과 CI가 같은 서명을 공유. 저장소는 git 외에 GCS·S3도 가능
* fastlane gym(또는 `xcodebuild`)으로 `.ipa` → fastlane pilot으로 TestFlight 업로드 (App Store Connect API 키)
* **사내 서버에서 직접 받아 설치하는 방식은 불가.** 그건 Apple Developer **Enterprise** Program 영역인데, **직원 100명 이상 법인 · 사내 전용 앱 · Apple 검증 인터뷰 통과**가 조건이고 연 299 USD다. 일반 사용자 배포에는 못 쓰고, Ad Hoc·TestFlight로 충분하면 쓰지 말라고 명시돼 있다

> 출처: [fastlane match](https://docs.fastlane.tools/actions/match/) · [Enterprise Program](https://developer.apple.com/programs/enterprise/) (2026-10-15 확인)

### OTA (선택)
expo-updates 라이브러리는 그대로 두고 **업데이트 서버만 사내 k8s**에 띄운다. Expo가 공개한 예제 서버(`expo/custom-expo-updates-server`)가 Expo Updates 프로토콜 구현체다.

| 할 일 | 내용 |
| ----------- | ----------- |
| 번들 업로드 | `npx expo export` 결과물을 서버에 올림 |
| 호환성 관리 | `runtimeVersion` — 네이티브 코드가 같은 빌드에만 업데이트를 내려야 한다. 정책 `appVersion`·`nativeVersion`·`fingerprint` 중 선택 |
| 업데이트 서명 | `updates.codeSigningCertificate` · `codeSigningMetadata`. 업데이트 서버는 공급망 공격 지점이라 사실상 필수 |

**OTA를 안 쓰면 모든 수정이 빌드 + 심사다.** 그러면 [[펫바타 앱 - React Native 채택 근거]]의 "RN도 웹뷰만큼 배포가 빠르다"는 근거가 약해진다. 내부로 옮길 때 OTA를 빼먹으면 안 되는 이유.

> 출처: [Custom updates server](https://docs.expo.dev/distribution/custom-updates-server/) (2026-10-15 확인)

### 자동화 예시
```
git tag v1.0.0 && git push --tags
        │
        ▼
  Gitea Actions (태그 트리거)
        │
        ├─ Linux 러너 (Docker)
        │    expo prebuild → ./gradlew bundleRelease → fastlane supply (Play 내부 테스트 트랙)
        │
        └─ Mac 러너 (Mac mini)
             expo prebuild → fastlane match → fastlane gym → fastlane pilot (TestFlight)
```

## 9. 추천

| 시점 | 방식 |
| ----------- | ----------- |
| 지금 (개발) | 로컬 실행 + 필요할 때만 EAS 빌드 |
| 테스트 배포 시작 | EAS `preview` (Android APK) + TestFlight |
| 출시 이후 | 빌드량·비용·보안 정책을 보고 EAS 유지 또는 내부 이전 |

### 내부로 가야 할 때
* 회사가 **소스 외부 반출을 금지**할 때
* 빌드량이 많아 **EAS 유료 비용 > Mac mini + 운영 인건비** 일 때
* 그때도 **Android 먼저**(기존 Docker/k8s 재사용), iOS는 Mac 도입 후

### 지금 바로 할 일
1. **회사 명의** Apple Developer · Play Console 신청 (D-U-N-S 조회·신청 포함)
2. 번들 ID 확정 (현재 `com.petbody.app` 은 임시값)
3. 소스를 외부 클라우드에 올려도 되는지 회사 정책 확인 → EAS 클라우드 빌드 계속 쓸지가 여기서 갈린다

### 펫바디 현황 (2026-10-15)

| 항목 | 상태 |
| ----------- | ----------- |
| 레포 | `petbody-client-app` (사내 Gitea) — EAS Workflows 자동 트리거 대상 아님 |
| EAS 프로젝트 | `@seraaa/petvatar` (개인 계정, 나중에 Organization 이전 예정) |
| 번들 ID | `com.petbody.app` (임시값). 개발 빌드는 `.dev` 접미사 |
| eas.json 프로필 | `development` / `preview`(Android APK) / `production`(autoIncrement) / `simulator`(iOS 시뮬레이터, Apple 계정 불필요) |
| Apple Developer | **계정 아직 없음** → 실기기 iOS 빌드·TestFlight 불가, 시뮬레이터만 |

## 정리
* 앱 배포 = **서명 + 심사**. 롤백이 없으니 단계적 출시와 OTA가 안전장치
* 일정의 최앞단은 개발이 아니라 **회사 명의 계정 + D-U-N-S + 번들 ID**
* TestFlight 내부 100명·심사 없음 / 외부 1만 명·첫 빌드 베타 심사 / Play 내부 테스트 100명
* 심사 반려 1순위 대비: 앱 내 계정 삭제(구글은 웹 링크도), 소셜 로그인 쓰면 동등 로그인 옵션, 권한 문구, 데모 계정
* 빌드 번호는 매번 증가 — 펫바디는 EAS remote가 관리
* EAS ↔ 내부는 빌드 기계·서명·업로드·OTA 담당만 다르다. 내부로 가면 **OTA 서버까지** 챙겨야 RN 채택 근거가 유지된다

#펫바디
