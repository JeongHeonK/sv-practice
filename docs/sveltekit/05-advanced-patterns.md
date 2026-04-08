# 고급 패턴

SvelteKit fetch, REST API 엔드포인트, 페이지네이션, Progressive Enhancement, 스트리밍, 네비게이션 훅, 캐싱/무효화 전략.

---

## 1. API 엔드포인트 — `+server.ts`

> **언제 `+server.ts`를 쓰는가?**
> Form Actions가 가능하면 우선 사용. `+server.ts`는 세 가지 경우에 적합하다:
> - **외부 클라이언트** — 모바일 앱, Postman 등 SvelteKit 외부에서 접근
> - **Webhook/OAuth** — Stripe, GitHub 이벤트 등 외부 서비스 콜백
> - **비HTML 응답** — 파일 다운로드, 바이너리 데이터 등

### 기본 구조

페이지(`+page.svelte`)가 아닌 **JSON 등을 반환하는 API 엔드포인트**를 만들려면 `+server.ts`를 사용한다.

```ts
// src/routes/api/posts/+server.ts
import { json } from '@sveltejs/kit'
import type { RequestHandler } from './$types'

export const GET: RequestHandler = async ({ fetch }) => {
  const response = await fetch('https://dummyjson.com/posts')
  const data = await response.json()

  return json(data)
}
```

