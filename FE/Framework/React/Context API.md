# Context API
----

props를 거치지 않고 **컴포넌트 트리 아래로 값을 내려보내는** React 내장 기능. prop drilling을 없애는 도구다.

이름 때문에 상태 관리 도구로 오해받지만, Context 자체는 **값을 저장하지 않는다.** 저장은 `useState` 나 다른 스토어가 하고, Context는 그 값을 **전달**만 한다. 정확히는 상태 관리가 아니라 **의존성 주입(DI)** 메커니즘이다.

## 1. 기본 사용

```tsx
const TenantContext = createContext<Tenant | null>(null);

function App() {
  return (
    <TenantContext.Provider value={tenant}>
      <Dashboard />
    </TenantContext.Provider>
  );
}

function StoreHeader() {
  const tenant = useContext(TenantContext);   // 몇 단계 아래든 바로 읽는다
}
```

React 19부터는 `<TenantContext>` 를 Provider로 바로 쓸 수 있고, `use(TenantContext)` 로도 읽는다.

### 커스텀 훅으로 감싸는 게 관례다

```tsx
export function useTenant() {
  const ctx = useContext(TenantContext);
  if (!ctx) throw new Error('useTenant must be used within TenantProvider');
  return ctx;
}
```

기본값을 `null` 로 두고 훅에서 던지면, **Provider를 깜빡한 실수가 런타임에 바로 드러난다.** 기본값을 그럴듯하게 채워두면 Provider 없이도 조용히 동작해서 버그를 늦게 발견한다.

## 2. 무엇에 쓰는가 — 자주 안 바뀌는 값

Context가 잘 맞는 건 **앱 전체가 알아야 하고, 거의 안 바뀌는 값**이다.

* 테마 (라이트/다크)
* 로그인 사용자 · 권한
* 현재 테넌트 / 매장 → [[멀티테넌시 데이터 모델]]
* i18n 로케일
* 라이브러리 클라이언트 인스턴스 (`QueryClient`, 로거, API 클라이언트)

마지막 항목이 Context의 가장 정직한 용법이다. **값이 아예 안 바뀌는 싱글턴을 주입**하는 것. [[TanStack Query]]의 `QueryClientProvider` 가 그렇게 동작한다.

## 3. 결정적 한계 — selector가 없다

Context를 상태 저장소로 쓰면 여기서 무너진다.

> **Provider의 `value` 가 바뀌면, `useContext` 로 구독하는 모든 컴포넌트가 예외 없이 리렌더된다.**
> 그 컴포넌트가 값의 **어느 부분을 쓰는지와 무관하게** 그렇다.

```tsx
// events가 1건 늘 때마다, tenant.name만 쓰는 헤더까지 리렌더된다
<AppContext.Provider value={{ tenant, theme, events }}>
```

React는 Context 값을 `Object.is` 로 비교해 "바뀌었는가"만 판단한다. **"바뀐 부분이 내가 쓰는 부분인가"를 물을 방법이 없다.** 이게 selector가 없다는 말의 의미다.

### 객체 리터럴을 그대로 넘기는 함정

```tsx
// ❌ 렌더할 때마다 새 객체 → 참조가 매번 달라 전원 리렌더
<AppContext.Provider value={{ tenant, setTenant }}>

// ✅ 참조 고정
const value = useMemo(() => ({ tenant, setTenant }), [tenant]);
<AppContext.Provider value={value}>
```

`useMemo` 를 빠뜨리면 **부모가 리렌더될 때마다 전 구독자가 리렌더된다.** Context 성능 문제의 대부분이 이 한 줄이다. ([[React Compiler]]를 켜면 자동으로 처리되는 부류의 실수이기도 하다 → [[메모이제이션(Memoization)]])

### 완화책과 그 한계

| 방법 | 효과 | 한계 |
| ----------- | ----------- | ----------- |
| `value` 를 `useMemo` 로 고정 | 불필요한 참조 변경 제거 | 값이 **진짜로** 바뀌면 여전히 전원 리렌더 |
| Context를 성격별로 쪼갬 (`ThemeContext` / `TenantContext`) | 리렌더 범위 축소 | Provider 중첩이 깊어짐. 근본 해결은 아님 |
| 값 Context / 디스패치 Context 분리 | 쓰기만 하는 컴포넌트는 리렌더 안 됨 | 읽는 쪽은 그대로 |
| 구독자를 `React.memo` 로 감쌈 | **효과 없음.** Context 갱신은 memo를 뚫는다 | — |

마지막 줄이 중요하다. `React.memo` 는 props 비교라서 **Context 구독에는 통하지 않는다.**

결국 Context 안의 값이 자주 바뀌면 답이 없다. 그때 필요한 게 selector를 지원하는 스토어, 즉 [[zustand]]다.

## 4. Context vs zustand — 어떻게 고를까

| 기준 | Context | [[zustand]] |
| ----------- | ----------- | ----------- |
| 값이 바뀌는 빈도 | **낮음** (로그인, 테마 전환) | 높음 (실시간 이벤트) |
| 구독 단위 | 전체 | 슬라이스 |
| 트리 밖에서 쓰기 | 불가 | `setState()` 로 가능 |
| 인스턴스 분리 | **자연스럽다** (Provider 단위) | 기본은 모듈 싱글턴 |
| 추가 의존성 | 없음 | 약 1KB |

> 판단 기준: **초당 여러 번 바뀌는가?** 그렇다면 Context는 안 된다.
> 반대로 **인스턴스가 여러 개여야 하는가?** 그렇다면 Context가 자연스럽다.

두 번째 줄이 자주 잊힌다. Context는 Provider 위치마다 값이 달라지므로 "같은 화면 두 개, 각각 다른 매장"이 공짜로 된다. 모듈 전역 zustand 스토어는 그게 안 된다. 실제로 zustand도 **인스턴스가 필요할 땐 `createStore` 를 Context로 내려주는 패턴**을 권한다 — 즉 둘은 대체재가 아니라 조합이다.

### 장점
* React 내장. 의존성 0
* Provider 경계로 **스코프가 명확**하고 인스턴스를 나누기 쉽다
* 테스트에서 Provider만 바꿔 끼우면 되므로 **모킹이 쉽다**
* 언마운트되면 값이 같이 사라진다 → [[전역 상태와 지역 상태]]

### 단점
* **selector 없음** — 값이 바뀌면 구독자 전원 리렌더
* `value` 참조 관리를 손으로 해야 함
* 성격이 다른 값을 담으려면 **Provider가 계속 중첩**된다
* 미들웨어·devtools·persist 같은 부가 기능이 없다
* React 트리 밖에서 접근 불가

## 정리

* Context는 상태 관리가 아니라 **값 전달(DI)** 이다. 저장은 다른 게 한다
* **거의 안 바뀌는 값**에 쓴다 — 테마, 사용자, 테넌트, 클라이언트 인스턴스
* `value` 는 반드시 **`useMemo` 로 참조를 고정**할 것
* 값이 바뀌면 **구독자 전원이 리렌더**되고 `React.memo` 로도 못 막는다
* 자주 바뀌는 값은 selector가 있는 [[zustand]]로. 인스턴스를 나눠야 하면 **둘을 조합**한다
