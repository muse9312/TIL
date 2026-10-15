# TanStack Router
----

**타입 안전을 1순위로 설계한 클라이언트 우선 라우터.** 경로·파라미터는 물론 **[[쿼리스트링(Query String)]]까지 타입을 추론**하고 스키마로 검증한다.

> 기준 버전: **v1** (`@tanstack/react-router`). 서버 렌더링까지 필요하면 이 위에 얹는 풀스택 [[프레임워크]]가 TanStack Start다 → [[TanStack]]

## 1. 핵심은 search params다

다른 라우터도 경로 파라미터 타입은 어떻게든 해준다. **차별점은 `?` 뒤쪽**이다.

```ts
// React Router — 전부 string | null. 검증도 기본값도 직접
const [sp] = useSearchParams();
const range = Number(sp.get('range') ?? 6);      // 'banana'면 NaN
const level = sp.get('level') ?? 'all';          // 오타를 잡아줄 방법이 없다
```

```ts
// TanStack Router — 스키마 하나로 파싱 · 검증 · 기본값 · 타입 추론
const searchSchema = z.object({
  level: z.enum(['all', 'warn', 'urgent']).catch('all'),
  range: z.coerce.number().int().min(1).max(72).catch(6),
  zone:  z.string().optional(),
});

export const Route = createFileRoute('/stores/$storeId/alerts')({
  validateSearch: searchSchema,
  component: AlertsPage,
});

function AlertsPage() {
  const { level, range } = Route.useSearch();   // level: 'all'|'warn'|'urgent', range: number
}
```

`validateSearch` 는 **Standard Schema를 그대로 받는다.** Zod 4 스키마는 어댑터 없이 바로 넘기면 되고, Zod 3이라면 `@tanstack/zod-adapter` 의 `zodValidator()` 와 `fallback()` 을 쓴다. valibot·arktype도 동일하게 동작한다.

`.catch()` (또는 어댑터의 `fallback()`)가 실무에서 중요하다. **잘못된 URL로 들어와도 라우트가 터지지 않고 안전한 기본값으로 떨어진다.** 사용자가 URL을 손으로 고치거나, 예전 링크를 열거나, 스키마를 바꾼 뒤 옛 링크가 살아 있는 상황이 실제로 흔하다.

### 쓰기도 타입이 붙는다

```tsx
<Link to="/stores/$storeId/alerts"
      params={{ storeId }}
      search={(prev) => ({ ...prev, level: 'urgent' })}>
  긴급만 보기
</Link>
```

`to` 의 오타, `params` 누락, `search` 의 잘못된 값이 **전부 컴파일 에러**다. URL을 상태 저장소로 쓰는 앱에서 이 안전망은 체감이 크다 → [[쿼리스트링(Query String)]]

### ⚠️ search는 자동으로 이어지지 않는다
가장 많이 물리는 함정. `<Link to="/stores/1/alerts">` 처럼 `search` 를 안 주면 **기존 search params가 전부 날아간다.** 브라우저 링크와 같은 동작이지만, 필터를 URL에 담아 쓰는 앱에서는 사고가 된다.

* 한 링크만이면 `search={(prev) => prev}`
* 라우트 전반에서 유지해야 하면 **search 미들웨어** `retainSearchParams(['zone', 'range'])`
* 기본값과 같은 값을 URL에서 지우고 싶으면 `stripSearchParams`

## 2. 파일 기반 라우팅

`src/routes/` 의 파일 구조가 곧 라우트 트리다. Vite 플러그인이 이를 읽어 **`routeTree.gen.ts` 를 생성**하고, 타입 추론은 이 생성 파일에서 나온다.

```ts
// vite.config.ts
import { tanstackRouter } from '@tanstack/router-plugin/vite';
export default defineConfig({
  plugins: [tanstackRouter({ target: 'react', autoCodeSplitting: true }), react()],
});
```

```
src/routes/
├── __root.tsx                        전체 레이아웃 (모든 라우트의 부모)
├── index.tsx                         /
├── _authed.tsx                       경로 없는 레이아웃 (인증 게이트)
├── _authed.stores.tsx                /stores  ← _authed 레이아웃 안
├── _authed.stores.$storeId.alerts.tsx  /stores/123/alerts
└── -components/AlertTable.tsx        - 로 시작하면 라우트가 아님
```

