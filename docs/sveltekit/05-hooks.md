# SvelteKit Hooks

## 1. Hooks 개요

Hooks는 SvelteKit에서 발생하는 **특정 이벤트에 연결(hook into)되는 함수**를 정의하여 프레임워크의 동작을 수정할 수 있는 메커니즘이다.

### Hooks로 할 수 있는 것

- 서버가 요청을 받을 때마다 함수 실행
- **요청(request)을 수정**하거나 **응답(response)을 수정**
- load 함수에서 사용하는 `fetch` 함수의 동작을 수정
- 에러 처리 커스터마이징 등

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

### React/Next.js와 비교

| 개념 | SvelteKit | Next.js |
|---|---|---|
| 서버 요청 가로채기 | `hooks.server.ts` → `handle` | `middleware.ts` |
| 클라이언트 훅 | `hooks.client.ts` | 없음 (개별 컴포넌트에서 처리) |
| 범용 훅 | `hooks.ts` | 없음 |
| 파일 위치 | `src/hooks.*.ts` | 프로젝트 루트 `middleware.ts` |

Next.js의 `middleware.ts`는 Edge Runtime에서 실행되며 요청을 가로채는 역할만 하지만, SvelteKit의 `hooks.server.ts`는 Node.js 환경에서 실행되며 요청 수정, 응답 수정, locals 주입 등 더 다양한 역할을 수행한다.

---

## 2. handle 훅 — 요청 처리의 핵심

`handle`은 SvelteKit 앱에서 **가장 많이 사용하는 훅**이다. 서버가 요청을 수신할 때마다 실행되며, 요청 처리 파이프라인 전체를 커스터마이징할 수 있다.

### 요청 처리 흐름

```
Client (브라우저 / Postman 등)
  │
  │  요청 (Request)
  ▼
┌─────────────────────────────────────────┐
│  handle({ event, resolve })             │
│                                         │
│  ① event 수정 (요청 가로채기)            │
│     - event.locals에 데이터 주입         │
│     - 쿠키/헤더 확인                     │
│                                         │
│  ② resolve(event) 호출                  │
│     - 페이지 요청 → load 함수 실행 → 렌더링│
│     - API 요청 → +server.ts 함수 실행    │
│                                         │
│  ③ 응답 수정 (response 가로채기)          │
│     - 응답 헤더 추가/수정                 │
│     - 응답 본문 변환                     │
│                                         │
│  ④ 응답 반환 → Client                   │
└─────────────────────────────────────────┘
```

### 기본 동작

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';

// 아무 커스터마이징 없이 그대로 통과시키는 기본 형태
export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event);
  return response;
};
```

- **`event`**: 요청 정보가 담긴 객체 (cookies, url, params, request, locals 등)
- **`resolve`**: SvelteKit의 기본 처리를 실행하는 함수 (load → 렌더링 또는 endpoint 호출)
- **반환값**: 클라이언트에게 보낼 `Response` 객체

### handle로 할 수 있는 3가지 패턴

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

### 여러 handle 함수 체이닝

하나의 거대한 `handle` 함수 대신, **`sequence`**를 사용해 역할별로 분리할 수 있다:

```ts
import { sequence } from '@sveltejs/kit/hooks';

const authHandle: Handle = async ({ event, resolve }) => {
  // 인증 처리
  return resolve(event);
};

const loggingHandle: Handle = async ({ event, resolve }) => {
  // 요청 로깅
  return resolve(event);
};

// 순서대로 실행: auth → logging → resolve
export const handle = sequence(authHandle, loggingHandle);
```

### Next.js middleware와의 차이

| 특성 | SvelteKit `handle` | Next.js `middleware` |
|---|---|---|
| 실행 환경 | Node.js (서버) | Edge Runtime |
| resolve 우회 | 가능 (직접 Response 반환) | 제한적 (NextResponse.next() 필수) |
| 응답 수정 | resolve 후 Response 직접 수정 가능 | 헤더/쿠키 수정만 가능 |
| 체이닝 | `sequence()` 헬퍼 제공 | 불가 (하나의 middleware만) |
| load 함수 데이터 전달 | `event.locals` | 없음 (헤더로 우회) |

### handle 실습 — 실제 동작 확인

#### 기본 로깅 예제

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  console.log('handle hook:', event.url.pathname);
  console.log('isDataRequest:', event.isDataRequest);

  return await resolve(event);
};
```

#### `event.isDataRequest` — 요청 유형 구분

