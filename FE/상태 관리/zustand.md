# zustand
----

훅 하나로 쓰는 **가벼운 전역 스토어.** 독일어로 "상태(state)". Provider도, 액션 타입도, 보일러플레이트도 없다.

핵심 차별점은 하나로 요약된다. **selector 기반 구독** — 컴포넌트가 스토어 전체가 아니라 **자기가 쓰는 조각만** 구독한다. [[Context API]]가 못 하는 게 정확히 이것이다.

## 1. 기본 사용

```ts
import { create } from 'zustand';

type EventStore = {
  events: AlertEvent[];
  push: (e: AlertEvent) => void;
  clear: () => void;
};

export const useEventStore = create<EventStore>((set, get) => ({
  events: [],
  push: (e) => set((s) => ({ events: [...s.events, e].slice(-500) })),
  clear: () => set({ events: [] }),
}));
```

읽을 때는 **반드시 selector를 준다.**

```ts
const events  = useEventStore((s) => s.events);   // events 바뀔 때만 리렌더
const push    = useEventStore((s) => s.push);     // 함수는 참조가 고정 → 영원히 안 바뀜
```

```ts
// ❌ 전체 구독 — 스토어의 아무 값이나 바뀌면 리렌더
const { events, push } = useEventStore();
```

`set` 은 **얕게 병합**한다. 리듀서처럼 전체 객체를 다시 만들 필요가 없다. (`set(partial, true)` 로 하면 교체)

### 컴포넌트 밖에서 읽고 쓰기
스토어가 훅과 분리돼 있어서 React 트리 바깥에서도 쓸 수 있다. **SSE 핸들러에서 스토어에 직접 쓰는** 게 이래서 가능하다.

```ts
const es = new EventSource('/api/stream');
es.addEventListener('alert', (e) => {
  useEventStore.getState().push(JSON.parse(e.data));   // 훅 아님
});
```

`getState()` 로 읽고 `setState()` 로 쓰고 `subscribe()` 로 관찰한다. 이게 [[전역 상태와 지역 상태]]에서 말한 "트리 밖에서 쓰기가 들어온다"의 실제 모습이다.

## 2. selector와 리렌더 — 실시간 UI의 핵심

selector가 반환한 값이 **이전 값과 `Object.is` 로 같으면 리렌더하지 않는다.** 그래서 새 객체·새 배열을 만들어 반환하면 selector를 써도 매번 리렌더된다.

```ts
// ❌ 매 렌더 새 배열 → 참조가 늘 달라 항상 리렌더
const urgent = useEventStore((s) => s.events.filter((e) => e.level === 'urgent'));
```

해법은 둘이다.

**a. 스토어에 파생 상태를 미리 만들어 둔다** (권장)

```ts
push: (e) => set((s) => ({
  events: [...s.events, e].slice(-500),
  byLevel: { ...s.byLevel, [e.level]: [...(s.byLevel[e.level] ?? []), e].slice(-50) },
  buckets: mergeIntoMinuteBucket(s.buckets, e),   // 차트용 1분 집계
})),
```

**b. `useShallow` 로 얕은 비교를 건다**

```ts
import { useShallow } from 'zustand/react/shallow';
const { urgent, warn } = useEventStore(useShallow((s) => ({ urgent: s.byLevel.urgent, warn: s.byLevel.warn })));
```

> ⚠️ **v5의 함정** — v4에서는 객체를 반환하는 selector에 `useShallow` 를 안 쓰면 "리렌더가 좀 많은" 정도였다. **v5에서는 무한 렌더 루프(`Maximum update depth exceeded`)로 터진다.** 객체·배열을 만들어 반환할 거면 `useShallow` 는 선택이 아니다.

### 펫바디 대시보드의 구독 구조
초당 여러 건의 이벤트가 들어오는데 전체 배열을 구독하면 **이벤트 하나에 대시보드 전체가 리렌더**된다.

```
SSE  ──▶  useEventStore
              │
   ┌──────────┼───────────┐
   ▼          ▼           ▼
알람 피드   재원 현황    추이 차트
byLevel     occupancy    buckets
(등급별      (개체 상태    (1분 집계
 최근 N건)    변화만)       버킷만)
```

차트가 원본 배열이 아니라 **1분 집계 버킷**을 구독하는 게 중요하다. 이벤트 1건마다 차트를 다시 그릴 이유가 없고, 버킷은 1분에 한 번만 참조가 바뀐다.

이건 [[메모이제이션(Memoization)]]으로 덮을 문제가 아니다. **구독 범위를 좁히는 게 먼저**고, 메모는 그 다음이다.

## 3. Context API와의 비교

| | [[Context API]] | zustand |
| ----------- | ----------- | ----------- |
| selector | **없음** | 있음 |
| 값이 바뀌면 | 구독자 **전원** 리렌더 | 해당 슬라이스 구독자만 |
| Provider | 필요 | 불필요 (원하면 가능) |
| 트리 밖 접근 | 불가 | `getState()` / `setState()` |
| 미들웨어 | 없음 | persist · immer · devtools |
| 번들 | 0 (내장) | 약 1KB |

