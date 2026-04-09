# SvelteKit 고급 훅

resolve 옵션, handleFetch, reroute, transport 등 고급/특수 훅.
기본 훅(handle, locals, handleError)은 [05-hooks.md](05-hooks.md)를 참고한다.

---

## 1. resolve 옵션 — transformPageChunk, preload, filterSerializedResponseHeaders

`resolve(event)`를 호출할 때 **두 번째 인수로 옵션 객체**(`transformPageChunk`, `preload`, `filterSerializedResponseHeaders`)를 전달하여 렌더링 동작을 세밀하게 제어할 수 있다.

### transformPageChunk — HTML 변환

페이지가 렌더링되기 전에 **HTML을 가로채서 변환**할 수 있다. HTML은 청크(chunk) 단위로 전달되며(`{ html, done }`), `done`이 `true`이면 마지막 청크다.

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

SSR 시 유니버설 load 함수의 fetch 응답은 HTML에 인라인으로 캡처되는데, **기본적으로 응답 헤더는 포함되지 않는다**. 클라이언트(hydration)에서 응답 헤더에 접근해야 한다면 이 옵션으로 포함할 헤더를 지정한다.

```ts
// src/hooks.server.ts
export const handle: Handle = async ({ event, resolve }) => {
  return await resolve(event, {
    filterSerializedResponseHeaders: (name) => {
      return name === 'content-type'; // 캡처 응답에 포함할 헤더만 true
    }
  });
};
// 모든 헤더를 직렬화하려면: filterSerializedResponseHeaders: () => true (보안상 비권장)
```

| 상황 | fetch 동작 | 헤더 접근 |
|---|---|---|
| SSR 시 서버에서 실행 | 실제 fetch | 모든 헤더 접근 가능 |
| Hydration 시 클라이언트에서 실행 | 캡처된 응답 읽기 | `filterSerializedResponseHeaders`에 등록된 헤더만 |
| 클라이언트 내비게이션 | 실제 fetch | 모든 헤더 접근 가능 |

---

## 2. handleFetch 훅 — 서버 fetch 동작 수정

`handleFetch`는 load 함수의 `fetch`가 **서버에서 실행될 때만** 동작을 수정할 수 있는 훅이다.

```ts
// src/hooks.server.ts
import type { HandleFetch } from '@sveltejs/kit';

export const handleFetch: HandleFetch = async ({ request, fetch, event }) => {
  // request: 원본 fetch 요청, fetch: 실제 fetch 함수, event: 요청 이벤트
  return await fetch(request); // 기본 동작
};
```

### 사용 사례 1: 서버에서 내부 URL로 교체

```ts
export const handleFetch: HandleFetch = async ({ request, fetch }) => {
  // 공개 URL → 내부 URL로 교체 (서버에서 더 빠름)
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

### 사용 사례 2: 크로스 오리진 요청에 쿠키 전달

```ts
// 기본적으로 동일 오리진에만 쿠키 상속 — 크로스 오리진에는 수동 전달 필요
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

## 3. reroute 훅 — URL을 다른 경로에 매칭

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

### reroute vs redirect 차이

| 특성 | `reroute` | `redirect` |
|---|---|---|
| 브라우저 URL | 변경 안 됨 (`/posts` 유지) | 변경됨 (`/blog`로 이동) |
| HTTP 응답 | 200 (정상) | 303/307 (리다이렉트) |
| 사용 위치 | `hooks.ts` (유니버설) | handle 훅, load 함수 등 |
| 용도 | URL 별칭, 다국어 경로 | 실제 페이지 이동 |

### 실용 예: 다국어 라우팅

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

---

## 4. transport 훅 — 직렬화 불가능한 데이터 전송

서버 load 함수에서 **클래스 인스턴스** 같은 직렬화할 수 없는 데이터를 반환하면 오류가 발생한다. `transport` 훅으로 인코딩/디코딩 방법을 정의하여 해결할 수 있다.

### 문제: 클래스 인스턴스는 직렬화 불가

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
    rect: new Rect(0, 0, 100, 50) // ❌ "Data returned from load is not serializable"
  };
};
```

### 해결: transport 훅으로 인코딩/디코딩 정의

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

### 동작 흐름

```
서버 load 함수
  │  return { rect: new Rect(0, 0, 100, 50) }
  ▼
transport.Rect.encode(rect)
  │  → [0, 0, 100, 50]  (직렬화 가능한 배열)
  ▼
HTML에 인라인 → 클라이언트로 전송
  ▼
transport.Rect.decode([0, 0, 100, 50])
  │  → new Rect(0, 0, 100, 50)  (클래스 인스턴스 복원)
  ▼
컴포넌트에서 data.rect.area 등 메서드 사용 가능
```

---

## 훅 종류 전체 정리

| 훅 | 파일 | 종류 | 설명 |
|---|---|---|---|
| `handle` | `hooks.server.ts` | 서버 전용 | 모든 서버 요청 가로채기 |
| `handleFetch` | `hooks.server.ts` | 서버 전용 | load의 fetch 동작 수정 |
| `handleError` | `.server.ts` + `.client.ts` | 공유 | 예상치 못한 오류 처리 (각각 정의) |
| `init` | `.server.ts` + `.client.ts` | 공유 | 초기화 (서버 시작 / 앱 첫 로드) |
| `reroute` | `hooks.ts` | 유니버설 | URL → 경로 매칭 변경 |
| `transport` | `hooks.ts` | 유니버설 | 직렬화 불가 데이터 인코딩/디코딩 |
