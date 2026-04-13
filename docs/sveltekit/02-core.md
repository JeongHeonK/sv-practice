# SvelteKit 핵심

파일 컨벤션, 데이터 플로우, load 함수, hooks, Form Actions, 에러 핸들링까지.

---

## 1. 파일 컨벤션 & 실행 환경 맵

```text
파일                      Server    Client    역할
─────────────────────────────────────────────────────
hooks.server.ts           [O]       [ ]       앱 전역 미들웨어
+layout.server.ts         [O]       [ ]       레이아웃 서버 데이터
+page.server.ts           [O]       [ ]       페이지 서버 데이터 + actions
+server.ts                [O]       [ ]       API endpoint (REST)
─────────────────────────────────────────────────────
+layout.ts / +page.ts     [O]       [O]       유니버설 load
+layout.svelte            [O]       [O]       레이아웃 UI (SSR + CSR)
+page.svelte              [O]       [O]       페이지 UI (SSR + CSR)
+error.svelte             [O]       [O]       에러 바운더리
```

**원칙**: `.server` 접미사 = 서버 전용. 없으면 양쪽에서 실행.

---

## 2. 데이터 플로우 시각화

```text
══════════════════════ 서버 경계 ══════════════════════
handle() → +layout.server.ts → +page.server.ts
               return { user }      return { posts }
                                      │ (직렬화된 data 객체)
──────────────────────────────────────▼───────────────
+layout.svelte  →  +page.svelte      (클라이언트 렌더링)
                   data = { user, posts } (병합 수신)
```

키 충돌 시 page load가 layout load를 오버라이드한다.

---

## 3. `load` 함수: server vs universal

> 상세 비교 → [04-loading-data.md](./04-loading-data.md) Section 1 참고

---

## 4. `+page.svelte`에서 `$props()`로 data 수신

```svelte
<script lang="ts">
  import type { PageProps } from './$types'
  let { data, form }: PageProps = $props()
</script>

<h1>{data.user.name}의 게시글</h1>
{#each data.posts as post}
  <article>{post.title}</article>
{/each}
```

`data`는 layout + page load가 자동 병합된 결과. `form`은 Form Action 결과.

---

## 5. `hooks.server.ts` — 앱 전역 미들웨어

모든 SSR 요청이 이 파일을 거친다. 요청을 가로채고 수정할 수 있는 **앱 전역 미들웨어**.

### `event.locals`로 인증 상태 전파

hooks에서 쿠키를 읽고 세션을 검증한 뒤 `event.locals`에 주입하면, 이후 모든 `load` 함수에서 매번 쿠키를 파싱하지 않아도 된다. **요청 단위 전역 저장소** 역할.

#### better-auth 연동 예시

`auth.api.getSession()`이 쿠키 파싱 + DB 조회 + 유효성 검증을 모두 처리한다:

```ts
// src/hooks.server.ts
import { svelteKit } from 'better-auth/sveltekit'
import { sequence } from '@sveltejs/kit/hooks'
import { auth } from '$lib/server/auth'
import type { Handle } from '@sveltejs/kit'

const authHandle: Handle = async ({ event, resolve }) => {
  const session = await auth.api.getSession({ headers: event.request.headers })
  if (session) event.locals.session = session
  return svelteKit({ auth, event, resolve })  // better-auth 엔드포인트(/api/auth/*) 등록
}

export const handle = sequence(authHandle)
```

> `svelteKit()` 핸들러가 없으면 `/api/auth/verify-email` 등 better-auth 엔드포인트가 404를 반환한다.

### `app.d.ts`에서 `App.Locals` 타입 선언

```ts
// src/app.d.ts
import type { Session, User } from 'better-auth'

declare global {
  namespace App {
    interface Locals {
      session: { session: Session; user: User } | null
    }
  }
}
export {}
```

> better-auth를 쓰지 않는 경우 `typeof users.$inferSelect` (Drizzle) 또는 `Prisma.UserGetPayload<{}>` (Prisma) 등 DB 스키마 타입을 직접 사용. 핵심은 **DB 스키마와 `App.Locals` 타입을 동기화**하는 것.

