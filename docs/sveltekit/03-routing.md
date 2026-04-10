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
GET /       →  +layout.svelte → children → +page.svelte (루트)
GET /about  →  +layout.svelte → children → about/+page.svelte
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
GET /about/team:

routes/+layout.svelte                    ← 1) 루트 레이아웃
  └── children →
      about/+layout.svelte               ← 2) about 레이아웃 (사이드바 등)
        └── children →
            about/team/+page.svelte      ← 3) /about/team 페이지 콘텐츠
```

**핵심**: 레이아웃은 해당 폴더와 **모든 하위 경로**에 적용된다. `page` state(`$app/state`)의 `url.pathname`으로 현재 경로를 확인하여 활성 링크를 강조할 수 있다.

> `+layout.server.ts`로 레이아웃별 데이터 로딩 → [04-loading-data.md](./04-loading-data.md) Section 2 참고

### 레이아웃 상속 체인

레이아웃은 **바깥에서 안으로** 쌓인다. 상위 레이아웃을 항상 상속한다.

| 경로 | 적용되는 레이아웃 |
|--|--|
| `/` | `routes/+layout.svelte` |
| `/about` | `routes/+layout.svelte` → `about/+layout.svelte` |
| `/about/team` | `routes/+layout.svelte` → `about/+layout.svelte` |
| `/blog` | `routes/+layout.svelte` |

---

## 4. 루트 레이아웃의 역할

루트 레이아웃(`src/routes/+layout.svelte`)은 **모든 페이지에서 렌더링**되므로:

- 글로벌 CSS / Tailwind import
- 공통 네비게이션 / 푸터
- 전역 상태 provider

---

## 5. 조건부 레이아웃 — 상속 우회

루트 레이아웃은 **모든 페이지**에 적용된다. 특정 경로에서 다른 레이아웃이 필요하면 두 가지 방법이 있다.

### 간단한 방법: `page.url.pathname`으로 조건 분기

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import '../app.css'
  import { page } from '$app/state'
  let { children } = $props()
</script>

{#if page.url.pathname.startsWith('/about')}
  {@render children()}
{:else}
  <nav>...</nav>
  {@render children()}
  <footer>...</footer>
{/if}
```

> 조건이 복잡해지면 **레이아웃 그룹**(Section 6)을 사용한다.

---

## 6. 레이아웃 그룹 — `(group)` 폴더

### 문제: 공통 경로 없이 서로 다른 레이아웃이 필요할 때

```text
마케팅 페이지  → 네비게이션 바 + 풋터       (/, /about, /blog)
인증 페이지    → 센터 카드 레이아웃          (/login, /register)
앱 페이지      → 사이드바 + 헤더            (/workspace/[id])
```

### 해결: 괄호 폴더 `(groupName)`

**괄호로 감싼 폴더**는 URL에 영향을 주지 않으면서 레이아웃을 그룹화한다.

```text
src/routes/
├── +layout.svelte                  ← 루트 레이아웃 (전체 공통)
├── (marketing)/
│   ├── +layout.svelte              ← 마케팅 전용 레이아웃 (nav 포함)
│   ├── +page.svelte                ← /
│   └── about/+page.svelte          ← /about
├── (auth)/
│   ├── +layout.svelte              ← 인증 전용 레이아웃
│   └── login/+page.svelte          ← /login
└── (app)/
    ├── +layout.svelte              ← 앱 전용 레이아웃
    └── workspace/+page.svelte      ← /workspace
```

**핵심**: `(marketing)`, `(auth)`, `(app)` 폴더명은 **URL에 포함되지 않는다**.

### 레이아웃 상속 흐름

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
| 확장성 | 조건이 늘어나면 복잡 | 그룹 추가로 깔끔하게 분리 |

---

## 7. 레이아웃 상속 리셋 — `@` 구문

중첩 레이아웃을 부분적으로 건너뛰고 싶을 때, 파일명에 `@`를 추가하고 **상속받을 레이아웃의 이름**을 지정한다.

```text
src/routes/
├── +layout.svelte                          ← 루트
└── (marketing)/
    ├── +layout.svelte                      ← 마케팅
    └── blog/
        ├── +layout.svelte                  ← 블로그
        └── [id]/
            ├── +layout.svelte              ← 블로그ID
            └── +page.svelte                ← /blog/123
```

### `@` 구문 정리

| 파일명 | 상속 체인 |
|--|--|
| `+page.svelte` | 루트 → 마케팅 → 블로그 → 블로그ID → 페이지 (기본) |
| `+page@.svelte` | 루트 → 페이지 (중간 모두 건너뜀) |
| `+page@(marketing).svelte` | 루트 → 마케팅 → 페이지 |
| `+page@blog.svelte` | 루트 → 마케팅 → 블로그 → 페이지 |
| `+layout@.svelte` | 레이아웃도 동일하게 리셋 가능 |

