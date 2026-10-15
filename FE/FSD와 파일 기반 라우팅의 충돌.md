# FSD와 파일 기반 라우팅의 충돌
----

[[Feature-Sliced Design(FSD)]]의 `pages` 레이어와 파일 기반 라우터의 `routes/` 디렉터리가 **같은 자리를 놓고 겹치는** 문제.

원인은 명확하다. **둘 다 "화면 하나 = 폴더/파일 하나"를 주장하는데, 폴더 배치 규칙이 서로 다르다.**

| | 배치 기준 |
| ----------- | ----------- |
| FSD `pages` | **업무 도메인** — `pages/alerts/{ui,api,model}` |
| 파일 기반 라우팅 | **URL 경로** — `routes/stores/$storeId/alerts.tsx` |

FSD를 그대로 따르면 라우터가 라우트를 못 찾고, 라우터를 그대로 따르면 `pages` 레이어가 사라진다. [[TanStack Router]]뿐 아니라 Next.js의 `app/`, Nuxt, SvelteKit 등 [[파일 기반 라우팅]]을 쓰는 모든 스택에서 같은 문제가 난다.

## 1. 해법 A — `routes/`를 얇은 어댑터로

라우트 파일은 **경로 선언만** 하고, 실제 화면은 `pages` 슬라이스에서 가져다 마운트한다.

```tsx
// src/routes/stores/$storeId/alerts.tsx   ← 이게 전부
import { createFileRoute } from '@tanstack/react-router';
import { AlertsPage } from '@/pages/alerts';

export const Route = createFileRoute('/stores/$storeId/alerts')({
  component: AlertsPage,
});
```

```
src/
├── routes/                      ← 라우터 소유. 3줄짜리 어댑터만
│   └── stores/$storeId/alerts.tsx
└── pages/
    └── alerts/                  ← FSD 소유. 진짜 화면
        ├── ui/AlertsPage.tsx
        ├── model/
        └── index.ts
```

### 장점
* **설정을 건드리지 않는다.** 플러그인 기본값 그대로, 라우터 문서가 그대로 통한다
* "파일 추가 = 라우트 생성"이 유지된다
* 두 체계의 소유권이 명확하다 — `routes/`는 라우터 것, `pages/`는 FSD 것
* `loader`·`validateSearch`·`beforeLoad`를 어댑터에 두면 **라우팅 관심사와 화면 관심사가 자연스럽게 분리**된다

### 단점
* **화면 하나에 파일 두 곳.** 새 화면을 만들 때마다 양쪽을 건드린다
* `routes/`가 FSD 레이어 어디에도 안 속한다. 엄밀히는 `app` 레이어에 가깝지만 스펙에 그런 자리가 없어서, **린터가 이 폴더를 어떻게 볼지 예외 설정이 필요**하다
* 어댑터에 로직이 스멀스멀 쌓인다. "여기 loader 하나만 더" 가 반복되면 두 번째 `pages`가 된다

> 어댑터 파일에 **UI를 절대 쓰지 않는다**는 규칙만 지키면 실무에서 가장 무난하다.

## 2. 해법 B — Virtual File Routes

라우터 플러그인의 `virtualRouteConfig` 옵션으로 **라우트 트리를 코드로 직접 정의**한다. 라우트 파일이 어디 있든 상관없어지므로 `pages/` 안에 그대로 둘 수 있다.

```ts
// routes.ts
import { rootRoute, route, index, layout, physical } from '@tanstack/virtual-file-routes';

export const routes = rootRoute('app/root.tsx', [
  index('pages/home/ui/HomePage.tsx'),
  layout('pages/store-layout.tsx', [
    route('/stores/$storeId', [
      route('/alerts', 'pages/alerts/ui/AlertsPage.tsx'),
      route('/monitoring', 'pages/monitoring/ui/MonitoringPage.tsx'),
    ]),
  ]),
  physical('/admin', 'pages/admin/routes'),   // 이 하위만 파일 기반으로
]);
```

```ts
// vite.config.ts
tanstackRouter({ target: 'react', virtualRouteConfig: routes })
```