### hooks와 prerender의 관계

hooks는 **SSR 요청에서만 실행**된다. `prerender = true`인 페이지는 빌드 시 정적 HTML로 생성되므로 hooks를 타지 않는다.

```text
/about     → prerender = true  → 빌드 시 정적 HTML, hooks 안 탐
/dashboard → SSR               → 요청마다 hooks 실행, event.locals 사용 가능
```

> prerender된 페이지에서 유저 정보가 필요하면 클라이언트에서 API(`+server.ts`)를 fetch한다.

### 보호된 라우트 — layout으로 인증 가드

`(괄호)` 레이아웃 그룹 + layout.server.ts 하나로 인증 가드를 구성한다:

```text
src/routes/
├── (public)/                    ← prerender = true, hooks 안 탐
│   ├── about/
│   └── pricing/
├── (app)/                       ← SSR, hooks에서 쿠키 검사
│   ├── +layout.server.ts        ← 인증 체크 (이것 하나로 끝)
│   ├── dashboard/+page.server.ts ← 인증 걱정 없이 데이터만 load
│   └── settings/+page.server.ts
```

```ts
// routes/(app)/+layout.server.ts
import { redirect } from '@sveltejs/kit'
import type { LayoutServerLoad } from './$types'

export const load: LayoutServerLoad = async ({ locals }) => {
  if (!locals.user) throw redirect(303, '/login')
  return { user: locals.user }
}
```

layout에서 한번만 체크하면 하위 모든 page는 인증이 보장된 상태. 관리자 권한 같은 추가 체크는 `admin/+layout.server.ts`에 중첩한다.

### 다중 hooks: `sequence()`

```ts
import { sequence } from '@sveltejs/kit/hooks'

const auth: Handle = async ({ event, resolve }) => {
  event.locals.user = await getSession(event.cookies)
  return resolve(event)
}

const logger: Handle = async ({ event, resolve }) => {
  console.log(`${event.request.method} ${event.url.pathname}`)
  return resolve(event)
}

export const handle = sequence(auth, logger)
```

### handle의 3가지 패턴

```text
패턴 1: resolve 전에 event 수정
  → event.locals에 데이터 주입 → resolve(event)

패턴 2: resolve 후에 응답 수정
  → const response = await resolve(event) → 헤더 추가 → return response

패턴 3: resolve 우회 — 직접 응답 반환
  → if (pathname === '/health') return new Response('OK')
  → resolve 미호출 시 SvelteKit 기본 처리 전혀 실행 안 됨
```

### locals vs 쿠키

| 특성 | `event.locals` | `event.cookies` |
|---|---|---|
| 생존 범위 | 요청 1회 한정 | 브라우저에 지속 |
| 저장 위치 | 서버 메모리 (요청 중) | 클라이언트 쿠키 |
| 용도 | 요청 처리 중 임시 데이터 전달 | 인증 토큰 등 영구 데이터 |

> **핵심**: "쿠키에서 토큰 확인 → 사용자 정보 조회 → locals에 주입"하면, 모든 load 함수에서 매번 쿠키를 파싱하지 않아도 된다.

---

## 6. Form Actions

### 기본 개념

HTML `<form>`은 JS 없이도 동작하는 웹의 기본 전송 메커니즘이다. `use:enhance`는 JS가 있을 때만 fetch로 업그레이드 — 이것이 **Progressive Enhancement**.

| 상황 | 동작 |
|--|--|
| JS 비활성화 | 전체 페이지 리로드 (기본 HTML form) |
| JS 활성화 | fetch로 파셜 업데이트, 리로드 없음 |

### Form Actions 라이프사이클

```text
<form method="POST" action="?/login">  — submit
  │
  ▼
+page.server.ts  (actions.login)
  │
  ├─ fail(400, { ... })    →  form 반응형 상태 업데이트  →  컴포넌트 리렌더
  ├─ redirect(303, '/')    →  goto('/')
  └─ return { success }    →  invalidateAll()  →  load 재실행  →  컴포넌트 갱신
```