> `@` 뒤의 이름은 **폴더명**(그룹이면 괄호 포함)과 정확히 일치해야 한다.

---

## 8. 동적 라우트 파라미터

### 동적 세그먼트 3종 비교

```text
유형          | 패턴         | 예시 URL          | params 값
-------------|-------------|-----------------|---------------------------
동적          | [id]        | /post/123       | { id: "123" }
나머지(rest)  | [...rest]   | /a/b/c          | { rest: "a/b/c" }
옵셔널        | [[lang]]    | /about, /en/about| { lang: undefined | "en" }
```

### `[param]` — 동적 경로

대괄호 폴더명이 **동적 파라미터**가 된다. `/blog/123` 접속 시 `[id]`에 `"123"`이 바인딩.

```svelte
<script lang="ts">
  import { page } from '$app/state'
  // 템플릿에서 직접 사용: page.params.id (자동 반응)
  // 스크립트에서 변수에 담을 때: $derived 필요
  let blogId = $derived(page.params.id)
</script>
```

> 템플릿에서 `page.params.id`를 **직접 사용**하면 `$derived` 없이도 반응한다 — `page` 자체가 reactive state이므로. 스크립트에서 **변수에 담아 쓸 때만** `$derived`가 필요하다.

### `[...param]` — 레스트 파라미터

나머지 **모든 세그먼트**를 하나의 문자열 파라미터로 캡처한다.

```text
src/routes/files/[...path]/
├── +page.svelte        ← /files/a/b/c     → path = "a/b/c"
└── edit/
    └── +page.svelte    ← /files/a/b/c/edit → path = "a/b/c"
```

rest 폴더 **안에** 추가 폴더를 만들면, 해당 세그먼트는 rest에 포함되지 않는다. rest 앞에 일반 `[param]`을 배치하여 조합할 수도 있다.

### `[[param]]` — 선택적 파라미터

있어도 되고 없어도 되는 파라미터. 다국어 URL에서 유용하다.

```text
src/routes/[[lang]]/about/+page.svelte
  /about       → lang = undefined
  /en/about    → lang = "en"
  /ko/about    → lang = "ko"
```

> **제약**: rest 파라미터 뒤에 선택적 파라미터를 배치할 수 없다 — `ko`가 rest의 일부인지 선택적 파라미터인지 구분 불가.

### `route.id`로 현재 라우트 판별

load 함수에서 `route.id`를 사용하면 **현재 어떤 라우트 패턴에 있는지** 확인할 수 있다. 공유 레이아웃에서 라우트에 따라 다른 로직을 실행할 때 유용하다.

```ts
// (workspace)/+layout.server.ts
export const load: LayoutServerLoad = async ({ params, route, untrack }) => {
  // route.id = "/(app)/(workspace)/w/[wId]" 또는 "/(app)/(workspace)/p/[pId]"
  // ⚠️ route.id를 읽으면 의존성 등록 → untrack으로 불필요한 재실행 방지
  const isPage = untrack(() => route.id.startsWith('/(app)/(workspace)/p/'));
  const workspaceId = isPage
    ? await getWorkspaceIdFromPageId(params.pId!)
    : params.wId!;
};
```

> `route.id`는 파일 시스템 경로 패턴이다 — 그룹명(`(app)`)과 파라미터 자리(`[wId]`)가 그대로 포함된다. 레이아웃 하위에 탭형 라우트가 있을 때, 레이아웃 데이터가 탭과 무관하다면 `untrack`으로 불필요한 재실행을 방지한다.

---

## 9. 파라미터 매처 — `[param=matcher]`

동적 파라미터가 너무 많은 것을 받아들이는 것을 방지한다. `src/params/` 폴더에 매처 파일을 생성하고, 폴더명에서 `=`으로 연결한다.

```ts
// src/params/authType.ts
import type { ParamMatcher } from '@sveltejs/kit'

export const match: ParamMatcher = (param): param is 'login' | 'register' => {
  return param === 'login' || param === 'register'
}
```

```text
src/routes/(auth)/[authType=authType]/+page.svelte

/login     → 매칭 (authType = "login")
/register  → 매칭 (authType = "register")
/anything  → 404 (매처가 false 반환)
```

### 경로 우선순위

같은 URL에 매칭 가능한 경로가 여러 개 있을 때:

```text
우선순위 (높음 → 낮음):
1. 리터럴 경로        /foo-abc           (정확히 일치)
2. 매처 있는 파라미터  [param=matcher]    (제한된 동적)
3. 일반 파라미터       [param]            (무제한 동적)
4. 선택적 파라미터     [[param]]
5. rest 파라미터       [...rest]          (가장 느슨함)
```

---

## 10. Shallow Routing

URL 변경 없이 히스토리 상태만 관리할 때 사용한다. 모달, 필터, 갤러리 등 전체 네비게이션 없이 뒤로가기/앞으로가기를 지원해야 하는 경우에 유용하다.

