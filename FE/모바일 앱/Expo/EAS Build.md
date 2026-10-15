# EAS Build
----

**[[EAS]]의 빌드 서비스.** [[Expo]]·[[React Native]] 프로젝트를 클라우드 서버에서 빌드해 **서명된 앱 바이너리(IPA/APK/AAB)** 를 만들어 준다.

> 기준: EAS Build 공식 문서 (2026-10-15 확인)
> 출처: https://docs.expo.dev/build/introduction/ · https://docs.expo.dev/build-reference/infrastructure/

## 1. 왜 쓰나 — iOS 빌드는 Mac이 있어야 한다

iOS 앱은 Xcode로만 빌드되고, Xcode는 macOS에서만 돈다. 그래서 원래는 **팀에 Mac이 반드시 있어야** iOS 빌드가 나온다. EAS Build는 그 Mac을 클라우드에서 빌려준다.

| 플랫폼 | 빌드 서버 | 기본 이미지 (2026-10-15 확인) |
| ----------- | ----------- | ----------- |
| iOS | Mac mini 호스트의 **macOS** | `macos-tahoe-26.5-xcode-26.6` (Xcode 26.6) |
| Android | GCP의 **Ubuntu** VM | `ubuntu-26.04-jdk-17-ndk-r27b-sdk-57` |

결과적으로 **Windows·Linux 팀원도 iOS 빌드를 낼 수 있다.** 같은 이유로 사내 Linux 서버(k8s)에서 iOS 빌드를 돌릴 수 없다는 제약도 우회한다 → [[지속적 통합(CI)]]

## 2. 하는 일

```
eas build --profile preview --platform ios
   │
   ├─ 1. 소스 업로드 (프로젝트를 압축해 EAS로)
   ├─ 2. 의존성 설치 · 네이티브 프로젝트 생성(prebuild)
   ├─ 3. 서명 자격 증명 주입      ← 없으면 생성 제안
   ├─ 4. Xcode / Gradle 빌드
   └─ 5. 결과물 + 설치 링크·QR을 대시보드에
```

### 서명 자동 관리
[[코드 서명(Code Signing)]]에 필요한 것들 — iOS 배포 인증서·프로비저닝 프로필·푸시 키, Android 키스토어 — 을 **자동 생성해 EAS 서버에 보관**하고 이후 빌드에 재사용한다. 공식 문서는 *"iOS 자격 증명을 한 번 만들고 나면 빌드할 때 Apple Developer 팀 접근 권한이 필요 없다"* 고 설명한다. 내 키를 직접 쓰는 것도 된다(`credentials.json`).

### 내부 배포 (스토어 거치지 않고 테스터에게)
빌드 프로필에 `"distribution": "internal"` 을 주면 스토어용 대신 **테스터 설치용** 결과물을 만든다.

```json
// eas.json
{
  "build": {
    "preview": { "distribution": "internal" },
    "production": {}
  }
}
```

| 플랫폼 | 결과물 | 조건 |
| ----------- | ----------- | ----------- |
| Android | **APK** (기본 AAB 대신) | 경고 수락 후 링크로 바로 설치 |
| iOS | **Ad Hoc** 빌드 | 유료 Apple Developer 계정 + **기기 UDID 사전 등록**(`eas device:create`). 연간 iPhone 100대 한도 |

iOS가 이렇게 까다로운 이유는 [[코드 서명(Code Signing)]] — 아이폰은 서명에 등록된 기기에만 설치를 허락한다.

## 3. 대안

| 방법 | 언제 |
| ----------- | ----------- |
| `npx expo run:ios` / `run:android` | 로컬 개발 빌드. Mac·Xcode·Android Studio가 있을 때 |
| `eas build --local` | **EAS와 같은 빌드 과정을 내 기기에서.** 소스를 외부로 못 올리는 보안 정책, 클라우드 빌드 실패 디버깅 |
| Xcode Archive / Gradle 직접 | 네이티브 프로젝트를 직접 관리할 때 |
| fastlane + 자체 CI (Mac 러너) | EAS를 안 쓰고 파이프라인을 직접 구축할 때 |

`--local` 의 제약 (공식 문서): 한 번에 한 플랫폼만(`all` 불가), **캐시 미지원**, 비밀 환경변수 미지원, `eas.json` 의 도구 버전 지정 무시, Windows 미지원. iOS는 당연히 Mac이 필요하다.

## 4. 장단점

### 장점
* Mac 없이 iOS 빌드
* 서명 지옥(인증서 만료·프로필 재발급·키스토어 공유)을 EAS가 대신
* 빌드 기록·로그·설치 QR이 대시보드에 남아 테스터 배포가 쉽다

### 단점
* **소스가 Expo 서버로 올라간다**
* 무료 플랜: 월 iOS 15 · Android 15회, **저우선 대기열**, 45분 타임아웃 (expo.dev/pricing, 2026-10-15 확인). 대기열이 몇십 분씩 밀리는 일이 흔하다
* 빌드 한 번이 수 분~수십 분 → JS만 바뀐 거라면 빌드하지 말고 [[EAS Update]]로 보내야 한다. **무엇이 새 빌드를 요구하는지**(네이티브 변경) 구분하는 감각이 핵심

## 정리
* EAS Build = 클라우드 Mac/Linux에서 빌드 + 서명까지
* iOS 빌드는 Mac 필수 → 이걸 대신해 주는 게 존재 이유
* 테스터 배포는 `distribution: internal` (Android APK, iOS Ad Hoc + UDID 등록)
* 소스를 밖으로 못 내보내면 `--local`

#펫바디
