# SvelteKit 핵심

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
보통은 다르게 사용함. tricky하게 사용하지 말 것

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

Next.js의 `middleware.ts`와 같은 역할. 모든 SSR 요청이 이 파일을 거친다.

### `event.locals`로 인증 상태 전파

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit'

export const handle: Handle = async ({ event, resolve }) => {
  event.locals.user = await getSession(event.cookies.get('token'))
  return resolve(event)
}
```

이후 모든 `load` 함수에서 `event.locals.user`로 접근 가능. **요청 단위 전역 저장소** 역할.

```text
hooks.server.ts에서 쿠키 검사 (한번만)
  → event.locals.user에 저장
  → 이후 모든 server load에서 꺼내 쓰기만 하면 됨
```

### hooks와 prerender의 관계

hooks는 **SSR 요청에서만 실행**된다. `prerender = true`인 페이지는 빌드 시 정적 HTML로 생성되므로 요청 자체가 없고, hooks를 타지 않는다.

```text
/about    → prerender = true  → 빌드 시 정적 HTML, hooks 안 탐
/dashboard → SSR              → 요청마다 hooks 실행, event.locals 사용 가능
/pricing  → prerender = true  → 정적 HTML, hooks 안 탐
```

Next.js는 middleware 존재 시 해당 라우트가 dynamic 강제되지만, SvelteKit은 **페이지별로 독립 제어**하므로 정적 페이지의 성능 이점을 포기할 필요가 없다.

> prerender된 페이지에서 유저 정보가 필요하면 클라이언트에서 API(`+server.ts`)를 fetch하는 방식을 사용한다.

### 보호된 라우트 — layout으로 인증 가드

`(괄호)` 레이아웃 그룹 + layout.server.ts 하나로 인증 가드를 구성한다:

```text
src/routes/
├── (public)/                    ← prerender = true, hooks 안 탐
│   ├── about/
│   └── pricing/
├── (app)/                       ← SSR, hooks에서 쿠키 검사
│   ├── +layout.server.ts        ← 인증 체크 (이것 하나로 끝)
│   ├── +layout.svelte           ← 공통 UI (사이드바 등)
│   ├── dashboard/
│   │   └── +page.server.ts      ← 인증 걱정 없이 데이터만 load
│   └── settings/
│       └── +page.server.ts      ← 여기도 마찬가지
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

layout에서 한번만 체크하면 하위 모든 page는 인증이 보장된 상태에서 자기 데이터 load만 신경 쓰면 된다. 각 page에서 일일이 인증 체크할 필요 없음.

### 레이아웃 단위 권한 분리 (관리자 등)

```text
src/routes/
├── (app)/
│   ├── +layout.server.ts        ← 일반 유저 인증
│   ├── admin/
│   │   ├── +layout.server.ts    ← /admin/* 관리자 권한 추가 체크
│   │   └── +page.server.ts
│   └── dashboard/
│       └── +page.server.ts
```

### 다중 hooks: `sequence()`

