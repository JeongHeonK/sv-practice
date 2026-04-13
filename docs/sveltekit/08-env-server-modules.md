# SvelteKit 환경 변수 & 서버 전용 모듈

환경 변수 시스템(`$env/*` 모듈), 접근 제어(public/private), 평가 시점(static/dynamic), 서버 전용 모듈(`*.server.ts`, `$lib/server/`).

---

## 1. 환경 변수 — 정의와 접근 제어

### 환경 변수 소스 2가지

| 소스 | 위치 | 예시 |
|---|---|---|
| `.env` 파일 | 프로젝트 루트 | `PUBLIC_API_URL=https://...` |
| 시스템 환경 변수 | OS / 서버 / CI | `export DB_PASSWORD=secret` |

> `.env` 파일은 내부적으로 `dotenv` 라이브러리로 로드된다.

### public / private 접두사

```bash
# .env
PUBLIC_APP_NAME=MyApp        # 공개 — 서버 + 클라이언트 모두 접근 가능
DATABASE_URL=postgres://...  # 비공개 — 서버에서만 접근 가능
```

| 구분 | 접두사 | 접근 범위 | 민감 데이터 |
|---|---|---|---|
| **Public** | `PUBLIC_` | 서버 + 클라이언트 | ❌ 넣으면 안 됨 |
| **Private** | 없음 (기본값) | 서버 전용 | ✅ API 키, DB 비밀번호 등 |

> 클라이언트에서 private 변수를 import하면 **빌드 에러**가 발생한다 — SvelteKit이 실수를 방지해준다.

### 접두사 커스터마이징 (보통 기본값 유지)

```ts
// svelte.config.js
const config = {
  kit: {
    env: {
      publicPrefix: 'PUBLIC_',  // 기본값
      privatePrefix: ''         // 기본값 (빈 문자열 = 접두사 없음)
    }
  }
};
```

> `privatePrefix`를 빈 문자열로 두는 것이 안전하다 — 접두사를 까먹어도 기본이 private이므로 유출 위험이 없다.

---

## 2. 4가지 `$env` 모듈

SvelteKit은 **접근 범위(public/private)** × **평가 시점(static/dynamic)** 조합으로 4개 모듈을 제공한다.

```
$env/static/private    — 빌드 타임 + 서버 전용
$env/static/public     — 빌드 타임 + 서버·클라이언트
$env/dynamic/private   — 런타임 + 서버 전용
$env/dynamic/public    — 런타임 + 서버·클라이언트
```

### static vs dynamic 핵심 차이

| 특성 | `static` | `dynamic` |
|---|---|---|
| **평가 시점** | 빌드 타임 (코드에 인라인) | 런타임 (서버에서 동적 평가) |
| **값 변경 시** | 앱 다시 빌드 필요 | 재시작만으로 반영 |
| **import 방식** | 개별 변수 named import | `env` 객체에서 접근 |
| **번들 영향** | 값이 코드에 하드코딩됨 | 서버에서 평가 후 클라이언트로 전송 |

### 사용 예: Private 환경 변수

```ts
// +page.server.ts (서버 전용 파일에서만 import 가능)

// ① static — 빌드 타임에 값이 코드에 주입됨
import { DATABASE_URL, API_SECRET } from '$env/static/private';

// ② dynamic — 런타임에 process.env 등에서 평가됨
import { env } from '$env/dynamic/private';

export const load = async () => {
  console.log(DATABASE_URL);        // static: 빌드 시 "postgres://..." 로 치환됨
  console.log(env.DATABASE_URL);    // dynamic: 런타임에 평가됨
};
```

### 사용 예: Public 환경 변수

```ts
// +page.ts 또는 +page.svelte (클라이언트에서도 접근 가능)

// ① static — 빌드 타임에 코드에 주입
import { PUBLIC_APP_NAME } from '$env/static/public';

// ② dynamic — 서버에서 평가 → 클라이언트로 전송
import { env } from '$env/dynamic/public';

console.log(PUBLIC_APP_NAME);       // 빌드 시 인라인됨
console.log(env.PUBLIC_APP_NAME);   // 서버에서 평가 후 전송됨 (네트워크 비용 약간 증가)
```

