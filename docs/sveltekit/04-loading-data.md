# 데이터 로딩

페이지와 레이아웃에 데이터를 로드하는 방법. load 함수의 종류, 스코프, 병렬 실행 원리.

---

## 1. load 함수 파일 — 두 가지 선택지

경로 폴더에 페이지 컴포넌트(`+page.svelte`) 외에 **데이터를 로드하는 파일**을 추가할 수 있다.

```text
src/routes/about/
├── +page.svelte          ← UI 컴포넌트
├── +page.ts              ← Universal load (서버 + 클라이언트)
└── +page.server.ts       ← Server load (서버 전용)
```

### `+page.ts` — Universal load

```ts
// src/routes/about/+page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async () => {
  return { message: 'Hello from universal load' }
}
```

- SSR 시 → 서버에서 실행
- CSR 네비게이션 시 → **브라우저에서 직접 실행**
- 클라이언트에 노출되므로 **비밀 정보, DB 쿼리, private API 호출 금지**

### `+page.server.ts` — Server load

```ts
// src/routes/about/+page.server.ts
import type { PageServerLoad } from './$types'
import * as db from '$lib/server/db'

export const load: PageServerLoad = async () => {
  const posts = await db.getPosts()
  return { posts }
}
```

- **항상 서버에서만 실행** — 클라이언트에 코드가 노출되지 않음
- DB 쿼리, private API, 비밀 자격 증명 안전하게 사용 가능
- CSR 네비게이션 시 SvelteKit이 서버에 JSON 요청을 보내 데이터를 가져옴

### 비교 정리

| | `+page.ts` (Universal) | `+page.server.ts` (Server) |
|--|--|--|
| 실행 환경 | 서버 + 브라우저 | 서버만 |
| DB / private env | X | O |
| 반환값 | 클래스 등 자유로움 | 직렬화 필수 (JSON) |
| CSR navigation | 브라우저에서 직접 실행 | 서버에 JSON 요청 |

> 대부분의 경우 `+page.server.ts` 하나로 충분하다. 두 파일이 동시에 존재할 수도 있지만 드문 케이스.

---

## 2. 레이아웃 load — 하위 페이지에 데이터 전파

레이아웃에도 load 함수를 정의할 수 있다. **레이아웃 load가 반환한 데이터는 해당 레이아웃을 사용하는 모든 하위 페이지에서 접근 가능**하다.

```text
src/routes/
├── +layout.server.ts          ← ① 루트 레이아웃 load (앱 전체에서 사용 가능)
├── (marketing)/
│   ├── +layout.server.ts      ← ② 마케팅 레이아웃 load (마케팅 그룹 하위에서 사용 가능)
│   ├── +page.svelte           ← 홈페이지 (①, ② 데이터 접근 가능)
│   └── about/
│       ├── +page.server.ts    ← ③ about 페이지 load (이 페이지에서만 사용)
│       └── +page.svelte       ← about 페이지 (①, ②, ③ 데이터 접근 가능)
```

> 레이아웃 상속 구조 → [03-routing.md](./03-routing.md) Section 4 참고

### 데이터 스코프 규칙

| load 파일 위치 | 데이터 사용 범위 |
|--|--|
| `routes/+layout.server.ts` | 앱 전체 (모든 페이지) |
| `(marketing)/+layout.server.ts` | 마케팅 그룹 내 모든 페이지 |
| `about/+page.server.ts` | about 페이지만 |

```text
/about 요청 시 실행되는 load 함수:

  routes/+layout.server.ts          ← 루트 레이아웃 (항상 실행)
  (marketing)/+layout.server.ts     ← 마케팅 레이아웃 (about이 이 그룹에 속하므로)
  about/+page.server.ts             ← about 페이지 자체
```

> 레이아웃 load는 `+layout.server.ts` 또는 `+layout.ts` 모두 가능하다. 선택 기준은 페이지 load와 동일 (서버 전용 작업이 필요하면 `.server`).

