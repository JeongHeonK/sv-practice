# SvelteKit

파일 컨벤션, 데이터 플로우, load 함수, hooks, Form Actions, 낙관적 UI, 에러 핸들링까지.

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
Request
  → hooks.server.ts        event.locals에 유저 주입
  → +layout.server.ts      return { user }   → 하위 모든 페이지에 전파
  → +page.server.ts        return { posts }  → 해당 페이지만
  → +page.svelte           data prop = { user, posts } (병합 수신)
  → Response (SSR HTML 또는 CSR JSON)
```

키 충돌 시 page load가 layout load를 오버라이드한다.

---

## 3. `load` 함수: server vs universal

| | `+page.server.ts` (ServerLoad) | `+page.ts` (PageLoad) |
|--|--|--|
| 실행 환경 | 서버만 | 서버 + 브라우저 |
| DB/private env | O | X (fetch만 가능) |
| 반환값 | 직렬화 필수 | 클래스 등 자유로움 |
| form actions | O | X |
| CSR navigation | 서버에 JSON 요청 | 브라우저에서 직접 실행 |

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

## 5. `hooks.server.ts` — 네비게이션 가드

### `event.locals`로 인증 상태 전파

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit'

export const handle: Handle = async ({ event, resolve }) => {
  event.locals.user = await getSession(event.cookies.get('token'))
  return resolve(event)
}
```

이후 모든 `load` 함수에서 `event.locals.user`로 접근 가능.

### 미인증 시 `redirect()`

```ts
// src/routes/admin/+layout.server.ts
import { redirect } from '@sveltejs/kit'
import type { LayoutServerLoad } from './$types'

export const load: LayoutServerLoad = async ({ locals }) => {
  if (!locals.user?.isAdmin) redirect(302, '/')
  return { role: 'admin' }
}
```

### 레이아웃 단위 권한 분리

```text
src/routes/
├── +layout.server.ts            ← 전체 공통 (루트 레이아웃)
├── admin/
│   ├── +layout.server.ts        ← /admin/* 공통 (관리자 권한 체크)
│   └── +page.server.ts          ← /admin 페이지만
└── dashboard/
    └── +page.server.ts          ← /dashboard 페이지만
```

---

## 6. Form Actions

### 기본 `<form action>` + `+page.server.ts` actions

Form Actions는 `<form>` 요소로 서버에 POST 데이터를 전송하는 방법. 클라이언트 JS 없이도 동작한다.

```ts
// src/routes/login/+page.server.ts
import { fail, redirect } from '@sveltejs/kit'
import type { Actions } from './$types'
import * as db from '$lib/server/db'

export const actions: Actions = {
  login: async ({ cookies, request }) => {
    const data = await request.formData()
    const email = data.get('email') as string
    const password = data.get('password') as string

    if (!email) {
      return fail(400, { email, missing: true })
    }

    const user = await db.getUser(email)

    if (!user || user.password !== db.hash(password)) {
      return fail(400, { email, incorrect: true })
    }

    cookies.set('sessionid', await db.createSession(user), { path: '/' })
    return { success: true }
  },

  register: async (event) => {
    // 사용자 등록 로직
  }
}
```

### Named actions (`?/login`, `?/register`)

```svelte
<!-- src/routes/login/+page.svelte -->
<form method="POST" action="?/login">
  {#if form?.missing}<p class="error">이메일은 필수입니다</p>{/if}
  {#if form?.incorrect}<p class="error">잘못된 인증 정보</p>{/if}

  <label>
    이메일
    <input name="email" type="email" value={form?.email ?? ''}>
  </label>
  <label>
    비밀번호
    <input name="password" type="password">
  </label>

  <button>로그인</button>
  <button formaction="?/register">회원가입</button>
</form>
```

### `use:enhance` — progressive enhancement

`use:enhance`를 추가하면 전체 페이지 리로드 없이 폼이 제출된다.

```svelte
<script lang="ts">
  import { enhance } from '$app/forms'
  import type { PageProps } from './$types'

  let { data, form }: PageProps = $props()
</script>

<form method="POST" action="?/login" use:enhance>
  <!-- ... -->
</form>
```

`use:enhance`는 기본적으로:
- 성공 시 `invalidateAll`로 모든 데이터 갱신
- redirect 응답 시 `goto` 호출
- 에러 시 가장 가까운 `+error.svelte` 렌더링
- `<form>` 리셋

### 성공 후 `invalidate` / `invalidateAll`로 데이터 갱신

Action이 완료되면 페이지의 `load` 함수가 자동으로 재실행되어 최신 데이터를 가져온다. `use:enhance`가 이를 자동 처리한다.

### 낙관적 UI: `use:enhance` 콜백에서 UI 선업데이트