모든 서버 요청이 handle을 트리거하지만, **요청의 성격**이 다를 수 있다:

| 상황 | `isDataRequest` | 설명 |
|---|---|---|
| 브라우저에서 직접 URL 접속 (SSR) | `false` | 서버에서 페이지 전체를 렌더링 |
| 클라이언트 측 내비게이션 | `true` | `__data.json` fetch 요청으로 데이터만 요청 |
| Postman 등 외부 API 호출 | `false` | +server.ts 엔드포인트 직접 호출 |

```
// SSR (직접 접속) → isDataRequest: false
브라우저 → /blog → handle 실행 → 전체 HTML 렌더링

// 클라이언트 내비게이션 → isDataRequest: true
SPA 내비게이션 → /blog/__data.json → handle 실행 → JSON 데이터만 반환
```

#### resolve 전에 event 수정 — 쿠키 설정 예제

handle에서 설정한 쿠키는 이후 모든 load 함수에서 접근 가능하다:

```ts
// src/hooks.server.ts
export const handle: Handle = async ({ event, resolve }) => {
  // resolve 전에 쿠키 설정
  event.cookies.set('test-cookie', 'hello', { path: '/' });

  return await resolve(event);
};
```

```ts
// src/routes/blog/+page.server.ts
export const load: PageServerLoad = async ({ cookies }) => {
  // handle에서 설정한 쿠키를 load 함수에서 읽을 수 있다
  const testCookie = cookies.getAll();
  console.log('cookies:', testCookie); // test-cookie: 'hello' 포함
  // ...
};
```

> **핵심**: handle에서 `event`를 수정하면 → `resolve(event)` → load 함수에 수정된 event가 전달된다.

#### resolve 후에 응답 수정 — 커스텀 헤더

```ts
export const handle: Handle = async ({ event, resolve }) => {
  const response = await resolve(event);

  // 응답 헤더에 커스텀 값 추가
  response.headers.set('X-Custom-Header', 'my-value');

  return response;
};
```

브라우저 DevTools → Network → Response Headers에서 `X-Custom-Header: my-value`를 확인할 수 있다.

#### resolve 우회 — 커스텀 응답 / 리다이렉트

```ts
import { redirect } from '@sveltejs/kit';
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  // 패턴 A: 존재하지 않는 경로에 커스텀 응답 반환
  if (event.url.pathname.startsWith('/posts')) {
    return new Response('custom response');
    // → load 함수, 페이지 렌더링 전혀 실행되지 않음
  }

  // 패턴 B: 특정 경로를 다른 경로로 리다이렉트
  if (event.url.pathname.startsWith('/posts')) {
    redirect(303, '/blog');
    // → /posts 접속 시 /blog로 리다이렉트
  }

  return await resolve(event);
};
```

#### sequence로 여러 handle 함수 체이닝

```ts
// src/hooks.server.ts
import { sequence } from '@sveltejs/kit/hooks';
import { redirect } from '@sveltejs/kit';
import type { Handle } from '@sveltejs/kit';

const handleOne: Handle = async ({ event, resolve }) => {
  if (event.url.pathname.startsWith('/posts')) {
    redirect(303, '/blog');
  }
  return await resolve(event);
};

const handleTwo: Handle = async ({ event, resolve }) => {
  // 다른 역할의 처리 (로깅, 인증 등)
  return await resolve(event);
};

// handleOne → handleTwo 순서로 실행
export const handle = sequence(handleOne, handleTwo);
```

> `sequence`는 내부적으로 각 handle의 `resolve`를 다음 handle 함수로 연결한다. 첫 번째 handle이 `resolve(event)`를 호출하면 두 번째 handle이 실행되고, 두 번째가 `resolve(event)`를 호출하면 SvelteKit의 실제 처리가 실행된다.

---

## 3. event.locals — 요청 단위 데이터 전달

### locals란?

`event.locals`는 **하나의 요청 동안만 살아있는 객체**로, handle 훅에서 데이터를 주입하면 이후 모든 load 함수에서 접근할 수 있다.

```
Client → 요청
  │
  ▼
handle({ event, resolve })
  │
  │  event.locals.user = { name: 'John' }  ← 여기서 주입
  │
  ▼
resolve(event)
  │
  ├─→ +layout.server.ts  → event.locals.user 접근 가능
  ├─→ +page.server.ts    → event.locals.user 접근 가능
  └─→ +server.ts         → event.locals.user 접근 가능
  │
  ▼
응답 반환 → locals 소멸 (다음 요청에선 다시 빈 객체)
```

