# 커밋 컨벤션 강제 도구(husky · lint-staged · commitlint)
----

[[Git 훅(Git Hooks)]]을 팀 전체에 배포하고(husky), 스테이징된 파일만 검사하고(lint-staged), 커밋 메시지 형식을 검사하는(commitlint) 세 도구의 조합.

셋은 경쟁 관계가 아니라 **역할이 완전히 갈린다.** 이름만 보면 겹쳐 보여서 흔히 헷갈리는데, 하나가 다른 하나를 대신하지 못한다.

> 기준 버전 (2026-09-01 npm 레지스트리 확인)
> `husky` **9.1.7** (2024-11-18 배포, node ≥18) · `lint-staged` **17.4.1** (2026-08-27, node ≥22.22.1) · `@commitlint/cli` **21.2.2** (2026-08-13, node ≥22.12.0)
> 출처: https://typicode.github.io/husky/ · https://commitlint.js.org/ · https://github.com/lint-staged/lint-staged

## 1. 역할 분담

| 도구 | 하는 일 | 안 하는 일 |
| ----------- | ----------- | ----------- |
| **husky** | 훅 스크립트를 **버전 관리 폴더에 두고 git 이 그걸 보게 만든다** | 검사 자체는 안 한다 |
| **lint-staged** | **스테이징된 파일 목록만** 뽑아서 명령에 넘긴다 | 린터가 아니다 |
| **commitlint** | **커밋 메시지**를 규칙과 대조한다 | 코드는 안 본다 |

```
git commit
   │
   ├─ pre-commit  ─→ husky 가 실행 ─→ lint-staged ─→ eslint · prettier (스테이징된 파일만)
   │
   └─ commit-msg  ─→ husky 가 실행 ─→ commitlint --edit $1
```

husky 는 **배달부**고, 실제 판단은 뒤의 둘이 한다.

## 2. husky — v9 기준 설정

**v4 → v9 사이에 설정 방식이 여러 번 바뀌었고, 인터넷의 예제 대부분이 옛 버전이다.** 버전을 확인하지 않고 붙여 넣으면 안 된다.

```bash
npm install --save-dev husky
npx husky init
```

`husky init` 이 하는 일은 둘이다 — **`.husky/pre-commit` 스크립트를 만들고, `package.json` 의 `prepare` 스크립트를 갱신한다.**

```json
// package.json
{ "scripts": { "prepare": "husky" } }
```

`prepare` 는 npm 이 `npm install` 후 자동으로 실행하는 라이프사이클 스크립트다. 즉 **팀원이 클론하고 `npm install` 만 하면 훅이 걸린다** — `.git/hooks` 가 전파되지 않는 문제(→ [[Git 훅(Git Hooks)]])를 이렇게 푼다.

훅 파일은 **그냥 셸 스크립트**다.

```sh
# .husky/pre-commit
npx lint-staged
```

```sh
# .husky/commit-msg
npx --no -- commitlint --edit $1
```

### 옛 버전과 뭐가 다른가

| 버전 | 설정 위치 | 형태 |
| ----------- | ----------- | ----------- |
| v4 | `package.json` 의 `husky.hooks` 객체 / `.huskyrc` | JSON 설정 |
| v8 | `.husky/` 파일 | 파일 맨 위 **두 줄이 필수** |
| **v9** | `.husky/` 파일 | **명령만.** 그 두 줄은 **폐기됨** |

v8 시절 모든 훅 파일 맨 위에 붙이던 이 두 줄이 v9 에서 **폐기됐다.**

```sh
#!/usr/bin/env sh                    # ← 폐기. 지울 것
. "$(dirname -- "$0")/_/husky.sh"    # ← 폐기. 지울 것
```

husky 릴리스 노트가 명시한다 — v9.1.2부터 자동 제거 대신 경고를 띄우고, **이 줄이 남아 있는 훅은 v10.0.0 에서 실패한다.** 블로그 예제를 복사하다 이 두 줄을 같이 가져오는 게 지금 가장 흔한 실수다.

v4 에서 올라오는 경우 인자 전달도 바뀌었다. `HUSKY_GIT_PARAMS` 환경변수가 사라지고 **네이티브 인자 `$1`** 을 쓴다.