| 헬퍼 | 역할 |
| ----------- | ----------- |
| `rootRoute` | 루트 레이아웃 |
| `route` | 경로 하나 (자식 배열을 받을 수 있음) |
| `index` | 부모 경로의 인덱스 |
| `layout` | URL에 안 나타나는 레이아웃 |
| `physical` | **이 지점부터는 파일 기반으로** 스캔 — 두 방식을 섞는 탈출구 |

### 장점
* **어댑터 파일이 사라진다.** 화면 하나에 파일 하나
* `pages` 레이어가 FSD 규칙대로 온전히 산다
* 라우트 트리 전체가 파일 하나에 보인다 — 어떤 경로가 있는지 한눈에 파악됨
* `physical`로 **부분 도입**이 된다. 도메인 일부만 가상 라우팅으로 옮길 수 있다

### 단점 · 함정
* **"파일 추가 = 라우트 생성"의 이점을 잃는다.** 라우트를 추가할 때마다 `routes.ts`를 손으로 고쳐야 한다. 파일 기반 라우팅을 쓰는 가장 큰 이유가 이건데, 그걸 반납하는 셈
* **`routes.ts`가 중앙 병목이 된다.** 여러 명이 각자 화면을 추가하면 이 파일에서 계속 충돌한다
* **virtual routing + `route.tsx`/`index.tsx` 조합에서 경로 충돌 이슈가 보고돼 있다.** 레이아웃 라우트와 인덱스를 섞을 때 생성된 트리가 기대와 달라지는 사례가 있으니, 도입 전에 실제 구조로 한 번 검증해봐야 한다
* 설정이 늘어난 만큼 **문서·예제와 멀어진다.** 대부분의 TanStack Router 자료는 파일 기반 전제로 쓰여 있다

## 3. 어느 쪽을 고르나

| | A: 어댑터 | B: Virtual Routes |
| ----------- | ----------- | ----------- |
| 화면당 파일 | 2개 | 1개 |
| 설정 | 기본값 그대로 | `virtualRouteConfig` |
| 라우트 추가 | 파일 만들면 끝 | 중앙 파일 수정 |
| 팀 충돌 | 없음 | `routes.ts`에 집중 |
| 문서 호환 | 좋음 | 떨어짐 |
| FSD 순정도 | `routes/`가 예외로 남음 | 깔끔 |

**A가 기본값이다.** 잃는 게 "3줄짜리 파일 하나"뿐인데, B가 잃는 건 파일 기반 라우팅의 핵심 이점과 검증 부담이다. 라우트가 수십 개를 넘어가고 여러 명이 동시에 추가하는 상황에서는 A의 비용이 거의 안 느껴지는 반면 B의 병목은 커진다.

B가 나은 경우는 **라우트 트리가 URL 구조와 크게 어긋날 때**다. 같은 컴포넌트를 여러 경로에 매다는 식의 매핑이 많으면 파일 이름으로 표현하는 것보다 코드로 쓰는 게 낫다.

### 세 번째 선택지 — `pages` 레이어를 안 쓴다
FSD는 **모든 레이어가 선택적**이다. `routes/`를 사실상 `pages` 레이어로 인정하고, 그 안에서는 라우터 규칙을 따르되 **`widgets` 이하로만 import한다**는 규칙만 지키는 방식이 있다. 충돌 자체를 없애는 대신 FSD의 `pages` 관련 규칙을 포기하는 절충이고, 도입 초기에 마찰을 줄이는 데는 가장 효과적이다.

## 정리

* 충돌의 원인은 **배치 기준이 다르다는 것** — FSD는 도메인, 라우터는 URL
* **A(어댑터)** — `routes/`에 3줄짜리 파일을 두고 `pages` 슬라이스를 마운트. 설정을 안 건드리고 문서가 그대로 통한다
* **B(Virtual File Routes)** — `virtualRouteConfig`로 트리를 코드로 정의. 어댑터가 사라지는 대신 **"파일 추가 = 라우트"를 잃고** 중앙 파일 병목과 경로 충돌 검증이 따라온다
* 기본은 **A**. B는 라우트 트리가 URL 구조와 크게 어긋날 때
* 아예 **`pages` 레이어를 포기하고 `routes/`로 대체**하는 절충도 스펙 위반이 아니다

#펫바디
