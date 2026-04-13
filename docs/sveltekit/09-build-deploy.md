# SvelteKit 빌드 & 배포

빌드 결과물, 어댑터 시스템, Node 서버 / Vercel 서버리스 두 가지 배포 방식.

---

## 1. 배포 방식 2가지

```
Traditional (Node 서버)          Serverless (Vercel / Netlify)
─────────────────────────        ─────────────────────────────
항상 실행 중인 서버 1대            요청마다 함수가 실행됨
요청 → 서버 → 응답                요청 → 함수 spin-up → 응답
                                  (콜드 스타트 있음)

장점: 소켓·트랜잭션·long-running   장점: 인프라 관리 불필요, 자동 스케일
단점: 유휴 시간에도 비용 발생        단점: 실행 시간 제한, DB 연결 방식 달라짐
```

SvelteKit은 두 방식을 **어댑터** 하나만 바꿔 전환할 수 있다.

---

## 2. 어댑터

`svelte.config.js`의 `adapter` 항목이 빌드 결과물을 결정한다.

| 어댑터 | 용도 |
|---|---|
| `adapter-auto` | 환경 자동 감지 (Vercel·Netlify·Cloudflare Pages·Azure·AWS·Google Cloud Run 지원) |
| `adapter-node` | Node.js 서버 생성 — 모든 Node 호스팅 서비스에서 사용 가능 |
| `adapter-static` | 정적 사이트 — 앱 전체가 pre-render 가능할 때만 |
| `adapter-vercel` | Vercel 전용, 세부 옵션(리전·함수 분리 등) 지정 가능 |

> 공식 어댑터 외에 커뮤니티 어댑터도 있다. `adapter-auto`를 쓰다가 플랫폼이 확정되면 해당 어댑터를 직접 설치하는 게 권장된다 (lockfile 추가, CI 설치 시간 단축).

---

## 3. Node 어댑터

### 설치 & 설정

```bash
npm install -D @sveltejs/adapter-node
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node';

export default {
  kit: { adapter: adapter() }
};
```

```json
// package.json — 서버 시작 스크립트 추가 필수
{
  "scripts": {
    "start": "node build"
  }
}
```

`npm run build` 실행 후 `build/` 폴더가 생성된다. 이 폴더가 배포 대상이다.

### ORIGIN 환경 변수 — CSRF 방지 필수

Node 서버로 폼을 제출하면 `cross-site POST form submissions are forbidden` 에러가 발생한다.  
Node 어댑터는 `ORIGIN`을 신뢰할 도메인으로 등록해야 한다.

```bash
HOST=localhost ORIGIN=http://localhost:3000 node build
```

프로덕션에서는 서버의 환경 변수에 `ORIGIN=https://your-domain.com`을 설정한다.

> `npm run preview`는 Node에서 실행되지만 어댑터 특화 context(`platform` 객체 등)가 적용되지 않는다 — 실제 배포 환경과 완전히 동일하지 않다.

---

## 4. Vercel 어댑터

### 설치 & 설정

```bash
npm install -D @sveltejs/adapter-vercel
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

export default {
  kit: { adapter: adapter() }  // 기본값으로 충분한 경우가 대부분
};
```

### 라우트별 Vercel 옵션 — per-route config

특정 레이아웃·페이지에만 다른 배포 옵션을 지정할 수 있다.

```ts
// +layout.server.ts 또는 +page.server.ts
import type { Config } from '@sveltejs/adapter-vercel';

export const config: Config = {
  regions: ['iad1'],  // 특정 리전 고정
  split: true,        // 이 라우트만 개별 함수로 분리 배포
};
```

> `runtime` 옵션은 **deprecated** (공식 문서 기준) — Vercel 대시보드의 프로젝트 설정에서 Node 버전을 지정하는 방식으로 대체됨.

> `adapter-auto`를 사용할 경우 per-route config를 지정할 수 없다 — 옵션이 필요하면 `adapter-vercel`을 직접 사용해야 한다.

---

## 5. Serverless 환경 DB 드라이버

Vercel 같은 서버리스 환경에서는 **DB 연결 유지 방식**이 달라진다.  
표준 `postgres` 드라이버 대신 **Neon Serverless 드라이버**를 사용해야 한다.

```
왜 달라지나?
서버리스 함수는 요청마다 새로 실행됨 → 영구 DB 연결 유지 불가
→ HTTP 또는 WebSocket 기반 단발성 연결 방식 필요
```

```bash
npm install @neondatabase/serverless
```

```ts
// src/lib/server/db.ts
import { building } from '$app/environment';
import { env } from '$env/dynamic/private';
import { Pool, neonConfig } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-serverless';
import ws from 'ws';

neonConfig.webSocketConstructor = ws;  // 트랜잭션 지원에 필요

export const db = building
  ? null
  : drizzle({ client: new Pool({ connectionString: env.DATABASE_URL }) });
```

> HTTP 방식(`neon()`)은 트랜잭션을 지원하지 않는다 — 트랜잭션이 있으면 반드시 WebSocket 방식(`Pool`)을 사용해야 한다.

---

## 6. 배포 체크리스트

```
□ 환경 변수 등록 (DATABASE_URL, BETTER_AUTH_SECRET, BETTER_AUTH_URL 등)
□ ORIGIN 설정 (Node 서버 배포 시)
□ npm run db:migrate 실행 — 새 마이그레이션이 있으면 프로덕션 DB에 적용
□ OAuth 앱 콜백 URL 프로덕션 도메인으로 업데이트
```

> CI/CD 워크플로에서 **앱 배포 + migrate 실행**을 묶어서 자동화하는 것이 일반적이다.

---

## React/Next.js 비교

| 특성 | SvelteKit | Next.js |
|---|---|---|
| **배포 추상화** | 어댑터 교체 1번으로 전환 | 기본이 Vercel, 다른 플랫폼은 별도 설정 |
| **Node 서버 배포** | `adapter-node` → `node build/` 실행 | `next start` |
| **서버리스 배포** | `adapter-vercel` 또는 `adapter-auto` | 기본 동작 (Vercel) |
| **라우트별 런타임** | `export const config` (파일 단위) | `export const runtime = 'edge'` |
| **CSRF 보호** | `ORIGIN` 환경 변수 필수 | 내장 처리 |
| **서버리스 DB** | 드라이버 교체 필요 (neon-serverless 등) | 드라이버 교체 필요 (동일) |
