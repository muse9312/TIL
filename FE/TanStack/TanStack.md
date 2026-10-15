# TanStack
----

Tanner Linsley가 만든 **헤드리스 · 타입 안전 · 프레임워크 중립** 유틸리티 [[라이브러리]] 모음. React Query(현 TanStack Query)에서 출발해 라우팅·테이블·폼·가상화로 넓어졌다.

이름의 "Stack"이 오해를 부르는데, **하나의 프레임워크가 아니라 서로 독립적인 라이브러리 묶음**이다. 필요한 것만 골라 쓰고, 하나만 써도 된다.

## 1. 구성원

| 라이브러리 | 하는 일 | 대체하는 것 |
| ----------- | ----------- | ----------- |
| **[[TanStack Query]]** | 서버 상태 캐싱·재검증·동기화 | 직접 만든 `useEffect` + fetch |
| **[[TanStack Router]]** | 타입 안전 라우팅 + URL 상태 | React Router |
| **TanStack Table** | 헤드리스 데이터그리드 (정렬·필터·그룹·페이징 로직) | AG Grid, MUI DataGrid |
| **TanStack Form** | 타입 안전 폼 상태 | react-hook-form, Formik |
| **TanStack Virtual** | 리스트 가상화 (보이는 것만 렌더) | react-window |
| TanStack Store | 프레임워크 중립 반응형 스토어 | zustand, nanostores |
| TanStack Pacer | 디바운스·스로틀·배칭 | lodash.debounce |
| TanStack Start | Router 기반 풀스택 [[프레임워크]] | Next.js, Remix |
| TanStack DB | 로컬 우선 동기화 클라이언트 스토어 | — |

성숙도 차이가 크다. **Query · Router · Table · Form · Virtual은 안정판**이고, Start는 RC, DB는 베타, Store는 알파 단계다(2026 중반 기준). 실무 도입 시 이 차이를 반드시 확인해야 한다.

## 2. 공통 철학 셋

### 헤드리스 — 로직만 주고 마크업은 안 준다
TanStack Table은 `<table>` 을 그려주지 않는다. **정렬 상태, 필터링된 행, 페이지 계산 같은 "로직"만** 주고 렌더는 전부 사용자 몫이다.

* **장점** — 디자인 시스템(Tailwind + shadcn/ui 등)과 충돌하지 않는다. 컴포넌트 라이브러리를 도입할 때 가장 큰 비용인 "우리 디자인에 맞추기"가 사라진다. 접근성·마크업 구조를 내가 통제한다
* **단점** — **처음 화면을 띄우기까지가 느리다.** AG Grid는 한 줄이면 표가 나오는데 여기선 `<thead>` 부터 직접 짜야 한다. 프로토타이핑엔 불리하고, 팀에 마크업 규약이 없으면 파일마다 표가 다르게 생긴다

### 타입 안전 — 추론이 세일즈 포인트다
타입 "지원"이 아니라 **추론**이다. [[TanStack Router]]가 경로와 [[쿼리스트링(Query String)]]에서 타입을 뽑아내고, Query가 `queryFn` 반환 타입에서 `data` 타입을 뽑는다.

* **단점** — 타입이 복잡해서 **에러 메시지가 길고 읽기 어렵다.** 타입 추론이 꼬이면 원인 찾기가 힘들고, IDE가 느려지는 경우도 있다. TS 버전 요구사항도 비교적 높다

### 프레임워크 중립 — 코어는 바닐라
`@tanstack/query-core` 같은 코어에 로직이 있고, React·Vue·Svelte·Solid·Angular 어댑터가 얇게 감싼다.

* **장점** — 개념을 한 번 배우면 프레임워크를 넘어 재사용된다. React 트리 밖에서도 쓸 수 있다(`queryClient.getQueryData()`)
* **단점** — 어댑터마다 **성숙도·문서 품질이 다르다.** Router와 Start는 사실상 React·Solid 중심

## 3. 어디까지 쓸까

전부 쓸 필요는 없다. 라이브러리마다 대체재와의 비교가 다르다.

| | 채택 판단 |
| ----------- | ----------- |
| Query | **거의 무조건.** 서버 상태를 직접 관리할 이유가 거의 없다 → [[서버 상태 vs 클라이언트 상태]] |
| Router | URL을 상태로 쓰는 앱이면 강하다. 반대로 SSR·파일 규약 생태계는 React Router v7이 두텁다 |
| Table | 표 요구사항이 복잡하고 디자인을 통제해야 하면 유리 |
| Form | 성숙도·생태계는 아직 react-hook-form이 앞선다. **굳이 통일하려고 갈아탈 이유는 약하다** |
| Virtual | 수천 행 이상을 그릴 때만 |
| Store | [[zustand]]가 이미 안정적이고 생태계가 크다 |

> "TanStack으로 통일한다"는 목표 자체는 근거가 아니다. 각 라이브러리는 **독립적으로 대체재와 비교**되어야 한다.

### 펫바디 대시보드의 선택
Query + Router만 쓰고, 클라이언트 전역 상태는 [[zustand]], 폼은 react-hook-form + zod로 갔다.

* **Router를 고른 이유** — 필터(구역·시간범위·알림 등급)를 URL에 담아야 하는데, search params를 **스키마로 검증하고 타입 추론까지** 해주는 건 여기뿐 → [[쿼리스트링(Query String)]]
* **Store 대신 zustand** — Store는 알파고, SSE 이벤트 버퍼는 이미 zustand로 충분
* **Form 대신 react-hook-form** — 이미 zod 리졸버 생태계가 있고 갈아탈 이득이 없다

전체 스택 결정 과정은 [[Vite React 템플릿 변형 비교]]에 있다.

## 정리

* 프레임워크가 아니라 **독립 라이브러리 묶음.** 하나씩 골라 쓴다
* 공통 철학은 **헤드리스 · 타입 추론 · 프레임워크 중립**
* 헤드리스의 대가는 **초기 구현량**이다. 로직은 주지만 화면은 안 준다
* 성숙도가 제각각이다 — **Query/Router/Table/Form/Virtual은 안정, Start/DB/Store는 아직**
* "통일"을 명분으로 도입하지 말고 **각각을 대체재와 비교**할 것

#펫바디