### 서버: `+page.server.ts`에 actions 정의

```ts
// src/routes/login/+page.server.ts
import { fail, redirect } from '@sveltejs/kit'
import type { Actions } from './$types'

export const actions: Actions = {
  login: async ({ cookies, request }) => {
    const data = await request.formData()
    const email = data.get('email') as string
    const password = data.get('password') as string

    if (!email) return fail(400, { email, missing: true })

    const user = await db.getUser(email)
    if (!user || user.password !== db.hash(password)) {
      return fail(400, { email, incorrect: true })
    }

    cookies.set('sessionid', await db.createSession(user), { path: '/' })
    return { success: true }
  },

  register: async (event) => { /* ... */ }
}
```

### 클라이언트: `use:enhance`로 Progressive Enhancement

```svelte
<script lang="ts">
  import { enhance } from '$app/forms'
  import type { PageProps } from './$types'
  let { form }: PageProps = $props()
</script>

<form method="POST" action="?/login" use:enhance>
  {#if form?.missing}<p class="error">이메일은 필수입니다</p>{/if}
  {#if form?.incorrect}<p class="error">잘못된 인증 정보</p>{/if}

  <input name="email" type="email" value={form?.email ?? ''}>
  <input name="password" type="password">
  <button>로그인</button>
  <button formaction="?/register">회원가입</button>
</form>
```

`use:enhance` 기본 동작: 성공 시 `invalidateAll`, redirect 시 `goto`, 에러 시 `+error.svelte`, form 리셋.

### action에서 쿠키 수동 설정

외부 auth 라이브러리(better-auth 등)는 로그인 응답의 `Set-Cookie` 헤더만 반환하고, 브라우저에 쿠키를 심는 것은 직접 해야 한다.

```ts
// +page.server.ts action
export const actions = {
  login: async ({ request, cookies }) => {
    // asResponse: true → 응답 객체 자체를 반환 (Set-Cookie 헤더 포함)
    const response = await auth.api.signInEmail({
      headers: request.headers,
      body: { email, password },
      asResponse: true
    })

    if (response.status !== 200) {
      const { message } = await response.json()
      return message(form, message ?? '오류가 발생했습니다.', { status: 400 })
    }

    // Set-Cookie 헤더 파싱 후 SvelteKit cookies API로 직접 설정
    import { parse } from 'cookie'  // SvelteKit 의존성, 별도 설치 불필요
    const setCookieHeader = response.headers.get('Set-Cookie')!
    const parsed = parse(setCookieHeader)

    cookies.set('better-auth.session_token', parsed['better-auth.session_token'], {
      path: '/',
      httpOnly: true,
      sameSite: 'lax',
      secure: !dev,                          // dev: $app/environment
      maxAge: +parsed['Max-Age']
    })
  }
}
```

> `dev`는 `import { dev } from '$app/environment'`로 가져온다. 로컬 개발은 HTTP이므로 `secure: false`가 필요.

### action에서 쿠키 삭제 (로그아웃)

```ts
export const actions = {
  logout: async ({ request, cookies, locals }) => {
    const response = await auth.api.signOut({
      headers: request.headers,
      asResponse: true
    })

    if (response.status !== 200) return fail(500, { message: '로그아웃 실패' })

    cookies.delete('better-auth.session_token', { path: '/' })

    // redirect 없이 locals에 의존하는 경우 수동 초기화 필요 (아래 참고)
    locals.session = null

    redirect(303, '/auth/sign-in')
  }
}
```

### ⚠️ handle hook과 action의 실행 순서

`handle` hook은 action 코드보다 **먼저** 실행된다. 로그아웃 action이 실행될 시점에 `event.locals.session`은 여전히 이전 세션값으로 채워져 있다.