---

## 2.5 layout vs page load 키 충돌 — 우선순위 규칙

layout load와 page load가 **같은 키를 반환**하면 page load의 값이 우선한다.

```text
layout.server.ts  →  { user: {...}, theme: 'dark' }
page.server.ts    →  { posts: [...], theme: 'light' }
                       ───────────────────────────────
컴포넌트 data      =  { user: {...}, posts: [...], theme: 'light' }
                                                   ↑ page 값이 우선
```

### 원칙

| load 파일 | 반환해야 할 데이터 |
|--|--|
| `+layout.server.ts` | 공통 데이터 — `user`, `config`, `navigation` 등 |
| `+page.server.ts` | 페이지별 데이터 — `posts`, `product`, `article` 등 |

같은 키를 반환하는 것은 의도적 오버라이드일 때만 허용. 일반적으로 layout과 page는 서로 다른 키를 담당한다.

### 키 충돌 확인 방법

TypeScript를 사용하면 `data`의 타입에서 충돌을 감지할 수 있다:

```ts
// +page.svelte에서 data 타입 확인
// data.theme이 layout의 'dark'인지 page의 'light'인지 헷갈리면
// 타입을 명시적으로 확인한다
import type { PageProps } from './$types'
let { data }: PageProps = $props()
// data.theme 타입이 'dark' | 'light'이면 충돌 신호
```

---

## 3. 병렬 실행 — 폭포(waterfall)가 아니다

요청에 관여하는 **모든 load 함수는 병렬로 실행**된다.

```text
/about 요청 시:

  routes/+layout.server.ts       ─┐
  (marketing)/+layout.server.ts  ─┼─ 동시에 병렬 실행
  about/+page.server.ts          ─┘
```

위에서 아래로 순차 실행(waterfall)이 **아니다**. 이는 성능에 유리하지만 중요한 제약이 있다:

- 페이지 load에서 부모 레이아웃 load의 반환값에 **기본적으로 접근할 수 없다** (병렬이므로 누가 먼저 끝날지 모름)
- 부모 데이터가 필요하면 `await parent()`로 명시적으로 대기해야 한다 (→ 후속 강의에서 다룸)

---

## 4. 요청 경로에 해당하는 load만 실행

```text
/ (홈페이지) 요청 시:

  ✅ routes/+layout.server.ts         ← 루트 레이아웃
  ✅ (marketing)/+layout.server.ts    ← 홈페이지가 속한 레이아웃
  ✅ (marketing)/+page.server.ts      ← 홈페이지 자체 (있다면)
  ❌ about/+page.server.ts            ← 요청과 무관 → 실행 안 됨
```

요청된 경로가 사용하는 레이아웃 체인 + 해당 페이지의 load만 실행된다. 다른 경로의 load는 실행되지 않는다.

---

## 5. 실습: Universal load 기본 동작

### `+page.ts` 작성

```ts
// src/routes/(marketing)/blog/+page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async () => {
  console.log('blog route universal load')

  return {
    title: 'Blog',
    count: 10
  }
}
```

`load`라는 이름으로 함수를 **export** 해야 한다. SvelteKit이 이 이름을 인식한다.

### `+page.svelte`에서 data 수신

```svelte
<!-- src/routes/(marketing)/blog/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types'

  let { data }: PageProps = $props()
</script>

<h1>{data.title}</h1>
```

- `$props()`로 `data`를 받으면, load 함수가 반환한 객체가 들어온다
- `PageProps` 타입을 사용하면 `data.title`(string), `data.count`(number) 등 **자동 완성과 타입 추론**이 작동한다
- 타입을 직접 정의할 필요 없음 — load 함수의 반환값을 기반으로 SvelteKit이 `$types`를 자동 생성

### Universal load의 실행 시점 확인