### locals vs 쿠키 — 왜 locals가 필요한가?

| 특성 | `event.locals` | `event.cookies` |
|---|---|---|
| 생존 범위 | 요청 1회 한정 | 브라우저에 지속 |
| 저장 위치 | 서버 메모리 (요청 중) | 클라이언트 쿠키 |
| 용도 | 요청 처리 중 임시 데이터 전달 | 인증 토큰 등 영구 데이터 |
| 보안 | 서버에만 존재, 외부 노출 없음 | 클라이언트에서 접근 가능 |

> **핵심**: "쿠키에서 토큰 확인 → 사용자 정보 조회 → locals에 주입"하면, 모든 load 함수에서 매번 쿠키를 파싱하지 않아도 된다.

### 인증에서 locals의 역할

#### 잘못된 패턴: 루트 레이아웃 load 함수에서 인증 체크

```ts
// src/routes/+layout.server.ts — ⚠️ 위험한 패턴
export const load: LayoutServerLoad = async ({ cookies, route }) => {
  const token = cookies.get('token');
  const user = token ? { name: 'John', id: 1 } : null;

  // 앱 그룹 접근 시 인증 체크
  if (!token && route.id?.startsWith('/(app)')) {
    redirect(303, '/login');
  }

  return { user };
};
```

**이 패턴이 위험한 이유:**

```
1. 브라우저에서 /app 직접 접속 (SSR)
   → 레이아웃 load 실행 ✅ → 토큰 없음 → 리다이렉트 ✅

2. 홈페이지에서 /app으로 클라이언트 내비게이션
   → 레이아웃 load 재실행 안 됨 ❌ → 토큰 체크 안 됨 → 접근 허용 ⚠️
```

루트 레이아웃의 load 함수는 **앱 최초 로드 시 1번만 실행**된다. 클라이언트 측 내비게이션에서는 해당 페이지의 load 함수만 실행되므로, 레이아웃 load에서 인증 체크를 하면 우회 가능하다.

> `route.id`는 경로의 폴더 구조를 그대로 반영한다: `/blog/1` → `/(marketing)/blog/[id]`

#### 차선책: await parent()로 레이아웃 load 재실행 강제

```ts
// src/routes/(app)/+page.server.ts — ⚠️ 작동은 하지만 비추천
export const load: PageServerLoad = async ({ parent }) => {
  await parent(); // 부모 레이아웃 load를 강제로 다시 실행
  // ...
};
```

**작동은 하지만 문제가 있다:**
- 루트 레이아웃에 많은 데이터 로드가 있으면, **매 요청마다 전부 재실행** → 성능 저하
- 기본적으로 모든 load 함수는 **병렬 실행**인데, `await parent()`는 **워터폴을 강제**함
- 인증 하나를 위해 전체 데이터 로딩 구조를 훼손하는 셈

#### 올바른 패턴: handle 훅에서 인증 체크

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

**왜 handle이 안전한가:**

- handle은 **모든 서버 요청**에서 실행됨 (SSR + 클라이언트 내비게이션의 `__data.json` 요청 포함)
- 클라이언트 내비게이션으로 `/app`에 접근해도 데이터 요청이 서버를 거치므로 handle이 실행됨
- 인증 우회가 불가능함

### locals의 TypeScript 타입 정의

`event.locals`에 임의의 데이터를 추가하면 TypeScript 오류가 발생한다. `src/app.d.ts`에서 타입을 정의해야 한다:

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

이후 모든 곳에서 `event.locals.user`를 타입 안전하게 사용 가능하다.

### 실습: locals로 인증 데이터 전달하기

#### handle에서 locals 주입 → 레이아웃에서 사용

```ts
// src/hooks.server.ts
export const handle: Handle = async ({ event, resolve }) => {
  const token = event.cookies.get('token');
  event.locals.user = token ? { name: 'John', id: 1 } : null;

  return await resolve(event);
};
```

```ts
// src/routes/+layout.server.ts
// 더 이상 쿠키를 직접 확인하지 않음 — locals에서 가져옴
export const load: LayoutServerLoad = async ({ locals }) => {
  return { user: locals.user };
};
```

```svelte
<!-- src/routes/(marketing)/+layout.svelte -->
<!-- 레이아웃에서 user 유무에 따라 UI 분기 -->
{#if data.user}
  <a href="/app">Dashboard</a>
{:else}
  <a href="/login">Login</a>
{/if}
```

#### 경로 보호: 2가지 전략

**전략 A: handle에서 중앙 집중 체크**

