# 라우팅 & 레이아웃

SvelteKit의 파일 기반 라우팅: 폴더 = 경로, `+page.svelte` = 페이지, `+layout.svelte` = 레이아웃.

---

## 1. 기본 라우팅 규칙

`src/routes/` 안의 **폴더 구조가 곧 URL 경로**다.

```text
src/routes/
├── +layout.svelte        ← 루트 레이아웃 (모든 페이지에 적용)
├── +page.svelte          ← /
├── about/
│   └── +page.svelte      ← /about
├── blog/
│   └── +page.svelte      ← /blog
└── contact/
    └── +page.svelte      ← /contact
```

**규칙**: 경로를 만들려면 폴더를 만들고, 그 안에 `+page.svelte`를 넣는다.

---

## 2. 레이아웃 동작 원리

### 루트 레이아웃

`src/routes/+layout.svelte`는 **모든 페이지**에 적용된다.

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import '../app.css'
  let { children } = $props()
</script>

<nav><!-- 네비게이션 --></nav>
{@render children()}
```

- `children`은 현재 요청한 경로의 페이지 콘텐츠를 렌더링하는 snippet
- 글로벌 CSS, 공통 네비게이션 등을 여기서 import

### 렌더링 흐름

```text
GET /  요청 시:

+layout.svelte          ← 항상 렌더링
  └── children → +page.svelte (루트)    ← / 의 콘텐츠
```

```text
GET /about  요청 시:

+layout.svelte          ← 항상 렌더링
  └── children → about/+page.svelte    ← /about 의 콘텐츠
```

루트 레이아웃은 변하지 않고, `children`이 경로에 따라 교체된다.

---

## 3. 중첩 레이아웃

하위 경로에 **전용 레이아웃**이 필요하면, 해당 폴더에 `+layout.svelte`를 추가한다.

```text
src/routes/
├── +layout.svelte              ← 루트 레이아웃 (전체)
├── +page.svelte                ← /
└── about/
    ├── +layout.svelte          ← /about 전용 레이아웃
    ├── +page.svelte            ← /about
    └── team/
        └── +page.svelte        ← /about/team
```

### 중첩 렌더링 흐름

```text
GET /about  요청 시:

routes/+layout.svelte                    ← 1) 루트 레이아웃
  └── children →
      about/+layout.svelte               ← 2) about 레이아웃 (사이드바 등)
        └── children →
            about/+page.svelte           ← 3) /about 페이지 콘텐츠
```

```text
GET /about/team  요청 시:

routes/+layout.svelte                    ← 1) 루트 레이아웃
  └── children →
      about/+layout.svelte               ← 2) about 레이아웃 (동일)
        └── children →
            about/team/+page.svelte      ← 3) /about/team 페이지 콘텐츠
```

**핵심**: 레이아웃은 해당 폴더와 **모든 하위 경로**에 적용된다.

> `+layout.server.ts`로 레이아웃별 데이터 로딩 → [04-loading-data.md](./04-loading-data.md) Section 2 참고

### 실전 예시: 사이드바 레이아웃

```svelte
<!-- src/routes/about/+layout.svelte (about 전용 레이아웃) -->
<script>
  import { page } from '$app/state'
  let { children } = $props()
</script>

<div class="about-layout">
  <h1>About Layout</h1>
  <div class="flex gap-4">
    <!-- 사이드바 -->
    <div class="sidebar">
      <nav>
        <ul>
          <li>
            <a href="/about" class:underline={page.url.pathname === '/about'}>
              About
            </a>
          </li>
          <li>
            <a href="/about/team" class:underline={page.url.pathname === '/about/team'}>
              Team
            </a>
          </li>
        </ul>
      </nav>
    </div>
    <!-- 콘텐츠 -->
    <div class="content">
      {@render children()}
    </div>
  </div>
</div>
```

### 활성 링크 강조: `page.url.pathname`

`page` state의 `url.pathname`으로 현재 경로를 확인하고, `class:` 디렉티브로 조건부 스타일 적용:

```svelte
<a href="/about" class:underline={page.url.pathname === '/about'}>
  About