### 클라이언트에서 private import 시 에러

```ts
// +page.ts (유니버설 — 클라이언트에서도 실행됨)
import { API_SECRET } from '$env/static/private';
// ❌ Error: Cannot import $env/static/private into client-side code
```

---

## 3. 어떤 모듈을 써야 하나? — 선택 가이드

```
환경 변수가 민감한가?
├─ YES → private 모듈 (서버 전용 파일에서만)
│   ├─ 값이 자주 바뀌나? → $env/dynamic/private
│   └─ 거의 안 바뀌나? → $env/static/private ✅ (추천)
│
└─ NO → public 모듈
    ├─ 값이 자주 바뀌나? → $env/dynamic/public
    └─ 거의 안 바뀌나? → $env/static/public ✅ (추천)
```

> **기본 원칙**: 가능하면 `static`을 쓴다 — 번들에 인라인되어 네트워크 비용이 없고 성능이 더 좋다. `dynamic`은 값이 서버에서 클라이언트로 전송되므로 약간의 오버헤드가 있다.

### static이 적합한 경우

- API base URL, 앱 이름, feature flag 등 **배포 후 거의 변하지 않는 값**
- DB 연결 문자열 (빌드 환경과 런타임 환경이 동일한 경우)

### dynamic이 필요한 경우

- 서버 재시작만으로 값을 바꿔야 하는 경우 (재빌드 없이)
- 빌드 환경과 런타임 환경의 환경 변수가 다른 경우
- `.env` 파일 값의 프로덕션 가용성이 보장되지 않는 플랫폼

---

## 4. `$app/environment` — 실행 컨텍스트 플래그

환경 변수와 달리 **SvelteKit이 제공하는 런타임 플래그**. 코드가 어느 상황에서 실행되는지 감지한다.

```ts
import { browser, dev, building } from '$app/environment'
```

| 플래그 | 설명 | 주요 용도 |
|---|---|---|
| `browser` | 브라우저에서 실행 중이면 `true` | `window` 접근, 클라이언트 전용 로직 분기 |
| `dev` | 개발 모드이면 `true` | 쿠키 `secure: !dev`, 디버그 로그 |
| `building` | `npm run build` 빌드 중이면 `true` | prerender 중 DB/동적 env 접근 방지 |

### `building` 플래그 — prerender 빌드 시 DB 초기화 방지

`prerender = true` 라우트가 있으면 빌드 시 해당 페이지를 실행한다. 이때 동적 환경 변수(`$env/dynamic/*`)를 읽으면 에러가 발생한다.

```ts
// src/lib/server/db.ts
import { building } from '$app/environment'
import { env } from '$env/dynamic/private'

// 빌드 중에는 DB 초기화 건너뜀
export const db = building ? null : createPool(env.DATABASE_URL)
```

> `$lib/server/` 안의 유틸 함수가 `$app/server`의 `getRequestEvent()` 등을 사용하면 서버 전용 함수다. 클라이언트에서 import 가능한 `$lib/utils.ts`에 넣으면 빌드 에러 발생 — 반드시 `$lib/server/`에 배치해야 한다.

---

## 요약 — 4개 모듈 한눈에

| 모듈 | 접근 범위 | 평가 시점 | import 방식 | 값 변경 시 |
|---|---|---|---|---|
| `$env/static/private` | 서버 전용 | 빌드 타임 | `import { VAR }` | 재빌드 |
| `$env/static/public` | 서버 + 클라이언트 | 빌드 타임 | `import { PUBLIC_VAR }` | 재빌드 |
| `$env/dynamic/private` | 서버 전용 | 런타임 | `import { env }` → `env.VAR` | 재시작 |
| `$env/dynamic/public` | 서버 + 클라이언트 | 런타임 | `import { env }` → `env.PUBLIC_VAR` | 재시작 |

---

