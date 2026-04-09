# Load 함수 재실행 & 프리로딩

load 함수가 언제 자동으로 재실행되며, 어떻게 직접 재실행을 트리거하는가.
링크 호버 시 데이터를 미리 로드하는 프리로딩 전략.

---

## 1. 실험 환경 구성 — 블로그 상세 페이지 + 사이드바

load 함수의 재실행 타이밍을 관찰하기 위해, 여러 레벨의 load 함수가 **동시에 활성화**되는 구조를 만든다.

```text
src/routes/
├── +layout.server.ts                       ← 루트 레이아웃 load
├── (marketing)/
│   ├── +layout.server.ts                   ← 마케팅 그룹 load
│   └── blog/
│       └── [id]/
│           ├── +layout.server.ts           ← 사이드바용 load
│           └── +page.server.ts             ← 단일 게시물 load
```

4개의 load 함수가 하나의 경로에서 **동시에 실행**될 수 있는 구조다. 각 load에 `console.log`를 추가하면 네비게이션 시 **어떤 load가 재실행되는지** 서버 터미널에서 관찰할 수 있다.

> **핵심 질문**: 같은 경로 그룹 내에서 페이지를 이동하면, 4개의 load 함수 중 어떤 것이 재실행될까? → 아래에서 확인.

---

## 2. 자동 재실행 — SvelteKit의 종속성 추적

SvelteKit은 load 함수 내부에서 **어떤 값을 참조했는지** 자동으로 추적하여 종속성 맵을 생성한다. 해당 종속성이 변경되면 load 함수가 자동으로 재실행된다.

```text
params.id 변경: /blog/1 → /blog/2

  routes/+layout.server.ts       ── ❌ params 미참조 → 재실행 안 됨
  (marketing)/+layout.server.ts  ── ❌ params 미참조 → 재실행 안 됨
  [id]/+layout.server.ts         ── ❌ params 미참조 → 재실행 안 됨
  [id]/+page.server.ts           ── ✅ params.id 참조 → 재실행
```

### 종속성 1: `params` — 경로 매개변수

```ts
// src/routes/(marketing)/blog/[id]/+page.server.ts
export const load: PageServerLoad = async ({ params }) => {
  console.log('[id] page load')
  const post = posts.find((p) => p.id === Number(params.id))
  return { post }
}
```

`/blog/1` → `/blog/2`로 이동하면:
- **경로(`route.id`)는 동일** — 여전히 `(marketing)/blog/[id]`
- **`params.id`가 변경** — `1` → `2`
- `params.id`를 참조한 **페이지 load만 재실행**, 레이아웃 load는 실행되지 않음

```text
# 사이드바에서 다른 게시물 클릭 시 서버 로그
[id] page load          ← ✅ params.id 의존 → 재실행
                        ← ❌ 레이아웃 load 3개는 실행 안 됨
```

> **중요**: 같은 경로 내 네비게이션에서는 컴포넌트가 **언마운트/재마운트되지 않는다**. `data` props만 업데이트된다. 따라서 `data`에 의존하는 값은 반드시 `$derived`로 선언해야 한다.

### 레이아웃 load에서 params를 참조하면?

```ts
// src/routes/(marketing)/blog/[id]/+layout.server.ts
export const load: LayoutServerLoad = async ({ params }) => {
  console.log('[id] layout load')
  console.log(params.id)  // ⚠️ params.id 참조 → 종속성 등록!

  const morePosts = posts.slice(0, 3)
  return { morePosts }
}
```

`params.id`를 단순히 로깅만 해도 **종속성으로 등록**된다. 이제 게시물 전환 시:

```text
[id] layout load        ← 😱 불필요한 재실행! morePosts는 변하지 않는데...
[id] page load
```

사이드바 데이터(`morePosts`)는 `id`와 무관한데, 로깅 때문에 매번 다시 fetching하게 된다.

> **tip**: 종속성 등록 없이 값을 사용해야 할 때는 `untrack(() => params.id)`로 감싼다. 이 load가 사용하는 값인데 변경에 반응하지 않길 원할 때만 사용하는 드문 케이스다. 키 형식은 반드시 `접두사:이름`(콜론 포함).

