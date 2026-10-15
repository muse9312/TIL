# OTA 업데이트
----

**OTA(Over-The-Air) 업데이트**는 앱 스토어를 거치지 않고 **앱의 일부(주로 JS 번들·에셋)를 네트워크로 내려받아 교체**하는 배포 방식이다.

원래 "OTA"는 무선으로 펌웨어·OS를 업데이트하는 걸 넓게 가리키는 말이다(스마트폰 OS 업데이트도 OTA). 앱 개발에서 말하는 OTA는 그중 **스토어 심사 없이 앱 코드 일부를 바꾸는 것**을 뜻한다.

## 1. 왜 가능한가

네이티브 앱(Swift/Kotlin)은 기계어로 컴파일된 바이너리라, 바꾸려면 새 바이너리를 스토어로 내야 한다. 반면 [[React Native]]·Cordova·Capacitor 같은 앱은 화면·로직이 **JS로 되어 있고 앱 안의 JS 엔진(인터프리터)이 실행**한다. JS 파일만 갈아끼우면 앱의 동작이 바뀐다.

```
스토어 경로:  코드 수정 → 빌드 → 서명 → 제출 → 심사(수 시간~수 일) → 사용자가 업데이트
OTA 경로:    코드 수정 → JS 번들 업로드 → 앱이 다음 실행 때 내려받음
```

## 2. 구현 예

| 구현 | 대상 | 상태 (2026-10-15 확인) |
| ----------- | ----------- | ----------- |
| [[EAS Update]] | Expo / React Native | 운영 중. Expo 공식 |
| 자체 서버 (Expo Updates 프로토콜) | Expo / React Native | `expo-updates` 가 커스텀 서버 지원 |
| Microsoft CodePush (App Center) | React Native · Cordova | **2025-03-31 App Center와 함께 종료.** 독립 실행형 `code-push-server` 는 공개됐으나 2025-05-20 저장소 아카이브 → 직접 유지보수해야 함 |
| Capgo 등 | Capacitor | 서드파티 (미검증) |

> CodePush 출처: [App Center 종료 안내](https://learn.microsoft.com/appcenter/retirement), [microsoft/code-push README](https://cdn.jsdelivr.net/gh/microsoft/code-push@master/README.md)

## 3. 스토어 정책 — 어디까지 허용되나

OTA는 "심사 우회"처럼 보이지만, 양 스토어 모두 **조건부로 허용**한다. 조건의 핵심은 같다: **인터프리터가 실행하는 코드만, 앱의 목적을 바꾸지 않는 범위에서.**

### Apple
**Apple Developer Program License Agreement 3.3.1(B) Executable Code** — 해석 코드(interpreted code)는 앱에 다운로드할 수 있지만, 다음을 **모두** 지켜야 한다.

* (a) 제출한 앱의 **주요 목적(primary purpose)을 바꾸지 않을 것**
* (b) 앱을 **중대하거나 실질적으로(material or substantial) 업데이트·수정하는 데 쓰지 않을 것**
* (c) 다른 코드·앱의 스토어를 만들지 않을 것
* (d) 시스템 보안 기능·자녀 보호 기능을 우회하지 않을 것
* (e) 이 계약과 App Store 심사 가이드라인을 위반하지 않을 것
* (f) 악성코드가 아닐 것

**App Store Review Guidelines 2.5.2** — 앱은 번들 안에서 자기완결적이어야 하고, *"앱의 기능을 도입하거나 변경하는 코드를 다운로드·설치·실행해서는 안 된다."*

> 출처: [Apple Developer Program License Agreement](https://developer.apple.com/support/terms/apple-developer-program-license-agreement/) · [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) (2026-10-15 확인)

2.5.2를 문자 그대로 읽으면 OTA 전체가 금지처럼 보이지만, 계약 3.3.1(B)가 해석 코드의 다운로드를 명시적으로 허용하고 있어 **버그 수정·작은 UI 개선 수준의 JS 업데이트는 실무에서 통용**된다. 선은 "**심사받은 앱과 다른 앱이 되는가**"다.

### Google Play
**Device and Network Abuse 정책** — *"Google Play로 배포된 앱은 Google Play의 업데이트 방식 외의 방법으로 자신을 수정·교체·업데이트할 수 없다. 또한 Google Play가 아닌 곳에서 실행 코드(dex, JAR, .so 등)를 다운로드할 수 없다."* 단 *"Android API에 간접 접근하는 가상 머신이나 인터프리터에서 실행되는 코드(웹뷰·브라우저의 JavaScript 등)에는 적용되지 않는다."*

> 출처: [Device and Network Abuse](https://support.google.com/googleplay/android-developer/answer/9888379) (2026-10-15 확인)

### 판단 기준

| OTA로 보내도 되나? | 예 |
| ----------- | ----------- |
| O 괜찮다 | 버그 수정, 문구·스타일 수정, 기존 기능의 작은 개선 |
| △ 애매 — 스토어 빌드 권장 | 눈에 띄는 신규 기능 추가, 화면 흐름의 큰 개편 |
| X 안 된다 | 앱 성격 변경(펫케어 앱 → 쇼핑몰), 심사 때 숨긴 기능 켜기, 결제 정책 우회 |

"심사 때 숨겨 놨다가 OTA로 켠다"는 [[피처 플래그(Feature Flag)]]도 같은 선에 걸린다. 스위치 방식이 무엇이든 **심사받은 앱과 다른 앱이 되면** 위반이다.

## 4. 장단점

### 장점
* 심사 대기 없이 **분 단위 배포** — 치명 버그 핫픽스에 결정적
* 사용자가 스토어에서 업데이트 버튼을 안 눌러도 반영
* 롤백도 같은 경로로 빠르다

### 단점·함정
* **네이티브 변경은 불가.** 네이티브 모듈·권한·SDK가 바뀌면 결국 스토어 경로
* 바이너리 버전과 JS 버전의 **호환성 관리**가 생긴다 (EAS Update는 runtimeVersion으로 해결)
* 사용자마다 실제 실행 중인 코드가 다를 수 있어 버그 재현이 어려워진다 → 크래시 리포트에 업데이트 ID를 같이 남길 것
* 남용하면 정책 위반 → 앱 삭제·계정 제재 위험
* 업데이트 서버가 곧 **공급망 공격 지점**이다. 서버나 배포 키가 털리면 모든 사용자에게 악성 JS가 간다 업데이트에도 [[코드 서명(Code Signing)]]을 걸 수 있으면 켜 둘 것. EAS Update는 공개키 서명을 기기에서 검증하는 기능을 제공하지만 **Production·Enterprise 플랜 전용**이다 ([공식 문서](https://docs.expo.dev/eas-update/code-signing/), 2026-10-15 확인)

## 정리
* OTA = 스토어 없이 JS·에셋만 교체. JS 엔진이 있는 앱이라서 가능
* Apple 3.3.1(B) · Google Device and Network Abuse 모두 **인터프리터 코드는 허용, 앱 목적 변경은 금지**
* 선 긋기: "심사받은 앱과 다른 앱이 되는가"
* CodePush는 2025-03 종료. RN에선 [[EAS Update]] 또는 자체 서버
