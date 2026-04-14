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
import type { PageServerLoad } from './$types'
import * as db from '$lib/server/db'

export const load: PageServerLoad = async () => {
  return { posts: await db.getPosts() }
}
```

- **항상 서버에서만 실행** — 클라이언트에 코드가 노출되지 않음
- DB 쿼리, private API, 비밀 자격 증명 안전하게 사용 가능
- CSR 네비게이션 시 SvelteKit이 서버에 `__data.json` 요청을 보내 데이터를 가져옴

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
├── +layout.server.ts          ← ① 루트 레이아웃 load (앱 전체)
├── (marketing)/
│   ├── +layout.server.ts      ← ② 마케팅 레이아웃 load (그룹 내)
│   └── about/
│       ├── +page.server.ts    ← ③ about 페이지 load (이 페이지만)
│       └── +page.svelte       ← ①, ②, ③ 데이터 모두 접근 가능
```

### 데이터 스코프 규칙

| load 파일 위치 | 데이터 사용 범위 |
|--|--|
| `routes/+layout.server.ts` | 앱 전체 (모든 페이지) |
| `(marketing)/+layout.server.ts` | 마케팅 그룹 내 모든 페이지 |
| `about/+page.server.ts` | about 페이지만 |

### 키 충돌 시 우선순위

layout load와 page load가 **같은 키를 반환**하면 page load의 값이 우선한다.

```text
layout.server.ts  →  { user: {...}, theme: 'dark' }
page.server.ts    →  { posts: [...], theme: 'light' }
                       ───────────────────────────────
컴포넌트 data      =  { user: {...}, posts: [...], theme: 'light' }
                                                   ↑ page 값이 우선
```

일반적으로 layout과 page는 서로 다른 키를 담당한다 — layout은 `user`, `config` 등 공통 데이터, page는 `posts`, `product` 등 페이지별 데이터.

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

- 페이지 load에서 부모 레이아웃 load의 반환값에 **기본적으로 접근할 수 없다**
- 부모 데이터가 필요하면 `await parent()`로 명시적으로 대기해야 한다

요청된 경로가 사용하는 레이아웃 체인 + 해당 페이지의 load만 실행된다. 다른 경로의 load는 실행되지 않는다.

---

## 4. `$types` — 자동 생성 타입 시스템

`'./$types'`에서 import하는 타입들은 `.svelte-kit/types/` 폴더에 자동 생성된다. 개발 서버 실행 시 `src/routes/` 구조를 미러링하여 생성.

### 생성되는 타입들

| 타입 | 용도 |
|--|--|
| `PageLoad` / `PageServerLoad` | load 함수 시그니처 |
| `PageProps` / `LayoutProps` | 컴포넌트 $props() 타입 |
| `PageData` | load 반환값 기반 자동 추론 |

### 동적 라우트의 params 타입

```ts
// src/routes/blog/[id]/+page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async ({ params }) => {
  console.log(params.id) // ✅ 자동 완성 — id: string
  // params.slug         // ❌ 타입 에러 — 존재하지 않는 파라미터
  return { postId: params.id }
}
```

`[id]` 폴더명을 기반으로 `params`의 타입이 `{ id: string }`으로 자동 생성된다.

> 타입을 명시하면 에디터에 의존하지 않고 어디서든 타입 안전성이 보장되므로 **명시하는 것을 권장**.

---

## 5. 두 파일 공존 — `+page.ts` + `+page.server.ts`

두 파일이 동시에 존재하면, **컴포넌트가 받는 `data`는 `+page.ts`(universal)의 반환값**이다.

```text
데이터 흐름:
  +page.server.ts  →  { title, count }
        │
        ▼ (universal load의 data 파라미터로 전달)
  +page.ts         →  { ...data, component: module.default }
        │
        ▼
  +page.svelte     →  data = { title, count, component }
```

```ts
// +page.ts — server load 데이터를 받아 확장
export const load: PageLoad = async ({ data }) => {
  // data = server load의 반환값
  return { ...data, extraField: 'value' }
}
```

> 실무에서 두 파일을 동시에 쓰는 경우: 서버에서 DB 데이터를 가져오되, 클라이언트에서 직렬화 불가능한 값(Svelte 컴포넌트, 클래스 인스턴스 등)을 추가해야 할 때. 대부분의 경우 `+page.server.ts` 하나로 충분하다.

---

## 6. `await parent()` — 부모 레이아웃 데이터 접근

load 함수는 **병렬 실행**이므로 부모 load가 아직 완료되지 않았을 수 있다. 부모 데이터가 필요하면 명시적으로 대기해야 한다.

