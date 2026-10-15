# React Native
----

**React Native(RN)** 는 Meta가 만든, **React로 iOS·Android의 진짜 네이티브 UI를 그리는 [[프레임워크]]** 다. 화면을 HTML로 그려 [[웹뷰]]에 띄우는 게 아니라, `<View>` 를 쓰면 iOS의 `UIView`, Android의 `android.view.View` 가 실제로 만들어진다.

"한 번 배워서 어디서나 쓴다(Learn once, write anywhere)"가 목표다. 코드 한 벌이 모든 플랫폼에서 똑같이 도는 게 아니라, **React라는 사고방식과 JS 생태계를 그대로 들고 네이티브 앱을 만드는 것**이다.

> 기준: React Native 0.86 ([[Expo]] SDK 57) (2026-10-15 확인)
> 출처: https://reactnative.dev/blog/2025/10/08/react-native-0.82 · https://reactnative.dev/docs/platform-specific-code

## 1. 동작 구조

```
  JS 스레드                                네이티브
┌────────────────────┐                ┌────────────────────┐
│ React 컴포넌트       │                │ UIView / View       │
│ <View><Text/></View>│ ──  JSI  ──▶  │ (실제 OS 위젯)        │
│ 상태·로직·fetch      │  (직접 호출)    │ 카메라·알림 등 모듈    │
└────────────────────┘                └────────────────────┘
       ▲ Hermes 엔진이 JS 번들 실행
```

* JS 번들은 앱 안의 JS 엔진(Hermes)이 실행한다. 그래서 JS만 갈아끼우는 [[OTA 업데이트]]가 가능하다 → [[EAS Update]]
* React가 "무엇을 그릴지" 계산하고, RN 렌더러가 그 결과를 **OS 네이티브 뷰**로 만든다
* 웹 React의 `react-dom` 자리에 RN 렌더러가 들어간 것이라고 보면 된다

### New Architecture
옛 구조는 JS와 네이티브가 JSON 메시지를 **비동기 브리지**로 주고받아 느리고 동기 호출이 불가능했다. [[New Architecture]] 는 이를 JSI(직접 호출) · Fabric(새 렌더러) · TurboModules(지연 로딩 네이티브 모듈)로 바꿨다.

* **RN 0.82부터 New Architecture만으로 돈다.** `newArchEnabled=false` 같은 설정은 무시된다 (공식 블로그, 2026-10-15 확인)
* 그래서 라이브러리를 고를 때 **New Architecture 지원 여부**가 1순위 체크 항목이다. 오래 방치된 라이브러리는 여기서 걸린다