## 6. 서버 전용 모듈 — `*.server.ts` & `$lib/server/`

환경 변수뿐 아니라 **모듈 자체를 서버 전용으로 제한**할 수 있다. DB 연결, 파일 시스템 접근 등 서버에서만 실행되어야 하는 코드를 클라이언트 번들에서 완전히 격리한다.

### 방법 1: 파일명에 `.server` 붙이기

```ts
// src/lib/db.server.ts — 서버 전용 모듈
import { DATABASE_URL } from '$env/static/private';

export function connectDB() {
  // DB 연결 로직 — 서버에서만 실행
}
```

```ts
// +page.server.ts — ✅ 서버 전용 파일에서 import 가능
import { connectDB } from '$lib/db.server';
```

```ts
// +page.ts — ❌ 클라이언트에서 실행될 수 있는 파일
import { connectDB } from '$lib/db.server';
// Error: Cannot import a server-only module into client-side code
```

### 방법 2: `$lib/server/` 폴더에 넣기

```
src/lib/
├── server/          ← 이 폴더 안의 모든 파일이 서버 전용
│   ├── db.ts
│   ├── auth.ts
│   └── email.ts
└── utils.ts         ← 일반 모듈 (서버 + 클라이언트)
```

```ts
// src/lib/server/db.ts — 폴더 자체가 서버 전용
import { DATABASE_URL } from '$env/static/private';

export function connectDB() { /* ... */ }
```

```ts
// +page.ts — ❌ 동일하게 에러
import { connectDB } from '$lib/server/db';
// Error: Cannot import $lib/server/db into client-side code
```

### 두 방법 비교

| 방법 | 사용 시점 | 예시 |
|---|---|---|
| `파일명.server.ts` | 서버 전용 파일이 1~2개일 때 | `db.server.ts`, `email.server.ts` |
| `$lib/server/` 폴더 | 서버 전용 코드가 많을 때 (권장) | `server/db.ts`, `server/auth.ts` |

### 서버 전용 모듈 안에서 private 환경 변수 사용

서버 전용 모듈에서는 private 환경 변수를 **안전하게 import**할 수 있다 — 클라이언트 번들에 포함되지 않으므로.

```ts
// src/lib/server/db.ts
import { DATABASE_URL } from '$env/static/private';  // ✅ 안전

export function connectDB() {
  return createPool(DATABASE_URL);
}
```

### ⚠️ 주의: 서버 전용 ≠ 비밀 저장소

```ts
// src/lib/server/config.ts
const SECRET_KEY = 'sk_live_abc123';  // ❌ 절대 하지 말 것!
```

서버 전용 모듈은 클라이언트 번들에는 포함되지 않지만, **코드는 Git 저장소에 푸시**된다. 비밀 값은 반드시 환경 변수에 저장해야 한다.

```ts
// ✅ 올바른 방법
import { SECRET_KEY } from '$env/static/private';
```

---

## React/Next.js 비교

| 특성 | SvelteKit | Next.js |
|---|---|---|
| **공개 접두사** | `PUBLIC_` | `NEXT_PUBLIC_` |
| **비공개** | 접두사 없음 (기본) | 접두사 없음 |
| **모듈 시스템** | 4개 전용 모듈 (`$env/*`) | `process.env` 단일 접근 |
| **빌드 vs 런타임** | `static` / `dynamic` 명시적 선택 | `NEXT_PUBLIC_`만 빌드 타임 인라인 |
| **잘못된 접근 방지** | 빌드 에러로 차단 | 서버 변수는 클라이언트에서 `undefined` |
| **접두사 커스터마이징** | `svelte.config.js`에서 가능 | 불가 |
| **서버 전용 파일** | `*.server.ts` 또는 `$lib/server/` | `'use server'` 디렉티브 |
| **서버 전용 import 차단** | 빌드 에러 | `server-only` 패키지 수동 설치 필요 |
| **보호 수준** | 프레임워크 내장 (자동) | 명시적 설정 필요 (`import 'server-only'`) |
