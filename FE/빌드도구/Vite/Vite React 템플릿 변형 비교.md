# Vite React 템플릿 변형 비교
----

`npm create vite@latest` 에서 React를 고르면 나오는 변형 목록.

> `create-vite` **9.2.0** 기준 (2026-09-01, 패키지 소스에서 변형 정의를 직접 확인). 아래 9개 항목과 순서가 그대로 일치하고, ↗ 4개도 여전히 유효하다.

```
● TypeScript
○ TypeScript + React Compiler
○ JavaScript
○ JavaScript + React Compiler
○ RSC
○ React Router v7   ↗ https://reactrouter.com
○ TanStack Router   ↗ https://tanstack.com/router
○ RedwoodSDK        ↗ https://rwsdk.com
○ Vike              ↗ https://vike.dev
```

**중요: ↗ 표시가 붙은 4개는 Vite 자체 템플릿이 아니다.** 해당 프레임워크의 CLI로 넘겨주는 링크라, 고르면 Vite 템플릿을 벗어나 그쪽 스캐폴딩이 실행된다(`npm create react-router@latest`, `npm create rwsdk@latest` 처럼 `customCommand` 를 그대로 실행하고 create-vite는 빠진다).

**RSC도 사실은 같은 부류다.** ↗ 표시는 없지만 로컬 템플릿이 아니라 `vitejs/vite-plugin-react` 저장소의 예제를 내려받는 명령으로 처리된다. 다만 Vite 조직 안이라 성격은 다르다. **진짜 로컬 템플릿은 위쪽 4개(TS / TS+Compiler / JS / JS+Compiler)뿐이다.**

## 1. Vite가 직접 관리하는 템플릿

| 항목 | 내용 |
| ----------- | ----------- |
| **TypeScript** | 순수 React [[싱글 페이지 애플리케이션(SPA)]] + TS. 라우터·[[상태 관리]]·서버 없음. 가장 얇은 베이스라인 |
| **TypeScript + React Compiler** | 위 + [[React Compiler]](빌드 타임 자동 [[메모이제이션(Memoization)]]) 활성화 |
| **JavaScript / JS + Compiler** | TS 미적용 버전 |
| **RSC** | React Server Components. 컴포넌트를 서버에서 실행·스트리밍. **Node 런타임이 상시 떠 있어야 함** |

### React Compiler란