```text
페이지 새로고침 (SSR):
  ✅ 서버 터미널에 "blog route universal load" 출력
  ✅ 브라우저 콘솔에도 "blog route universal load" 출력
  → 서버에서 먼저 실행 후, 클라이언트에서 하이드레이션 시 다시 실행

클라이언트 사이드 네비게이션 (다른 페이지 → /blog):
  ❌ 서버 터미널에 출력 없음
  ✅ 브라우저 콘솔에만 출력
  → 브라우저에서 직접 실행 (서버 요청 없음)
```

> 이것이 universal load의 핵심 동작이다. 서버 전용 작업(DB, 비밀키)이 필요하면 `+page.server.ts`를 사용해야 한다.

---

## 6. `$types` — 자동 생성 타입 시스템

### `$types`는 어디서 오는가?

`'./$types'`에서 import하는 타입들은 **실제 경로 폴더에 존재하지 않는다**. 개발 서버를 실행하거나 빌드할 때 `.svelte-kit/types/` 폴더에 자동 생성된다.

```text
.svelte-kit/types/src/routes/
├── (marketing)/
│   ├── blog/
│   │   ├── $types.d.ts       ← blog 경로의 생성된 타입
│   │   └── [id]/
│   │       └── $types.d.ts   ← blog/[id] 경로의 생성된 타입
│   └── ...
```

이 폴더는 실제 `src/routes/` 구조를 **미러링**한다. TypeScript가 두 폴더를 하나로 인식하는 이유:

1. 프로젝트의 `tsconfig.json`이 `.svelte-kit/tsconfig.json`을 extends
2. `.svelte-kit/tsconfig.json`에서 `rootDirs`로 실제 프로젝트 폴더와 types 폴더를 모두 지정
3. TypeScript가 두 폴더를 **하나의 가상 폴더**로 취급 → `'./$types'`가 정상 resolve

### 생성되는 타입들

```ts
// .svelte-kit/types/src/routes/(marketing)/blog/$types.d.ts 내부 (개념)

// load 함수의 반환값으로부터 추론
type PageData = { title: string; count: number }

// $props()에서 사용
type PageProps = { data: PageData }

// load 함수 시그니처
type PageLoad = (event: { params: {}; ... }) => MaybePromise<PageData>
```

### 동적 라우트의 params 타입

```ts
// src/routes/(marketing)/blog/[id]/+page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async ({ params }) => {
  console.log(params.id) // ✅ 자동 완성 — id: string
  // params.slug         // ❌ 타입 에러 — 존재하지 않는 파라미터

  return { postId: params.id }
}
```

`[id]` 폴더명을 기반으로 `params`의 타입이 `{ id: string }`으로 자동 생성된다. 잘못된 파라미터명을 사용하면 컴파일 타임에 에러가 발생한다.

### 타입 어노테이션 생략 가능

Svelte VS Code 확장이 설치되어 있으면 타입을 명시하지 않아도 추론이 작동한다:

```ts
// 명시적 타입 — 두 가지 방식 모두 가능
export const load: PageLoad = async ({ params }) => { ... }
export const load = (async ({ params }) => { ... }) satisfies PageLoad

// 타입 생략 — VS Code 확장이 백그라운드에서 타입 추가
export const load = async ({ params }) => { ... }  // 여전히 자동 완성 작동
```

> 타입을 명시하면 에디터에 의존하지 않고 어디서든 타입 안전성이 보장되므로 **명시하는 것을 권장**.

---

## 7. Server load 실습 + 두 파일 공존

> **고급**: 이 섹션은 `+page.ts`(universal)와 `+page.server.ts`(server) 둘 다를 이해한 뒤 읽을 것.
> 기초 학습 중이라면 건너뛰어도 무방 — Sec 8(레이아웃 load 실습)로 바로 이동.

### `+page.server.ts`로 변경

```ts
// src/routes/(marketing)/blog/+page.server.ts
import type { PageServerLoad } from './$types'

export const load: PageServerLoad = async () => {
  console.log('blog route server load')

  return {
    title: 'Blog',
    count: 10
  }
}
```

