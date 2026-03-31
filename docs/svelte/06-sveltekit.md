# SvelteKit — Next.js와 비교

## 핵심 대응표

| Next.js | SvelteKit | 설명 |
| -- | -- | -- |
| `middleware.ts` | `hooks.server.ts` | 모든 요청 가로채기 |
| `layout.tsx` | `+layout.svelte` | 공유 레이아웃 |
| `page.tsx` | `+page.svelte` | 페이지 컴포넌트 |
| `getServerSideProps` | `+page.server.ts` (load) | 서버 데이터 로딩 |
| `error.tsx` | `+error.svelte` | 에러 페이지 |

---

## hooks.server.ts — Next.js의 middleware

모든 서버 요청이 여기를 먼저 통과한다. 인증 체크, `event.locals`에 데이터 주입 등.

```ts
import type { Handle } from '@sveltejs/kit'

export const handle: Handle = async ({ event, resolve }) => {
  // 인증 토큰 검사, event.locals에 유저 정보 주입 등
  return resolve(event)
}
```

### hooks.ts — 공용 에러 처리

서버 + 클라이언트 모두에서 실행.

```ts
export const handleError = ({ error }) => {
  sendToSentry(error)
  return { message: '오류가 발생했습니다' }
}
```

---

## 폴더별 공통 로직 — +layout.server.ts

Next.js의 `layout.tsx`에서 서버 로직을 분리한 형태. 폴더 기준으로 하위 페이지에 적용.

```text
src/routes/
├── +layout.server.ts           ← 전체 공통
├── admin/
│   ├── +layout.server.ts       ← /admin/* 공통 (권한 체크)
│   └── +page.svelte
└── dashboard/
    ├── +layout.server.ts       ← /dashboard/* 공통
    └── +page.svelte
```

```ts
// src/routes/admin/+layout.server.ts
export const load = async ({ locals }) => {
  if (!locals.user?.isAdmin) redirect(302, '/')
}
```

### 요청 흐름

```text
요청 → hooks.server.ts → +layout.server.ts → +page.server.ts → 응답
       (미들웨어)         (레이아웃 load)       (페이지 load)
```

| 파일 | 범위 |
| -- | -- |
| `hooks.server.ts` | 앱 전체 모든 요청 |
| `+layout.server.ts` | 해당 폴더 하위 페이지 |
| `+page.server.ts` | 해당 페이지만 |
