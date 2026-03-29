# SvelteKit

## hooks 파일

파일명은 반드시 `hooks`여야 한다. SvelteKit이 약속된 이름만 인식한다.

### hooks.server.ts

모든 서버 요청이 여기를 먼저 통과한다.

```ts
import type { Handle } from '@sveltejs/kit'

export const handle: Handle = async ({ event, resolve }) => {
  // 인증 토큰 검사
  // event.locals에 데이터 주입
  // 특정 경로 접근 차단
  return resolve(event)
}
```

### hooks.ts

주로 클라이언트 에러 처리.

```ts
export const handleError = ({ error }) => {
  sendToSentry(error)
  return { message: '오류가 발생했습니다' }
}
```

### 비교

| | `hooks.ts` | `hooks.server.ts` |
| -- | -- | -- |
| 실행 환경 | 서버 + 클라이언트 | 서버만 |
| 주 용도 | 에러 처리 | 요청 가로채기, 인증 |

---

## 컴포넌트 script vs hooks

| | hooks | 컴포넌트 script |
| -- | -- | -- |
| 적용 범위 | 앱 전체 | 해당 컴포넌트만 |
| 실행 시점 | 요청/에러 발생 시 | 컴포넌트 마운트 시 |
| 인스턴스 | 1개 | 컴포넌트 수만큼 |

모든 페이지에 반복 작성하기 싫은 공통 로직 → hooks

---

## 폴더별 공통 로직 — +layout.server.ts

hooks는 `src/` 루트에만 위치 가능. 폴더별 공통 로직은 `+layout.server.ts`로.

```text
src/routes/
├── +layout.server.ts        ← 전체 공통 로직
├── admin/
│   ├── +layout.server.ts    ← /admin/* 공통 로직 (관리자 권한 체크 등)
│   └── +page.svelte
└── dashboard/
    ├── +layout.server.ts    ← /dashboard/* 공통 로직
    └── +page.svelte
```

```ts
// src/routes/admin/+layout.server.ts
export const load = async ({ locals }) => {
  if (!locals.user?.isAdmin) {
    redirect(302, '/')
  }
}
```

### 요청 흐름

```text
브라우저 요청
    ↓
hooks.server.ts (handle)   ← 모든 요청 통과
    ↓
+layout.server.ts (load)
    ↓
+page.server.ts (load)
    ↓
응답
```

| 파일 | 범위 |
| -- | -- |
| `src/hooks.server.ts` | 앱 전체 모든 요청 |
| `src/routes/+layout.server.ts` | 전체 페이지 load 시 |
| `src/routes/admin/+layout.server.ts` | /admin/* 페이지만 |
