# Docker 멀티스테이지 빌드
----

하나의 `Dockerfile` 안에 `FROM` 을 여러 번 써서 빌드 단계를 나누고, **최종 이미지에는 실행에 필요한 것만 담는** 기법.

## 1. 왜 필요한가

단일 스테이지로 만들면 최종 이미지에 이런 게 전부 남는다.
- TypeScript 컴파일러, 빌드 도구 같은 **devDependencies**
- 소스코드 원본 (`src/`)
- 빌드 중 생긴 캐시

실행에는 하나도 필요 없는데 이미지 크기와 **공격 표면**만 키운다.

멀티스테이지는 `COPY --from=<스테이지>` 로 **앞 단계의 결과물만 골라서** 가져온다. 나머지 레이어는 최종 이미지에 포함되지 않는다.

## 2. Node 서버 4단계 구성

> ⚠️ **베이스 이미지 태그는 유통기한이 있다.** 원래 이 문서는 `node:20-bookworm-slim` 으로 적혀 있었는데, **Node 20은 2026-04-30에 EOL** 이라 보안 패치를 못 받는다. 2026-09-01 기준 [Node 릴리스 일정](https://github.com/nodejs/Release)상 **24가 Active LTS**(2028-04-30까지), 22는 유지보수(2027-04-30까지)다. 아래는 24로 갱신했다.
> Debian 쪽도 13(trixie) 태그가 나와 있어(`node:24-trixie-slim`) bookworm은 oldstable로 밀렸다. 여기서는 문서 흐름을 위해 bookworm을 유지한다.

```dockerfile
# ---- deps: 빌드용 의존성 전체 설치 (devDependencies 포함) ----
FROM node:24-bookworm-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# ---- build: TypeScript → dist/ ----
FROM deps AS build
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# ---- prod-deps: 런타임 의존성만 다시 설치 ----
FROM node:24-bookworm-slim AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ---- runtime: 최종 이미지 ----
FROM node:24-bookworm-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./
EXPOSE 3300
CMD ["node", "dist/index.js"]
```

| 스테이지 | 역할 |
| ----------- | ----------- |
| `deps` | `npm ci` — devDependencies 포함, 빌드용 |
| `build` | 컴파일. `src` → `dist` |
| `prod-deps` | `npm ci --omit=dev` — 런타임 전용 의존성만 |
| `runtime` | `dist` + 프로덕션 `node_modules` 만 담아 최종 구성 |

### `prod-deps` 를 왜 따로 두나

`deps` 에서 `npm prune --production` 을 해도 되지만, 그러면 **이미 devDependencies가 설치된 레이어 위에** 삭제 레이어가 쌓인다. Docker 레이어는 append-only라 **삭제해도 이전 레이어의 용량은 남는다.**

이건 감이 아니라 [공식 문서](https://docs.docker.com/engine/storage/drivers/)에 적힌 동작이다 (2026-09-01 확인).

> "Each layer except the very last one is read-only."
> "Note that both *adding*, and *removing* files will result in a new layer. In the example above, the `$HOME/.cache` directory is removed, but **will still be available in the previous layer and add up to the image's total size**."

즉 `RUN rm -rf ...` 는 **상위 레이어에 삭제 마커(whiteout)를 하나 더 얹을 뿐**, 하위 레이어의 실제 데이터는 그대로 이미지에 실려 다닌다. 지우는 것도 레이어를 늘린다는 게 핵심.

깨끗한 베이스에서 `--omit=dev` 로 새로 설치하는 게 실제로 작은 이미지를 만든다.

## 3. 레이어 캐시 — COPY 순서가 전부다

```dockerfile
COPY package.json package-lock.json ./   # ① 거의 안 바뀜
RUN npm ci                               # ② 그래서 캐시가 유지됨
COPY src ./src                           # ③ 자주 바뀜
RUN npm run build                        # ④ 여기부터만 다시 실행
```

`COPY . .` 을 먼저 해버리면 소스 한 줄만 고쳐도 `npm ci` 부터 다시 돈다. **변경 빈도가 낮은 것부터 COPY** 하는 게 원칙.

`.dockerignore` 도 같이 챙길 것. `node_modules`, `dist`, `.git` 을 빼야 빌드 컨텍스트 전송이 빨라지고, 호스트의 `node_modules` 가 이미지에 섞여 들어가는 사고도 막는다.

## 4. alpine vs bookworm-slim

이미지 크기만 보면 alpine이 훨씬 작지만, **alpine은 musl libc, debian 계열은 glibc** 를 쓴다. Node 공식 이미지 설명에 그대로 적혀 있는 주의사항이다 (2026-09-01 확인).

> "The main caveat to note is that it does use **musl libc** instead of **glibc and friends**, so software will often run into issues depending on the depth of their libc requirements/assumptions."

네이티브 바이너리를 포함한 패키지는 대개 **glibc 기준으로 빌드**되어 있어서 alpine에서 깨지거나, 되더라도 성능 이슈가 있을 수 있다. `optionalDependencies` 로 플랫폼별 바이너리를 내려받는 패키지들(예: 네이티브 TS 컴파일러, sharp, esbuild 일부 조합)이 특히 그렇다.

- 순수 JS [[라이브러리]]만 있다 → **alpine 가도 됨**
- 네이티브 바이너리가 하나라도 있다 → **`slim` 계열이 안전.** 크기 차이보다 "빌드가 되느냐"가 우선

빌드 스테이지와 런타임 스테이지의 베이스를 굳이 다르게 할 이유도 별로 없다. 같은 베이스로 두면 레이어를 공유해서 실제 디스크 사용량도 줄어든다.

## 5. 실제로 겪은 함정 — 절대경로 하드코딩

로컬(Windows)에서 잘 돌던 코드가 컨테이너에서 죽었다.

```ts
// 변경 전
app.register(require('@fastify/static'), {
  root: path.join('W:\\petbody', 'static'),
  prefix: '/static/',
});
```

Linux 컨테이너에서 `path.isAbsolute('W:\\petbody\\static')` 는 **`false`** 로 평가된다. POSIX 경로 규칙상 `W:...` 는 절대경로가 아니기 때문. `@fastify/static` 이 등록 시점에 이렇게 던지고 서버가 죽는다.

```
FST_STATIC_INVALID_OPTION('root', 'must be an absolute path')
```

(이 에러 문자열은 `@fastify/static` 소스에서 그대로 확인했다 — 2026-09-01. `path.isAbsolute(rootPath) === false` 일 때 던진다.)

**환경변수로 빼고 기본값을 두는 게 정답.**

```ts
// 변경 후
root: process.env.STATIC_DIR
  ? path.resolve(process.env.STATIC_DIR)
  : path.join(__dirname, '..', 'static'),
```

> 교훈: **호스트 경로·포트·자격증명은 코드에 박지 말 것.** 컨테이너화는 이런 걸 전부 드러낸다. 반대로 말하면, Docker로 한 번 돌려보는 것만으로 환경 의존성을 찾아낼 수 있다.

## 6. 검증 방법

빌드만 되는 게 아니라 **실제로 뜨는지**를 봐야 한다. 임시 네트워크에 DB를 띄워 붙여보는 게 가장 확실하다.

```bash
docker network create petbody-net

docker run -d --name mysql-test --network petbody-net \
  -e MYSQL_ROOT_PASSWORD=rootpass -e MYSQL_DATABASE=petbody \
  -e MYSQL_USER=petbody -e MYSQL_PASSWORD=petbodypass mysql:8.0

docker run -d --name app-test --network petbody-net -p 3304:3300 \
  -e DB_HOST=mysql-test -e DB_USER=petbody -e DB_PASSWORD=petbodypass -e DB_NAME=petbody \
  petbody-server

curl http://localhost:3304/health
# {"status":"ok"}
```

- 같은 네트워크 안에서는 **컨테이너 이름이 곧 호스트명**이 된다 (`DB_HOST=mysql-test`)
- 컨테이너에서 **호스트 머신**의 서비스에 붙으려면 `host.docker.internal`
- 검증 끝나면 컨테이너·네트워크·이미지 정리할 것

**단계적으로 확인하는 게 중요하다.** 먼저 존재하지 않는 DB 호스트로 띄워서 "정적 경로 에러 없이 DB 연결 에러만 나는지" 보면, 고치려던 문제가 실제로 사라졌는지 분리해서 검증할 수 있다.

## 7. 정리

- 멀티스테이지 = **빌드 산출물만 최종 이미지에 담는 것**
- `prod-deps` 를 별도 스테이지로 — 레이어는 append-only라 **삭제해도 용량이 안 준다**
- **COPY 순서가 캐시 효율을 결정** — 안 바뀌는 것부터
- alpine(musl) vs slim(glibc) — **네이티브 바이너리가 있으면 slim**
- **베이스 이미지 메이저는 주기적으로 갱신할 것.** LTS도 EOL이 온다 (Node 20 = 2026-04-30 종료)
- 컨테이너화는 **하드코딩된 환경 의존성을 드러낸다.** 경로·포트·자격증명은 환경변수로
- 빌드 성공이 아니라 **실제 기동**까지 확인할 것

이 이미지가 띄우는 서버의 엔티티 설계는 [[TypeORM 엔티티 설계]]에 정리했다.

#펫바디
