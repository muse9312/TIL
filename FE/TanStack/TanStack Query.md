# TanStack Query
----

**서버 상태 전용 캐시 라이브러리.** 구 React Query. 상태 관리 도구가 아니라 **비동기 캐시 + 재검증 엔진**으로 이해해야 정확하다.

> 기준 버전: **v5** (React 어댑터 `@tanstack/react-query`). v4에서 이름이 많이 바뀌었으니 오래된 글을 볼 때 주의 (아래 8절)

푸는 문제는 [[서버 상태 vs 클라이언트 상태]]에 적은 그 목록이다 — 캐싱, 재검증, 중복 요청 제거, 로딩/에러, 경쟁 상태, 재시도.

```ts
const { data, isPending, isError } = useQuery({
  queryKey: ['alerts', storeId, { level }],
  queryFn: () => api.getAlerts(storeId, { level }),
});
```

## 1. staleTime vs gcTime — 가장 많이 헷갈리는 둘

이름이 비슷해서 섞이는데, **다루는 대상이 다르다.**

| | `staleTime` | `gcTime` |
| ----------- | ----------- | ----------- |
| 대상 | **데이터의 신선도** | **캐시 엔트리의 수명** |
| 기본값 | `0` (즉시 stale) | `5분` |
| 세는 시점 | 데이터를 받은 순간부터 | **관찰자가 0이 된 순간부터** |
| 지나면 | 재검증 트리거에서 다시 받아온다 | 캐시에서 **삭제**된다 |

```
fetch 성공
   │
   ├── fresh (staleTime 이내) ──▶ 트리거가 와도 재요청 안 함. 캐시 그대로
   │
   └── stale (staleTime 경과) ──▶ 트리거가 오면 백그라운드 재요청
                                   (그동안 낡은 데이터는 계속 보여준다)

컴포넌트 언마운트 (관찰자 0)
   │
   └── gcTime 카운트 시작 ──▶ 경과하면 캐시 삭제
                              (그 전에 다시 마운트되면 즉시 표시 + 재검증)
```

핵심 오해 둘.

* **`staleTime: 0` 이 "캐시를 안 쓴다"는 뜻이 아니다.** 캐시는 즉시 반환하고, **동시에** 백그라운드로 재요청한다(stale-while-revalidate). 화면이 비지 않는다
* **`gcTime` 은 관찰 중인 쿼리에는 아무 영향이 없다.** 화면에 떠 있는 동안은 5분이 지나도 안 지워진다

### 어떻게 정할까
`staleTime` 은 **"이 데이터가 몇 초쯤 낡아도 괜찮은가"** 로 정한다.

| 데이터 | staleTime | 이유 |
| ----------- | ----------- | ----------- |
| 매장 목록, 구역 마스터 | `5분` ~ `Infinity` | 거의 안 바뀐다. 매번 받을 이유가 없다 |
| 실시간 알림 카운트 | `0` ~ `10초` | 낡으면 바로 의미가 없다 |
| SSE로 델타가 오는 데이터 | **길게** (`Infinity` 도 가능) | 갱신은 푸시가 하고, 재조회는 복구용으로만 |

마지막 줄이 중요하다. **푸시 채널이 있으면 폴링성 재검증을 줄이는 게 맞다.** 안 그러면 같은 데이터를 두 경로로 계속 받는다.

## 2. 쿼리 키 설계

쿼리 키는 **캐시의 주소이자 의존성 배열**이다. 키가 바뀌면 다른 쿼리로 취급돼 새로 요청한다.

```ts
['alerts', storeId, { level, range }]
```

* 배열의 **순서는 의미가 있다**. 객체 안 **속성 순서는 무관**하다 (결정적으로 해시된다)
* `queryFn` 이 쓰는 값은 **전부 키에 들어가야 한다.** 빠뜨리면 필터를 바꿔도 옛 데이터가 나온다
* **넓은 것 → 좁은 것** 순으로 계층을 만든다. 무효화가 접두사 매칭이기 때문