Context는 "값을 트리 아래로 전달하는 **DI 메커니즘**"이지 상태 관리 도구가 아니다. 저장소로 쓰려면 selector가 필요하고, 그걸 붙인 게 zustand다.

## 4. 미들웨어

`create()` 를 감싸는 함수들. **감싸는 순서가 동작을 바꾼다.**

```ts
export const useUiStore = create<UiStore>()(
  devtools(                    // 가장 바깥: 모든 변경을 로깅
    persist(                   // 중간: 최종 상태를 저장
      immer((set) => ({        // 가장 안쪽: set 안에서 mutable 문법 허용
        filters: { level: 'all', zone: null },
        setLevel: (level) => set((s) => { s.filters.level = level; }),
      })),
      { name: 'ui', partialize: (s) => ({ filters: s.filters }) },
    ),
  ),
);
```

| 미들웨어 | 하는 일 | 주의할 점 |
| ----------- | ----------- | ----------- |
| `persist` | localStorage 등에 저장·복원 | **하이드레이션 타이밍** — 첫 렌더에는 기본값이 보인다. `onRehydrateStorage` / `hasHydrated` 로 처리. 저장 형식이 바뀌면 `version` + `migrate` 필요. `partialize` 로 저장 범위를 좁힐 것 |
| `immer` | `set` 안에서 mutable하게 써도 [[불변성(Immutability)]]을 지켜줌 | 프록시 오버헤드. 얕은 갱신엔 과함 |
| `devtools` | Redux DevTools 연결, 시간여행 | **프로덕션에선 끌 것.** 액션 이름을 안 주면 로그가 전부 `anonymous` |
| `subscribeWithSelector` | 트리 밖 `subscribe` 에 selector 사용 | 안 감싸면 `subscribe` 는 전체 변경만 받는다 |

> `persist` 에 서버에서 온 데이터를 넣지 말 것. 낡은 캐시를 localStorage에 영구 박제하는 셈이다. 그건 [[TanStack Query]]의 persister가 만료 관리까지 해서 하는 일이다.

## 5. 언제 과한가

### 장점
* 보일러플레이트가 거의 없다 — 스토어 정의가 곧 타입
* **selector 구독**으로 리렌더를 자연스럽게 좁힌다
* 트리 밖에서 읽고 쓸 수 있다 — 실시간·이벤트 소스와 궁합이 좋다
* 번들이 작고 React 의존이 얇다 (vanilla 스토어로 분리 가능)

### 단점 · 함정
* **너무 쉬워서 다 넣게 된다.** 지역이면 충분한 걸 전역화하는 진입장벽이 사실상 0 → [[전역 상태와 지역 상태]]
* **모듈 전역 싱글턴이다.** 스토어를 모듈 스코프에 만들면 앱 전체에 인스턴스가 하나뿐이다. 같은 화면을 두 개 띄우거나 테스트를 병렬로 돌리면 상태가 샌다. 필요하면 `createStore` + Context로 인스턴스를 나눈다 → [[싱글톤 패턴(Singleton pattern)]]
* **리셋을 직접 해야 한다.** 언마운트해도 값이 남는다. 로그아웃·매장 전환 시 `reset()` 호출을 빠뜨리면 이전 테넌트의 데이터가 보인다 → [[멀티테넌시 데이터 모델]]
* **미들웨어를 겹치면 타입이 어려워진다.** `create<T>()(...)` 의 커링 형태를 지켜야 추론이 산다
* 액션 추적이 Redux보다 약하다. 큰 팀에서 상태 전이가 복잡하면 Redux Toolkit의 규율이 나을 수 있다

### 안 써도 되는 경우
* 상태가 한 화면 안에서만 산다 → `useState` / `useReducer`
* 원본이 서버에 있다 → [[TanStack Query]]. **zustand에 서버 데이터를 넣는 게 가장 흔한 오용**이다 → [[서버 상태 vs 클라이언트 상태]]
* 공유·북마크돼야 한다 → [[쿼리스트링(Query String)]]
* 로그인 사용자·테마처럼 거의 안 바뀐다 → [[Context API]]로 충분

## 정리

* **selector로 조각만 구독**하는 게 존재 이유다. 전체 구조분해는 안 쓰는 것과 같다
* 객체·배열을 반환하는 selector에는 **`useShallow` 필수** (v5는 안 쓰면 무한 루프)
* 파생 상태(등급별 · 집계 버킷)는 **스토어 쓰기 시점에 미리 만들어 두는 게** selector에서 계산하는 것보다 낫다
* 트리 밖에서 쓸 수 있다는 게 **SSE 같은 외부 이벤트 소스와 맞는 이유**
* 미들웨어는 **감싸는 순서가 의미를 바꾼다** — `devtools(persist(immer(...)))`
* 가장 흔한 오용은 **서버 상태를 여기에 넣는 것**

#펫바디