### React Compiler
웹과 같은 [[React Compiler]] 를 RN에서도 쓸 수 있다. 빌드 때 [[메모이제이션(Memoization)]]을 자동 삽입한다. Expo에서는 앱 설정 `experiments.reactCompiler: true` 로 켠다 ([Expo 문서](https://docs.expo.dev/guides/react-compiler/), 2026-10-15 확인. 새 템플릿에서 기본으로 켜져 있는지는 **미검증**).

## 2. 웹 React와 무엇이 같고 무엇이 다른가

웹 개발자가 RN으로 넘어올 때 **바뀌는 건 UI 계층뿐**이다.

| 계층 | 웹 (petbody-client) | RN (펫바타) |
| ----------- | ----------- | ----------- |
| 언어·컴포넌트 모델 | React + TS | **같음** |
| 서버 상태 | [[TanStack Query]] | **같음** |
| 클라이언트 상태 | [[zustand]] | **같음** |
| 폼·검증 | react-hook-form + zod | **같음** |
| 기본 요소 | `div` · `span` · `img` | `View` · `Text` · `Image` |
| 부품 | shadcn/ui | RN 컴포넌트 (shadcn은 DOM 전용이라 못 씀) |
| 스타일 | Tailwind (CSS) | `StyleSheet` 또는 NativeWind (Tailwind 문법 → RN 스타일) |
| 라우팅 | [[TanStack Router]] | expo-router ([[파일 기반 라우팅]]) |
| 스크롤 | 브라우저가 알아서 | `ScrollView` · `FlatList` 를 **직접** 골라야 함 |

### 처음 걸리는 차이
* **텍스트는 반드시 `<Text>` 안에.** `<View>안녕</View>` 은 에러다
* CSS 상속·캐스케이드가 없다. `color` 를 부모 `View` 에 줘도 자식 `Text` 에 안 내려간다
* 레이아웃은 Flexbox뿐이고 **기본 방향이 `column`** 이다 (웹은 `row`)
* 긴 목록을 `map` 으로 그리면 느려진다 → `FlatList` 로 가상화

## 3. 플랫폼 분기

같은 import 경로에 플랫폼별 파일을 두면 번들러(Metro)가 알아서 고른다.

```
Haptic.ios.ts       # iOS
Haptic.android.ts   # Android
Haptic.web.ts       # 웹 export (Expo)
Haptic.ts           # 나머지 기본값

import { haptic } from './Haptic';   // 확장자 없이 import
```

짧은 분기는 `Platform.select({ ios: ..., android: ..., default: ... })` 로 한다. **파일이 통째로 다르면 확장자, 값 몇 개만 다르면 `Platform.select`** 가 기준이다.

## 4. 대안 비교

| 방식 | UI를 누가 그리나 | 코드 공유 | 네이티브 기능 |
| ----------- | ----------- | ----------- | ----------- |
| **React Native** | OS 네이티브 뷰 | iOS·Android (+웹 일부) | 라이브러리 / 직접 모듈 |
| [[하이브리드 앱]] (Capacitor) | [[웹뷰]] 안의 HTML | 웹과 100% | 플러그인 + [[네이티브 브리지]] |
| PWA | 브라우저 | 웹 그 자체 | 브라우저가 허락한 만큼 |
| Flutter | 자체 렌더링 엔진(Skia/Impeller) | iOS·Android·웹 | 플랫폼 채널 |
| 순수 네이티브 (Swift/Kotlin) | OS 네이티브 뷰 | 없음 (두 벌) | 전부, 가장 먼저 |

### React Native
* 장점: 웹 React 인력·라이브러리를 그대로 쓴다. 진짜 네이티브 UI라 제스처·키보드·스크롤 품질이 좋다. OTA로 JS 배포
* 단점: 위젯·[[Live Activity]] 같은 OS 확장은 여전히 Swift/Kotlin 영역. 네이티브 모듈 버전 충돌을 가끔 직접 풀어야 한다

### 하이브리드 앱 (Capacitor)
* 장점: 기존 웹 코드를 거의 그대로 앱으로. 모바일 웹과 화면 공유
* 단점: 기기 웹뷰 성능에 좌우된다(저가 안드로이드). 웹을 감싼 앱은 스토어 심사에서 "최소 기능" 반려 위험

### PWA
* 장점: 스토어 없이 배포, 설치 비용 0
* 단점: 홈 화면 위젯 불가, iOS 푸시는 홈 화면 설치 후에만. 스토어 노출 없음

### Flutter
* 장점: 자체 엔진이라 플랫폼 간 UI가 픽셀 단위로 같다. 성능 안정적
* 단점: Dart라는 별도 언어·생태계. 웹 React 팀과 **아무것도 공유하지 못한다**

### 순수 네이티브
* 장점: 최신 OS 기능을 가장 먼저, 제약 없이. 성능 최상
* 단점: iOS·Android 두 벌을 두 팀이. 비용이 가장 크다

### 판단 기준
* 팀이 **웹 React** 중심이고 네이티브 기능(푸시·카메라·위젯)이 필요하다 → **React Native**
* 이미 웹 서비스가 있고 앱은 **껍데기** 수준이면 된다 → 하이브리드 / PWA
* 팀이 처음부터 모바일 전담이고 웹과 공유할 게 없다 → Flutter도 후보
* OS 신기능이 제품의 핵심이거나 성능 한계까지 써야 한다 → 순수 네이티브

펫바타가 이 기준으로 RN을 고른 과정은 [[펫바타 앱 - React Native 채택 근거]]에 상사 설득 논리까지 정리했다. RN 위에서 실제로 앱을 굴리는 도구는 [[Expo]] 다.

## 정리
* RN은 웹뷰가 아니다. React로 **OS 네이티브 뷰**를 만든다
* 0.82부터 New Architecture 전용 → 라이브러리 고를 때 지원 여부 먼저 확인
* 웹 React와 다른 건 UI 계층뿐. 데이터·상태·폼 라이브러리는 그대로
* 플랫폼 차이는 `.ios`/`.android`/`.web` 확장자 또는 `Platform.select`
* 웹 팀 + 네이티브 기능 필요 = RN이 가장 싸다. 단 [[홈 화면 위젯]]은 RN으로도 결국 네이티브 영역

#펫바디