```
['alerts']                                  ← 전체 알림 관련
['alerts', storeId]                         ← 특정 매장
['alerts', storeId, { level: 'urgent' }]    ← 특정 매장의 긴급만
```

`invalidateQueries({ queryKey: ['alerts', storeId] })` 하면 그 아래가 **전부** 무효화된다. 계층을 잘못 잡으면 무효화 범위를 원하는 대로 못 자른다.

### 키 팩토리로 묶어라
문자열을 손으로 쓰면 오타 하나로 캐시가 갈라진다. **키는 한 파일에 모으는 게 정석**이다.

```ts
export const alertKeys = {
  all:    ['alerts'] as const,
  store:  (id: string) => [...alertKeys.all, id] as const,
  list:   (id: string, f: AlertFilter) => [...alertKeys.store(id), f] as const,
};
```

v5의 `queryOptions()` 헬퍼를 쓰면 키와 `queryFn` 을 **한 덩어리로 묶어 타입까지 공유**할 수 있다. `useQuery` · `prefetchQuery` · `setQueryData` 가 같은 정의를 쓰게 되므로 어긋날 여지가 없다.

```ts
export const alertListQuery = (id: string, f: AlertFilter) =>
  queryOptions({ queryKey: alertKeys.list(id, f), queryFn: () => api.getAlerts(id, f) });
```

## 3. 재검증 트리거

기본적으로 **stale인 쿼리**가 다음 순간에 다시 받아온다.

| 트리거 | 옵션 | 기본 |
| ----------- | ----------- | ----------- |
| 새 컴포넌트가 마운트될 때 | `refetchOnMount` | `true` |
| **창에 포커스가 돌아올 때** | `refetchOnWindowFocus` | `true` |
| **네트워크가 재연결될 때** | `refetchOnReconnect` | `true` |
| 주기적으로 | `refetchInterval` | 꺼짐 |

포커스 재검증이 이 라이브러리의 인상적인 기본값이다. 탭을 갔다 오면 화면이 알아서 최신이 된다. 다만 **`staleTime: 0` + 포커스 재검증**이면 탭 전환마다 요청이 나가므로, 조용해야 하는 화면에서는 `staleTime` 을 올리는 게 먼저다(옵션을 끄는 것보다).

## 4. 무효화(invalidation)

"이 데이터는 이제 못 믿는다"고 **표시**하는 것. 삭제가 아니다.

```ts
queryClient.invalidateQueries({ queryKey: alertKeys.store(storeId) });
```

* 매칭된 쿼리를 **stale로 표시**하고
* 그중 **화면에 떠 있는(active) 쿼리는 즉시 재요청**한다
* 안 떠 있는 쿼리는 다음에 마운트될 때 받아온다

데이터는 지우지 않으므로 **화면이 비지 않는다.** 이게 `removeQueries` 나 `resetQueries` 와 다른 점이고, 대부분의 경우 정답이다.

```ts
// 뮤테이션 후: 서버가 진실이니 관련 캐시를 재검증한다
useMutation({
  mutationFn: (id: string) => api.ackAlert(id),
  onSettled: () => queryClient.invalidateQueries({ queryKey: alertKeys.all }),
});
```

> 무효화 범위를 넓게 잡는 것 자체는 나쁘지 않다. **정합성이 성능보다 먼저**고, active가 아닌 쿼리는 어차피 요청이 안 나간다.

## 5. 캐시 직접 쓰기 — setQueryData

서버 왕복 없이 캐시를 고치는 방법. **[[낙관적 업데이트(Optimistic Update)]]와 실시간 델타 반영의 기반**이다.

```ts
queryClient.setQueryData(alertKeys.list(storeId, filter), (old) =>
  old ? { ...old, items: upsertById(old.items, event) } : old,
);
```