- 타입이 `PageLoad`가 아닌 **`PageServerLoad`** — 서버 전용 load에는 다른 이벤트 속성이 제공됨
- DB 쿼리, 비밀 자격 증명 등 서버 전용 작업 안전

### 실행 확인

```text
페이지 새로고침:
  ❌ 브라우저 콘솔에 출력 없음
  ✅ 서버 터미널에만 출력
  → 서버에서만 실행됨

클라이언트 사이드 네비게이션 (다른 페이지 → /blog):
  ❌ 브라우저에서 함수 실행 안 됨
  ✅ 서버에서 실행 → __data.json fetch 요청으로 데이터 전달
```

CSR 네비게이션 시 네트워크 탭을 보면 `__data.json` 요청이 발생한다. SvelteKit이 서버에서 load를 실행하고 결과를 JSON으로 클라이언트에 전달하는 방식이다.

### `+page.ts` + `+page.server.ts` 동시 존재

두 파일이 동시에 존재하면, **컴포넌트가 받는 `data`는 `+page.ts`(universal)의 반환값**이다. `+page.server.ts`의 반환값은 컴포넌트에 직접 전달되지 않는다.

```ts
// src/routes/(marketing)/blog/+page.server.ts
export const load: PageServerLoad = async () => {
  return { title: 'Blog', count: 10 }
}
```

```ts
// src/routes/(marketing)/blog/+page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async ({ data }) => {
  // data = server load의 반환값 { title, count }
  return {
    ...data,  // server load 데이터를 그대로 전달
    x: 1      // universal에서 추가 데이터 병합
  }
}
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types'
  let { data }: PageProps = $props()
</script>

<!-- data.title ✅, data.count ✅, data.x ✅ -->
<h1>{data.title}</h1>
```

**핵심 규칙**: 두 파일 공존 시 데이터 흐름은 `server load → universal load의 data 파라미터 → 컴포넌트`. Universal load가 server load의 데이터를 **중계**하는 구조다.

> 실무에서 두 파일을 동시에 쓰는 경우: 서버에서 DB 데이터를 가져오되, 클라이언트에서 직렬화 불가능한 값(Svelte 컴포넌트, 클래스 인스턴스 등)을 추가해야 할 때.

### 실전 예제: A/B 테스트 — 컴포넌트를 load에서 전달

Svelte 컴포넌트는 직렬화할 수 없으므로 `+page.server.ts`에서 반환할 수 없다. 서버에서 데이터를 가져오면서 동시에 컴포넌트를 동적으로 선택해야 하는 경우, 두 파일이 모두 필요하다.

```ts
// src/routes/(marketing)/blog/+page.server.ts
export const load: PageServerLoad = async ({ fetch }) => {
  const response = await fetch('/api/posts')
  const postsResponse: PostsResponse = await response.json()

  // A/B 테스트: 서버에서 표시할 게시물 스타일 결정
  const postType = Math.random() > 0.5 ? 1 : 2

  return { title: 'Blog', posts: postsResponse, postType }
}
```

```ts
// src/routes/(marketing)/blog/+page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async ({ data }) => {
  // server load에서 결정된 postType에 따라 컴포넌트 동적 import
  const module = data.postType === 1
    ? await import('./Post-1.svelte')
    : await import('./Post-2.svelte')

  return {
    ...data,
    component: module.default  // Svelte 컴포넌트 (직렬화 불가 → universal에서만 가능)
  }
}
```

```svelte
<!-- src/routes/(marketing)/blog/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types'
  let { data }: PageProps = $props()
</script>

<h1>{data.title}</h1>

{#each data.posts.posts as post}
  <data.component {post} />
{/each}
```

**경로 폴더 안에 일반 컴포넌트를 둘 수 있다** — `+page`, `+layout`, `+server` 등의 접두사가 없는 파일은 라우팅에 영향을 주지 않으므로, 해당 경로에서만 사용하는 컴포넌트를 가까이 배치할 수 있다.