```ts
// src/hooks.server.ts
export const handle: Handle = async ({ event, resolve }) => {
  const token = event.cookies.get('token');
  event.locals.user = token ? { name: 'John', id: 1 } : null;

  // 보호 경로 일괄 체크
  if (!event.locals.user && event.url.pathname.startsWith('/app')) {
    redirect(303, '/login');
  }

  return await resolve(event);
};
```

> 장점: 한 곳에서 관리, 빠뜨릴 위험 없음
> 단점: 복잡한 앱에서 조건이 많아질 수 있음

**전략 B: 개별 페이지 load에서 분산 체크**

```ts
// src/hooks.server.ts — locals만 주입, 리다이렉트는 안 함
export const handle: Handle = async ({ event, resolve }) => {
  const token = event.cookies.get('token');
  event.locals.user = token ? { name: 'John', id: 1 } : null;
  return await resolve(event);
};
```

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

### 전체 흐름 정리

```
요청 → handle 훅
         │
         ├─ 쿠키에서 토큰 확인
         ├─ locals.user 주입
         ├─ (전략A) 보호 경로 체크 → 미인증이면 redirect
         │
         ▼
       resolve(event)
         │
         ├─ +layout.server.ts → locals.user → return { user }
         ├─ +page.server.ts   → (전략B) locals.user 체크
         │
         ▼
       응답 → locals 소멸
```

### React/Next.js와 비교

| 개념 | SvelteKit | Next.js |
|---|---|---|
| 요청 단위 데이터 전달 | `event.locals` | 없음 (헤더/쿠키로 우회) |
| locals 타입 정의 | `src/app.d.ts` → `App.Locals` | N/A |
| 인증 체크 위치 | `hooks.server.ts` (handle) | `middleware.ts` |
| 인증 우회 위험 | layout load에서만 하면 위험 | layout.tsx에서만 하면 동일 위험 |
| 안전한 패턴 | handle → locals → load | middleware → 헤더 → RSC |

---

## 4. resolve 옵션 — transformPageChunk, preload

`resolve(event)`를 호출할 때 **두 번째 인수로 옵션 객체**를 전달하여 렌더링 동작을 세밀하게 제어할 수 있다.

```ts
const response = await resolve(event, {
  transformPageChunk,
  preload,
  filterSerializedResponseHeaders
});
```

### transformPageChunk — HTML 변환

페이지가 렌더링되기 전에 **HTML을 가로채서 변환**할 수 있다. HTML은 청크(chunk) 단위로 전달되며, `done` 플래그가 `true`이면 마지막 청크다.

```ts
export const handle: Handle = async ({ event, resolve }) => {
  const lang = event.cookies.get('lang') || 'en';

  return await resolve(event, {
    transformPageChunk: ({ html, done }) => {
      // HTML 내 플레이스홀더를 실제 값으로 치환
      return html.replace('%lang%', lang);
    }
  });
};
```

#### 사용 예: 다국어 lang 속성

```html
<!-- src/app.html -->
<html lang="%lang%">
  <head>%sveltekit.head%</head>
  <body>%sveltekit.body%</body>
</html>
```

```ts
// src/hooks.server.ts
export const handle: Handle = async ({ event, resolve }) => {
  const lang = event.cookies.get('lang') || 'en';

  return await resolve(event, {
    transformPageChunk: ({ html }) => {
      return html.replace('%lang%', lang);
    }
  });
};
```

- 쿠키에 `lang=fr`이 설정되면 → `<html lang="fr">`로 렌더링
- 쿠키가 없으면 → `<html lang="en">`으로 폴백

> **청크 동작**: 짧은 페이지는 보통 1개 청크로 전달된다. 긴 페이지는 여러 청크로 나뉠 수 있지만, SvelteKit은 플레이스홀더(`%...%`)가 청크 경계에서 쪼개지지 않도록 보장한다.

### preload — 프리로드 파일 필터링

기본적으로 SvelteKit은 JS와 CSS 파일을 프리로드한다. `preload` 옵션으로 **프리로드할 파일 종류를 제어**할 수 있다.

```ts
export const handle: Handle = async ({ event, resolve }) => {
  return await resolve(event, {
    preload: ({ type, path }) => {
      // type: 'js' | 'css' | 'font' | 'asset'
      // 기본 JS/CSS + 폰트도 프리로드
      return type === 'js' || type === 'css' || type === 'font';
    }
  });
};
```