```ts
// src/hooks.server.ts
import { sequence } from '@sveltejs/kit/hooks'
import type { Handle } from '@sveltejs/kit'

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

`sequence`는 여러 `handle` 함수를 순서대로 체이닝한다. 각 함수가 `resolve(event)`를 호출하면 다음 함수로 넘어간다. 실무에서는 인증 + 로깅 + i18n + rate limiting 등을 분리하여 관리한다.

### handle의 3가지 패턴

#### 패턴 1: resolve 전에 event 수정

```ts
export const handle: Handle = async ({ event, resolve }) => {
  // resolve 호출 전 — event를 수정하여 load 함수에 데이터 전달
  const token = event.cookies.get('token')
  event.locals.user = token ? await getUser(token) : null

  return await resolve(event)
}
```

#### 패턴 2: resolve 후에 응답 수정

```ts
export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event)

  // resolve 호출 후 — 응답 헤더 추가
  response.headers.set('X-Custom-Header', 'my-value')

  return response
}
```

#### 패턴 3: resolve를 우회하여 직접 응답 반환

```ts
export const handle: Handle = async ({ event, resolve }) => {
  if (event.url.pathname === '/health') {
    return new Response('OK', { status: 200 })
  }

  return await resolve(event)
}
```

> resolve를 호출하지 않으면 load 함수, 페이지 렌더링 등 SvelteKit의 기본 처리가 **전혀 실행되지 않는다**.

### `event.isDataRequest` — 요청 유형 구분

| 상황 | `isDataRequest` | 설명 |
|---|---|---|
| 브라우저에서 직접 URL 접속 (SSR) | `false` | 서버에서 페이지 전체를 렌더링 |
| 클라이언트 측 내비게이션 | `true` | `__data.json` fetch 요청으로 데이터만 요청 |
| Postman 등 외부 API 호출 | `false` | +server.ts 엔드포인트 직접 호출 |

### locals vs 쿠키

| 특성 | `event.locals` | `event.cookies` |
|---|---|---|
| 생존 범위 | 요청 1회 한정 | 브라우저에 지속 |
| 저장 위치 | 서버 메모리 (요청 중) | 클라이언트 쿠키 |
| 용도 | 요청 처리 중 임시 데이터 전달 | 인증 토큰 등 영구 데이터 |
| 보안 | 서버에만 존재, 외부 노출 없음 | 클라이언트에서 접근 가능 |

> **핵심**: "쿠키에서 토큰 확인 → 사용자 정보 조회 → locals에 주입"하면, 모든 load 함수에서 매번 쿠키를 파싱하지 않아도 된다.

---

## 6. Form Actions

### 기본 `<form action>` + `+page.server.ts` actions

> **왜 Form Actions인가?**
> HTML `<form>`은 JS 없이도 동작하는 웹의 기본 전송 메커니즘이다.
> `use:enhance`는 JS가 있을 때만 fetch로 업그레이드 — 이것이 **Progressive Enhancement**.
>
> | 상황 | 동작 |
> |--|--|
> | JS 비활성화 | 전체 페이지 리로드 (기본 HTML form 동작) |
> | JS 활성화 | fetch로 파셜 업데이트, 리로드 없음 |
>
> 기본 동작이 먼저 보장되므로 네트워크 불량, 구형 브라우저에서도 폼이 작동한다.

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

### Form Actions 라이프사이클

```text
<form method="POST" action="?/login">  — submit
  │
  ▼
+page.server.ts  (actions.login)
  │
  ├─ fail(400, { ... })    →  form 반응형 상태 업데이트  →  컴포넌트 리렌더
  │
  ├─ redirect(302, '/')    →  goto('/')
  │
  └─ return { success }    →  invalidateAll()  →  load 재실행  →  컴포넌트 갱신
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

핵심: **return 전**(콜백 바깥)에서 UI를 먼저 바꾸고, **return 안**에서 서버 결과에 따라 확정/롤백.

```svelte
<script lang="ts">
  import { enhance } from '$app/forms'
  import { invalidateAll } from '$app/navigation'

  let { data } = $props()
  let todos = $state(data.todos)
</script>

<form
  method="POST"
  action="?/addTodo"
  use:enhance={({ formData }) => {
    // ✅ 서버 응답 전에 즉시 UI에 추가 (낙관적)
    const text = formData.get('text')
    const tempId = crypto.randomUUID()
    todos.push({ id: tempId, text, pending: true })

    return async ({ result }) => {
      if (result.type === 'success') {
        // 서버 확인 → 실제 데이터로 교체
        await invalidateAll()
      } else {
        // 실패 → 롤백
        todos = todos.filter(t => t.id !== tempId)
      }
    }
  }}
>
  <input name="text" />
  <button>추가</button>
</form>

{#each todos as todo}
  <li class:pending={todo.pending}>{todo.text}</li>
{/each}
```

```
흐름:
  1. 폼 제출 → 즉시 todos에 추가 (pending: true)
  2. 서버 응답 대기 중에도 UI에 이미 보임
  3. 성공 → invalidateAll()로 실제 데이터 반영
     실패 → 임시 항목 제거 (롤백)
```

> **use:enhance 없이 (`<form>` 기본 동작)**: 전체 페이지 리로드. `use:enhance`를 쓰면 JS로 제출하되 기본적으로 `invalidateAll()` + form 리셋을 해준다. 콜백을 추가하면 낙관적 UI까지 구현 가능.