```text
src/routes/(marketing)/blog/
├── +page.svelte
├── +page.ts
├── +page.server.ts
├── Post-1.svelte        ← 태그 표시 O (A 변형)
└── Post-2.svelte        ← 태그 표시 X (B 변형)
```

---

## 8. 레이아웃 load 실습 — 자식 페이지에 데이터 전파

### 루트 레이아웃에서 사용자 정보 반환

```ts
// src/routes/+layout.server.ts
import type { LayoutServerLoad } from './$types'

export const load: LayoutServerLoad = async ({ cookies }) => {
  const token = cookies.get('token')

  const user = token
    ? { name: 'John', id: 1 }  // 실제로는 DB에서 조회
    : null

  return { user }
}
```

- `cookies` 객체는 load 함수의 이벤트에서 제공됨 — `get`, `set` 등 메서드 보유
- 루트 레이아웃이므로 **앱 전체 모든 페이지**에서 `data.user`에 접근 가능

### 자식 페이지에서 사용

```svelte
<!-- src/routes/(marketing)/blog/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types'
  let { data }: PageProps = $props()
</script>

<!-- 페이지 자체 데이터 (title, count) + 레이아웃 데이터 (user) 자동 병합 -->
<h1>{data.title}</h1>
<p>{data.user?.name}</p>
```

타입도 자동으로 병합된다 — `data`에는 페이지 load의 `title`, `count`와 레이아웃 load의 `user`가 모두 포함.

### 레이아웃 자체에서도 사용

```svelte
<!-- src/routes/(marketing)/+layout.svelte -->
<script lang="ts">
  import type { LayoutProps } from './$types'
  let { data, children }: LayoutProps = $props()
</script>

<nav>
  <!-- ... 메뉴 항목들 ... -->
  {#if data.user}
    <a href="/dashboard" class="text-orange-500">대시보드</a>
  {:else}
    <a href="/login">로그인</a>
  {/if}
</nav>

{@render children()}
```

### 데이터 병합 시각화

```text
/blog 요청 시 data 구성:

  routes/+layout.server.ts     → { user }
  (marketing)/blog/+page.ts    → { title, count, x }
                                 ─────────────────────
  +page.svelte가 받는 data     = { user, title, count, x }  (자동 병합)
```

> 이 예시의 인증은 개념 설명용이다. 실무 인증은 `hooks.server.ts`의 `handle` + `event.locals` 조합이 권장됨 (→ 02-core.md Section 5 참조).

---

## 9. `await parent()` — 부모 레이아웃 데이터 접근

### 문제: load 함수에서 부모 레이아웃 데이터가 필요하다

컴포넌트(`+page.svelte`, `+layout.svelte`)에서는 `data`에 레이아웃 데이터가 자동 병합된다. 하지만 **다른 load 함수 내부**에서 부모 레이아웃의 데이터에 접근하려면?

load 함수는 기본적으로 **병렬 실행**이므로, 부모 load가 아직 완료되지 않았을 수 있다. 명시적으로 대기해야 한다.

### `parent()` 사용법

```ts
// src/routes/(marketing)/blog/+page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async ({ parent }) => {
  const parentData = await parent()
  // parentData = 모든 부모 레이아웃 load의 병합 결과
  // { user, marketingLayoutData } (루트 + 마케팅 레이아웃)

  console.log(parentData.user)
  console.log(parentData.marketingLayoutData)

  return { title: 'Blog' }
}
```

`await parent()`는 **직접 부모뿐 아니라 모든 상위 레이아웃**의 load가 완료될 때까지 대기하고, 그 반환값을 병합한 결과를 돌려준다.

### 레이아웃 load에서도 사용 가능

```ts
// src/routes/(marketing)/+layout.server.ts
import type { LayoutServerLoad } from './$types'

export const load: LayoutServerLoad = async ({ parent }) => {
  const parentData = await parent()
  // parentData = 루트 레이아웃의 데이터 { user }

  return { marketingLayoutData: 'some data' }
}
```