### 종속성 2: `url.searchParams` — 검색 매개변수

```ts
// src/routes/(marketing)/blog/+page.server.ts
export const load: PageServerLoad = async ({ url }) => {
  console.log('blog page load')
  const page = Number(url.searchParams.get('page') ?? '1')
  // page 기반으로 게시물 로드...
  return { posts: paginatedPosts }
}
```

- `/blog?page=1` → `/blog?page=2`: **`page` 검색 매개변수를 참조**하므로 load 재실행
- `/blog?page=1` → `/blog?page=1&x=123`: `x`는 load에서 참조하지 않으므로 **재실행 안 됨**

```text
# ?page=2 클릭 시
blog page load          ← ✅ url.searchParams.get('page') 의존

# ?x=123 추가 시
                        ← ❌ 실행 안 됨 (x를 참조하지 않음)
```

**SvelteKit은 참조한 검색 매개변수만 추적한다.** 관련 없는 매개변수 변경에는 반응하지 않는다.

### 종속성 3: `await parent()` — 부모 load 의존

```ts
// src/routes/(marketing)/blog/[id]/+page.server.ts
export const load: PageServerLoad = async ({ params, parent }) => {
  await parent()  // ⚠️ 부모 load에 대한 종속성 등록
  console.log('[id] page load')
  const post = posts.find((p) => p.id === Number(params.id))
  return { post }
}
```

`await parent()`를 호출하면 **양방향 종속성**이 생긴다:

1. **이 load가 재실행되면 → 부모 load도 먼저 재실행** (부모 데이터 최신화를 위해)
2. **부모 load가 재실행되면 → 이 load도 재실행** (의존하고 있으므로)

```text
# 게시물 전환 시 (params.id 변경)
root layout load        ← 😱 부모 전부 재실행
marketing layout load   ← 😱
[id] layout load        ← 😱
[id] page load          ← params.id 의존 + parent() 의존
```

> **주의**: `await parent()`는 필요한 경우에만 사용할 것. 무분별한 사용은 **모든 부모 load의 불필요한 재실행**을 유발한다.

### 자동 재실행 종속성 요약

| 종속성 | 변경 조건 | 재실행 범위 |
|--------|----------|------------|
| `params.*` | URL 매개변수 변경 (`/blog/1` → `/blog/2`) | 해당 param을 참조한 load만 |
| `url.searchParams` | 검색 매개변수 변경 (`?page=1` → `?page=2`) | 해당 param을 참조한 load만 |
| `await parent()` | 부모 load 재실행 시 | 자신 + 모든 부모 load |
| `fetch(url)` | URL이 바뀌면 | 해당 load (다음 섹션에서 상세) |

> **핵심**: SvelteKit은 load 함수의 종속성을 자동 추적하여, **필요한 load만 최소한으로 재실행**한다.

---

## 3. 수동 무효화 — `invalidateAll()`, `invalidate()`, `depends()`

자동 종속성 추적만으로 부족할 때, 직접 load 함수를 다시 실행하는 방법.

### `invalidateAll()` — 전부 재실행

```svelte
<!-- src/routes/(marketing)/blog/[id]/+page.svelte -->
<script lang="ts">
  import { invalidateAll } from '$app/navigation'
</script>

<button onclick={() => invalidateAll()}>
  Reload
</button>
```

현재 페이지에서 **활성화된 모든 load 함수**를 재실행한다.

```text
# /blog/1에서 Reload 클릭 시
root layout load        ← 재실행
marketing layout load   ← 재실행
[id] layout load        ← 재실행
[id] page load          ← 재실행
```

간단하지만, 루트 레이아웃처럼 **변경될 일이 없는 데이터까지 전부 다시 로드**하므로 과도할 수 있다.

### `invalidate(url)` — 특정 URL에 의존하는 load만 재실행

```svelte
<script lang="ts">
  import { invalidate } from '$app/navigation'
  import { page } from '$app/state'
</script>

<button onclick={() => invalidate(`https://dummyjson.com/posts/${page.params.id}`)}>
  Reload
