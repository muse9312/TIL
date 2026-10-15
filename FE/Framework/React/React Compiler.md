# React Compiler
----

**빌드 타임에 자동으로 메모이제이션을 삽입해주는** React 공식 컴파일러. React 19와 함께 나왔다.

`useMemo` · `useCallback` · `React.memo` 를 손으로 붙이던 일을 컴파일러가 대신한다.

```jsx
// 내가 쓰는 코드
function AlertFeed({ events, level }) {
  const filtered = events.filter(e => e.level === level);
  return <List items={filtered} />;
}

// 컴파일러가 만드는 코드 (개념적으로)
function AlertFeed({ events, level }) {
  const $ = useMemoCache(2);
  let filtered;
  if ($[0] !== events || $[1] !== level) {
    filtered = events.filter(e => e.level === level);
    $[0] = events; $[1] = level; $[2] = filtered;
  } else {
    filtered = $[2];
  }
  return <List items={filtered} />;
}
```

## 왜 나왔나

[[메모이제이션(Memoization)]] 을 사람이 손으로 관리할 때의 문제
* 어디에 붙여야 하는지 판단이 어렵다
* **의존성 배열을 빠뜨리면** 낡은 값을 붙잡는다 (stale closure)
* 로직보다 최적화 코드가 눈에 먼저 들어온다
* 컴포넌트를 리팩터링하면 메모 구조가 같이 깨진다

컴파일러는 코드를 정적 분석해 **어떤 값이 어떤 입력에 의존하는지** 스스로 계산한다. 사람보다 정확하고 빠뜨리지 않는다.

### 장점
* `useMemo`/`useCallback` 을 거의 안 써도 됨 → **코드가 로직만 남는다**
* 의존성 배열 실수가 원천적으로 사라짐
* 사람이 놓치는 지점까지 메모이제이션 — 대개 손으로 한 것보다 촘촘하다
* 도입/제거가 **babel 플러그인 한 줄** — 언제든 켜고 끌 수 있다

### 단점
* **빌드가 느려진다** — 정적 분석 비용
* **Rules of React를 어기면 조용히 포기(bail out)한다.** 에러가 아니라 "그 컴포넌트만 최적화 안 함"이라 눈치채기 어렵다
* 생성된 코드가 디버깅 시 낯설다
* 모든 병목을 해결해주지 않는다 — 구조적 문제는 그대로

## Rules of React — 지켜야 컴파일러가 일한다

컴파일러는 **컴포넌트가 순수하다**는 전제 위에서 동작한다.

* 렌더 중에 props·state를 **변경하지 않는다**
* 렌더 중에 외부 변수를 읽거나 쓰지 않는다
* 훅은 조건문·반복문 안에서 호출하지 않는다
* 같은 입력이면 같은 JSX를 반환한다

이걸 검사하려면 **`eslint-plugin-react-hooks` 를 같이 써야 한다.** 컴파일러가 bail out한 컴포넌트를 알려주는 규칙이 들어 있다. 컴파일러만 켜고 린트를 안 켜면 "켰는데 효과가 없는" 상태를 모른 채 지나간다.

## 켤까 말까

**켜도 되는 경우** — 새 프로젝트, Rules of React를 지키는 코드베이스
**신중할 경우** — 레거시가 많아 순수하지 않은 컴포넌트가 섞인 프로젝트. 이때는 bail out이 많아 효과가 반감된다

> 스캐폴딩 시점에 고민할 문제가 아니다. 플러그인 한 줄이라 나중에 켜도 되고, **진짜 성능 병목은 대개 메모이제이션이 아니라 [[상태 관리]] 구조**(구독 범위를 얼마나 좁혔는가)에서 온다. 구조를 먼저 잡고, 프로파일링한 뒤 켜는 게 순서다.

## 관련

* [[메모이제이션(Memoization)]] — 컴파일러가 자동화하는 대상
* [[상태 관리]] — 컴파일러로 해결되지 않는 층위
