# Expo
----

**Expo**는 [[React Native]] 위에서 앱을 만들게 해 주는 **오픈소스 [[프레임워크]]** 다. 네이티브 기능을 감싼 **SDK 모듈 모음**, 개발 서버·빌드를 다루는 **CLI**, `app/` 폴더 구조로 화면을 정하는 **expo-router**([[파일 기반 라우팅]])로 이뤄진다.

React Native만 쓰면 Xcode·Gradle 프로젝트를 직접 손보고, 카메라·알림 같은 기능마다 라이브러리 호환을 맞춰야 한다. Expo는 그 네이티브 쪽 잡일을 **설정 파일과 규칙으로 바꿔 주는 층**이다.

> 기준: Expo SDK 57 (2026-10-15 확인)
> 출처: https://expo.dev/changelog/sdk-57 · https://docs.expo.dev/workflow/continuous-native-generation/ · https://docs.expo.dev/develop/development-builds/introduction/ · https://docs.expo.dev/more/expo-cli/ · https://docs.expo.dev/workflow/configuration/

## 1. Expo와 EAS는 다르다

| | Expo | [[EAS]] |
| ----------- | ----------- | ----------- |
| 정체 | 앱을 **만드는** 프레임워크 | 앱을 **빌드·배포하는** 클라우드 서비스 |
| 비용 | 무료(오픈소스) | 무료 구간 + 유료 |
| 없이도 되나 | — | 된다. `npx expo run:ios` 로 로컬 빌드 |
| 비유 | Next.js | Vercel |

EAS 쪽 이야기(빌드·제출·OTA)는 [[EAS]] 노트에 있다. 여기서는 **코드를 쓰는 쪽**만 다룬다.

## 2. SDK 버전 — RN 버전을 묶어서 판다

Expo는 **SDK 번호 단위로 React Native·React 버전과 Expo 모듈 버전을 한 묶음으로** 릴리스한다. "RN 0.86에 어떤 expo-camera가 맞지?"를 고민하는 대신 "SDK 57"이라는 숫자 하나만 맞추면 된다.

| SDK | React Native | React | 릴리스 |
| ----------- | ----------- | ----------- | ----------- |
| 57 | 0.86 | 19.2 | 2026-06-30 |