| type | 설명 | 기본 프리로드 |
|---|---|---|
| `js` | JavaScript 파일 | O |
| `css` | CSS 파일 | O |
| `font` | 폰트 파일 | X |
| `asset` | 이미지 등 기타 에셋 | X |

> 폰트 프리로드는 FOUT(Flash of Unstyled Text)를 줄이는 데 유용하다.

### filterSerializedResponseHeaders — 직렬화 헤더 필터링

이 옵션을 이해하려면 먼저 **유니버설 load 함수에서 fetch의 응답이 어떻게 캡처되는지** 알아야 한다.

#### 배경: 유니버설 load의 fetch 응답 캡처

유니버설 load 함수(`+page.ts`)는 서버와 클라이언트에서 **모두** 실행된다. 페이지를 새로고침하면:

1. **서버에서 실행**: fetch 요청 → 응답 받음 → HTML에 응답을 **인라인으로 캡처**
2. **클라이언트에서 실행 (hydration)**: 실제 fetch를 보내지 않고, 캡처된 응답을 읽음

```
서버 렌더링 시:
  +page.ts의 fetch → 실제 요청 → 응답 캡처 → HTML에 인라인

  <script>
    __sveltekit_fetched = {
      url: "https://api.example.com/posts",
      body: "[{...}, {...}]",    // 응답 본문 ✅
      headers: {}                // 헤더는 기본적으로 비어있음 ⚠️
    }
  </script>

클라이언트 hydration 시:
  +page.ts의 fetch → 캡처된 응답에서 읽음 (실제 요청 안 보냄)
```

> 클라이언트 측 내비게이션 시에는 캡처가 아닌 **실제 fetch 요청**이 전송된다. 캡처는 SSR → hydration 시에만 발생한다.

#### 문제: 클라이언트에서 응답 헤더에 접근할 수 없다

```ts
// src/routes/blog/+page.ts
export const load: PageLoad = async ({ fetch }) => {
  const response = await fetch('https://api.example.com/posts');

  // 서버에서는 정상 작동
  const contentType = response.headers.get('content-type'); // ✅ "application/json"

  // 클라이언트(hydration)에서는 캡처된 응답에 헤더가 없으므로:
  // ⚠️ 경고: Failed to get response header "content-type"
  // — filterSerializedResponseHeaders에 포함해야 합니다

  const posts = await response.json();
  return { posts };
};
```

#### 해결: filterSerializedResponseHeaders

```ts
// src/hooks.server.ts
export const handle: Handle = async ({ event, resolve }) => {
  return await resolve(event, {
    filterSerializedResponseHeaders: (name) => {
      // 캡처된 응답에 포함할 헤더 이름을 true로 반환
      return name === 'content-type';
    }
  });
};
```

이제 캡처된 응답에 `content-type` 헤더가 포함되므로, 클라이언트에서도 `response.headers.get('content-type')`이 정상 작동한다.

```ts
// 모든 헤더를 직렬화하려면 (권장하지 않음 — 보안상 필요한 것만)
filterSerializedResponseHeaders: () => true
```

#### 정리

| 상황 | fetch 동작 | 헤더 접근 |
|---|---|---|
| SSR 시 서버에서 실행 | 실제 fetch | 모든 헤더 접근 가능 |
| Hydration 시 클라이언트에서 실행 | 캡처된 응답 읽기 | `filterSerializedResponseHeaders`에 등록된 헤더만 |
| 클라이언트 내비게이션 | 실제 fetch | 모든 헤더 접근 가능 |

---

## 5. handleFetch 훅 — 서버 fetch 동작 수정

`handleFetch`는 load 함수의 `fetch`가 **서버에서 실행될 때만** 동작을 수정할 수 있는 훅이다.

```ts
// src/hooks.server.ts
import type { HandleFetch } from '@sveltejs/kit';

export const handleFetch: HandleFetch = async ({ request, fetch, event }) => {
  // request: 원본 fetch 요청
  // fetch: 실제 fetch 함수
  // event: 요청 이벤트 (cookies, locals 등 접근 가능)

  return await fetch(request); // 기본 동작
};
```

### 사용 사례 1: 서버에서 내부 URL로 교체

```ts
// 클라이언트: https://api.example.com/posts (공개 URL)
// 서버: http://localhost:3000/posts (내부 URL — 더 빠름)

export const handleFetch: HandleFetch = async ({ request, fetch }) => {
  if (request.url.startsWith('https://api.example.com/')) {
    const url = request.url.replace(
      'https://api.example.com/',
      'http://localhost:3000/'
    );
    request = new Request(url, request);
  }

  return await fetch(request);
};
```