| 규칙 | 의미 |
| ----------- | ----------- |
| `.` (점) | 경로 중첩 — `posts.index.tsx` → `/posts/` |
| `$param` | 동적 파라미터 |
| `_` 접두사 | **경로 없는 레이아웃** — URL에는 안 나오고 레이아웃만 공유 |
| `_` 접미사 (`posts_.$id`) | 부모 레이아웃에서 **빠져나옴** |
| `-` 접두사 | 라우트에서 제외 (같은 폴더에 컴포넌트를 둘 때) |

`autoCodeSplitting` 을 켜면 컴포넌트가 자동으로 분리돼 라우트 단위 지연 로딩이 된다.

> 코드 기반(`createRoute`)으로도 전부 쓸 수 있다. 파일 기반은 **타입 추론과 코드 스플리팅이 자동**이라는 게 이점이지, 필수가 아니다.

### 폴더 구조 방법론과 부딪힌다
`src/routes/` 는 **URL 경로**로 파일을 배치한다. 그런데 [[Feature-Sliced Design(FSD)]]처럼 **업무 도메인**으로 폴더를 나누는 방법론을 쓰면 `pages` 레이어와 이 디렉터리가 같은 자리를 놓고 겹친다. 어댑터를 두거나 `virtualRouteConfig`로 라우트 트리를 코드로 정의하는 두 갈래가 있는데, 각각의 대가는 [[FSD와 파일 기반 라우팅의 충돌]]에 정리했다.

## 3. loader와 데이터 로딩

라우트가 활성화되기 전에 데이터를 미리 받는다. **컴포넌트가 마운트되고 나서야 fetch가 시작되는 워터폴**을 없애는 게 목적이다.

```ts
export const Route = createFileRoute('/stores/$storeId/alerts')({
  validateSearch: searchSchema,
  loaderDeps: ({ search }) => ({ level: search.level, range: search.range }),
  loader: ({ context: { queryClient }, params, deps }) =>
    queryClient.ensureQueryData(alertListQuery(params.storeId, deps)),
  component: AlertsPage,
});
```

### loaderDeps를 빠뜨리는 함정
**loader는 기본적으로 search params를 의존성으로 보지 않는다.** `loaderDeps` 로 명시하지 않으면 필터를 바꿔도 loader가 다시 실행되지 않는다. 의도적인 설계지만(모든 search 변경마다 재로딩하면 낭비다) 처음엔 반드시 한 번 물린다.

### Query와 어떻게 나눌까
Router에도 자체 loader 캐시(`staleTime` · `gcTime` · `defaultPreload`)가 있어서 [[TanStack Query]]와 역할이 겹쳐 보인다. **둘을 각각 캐시로 쓰면 같은 데이터를 두 번 받는다.**

정석은 역할 분담이다.

| | 역할 |
| ----------- | ----------- |
| Router loader | **언제 요청을 시작할지** — 라우팅 시점에 미리 트리거 |
| Query | **캐시의 진실** — 저장·재검증·무효화 |

즉 loader에서 `queryClient.ensureQueryData()` 를 호출하고, 컴포넌트에서는 평범하게 `useQuery` 를 쓴다. 캐시는 한 군데(QueryClient)에만 있고, loader는 "미리 데우기" 역할만 한다. `queryClient` 는 router context로 주입한다.

```ts
const router = createRouter({ routeTree, context: { queryClient } });
```

`defaultPreload: 'intent'` 를 켜면 링크에 **마우스를 올리는 순간** loader가 돌아, 클릭 시점엔 이미 데이터가 있다.

## 4. React Router v7과의 차이

| | TanStack Router v1 | React Router v7 |
| ----------- | ----------- | ----------- |
| 지향 | **클라이언트 우선**, SSR은 Start로 | **SSR 우선**(framework mode), SPA 모드도 가능 |
| search params | **스키마 검증 + 타입 추론** | `URLSearchParams` — 직접 파싱 |
| 경로 타입 | 생성 파일 기반 완전 추론 | typegen 있음. search는 미포함 |
| 데이터 | `loader` + Query 조합 권장 | `loader`/`action` 내장, 뮤테이션까지 담당 |
| 캐시 | loader 캐시 + Query | 자체 재검증 모델 |
| 성숙도·생태계 | 작지만 빠르게 성장 | **압도적으로 크다** (구 Remix 계보) |