```text
요청 도착
  → handle hook 실행  (locals.session = 기존 세션으로 주입)
  → action 실행       (이 시점에 locals.session은 아직 유효)
  → 로그아웃 처리
  → redirect          (다음 요청에서 hook 재실행 → locals 자동 갱신)
```

| 상황 | locals 수동 초기화 필요? |
|---|---|
| 로그아웃 후 redirect | 불필요 — 다음 요청에서 hook이 재실행됨 |
| 같은 요청 안에서 locals에 의존 | 필요 — `locals.session = null` 직접 처리 |

### ⚠️ `redirect()`를 `try/catch` 안에 넣지 말 것

`redirect()`는 내부적으로 **에러를 throw**한다. `try/catch` 안에서 호출하면 catch가 잡아버린다. 항상 `try/catch` **바깥**에서 호출할 것.

### `use:enhance` 콜백 — 커스텀 동작

콜백을 추가하면 로딩 상태, 낙관적 UI, 커스텀 에러 처리가 가능하다.

```svelte
<form
  method="POST"
  use:enhance={({ formData, cancel }) => {
    // --- 제출 시점 (서버 응답 전) ---
    isSubmitting = true

    return async ({ result, update }) => {
      // --- 응답 수신 후 ---
      isSubmitting = false

      if (result.type === 'failure') {
        applyAction(result)  // form prop + page.status 업데이트
      } else {
        update()  // 기본 동작 전부 실행
      }
    }
  }}
>
```

| 함수 | 역할 |
|---|---|
| `update()` | **모든** 기본 동작 실행 (form 업데이트, invalidateAll, 리다이렉트 등) |
| `applyAction(result)` | **해당 result.type에 맞는** 기본 동작만 실행 |

> 콜백에서 `return` 함수를 정의하면 **기본 동작이 비활성화**된다. 필요한 result type에는 반드시 `update()` 또는 `applyAction(result)`를 호출해야 한다.

### 낙관적 UI

서버 응답 전에 UI를 먼저 업데이트하고, 서버 결과에 따라 확정/롤백하는 패턴. `use:enhance` 콜백의 **return 전**에서 UI를 바꾸고, **return 안**에서 서버 결과에 따라 처리한다.

### 폼 유효성 검사 — zod 연동

```ts
// +page.server.ts
const LoginSchema = z.object({
  email: z.string().email('유효한 이메일을 입력하세요'),
  password: z.string().min(8, '비밀번호는 8자 이상')
})

export const actions: Actions = {
  login: async ({ request }) => {
    const formData = await request.formData()
    const result = LoginSchema.safeParse({
      email: formData.get('email'),
      password: formData.get('password')
    })

    if (!result.success) {
      return fail(400, { errors: result.error.flatten().fieldErrors })
    }
    // 검증 통과 → 로그인 처리
  }
}
```

`safeParse`는 예외를 던지지 않고 `{ success, data, error }` 객체를 반환한다. `error.flatten().fieldErrors`로 필드명별 에러 메시지 배열을 얻는다.

> 폼이 많아지면 **Superforms** + **Formsnap** 라이브러리로 검증/상태관리 보일러플레이트를 줄일 수 있다. Zod 스키마를 서버/클라이언트 양쪽에서 재사용하고, 필드별 에러 표시를 자동화한다.

---

## 7. 에러 핸들링

### Expected vs Unexpected 에러

```text
에러 발생
  │
  ├─ error() helper      →  Expected Error
  │   └─ 상태코드 + 메시지 그대로  →  page.error 노출
  │                                    └─ +error.svelte 렌더링
  │
  └─ 기타 throw (예외)   →  Unexpected Error
      └─ handleError() 통과
          ├─ 내부: errorId + Sentry 등 로깅
          └─ 외부(page.error): "오류가 발생했습니다" (안전한 메시지)
                                 └─ +error.svelte 렌더링
```

- **Expected**: `error(404, ...)` → 상태 코드와 메시지가 그대로 사용자에게 전달
- **Unexpected**: 민감한 정보 포함 가능 → `handleError`에서 안전한 메시지로 변환