> 클라이언트에서는 공개 URL을 사용하지만, 서버에서는 네트워크를 거치지 않고 로컬 API를 직접 호출하여 지연 시간을 줄인다.

### 사용 사례 2: 크로스 오리진 요청에 쿠키 전달

```ts
// 기본적으로 SvelteKit의 fetch는 동일 오리진에만 쿠키를 상속
// 크로스 오리진 요청에는 쿠키가 포함되지 않음

export const handleFetch: HandleFetch = async ({ request, fetch, event }) => {
  if (request.url.startsWith('https://other-api.example.com/')) {
    request.headers.set('cookie', event.request.headers.get('cookie') ?? '');
  }

  return await fetch(request);
};
```

### handleFetch vs handle 비교

| 특성 | `handle` | `handleFetch` |
|---|---|---|
| 트리거 | 모든 서버 요청 | load 함수의 fetch (서버 실행 시만) |
| 용도 | 요청/응답 커스터마이징, 인증 | fetch 동작 수정 (URL 교체, 헤더 추가) |
| 실행 빈도 | 매 요청마다 | load의 fetch 호출마다 |

---

## 6. 에러 페이지 커스터마이징

SvelteKit에서 오류는 **예상된 오류(expected)**와 **예상치 못한 오류(unexpected)** 두 종류가 있으며, 각각 다른 방식으로 표시된다.

### 오류의 두 종류

| 종류 | 발생 방식 | 예시 | HTTP 상태 |
|---|---|---|---|
| 예상된 오류 | `error()` 함수로 직접 throw | 존재하지 않는 게시물 요청 | 개발자가 지정 (404, 403 등) |
| 예상치 못한 오류 | 코드 버그, 타입 에러 등 | `fetc`(오타) 호출 | 500 |

```ts
// 예상된 오류 — 사용자에게 보여줄 의도가 있음
import { error } from '@sveltejs/kit';

if (!post) {
  error(404, 'Post not found');
}

// 예상치 못한 오류 — 코드 버그
const res = await fetc('/api/posts'); // ← 오타 → 500 Internal Error
```

### +error.svelte 파일 — 경로별 오류 페이지

`+error.svelte` 파일을 경로 폴더에 배치하여 **경로별로 다른 오류 페이지**를 표시할 수 있다.

```
src/routes/
├── +error.svelte              ← 모든 경로의 폴백 오류 페이지
├── +layout.svelte
├── (marketing)/
│   ├── +error.svelte          ← 마케팅 그룹 전용 오류 페이지
│   └── blog/
│       └── [id]/
│           ├── +error.svelte  ← blog/[id] 전용 오류 페이지
│           └── +page.svelte
└── (app)/
    └── +page.svelte
```

#### 오류 페이지 탐색 순서

오류가 발생하면 SvelteKit은 **해당 경로에서 위로 올라가며** 가장 가까운 `+error.svelte`를 찾는다:

```
/blog/999 에서 오류 발생
  → routes/(marketing)/blog/[id]/+error.svelte  ← 있으면 사용
  → routes/(marketing)/+error.svelte            ← 없으면 상위로
  → routes/+error.svelte                        ← 최종 폴백
```

#### 오류 정보 접근 — page 상태

```svelte
<!-- src/routes/(marketing)/blog/[id]/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state';
</script>

<h1>{page.status}</h1>
<p>{page.error?.message}</p>
{#if page.error?.code}
  <p>Error code: {page.error.code}</p>
{/if}
```

#### 커스텀 에러 데이터 전달

`error()` 함수에 문자열 대신 **객체를 전달**하여 추가 정보를 포함할 수 있다:

```ts
// +page.server.ts
error(404, {
  message: 'Post not found',
  code: 'POST_NOT_FOUND'  // 커스텀 필드
});
```

커스텀 필드를 추가하면 `src/app.d.ts`에 타입을 정의해야 한다:

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Error {
      message: string;   // 기본 포함
      code?: string;     // 커스텀 추가
    }
  }
}
export {};
```

### 레이아웃 오류의 특수 동작

**레이아웃 load 함수에서 오류가 발생하면**, 같은 레벨의 `+error.svelte`가 아닌 **상위 레벨의 `+error.svelte`**가 사용된다:

```
(marketing)/+layout.server.ts 에서 오류 발생
  → (marketing)/+error.svelte    ❌ 사용되지 않음 (같은 레벨)
  → routes/+error.svelte         ✅ 상위 레벨 사용
