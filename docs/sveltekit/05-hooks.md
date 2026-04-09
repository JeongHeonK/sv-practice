# SvelteKit Hooks

## 1. Hooks 개요

Hooks는 SvelteKit에서 발생하는 **특정 이벤트에 연결(hook into)되는 함수**를 정의하여 프레임워크의 동작을 수정할 수 있는 메커니즘이다.

### Hooks의 3가지 종류

| 파일 | 실행 환경 | 설명 |
|---|---|---|
| `src/hooks.server.ts` | 서버 only | 서버에서 발생하는 이벤트에 의해 트리거 |
| `src/hooks.client.ts` | 클라이언트 only | 클라이언트에서 발생하는 이벤트에 의해 트리거 |
| `src/hooks.ts` | 서버 + 클라이언트 | 양쪽 모두에서 실행되는 범용(universal) 훅 |

```
src/
├── hooks.server.ts    ← 서버 훅
├── hooks.client.ts    ← 클라이언트 훅
├── hooks.ts           ← 범용 훅
├── routes/
└── ...
```

> **참고**: 훅 파일의 경로는 `svelte.config.js`의 `kit.files.hooks` 옵션으로 커스터마이징할 수 있다. 기본값은 `src/hooks.*`이다.

---

## 2. handle 훅 — 요청 처리의 핵심

서버가 요청을 수신할 때마다 실행되며, 요청 처리 파이프라인 전체를 커스터마이징할 수 있는 **가장 핵심적인 훅**이다.

### 요청 처리 흐름

```
요청 → handle({ event, resolve })
         ① event 수정 (locals 주입, 쿠키/헤더 확인)
         ② resolve(event) → load 실행 → 렌더링 (또는 +server.ts 실행)
         ③ 응답 수정 (헤더 추가 등)
         ④ 응답 반환 → Client
```

### 기본 동작

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event);
  return response;
};
```

- **`event`**: 요청 정보 객체 (cookies, url, params, request, locals 등)
- **`resolve`**: SvelteKit의 기본 처리 실행 (load → 렌더링 또는 endpoint 호출)
- **반환값**: `Response` 객체

### handle의 3가지 패턴

#### 패턴 1: resolve 전에 event 수정

```ts
export const handle: Handle = async ({ event, resolve }) => {
  // resolve 호출 전 — event를 수정하여 load 함수에 데이터 전달
  const token = event.cookies.get('token');
  event.locals.user = token ? await getUser(token) : null;

  const response = await resolve(event);
  return response;
};
```

> `event.locals`에 주입한 데이터는 이후 모든 load 함수에서 접근 가능하다.

#### 패턴 2: resolve 후에 응답 수정

```ts
export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event);

  // resolve 호출 후 — 응답 헤더 추가
  response.headers.set('X-Custom-Header', 'my-value');

  return response;
};
```

#### 패턴 3: resolve를 우회하여 직접 응답 반환

```ts
export const handle: Handle = async ({ event, resolve }) => {
  // 특정 경로에 대해 resolve를 건너뛰고 직접 응답
  if (event.url.pathname === '/health') {
    return new Response('OK', { status: 200 });
  }

  return await resolve(event);
};
```

> resolve를 호출하지 않으면 load 함수, 페이지 렌더링 등 SvelteKit의 기본 처리가 **전혀 실행되지 않는다**.

### `event.isDataRequest` — 요청 유형 구분

| 상황 | `isDataRequest` | 설명 |
|---|---|---|
| 브라우저에서 직접 URL 접속 (SSR) | `false` | 서버에서 페이지 전체를 렌더링 |
| 클라이언트 측 내비게이션 | `true` | `__data.json` fetch 요청으로 데이터만 요청 |
| Postman 등 외부 API 호출 | `false` | +server.ts 엔드포인트 직접 호출 |

### 여러 handle 함수 체이닝 — sequence

`sequence`를 사용해 역할별로 handle을 분리하고 순서대로 실행할 수 있다:

```ts
import { sequence } from '@sveltejs/kit/hooks';
// 순서대로 실행: authHandle → loggingHandle → resolve
export const handle = sequence(authHandle, loggingHandle);
```

---

## 3. event.locals — 요청 단위 데이터 전달

`event.locals`는 **하나의 요청 동안만 살아있는 객체**로, handle에서 주입하면 이후 모든 load 함수에서 접근할 수 있다.

```
요청 → handle (locals.user 주입) → resolve → load 함수들 (locals.user 접근) → 응답 → locals 소멸
```

### locals vs 쿠키

| 특성 | `event.locals` | `event.cookies` |
|---|---|---|
| 생존 범위 | 요청 1회 한정 | 브라우저에 지속 |
| 저장 위치 | 서버 메모리 (요청 중) | 클라이언트 쿠키 |
| 용도 | 요청 처리 중 임시 데이터 전달 | 인증 토큰 등 영구 데이터 |
| 보안 | 서버에만 존재, 외부 노출 없음 | 클라이언트에서 접근 가능 |

> **핵심**: "쿠키에서 토큰 확인 → 사용자 정보 조회 → locals에 주입"하면, 모든 load 함수에서 매번 쿠키를 파싱하지 않아도 된다.

### 올바른 패턴: handle 훅에서 인증 체크

```ts
// src/hooks.server.ts — ✅ 안전한 패턴
import { redirect } from '@sveltejs/kit';
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  // 1. 쿠키에서 토큰 확인
  const token = event.cookies.get('token');

  // 2. 사용자 정보를 locals에 주입 (요청 단위)
  event.locals.user = token ? { name: 'John', id: 1 } : null;

  // 3. 보호된 경로 접근 시 인증 체크
  if (!event.locals.user && event.url.pathname.startsWith('/app')) {
    redirect(303, '/login');
  }

  return await resolve(event);
};
```

> handle은 **모든 서버 요청**에서 실행되므로(SSR + 클라이언트 내비게이션의 `__data.json` 요청 포함) 인증 우회가 불가능하다.

### locals의 TypeScript 타입 정의

`src/app.d.ts`에서 locals 타입을 정의해야 TypeScript 오류 없이 사용할 수 있다:

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Locals {
      user: { name: string; id: number } | null;
    }
  }
}

export {};
```