페이지 load 뿐 아니라 레이아웃 load에서도 `parent()`를 사용할 수 있다.

### 주의: waterfall을 최소화하라

`await parent()`는 부모 load가 끝날 때까지 **현재 함수 실행을 차단**한다. 불필요한 waterfall을 피하려면:

```ts
export const load: PageLoad = async ({ parent, fetch }) => {
  // ✅ 부모에 의존하지 않는 작업을 먼저 수행
  const posts = await fetch('/api/posts').then(r => r.json())

  // ✅ 부모 데이터가 필요한 시점에서만 대기
  const parentData = await parent()

  // ✅ 부모 데이터에 의존하는 작업
  const filtered = posts.filter(p => p.userId === parentData.user?.id)

  return { posts: filtered }
}
```

```ts
// ❌ 안티패턴: 부모를 먼저 기다린 후 독립적인 작업 수행
export const load: PageLoad = async ({ parent, fetch }) => {
  const parentData = await parent()  // 불필요하게 차단
  const posts = await fetch('/api/posts').then(r => r.json())  // 부모와 무관
  return { posts }
}
```

> **원칙**: 부모 데이터에 의존하지 않는 작업 → 먼저 실행. `await parent()` → 실제로 필요한 시점에만 호출.

---

## 10. `page` 상태로 자식 → 부모 데이터 전달 (SEO 실습)

### 문제: 레이아웃에서 페이지별 데이터가 필요하다

`<svelte:head>`를 루트 레이아웃에 한 번만 두고, 현재 페이지의 load 데이터에 따라 `<title>`과 `<meta>`를 동적으로 바꾸고 싶다. 하지만 데이터 흐름은 부모 → 자식 방향이다.

### 해결: `page` 상태 — 현재 경로의 모든 데이터 포함

`$app/state`의 `page` 객체에는 현재 경로의 **모든 load 데이터가 병합**되어 있다. 레이아웃에서 `page.data`로 자식 페이지의 데이터에 접근 가능.

### 페이지 load에서 SEO 데이터 반환

```ts
// src/routes/(marketing)/blog/+page.server.ts
export const load: PageServerLoad = async () => {
  return {
    title: 'Blog',
    description: '최신 블로그 게시물 목록',
    count: 10
  }
}
```

### 루트 레이아웃에서 `<svelte:head>` 동적 렌더링

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { page } from '$app/state'
  import type { LayoutProps } from './$types'

  let { data, children }: LayoutProps = $props()
</script>

<svelte:head>
  <title>{page.data.title ? `${page.data.title} | My App` : 'My App'}</title>
  {#if page.data.description}
    <meta name="description" content={page.data.description} />
  {/if}
</svelte:head>

{@render children()}
```

### 동작 결과

```text
/ (홈페이지)       → title 없음 → "My App" (폴백)
/about             → title 없음 → "My App" (폴백)
/blog              → title: "Blog" → "Blog | My App"
                     description: "최신 블로그 게시물 목록" (meta 태그 추가)
```

### 개별 페이지에서 `<svelte:head>` 직접 사용도 가능

```svelte
<!-- src/routes/(marketing)/+page.svelte (홈페이지) -->
<svelte:head>
  <title>Home | My App</title>
</svelte:head>

<h1>Welcome</h1>
```

각 페이지에서 직접 `<svelte:head>`를 사용할 수도 있지만, 누락 위험이 있다. 루트 레이아웃에서 `page.data` 기반 폴백을 두면 **누락 방지 + 일관성**을 확보할 수 있다.

> **핵심**: `page.data`는 레이아웃 + 페이지 load의 전체 병합 결과이므로, 부모 레이아웃에서도 자식 페이지의 load 데이터에 자유롭게 접근할 수 있다.

---

> 더 많은 패턴 → [05-advanced-patterns.md](./05-advanced-patterns.md)