</button>
```

`invalidate()`에 URL을 전달하면, **해당 URL을 `fetch()`한 load 함수만** 재실행된다.

#### ⚠️ Universal load에서만 동작

```ts
// ✅ +page.ts (Universal load) — fetch URL이 종속성으로 등록됨
export const load: PageLoad = async ({ fetch, params }) => {
  const res = await fetch(`https://dummyjson.com/posts/${params.id}`)
  // fetch URL → 자동 종속성 등록
  return { post: await res.json() }
}

// ❌ +page.server.ts (Server load) — fetch URL이 종속성으로 등록 안 됨
export const load: PageServerLoad = async ({ fetch, params }) => {
  const res = await fetch(`https://dummyjson.com/posts/${params.id}`)
  // 서버 전용이므로 클라이언트에서 URL 추적 불가
  return { post: await res.json() }
}
```

**Server load의 fetch는 종속성으로 추적되지 않는다.** 비밀 URL이 클라이언트에 노출되면 안 되기 때문이다.

#### URL은 정확히 일치해야 한다

```ts
// ✅ 동작 — 정확히 동일한 URL
invalidate('https://dummyjson.com/posts/1')

// ❌ 동작 안 함 — 뒤에 슬래시 추가
invalidate('https://dummyjson.com/posts/1/')
```

#### 함수로 유연하게 매칭

정확한 URL 대신 **함수를 전달**하여 유연하게 매칭할 수 있다.

```svelte
<script lang="ts">
  import { invalidate } from '$app/navigation'
</script>

<button onclick={() => invalidate((url) => {
  // url: load 함수가 fetch한 모든 URL이 순서대로 전달됨
  return url.hostname === 'dummyjson.com'
      && url.pathname.startsWith('/posts')
})}>
  Reload
</button>
```

조건에 `true`를 반환하면 해당 URL에 의존하는 load 함수가 재실행된다.

### `depends()` — 커스텀 종속성 키 (Server load에서도 동작)

Server load의 fetch URL은 자동 추적이 안 되지만, `depends()`로 **임의의 문자열 키**를 종속성으로 등록할 수 있다.

```ts
// src/routes/(marketing)/blog/[id]/+layout.server.ts ← Server load!
export const load: LayoutServerLoad = async ({ fetch, depends }) => {
  console.log('[id] layout load')

  depends('blog:sidebar')  // ✅ 커스텀 종속성 키 등록

  const res = await fetch('https://dummyjson.com/posts?limit=3')
  const { posts } = await res.json()
  return { morePosts: posts }
}
```

#### 키 형식 규칙

```ts
depends('blog:sidebar')    // ✅ "접두사:이름" 형식
depends('app:user')        // ✅
depends('sidebar')         // ❌ 콜론이 없으면 에러
```

반드시 **`접두사:이름`** 형식(콜론 포함)이어야 한다.

#### 페이지에서 커스텀 키로 무효화

```svelte
<!-- src/routes/(marketing)/blog/[id]/+page.svelte -->
<script lang="ts">
  import { invalidate } from '$app/navigation'
  import { page } from '$app/state'
</script>

<button onclick={() => {
  // Universal load의 fetch URL 무효화
  invalidate(`https://dummyjson.com/posts/${page.params.id}`)
  // Server load의 커스텀 키 무효화
  invalidate('blog:sidebar')
}}>
  Reload
</button>
```

```text
# Reload 클릭 시

# 서버 터미널 (Server load)
[id] layout load        ← depends('blog:sidebar') 무효화

# 브라우저 콘솔 (Universal load)
[id] page load          ← fetch URL 무효화