```

> 이유: 레이아웃에 오류가 있으면 같은 레벨의 오류 페이지도 그 레이아웃에 의존하므로 렌더링할 수 없다.

### 루트 레이아웃 오류 → 정적 폴백 페이지

루트 레이아웃(`src/routes/+layout.server.ts`)에서 오류가 발생하면 위로 올라갈 곳이 없다. 이 경우 **정적 HTML 폴백 페이지**가 표시된다.

```html
<!-- src/error.html — 정적 폴백 페이지 -->
<!DOCTYPE html>
<html>
<head>
  <title>Error</title>
</head>
<body>
  <h1>%sveltekit.status%</h1>
  <p>%sveltekit.error.message%</p>
</body>
</html>
```

- `%sveltekit.status%` → HTTP 상태 코드로 치환
- `%sveltekit.error.message%` → 오류 메시지로 치환
- Svelte 컴포넌트가 아닌 **순수 HTML** 파일이다 (SvelteKit 런타임이 로드되지 않은 상태)

### +error.svelte 자체에 오류가 있으면?

오류 페이지 자체에 버그가 있으면 **상위 오류 페이지로 폴백**한다:

```
blog/[id]/+error.svelte 에 오류 있음
  → routes/+error.svelte 로 폴백

routes/+error.svelte 에도 오류 있음
  → src/error.html 정적 페이지로 폴백
```

### 오류 페이지 체계 정리

```
오류 발생
  │
  ▼
해당 경로의 +error.svelte 있나?
  ├─ Yes → 해당 오류 페이지 렌더링
  └─ No  → 상위 경로로 올라가며 탐색
              │
              ├─ 찾음 → 해당 오류 페이지 렌더링
              └─ 루트까지 없음 → src/error.html (정적 폴백)

※ 레이아웃 오류: 같은 레벨 건너뛰고 상위에서 시작
※ +error.svelte 자체 오류: 상위로 폴백
```

---

## 7. handleError 훅 — 예상치 못한 오류 처리

`handleError`는 **예상치 못한 오류**(코드 버그 등)가 발생할 때 실행되는 훅이다. `error()` 함수로 직접 던진 예상된 오류에는 실행되지 않는다.

### 공유 훅(Shared Hook) vs 유니버설 훅(Universal Hook)

| 종류 | 파일 | 설명 |
|---|---|---|
| 유니버설 훅 | `hooks.ts` | **하나의 함수**가 서버와 클라이언트 모두에서 실행 |
| 공유 훅 | `hooks.server.ts` + `hooks.client.ts` | 서버/클라이언트에 **각각 별도 함수**를 정의 |

`handleError`는 **공유 훅**이다. `hooks.ts`에 하나만 둘 수 없고, 서버/클라이언트 각각에 별도로 정의해야 한다.

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

```
브라우저에서 직접 URL 접속 → SSR → +page.server.ts 오류
  → hooks.server.ts의 handleError 실행
  → 터미널에 로그

클라이언트 내비게이션 → +page.ts 오류 (브라우저에서 실행)
  → hooks.client.ts의 handleError 실행
  → 브라우저 콘솔에 로그
```

### init 훅 (간단 참고)

`init`도 공유 훅으로, 서버/클라이언트 각각에 배치 가능하다:

```ts
// src/hooks.server.ts — 서버 시작 시 1번만 실행
export const init = async () => {
  await connectDatabase();
};

// src/hooks.client.ts — 브라우저에서 앱 첫 로드 시 1번만 실행
export const init = async () => {
  initAnalytics();
};
```

### 예상된 오류 vs 예상치 못한 오류 — 전체 비교

| 특성 | 예상된 오류 | 예상치 못한 오류 |
|---|---|---|
| 발생 방식 | `error(404, 'Not found')` | 코드 버그, 런타임 에러 |
| `handleError` 실행 | X | O |
| 사용자에게 보이는 메시지 | `error()`에 전달한 메시지 그대로 | 기본 "Internal Error" 또는 `handleError` 반환값 |
| 실제 오류 정보 | 사용자에게 노출 가능 | **민감 정보** → 서버 로그에만, 사용자에게 노출 금지 |
| HTTP 상태 | 개발자가 지정 | 500 |

---

## 8. 유니버설 훅 — reroute, transport

유니버설 훅은 `src/hooks.ts`에 하나의 함수로 정의하면 **서버와 클라이언트 모두에서 실행**된다. 공유 훅(`handleError`)과 달리 별도의 파일에 각각 정의할 필요가 없다.

### reroute — URL을 다른 경로에 매칭

URL을 변경하지 않으면서, 특정 URL 요청을 **다른 경로의 파일로 매칭**시킬 수 있다. 리다이렉트와 달리 **브라우저 URL이 바뀌지 않는다**.

```ts
// src/hooks.ts
import type { Reroute } from '@sveltejs/kit';

