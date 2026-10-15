# TypeORM 엔티티 설계
----

TypeScript용 ORM [[라이브러리]]. 클래스에 **데코레이터**를 붙여 테이블을 정의한다.

> **TypeORM 1.1.0 기준** (2026-09-01 확인). 오래 0.3.x에 머물러 있다가 **2026-05-19에 1.0이 나왔다.** 인터넷의 0.3 시절 글을 그대로 따라 하면 어긋나는 지점이 있다.
> 1.0의 파급이 큰 변경 — **Node.js 20 이상 요구**, `Connection`/`ConnectionOptions` 완전 제거(`DataSource` 만 남음), `createConnection`·`getConnection` 같은 전역 함수 제거, **`@EntityRepository` 와 `AbstractRepository` 제거**(`Repository.extend()` 로 대체), MySQL은 **`mysql2` 드라이버만 지원**, SQLite 기본이 `better-sqlite3`. 공식 [1.0 릴리스 노트](https://typeorm.io/docs/releases/1.0/release-notes) · [업그레이드 가이드](https://typeorm.io/docs/releases/1.0/upgrading-from-0.3) 에 코드모드(`@typeorm/codemod`)까지 나와 있다.

## 0. 시작 전 필수

```ts
import 'reflect-metadata';   // 앱 진입점 최상단에서 단 한 번
```

데코레이터가 타입 정보를 런타임에 읽으려면 `reflect-metadata` 가 필요하다. 빠뜨리면 엔티티 등록 시점에 알 수 없는 에러가 난다. **1.0에서도 이 요구는 그대로다.** `tsconfig.json` 에 `experimentalDecorators`, `emitDecoratorMetadata` 도 켜야 하고, 공식 문서가 요구하는 **TypeScript 최소 버전은 4.5** 다.

TS 5의 표준 데코레이터(Stage 3)와는 다른 **레거시 데코레이터** 를 쓴다는 점에 유의. 둘은 호환되지 않으므로 `experimentalDecorators` 를 끄면 안 된다. 참고로 TypeScript 7(네이티브 포트)도 `experimentalDecorators` / `emitDecoratorMetadata` 를 지원한다.

### 곁다리 — `paths` 는 `baseUrl` 없이도 동작한다

실제로 이 프로젝트에서 헷갈렸던 부분이라 같이 적어둔다. `@/entities/*` 같은 경로 별칭을 쓰려고 `baseUrl` 을 꼭 넣어야 한다고 알려진 경우가 많은데, **아니다.**

```jsonc
{
  "compilerOptions": {
    // baseUrl 없음
    "paths": { "@/*": ["./src/*"] }
  }
}
```

공식 tsconfig 문서가 `paths` 를 **"`baseUrl` 이 설정돼 있으면 그것을, 아니면 tsconfig 파일 자체를 기준으로"** 재매핑한다고 명시한다. 즉 `baseUrl` 없이 쓰면 **tsconfig 위치 기준 상대경로**가 된다.

- 판단 기준 — `baseUrl` 은 부작용이 있다. 설정하는 순간 `node_modules` 가 아닌 **프로젝트 내부 디렉터리 이름이 그냥 `import 'utils'` 로 해석되는 길**이 열려서, 패키지와 로컬 디렉터리가 이름 충돌할 때 조용히 엉뚱한 걸 잡는다. **별칭만 원한다면 `paths` 만 쓰는 게 낫다**
- 함정 — 이건 **타입 검사기만의 규칙**이다. 런타임 해석은 별개라 `tsconfig-paths`, 번들러 `resolve.alias`, 또는 Node의 `imports` 필드로 **따로 맞춰줘야 한다.** 빌드는 통과하는데 `node dist/index.js` 가 모듈을 못 찾는 사고가 여기서 난다

## 1. 기본 형태

```ts
@Entity({ name: 'Identity_Users' })          // 실제 테이블명 지정
@Index('IX_Users_TenantId_IsActive', ['TenantId', 'IsActive'])   // 복합 인덱스
export class IdentityUser {
  @PrimaryColumn({ type: 'varchar', length: 255 })
  Id!: string;

  @Column({ type: 'varchar', length: 256, nullable: true })
  Email!: string | null;

  @Column({ type: 'datetime', precision: 6 })
  CreatedOn!: Date;
}
```

- `@Entity({ name })` — 클래스명과 테이블명이 다를 때 명시. 레거시 스키마를 그대로 쓸 때 필수
- `!` (definite assignment) — 생성자에서 할당하지 않으므로 TS를 달래는 용도
- `| null` 과 `nullable: true` 를 **둘 다** 맞춰줘야 타입과 스키마가 일치한다

## 2. PK — Primary vs PrimaryGenerated

| 데코레이터 | 용도 |
| ----------- | ----------- |
| `@PrimaryGeneratedColumn()` | auto-increment 정수 |
| `@PrimaryGeneratedColumn('uuid')` | DB가 UUID 생성 |
| `@PrimaryColumn()` | **애플리케이션이 직접 값을 넣음** |

GUID를 쓰되 애플리케이션에서 생성하는 정책이면 `@PrimaryColumn` 이 맞다.

```ts
const id = crypto.randomUUID();   // Node 내장. 별도 패키지 불필요
```

### 왜 auto-increment 대신 GUID인가
- **삽입 전에 ID를 알 수 있다** — 연관 레코드를 한 트랜잭션에서 같이 만들기 쉬움
- 여러 DB/샤드를 합쳐도 충돌하지 않음
- 순차 정수처럼 **총 레코드 수가 URL로 새어나가지 않음**
- 단점 — 인덱스가 커지고, 랜덤이라 B-Tree 삽입 지역성이 나쁨. MySQL InnoDB에서는 이게 실제 성능 차이로 나타난다

## 3. 인덱스 — 컬럼 레벨 vs 엔티티 레벨

```ts
// 컬럼 레벨 — 단일 컬럼
@Index('UserNameIndex', { unique: true })
@Column({ type: 'varchar', length: 256, nullable: true })
NormalizedUserName!: string | null;

// 엔티티 레벨 — 복합 인덱스
@Index('IX_Users_TenantId_IsActive', ['TenantId', 'IsActive'])
export class IdentityUser { ... }
```

**복합 인덱스는 컬럼 순서가 전부다.** `(TenantId, IsActive)` 인덱스는
- `WHERE TenantId = ?` → 사용됨
- `WHERE TenantId = ? AND IsActive = ?` → 사용됨
- `WHERE IsActive = ?` → **사용 안 됨** (선두 컬럼이 없음)

멀티테넌시라면 **선두 컬럼을 항상 TenantId로** 두는 게 좋다. 모든 조회가 테넌트로 먼저 좁혀지기 때문. → [[멀티테넌시 데이터 모델]]

## 4. 관계 매핑 — FK 컬럼과 관계를 같이 선언하는 패턴

```ts
@Column({ type: 'varchar', length: 255, nullable: true })
TenantId!: string | null;                    // ① 스칼라 FK 컬럼

@ManyToOne(() => Tenant, { nullable: true, onDelete: 'RESTRICT' })
@JoinColumn({ name: 'TenantId' })            // ② 같은 컬럼에 관계를 매핑
Tenant?: Tenant;
```

①과 ②가 **같은 물리 컬럼**을 가리킨다. 왜 둘 다 선언하나?

- **① 만 쓰면** — 조인이 안 됨. 관계 로딩(`relations: ['Tenant']`)을 못 씀
- **② 만 쓰면** — ID만 알고 있을 때도 엔티티 객체를 통째로 만들어 넣어야 함. `insert({ TenantId: 'root' })` 처럼 못 씀
- **둘 다 두면** — 쓸 때는 ID로 간단하게, 읽을 때는 조인으로 편하게

`() => Tenant` 처럼 **화살표 함수로 감싸는 이유**는 순환 참조 때문. 모듈이 서로를 import할 때 클래스를 직접 참조하면 `undefined` 가 될 수 있어서, 평가를 지연시킨다.

### onDelete 정책

| 값 | 동작 | 언제 |
| ----------- | ----------- | ----------- |
| `CASCADE` | 부모 삭제 시 자식도 삭제 | 부모 없이는 의미가 없는 데이터 (사용자 → 리프레시 토큰) |
| `RESTRICT` | 자식이 있으면 부모 삭제 거부 | 실수로 지우면 안 되는 마스터 데이터 (테넌트) |
| `SET NULL` | 자식의 FK를 NULL로 | 부모가 없어도 자식이 유효할 때 |

기본값에 기대지 말고 **명시적으로 정할 것.** 지워도 되는지 아닌지는 도메인 지식이라 ORM이 대신 판단해줄 수 없다.

## 5. `synchronize: true` 는 개발 전용

```ts
export const AppDataSource = new DataSource({
  type: 'mysql',
  synchronize: true,   // ⚠️ 엔티티 정의에 맞춰 스키마를 자동 변경
  entities: [ ... ],
});
```

엔티티를 고치면 앱 시작 시 DDL을 알아서 실행해준다. 초기 개발엔 편하지만 **프로덕션에서는 절대 금지.** 이건 커뮤니티 통념이 아니라 [공식 옵션 문서](https://typeorm.io/docs/data-source/data-source-options)에 적힌 경고다.

> "Indicates if database schema should be auto created on every application launch. **Be careful with this option and don't use this in production - otherwise you can lose production data.** This option is useful during debug and development."

- 컬럼명을 바꾸면 → **기존 컬럼을 DROP하고 새로 만든다.** 데이터가 사라진다
- 실행되는 DDL을 사전에 검토할 방법이 없다
- 롤백 경로가 없다

→ 스키마가 안정되면 `synchronize: false` 로 바꾸고 **마이그레이션**으로 전환할 것.

```bash
typeorm migration:generate -d src/db/data-source.ts src/db/migrations/AddPetTable
typeorm migration:run -- -d src/db/data-source.ts
```

**여기서 한 번 걸린다 — `migration:generate` 는 `.ts` 를 만드는데, `migration:run` 과 `migration:revert` 는 `.js` 에서만 동작한다.** 컴파일 없이 TS 마이그레이션을 돌리려면 전용 바이너리를 써야 한다.

```bash
npx typeorm-ts-node-commonjs migration:run -- -d src/db/data-source.ts
npx typeorm-ts-node-esm     migration:run -- -d src/db/data-source.ts   # ESM 프로젝트
```

그리고 DataSource 쪽 `migrations` 글롭에 **두 확장자를 다 넣어두는 게 정석**이다. 개발에서는 `.ts` 로, [[Docker 멀티스테이지 빌드]]로 만든 이미지 안에서는 컴파일된 `.js` 로 같은 설정이 돌아간다.

```ts
migrations: [__dirname + '/migrations/**/*{.js,.ts}'],
```

최소한 **환경변수로 분기**시켜 두는 게 안전하다.
```ts
synchronize: process.env.NODE_ENV !== 'production',
```

## 6. 정규화(Normalized) 컬럼 패턴

ASP.NET Identity에서 온 패턴인데, 알아둘 가치가 있다.

```
Email              admin@aiet.co.kr        ← 원본. 화면 표시용
NormalizedEmail    ADMIN@AIET.CO.KR        ← 대문자. 검색·인덱스용
```

**왜 이렇게 하나** — MySQL은 콜레이션에 따라 대소문자 구분 여부가 달라지고, DB마다 정책이 다르다. DB의 콜레이션에 의존하는 대신 **애플리케이션이 정규화 규칙을 소유**하면 동작이 예측 가능해진다.

- 조회는 항상 `NormalizedEmail` 로 (인덱스가 확실히 탄다)
- 표시는 `Email` 로 (사용자가 입력한 원본 유지)
- **주의** — 저장 시 두 컬럼을 반드시 같이 갱신해야 한다. 한쪽만 바꾸면 조용히 어긋난다. 리포지토리 계층에서 강제하는 게 좋다

## 7. 정리

- `reflect-metadata` 를 진입점에서 import — 안 하면 원인 모를 에러
- `@PrimaryColumn` + `crypto.randomUUID()` = 애플리케이션 생성 GUID. **삽입 전에 ID를 알 수 있는 게 실질적 이점**
- 복합 인덱스는 **선두 컬럼이 없으면 안 탄다.** 멀티테넌시면 TenantId를 앞에
- 스칼라 FK 컬럼 + `@ManyToOne` 을 **둘 다** 선언하는 게 실용적
- 관계에 `() => Entity` 를 쓰는 건 **순환 참조 회피**
- `onDelete` 는 기본값에 기대지 말고 명시
- **`synchronize: true` 는 프로덕션 금지.** 컬럼명 변경 = 데이터 손실. 공식 문서가 "you can lose production data" 라고 직접 경고한다
- 마이그레이션은 **`.js` 에서만 실행된다.** TS로 바로 돌리려면 `typeorm-ts-node-commonjs`
- `paths` 는 **`baseUrl` 없이도 된다.** 단 타입 검사기 얘기일 뿐, 런타임 해석은 따로 맞춰야 한다
- 정규화 컬럼은 **DB 콜레이션에 의존하지 않으려는 패턴.** 대신 갱신 누락에 주의

테넌트 격리 전략은 [[멀티테넌시 데이터 모델]], 배포는 [[Docker 멀티스테이지 빌드]]에 정리했다.

#펫바디