### `error()` helper

```ts
import { error } from '@sveltejs/kit'

export async function load({ params }) {
  const post = await db.getPost(params.slug)
  if (!post) error(404, { message: '게시글을 찾을 수 없습니다' })
  return { post }
}
```

### `handleError` — 로깅 vs 사용자 메시지 분리

```ts
// src/hooks.server.ts
export const handleError: HandleServerError = async ({ error, event }) => {
  const errorId = crypto.randomUUID()
  console.error(errorId, error)  // 내부 로깅

  return {
    message: '오류가 발생했습니다',  // 사용자 노출
    errorId
  }
}
```

### `+error.svelte` — 에러 UI

가장 가까운 `+error.svelte`가 렌더링된다. 레이아웃 계층에 따라 다른 에러 UI를 표시할 수 있다.

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state'
</script>

<h1>{page.status}</h1>
<p>{page.error?.message}</p>
<button onclick={() => location.reload()}>다시 시도</button>
<button onclick={() => goto('/')}>홈으로</button>
```

> 레이아웃 load 함수에서 에러가 발생하면, 같은 레벨이 아닌 **상위 레벨**의 `+error.svelte`가 사용된다 — 같은 레벨의 에러 페이지도 그 레이아웃에 의존하므로 렌더링할 수 없기 때문.

---

## 8. 서버 전용 API: `+server.ts`

> Form Actions가 가능하면 우선 사용. `+server.ts`는 외부 클라이언트(모바일 앱, Webhook)나 비HTML 응답이 필요할 때 적합하다.

```ts
// src/routes/api/users/+server.ts
import { json } from '@sveltejs/kit'
import type { RequestHandler } from './$types'

export const GET: RequestHandler = async ({ locals }) => {
  const users = await db.getUsers()
  return json(users)
}

export const POST: RequestHandler = async ({ request }) => {
  const data = await request.json()
  const user = await db.createUser(data)
  return json(user, { status: 201 })
}
```

---

## 9. SSR & 하이드레이션

```text
1. 서버에서 HTML 렌더링 → 브라우저에 전송
2. 브라우저에서 JS 로드 → 하이드레이션 (이벤트 핸들러 연결)
3. 이후 네비게이션은 클라이언트 사이드로 처리
```

SvelteKit은 기본적으로 SSR을 수행한다. JS가 비활성화되어도 서버 렌더링된 HTML이 표시된다. 페이지별 SSR 비활성화는 `export const ssr = false`로 설정한다.

---

## 10. 페이지 옵션

```ts
// +page.ts 또는 +layout.ts에서 export
export const ssr = true       // false → CSR only
export const csr = true       // false → JS 미전송 (정적 HTML만)
export const prerender = false // true → 빌드 시 정적 HTML 생성
export const trailingSlash = 'never'  // 'always' | 'ignore' | 'never'(기본값)
```

| 조합 | 결과 | 용도 |
|--|--|--|
| `ssr + csr` (기본값) | SSR → 하이드레이션 | 일반 동적 페이지 |
| `ssr + !csr` | SSR만, JS 없음 | 정적 콘텐츠 (블로그 등) |
| `!ssr + csr` | CSR only | 관리자 대시보드, canvas 앱 |
| `prerender` | 빌드 시 HTML 생성 | 마케팅 페이지, 문서 |

`trailingSlash`: `never`(기본) = 슬래시 제거 후 redirect, `always` = 슬래시 추가, `ignore` = 그대로. `+server.ts` 엔드포인트에서도 사용 가능한 유일한 옵션.

레이아웃에 설정하면 하위 모든 페이지에 cascade된다. 특정 페이지에서 다시 export하면 override 가능.

### 동적 라우트 prerender — `entries` 함수

`prerender = true`인 동적 라우트(`[id]`)는 SvelteKit이 어떤 파라미터를 생성해야 할지 알 수 없다. 두 가지 방법으로 지정한다.

**방법 1: 파일에서 `entries` export (권장, async 가능)**

```ts
// blog/[id]/+page.server.ts
import type { EntryGenerator } from './$types'

