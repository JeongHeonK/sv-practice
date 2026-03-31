# SvelteKit — Next.js와 비교

## 핵심 대응표

| Next.js | SvelteKit | 설명 |
| -- | -- | -- |
| `middleware.ts` | `hooks.server.ts` | 모든 요청 가로채기 |
| `layout.tsx` | `+layout.svelte` | 공유 레이아웃 |
| `page.tsx` | `+page.svelte` | 페이지 컴포넌트 |
| `getServerSideProps` | `+page.server.ts` (load) | 서버 데이터 로딩 |
| `error.tsx` | `+error.svelte` | 에러 페이지 |
| `route.ts` (API) | `+server.ts` | API endpoint |

---

## 파일 컨벤션 — 실행 환경 맵

> Next.js는 `"use client"` 지시어로 구분하지만, SvelteKit은 **파일 이름**으로 결정된다.

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

## 데이터 플로우 시각화

```text
Request
  → hooks.server.ts        event.locals에 유저 주입 (= Next.js middleware)
  → +layout.server.ts      return { user }   → 하위 모든 페이지에 전파
  → +page.server.ts        return { posts }  → 해당 페이지만
  → +page.svelte           data prop = { user, posts } (병합 수신)
  → Response (SSR HTML 또는 CSR JSON)
```

### 데이터 수신 — `let { data } = $props()`

```svelte
<!-- +page.svelte — Next.js의 getServerSideProps → props와 동일한 멘탈 모델 -->
<script>
  let { data } = $props();  // { user, posts } ← layout + page load 자동 병합
</script>
<h1>{data.user.name}의 게시글</h1>
{#each data.posts as post}
  <article>{post.title}</article>
{/each}
```

키 충돌 시 page load가 우선.

---

## hooks.server.ts — `event.locals`로 데이터 전달

```ts
export const handle: Handle = async ({ event, resolve }) => {
  event.locals.user = await getSession(event.cookies.get('token'));
  return resolve(event);  // 이후 모든 load()에서 event.locals.user 접근 가능
};
```

---

## load 함수 — server vs universal

| | `+page.server.ts` (ServerLoad) | `+page.ts` (PageLoad) |
| -- | -- | -- |
| 실행 환경 | 서버만 | 서버 + 브라우저 |
| DB/private env | O | X (fetch만 가능) |
| 반환값 | 직렬화 필수 | 클래스 등 자유로움 |
| form actions | O | X |
| CSR navigation | 서버에 JSON 요청 | 브라우저에서 직접 실행 |

---

## 폴더별 스코프

```text
src/routes/
├── +layout.server.ts            ← 전체 공통 (루트 레이아웃)
├── admin/
│   ├── +layout.server.ts        ← /admin/* 공통 (권한 체크)
│   └── +page.server.ts          ← /admin 페이지만
└── dashboard/
    └── +page.server.ts          ← /dashboard 페이지만
```

```ts
// src/routes/admin/+layout.server.ts — 하위 모든 페이지 진입 전 실행
export const load = async ({ locals }) => {
  if (!locals.user?.isAdmin) redirect(302, '/');
  return { role: 'admin' };
};
```

---

## handleError — 로깅과 사용자 메시지 분리

서버/클라이언트 각각 정의. Next.js `error.tsx`와 달리 **로깅 ↔ 노출 메시지를 명시적으로 분리**.

```ts
// src/hooks.server.ts (또는 hooks.client.ts)
export const handleError = ({ error }) => {
  console.error(error);                        // 내부 로깅
  return { message: '오류가 발생했습니다' };     // 사용자 노출 (page.error)
};
```