```ts
export const load: PageLoad = async ({ parent }) => {
  const parentData = await parent()
  // parentData = 모든 상위 레이아웃 load의 병합 결과
  return { title: 'Blog' }
}
```

### 주의: waterfall을 최소화하라

`await parent()`는 부모 load가 끝날 때까지 **현재 함수 실행을 차단**한다.

```ts
// ✅ 부모에 의존하지 않는 작업을 먼저 수행
export const load: PageLoad = async ({ parent, fetch }) => {
  const posts = await fetch('/api/posts').then(r => r.json())  // 독립 작업 먼저
  const parentData = await parent()                             // 필요한 시점에 대기
  return { posts: posts.filter(p => p.userId === parentData.user?.id) }
}

// ❌ 안티패턴: 부모를 먼저 기다린 후 독립적인 작업 수행
export const load: PageLoad = async ({ parent, fetch }) => {
  const parentData = await parent()  // 불필요하게 차단
  const posts = await fetch('/api/posts').then(r => r.json())  // 부모와 무관
  return { posts }
}
```

> **원칙**: 부모 데이터에 의존하지 않는 작업 → 먼저 실행. `await parent()` → 실제로 필요한 시점에만 호출.

---

## 7. `page.data` — 레이아웃에서 페이지 데이터 접근 (SEO 활용)

데이터 흐름은 부모 → 자식 방향이지만, `$app/state`의 `page` 객체에는 현재 경로의 **모든 load 데이터가 병합**되어 있다. 레이아웃에서 `page.data`로 자식 페이지의 데이터에 접근 가능.

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { page } from '$app/state'
  let { children }: LayoutProps = $props()
</script>

<svelte:head>
  <title>{page.data.title ? `${page.data.title} | My App` : 'My App'}</title>
  {#if page.data.description}
    <meta name="description" content={page.data.description} />
  {/if}
</svelte:head>

{@render children()}
```

루트 레이아웃에서 `page.data` 기반 폴백을 두면 SEO 메타 태그 누락을 방지할 수 있다. 각 페이지에서 직접 `<svelte:head>`를 사용할 수도 있다.

---

## 8. Form Actions — 서버 뮤테이션

`+page.server.ts`에서 `actions` 객체를 export하면 HTML `<form>`으로 서버 뮤테이션을 처리할 수 있다.

### 기본 구조

named actions 패턴으로 여러 액션을 한 파일에 정의한다:

```text
actions 객체 (named actions):
  ?/create  → createMessage 처리 함수
  ?/delete  → deleteMessage 처리 함수

컴포넌트:
  <form method="POST" action="?/create">...</form>
  <form method="POST" action="?/delete">...</form>
```

### 반환값 — ActionData

action 함수가 반환한 값은 컴포넌트에서 `form` prop으로 수신한다.

```text
ActionData 흐름:
  action 함수 return { success, error }
       ↓
  컴포넌트 $props()의 form: ActionData
       ↓
  {#if form?.error} 에러 메시지 표시
  {#if form?.success} 성공 처리
```

### `use:enhance` — 점진적 향상

`use:enhance`를 붙이면 브라우저 기본 form submit을 인터셉트해 fetch로 처리한다.
붙이지 않아도 동작하지만 전체 페이지 새로고침이 발생한다.

```text
use:enhance 없음:
  form POST → 서버 액션 실행 → 전체 페이지 새로고침

use:enhance 있음:
  form submit → fetch POST → 서버 액션 실행 → 부분 업데이트
```

> load 재실행·무효화 상세 패턴 → [07-load-invalidation.md](./07-load-invalidation.md)

---

## React/Next.js 비교

| 개념 | Next.js (App Router) | SvelteKit |
|--|--|--|
| 서버 데이터 로딩 | Server Component에서 직접 fetch | `+page.server.ts` load 함수 |
| 클라이언트 데이터 | `'use client'` + `useEffect`/SWR | `+page.ts` universal load |
| 레이아웃 데이터 공유 | props drilling 또는 Context | 자동 병합 (layout → page data) |
| 부모 데이터 접근 | Context/props | `await parent()` |
| 병렬 실행 | `Promise.all` 수동 | 자동 (load 함수 병렬) |
| 타입 안전성 | 수동 정의 | `$types` 자동 생성 |
| SEO 메타 | `metadata` export / `generateMetadata` | `<svelte:head>` + `page.data` |
| 데이터 직렬화 | RSC가 자동 처리 | server load는 JSON 직렬화 필수 |

> 더 많은 패턴 → [05-advanced-patterns.md](./05-advanced-patterns.md)