> 출처: [Expo changelog SDK 57](https://expo.dev/changelog/sdk-57) (2026-10-15 확인). SDK 56과 React 버전(19.2)은 같다

### `npx expo install` 로 설치한다
React Native는 하위 호환이 약해서 **패키지가 설치된 RN 버전과 정확히 맞아야** 한다. `npm install` 은 그냥 최신을 받지만, `npx expo install` 은 Expo가 관리하는 **SDK별 검증된 버전 목록**에서 골라 고정한다.

```bash
npx expo install expo-camera      # SDK에 맞는 버전으로 설치
npx expo install --check          # 어긋난 패키지 점검 (CI에서는 실패 코드로 종료)
npx expo install --fix            # 어긋난 버전 일괄 교정 — SDK 올린 직후에 쓴다
```

## 3. CNG — 네이티브 폴더를 커밋하지 않는다

**CNG(Continuous Native Generation)** 는 Expo의 가장 중요한 개념이다. `ios/`·`android/` 폴더를 **소스로 관리하지 않고, 필요할 때마다 설정에서 생성**한다.

```
app.config.ts  +  config plugin들  +  package.json의 네이티브 모듈
        │
        ▼   npx expo prebuild --clean
   ios/  android/   (임시 산출물, .gitignore)
        │
        ▼
   Xcode / Gradle 빌드
```

* 공식 문서: 네이티브 프로젝트는 *"디버깅이나 빌드처럼 필요할 때만 생성되는 단명(short-lived) 프로젝트"* 이고, 개발자는 네이티브 코드 전체가 아니라 **커스터마이징의 정의만** 관리한다
* 새 프로젝트를 만들면 `ios/`·`android/` 가 `.gitignore` 에 자동으로 들어간다
* `--clean` 은 기존 네이티브 폴더를 지우고 새로 만든다. 일부 config plugin이 멱등하지 않아서 **`--clean` 이 가장 안전한 방법**이라고 문서가 권한다

### 왜 이렇게 하나
웹으로 치면 `dist/` 를 커밋하지 않는 것과 같다. 네이티브 폴더는 수천 줄짜리 생성물이라, 커밋하면

* SDK 올릴 때마다 Xcode 프로젝트 diff가 수백 줄 생기고 충돌이 난다
* 누가 Xcode에서 손으로 고친 설정이 어디 있는지 아무도 모른다

CNG면 SDK 업그레이드가 "package.json 올리고 → `prebuild --clean`"으로 끝난다. SDK 57 업그레이드 가이드도 *"이전 SDK에서 생성한 android/ios 폴더는 지우라"* 고 한다.

### 함정
* `ios/` 안의 파일을 **직접 고치면 다음 prebuild 때 날아간다.** 고칠 일이 있으면 반드시 config plugin으로
* 반대로 네이티브 폴더를 커밋하기 시작하면(bare 워크플로) 그때부터 CNG의 이점이 사라진다. 섞어 쓰지 말 것

## 4. Config plugin — 네이티브 수정을 코드로 남긴다

**config plugin** 은 *"prebuild 중에 네이티브 프로젝트를 수정하는 함수"* 다. Info.plist에 권한 문구 넣기, Gradle에 의존성 추가, 위젯 익스텐션 타깃 추가 같은 일을 한다. CNG에서 네이티브를 바꾸는 **유일한 정식 통로**다.

```ts
// app.config.ts
plugins: [
  'expo-router',
  ['expo-font', { fonts: ['./assets/fonts/Pretendard-Regular.otf'] }],
  './plugins/withWorkmanagerVersion',   // 직접 만든 플러그인
],
```

### 실제 사례
* **expo-font**: Pretendard를 **빌드 시점에 앱 바이너리에 포함**한다. 런타임 `useFonts` 로 불러오는 것보다 효율적이다(공식 문서). 단 이 방식은 **Android·iOS 전용**이라 웹에서는 안 먹는다
* **piew-dev의 workmanager 버전 충돌**: 두 라이브러리가 서로 다른 androidx WorkManager 버전을 끌어와 Gradle 빌드가 깨졌다. `android/build.gradle` 을 손으로 고치면 prebuild 때 사라지므로, 버전을 강제하는 **작은 플러그인을 직접 작성**해 해결했다

## 5. 동적 설정 — `app.config.ts`

| | `app.json` (정적) | `app.config.ts` (동적) |
| ----------- | ----------- | ----------- |
| 내용 | JSON | 코드 (TS, 주석, 변수, 환경변수) |
| CLI가 자동 수정 | **된다** | **안 된다** — 직접 고쳐야 함 |
| 쓰는 이유 | 단순한 앱 | 빌드 종류별로 값이 달라야 할 때 |

공식 문서: *"정적 설정은 CLI 도구가 자동으로 고칠 수 있지만, 동적 설정은 개발자가 직접 고쳐야 한다."*

```ts
// app.config.ts — APP_VARIANT로 dev/prod를 한 기기에 같이 설치
const IS_DEV = process.env.APP_VARIANT === 'development';

export default {
  name: IS_DEV ? 'petbody (Dev)' : 'petbody',
  ios: { bundleIdentifier: IS_DEV ? 'com.example.petbody.dev' : 'com.example.petbody' },
  android: { package: IS_DEV ? 'com.example.petbody.dev' : 'com.example.petbody' },
};
```

번들 ID가 다르면 OS는 **별개의 앱**으로 본다. 그래서 개발 빌드와 운영 빌드를 한 폰에 나란히 깔 수 있다. 이 패턴은 [Expo 공식 튜토리얼](https://docs.expo.dev/tutorial/eas/multiple-app-variants/)에 그대로 나온다.

### 단점
* `eas init` 이 `extra.eas.projectId` 를 못 써 넣는 것처럼, **CLI가 설정을 고쳐 주는 기능이 전부 수동**이 된다 → [[EAS]]
* 번들 ID가 변형마다 늘어나면 [[코드 서명(Code Signing)]] 자격 증명·App Group·푸시 설정도 **변형 수만큼** 필요하다

## 6. Expo Go vs Development Build

| | Expo Go | Development Build |
| ----------- | ----------- | ----------- |
| 정체 | 스토어에서 받는 **Expo가 만든 앱** | **내 앱**의 디버그 빌드 (`expo-dev-client` 포함) |
| 네이티브 모듈 | Expo Go에 **미리 들어 있는 것만** | 아무거나 |
| 네이티브 설정 변경 | 불가 | 가능 |
| 시작 비용 | 0 (QR 찍으면 끝) | 한 번 빌드해야 함 ([[EAS Build]] 또는 로컬) |

공식 문서: Development Build는 *"아무 네이티브 라이브러리나 쓰고 네이티브 설정을 바꿀 수 있는 나만의 Expo Go"* 다.

**판단 기준**: [[홈 화면 위젯]], 카카오 같은 [[소셜 로그인]] SDK, 커스텀 config plugin이 **하나라도 들어가는 순간 Expo Go는 못 쓴다.** 펫바타는 처음부터 Development Build로 간다. Expo Go는 튜토리얼·순수 JS 프로토타입용이다.

> 참고: SDK 57용 Expo Go는 iOS 스토어 승인 대기로 `eas go` 로 배포 중이라고 changelog에 적혀 있다 (2026-10-15 기준. 이후 바뀌었을 수 있음)

## 7. 웹 export

react-native-web 덕분에 같은 코드를 `npx expo export --platform web` 으로 웹에도 낼 수 있다. 하지만 **네이티브 전용 모듈은 웹에서 안 돈다.**

| 모듈 | 웹 |
| ----------- | ----------- |
| `react-native-webview` | 미지원 (piew-dev에서 겪음) |
| `expo-secure-store` | 미지원 (piew-dev에서 겪음) |
| expo-font 빌드 시점 포함 | 미지원 (공식 문서) — 웹은 `useFonts` 로 |

이럴 땐 **플랫폼 확장자 파일**로 갈라 쓴다. Metro가 플랫폼에 맞는 파일을 자동으로 고른다.

```
storage.ts          # 공통 (또는 네이티브)
storage.web.ts      # 웹 전용 — localStorage 등으로 대체
```

웹 export는 "덤"이다. 모바일 웹을 본격 서비스해야 하면 [[펫바타 앱 - React Native 채택 근거]]에서 정리한 것처럼 따로 판단한다.

## 8. 함정 — 첫 `pod install` 이 멈춘 것처럼 보인다

최근 RN은 iOS 빌드 시간을 줄이려고 **미리 컴파일된 RN 코어(prebuilt)** 를 내려받는다. 첫 `pod install` 에서 이 파일(체감 약 99MB, piew-dev 기준)을 Maven 저장소에서 받느라 한참 걸리는데, **Expo CLI 스피너에는 진행률이 안 나온다.** 멈춘 줄 알고 끄면 다시 처음부터다.

* 기다리면 끝난다. 두 번째부터는 캐시를 쓴다
* 확인하고 싶으면 `ios/` 에서 `pod install --verbose` 로 직접 돌려 다운로드 로그를 본다
* 크기·다운로드 출처는 겪은 내용 기준이며 공식 수치는 **미검증**

## 장단점

### 장점
* SDK 번호 하나로 RN·라이브러리 버전 호환이 해결된다
* CNG로 네이티브 폴더를 관리하지 않는다 → SDK 업그레이드·충돌 비용이 작다
* 위젯·폰트·권한 같은 네이티브 변경도 config plugin으로 **코드 리뷰 가능한 형태**로 남는다
* 원하면 [[EAS]]로 빌드·배포까지 한 흐름

### 단점
* **Expo가 한 겹 더 있다.** RN 최신 버전이 나와도 다음 SDK가 나올 때까지 기다려야 한다
* config plugin이 없는 네이티브 라이브러리는 플러그인을 **직접 짜야** 한다 (workmanager 사례)
* 네이티브 폴더를 직접 고치는 데 익숙한 개발자에겐 "고쳐도 사라지는" 구조가 낯설다
* 동적 설정을 쓰면 CLI 자동화가 줄어든다

## 정리
* Expo = RN 위의 프레임워크(SDK 모듈 + CLI + expo-router). EAS는 별개의 클라우드
* SDK 단위로 RN 버전이 묶인다. 패키지는 `npx expo install` 로
* `ios/`·`android/` 는 생성물. 네이티브 수정은 반드시 config plugin으로 (`prebuild --clean`)
* 네이티브 모듈이 하나라도 붙으면 Expo Go 대신 Development Build
* 웹 export는 되지만 네이티브 전용 모듈은 `.web.ts` 로 갈라야 한다

#펫바디