```ts
import { pushState, replaceState } from '$app/navigation'
import { page } from '$app/state'

// URL 변경 없이 상태만 히스토리에 푸시
pushState('', { showModal: true })

// 현재 히스토리 엔트리를 교체 (새 엔트리 안 만듦)
replaceState('', { filter: 'active' })
```

```svelte
{#if page.state.showModal}
  <Modal onclose={() => history.back()} />
{/if}
```

- SSR 시 `page.state`는 항상 빈 객체. 페이지 새로고침 시에도 state가 복원되지 않으므로, JS가 필수인 기능에만 사용한다.

### 실전 패턴: 라우트를 모달로 열기

`preloadData` + `pushState` 조합으로 URL은 변경하되 실제 내비게이션 없이 모달에 라우트를 렌더링할 수 있다. JS 비활성화 시에는 일반 페이지로 이동한다.

```text
[동작 흐름]

1. App.PageState 타입 정의 (app.d.ts)
2. 링크 클릭 시 preloadData(href) → pushState(href, { data })
3. 레이아웃에서 page.state 확인 → 모달 렌더링
4. 뒤로가기 → state 사라짐 → 모달 닫힘

[JS 비활성화 시]
일반 <a> 동작 → 전체 페이지로 이동
```

핵심 코드:

```svelte
<script lang="ts">
  import { pushState, preloadData, goto } from '$app/navigation'
  import { page } from '$app/state'

  async function openInModal(e: MouseEvent) {
    if (e.metaKey || e.ctrlKey || e.shiftKey) return
    e.preventDefault()

    const href = (e.currentTarget as HTMLAnchorElement).href
    const result = await preloadData(href)

    if (result.type === 'loaded' && result.status === 200) {
      pushState(href, { addWorkspaceData: result.data })
    } else {
      goto(href)
    }
  }
</script>

<a href="/new" onclick={openInModal}>New Workspace</a>
```

> `app.d.ts`의 `App.PageState`에 state 타입을 정의해야 TypeScript 에러가 없다. 모달 내에서 페이지 컴포넌트를 렌더링할 때는 `form={null}`을 전달한다.

---

## 11. 링크 프리로딩

SvelteKit은 `data-sveltekit-*` 속성으로 링크의 프리로딩 동작을 제어한다.

```svelte
<!-- 호버 시 데이터 프리로드 (기본값) -->
<a href="/blog" data-sveltekit-preload-data="hover">Blog</a>

<!-- 탭/클릭 시점에 프리로드 -->
<a href="/blog" data-sveltekit-preload-data="tap">Blog</a>

<!-- 코드만 프리로드 (데이터 제외) -->
<a href="/blog" data-sveltekit-preload-code="hover">Blog</a>

<!-- 뷰포트 진입 시 코드 즉시 프리로드 -->
<a href="/blog" data-sveltekit-preload-code="eager">Blog</a>
```

- `data-sveltekit-preload-data`: 링크의 `load` 함수 데이터를 미리 가져옴. 값은 `"hover"` | `"tap"` | `"false"`.
- `data-sveltekit-preload-code`: JS 코드만 미리 로드. 값은 `"eager"` | `"viewport"` | `"hover"` | `"tap"`.
- 부모 요소에 설정하면 하위 모든 `<a>`에 일괄 적용됨.
- 기본 프로젝트 템플릿은 `<body data-sveltekit-preload-data="hover">`로 설정되어 있다.

---

## 12. 경로 폴더 커스터마이징

`svelte.config.js`에서 `routes` 디렉토리 경로를 변경할 수 있다:

```js
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

## React/Next.js 비교

| 개념 | Next.js (App Router) | SvelteKit |
|--|--|--|
| 라우팅 방식 | `app/` 폴더 기반 | `src/routes/` 폴더 기반 |
| 페이지 파일 | `page.tsx` | `+page.svelte` |
| 레이아웃 파일 | `layout.tsx` | `+layout.svelte` |
| 자식 렌더링 | `{children}` (React prop) | `{@render children()}` (Svelte snippet) |
| 레이아웃 상속 리셋 | 없음 (Route Groups + Template 조합) | `@` 구문으로 유연하게 리셋 |
| 동적 경로 | `[id]/page.tsx` | `[id]/+page.svelte` |
| 파라미터 접근 | `useParams()` 또는 `params` prop | `page.params.id` (reactive state) |
| Rest 파라미터 결과 | `string[]` (배열) | `string` (슬래시 구분 문자열) |
| 선택적 단일 파라미터 | 공식 지원 없음 | `[[param]]` |
| 파라미터 매처 | 없음 (`middleware.ts`에서 검증) | `[param=matcher]` (라우팅 레벨 검증) |
| 모달 라우트 | Intercepting Routes (`(.)route`) 내장 | `pushState` + `preloadData` 수동 조합 |
| 프리로딩 | Router prefetch (자동) | `data-sveltekit-preload-*` 속성으로 세밀 제어 |