export const prerender = true

// 빌드 시 어떤 id를 pre-render할지 반환
export const entries: EntryGenerator = async () => {
  const posts = await fetchAllPostIds()  // CMS/DB에서 ID 목록 가져오기
  return posts.map(id => ({ id: String(id) }))
}
```

**방법 2: `svelte.config.js`의 `prerender.entries`**

```js
// svelte.config.js
kit: {
  prerender: {
    crawl: true,                    // 링크 자동 탐색 (기본값 true)
    entries: ['*', '/blog/6']       // '*' = 정적 라우트 전체 + 추가 지정
  }
}
```

> `crawl: true`일 때 — 블로그 목록 페이지를 생성하면서 발견된 링크도 자동으로 prerender. 단 동적 라우트는 링크가 실제로 존재해야 탐색 가능.

### `prerender = 'auto'` — 점진적 정적 생성

빌드 시 발견된 라우트는 정적 HTML로, 나머지는 SSR로 fallback:

```ts
export const prerender = 'auto'
// 빌드 시 크롤에서 발견된 ID → 정적 HTML
// 그 외 ID → 요청 시 SSR 실행 (404 대신 동적 렌더링)
```

| 값 | 동작 |
|---|---|
| `true` | 전부 정적 HTML, 미생성 ID는 404 |
| `false` | 항상 SSR |
| `'auto'` | 발견된 것만 정적, 나머지는 SSR fallback |

---

## 11. 데이터 무효화 (Invalidation)

`use:enhance`는 기본적으로 `invalidateAll()`을 호출하여 모든 `load` 함수를 재실행한다. 선택적 무효화가 필요할 때는 `depends`와 `invalidate`를 조합한다.

```ts
// +page.server.ts의 load
export const load: PageServerLoad = async ({ depends }) => {
  depends('app:posts')
  return { posts: await db.getPosts() }
}
```

```ts
import { invalidate, invalidateAll } from '$app/navigation'

invalidate('app:posts')  // 'app:posts'에 의존하는 load만 재실행
invalidateAll()          // 모든 load 재실행
```

> 상세 내용 → [07-load-invalidation.md](./07-load-invalidation.md)

---

## 12. 네비게이션

SvelteKit은 별도 `<Link>` 컴포넌트가 없다. 일반 `<a>` 태그를 사용하면 자동으로 클라이언트 사이드 네비게이션으로 가로챈다.

```svelte
<script>
  import { goto } from '$app/navigation'
  import { page } from '$app/state'
</script>

<a href="/about" class:active={page.url.pathname === '/about'}>About</a>
<button onclick={() => goto('/about')}>이동</button>
```

---

## React/Next.js 비교

| 개념 | Next.js (App Router) | SvelteKit |
|--|--|--|
| 미들웨어 | `middleware.ts` (요청만 수정) | `hooks.server.ts` handle (요청+응답 수정) |
| 인증 상태 전파 | 쿠키/세션 직접 전달 | `event.locals` (요청 단위 저장소) |
| 폼 처리 | Server Actions (`'use server'`) | Form Actions (`+page.server.ts` actions) |
| Progressive Enhancement | 기본 미지원 | `<form>` 기본 동작 + `use:enhance` 업그레이드 |
| 에러 바운더리 | `error.tsx` (React Error Boundary) | `+error.svelte` (라우트별 배치) |
| 링크 컴포넌트 | `<Link href="/">` | `<a href="/">` (자동 가로채기) |
| 프로그래매틱 이동 | `useRouter().push()` | `goto()` |
| SSR 비활성화 | `'use client'` + dynamic import | `export const ssr = false` |
| 정적 생성 | `generateStaticParams` | `export const prerender = true` |
| 데이터 무효화 | `revalidatePath` / `revalidateTag` | `invalidate(url)` / `invalidateAll()` |