```svelte
<script lang="ts">
  import { enhance } from '$app/forms'

  let saving = $state(false)
</script>

<form
  method="POST"
  action="?/login"
  use:enhance={({ formData, cancel }) => {
    saving = true

    return async ({ result, update }) => {
      saving = false

      if (result.type === 'success') {
        // 기본 동작 (invalidateAll + form 리셋)
        await update()
      } else if (result.type === 'failure') {
        // 기본 동작 적용 (form 에러 표시)
        await update()
      }
    }
  }}
>
  <button disabled={saving}>
    {saving ? '처리 중...' : '로그인'}
  </button>
</form>
```

---

## 7. 에러 핸들링

### `+error.svelte` — 경로별 에러 UI

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state'
</script>

<h1>{page.status}</h1>
<p>{page.error?.message}</p>
```

가장 가까운 `+error.svelte`가 렌더링된다. 레이아웃 계층에 따라 다른 에러 UI를 표시할 수 있다.

### `error()` helper로 expected error throw

```ts
import { error } from '@sveltejs/kit'
import * as db from '$lib/server/db'

export async function load({ params }) {
  const post = await db.getPost(params.slug)

  if (!post) {
    error(404, { message: '게시글을 찾을 수 없습니다' })
  }

  return { post }
}
```

### `handleError` — 로깅 vs 사용자 메시지 분리

```ts
// src/hooks.server.ts
import type { HandleServerError } from '@sveltejs/kit'

export const handleError: HandleServerError = async ({ error, event, status, message }) => {
  const errorId = crypto.randomUUID()

  // 내부 로깅 (Sentry 등)
  console.error(errorId, error)

  // 사용자에게 노출 (page.error)
  return {
    message: '오류가 발생했습니다',
    errorId
  }
}
```

Expected error(`error()` helper)와 unexpected error(그 외 모든 에러)가 구분된다:
- **Expected**: `error(404, ...)` → 상태 코드와 메시지가 그대로 사용자에게 전달
- **Unexpected**: 민감한 정보 포함 가능 → `handleError`에서 안전한 메시지로 변환

---

## 8. 서버 전용 API: `+server.ts`

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

Form Actions가 선호되지만, JSON API가 필요한 경우 `+server.ts`를 사용한다.

---

## 실무 예시: 로그인 폼 전체 흐름

Form Action + use:enhance + 에러 처리를 조합한 실전 패턴.

### 서버 — `+page.server.ts`

```ts
// src/routes/login/+page.server.ts
import { fail, redirect } from '@sveltejs/kit'
import type { Actions, PageServerLoad } from './$types'
import * as db from '$lib/server/db'

export const load: PageServerLoad = async ({ locals }) => {
  if (locals.user) redirect(302, '/dashboard')
}

export const actions: Actions = {
  default: async ({ cookies, request, url }) => {
    const formData = await request.formData()
    const email = formData.get('email') as string
    const password = formData.get('password') as string

    // 유효성 검사
    if (!email || !password) {
      return fail(400, {
        email,
        error: '이메일과 비밀번호를 모두 입력해주세요'
      })
    }

    // 인증
    const user = await db.getUser(email)
    if (!user || !await db.verifyPassword(user, password)) {
      return fail(400, {
        email,
        error: '이메일 또는 비밀번호가 올바르지 않습니다'
      })
    }

    // 세션 생성
    cookies.set('sessionid', await db.createSession(user), { path: '/' })

    // 리다이렉트
    const redirectTo = url.searchParams.get('redirectTo') ?? '/dashboard'
    redirect(303, redirectTo)
  }
}
```

### 클라이언트 — `+page.svelte`

```svelte
<!-- src/routes/login/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms'
  import type { PageProps } from './$types'

  let { form }: PageProps = $props()
  let submitting = $state(false)
</script>

<svelte:head>
  <title>로그인</title>
</svelte:head>

<h1>로그인</h1>

{#if form?.error}
  <div class="error" role="alert">{form.error}</div>
{/if}

<form
  method="POST"
  use:enhance={() => {
    submitting = true
    return async ({ update }) => {
      submitting = false
      await update()
    }
  }}
>
  <label>
    이메일
    <input name="email" type="email" value={form?.email ?? ''} required>
  </label>
  <label>
    비밀번호
    <input name="password" type="password" required>
  </label>
  <button disabled={submitting}>
    {submitting ? '로그인 중...' : '로그인'}
  </button>
</form>
```

### 인증 가드 — `hooks.server.ts`

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit'
import { redirect } from '@sveltejs/kit'
import * as db from '$lib/server/db'

export const handle: Handle = async ({ event, resolve }) => {
  const sessionId = event.cookies.get('sessionid')
  event.locals.user = sessionId ? await db.getUserFromSession(sessionId) : null

  // 보호된 경로 가드
  if (event.url.pathname.startsWith('/dashboard') && !event.locals.user) {
    redirect(302, `/login?redirectTo=${event.url.pathname}`)
  }

  return resolve(event)
}
```