export const reroute: Reroute = ({ url }) => {
  // /posts 요청 → /(marketing)/blog 경로 파일로 매칭
  if (url.pathname === '/posts') {
    return '/(marketing)/blog';
  }
};
```

#### reroute vs redirect 차이

| 특성 | `reroute` | `redirect` |
|---|---|---|
| 브라우저 URL | 변경 안 됨 (`/posts` 유지) | 변경됨 (`/blog`로 이동) |
| HTTP 응답 | 200 (정상) | 303/307 (리다이렉트) |
| 사용 위치 | `hooks.ts` (유니버설) | handle 훅, load 함수 등 |
| 용도 | URL 별칭, 다국어 경로 | 실제 페이지 이동 |

#### 실용 예: 다국어 라우팅

```ts
// src/hooks.ts
const translations: Record<string, string> = {
  '/de/ueber-uns': '/about',   // 독일어
  '/fr/a-propos': '/about',    // 프랑스어
};

export const reroute: Reroute = ({ url }) => {
  return translations[url.pathname];
};
```

> `/de/ueber-uns`를 요청하면 URL은 그대로 유지되지만, 내부적으로 `/about` 경로의 파일이 사용된다.

### transport — 직렬화 불가능한 데이터 전송

서버 load 함수에서 **클래스 인스턴스** 같은 직렬화할 수 없는 데이터를 반환하면 오류가 발생한다. `transport` 훅으로 인코딩/디코딩 방법을 정의하여 해결할 수 있다.

#### 문제: 클래스 인스턴스는 직렬화 불가

```ts
// src/lib/Rect.ts
export class Rect {
  constructor(
    public x: number,
    public y: number,
    public width: number,
    public height: number
  ) {}

  get area() {
    return this.width * this.height;
  }
}
```

```ts
// +page.server.ts
import { Rect } from '$lib/Rect';

export const load = async () => {
  return {
    rect: new Rect(0, 0, 100, 50) // ❌ 직렬화 불가 → 오류
  };
};
```

> "Data returned from load function is not serializable" 오류 발생

#### 해결: transport 훅으로 인코딩/디코딩 정의

```ts
// src/hooks.ts
import type { Transport } from '@sveltejs/kit';
import { Rect } from '$lib/Rect';

export const transport: Transport = {
  Rect: {
    // 서버에서 실행: 클래스 인스턴스 → 직렬화 가능한 형태로 인코딩
    encode: (value) => {
      if (value instanceof Rect) {
        return [value.x, value.y, value.width, value.height];
      }
    },

    // 클라이언트에서 실행: 인코딩된 데이터 → 클래스 인스턴스로 복원
    decode: ([x, y, width, height]) => {
      return new Rect(x, y, width, height);
    }
  }
};
```

#### 동작 흐름

```
서버 load 함수
  │
  │  return { rect: new Rect(0, 0, 100, 50) }
  │
  ▼
transport.Rect.encode(rect)
  │
  │  → [0, 0, 100, 50]  (직렬화 가능한 배열)
  │
  ▼
HTML에 인라인 → 클라이언트로 전송
  │
  ▼
transport.Rect.decode([0, 0, 100, 50])
  │
  │  → new Rect(0, 0, 100, 50)  (클래스 인스턴스 복원)
  │
  ▼
컴포넌트에서 data.rect.area 등 메서드 사용 가능
```

### 훅 종류 전체 정리

| 훅 | 파일 | 종류 | 설명 |
|---|---|---|---|
| `handle` | `hooks.server.ts` | 서버 전용 | 모든 서버 요청 가로채기 |
| `handleFetch` | `hooks.server.ts` | 서버 전용 | load의 fetch 동작 수정 |
| `handleError` | `.server.ts` + `.client.ts` | 공유 | 예상치 못한 오류 처리 (각각 정의) |
| `init` | `.server.ts` + `.client.ts` | 공유 | 초기화 (서버 시작 / 앱 첫 로드) |
| `reroute` | `hooks.ts` | 유니버설 | URL → 경로 매칭 변경 |
| `transport` | `hooks.ts` | 유니버설 | 직렬화 불가 데이터 인코딩/디코딩 |