---

## 6.5 폼 유효성 검사 — zod 연동

서버에서 폼 데이터를 검사하지 않으면 잘못된 데이터가 DB에 저장될 수 있다. zod를 사용하면 타입 안전하게 검사하고 필드별 에러 메시지를 반환할 수 있다.

```bash
pnpm add zod
```

```ts
// src/routes/login/+page.server.ts
import { z } from 'zod'
import { fail, redirect } from '@sveltejs/kit'
import type { Actions } from './$types'

const LoginSchema = z.object({
  email: z.string().email('유효한 이메일을 입력하세요'),
  password: z.string().min(8, '비밀번호는 8자 이상이어야 합니다')
})

export const actions: Actions = {
  login: async ({ request, cookies }) => {
    const formData = await request.formData()

    const result = LoginSchema.safeParse({
      email: formData.get('email'),
      password: formData.get('password')
    })

    if (!result.success) {
      // flatten()으로 필드별 에러 메시지 추출
      return fail(400, {
        errors: result.error.flatten().fieldErrors
      })
    }

    // 검증 통과 → 로그인 처리
    const { email, password } = result.data
    // ... DB 조회, 세션 생성 등
    redirect(302, '/dashboard')
  }
}
```

```svelte
<!-- src/routes/login/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms'
  import type { PageProps } from './$types'
  let { form }: PageProps = $props()
</script>

<form method="POST" action="?/login" use:enhance>
  <label>
    이메일
    <input name="email" type="email" />
    {#if form?.errors?.email}
      <p class="text-red-500 text-sm">{form.errors.email[0]}</p>
    {/if}
  </label>

  <label>
    비밀번호
    <input name="password" type="password" />
    {#if form?.errors?.password}
      <p class="text-red-500 text-sm">{form.errors.password[0]}</p>
    {/if}
  </label>

  <button>로그인</button>
</form>
```

**데이터 흐름**:
```text
form submit
  → actions.login
      → LoginSchema.safeParse()
          ├─ 실패 → fail(400, { errors: { email: [...], password: [...] } })
          │            → form 반응형 상태 → 각 필드 아래 에러 표시
          └─ 성공 → 로그인 처리 → redirect(302, '/dashboard')
```

`safeParse`는 예외를 던지지 않고 `{ success, data, error }` 객체를 반환한다. `error.flatten().fieldErrors`로 필드명별 에러 메시지 배열을 얻는다.

---

## 7. 에러 핸들링

### `+error.svelte` — 경로별 에러 UI

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

## 7.5 에러 복구 UX

`+error.svelte`에 재시도/홈 복귀 버튼을 추가하면 사용자가 막힌 상태에서 벗어날 수 있다.

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state'
  import { goto } from '$app/navigation'
</script>

<div class="error-page">
  <h1>오류 {page.status}</h1>
  <p>{page.error?.message ?? '알 수 없는 오류가 발생했습니다'}</p>

  <div class="actions">
    <button onclick={() => location.reload()}>다시 시도</button>
    <button onclick={() => goto('/')}>홈으로</button>
  </div>