```sh
# v4:  "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
# v9:  commitlint --edit $1
```

### 끄는 방법과 CI

```bash
HUSKY=0 git commit ...        # 이번만
export HUSKY=0                # 리베이스·머지 동안
```

CI·Docker 에서는 환경변수 `HUSKY=0` 을 주는 게 공식 권장이다. `devDependencies` 없이 설치하는 환경에서는 `prepare` 가 실패하므로 `"prepare": "husky || true"` 로 두거나, `.husky/install.mjs` 를 만들어 `NODE_ENV=production` 이나 `CI=true` 면 건너뛰게 한다.

### GUI 에서 `command not found` 가 날 때

nvm·fnm·volta 로 node 를 설치했으면 GUI 클라이언트에서 훅이 깨진다. GUI 는 셸 초기화 파일을 읽지 않아 `PATH` 에 node 경로가 없기 때문이다.

husky 는 **훅 실행 전에 `~/.config/husky/init.sh` 를 source 한다.** 여기에 버전 매니저 초기화를 넣으면 해결된다. 이건 저장소가 아니라 **각자 로컬에 있는 파일**이라, 팀원마다 한 번씩 해줘야 하는 온보딩 항목이 하나 생긴다는 뜻이기도 하다.

## 3. lint-staged — 왜 따로 필요한가

`npm run lint` 를 훅에 넣으면 될 것 같지만 두 가지가 안 된다.

1. **프로젝트 전체를 검사해서 느리다.** 커밋마다 수 초~수십 초가 붙는다
2. **내가 안 건드린 파일의 에러 때문에 막힌다.** 남이 만든 기존 위반이 내 커밋을 막으면 사람은 `--no-verify` 를 배운다