> 판단 기준은 하나다. **URL을 상태 저장소로 쓰는가?**
> 필터·정렬·시간범위를 URL에 담고 그 링크를 공유하는 대시보드라면 TanStack Router. **SSR·SEO·중첩 폼 액션**이 중심이면 React Router v7의 생태계가 낫다.

### 펫바디 대시보드의 선택
운영자가 "이 상황 좀 봐줘"라며 링크를 붙여넣는 일이 잦다. 구역·시간범위·알림 등급이 **URL에 있어야 하고, 새로고침해도 유지돼야** 한다. 로그인 뒤 화면이라 SEO는 무의미하고 [[서버 사이드 렌더링(SSR)]]의 명분도 없다([[싱글 페이지 애플리케이션(SPA)]]로 확정) → **search params 타입 안전이 유일한 결정 요인**이 됐다. 이 비교는 [[Vite React 템플릿 변형 비교]]에 자세히 적었다.

## 5. 단점 · 함정

### 단점
* **타입 컴파일이 무겁다.** 라우트가 수십 개를 넘어가면 IDE가 느려지고 `tsc` 시간이 눈에 띄게 는다. 타입 에러 메시지도 길고 읽기 어렵다
* **폴더 구조를 라우터가 결정한다.** `src/routes/` 배치가 곧 URL이라, 도메인 기준으로 폴더를 나누는 [[폴더 구조 전략]]과 충돌한다 → [[FSD와 파일 기반 라우팅의 충돌]]
* **생성 파일에 의존한다.** `routeTree.gen.ts` 가 낡으면 "분명 맞는 경로인데 타입 에러"가 난다. 플러그인이 도는 dev 서버를 띄우거나 재생성해야 한다. 커밋할지 gitignore할지 팀 규약이 필요
* **생태계가 작다.** 예제·블로그·스택오버플로 답변의 양이 React Router와 비교가 안 된다
* SSR은 Router 단독으로는 안 된다. TanStack Start가 필요하고 아직 RC 단계

### 함정
* **search가 자동 상속되지 않는다** (1절) — 가장 흔한 사고
* **`loaderDeps` 누락** — 필터를 바꿔도 loader가 안 돈다
* **search는 JSON 직렬화 가능해야 한다.** `Date`·`Map` 을 그대로 넣으면 안 되고 스키마에서 변환해야 한다. URL 길이 제한도 그대로 적용된다
* **`validateSearch` 가 throw하면 라우트 에러 화면**으로 간다. `.catch()` / `fallback()` 으로 필드별 복구를 걸어두는 게 안전하다
* **router 타입 등록을 빠뜨리면 추론이 전부 죽는다.** `declare module '@tanstack/react-router' { interface Register { router: typeof router } }` 한 번은 반드시
* **loader와 `useQuery` 를 각각 캐시로 쓰면 이중 요청**이 난다 (3절)
* 인증 리다이렉트는 `beforeLoad` 에서 `redirect()` 를 던진다. 컴포넌트 안에서 처리하면 이미 렌더가 시작된 뒤라 깜빡인다 → [[세션 vs JWT vs OAuth]]

## 정리

* 존재 이유는 **search params 타입 안전.** 다른 라우터는 여기가 전부 string이다
* `validateSearch` 에 스키마를 그대로 넘기고 **`.catch()` 로 필드별 복구**를 걸어라
* **search는 자동으로 이어지지 않는다.** `retainSearchParams` 를 쓸 것
* 파일 기반 라우팅은 **타입 추론 + 코드 스플리팅 자동화**가 이점이지 필수는 아니다
* loader는 **"언제 요청할지"**, [[TanStack Query]]는 **"캐시의 진실"**. 둘을 다 캐시로 쓰면 이중 요청
* 대가는 **타입 컴파일 비용과 작은 생태계.** URL을 상태로 쓰지 않는 앱이면 명분이 약하다

#펫바디