</a>
```

> `page`는 `$app/state`에서 import. URL, params, status 등 현재 페이지 정보를 담고 있다.

---

## 4. 레이아웃 상속 체인

레이아웃은 **바깥에서 안으로** 쌓인다. 상위 레이아웃을 항상 상속한다.

```text
루트 레이아웃 → about 레이아웃 → 페이지
(nav, footer)    (sidebar)        (콘텐츠)
```

| 경로 | 적용되는 레이아웃 |
|--|--|
| `/` | `routes/+layout.svelte` |
| `/about` | `routes/+layout.svelte` → `about/+layout.svelte` |
| `/about/team` | `routes/+layout.svelte` → `about/+layout.svelte` |
| `/blog` | `routes/+layout.svelte` |

---

## 5. 루트 레이아웃의 역할

루트 레이아웃(`src/routes/+layout.svelte`)은 **모든 페이지에서 렌더링**되므로:

- 글로벌 CSS import (`import '../app.css'`)
- Tailwind 스타일 import
- 공통 네비게이션 / 푸터
- 전역 상태 provider

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import '../app.css'  // ← 모든 페이지에 적용 보장
  let { children } = $props()
</script>

<nav>...</nav>
{@render children()}
<footer>...</footer>
```

---

## 6. 조건부 레이아웃 — 상속 우회

### 문제: 특정 경로에서 루트 ��이아웃이 필요 없을 때

루트 레이아웃은 **모든 페이지**에 적용된다. 하지만 `/about` 이하 경로에서는 자체 레이아웃만 쓰고 루트 ���이아웃의 네비게이션이나 배경이 필요 없을 수 있다.

### 간단한 해결: `page.url.pathname`으로 조건 분기

루트 레이아웃을 완전히 건너뛸 수는 없지만, 경로에 따라 **렌더링 내용을 바꿀 수** 있다.

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import '../app.css'
  import { page } from '$app/state'
  let { children } = $props()
</script>