> React Compiler **1.0** / `babel-plugin-react-compiler@1.0.0` 기준 (2026-09-01 확인). [v1.0 릴리스](https://react.dev/blog/2025/10/07/react-compiler-1) · [설치 문서](https://react.dev/learn/react-compiler/installation)

`useMemo` / `useCallback` / `React.memo` 를 손으로 붙이지 않아도 **빌드 타임에 자동으로 [[메모이제이션(Memoization)]]**을 삽입한다. 자세한 건 [[React Compiler]].

**2025-10-07에 1.0 안정판이 나왔다.** 실험 딱지는 떨어진 상태다. "React 19와 함께 나온 것"으로 묶이지만 실제로는 **React 17 이상 호환**이고, 17/18에서는 컴파일러 설정에 최소 타깃을 지정하고 `react-compiler-runtime` 을 의존성에 추가하면 쓸 수 있다.

- 장점 — 메모 관리 코드가 사라짐. 리렌더 최적화를 사람이 신경 쓸 일이 줄어듦
- 단점 — **빌드가 느려진다.** create-vite가 이 변형에 붙여주는 README에 대놓고 "This will impact Vite dev & build performances" 라고 적혀 있다
- 단점 — Rules of React를 어기면 **조용히 최적화를 포기(bail out)** 한다. 그래서 린트를 같이 켜야 하는데, **별도 패키지가 아니라 `eslint-plugin-react-hooks` 의 `recommended` 프리셋**에 컴파일러 기반 룰이 들어갔다. 이미 쓰고 있다면 버전만 올리면 됨
- **나중에 켜고 끄는 게 쉽다.** 스캐폴딩 시점에 고민할 필요 없음
	- create-vite도 컴파일러 변형을 **별도 템플릿으로 두지 않는다.** 기본 react 템플릿에 devDependency 3개(`babel-plugin-react-compiler`, `@rolldown/plugin-babel`, `@babel/core`)를 넣고 `vite.config` 를 고치는 **후처리**로 만든다. 되돌리는 비용도 딱 그만큼

"babel 플러그인 한 줄"이라는 설명은 이제 정확하지 않다. Vite 6 이상에서 권장하는 형태는 이렇다.

```ts
import react, { reactCompilerPreset } from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [
    react(),
    babel({ presets: [reactCompilerPreset()] }),
  ],
});
```

Babel을 아예 태우기 싫으면 `@vitejs/plugin-react` 옵션에 `compiler: true` 를 주는 **네이티브(Rust) 경로**도 있다. 다만 **실험 단계**라 프로덕션 판단은 따로 해야 한다.

## 2. 외부 프레임워크로 넘어가는 항목

| 항목 | 내용 | 성격 |
| ----------- | ----------- | ----------- |
| **React Router v7** | 구 Remix. 파일 기반 라우팅 + `loader`/`action` 데이터 계층 + [[서버 사이드 렌더링(SSR)]](끌 수 있음) | 풀스택 [[프레임워크]] |
| **[[TanStack Router]]** | 타입 안전 라우팅이 핵심. 경로 파라미터는 물론 **[[쿼리스트링(Query String)]]까지 타입 추론**. TanStack Start와 묶이면 풀스택 | [[라이브러리]](+선택적 풀스택) |
| **RedwoodSDK** | Cloudflare Workers 전제의 RSC 기반 풀스택. 배포 타깃이 CF에 묶임 | 니치 |
| **Vike** | 구 vite-plugin-ssr. [[서버 사이드 렌더링(SSR)]]/[[정적 사이트 생성(SSG)]]를 저수준으로 직접 조립. "직접 만드는 Next.js" | 메타 [[프레임워크]] |

## 3. 어떻게 고를까 — [[싱글 페이지 애플리케이션(SPA)]]인가 [[서버 사이드 렌더링(SSR)]]인가

이 판단이 9개를 2~3개로 줄여준다.

| 조건 | 함의 |
| ----------- | ----------- |
| 로그인 게이트 뒤의 내부 도구 | SEO 불필요 → **[[서버 사이드 렌더링(SSR)]]의 최대 명분이 사라짐** |
| 데이터가 실시간(SSE/WebSocket) | 서버가 초기 HTML을 그려줘도 1초 뒤 다 갈아엎힘 → **SSR이 오히려 손해** |
| 백엔드가 이미 별도 서버로 분리 | 프론트에 Node 서버를 또 띄우면 **배포 대상이 2개** → 운영 복잡도 증가 |
| 인증이 httpOnly 쿠키 + 서버 세션 | 정적 SPA + 쿠키로 충분. SSR 세션 전달 이슈를 만들 이유 없음 |

반대로 [[서버 사이드 렌더링(SSR)]]/RSC가 값을 하는 경우
- 검색 노출이 필요한 공개 페이지 (마케팅, 상세 페이지)
- 초기 로딩 성능이 매출에 직결 (커머스)
- 데이터가 대체로 정적이고 서버에서 확정 가능

### 라우터 템플릿을 고르지 않아도 되는 이유
React Router v7 / TanStack Router 템플릿은 **SSR 서버 엔트리까지 깔아준다.** 라우터 "기능"만 필요한 거라면 순수 SPA로 시작해서 **라이브러리로 붙이는 게 통제하기 쉽다.**

```bash
npm create vite@latest my-app -- --template react-ts
npm i @tanstack/react-router
npm i -D @tanstack/router-plugin
```

## 4. 관제 대시보드에 적용한 결과

펫바디 B2B 관제 대시보드는 위 4개 조건에 전부 해당 → **`TypeScript` (순수 SPA)** 선택.
RSC · RedwoodSDK · Vike는 서버 런타임 전제라 탈락. 백엔드가 이미 Fastify로 따로 있는데 프론트에 서버를 또 둘 이유가 없다.

붙인 라이브러리
- `@tanstack/react-router` — 라우팅 → [[TanStack Router]]
- `@tanstack/react-query` — 서버 상태 → [[TanStack Query]]
- `zustand` — 실시간 이벤트 버퍼 → [[zustand]]
- `recharts` — 차트
- `hls.js` — 영상

**[[TanStack]] 중 Router를 고른 이유**: 대시보드는 필터 상태(구역, 시간범위, 알림 등급)를 **URL [[쿼리스트링(Query String)]]에 담아야** 한다. 운영자가 "이 상황"을 링크로 공유하거나 새로고침해도 유지돼야 하니까. TanStack Router는 search params를 스키마로 검증하고 타입 추론까지 해줘서 이 요구와 정확히 맞는다. React Router v7도 되지만 search params는 직접 파싱해야 한다.

## 5. 정리

- ↗ 항목은 **다른 CLI로 나가는 문**이다. Vite 템플릿이 아님
- **[[서버 사이드 렌더링(SSR)]]이 필요한지부터 판단**하면 선택지가 확 준다. 로그인 뒤 실시간 대시보드면 거의 항상 [[싱글 페이지 애플리케이션(SPA)]]
- 라우터는 템플릿으로 고르지 말고 **라이브러리로 붙이는 편**이 통제하기 쉽다
- [[React Compiler]]는 언제든 켤 수 있으니 **스캐폴딩 시점에 고민하지 말 것**

이 SPA가 실시간 데이터를 받는 방법은 [[SSE vs WebSocket]].

#펫바디