# 네트워크 탭
├── posts/1             ← Universal load → 클라이언트에서 직접 요청
├── comments             ← Universal load → 클라이언트에서 직접 요청
└── __data.json         ← Server load → SvelteKit 내부 데이터 요청
```

### 무효화 방법 비교

| 방법 | 대상 | Server load | Universal load | 사용 시점 |
|------|------|:-----------:|:--------------:|----------|
| `invalidateAll()` | 모든 활성 load | ✅ | ✅ | 전체 새로고침이 필요할 때 |
| `invalidate(url)` | 해당 URL을 fetch한 load | ❌ | ✅ | 특정 API 응답 갱신 |
| `invalidate(fn)` | 조건 매칭된 URL의 load | ❌ | ✅ | 패턴 기반 유연한 매칭 |
| `invalidate(key)` | `depends(key)` 등록한 load | ✅ | ✅ | Server load 선택적 무효화 |

> **핵심**: `invalidate(url)`은 Universal load 전용, `depends()` + `invalidate(key)`는 Server load에서도 동작. 대부분의 경우 `depends()`를 사용하는 것이 더 안전하고 유연하다.

---

## 4. 실전 패턴 — 오류 복구와 선택적 무효화

`invalidate()`와 `depends()`를 실제 오류 처리 시나리오에 적용하는 패턴.

### 패턴 1: 오류 페이지에서 다시 시도

페이지 load에서 오류가 발생하면 `+error.svelte`가 표시된다. 이때 **다시 시도 버튼**으로 해당 load만 재실행할 수 있다.

#### load에 `depends()` 등록

```ts
// src/routes/(marketing)/blog/[id]/+page.server.ts
export const load: PageServerLoad = async ({ params, fetch, depends }) => {
  depends('blog:post')  // ✅ 커스텀 키 등록
  console.log('[id] page load')

  if (Math.random() > 0.5) {
    throw error(500, '게시물 로드 실패')  // 무작위 오류 시뮬레이션
  }

  const res = await fetch(`https://dummyjson.com/posts/${params.id}`)
  return { post: await res.json() }
}
```

#### 오류 페이지에서 선택적 무효화

```svelte
<!-- src/routes/(marketing)/blog/[id]/+error.svelte -->
<script lang="ts">
  import { invalidate } from '$app/navigation'
  import { page } from '$app/state'
</script>

<h1>{page.error?.message}</h1>

<button onclick={() => invalidate('blog:post')}>
  Reload
</button>
```

`invalidateAll()` 대신 `invalidate('blog:post')`를 사용하면, **오류가 발생한 페이지 load만 재실행**하고 루트 레이아웃 등 불필요한 load는 건드리지 않는다.

필요에 따라 레이아웃 load도 함께 무효화할 수 있다:

```svelte
<button onclick={() => {
  invalidate('blog:post')     // 페이지 load 재실행
  invalidate('blog:sidebar')  // 레이아웃 load도 재실행 (네트워크 오류 시 사이드바도 실패했을 수 있음)
}}>
  Reload
</button>
```

### 패턴 2: 부분 오류 — 사이드바만 실패, 페이지는 정상

모든 오류가 `throw error()`로 전체 페이지를 날려야 하는 것은 아니다. **핵심이 아닌 데이터**(사이드바, 추천 목록 등)가 실패하면 오류를 던지지 않고 **빈 상태 + 재시도 버튼**을 보여주는 것이 더 나은 UX다.

#### 레이아웃 load — 오류를 던지지 않고 데이터로 반환

```ts
// src/routes/(marketing)/blog/[id]/+layout.server.ts
import type { LayoutServerLoad } from './$types'

export const load: LayoutServerLoad = async ({ fetch, depends }) => {
  depends('blog:sidebar')
  console.log('[id] layout load')

  const res = await fetch('https://dummyjson.com/posts?limit=3')

  if (!res.ok) {
    // ❌ throw error() → 전체 페이지 오류
    // ✅ 오류 객체를 데이터로 반환 → 사이드바만 오류 표시
    return {
      morePosts: { error: '사이드바 게시물을 불러올 수 없습니다' }
    }
  }

  const { posts } = await res.json()
  return { morePosts: posts }
}
```

> **핵심 판단**: `throw error()` vs 데이터로 반환
> - **페이지의 핵심 데이터** 실패 → `throw error()` → `+error.svelte` 표시
> - **부가 데이터** 실패 → 오류 객체 반환 → 해당 영역만 오류 UI 표시

#### 레이아웃 UI — 조건부 렌더링 + 선택적 무효화

```svelte
<!-- src/routes/(marketing)/blog/[id]/+layout.svelte -->
<script lang="ts">
  import { invalidate } from '$app/navigation'
  import type { LayoutProps } from './$types'

  let { data, children }: LayoutProps = $props()
</script>