{#if page.url.pathname.startsWith('/about')}
  <!-- /about 이하: 루트 레이아웃 장식 없이 children만 렌더링 -->
  {@render children()}
{:else}
  <!-- 그 외: 네비게이션 + 배경 포함 -->
  <div class="border-2 bg-pink-900 p-2">
    <h1>Root Layout</h1>
    <nav class="flex gap-4 p-4">
      <a href="/">Home</a>
      <a href="/about">About</a>
    </nav>
    {@render children()}
  </div>
{/if}
```

> 조건이 복잡해지면 **레이아웃 그룹**(Section 7)을 사용한다.

---

## 7. 레이아웃 그룹 — `(group)` 폴더

### 문제: 공통 경로 없이 서로 다른 레이아웃이 필요할 때

실제 앱에서는 완전히 다른 레이아웃이 필요한 페이지 그룹이 존재한다:

```text
마케팅 페이지  → 네비게이션 바 + 풋터       (/, /about, /blog)
인증 페이지    → 센터 카드 레이아웃          (/login, /register)
앱 페이지      → 사이드바 + 헤더            (/workspace/[id])
```

이들은 URL에서 공통 접두사(`/marketing/...`)를 공유하지 않으므로, 일반 폴더 중첩으로는 레이아웃을 분리할 수 없다.

### 해결: 괄호 폴더 `(groupName)`

**괄호로 감싼 폴더**는 URL에 영향을 주지 않으면서 레이아웃을 그룹화한다.

```text
src/routes/
├── +layout.svelte                  ← 루트 레이아웃 (전체 공통)
├── (marketing)/
│   ├── +layout.svelte              ← 마케팅 전용 레이아웃 (nav 포함)
│   ├── +page.svelte                ← /
│   ├── about/
│   │   └── +page.svelte            ← /about
│   └── blog/
│       └── +page.svelte            ← /blog
├── (auth)/
│   ├── +layout.svelte              ← 인증 전용 레이아웃
│   └── login/
│       └── +page.svelte            ← /login
└── (app)/
    ├── +layout.svelte              ← 앱 전용 레이아웃
    └── workspace/
        └── +page.svelte            ← /workspace
```

**핵심**: `(marketing)`, `(auth)`, `(app)` 폴더명은 **URL에 포함되지 않는다**.

### 각 그룹별 레이아웃 예시

```svelte
<!-- src/routes/(marketing)/+layout.svelte -->
<script>
  let { children } = $props()
</script>

<div class="border-2 bg-green-900 p-2">
  <h1>Marketing Group Layout</h1>
  <nav class="flex gap-4 p-4">
    <a href="/">Home</a>
    <a href="/about">About</a>
    <a href="/blog">Blog</a>
    <a href="/login">Login</a>
  </nav>
  {@render children()}
</div>
```

```svelte
<!-- src/routes/(auth)/+layout.svelte -->
<script>
  let { children } = $props()
</script>

<div class="border-2 bg-yellow-900 p-2">
  <h1>Auth Layout</h1>
  <!-- 네비게이션 없음 — 센터 카드 등 인증 전용 UI -->
  {@render children()}
</div>
```

### 레이아웃 상속 흐름

```text
GET /  요청 시:
  routes/+layout.svelte  →  (marketing)/+layout.svelte  →  (marketing)/+page.svelte

GET /login  요청 시:
  routes/+layout.svelte  →  (auth)/+layout.svelte  →  (auth)/login/+page.svelte
```

| 경로 | 적용 레이아웃 |
|--|--|
| `/` | root → (marketing) |
| `/about` | root → (marketing) |
| `/login` | root → (auth) |
| `/workspace` | root → (app) |

### 조건부 레이아웃 vs 레이아웃 그룹

| | 조건부 (`if` 분기) | 레이아웃 그룹 (`(group)`) |
|--|--|--|
| 적합한 경우 | 간단한 차이 (nav 숨기기 등) | 완전히 다른 레이아웃 |
| URL 영향 | 없음 | 없음 |
| 루트 레이아웃 | 항상 마운트 (조건부 렌더링) | 항상 마운트 (공통 부분만) |
| 확장성 | 조건이 늘어나면 복잡 | 그룹 추가로 깔끔하게 분리 |

---

## 8. 레이아웃 상속 리셋 — `@` 구문

### 문제: 중첩 레이아웃을 부분적으로 건너뛰고 싶을 때

```text
src/routes/
├── +layout.svelte                          ← 루트 (항상 적용)
└── (marketing)/
    ├── +layout.svelte                      ← 마케팅 레이아웃
    └── blog/
        ├── +layout.svelte                  ← 블로그 레이아웃
        ├── +page.svelte                    ← /blog
        └── [id]/
            ├── +layout.svelte              ← 블로그 ID 레이아웃
            └── +page.svelte                ← /blog/123
```

기본 상속: `/blog/123` → 루트 → 마케팅 → 블로그 → 블로그ID → 페이지

단일 블로그 게시물 페이지에서 중간 레이아웃을 건너뛰고 싶다면?

### 해결: `+page@.svelte` / `+layout@.svelte`

파일명 끝에 `@`를 추가하고, **상속받을 레이아웃의 이름**을 지정한다.

### 모든 레이아웃 건너뛰기 (루트만 상속)

```text
blog/[id]/+page@.svelte       ← @ 뒤에 아무것도 없음 = 루트 레이아웃만
```

```text
상속 체인: 루트 → 페이지  (마케팅, 블로그, 블로그ID 모두 건너뜀)
```

### 특정 레이아웃까지만 상속

```text
blog/[id]/+page@(marketing).svelte    ← 마케팅 레이아웃까지만 상속
```

```text
상속 체인: 루트 → 마케팅 → 페이지  (블로그, 블로그ID 건너뜀)
```

```text
blog/[id]/+page@blog.svelte           ← 블로그 레이아웃까지 상속
```

```text
상속 체인: 루트 → 마케팅 → 블로그 → 페이지  (블로그ID만 건너뜀)
```

### 레이아웃 파일에도 적용 가능

`+layout` 파일도 같은 방식으로 상속을 리셋할 수 있다.

```text
blog/+layout@.svelte           ← 블로그 레이아웃이 루트에서만 상속
```

```text
기본: 루트 → 마케팅 → 블로그 레이아웃
리셋: 루트 → 블로그 레이아웃  (마케팅 건너뜀)
```

### 상속 체인 — before / after 비교

```text
[기본: +page.svelte]
루트 → (marketing) → blog → [id] → 페이지
(모든 레이아웃 상속)

[+page@.svelte — 루트만]
루트 → 페이지
(marketing, blog, [id] 건너뜀)

[+page@(marketing).svelte — marketing까지]
루트 → (marketing) → 페이지
(blog, [id] 건너뜀)

[+page@blog.svelte — blog까지]
루트 → (marketing) → blog → 페이지
([id] 건너뜀)
```

### `@` 구문 정리

| 파일명 | 상속 대상 |
|--|--|
| `+page.svelte` | 기본 (모든 상위 레이아웃) |
| `+page@.svelte` | 루트 레이아웃만 |
| `+page@(marketing).svelte` | `(marketing)` 그룹 레이아웃까지 |
| `+page@blog.svelte` | `blog` 폴더 레이아웃까지 |
| `+layout@.svelte` | 레이아웃도 동일하게 리셋 가능 |

> `@` 뒤의 이름은 **폴더명**(그룹이면 괄호 포함)과 정확히 일치해야 한다.

### React/Next.js 비교

Next.js App Router에는 이에 대응하는 기능이 없다. Route Groups와 Template 조합으로 유사하게 구현해야 하지만, SvelteKit의 `@` 구문만큼 유연하지 않다.

---

---

## 9. 동적 라우트 파라미터

### `[param]` 폴더로 동적 경로 생성

대괄호로 감싼 폴더명이 **동적 파라미터**가 된다.

```text
src/routes/
└── blog/
    ├── +page.svelte          ← /blog
    └── [id]/
        └── +page.svelte      ← /blog/1, /blog/2, /blog/abc ...
```

`/blog/123` 접속 시 `[id]`에 `"123"`이 바인딩된다.

### `page` state로 파라미터 읽기

`$app/state`의 `page` 객체에서 현재 경로의 파라미터에 접근한다.

```svelte
<!-- src/routes/blog/[id]/+page.svelte -->
<script lang="ts">
  import { page } from '$app/state'
</script>

<h1>Blog Post {page.params.id}</h1>

<nav>
  <a href="/blog/1">Post 1</a>
  <a href="/blog/2">Post 2</a>
  <a href="/blog/3">Post 3</a>
</nav>
```

`page.params.id`가 URL 변경에 따라 자동으로 반응한다.

### 주의: 스크립트에서 파라미터를 변수에 할당할 때

`<script>`는 컴포넌트 마운트 시 **한 번만 실행**된다. 같은 컴포넌트를 쓰는 경로 간 이동 시 컴포넌트가 재마운트되지 않으므로, 일반 변수는 업데이트되지 않는다.

```svelte
<script lang="ts">
  import { page } from '$app/state'

  // ❌ 마운트 시 한 번만 평가 — /blog/1 → /blog/2 이동 시 갱신 안 됨
  let blogId = page.params.id
</script>

<h1>Blog {blogId}</h1>  <!-- 항상 최초 값만 표시 -->
```

**해결: `$derived` 사용**

```svelte
<script lang="ts">
  import { page } from '$app/state'

  // ✅ page.params.id가 바뀔 때마다 재평가
  let blogId = $derived(page.params.id)
</script>

<h1>Blog {blogId}</h1>  <!-- URL 변경에 따라 업데이트 -->
```

> 템플릿에서 `page.params.id`를 **직접 사용**하면 `$derived` 없이도 반응한다 — `page` 자체가 reactive state이므로. 스크립트에서 **변수에 담아 쓸 때만** `$derived`가 필요하다.

### React 비교

| | Next.js | SvelteKit |
|--|--|--|
| 동적 경로 | `app/blog/[id]/page.tsx` | `routes/blog/[id]/+page.svelte` |
| 파라미터 접근 | `useParams()` 또는 `params` prop | `page.params.id` |
| 반응성 | Hook이 자동 리렌더 | `$derived` 또는 템플릿 직접 참조 |

### `route.id`로 현재 라우트 판별

load 함수에서 `route.id`를 사용하면 **현재 어떤 라우트 패턴에 있는지** 확인할 수 있다. 공유 레이아웃에서 라우트에 따라 다른 로직을 실행할 때 유용하다.

```text
src/routes/(app)/(workspace)/
├── +layout.server.ts          ← 공유 레이아웃 (사이드바)
├── w/[wId]/+page.svelte       ← /w/123  (워크스페이스)
└── p/[pId]/+page.svelte       ← /p/456  (페이지)
```

```ts
// (workspace)/+layout.server.ts
export const load: LayoutServerLoad = async ({ params, route }) => {
  // route.id = "/(app)/(workspace)/w/[wId]" 또는 "/(app)/(workspace)/p/[pId]"
  const isPage = route.id.startsWith('/(app)/(workspace)/p/');

  // 라우트에 따라 다른 파라미터에서 워크스페이스 ID 추출
  const workspaceId = isPage
    ? await getWorkspaceIdFromPageId(params.pId!)  // 페이지 → DB 조회 필요
    : params.wId!;                                   // 워크스페이스 → 직접 사용

  // 이하 공통 로직...
};
```

> `route.id`는 파일 시스템 경로 패턴이다 — 그룹명(`(app)`)과 파라미터 자리(`[wId]`)가 그대로 포함된다. 실제 URL이 아닌 **라우트 정의**를 반환한다는 점에 주의.

### ⚠️ `route.id`는 자동 의존성이 된다 — `untrack` 활용

`route.id`를 load 함수에서 읽으면 **의존성으로 등록**되어, 같은 레이아웃 내에서 하위 라우트를 전환할 때마다 load가 불필요하게 재실행된다.

```ts
// ❌ 탭 전환(pages → members → settings)마다 레이아웃 load 재실행
export const load: LayoutServerLoad = async ({ params, route }) => {
  const isPage = route.id.startsWith('/(app)/(workspace)/p/');  // route.id 의존성 등록!
  const workspaceId = isPage ? await getWorkspaceIdFromPageId(params.pId!) : params.wId!;
  // ...DB 쿼리 (불필요하게 반복 실행)
};
```

```ts
// ✅ untrack으로 route.id 의존성 제거 — 탭 전환 시 재실행 안 됨
export const load: LayoutServerLoad = async ({ params, route, untrack }) => {
  const isPage = untrack(() => route.id.startsWith('/(app)/(workspace)/p/'));
  const workspaceId = isPage ? await getWorkspaceIdFromPageId(params.pId!) : params.wId!;
  // ...DB 쿼리 (최초 1회만 실행)
};
```

> 레이아웃 하위에 탭형 라우트(`/settings`, `/members` 등)가 있을 때, 레이아웃 데이터가 탭과 무관하다면 `untrack`으로 불필요한 재실행을 방지한다.

---

## 10. 레스트 파라미터 — `[...param]`

### 세그먼트 수를 알 수 없는 경로

파일 경로처럼 `/files/a/b/c` 깊이가 가변적인 URL을 처리해야 할 때, 단일 `[id]`로는 첫 세그먼트만 캡처된다.

**`[...param]`** (rest 파라미터)은 나머지 **모든 세그먼트**를 하나의 파라미터로 캡처한다.

```text
src/routes/(marketing)/files/
└── [...path]/
    ├── +page.svelte            ← /files/a/b/c
    └── edit/
        └── +page.svelte        ← /files/a/b/c/edit
```

| URL | `page.params.path` |
|--|--|
| `/files/a` | `"a"` |
| `/files/a/b/c` | `"a/b/c"` |
| `/files/a/b/c/edit` | `"a/b/c"` (edit는 별도 경로) |

### 기본 사용법

```svelte
<!-- src/routes/(marketing)/files/[...path]/+page.svelte -->
<script lang="ts">
  import { page } from '$app/state'
</script>

<h1>File: {page.params.path}</h1>
<pre>{JSON.stringify(page.params, null, 2)}</pre>
```

### rest 파라미터 뒤에 하위 경로 추가

rest 폴더 **안에** 추가 폴더를 만들면, 해당 세그먼트는 rest에 포함되지 않는다.

```text
[...path]/
├── +page.svelte        ← /files/a/b/c     → path = "a/b/c"
└── edit/
    └── +page.svelte    ← /files/a/b/c/edit → path = "a/b/c"
```

`edit`은 별도 경로이므로 `path`에 포함되지 않는다.

### rest + 일반 동적 파라미터 조합

rest 앞에 일반 `[param]`을 배치할 수 있다.

```text
src/routes/(marketing)/files/
└── [id]/
    └── [...path]/
        ├── +page.svelte        ← /files/123/a/b/c
        └── edit/
            └── +page.svelte    ← /files/123/a/b/c/edit
```

| URL | `params.id` | `params.path` |
|--|--|--|
| `/files/123/a/b` | `"123"` | `"a/b"` |
| `/files/123/a/b/edit` | `"123"` | `"a/b"` |

```svelte
<script lang="ts">
  import { page } from '$app/state'
</script>

<h1>File #{page.params.id}</h1>
<p>Path: {page.params.path}</p>
```

### React/Next.js 비교

| | Next.js | SvelteKit |
|--|--|--|
| Rest 파라미터 | `app/files/[...path]/page.tsx` | `routes/files/[...path]/+page.svelte` |
| 선택적 catch-all | `[[...path]]` (빈 경로도 매칭) | `[[...path]]` (동일 구문) |
| 결과 형태 | `params.path: string[]` (배열) | `params.path: string` (슬래시 구분 문자열) |

> **주의**: Next.js는 배열(`["a","b","c"]`)로 반환하지만, SvelteKit은 문자열(`"a/b/c"`)로 반환한다. 배열이 필요하면 `params.path.split('/')`을 사용.

---

## 11. 선택적 파라미터 — `[[param]]`

### 이중 대괄호 = 있어도 되고 없어도 되는 파라미터

다국어 앱에서 URL에 언어 코드를 선택적으로 포함하는 경우:

```text
/about          ← 기본 언어 (lang 없음)
/en/about       ← 영어
/ko/about       ← 한국어
```

### 사용법

폴더명에 **이중 대괄호** `[[param]]`을 사용한다.

```text
src/routes/(marketing)/
└── [[lang]]/
    ├── +page.svelte            ← / 또는 /en
    ├── about/
    │   └── +page.svelte        ← /about 또는 /en/about
    └── blog/
        └── +page.svelte        ← /blog 또는 /ko/blog
```

| URL | `page.params.lang` |
|--|--|
| `/` | `undefined` |
| `/about` | `undefined` |
| `/en` | `"en"` |
| `/en/about` | `"en"` |
| `/ko/blog` | `"ko"` |

### `[param]` vs `[[param]]` 비교

| | `[lang]` (필수) | `[[lang]]` (선택적) |
|--|--|--|
| `/about` | 404 | 정상 (`lang` = undefined) |
| `/en/about` | 정상 (`lang` = `"en"`) | 정상 (`lang` = `"en"`) |

### 제약: rest 파라미터 뒤에 선택적 파라미터 불가

```text
[...path]/[[lang]]/     ← 작동하지 않음
```

`/files/a/b/c/ko`에서 `ko`가 rest의 일부인지 선택적 파라미터인지 구분할 수 없기 때문이다. 항상 rest 파라미터에 흡수된다.

### React/Next.js 비교

| | Next.js | SvelteKit |
|--|--|--|
| 선택적 단일 파라미터 | 공식 지원 없음 | `[[param]]`으로 지원 |
| 선택적 catch-all | `[[...slug]]` | `[[...rest]]` |

Next.js에서 `/about`과 `/en/about`을 하나의 라우트로 처리하려면 `[[...slug]]`로 감싸고 내부에서 분기해야 한다. SvelteKit은 `[[lang]]` 하나로 단일 세그먼트만 선택적으로 만들 수 있다.

---

## 12. 파라미터 매처 — `[param=matcher]`

### 문제: 동적 파라라미터가 너무 많은 것을 받아들일 때

`(auth)/[authType]/+page.svelte`는 `/login`, `/register`뿐 아니라 `/anything`도 매칭한다. 허용되지 않는 경로에 대해 404를 반환해야 한다.

### 매처 함수 정의

`src/params/` 폴더에 매처 파일을 생성한다. **파일명이 매처 이름**이 된다.

```ts
// src/params/authType.ts
import type { ParamMatcher } from '@sveltejs/kit'

export const match: ParamMatcher = (param): param is 'login' | 'register' => {
  return param === 'login' || param === 'register'
}
```

- 함수명은 반드시 `match`
- `param`을 받아 `boolean` 반환 — `true`면 경로 매칭, `false`면 매칭 실패 (404)
- `param is 'login' | 'register'`는 TypeScript 타입 술어(type predicate) — 타입을 좁혀줌

### 폴더에 매처 연결

폴더명에서 `=`으로 매처를 지정한다.

```text
src/routes/(auth)/
└── [authType=authType]/        ← 파라미터명=매처파일명
    └── +page.svelte
```

| URL | 매칭 결과 |
|--|--|
| `/login` | 매칭 (`authType` = `"login"`) |
| `/register` | 매칭 (`authType` = `"register"`) |
| `/anything` | 404 (매처가 `false` 반환) |

### 페이지에서 활용

```svelte
<!-- src/routes/(auth)/[authType=authType]/+page.svelte -->
<script lang="ts">
  import { page } from '$app/state'
</script>

{#if page.params.authType === 'register'}
  <h1>Sign Up</h1>
  <!-- 추가 입력 필드 -->
{:else}
  <h1>Login</h1>
{/if}
```

하나의 컴포넌트에서 파라미터에 따라 분기하여 중복을 줄인다.

### 다른 매처 예시: 숫자만 허용

```ts
// src/params/integer.ts
import type { ParamMatcher } from '@sveltejs/kit'

export const match: ParamMatcher = (param) => {
  return /^\d+$/.test(param)
}
```

```text
blog/[id=integer]/+page.svelte    ← /blog/123 매칭, /blog/abc 404
```

### 경로 우선순위

같은 URL에 매칭 가능한 경로가 여러 개 있을 때, SvelteKit은 **구체적인 것 우선**으로 결정한다:

```text
우선순위 (높음 → 낮음):
1. 리터럴 경로        /foo-abc           (정확히 일치)
2. 매처 있는 파라미터  [param=matcher]    (제한된 동적)
3. 일반 파라미터       [param]            (무제한 동적)
4. 선택적 파라미터     [[param]]
5. rest 파라미터       [...rest]          (가장 느슨함)
```

### React/Next.js 비교

Next.js에는 파라미터 매처에 대응하는 기본 기능이 없다. `middleware.ts`에서 URL을 검사하거나 `generateStaticParams`로 유효 값을 열거해야 한다. SvelteKit의 매처는 라우팅 레벨에서 바로 검증하므로 더 깔끔하다.

---

## 13. Shallow Routing

URL 변경 없이 히스토리 상태만 관리할 때 사용한다. 모달, 필터, 갤러리 등 전체 네비게이션 없이 뒤로가기/앞으로가기를 지원해야 하는 경우에 유용하다.

```ts
import { pushState } from '$app/navigation'
import { page } from '$app/state'

// URL 변경 없이 상태만 히스토리에 푸시
function openModal() {
  pushState('', { showModal: true })
}

// replaceState: 현재 히스토리 엔트리를 교체
import { replaceState } from '$app/navigation'
replaceState('', { filter: 'active' })
```

```svelte
<!-- 페이지에서 state 읽기 -->
{#if page.state.showModal}
  <Modal onclose={() => history.back()} />
{/if}
```

- `pushState(url, state)`: 새 히스토리 엔트리 추가. url이 빈 문자열이면 현재 URL 유지.
- `replaceState(url, state)`: 새 엔트리를 만들지 않고 현재 히스토리 엔트리의 state를 교체.
- 뒤로가기 시 이전 state로 복원됨.
- SSR 시 `page.state`는 항상 빈 객체. 페이지 새로고침 시에도 state가 복원되지 않으므로, JS가 필수인 기능에만 사용한다.

---

## 14. 링크 프리로딩

SvelteKit은 `data-sveltekit-*` 속성으로 링크의 프리로딩 동작을 제어한다.

```svelte
<!-- 호버 시 데이터 프리로드 (기본값) -->
<a href="/blog" data-sveltekit-preload-data="hover">Blog</a>

<!-- 탭/클릭 시점에 프리로드 -->
<a href="/blog" data-sveltekit-preload-data="tap">Blog</a>

<!-- 프리로드 비활성화 -->
<a href="/blog" data-sveltekit-preload-data="false">Blog</a>

<!-- 코드만 프리로드 (데이터 제외) -->
<a href="/blog" data-sveltekit-preload-code="hover">Blog</a>

<!-- 뷰포트 진입 시 코드 즉시 프리로드 -->
<a href="/blog" data-sveltekit-preload-code="eager">Blog</a>
```

- `data-sveltekit-preload-data`: 링크의 `load` 함수 데이터를 미리 가져옴. 값은 `"hover"` | `"tap"` | `"false"`.
- `data-sveltekit-preload-code`: JS 코드만 미리 로드 (데이터는 네비게이션 시). 값은 `"eager"` | `"viewport"` | `"hover"` | `"tap"`.
- 부모 요소에 설정하면 하위 모든 `<a>`에 일괄 적용됨.
- 기본 프로젝트 템플릿은 `<body data-sveltekit-preload-data="hover">`로 설정되어 있다.
- 사용자가 데이터 절약 모드(`navigator.connection.saveData`)를 사용 중이면 프리로드가 무시된다.

---

## 15. 경로 폴더 커스터마이징

`svelte.config.js`에서 `routes` 디렉토리 경로를 변경할 수 있다:

```js
// svelte.config.js
const config = {
  kit: {
    files: {
      routes: 'src/pages'  // 기본값: 'src/routes'
    }
  }
}
```

> 일반적으로 기본값(`src/routes`)을 유지하는 것을 권장.

---

## 16. React/Next.js 비교

| 개념 | Next.js (App Router) | SvelteKit |
|--|--|--|
| 라우팅 방식 | `app/` 폴더 기반 | `src/routes/` 폴더 기반 |
| 페이지 파일 | `page.tsx` | `+page.svelte` |
| 레이아웃 파일 | `layout.tsx` | `+layout.svelte` |
| 자식 렌더링 | `{children}` (React prop) | `{@render children()}` (Svelte snippet) |
| 레이아웃 상속 | 자동 중첩 | 자동 중첩 (동일) |
| 글로벌 스타일 | Root Layout에서 import | 루트 `+layout.svelte`에서 import |
| 경로 생성 | 폴더 생성 | 폴더 생성 (동일) |
