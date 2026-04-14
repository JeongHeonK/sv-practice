# 고급 패턴

SvelteKit fetch, REST API 엔드포인트, 페이지네이션, 제로 UI, 스트리밍, 네비게이션 훅.

---

## 1. API 엔드포인트 — `+server.ts`

> **언제 `+server.ts`를 쓰는가?**
> Form Actions가 가능하면 우선 사용. `+server.ts`는 세 가지 경우에 적합하다:
> - **외부 클라이언트** — 모바일 앱, Postman 등 SvelteKit 외부에서 접근
> - **Webhook/OAuth** — Stripe, GitHub 이벤트 등 외부 서비스 콜백
> - **비HTML 응답** — 파일 다운로드, 바이너리 데이터 등

### 기본 구조

```ts
// src/routes/api/posts/+server.ts
import { json, error } from '@sveltejs/kit'
import type { RequestHandler } from './$types'

export const GET: RequestHandler = async ({ fetch }) => {
  const response = await fetch('https://dummyjson.com/posts')
  const data = await response.json()
  return json(data)
}

export const POST: RequestHandler = async ({ request }) => {
  const post = await request.json()
  if (!post.title) error(400, '게시물 제목은 필수입니다')
  return json({ id: crypto.randomUUID(), title: post.title })
}
```