<div class="container">
  <div class="grid">
    <div>
      {@render children()}
    </div>

    <aside>
      <h2>More Posts</h2>
      {#if 'error' in data.morePosts}
        <!-- 오류 상태: 메시지 + 재시도 -->
        <p>{data.morePosts.error}</p>
        <button onclick={() => invalidate('blog:sidebar')}>
          Reload
        </button>
      {:else}
        <!-- 정상 상태: 게시물 목록 -->
        {#each data.morePosts as post}
          <article>
            <h3>{post.title}</h3>
            <p>{post.body}</p>
          </article>
        {/each}
      {/if}
    </aside>
  </div>
</div>
```

`invalidate('blog:sidebar')`는 **레이아웃 load만 재실행**하므로, 정상 로드된 페이지 데이터에 영향을 주지 않는다.

### 오류 복구 전략 정리

| 시나리오 | load 처리 | UI 처리 | 무효화 |
|---------|----------|---------|--------|
| **핵심 데이터 실패** (게시물 본문) | `throw error(500, msg)` | `+error.svelte` 표시 | `invalidate('blog:post')` |
| **부가 데이터 실패** (사이드바) | `return { error: msg }` | 인라인 오류 + 재시도 버튼 | `invalidate('blog:sidebar')` |
| **전체 새로고침** | — | 새로고침 버튼 | `invalidateAll()` |

> **실무 원칙**: 오류의 중요도에 따라 `throw error()` vs 데이터 반환을 구분하고, `depends()` 키를 분리하여 **오류가 발생한 load만 정밀하게 재시도**할 수 있게 설계한다.

---

## 5. 프리로딩 — 링크 위에 마우스를 올리면 미리 로드

SvelteKit은 사용자가 링크를 **클릭하기 전에** 데이터와 코드를 미리 로드하여 체감 속도를 높일 수 있다.

```text
                     시간 축
────────────────────────────────────────────────▶
[호버]  [코드 + __data.json 요청]  [클릭] [즉시 표시]
  │         │                       │
  │         └── 완료 (캐시됨) ──────┘
  │
  └── data-sveltekit-preload-data="hover" 설정 시
```

### `data-sveltekit-preload-data` — 데이터 + 코드 프리로드

```html
<!-- src/app.html -->
<body data-sveltekit-preload-data="hover">
  %sveltekit.body%
</body>
```

`hover`로 설정하면, 링크 위에 마우스를 **올리기만 해도** 해당 경로의 load 함수가 실행되고 코드가 로드된다. 실제 클릭 시에는 이미 데이터가 준비되어 있어 **즉시 페이지가 표시**된다.

#### 옵션 값

| 값 | 트리거 시점 | 설명 |
|---|-----------|------|
| `"hover"` | 마우스 오버 | 가장 일반적. 호버 → 클릭 사이 시간에 데이터 로드 |
| `"tap"` | 마우스 다운 (클릭 직전) | 호버보다 늦지만, 클릭 완료 전에 로드 시작 |
| `"off"` | 프리로드 안 함 | 클릭 후에야 로드 시작 |

#### 첫 방문 vs 재방문

```text
# 첫 방문: /about에서 블로그 링크 호버
├── +page.svelte        ← 코드 로드
├── +page.server.ts     ← 코드 로드
├── posts 컴포넌트       ← 코드 로드
└── __data.json         ← 데이터 로드

# 재방문: 블로그 다녀온 후 다시 호버
└── __data.json         ← 데이터만 로드 (코드는 캐시됨)
```

코드는 한 번 로드되면 **캐시**되어 재요청하지 않는다. 데이터는 변경 가능성이 있으므로 **매번 다시 요청**한다.

### 적용 범위 — 상속과 오버라이드

`body`에 설정하면 앱 전체에 적용되지만, **하위 요소에서 개별 오버라이드** 가능하다.

```html
<!-- src/app.html — 앱 전체 기본값 -->
<body data-sveltekit-preload-data="hover">
```

```svelte
<!-- 특정 nav에서만 끄기 -->
<nav data-sveltekit-preload-data="off">
  <a href="/blog">Blog</a>   <!-- 프리로드 안 됨 -->
  <a href="/about">About</a> <!-- 프리로드 안 됨 -->
</nav>

<!-- 특정 링크만 다르게 -->
<a href="/heavy-page" data-sveltekit-preload-data="tap">Heavy Page</a>
```

부모 요소에 설정하면 **내부의 모든 `<a>` 태그에 상속**된다.

### `data-sveltekit-preload-code` — 코드만 프리로드

데이터는 미리 로드하지 않고 **코드(컴포넌트, load 함수)만** 미리 로드한다. 데이터가 자주 바뀌어 호버 시점에 로드하면 클릭 시점에 이미 stale할 수 있는 경우 유용.

```html
<nav data-sveltekit-preload-code="hover">
  <a href="/blog">Blog</a>
</nav>
```

```text
# 블로그 링크 호버 시
├── +page.svelte        ← ✅ 코드 로드
├── +page.server.ts     ← ✅ 코드 로드
└── __data.json         ← ❌ 데이터는 로드 안 됨 (클릭 시 로드)
```

#### 코드 프리로드 전용 옵션

| 값 | 트리거 시점 |
|---|-----------|
| `"hover"` | 마우스 오버 |
| `"tap"` | 마우스 다운 |
| `"eager"` | 페이지 로드 시 **모든 링크**의 코드를 즉시 로드 |
| `"viewport"` | 링크가 **뷰포트에 보이면** 코드 로드 |
| `"off"` | 프리로드 안 함 |

```html
<!-- 페이지 내 모든 링크의 코드를 즉시 프리로드 -->
<nav data-sveltekit-preload-code="eager">
  <a href="/blog">Blog</a>
  <a href="/about">About</a>
</nav>

<!-- 뷰포트에 보이는 링크만 코드 프리로드 -->
<div data-sveltekit-preload-code="viewport">
  {#each posts as post}
    <a href="/blog/{post.id}">{post.title}</a>
  {/each}
</div>
```

### 프로그래밍 방식 프리로드

```ts
import { preloadData, preloadCode } from '$app/navigation'

// 데이터 + 코드 프리로드 (Promise 반환)
await preloadData('/blog/1')

// 코드만 프리로드
await preloadCode('/blog/1')
```

`preloadData()`는 나중에 **모달로 경로 콘텐츠를 표시**하는 패턴에서 사용한다.

### 기타 앵커 태그 속성

| 속성 | 기본값 | 설명 |
|------|-------|------|
| `data-sveltekit-preload-data` | — | 데이터 + 코드 프리로드 (`hover`, `tap`, `off`) |
| `data-sveltekit-preload-code` | — | 코드만 프리로드 (`hover`, `tap`, `eager`, `viewport`, `off`) |
| `data-sveltekit-reload` | — | 클라이언트 라우팅 대신 **브라우저 전체 새로고침** |
| `data-sveltekit-replacestate` | — | `history.pushState` 대신 `history.replaceState` (뒤로가기 기록 안 남김) |
| `data-sveltekit-keepfocus` | — | 네비게이션 후 **현재 포커스 유지** (기본은 리셋) |
| `data-sveltekit-noscroll` | — | 네비게이션 후 **스크롤 위치 유지** (기본은 최상단으로 스크롤) |

> **핵심**: `data-sveltekit-preload-data="hover"`는 **거의 모든 SvelteKit 앱에서 기본 설정**으로 사용한다. `body`에 한 번 설정하고, 필요한 곳에서만 개별 오버라이드하는 패턴이 권장된다.

---

## React/Next.js 비교

| 개념 | SvelteKit | Next.js (App Router) |
|------|-----------|---------------------|
| 종속성 추적 | 자동 (load 내 참조 추적) | 없음 (캐시/revalidate 수동) |
| 선택적 재실행 | 참조한 param 변경 → 해당 load만 | `revalidateTag()` / `revalidatePath()` |
| 추적 방지 | `untrack()` | 해당 개념 없음 |
| 전체 무효화 | `invalidateAll()` | `router.refresh()` |
| 링크 프리로드 | `data-sveltekit-preload-data="hover"` | `<Link>` 자동 (뷰포트 진입 시) |
| 프로그래밍 프리로드 | `preloadData()` / `preloadCode()` | `router.prefetch()` |