* 업데이터는 **반드시 [[불변성(Immutability)]]을 지켜야 한다.** 기존 객체를 직접 고치면 리렌더가 안 일어난다
* `old` 가 `undefined` 일 수 있다 — 아직 안 받았거나 gc된 경우. **이때 새로 만들어 넣으면 안 된다.** 부분 데이터를 완성된 캐시로 착각하게 된다. 그냥 두고 재조회에 맡긴다
* 이 함수는 **동기적**이고 서버에 아무것도 안 보낸다. 진실은 여전히 서버에 있다

### 낙관적 업데이트

```ts
useMutation({
  mutationFn: api.ackAlert,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: alertKeys.all });   // ① 진행 중 요청 취소
    const prev = queryClient.getQueryData(key);                      // ② 스냅샷
    queryClient.setQueryData(key, (old) => markAcked(old, id));      // ③ 낙관적 반영
    return { prev };
  },
  onError: (_e, _v, ctx) => queryClient.setQueryData(key, ctx.prev), // ④ 롤백
  onSettled: () => queryClient.invalidateQueries({ queryKey: key }), // ⑤ 서버와 정합
});
```

①을 빠뜨리면 **진행 중이던 이전 요청의 응답이 낙관적 값을 덮어써서** 화면이 튄다. ⑤를 빠뜨리면 서버가 채운 값(확정 시각 등)과 어긋난 채 남는다. 판단 기준과 함정은 [[낙관적 업데이트(Optimistic Update)]]에 정리했다.

## 6. SSE 델타를 캐시에 반영하기 — 펫바디 패턴

관제 대시보드의 핵심 설계다. 실시간을 **한 채널로만 하면 안 된다.**

```
스냅샷 (REST + Query)  ──▶  현재 상태 전체   ← 최초 진입 · 재연결 직후
델타   (SSE)           ──▶  변경분만        ← 연결 유지 중
```

SSE만 쓰면 연결이 끊겼다 붙는 사이 **놓친 이벤트만큼 화면이 조용히 틀어진다.** 에러도 안 나고 값만 안 맞는다. 그래서 **복구 경로**가 반드시 있어야 한다.

```ts
const qc = useQueryClient();

useEffect(() => {
  const es = new EventSource('/api/stream', { withCredentials: true });

  // 델타 도착 → 캐시에 병합
  es.addEventListener('alert', (e) => {
    const ev = JSON.parse(e.data);
    qc.setQueryData(alertKeys.store(ev.storeId), (old) =>
      old ? { ...old, items: upsertById(old.items, ev) } : old,   // id 기준 upsert
    );
  });

  // 재연결 → 스냅샷 재조회로 유실 복구
  es.addEventListener('open', () => {
    qc.invalidateQueries({ queryKey: alertKeys.all });
  });

  return () => es.close();
}, [qc]);
```

포인트 넷.

* **`upsertById`.** append하면 SSE 델타와 재조회 결과, 낙관적 업데이트가 **같은 항목을 중복으로** 넣는다. 반드시 id 기준 병합
* **`open` 이벤트로 무효화.** `EventSource` 는 끊기면 브라우저가 알아서 재연결하고 성공 시 `open` 이 다시 발생한다. 최초 연결에서도 한 번 뜨는데, 그때 무효화는 무해하다
* **델타가 오는 쿼리는 `staleTime` 을 길게.** 갱신 책임이 푸시에 있으므로 포커스마다 재조회할 이유가 없다
* **고빈도 이벤트는 캐시에 넣지 않는다.** 초당 수십 건짜리 원시 스트림은 [[zustand]] 버퍼로 받고, 캐시에는 "확정된 목록"만 둔다 → [[상태 관리]]

> `setQueryData` 로 반영한 값은 **재조회가 오면 덮인다.** 이건 버그가 아니라 설계다. 서버가 늘 진실이고, 델타는 그때까지의 근사치다.

## 7. 함정

### 상태 플래그 (v5)
`status` 는 데이터 존재 여부, `fetchStatus` 는 요청 진행 여부로 **축이 분리돼 있다.**