- HTTP 메서드 이름(`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)으로 함수를 export
- `json()` 헬퍼: 자동 직렬화 + Content-Type 설정
- `error()` 헬퍼: 즉시 에러 응답

### `fallback` — 나머지 메서드 일괄 처리

```ts
export const fallback: RequestHandler = async ({ request }) => {
  return text(`${request.method} 요청 수신됨`)
}
```

명시적 핸들러가 없는 메서드가 모두 이 함수로 라우팅된다.

### `+page.svelte`와 `+server.ts`가 같은 폴더에 있을 때

| 요청 조건 | 처리 파일 |
|--|--|
| `Accept: text/html` (브라우저 기본) | `+page.svelte` (HTML) |
| `Accept: application/json` 또는 미지정 | `+server.ts` (JSON 등) |
| `PUT`, `PATCH`, `DELETE` 등 | 항상 `+server.ts` |

> 실무에서는 엔드포인트를 `/api/` 하위에 분리하는 것이 명확하다.

---

## 2. SvelteKit `fetch` — event.fetch를 써야 하는 이유

load 함수와 `+server.ts`에서 제공되는 `fetch`는 일반 전역 `fetch`와 다르다:

| 기능 | event.fetch | 전역 fetch |
|--|--|--|
| 상대 경로 (`/api/posts`) | O | X (전체 URL 필요) |
| 서버→서버 직접 호출 (HTTP 오버헤드 0) | O | X |
| 쿠키/Authorization 자동 상속 | O | X |

```text
+page.server.ts → fetch('/api/posts') → +server.ts의 GET 함수 직접 호출
                                         (HTTP 요청 X, 함수 호출 O)
```

자체 엔드포인트 호출 시 실제 HTTP 요청을 보내지 않고 함수를 직접 호출한다. 쿠키도 자동 전달되므로 인증이 필요한 API 호출에서 별도 처리가 불필요하다.

---

## 3. URL 검색 파라미터로 페이지네이션

### load 함수에서 `url.searchParams` 접근

```ts
export const load: PageServerLoad = async ({ fetch, url }) => {
  const page = +(url.searchParams.get('page') ?? '1')
  const skip = (page - 1) * POSTS_PER_PAGE

  const response = await fetch(
    `https://dummyjson.com/posts?limit=${POSTS_PER_PAGE}&skip=${skip}`
  )
  if (!response.ok) error(response.status, '게시물을 불러올 수 없습니다')

  const postsResponse: PostsResponse = await response.json()
  return { title: 'Blog', posts: postsResponse }
}
```

### 핵심 동작 원리

검색 파라미터(`?page=2`)가 변경되면 SvelteKit이 **load 함수를 자동으로 다시 실행**한다. load 함수가 `url.searchParams`에 의존하므로, 파라미터 변경이 곧 데이터 재로드를 트리거한다.

```text
/blog?page=1 → load 실행 (skip=0)  → 1~10번 게시물
/blog?page=2 → load 재실행 (skip=10) → 11~20번 게시물
```

### URL 상태의 장점

검색 파라미터로 상태를 관리하면 공유/북마크/뒤로가기가 모두 작동한다. 페이지네이션, 필터링, 정렬 등은 컴포넌트 내부 상태 대신 URL을 사용하는 것이 권장된다.

> 컴포넌트에서 URL 파라미터에 의존하는 값은 반드시 `$derived`로 선언해야 페이지 이동 시 올바르게 초기화된다.

---

## 4. 제로 UI — 대화창 실행 가능 설계

**제로 UI**란 브라우저 화면 없이 Claude Code 같은 AI 대화창·CLI에서도 핵심 기능을 직접 실행할 수 있는 설계 원칙이다.

### 왜 중요한가

공간 운영 AI 제품은 브라우저를 열기 어려운 환경에서도 동작해야 한다:
- 현장 운영자가 CLI·대화창으로 빠르게 조작
- 자동화 파이프라인에서 UI 없이 API 직접 호출
- Claude Code 같은 AI 에이전트가 MCP 도구로 기능 실행

### 호출 흐름

```text
Claude Code (대화창)
    │  자연어 명령 → MCP 도구 호출 or REST 직접 호출
    ▼
+server.ts API 엔드포인트
    │  JSON 요청/응답
    ▼
DB / 실제 서비스 (온도 조절, 밝기 조절, 통계 조회 등)
```

### SvelteKit에서 적용하는 방식

- `+server.ts`로 REST 엔드포인트 노출 — 브라우저와 대화창 모두에서 호출 가능
- SvelteKit을 MCP 서버로 등록하면 Claude Code가 엔드포인트를 도구로 인식
- 응답은 JSON 우선 설계 — UI 렌더링 없이도 의미 있는 데이터 반환

Form Actions와의 관계 → 브라우저 UI가 있을 때는 Form Actions로 뮤테이션, 대화창 실행 시에는 `+server.ts` REST 엔드포인트를 직접 호출한다.

---

## 5. 병렬 실행 & 스트리밍

### `Promise.all`로 병렬 실행

독립적인 fetch 요청은 `Promise.all`로 동시 실행한다:

```text
❌ 순차 (waterfall):
  fetchPost start ─────── end
                           fetchComments start ─── end
  총 시간: A + B

✅ 병렬 (Promise.all):
  fetchPost start ─────── end
  fetchComments start ─── end
  총 시간: max(A, B)
```

```ts
export const load: PageLoad = async ({ fetch, params }) => {
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
| 게시물 | `error()` → 에러 페이지 | 핵심 데이터 — 없으면 페이지 자체가 의미 없음 |
| 댓글 | 빈 배열 반환 | 부가 데이터 — 없어도 게시물은 표시 가능 |

### Universal load에서 네트워크 동작

```text
+page.ts (universal) + CSR 네비게이션:
  브라우저 → 외부 API (직접 호출, 서버 미경유)

+page.server.ts (server) + CSR 네비게이션:
  브라우저 → SvelteKit 서버 → 외부 API (서버 경유)
```

공개 API라면 `+page.ts`가 더 효율적. 비밀 자격 증명이 필요하면 `+page.server.ts` 필수.

### 스트리밍 — `await` 없이 Promise를 반환

핵심 데이터만 기다리고, 부가 데이터는 **Promise 그대로 반환**하면 SvelteKit이 서버에서 클라이언트로 스트리밍한다.

```ts
// +page.server.ts — 스트리밍은 server load에서만 가능
export const load: PageServerLoad = async ({ fetch, params }) => {
  const commentsPromise = fetchComments(fetch, params.id)  // await 없이 시작
  const post = await fetchPost(fetch, params.id)            // 핵심만 await

  return {
    post,
    comments: commentsPromise  // ← Promise 그대로 = 스트리밍
  }
}
```

**순서가 중요**: 스트리밍할 Promise를 먼저 생성 → 핵심 데이터 await → return. 이렇게 하면 두 fetch가 **동시에 시작**되면서, 핵심 완료 즉시 페이지가 렌더링된다.

컴포넌트에서는 `{#await}` 블록으로 처리:

```svelte
{#await data.comments}
  <div class="skeleton h-16 w-full"></div>
{:then comments}
  {#each comments as comment}
    <div>{comment.body}</div>
  {/each}
{/await}
```

### 스트리밍 동작 시각화

```text
[Promise.all — 모두 기다림]
  fetchPost     ─────────── end ─┐
  fetchComments ───── end ────────┤ → 모두 완료 후 페이지 렌더링
                                   총 시간: max(A, B)

[스트리밍 — 핵심만 기다림]
  fetchPost     ─────────── end → 페이지 렌더링 → 게시물 표시
  fetchComments ───── end ───────────────────────→ 댓글 도착 → 스켈레톤 교체
  첫 화면: fetchPost 완료 시점
```

### 스트리밍 주의사항

| 항목 | 설명 |
|--|--|
| **server load 전용** | `+page.server.ts`에서만 동작 |
| **JS 비활성화** | 스트리밍 데이터 렌더링 불가 → `no-js:hidden`으로 숨기기 권장 |
| **SEO** | 초기 HTML 미포함 → 핵심 SEO 데이터는 반드시 `await`할 것 |

---

## 6. 네비게이션 진행률 표시 — `beforeNavigate` / `afterNavigate`

`$app/navigation`에서 네비게이션 이벤트에 훅을 걸 수 있다:

```ts
import { beforeNavigate, afterNavigate } from '$app/navigation'

beforeNavigate((navigation) => {
  // navigation.from, navigation.to, navigation.type
  // navigation.cancel() — 네비게이션 취소
})

afterNavigate((navigation) => {
  // 네비게이션 완료 후 실행
})
```

### nprogress 연동 개념

`beforeNavigate`에서 진행률 바 시작, `afterNavigate`에서 완료. 타임아웃(0.5초)을 걸어 빠른 네비게이션에서 깜빡임을 방지한다.

```text
네비게이션 시작 → 0.5초 타이머 시작
  ├─ 0.5초 이내 완료 → clearTimeout → 진행률 바 표시 안 함
  └─ 0.5초 초과 → NProgress.start() → 표시 → 완료 시 NProgress.done()
```

> `navigating` 스토어(`$app/stores`)로도 네비게이션 상태를 감지할 수 있다. `navigating`이 `null`이면 유휴, 객체면 네비게이션 중.

---

## 7. 코드 재실행 제어

> `depends()` + `invalidate()`로 load 재실행 범위를 최소화한다.
> Form Actions에서의 실전 패턴 포함 → [07-load-invalidation.md](./07-load-invalidation.md) 참조

---

## React/Next.js 비교

| 개념 | Next.js (App Router) | SvelteKit |
|--|--|--|
| API 엔드포인트 | `app/api/*/route.ts` | `+server.ts` |
| JSON 헬퍼 | `NextResponse.json()` | `json()` from `@sveltejs/kit` |
| 서버 fetch 최적화 | 없음 (항상 HTTP) | 자체 엔드포인트 직접 함수 호출 |
| 쿠키 상속 | 수동 전달 | event.fetch가 자동 상속 |
| 네비게이션 이벤트 | `usePathname` + `useEffect` | `beforeNavigate` / `afterNavigate` |
| 스트리밍 | `<Suspense>` + async Server Component | `await` 없이 Promise 반환 + `{#await}` |
| 제로 UI (API-first) | 별도 구현 필요 | `+server.ts` + MCP 서버 등록 |
| 선택적 무효화 | `revalidateTag()` / `revalidatePath()` | `invalidate('app:tag')` |
| 전체 무효화 | `router.refresh()` | `invalidateAll()` |