lint-staged 는 **스테이징된 파일 목록만** 명령에 인자로 붙여 실행한다.

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["prettier --write", "eslint --fix"],
    "*.{json,md,css}": "prettier --write"
  }
}
```

키는 glob(picomatch), 값은 명령이다. **배열이면 순서대로, 배열을 한 겹 더 중첩하면 병렬**로 돈다. 기본은 glob 끼리 동시 실행이라, glob 이 겹치면서 둘 다 파일을 고치면 경쟁 상태가 생긴다.

포맷터가 파일을 고치는 경우도 알아서 처리한다 — 원래 상태를 백업하고, 태스크를 돌리고, **태스크가 만든 변경을 다시 스테이징하고**, 임시 파일을 정리한다. [[Git 훅(Git Hooks)]]의 마지막 함정(훅이 고친 내용이 커밋에 안 들어감)이 여기서 해결된다.

설정 파일은 `package.json` 의 `lint-staged` 필드, `.lintstagedrc*`, `lint-staged.config.js` 등 여러 형태를 지원한다. JS 설정에서는 `defineConfig` 헬퍼로 타입을 받을 수 있다.

## 4. commitlint — 커밋 메시지 검사

```bash
npm install -D @commitlint/cli @commitlint/config-conventional
echo "export default { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js
```

설정은 cosmiconfig 로 찾으므로 `commitlint.config.{js,ts,mjs,cjs}` · `.commitlintrc*` · `package.json` 의 `commitlint` 필드 중 아무거나 쓸 수 있다.

### `config-conventional` 이 실제로 켜는 규칙

**[[Conventional Commits]] 스펙과 이 프리셋은 같은 게 아니다.** 프리셋은 스펙에 없는 규칙까지 켠다. v21.2.2 소스 기준으로 확인한 전부다.

| 규칙 | 기본값 | 스펙에 있나 |
| ----------- | ----------- | ----------- |
| `type-enum` | `build` `chore` `ci` `docs` `feat` `fix` `perf` `refactor` `revert` `style` `test` **11개** | ❌ 스펙은 `feat`·`fix` 만 정한다 |
| `type-case` | `lower-case` | ❌ 스펙은 대소문자를 안 가린다 |
| `type-empty` | 금지 | ✅ |
| `subject-empty` | 금지 | ✅ |
| `subject-full-stop` | 마침표 `.` 로 끝나면 안 됨 | ❌ 관습 |
| `subject-case` | `sentence-case` · `start-case` · `pascal-case` · `upper-case` **금지** | ❌ 관습 |
| `header-max-length` | **100** | ❌ |
| `body-max-line-length` | 100 | ❌ |
| `footer-max-line-length` | 100 | ❌ |
| `body-leading-blank` · `footer-leading-blank` | 경고(Warning) | ✅ (스펙은 MUST 인데 프리셋은 경고) |

두 가지가 눈에 띈다.

* **`scope-enum` 이 기본값에 없다.** scope 어휘를 고정하고 싶으면 직접 넣어야 한다
* **`type-enum` 에 `wip` 가 없다.** 작업 중 커밋을 `wip:` 으로 남기던 습관은 이 프리셋에서 바로 막힌다

`header-max-length` 가 **100**이라는 것도 알아둘 만하다. 흔히 말하는 "제목 50자, 본문 72자"는 git 관습이지 이 프리셋의 값이 아니다. 더 짧게 하고 싶으면 직접 덮어써야 한다.

```js
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'scope-enum': [2, 'always', ['app', 'pages', 'widgets', 'features', 'entities', 'shared', 'deps', 'config']],
    'header-max-length': [2, 'always', 72],
  },
}
```

한국어로 커밋을 쓸 때 `subject-case` 는 문제가 안 된다. 한글에는 대소문자가 없어서 어떤 케이스에도 걸리지 않는다. 반면 `subject-full-stop` 은 그대로 적용되므로 **"~ 구현."** 처럼 마침표를 붙이면 거부된다.

## 5. 왜 문서만으로는 안 되는가

`CONTRIBUTING.md` 를 아무리 잘 써도 규칙이 안 지켜지는 이유는 게으름이 아니라 구조다.

* **읽는 시점과 지키는 시점이 다르다.** 문서는 온보딩 때 한 번 읽고, 커밋은 하루에 열 번 한다
* **어긴 게 언제 드러나는지가 너무 늦다.** 리뷰에서 지적받거나, 아무도 지적하지 않으면 영원히 안 드러난다
* **어긴 커밋 하나가 규칙 전체를 무효화한다.** 로그에 규약 위반이 섞이면 `git log --grep` 도 체인지로그 생성도 못 믿게 되고, **못 믿는 순간 지킬 이유가 사라진다.** [[Feature-Sliced Design(FSD)]]이 린터 없이 몇 주 만에 무너지는 것과 정확히 같은 구조다
* **커밋 메시지는 사후 수정이 사실상 불가능하다.** 코드 스타일은 나중에 포매터를 한 번 돌리면 되지만, 푸시된 커밋 메시지를 고치려면 히스토리를 다시 써야 한다 → [[Git 머지 전략(merge · squash · rebase)]]

마지막 항목 때문에 **"나중에 붙이자"가 코드 린터보다 훨씬 비싸다.** 첫 커밋에 같이 넣는 게 압도적으로 싸다.

### 장점
* **규칙 위반이 발견되는 시점이 리뷰가 아니라 커밋 순간**으로 당겨진다
* **리뷰가 형식 지적에서 해방된다.** 리뷰어가 설계 이야기만 하게 된다
* `npm install` 한 번으로 팀 전체에 배포된다
* lint-staged 덕분에 **기존 위반이 새 커밋을 막지 않는다**

### 단점
* **`--no-verify` 로 우회된다.** 훅의 근본 한계 → [[Git 훅(Git Hooks)]]
* **커밋이 느려진다.** husky 자체는 ~1ms 지만 뒤의 eslint·prettier·commitlint 가 붙는다. 몇 초를 넘으면 사람은 커밋을 뭉치기 시작하고, 그건 [[Conventional Commits]]이 노리던 "작게 자주 커밋"과 반대 방향이다
* **팀원 환경 문제.** GUI + 버전 매니저 조합의 `command not found`, `~/.config/husky/init.sh` 로 각자 해결해야 함
* **의존성 세 개와 설정 파일 두세 개가 늘어난다.** 그리고 세 도구가 각자 node 버전 요구가 다르다 — lint-staged 17 은 node ≥22.22.1, commitlint 21 은 ≥22.12.0, husky 9 는 ≥18 이다. **팀 안에 낮은 node 를 쓰는 사람이 하나 있으면 거기서 막힌다**
* **스쿼시 머지 워크플로에서는 효과가 반감된다.** 작업 중 커밋 메시지는 어차피 사라지고, 최종 메시지는 GitLab MR UI 에서 만들어져 로컬 훅을 안 거친다. 그래서 **훅과 별개로 MR 제목에 대한 서버 측 검사가 따로 필요**해진다
* **husky 는 배포가 뜸하다.** 최신이 2024년 11월의 9.1.7 이다. 안정된 것으로 볼 수도 있고(2kB, 의존성 없음) 정체로 볼 수도 있다

## 6. 펫바디에서의 검토 (초안 · 미확정)

`petbody-client` 에는 **세 도구가 모두 설치돼 있지 않다.** `CONTRIBUTING.md` 초안의 미결 항목 중 하나가 "붙일지 여부"다.

이미 있는 것과 없는 것이 뚜렷하다.

| 검사 | 도구 | 현재 |
| ----------- | ----------- | ----------- |
| 포맷 | Prettier | ✅ `npm run format` |
| 린트 | ESLint (flat config) | ✅ `npm run lint` |
| 타입 | tsc | ✅ `npm run typecheck` |
| FSD 레이어 규칙 | steiger | ✅ `npm run fsd` |
| **자동 실행** | husky | ❌ |
| **스테이징 파일만** | lint-staged | ❌ |
| **커밋 메시지** | commitlint | ❌ |

즉 **검사 도구는 다 있고 실행 시점을 강제하는 층만 비어 있다.** 지금은 사람이 기억해서 다섯 개 명령을 돌려야 하고, 안 돌려도 아무 일이 일어나지 않는다.

붙인다면 scope 어휘를 [[Feature-Sliced Design(FSD)]] 레이어로 고정하는 `scope-enum` 이 함께 들어가야 한다(→ [[Conventional Commits]]). 여섯 레이어는 스펙이 고정한 값이라 늘어나지 않는다는 게 이 선택의 근거였다 — **`scope-enum` 목록을 자주 고쳐야 하는 순간이 규칙을 끄게 되는 순간**이기 때문이다.

**아직 팀 논의 전이다.** 3인 팀에서 훅이 주는 이득과 환경 문제·속도 저하의 비용이 어느 쪽이 큰지는 붙여봐야 안다. 다만 5절 마지막 줄대로 **소스가 60여 개인 지금이 가장 싼 시점**이라는 점은 분명하다.

## 정리

* 셋의 역할은 **husky = 배달, lint-staged = 대상 좁히기, commitlint = 메시지 검사.** 서로 대체 불가
* **husky v9: `npx husky init` → `.husky/` 파일 + `prepare: husky`.** 훅 파일은 명령만 쓴다
* v8 의 `#!/usr/bin/env sh` 와 `. "$(dirname -- "$0")/_/husky.sh"` **두 줄은 폐기됐고 v10 에서 실패한다.** 옛 예제 복사가 지금 가장 흔한 실수
* v4 의 `HUSKY_GIT_PARAMS` 는 사라지고 **`$1`** 을 쓴다. `commitlint --edit $1`
* `HUSKY=0` 으로 끈다. CI·Docker 는 이걸 쓰고, GUI `command not found` 는 `~/.config/husky/init.sh`
* lint-staged 는 **스테이징된 파일만** 넘기고, **태스크가 고친 파일을 다시 스테이징**해준다
* **`config-conventional` ≠ 스펙.** type 11개·`header-max-length` 100 등은 프리셋의 선택이고, **`scope-enum` 은 기본값에 없다**
* 문서만으로 안 되는 이유는 게으름이 아니라 **어긴 게 드러나는 시점이 너무 늦고, 커밋 메시지는 사후 수정이 안 되기 때문**이다
* 한계는 **`--no-verify` 우회 · 커밋 속도 · 팀원 환경 · node 버전 요구 불일치**, 그리고 **스쿼시 워크플로에서의 효과 반감**

#펫바디