### 경로 보호: 2가지 전략

**전략 A: handle에서 중앙 집중 체크** — 위의 "올바른 패턴" 코드처럼 handle에서 보호 경로를 일괄 체크한다. 한 곳에서 관리하므로 빠뜨릴 위험이 없지만, 복잡한 앱에서 조건이 많아질 수 있다.

**전략 B: 개별 페이지 load에서 분산 체크** — handle에서는 locals만 주입하고, 각 페이지 load에서 인증을 체크한다:

```ts
// src/routes/(app)/+page.server.ts — 각 페이지에서 체크
export const load: PageServerLoad = async ({ locals }) => {
  if (!locals.user) {
    redirect(303, '/login');
  }

  return { dashboard: await getDashboard(locals.user.id) };
};
```

> 장점: 페이지별 세밀한 제어 가능
> 단점: 보호할 페이지마다 체크를 추가해야 함 (빠뜨리면 보안 구멍)

---

## 4. +error.svelte 페이지

> 에러 페이지 기본 구조(+error.svelte, 오류 탐색 순서, 커스텀 에러 데이터)는 [02-core.md Section 7](02-core.md#7-에러-처리)을 참고한다.

### 레이아웃 오류의 특수 동작

**레이아웃 load 함수에서 오류가 발생하면**, 같은 레벨의 `+error.svelte`가 아닌 **상위 레벨의 `+error.svelte`**가 사용된다:

```
(marketing)/+layout.server.ts 에서 오류 발생
  → (marketing)/+error.svelte    ❌ 사용되지 않음 (같은 레벨)
  → routes/+error.svelte         ✅ 상위 레벨 사용
```

> 이유: 레이아웃에 오류가 있으면 같은 레벨의 오류 페이지도 그 레이아웃에 의존하므로 렌더링할 수 없다.

---

## 5. handleError 훅 — 예상치 못한 오류 처리

`handleError`는 **예상치 못한 오류**(코드 버그 등)가 발생할 때 실행되는 **공유 훅**이다. `error()` 함수로 직접 던진 예상된 오류에는 실행되지 않는다. `hooks.server.ts`와 `hooks.client.ts`에 각각 별도로 정의해야 한다.

### 서버 handleError

```ts
// src/hooks.server.ts
import type { HandleServerError } from '@sveltejs/kit';

export const handleError: HandleServerError = async ({ error, event, status, message }) => {
  // error: 실제 발생한 오류 객체 (민감한 정보 포함 가능)
  // event: 요청 이벤트 (cookies, locals, url 등)
  // status: HTTP 상태 코드 (보통 500)
  // message: 기본 메시지 ("Internal Error")

  // 1. 오류 로깅
  console.error('Unexpected server error:', error);
  console.error('URL:', event.url.pathname);
  console.error('User:', event.locals.user);

  // 2. Sentry 등 오류 보고 서비스에 전송
  // Sentry.captureException(error, {
  //   extra: { url: event.url.pathname, status },
  //   user: event.locals.user ? { id: event.locals.user.id } : undefined
  // });

  // 3. 사용자에게 보여줄 메시지 커스터마이징 (반환값)
  return {
    message: '예기치 않은 오류가 발생했습니다',
    code: 'UNEXPECTED_ERROR'
  };
};
```

#### 핵심 동작

- `handleError`를 정의하지 않으면 → 기본적으로 터미널에 오류 로그 출력
- `handleError`를 정의하면 → **기본 로깅이 사라짐**, 직접 `console.error` 해야 함
- **반환값**: `+error.svelte`에서 `page.error`로 접근할 수 있는 객체 (반환하지 않으면 기본 "Internal Error" 메시지)
- **예상된 오류(`error()` 함수)에는 실행되지 않음** — 예상된 오류의 메시지는 이미 사용자에게 보여줄 의도로 작성되었으므로

### 클라이언트 handleError

```ts
// src/hooks.client.ts
import type { HandleClientError } from '@sveltejs/kit';

export const handleError: HandleClientError = async ({ error, status, message }) => {
  // ⚠️ 서버와 다른 점: event가 NavigationEvent (params, route, url만 있음)
  // cookies, locals, fetch 등 서버 전용 속성 없음

  console.error('Unexpected client error:', error);

  // Sentry 등에 전송
  // Sentry.captureException(error);

  return {
    message: '예기치 않은 클라이언트 오류가 발생했습니다',
    code: 'UNEXPECTED_CLIENT_ERROR'
  };
};
```

### 서버 오류 vs 클라이언트 오류 발생 시점

- **SSR 오류** (+page.server.ts 등) → `hooks.server.ts`의 handleError → 터미널 로그
- **클라이언트 오류** (+page.ts 등, 브라우저 실행) → `hooks.client.ts`의 handleError → 브라우저 콘솔 로그

### init 훅

`init`도 공유 훅으로, 서버 시작 시(`hooks.server.ts`) 또는 앱 첫 로드 시(`hooks.client.ts`) **1번만** 실행된다. DB 연결, 애널리틱스 초기화 등에 사용한다.

### 예상된 오류 vs 예상치 못한 오류 — 전체 비교

| 특성 | 예상된 오류 | 예상치 못한 오류 |
|---|---|---|
| 발생 방식 | `error(404, 'Not found')` | 코드 버그, 런타임 에러 |
| `handleError` 실행 | X | O |
| 사용자에게 보이는 메시지 | `error()`에 전달한 메시지 그대로 | 기본 "Internal Error" 또는 `handleError` 반환값 |
| 실제 오류 정보 | 사용자에게 노출 가능 | **민감 정보** → 서버 로그에만, 사용자에게 노출 금지 |
| HTTP 상태 | 개발자가 지정 | 500 |

---

> resolve 옵션(transformPageChunk, preload, filterSerializedResponseHeaders), handleFetch, reroute, transport 등 고급 훅은 [06-advanced-hooks.md](06-advanced-hooks.md)를 참고한다.