</div>
```

| 버튼 | 동작 | 적합한 상황 |
|--|--|--|
| `location.reload()` | 현재 URL 재요청 — load 함수 재실행 | 일시적 네트워크 오류 |
| `goto('/')` | 안전한 경로로 이동 | 권한 없음, 삭제된 리소스 |

> **레이아웃 계층별 `+error.svelte`**: 더 구체적인 경로에 `+error.svelte`를 두면 해당 경로의 에러만 담당한다. 루트의 `+error.svelte`는 모든 미처리 에러의 폴백이 된다.
>
> ```text
> routes/+error.svelte           ← 전체 폴백 (항상 필요)
> routes/admin/+error.svelte     ← /admin/* 에러 전용
> routes/blog/+error.svelte      ← /blog/* 에러 전용
> ```

---

## 8. 서버 전용 API: `+server.ts`

> Form Actions가 가능하면 우선 사용. `+server.ts`는 외부 클라이언트(모바일 앱, Webhook)나 비HTML 응답이 필요할 때 적합하다. 상세 내용 → [05-advanced-patterns.md](./05-advanced-patterns.md) Section 1.

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

## 9. SSR & 하이드레이션

> **왜 SSR인가?**
> 초기 HTML을 서버에서 생성하면:
> - 검색 로봇(Google 등)이 콘텐츠를 읽을 수 있다 (SEO)
> - JS 비활성화 환경에서도 콘텐츠가 표시된다 (접근성)
>
> **하이드레이션**이란 서버에서 만든 HTML에 JS 이벤트 핸들러를 연결하는 과정이다.
> SvelteKit은 기본적으로 SSR을 수행하며, 이후 네비게이션은 클라이언트 사이드로 처리한다.

```text
1. 서버에서 HTML 렌더링 → 브라우저에 전송
2. 브라우저에서 JS 로드 → 하이드레이션 (이벤트 핸들러 연결)
3. 이후 네비게이션은 클라이언트 사이드로 처리
```

SvelteKit은 기본적으로 SSR을 수행한다. JS가 비활성화되어도 서버 렌더링된 HTML이 표시된다 (단, 이벤트 핸들러와 CSR 네비게이션은 비활성화). 페이지별 SSR 비활성화는 `export const ssr = false`로 설정한다 (→ 페이지 옵션 참조).

**React/Next.js 비교**

| | Next.js (App Router) | SvelteKit |
|--|--|--|
| 기본 렌더링 | SSR (Server Components) | SSR |
| 하이드레이션 | 자동 | 자동 |
| SSR 비활성화 | `'use client'` + dynamic import | `export const ssr = false` |

---

## 10. 페이지 옵션

```ts
// +page.ts 또는 +layout.ts에서 export
export const ssr = true       // false → 서버 렌더링 비활성화 (CSR only)
export const csr = true       // false → JS 미전송 (정적 HTML만)
export const prerender = false // true → 빌드 시 정적 HTML 생성
```

| 조합 | 결과 | 용도 |
|--|--|--|
| `ssr + csr` (기본값) | SSR → 하이드레이션 | 일반적인 동적 페이지 |
| `ssr + !csr` | SSR만, JS 없음 | 정적 콘텐츠 (블로그 등) |
| `!ssr + csr` | CSR only | 관리자 대시보드, canvas 앱 |
| `prerender` | 빌드 시 HTML 생성 | 마케팅 페이지, 문서 |

레이아웃에 설정하면 하위 모든 페이지에 cascade된다.

---

## 11. 데이터 무효화 (Invalidation)

`use:enhance`는 기본적으로 `invalidateAll()`을 호출하여 모든 `load` 함수를 재실행한다. 선택적 무효화가 필요할 때는 `depends`와 `invalidate`를 조합한다.

```ts
// +page.server.ts의 load
export const load: PageServerLoad = async ({ depends }) => {
  depends('app:posts')
  const posts = await db.getPosts()
  return { posts }
}
```

```ts
// 특정 데이터만 무효화
import { invalidate } from '$app/navigation'
invalidate('app:posts')  // 'app:posts'에 의존하는 load만 재실행

// 모든 load 재실행
import { invalidateAll } from '$app/navigation'
invalidateAll()
```

- `depends(url)`: load 함수에 커스텀 의존성 등록
- `invalidate(url)`: 해당 의존성을 가진 load만 재실행
- `invalidateAll()`: 모든 load 재실행

---

## 12. 네비게이션

SvelteKit은 별도 `<Link>` 컴포넌트가 없다. 일반 `<a>` 태그를 사용하면 SvelteKit이 자동으로 클라이언트 사이드 네비게이션으로 가로챈다.

```svelte
<nav>
  <a href="/">Home</a>
  <a href="/about">About</a>
</nav>
```

### 프로그래매틱 네비게이션

```svelte
<script>
  import { goto } from '$app/navigation'
</script>

<button onclick={() => goto('/about')}>이동</button>
```

### 활성 링크 감지

```svelte
<script>
  import { page } from '$app/state'
</script>

<a href="/about" class:active={page.url.pathname === '/about'}>About</a>
```

**React 비교**

| | Next.js | SvelteKit |
|--|--|--|
| 링크 | `<Link href="/about">` | `<a href="/about">` |
| 프로그래매틱 | `useRouter().push()` | `goto()` |