| 플래그 | 의미 | 쓸 곳 |
| ----------- | ----------- | ----------- |
| `isPending` | **아직 데이터가 없다** (첫 로딩) | 스켈레톤 |
| `isFetching` | 요청이 진행 중 (배경 재검증 포함) | 상단 미세 인디케이터 |
| `isLoading` | `isPending && isFetching` | 대부분의 "로딩 중" |
| `isError` / `error` | 실패 | 에러 UI |

`isFetching` 으로 전체 스켈레톤을 띄우면 **재검증할 때마다 화면이 통째로 깜빡인다.** 매우 흔한 실수다.

### 그 외
* **`queryFn` 이 `undefined` 를 반환하면 안 된다.** 에러로 처리된다. 데이터 없음은 `null` 이나 빈 배열로
* **`enabled: false` 는 "로딩 안 함"이 아니다.** 이 쿼리는 `isPending` 상태로 영원히 머문다. 조건부 쿼리에서 스피너가 안 사라지는 원인
* **키에 안 넣은 값을 `queryFn` 에서 참조**하면 낡은 값을 계속 쓴다. 가장 흔한 캐시 버그
* **페이지 전환 시 깜빡임** — 키가 바뀌면 새 쿼리라 `isPending` 이 된다. `placeholderData: keepPreviousData` 로 이전 데이터를 유지
* **`select` 는 매 렌더 실행된다.** 무거운 변환이면 참조가 매번 바뀌어 리렌더가 는다 → [[메모이제이션(Memoization)]]
* **서버 상태가 아닌 걸 넣지 말 것.** 모달 열림 여부에 캐시 키를 만들 이유가 없다
* MSW로 개발할 때 **모킹 지연이 0이면** 로딩 UI의 문제를 못 잡는다. 의도적으로 지연을 넣어야 한다

## 8. v4에서 v5로 바뀐 것

오래된 블로그 글을 그대로 따라 하면 안 되는 부분들.

| v4 | v5 | 비고 |
| ----------- | ----------- | ----------- |
| `cacheTime` | **`gcTime`** | 이름이 실제 동작(가비지 컬렉션)에 맞게 바뀜 |
| `status: 'loading'` | **`status: 'pending'`** | |
| `isLoading` | **`isPending`** | |
| `isInitialLoading` | **`isLoading`** | 즉 `isLoading` 의 **의미가 바뀌었다** (`isPending && isFetching`) |
| `useQuery(key, fn, opts)` | **객체 하나만** — `useQuery({ queryKey, queryFn })` | 오버로드 제거 |
| `useQuery` 의 `onSuccess`/`onError`/`onSettled` | **제거됨** | `useMutation` 에는 남아 있다. 쿼리에서는 렌더 결과로 파생하거나 `useEffect` 로 |
| `keepPreviousData: true` | **`placeholderData: keepPreviousData`** | 함수를 넘긴다 |
| `invalidateQueries(key)` | **`invalidateQueries({ queryKey })`** | 객체 필터 형태로 통일 |
| 무한 쿼리 | **`initialPageParam` 필수** | |

`isLoading` 의 의미 변화가 제일 위험하다. **이름은 그대로인데 동작이 다르다.**

## 정리

* 상태 관리가 아니라 **비동기 캐시 + 재검증 엔진**
* `staleTime` = 데이터 신선도, `gcTime` = 캐시 엔트리 수명. **세는 시점이 다르다**
* `staleTime: 0` 은 캐시를 안 쓴다는 뜻이 **아니다** (stale-while-revalidate)
* 쿼리 키는 **의존성 배열**이다. `queryFn` 이 쓰는 값은 전부 넣고, 키 팩토리로 모을 것
* 무효화는 삭제가 아니라 **표시** — 화면이 비지 않는다
* 실시간은 **델타 `setQueryData` + 재연결 `invalidateQueries`.** 복구 경로 없이 푸시만 믿으면 조용히 틀어진다
* 델타 병합은 **반드시 id 기준 upsert**
* v5의 `isLoading` 은 **v4와 의미가 다르다**

#펫바디