- HTTP 메서드 이름(`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)으로 함수를 export
- `RequestHandler` 타입 사용
- 이벤트에서 제공하는 `fetch`를 사용 (일반 `fetch` 대신 — 이유는 후속 강의에서 설명)

### 응답 반환 방식

```ts
// 방법 1: Response 객체 직접 사용
export const GET: RequestHandler = async () => {
  return new Response(JSON.stringify({ message: 'hello' }), {
    status: 200
  })
}

// 방법 2: json() 헬퍼 (권장 — 더 간결)
import { json } from '@sveltejs/kit'

export const GET: RequestHandler = async () => {
  return json({ message: 'hello' })  // 자동 직렬화 + Content-Type 설정
}
```

### 에러 응답

```ts
import { json, error } from '@sveltejs/kit'

export const GET: RequestHandler = async () => {
  // 방법 1: Response로 직접
  return new Response(JSON.stringify({ message: '에러 발생' }), {
    status: 401
  })

  // 방법 2: error() 헬퍼
  error(401, '에러 발생')  // { message: '에러 발생' } JSON + 401 상태
}
```

### 실무 사용 사례

- **Private API 프록시** — 비밀 자격 증명이 필요한 외부 API를 서버에서 호출하고 클라이언트에 결과만 전달
- **Webhook 수신** — Stripe 결제 완료, GitHub 이벤트 등 외부 서비스의 콜백 처리
- **OAuth 콜백** — Google, GitHub, Facebook 등의 인증 리다이렉트 처리
- **JSON API 제공** — 모바일 앱 등 SvelteKit 외부 클라이언트에 데이터 제공

> `+server.ts`는 서버에서만 실행된다. 페이지에 데이터를 로드하는 용도라면 `+page.server.ts`의 load 함수가 더 적합하다. 엔드포인트는 **페이지 외부에서 접근해야 하는 API**에 사용.

### POST 요청 처리

```ts
// src/routes/api/posts/+server.ts
import { json, error, text } from '@sveltejs/kit'
import type { RequestHandler } from './$types'

export const GET: RequestHandler = async ({ fetch }) => {
  const response = await fetch('https://dummyjson.com/posts')
  const data = await response.json()
  return json(data, { status: response.status })
}

export const POST: RequestHandler = async ({ request }) => {
  const post = await request.json()

  if (!post.title) {
    error(400, '게시물 제목은 필수입니다')
  }

  // 실제로는 DB에 삽입
  return json({
    id: crypto.randomUUID(),
    title: post.title
  })
}
```

- `request.json()`으로 요청 본문의 JSON 파싱
- 유효성 검사 실패 시 `error()`로 즉시 에러 응답
- 성공 시 `json()`으로 생성된 데이터 반환 (기본 status 200)

### `fallback` — 나머지 메서드 일괄 처리

`GET`, `POST` 등으로 명시하지 않은 메서드를 한 번에 처리하려면 `fallback`을 export한다.

```ts
export const fallback: RequestHandler = async ({ request }) => {
  return text(`${request.method} 요청 수신됨`)
}
```

명시적 핸들러가 없는 `PUT`, `PATCH`, `DELETE` 등의 요청이 모두 이 함수로 라우팅된다. 명시적 핸들러(`GET`, `POST`)가 있는 메서드는 fallback보다 우선한다.

### `+page.svelte`와 `+server.ts`가 같은 폴더에 있을 때

동일 경로에 페이지(`+page.svelte`)와 엔드포인트(`+server.ts`)가 공존하면, **`Accept` 헤더**에 따라 처리가 갈린다.

| 요청 조건 | 처리 파일 | 반환 |
|--|--|--|
| `Accept: text/html` (브라우저 기본) | `+page.svelte` | HTML |
| `Accept: application/json` 또는 미지정 | `+server.ts` | JSON 등 |
| `PUT`, `PATCH`, `DELETE` 등 | 항상 `+server.ts` | — |

```text
브라우저에서 /blog 접속     → Accept: text/html → +page.svelte (HTML)
fetch('/blog') (JS에서)     → Accept 미지정     → +server.ts (JSON)
Postman에서 GET /blog       → Accept 미지정     → +server.ts (JSON)
```

> 실무에서는 페이지와 엔드포인트를 같은 경로에 두는 것보다, 엔드포인트는 `/api/` 하위에 분리하는 것이 명확하다.

---

## 2. SvelteKit `fetch` — 이벤트에서 제공하는 fetch를 써야 하는 이유

> **왜 event.fetch를 써야 하는가?**
> load 함수와 `+server.ts`에서 제공되는 `fetch`는 일반 전역 `fetch`와 다르다. 세 가지 이유:
> 1. **상대 경로** — `/api/posts` 처럼 전체 URL 없이 호출 가능
> 2. **HTTP 오버헤드 0** — 서버→서버 자체 호출 시 실제 네트워크 요청 없이 함수 직접 실행
> 3. **쿠키 자동 상속** — 원래 페이지 요청의 쿠키/Authorization 헤더 자동 전달

### 1) 상대 경로로 자체 엔드포인트 호출

```ts
// ❌ 일반 fetch — 전체 URL 필요
const res = await fetch('http://localhost:5173/api/posts')

// ✅ SvelteKit fetch — 상대 경로 가능
export const load: PageServerLoad = async ({ fetch }) => {
  const res = await fetch('/api/posts')
  // ...
}
```

### 2) 서버 → 서버 호출 시 HTTP 오버헤드 제거

자체 엔드포인트(`+server.ts`)를 호출할 때, SvelteKit fetch는 **실제 HTTP 요청을 보내지 않고 함수를 직접 호출**한다. 네트워크 오버헤드가 없다.

```text
+page.server.ts → fetch('/api/posts') → +server.ts의 GET 함수 직접 호출
                                         (HTTP 요청 X, 함수 호출 O)
```

### 3) 쿠키 및 인증 헤더 자동 상속

SvelteKit fetch는 원래 **페이지 요청의 쿠키와 Authorization 헤더를 자동으로 전달**한다. 일반 fetch는 이를 수동으로 처리해야 한다.

```ts
// +page.server.ts
export const load: PageServerLoad = async ({ fetch }) => {
  // 사용자의 쿠키가 자동으로 /api/posts 요청에 포함됨
  const res = await fetch('/api/posts')
  // ...
}

// api/posts/+server.ts
export const GET: RequestHandler = async ({ cookies }) => {
  const token = cookies.get('token')  // ✅ 페이지 요청의 쿠키 접근 가능
  // ...
}
```

```ts
// ❌ 일반 fetch — 쿠키 전달 안 됨
const res = await fetch('http://localhost:5173/api/posts')
// api/posts에서 cookies.getAll() → [] (빈 배열)
```

### 실습: 블로그 목록 로드

```ts
// src/routes/(marketing)/blog/+page.server.ts
import { error } from '@sveltejs/kit'
import type { PageServerLoad } from './$types'
import type { PostsResponse } from '$lib/types'

export const load: PageServerLoad = async ({ fetch }) => {
  const response = await fetch('/api/posts')

  if (!response.ok) {
    error(response.status, '게시물을 불러올 수 없습니다')
  }

  const postsResponse: PostsResponse = await response.json()

  return {
    title: 'Blog',
    posts: postsResponse
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
  <a href="/blog/{post.id}">
    <h2>{post.title}</h2>
    <p>{post.body.slice(0, 100)}...</p>
    <div>
      {#each post.tags as tag}
        <span>{tag}</span>
      {/each}
    </div>
  </a>
{/each}
```

---

## 3. URL 검색 파라미터로 페이지네이션

### load 함수에서 `url.searchParams` 접근

```ts
// src/lib/constants.ts
export const POSTS_PER_PAGE = 10
```

```ts
// src/routes/(marketing)/blog/+page.server.ts
import { error } from '@sveltejs/kit'
import type { PageServerLoad } from './$types'
import type { PostsResponse } from '$lib/types'
import { POSTS_PER_PAGE } from '$lib/constants'

export const load: PageServerLoad = async ({ fetch, url }) => {
  const page = +(url.searchParams.get('page') ?? '1')
  const skip = (page - 1) * POSTS_PER_PAGE

  const response = await fetch(
    `https://dummyjson.com/posts?limit=${POSTS_PER_PAGE}&skip=${skip}`
  )

  if (!response.ok) {
    error(response.status, '게시물을 불러올 수 없습니다')
  }

  const postsResponse: PostsResponse = await response.json()

  return { title: 'Blog', posts: postsResponse }
}
```

- `url.searchParams.get('page')` — URL의 `?page=2` 파라미터 접근
- `skip = (page - 1) * POSTS_PER_PAGE` — 페이지 번호를 건너뛸 개수로 변환

### 페이지네이션 UI — `$derived`로 URL 동기화

```svelte
<!-- src/routes/(marketing)/blog/+page.svelte -->
<script lang="ts">
  import { page } from '$app/state'
  import { POSTS_PER_PAGE } from '$lib/constants'
  import type { PageProps } from './$types'

  let { data }: PageProps = $props()

  // ⚠️ $derived 필수 — URL 파라미터 변경 시 스크립트가 재실행되지 않으므로
  let currentPage = $derived(+(page.url.searchParams.get('page') ?? '1'))
  let lastPage = $derived(Math.ceil(data.posts.total / POSTS_PER_PAGE))
</script>

<h1>{data.title}</h1>

{#each data.posts.posts as post}
  <a href="/blog/{post.id}">
    <h2>{post.title}</h2>
    <p>{post.body.slice(0, 100)}...</p>
  </a>
{/each}

<nav>
  {#if currentPage > 1}
    <a href="/blog?page={currentPage - 1}">이전</a>
  {/if}
  {#if currentPage < lastPage}
    <a href="/blog?page={currentPage + 1}">다음</a>
  {/if}
</nav>
```

### 핵심 동작 원리

검색 파라미터(`?page=2`)가 변경되면 SvelteKit이 **load 함수를 자동으로 다시 실행**한다. load 함수가 `url.searchParams`에 의존하고 있으므로, 파라미터 변경이 곧 데이터 재로드를 트리거한다 (무효화).

```text
/blog?page=1 → load 실행 (skip=0)  → 1~10번 게시물
/blog?page=2 → load 재실행 (skip=10) → 11~20번 게시물
/blog?page=3 → load 재실행 (skip=20) → 21~30번 게시물
```

### URL 상태의 장점

검색 파라미터로 상태를 관리하면:
- **공유 가능** — URL을 복사하면 같은 페이지/필터/정렬 상태로 접근
- **북마크 가능** — 브라우저 즐겨찾기로 특정 상태 저장
- **뒤로 가기 작동** — 브라우저 히스토리에 상태가 자동 기록

> 페이지네이션뿐 아니라 필터링, 정렬 등도 동일한 패턴으로 URL 검색 파라미터에 저장할 수 있다. 컴포넌트 내부 상태 대신 URL을 사용하는 것이 권장됨.

---

## 4. Progressive Enhancement — JS 유무에 따른 UI 분기

> **Progressive Enhancement란?**
> JS 없이도 동작하는 기본 기능을 먼저 만들고, JS가 있을 때 더 나은 UX로 업그레이드하는 전략.
> SvelteKit은 SSR 덕분에 JS 비활성화 환경(구형 브라우저, 접근성 도구)에서도 HTML이 표시된다.
> 이 패턴으로 JS 유무에 따라 서로 다른 UI를 제공할 수 있다.

### JS 비활성화 감지: `noscript` 클래스 패턴

`app.html`에서 HTML 태그에 `no-js` 클래스를 기본으로 추가하고, JS가 활성화되면 즉시 제거하는 패턴:

```html
<!-- src/app.html -->
<!doctype html>
<html lang="en" class="no-js">
  <head>
    <script>
      document.documentElement.classList.replace('no-js', 'js')
    </script>
    <meta charset="utf-8" />
    <!-- ... -->
    %sveltekit.head%
  </head>
  <body>
    <div>%sveltekit.body%</div>
  </body>
</html>
```

- JS 활성화 → `<html class="js">` (스크립트 실행)
- JS 비활성화 → `<html class="no-js">` (스크립트 미실행)

### Tailwind 커스텀 variant 정의

```css
/* src/app.css */
@custom-variant no-js (.no-js *);
```

이제 `no-js:` 접두사로 JS가 비활성화된 경우에만 적용되는 스타일을 지정할 수 있다:

```svelte
<!-- JS 비활성화 시에만 표시되는 페이지네이션 -->
<nav class="hidden no-js:flex">
  {#if currentPage > 1}
    <a href="/blog?page={currentPage - 1}">이전</a>
  {/if}
  {#if currentPage < lastPage}
    <a href="/blog?page={currentPage + 1}">다음</a>
  {/if}
</nav>

<!-- JS 활성화 시에만 표시되는 더보기 버튼 -->
<div class="flex no-js:hidden mt-4 justify-center">
  <button>더 로드</button>
</div>
```

### 동작 결과

```text
JS 활성화:    "더 로드" 버튼 표시, 이전/다음 숨김
JS 비활성화:  이전/다음 표시 (URL 기반 페이지네이션), 더 로드 숨김
```

> **Progressive Enhancement**: JS 없이도 기본 페이지네이션이 작동하고, JS가 있으면 더 나은 UX(더 로드 버튼)를 제공. SvelteKit의 SSR + 이 패턴으로 JS 비활성화 환경도 지원할 수 있다.

### "더 로드" 버튼 구현 — 무한 스크롤 패턴

```svelte
<!-- src/routes/(marketing)/blog/+page.svelte -->
<script lang="ts">
  import { page } from '$app/state'
  import { POSTS_PER_PAGE } from '$lib/constants'
  import type { Post, PostsResponse } from '$lib/types'
  import type { PageProps } from './$types'

  let { data }: PageProps = $props()

  // $derived에 쓰기 가능 (Svelte 5) — load 데이터 기반 초기화 + 추가 가능
  let posts: Post[] = $derived(data.posts.posts)

  let isLoading = $state(false)

  // ⚠️ $derived 필수 — URL 변경 시 자동 초기화
  let firstLoadedPage = $derived(+(page.url.searchParams.get('page') ?? '1'))
  let lastLoadedPage = $derived(+(page.url.searchParams.get('page') ?? '1'))

  let lastPage = $derived(Math.ceil(data.posts.total / POSTS_PER_PAGE))

  async function loadMorePosts() {
    isLoading = true

    const skip = lastLoadedPage * POSTS_PER_PAGE
    const response = await fetch(
      `https://dummyjson.com/posts?limit=${POSTS_PER_PAGE}&skip=${skip}`
    )
    const newPosts: PostsResponse = await response.json()

    // $derived에 쓰기 — 기존 게시물에 추가
    posts = [...posts, ...newPosts.posts]

    // 마지막으로 로드한 페이지 업데이트
    lastLoadedPage = newPosts.skip / POSTS_PER_PAGE + 1

    isLoading = false
  }
</script>

<h1>{data.title}</h1>

{#each posts as post}
  <a href="/blog/{post.id}">
    <h2>{post.title}</h2>
    <p>{post.body.slice(0, 100)}...</p>
  </a>
{/each}

<!-- JS 활성화: 더 로드 버튼 -->
{#if lastLoadedPage < lastPage}
  <div class="flex no-js:hidden mt-4 justify-center">
    <button onclick={loadMorePosts} disabled={isLoading}>
      {isLoading ? '로딩 중...' : '더 로드'}
    </button>
  </div>
{:else}
  <p class="no-js:hidden text-center mt-4">더 이상 게시물이 없습니다</p>
{/if}

<!-- JS 비활성화: 기존 페이지네이션 (폴백) -->
<nav class="hidden no-js:flex">
  <!-- ... 이전/다음 버튼 ... -->
</nav>
```

### `$derived`에 쓰기 — Svelte 5 업데이트

Svelte 5에서 `$derived`는 **읽기 + 쓰기 가능**하다. 초기값은 원본 데이터에서 파생되지만, 이후에 직접 값을 할당할 수 있다.

```ts
let posts: Post[] = $derived(data.posts.posts)

// 초기: data.posts.posts에서 파생
// loadMorePosts() 호출 시: posts = [...posts, ...newPosts.posts] 로 직접 할당
// URL 변경 (페이지 이동) 시: 다시 data.posts.posts에서 파생 (자동 초기화)
```

### `$derived` vs `$state` — 왜 `$derived`인가?

```ts
// ❌ $state — URL 변경 시 초기화되지 않음
let firstLoadedPage = $state(currentPage)
// 1페이지 → 2페이지 이동해도 firstLoadedPage가 1로 남아 버그 발생

// ✅ $derived — URL 변경 시 자동 재계산
let firstLoadedPage = $derived(+(page.url.searchParams.get('page') ?? '1'))
// 1페이지 → 2페이지 이동하면 firstLoadedPage도 2로 업데이트
```

URL 파라미터에 의존하는 상태는 반드시 `$derived`로 선언해야 페이지 이동 시 올바르게 초기화된다.

---

## 5. 단일 게시물 — 복수 fetch + 스트리밍

### load 함수에서 게시물 + 댓글 로드

```ts
// src/routes/(marketing)/blog/[id]/+page.ts
import { error } from '@sveltejs/kit'
import type { PageLoad } from './$types'
import type { Post, PostComment } from '$lib/types'

async function fetchPost(fetch: typeof globalThis.fetch, id: string): Promise<Post> {
  console.log('fetching post start')
  const response = await fetch(`https://dummyjson.com/posts/${id}`)

  if (response.status !== 200) {
    error(response.status, '게시물을 로드하지 못했습니다')
  }

  const post: Post = await response.json()
  console.log('fetching post end')
  return post
}

async function fetchComments(fetch: typeof globalThis.fetch, id: string): Promise<PostComment[]> {
  console.log('fetching comments start')
  const response = await fetch(`https://dummyjson.com/posts/${id}/comments`)

  if (!response.ok) {
    return []  // 댓글 로드 실패 시 빈 배열 (에러 미발생)
  }

  const data = await response.json()
  console.log('fetching comments end')
  return data.comments
}

export const load: PageLoad = async ({ fetch, params }) => {
  // ❌ 순차 실행 (waterfall) — 댓글은 게시물 로드 완료까지 대기
  // const post = await fetchPost(fetch, params.id)
  // const comments = await fetchComments(fetch, params.id)

  // ✅ 병렬 실행 — 두 요청 동시 시작
  const [post, comments] = await Promise.all([
    fetchPost(fetch, params.id),
    fetchComments(fetch, params.id)
  ])

  return { post, comments }
}
```

### 에러 처리 전략 차이

| 데이터 | 실패 시 | 이유 |
|--|--|--|
| 게시물 | `error()` → 에러 페이지 표시 | 핵심 데이터 — 없으면 페이지 자체가 의미 없음 |
| 댓글 | 빈 배열 반환 | 부가 데이터 — 없어도 게시물은 표시 가능 |

### 페이지 컴포넌트

```svelte
<!-- src/routes/(marketing)/blog/[id]/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types'
  let { data }: PageProps = $props()
</script>

<h1>{data.post.title}</h1>
<p>{data.post.body}</p>

<div>
  {#each data.post.tags as tag}
    <span>{tag}</span>
  {/each}
</div>

<hr />

{#each data.comments as comment}
  <div>
    <p>{comment.body}</p>
    <span>{comment.user.fullName}</span>
  </div>
{/each}
```

### Universal load에서 네트워크 요청 동작

`+page.ts`(universal)를 사용하면 CSR 네비게이션 시 **브라우저에서 외부 API를 직접 호출**한다. 서버를 거치지 않으므로 한 단계 적은 네트워크 홉.

```text
+page.ts (universal) + CSR 네비게이션:
  브라우저 → dummyjson.com (직접 호출, 서버 미경유)

+page.server.ts (server) + CSR 네비게이션:
  브라우저 → SvelteKit 서버 → dummyjson.com (서버 경유)
```

공개 API라면 `+page.ts`가 더 효율적. 비밀 자격 증명이 필요하면 `+page.server.ts` 필수.

### `Promise.all`로 병렬 실행

독립적인 fetch 요청을 순차로 `await`하면 waterfall이 발생한다:

```text
❌ 순차 (waterfall):
  fetchPost start ─────── fetchPost end
                           fetchComments start ─── fetchComments end
  총 시간: A + B

✅ 병렬 (Promise.all):
  fetchPost start ─────── fetchPost end
  fetchComments start ─── fetchComments end
  총 시간: max(A, B)
```

```ts
// ❌ 순차 — 댓글은 게시물 완료까지 차단됨
const post = await fetchPost(fetch, params.id)
const comments = await fetchComments(fetch, params.id)

// ✅ 병렬 — 동시 시작, 모두 완료되면 진행
const [post, comments] = await Promise.all([
  fetchPost(fetch, params.id),
  fetchComments(fetch, params.id)
])
```

> `Promise.all`은 페이지 렌더링 전에 **모든 데이터가 준비**되어야 할 때 사용. 일부 데이터를 먼저 보여주고 싶다면 → 스트리밍.

### 스트리밍 — `await` 없이 Promise를 반환

핵심 데이터(게시물)만 기다리고, 부가 데이터(댓글)는 **Promise 그대로 반환**하면 SvelteKit이 서버에서 클라이언트로 스트리밍한다. 페이지가 더 빠르게 렌더링된다.

```ts
// src/routes/(marketing)/blog/[id]/+page.server.ts  ← 스트리밍은 server load에서만 가능
import { error } from '@sveltejs/kit'
import type { PageServerLoad } from './$types'
import type { Post, PostComment } from '$lib/types'

export const load: PageServerLoad = async ({ fetch, params }) => {
  // 스트리밍할 Promise를 먼저 생성 (await 없이)
  const commentsPromise = fetchComments(fetch, params.id)

  // 핵심 데이터만 await — 이것이 완료되면 페이지 렌더링 시작
  const post = await fetchPost(fetch, params.id)

  return {
    post,
    title: post.title,
    description: post.body.slice(0, 160),
    comments: commentsPromise  // ← await 없이 Promise 반환 = 스트리밍
  }
}
```

**순서가 중요**: 스트리밍할 Promise를 먼저 생성(`commentsPromise`) → `await fetchPost` → return. 이렇게 하면 댓글 fetch가 게시물 fetch와 **동시에 시작**되면서, 게시물 완료 즉시 페이지가 렌더링된다.

> **⚠️ SEO 주의**: 스트리밍된 데이터는 초기 HTML에 포함되지 않아 검색 로봇이 읽지 못한다.
> 타이틀, 설명, 핵심 콘텐츠 등 SEO에 필요한 데이터는 반드시 `await`할 것.
> 스트리밍은 댓글, 추천 게시물 등 **부가 데이터**에만 사용.

### 컴포넌트에서 `{#await}` 블록 사용

```svelte
<!-- src/routes/(marketing)/blog/[id]/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types'
  let { data }: PageProps = $props()
</script>

<h1>{data.post.title}</h1>
<p>{data.post.body}</p>

<hr />

<!-- data.comments는 Promise → {#await} 블록으로 처리 -->
<div class="no-js:hidden">
  {#await data.comments}
    <!-- 로딩 상태: 스켈레톤 -->
    {#each { length: 4 } as _}
      <div class="skeleton h-16 w-full mb-2"></div>
    {/each}
  {:then comments}
    <!-- 해결됨: 댓글 렌더링 -->
    {#each comments as comment}
      <div>
        <p>{comment.body}</p>
        <span>{comment.user.fullName}</span>
      </div>
    {/each}
  {/await}
</div>
```

### 스트리밍 동작 시각화

```text
[Promise.all — 모두 기다림]
  fetchPost     ─────────── end ─┐
  fetchComments ───── end ────────┤ → 모두 완료 후 페이지 렌더링 시작
                                   총 시간: max(A, B)

[스트리밍 — 핵심만 기다림]
  fetchPost     ─────────── end → 페이지 렌더링 시작 → 게시물 표시
  fetchComments ───── end ───────────────────────────→ 댓글 도착 → 댓글 교체
                                          (로딩 스켈레톤 → 실제 댓글)
  총 시간: fetchPost 완료 시점 = 첫 화면
```

### 스트리밍 주의사항

| 항목 | 설명 |
|--|--|
| **server load 전용** | `+page.server.ts`에서만 동작. `+page.ts`(universal)에서는 불가 |
| **JS 비활성화** | 스트리밍된 데이터가 도착해도 렌더링 불가 → 무한 로딩 상태. `no-js:hidden`으로 숨기는 것을 권장 |
| **SSR 미포함** | 스트리밍 데이터는 초기 HTML에 포함되지 않음 (SEO 불리). 핵심 SEO 데이터는 반드시 `await`할 것 |

---

## 6. 네비게이션 진행률 표시줄 — `beforeNavigate` / `afterNavigate`

### 네비게이션 라이프사이클 함수

`$app/navigation`에서 네비게이션 이벤트에 훅을 걸 수 있다:

```ts
import { beforeNavigate, afterNavigate } from '$app/navigation'

beforeNavigate((navigation) => {
  // navigation.from — 출발 URL
  // navigation.to — 도착 URL
  // navigation.type — 'link' | 'goto' | 'popstate' | ...
  // navigation.cancel() — 네비게이션 취소
  // navigation.complete — 완료 시 resolve되는 Promise
})

afterNavigate((navigation) => {
  // 네비게이션 완료 후 실행
  // navigation.from, navigation.to, navigation.type 동일
})
```

### 진행률 표시줄 구현 (nprogress)

```bash
pnpm add nprogress
pnpm add -D @types/nprogress
```

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import NProgress from 'nprogress'
  import 'nprogress/nprogress.css'
  import { beforeNavigate, afterNavigate } from '$app/navigation'
  import { page } from '$app/state'
  import type { LayoutProps } from './$types'

  let { data, children }: LayoutProps = $props()

  NProgress.configure({ showSpinner: false })

  let loadingTimeout: number

  beforeNavigate(() => {
    // 0.5초 이상 걸릴 때만 진행률 표시 (빠른 네비게이션 시 불필요)
    loadingTimeout = setTimeout(() => NProgress.start(), 500)
  })

  afterNavigate(() => {
    clearTimeout(loadingTimeout)
    NProgress.done()
  })
</script>

<svelte:head>
  <title>{page.data.title ? `${page.data.title} | My App` : 'My App'}</title>
</svelte:head>

{@render children()}
```

### 진행률 바 색상 커스터마이징

```css
/* src/app.css */
#nprogress .bar {
  background: orange !important;
}
```

### 타임아웃 패턴

```text
네비게이션 시작 → 0.5초 타이머 시작
  ├─ 0.5초 이내 완료 → clearTimeout → 진행률 바 표시 안 함
  └─ 0.5초 초과 → NProgress.start() → 진행률 바 표시 → 완료 시 NProgress.done()
```

빠른 네비게이션에서 진행률 바가 깜빡이는 것을 방지한다. 느린 네트워크에서만 표시.

---

## 7. 캐싱/무효화 전략

> **왜 중요한가?**
> load 함수는 네비게이션마다 재실행된다. `invalidateAll()`은 모든 load를 재실행 → 불필요한 서버 요청.
> `depends` + `invalidate`로 재실행 범위를 최소화할 수 있다.

### `depends` — load에 의존성 이름 등록

```ts
// src/routes/(marketing)/blog/+page.server.ts
export const load: PageServerLoad = async ({ depends, fetch }) => {
  depends('app:posts')  // 이 load의 의존성 이름 등록
  const response = await fetch('/api/posts')
  const data = await response.json()
  return { posts: data }
}
```

### `invalidate` vs `invalidateAll` — 선택적 vs 전체 무효화

```ts
import { invalidate, invalidateAll } from '$app/navigation'

// 특정 의존성만 무효화 — 'app:posts'에 depends한 load만 재실행
await invalidate('app:posts')

// 모든 load 재실행 (use:enhance의 기본 동작)
await invalidateAll()
```

| 함수 | 재실행 범위 | 사용 시점 |
|--|--|--|
| `invalidate('app:posts')` | 해당 의존성 load만 | 게시물 목록 갱신 등 특정 데이터만 |
| `invalidateAll()` | 모든 load | 로그아웃, 전역 상태 변경 |

### 실전 패턴 — 낙관적 업데이트 후 정확한 무효화

```svelte
<script lang="ts">
  import { enhance } from '$app/forms'
  import { invalidate } from '$app/navigation'
</script>

<!-- 게시물 삭제 폼 -->
<form
  method="POST"
  action="?/deletePost"
  use:enhance={() => {
    return async ({ result, update }) => {
      if (result.type === 'success') {
        // 게시물 목록만 재로드 (전체 invalidateAll 대신)
        await invalidate('app:posts')
      } else {
        await update()
      }
    }
  }}
>
  <button type="submit">삭제</button>
</form>
```

`use:enhance`의 커스텀 콜백에서 `invalidateAll()` 대신 `invalidate('app:posts')`를 사용하면 게시물 목록 load만 재실행된다. 다른 load 함수(예: 사용자 정보, 레이아웃 데이터)는 재실행되지 않아 불필요한 서버 요청을 줄인다.

---

## React/Next.js 비교

| 개념 | Next.js (App Router) | SvelteKit |
|--|--|--|
| 서버 데이터 로딩 | Server Component (`async function`) | `+page.server.ts` load |
| 클라이언트 데이터 | Client Component + `useEffect`/SWR | `+page.ts` load |
| 레이아웃 데이터 전파 | `layout.tsx`에서 fetch → props drilling 또는 context | `+layout.server.ts` → 자동 병합 |
| 실행 모델 | Server Components = 서버 전용, 순차 | load 함수 = 병렬 실행 |
| 부모 데이터 접근 | props로 전달 또는 별도 fetch | `await parent()` |
| API 엔드포인트 | `app/api/*/route.ts` | `+server.ts` |
| JSON 헬퍼 | `NextResponse.json()` | `json()` from `@sveltejs/kit` |
| 네비게이션 이벤트 | `usePathname` + `useEffect` | `beforeNavigate` / `afterNavigate` |
| 스트리밍 | `<Suspense>` + async Server Component | `await` 없이 Promise 반환 + `{#await}` |
| 선택적 무효화 | `revalidateTag()` / `revalidatePath()` | `invalidate('app:tag')` |
| 전체 무효화 | `router.refresh()` | `invalidateAll()` |
