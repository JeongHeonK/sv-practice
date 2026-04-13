# Tailwind CSS v4

React 경험자 + Tailwind v3 사용자를 위한 v4 전환 가이드. CSS-first 접근법과 SvelteKit 통합 방법을 다룬다.

---

## 1. v4의 핵심 변화 — CSS-first 접근

v4의 가장 큰 변화는 설정 방식이다. `tailwind.config.js`가 사라지고 CSS 파일 하나로 모든 것을 관리한다.

```text
v3 방식                         v4 방식
────────────────────────────    ────────────────────────────
tailwind.config.js              app.css
  └─ theme.extend.colors        └─ @import "tailwindcss"
  └─ content: [...]             └─ @theme { --color-... }
  └─ plugins: [...]
```

**진입점 변화**

```css
/* v3 */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* v4 */
@import "tailwindcss";
```

한 줄로 base, components, utilities가 모두 포함된다.

**content 배열 제거**

v3에서 명시해야 했던 `content` 경로 설정이 v4에서는 자동 감지된다.
Vite 플러그인 사용 시 모듈 그래프를 기반으로 실제 사용 파일만 정확하게 추적한다.

---

## 2. 테마 커스터마이징 — `@theme` 블록

`theme.extend`를 CSS 변수로 대체한다. `@theme` 블록에 선언한 변수는 Tailwind가 유틸리티 클래스로 변환한다.

```css
@import "tailwindcss";

@theme {
  /* 색상 */
  --color-primary: oklch(0.84 0.18 117.33);
  --color-surface: oklch(0.98 0 0);

  /* 폰트 */
  --font-sans: "Inter", sans-serif;
  --font-display: "Satoshi", sans-serif;

  /* 브레이크포인트 */
  --breakpoint-3xl: 1920px;

  /* 이징 */
  --ease-fluid: cubic-bezier(0.3, 0, 0, 1);
}
```

`--color-primary` 선언 하나로 `bg-primary`, `text-primary`, `border-primary` 등 관련 유틸리티가 자동 생성된다.

```text
@theme 변수 선언
  └─ --color-primary  →  bg-primary / text-primary / border-primary / ...
  └─ --breakpoint-3xl →  3xl:flex / 3xl:hidden / ...
  └─ --font-display   →  font-display
```

**`@theme`과 `@layer`의 관계** — `@theme`은 Tailwind 전용, `@layer`와는 독립적이다. 커스텀 컴포넌트 스타일은 `@layer components`에 작성한다.

```css
@import "tailwindcss";
@theme { --color-brand: oklch(0.65 0.2 250); }

@layer components {
  .btn-brand { @apply bg-brand text-white px-4 py-2 rounded; }
}
```

---

## 3. SvelteKit + Tailwind v4 통합

### Vite 플러그인 설치

```bash
pnpm add tailwindcss @tailwindcss/vite
```

### `vite.config.ts` 설정

```ts
import { defineConfig } from "vite";
import { sveltekit } from "@sveltejs/kit/vite";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [
    tailwindcss(),
    sveltekit(),
  ],
});
```

`tailwindcss()`를 `sveltekit()` 앞에 위치시킨다.

### `app.css` 작성

```css
@import "tailwindcss";

@theme {
  --color-primary: oklch(0.65 0.2 250);
  --font-sans: "Inter", sans-serif;
}
```

### `+layout.svelte`에서 임포트

```svelte
<script>
  import "../app.css";
  let { children } = $props();
</script>

{@render children()}
```

### Svelte scoped CSS와 Tailwind 조합

Svelte `<style>` 블록은 컴포넌트 스코프로 격리된다.

```text
Tailwind 유틸리티  →  레이아웃, 간격, 색상 등 범용 스타일
Svelte <style>    →  animation, clip-path 등 컴포넌트 고유 스타일
```

### `class:` 디렉티브와 Tailwind 조합

```svelte
<script>
  let isActive = $state(false);
</script>

<button
  class="px-4 py-2 rounded transition-colors"
  class:bg-primary={isActive}
  class:bg-surface={!isActive}
  onclick={() => isActive = !isActive}
>토글</button>
```

---

## 4. 유용한 패턴

### 동적 클래스 처리 — clsx / cn 유틸리티

문자열 조합으로 동적 클래스를 관리하면 코드가 지저분해진다. `clsx`와 `tailwind-merge`를 조합한 `cn` 유틸리티를 쓴다.

```bash
pnpm add clsx tailwind-merge
```

```ts
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

// twMerge: 충돌 클래스 자동 해결 (px-2 px-4 → px-4)
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

사용 예시 — `variant` props를 받는 버튼:

```svelte
<script>
  import { cn } from "$lib/utils";
  let { variant = "default", class: className, children } = $props();
</script>

<button class={cn(
  "px-4 py-2 rounded font-medium",
  variant === "primary" && "bg-primary text-white",
  variant === "ghost"   && "bg-transparent hover:bg-surface",
  className
)}>{@render children?.()}</button>
```

### 조건부 스타일링 패턴

복잡한 변형(variant)은 객체 맵으로 관리한다.

```ts
// 클래스를 객체 맵으로 선언 → cn()에 전달
const sizeMap = { sm: "px-2 py-1 text-sm", md: "px-4 py-2", lg: "px-6 py-3 text-lg" };
const intentMap = {
  primary: "bg-primary text-white",
  ghost:   "bg-transparent border border-current",
};

// 사용: cn(sizeMap[size], intentMap[intent])
```

---

## 5. v3 → v4 마이그레이션 체크리스트

### 자동 마이그레이션 도구

```bash
# Node.js 20+ 필요
npx @tailwindcss/upgrade
```

프로젝트 내 설정 파일과 클래스명을 자동으로 변환한다. 실행 후 수동 확인 항목을 검토한다.

### 수동 확인 항목

**설정 파일 이전**

```text
[ ] tailwind.config.js 삭제
[ ] theme.extend 내용을 app.css @theme {} 블록으로 이전
[ ] plugins 배열 → @plugin 지시어로 이전
[ ] content 배열 제거 (자동 감지로 전환)
```

**유틸리티명 변경**

v4에서 스케일이 재조정되어 일부 클래스명이 변경됐다.

```text
[ ] shadow-sm  → shadow-xs
[ ] shadow     → shadow-sm
[ ] ring       → ring-3  (기본값 3px → 1px 변경)
    ring의 기본 색상도 blue-500 → currentColor 로 변경
[ ] blur-sm    → blur-xs
[ ] blur       → blur-sm
[ ] rounded-sm → rounded-xs
[ ] rounded    → rounded-sm
```

**border 기본색 변경**

```text
[ ] border 기본색: gray-200 → currentColor
    명시적 색상 없이 border만 쓰던 곳 확인
```

**`@apply` 변경**

```text
[ ] @layer 없이 @apply 사용 시 동작 변경됨
    커스텀 컴포넌트 스타일은 반드시 @layer components {} 안에서 @apply 사용
```

**제거된 유틸리티**

```text
[ ] text-opacity-* → text-{color}/* 로 대체
[ ] flex-grow-*    → grow-* 로 대체
[ ] decoration-slice → box-decoration-slice 로 대체
```

**패키지 분리**

```text
[ ] PostCSS 사용 시: tailwindcss → @tailwindcss/postcss
[ ] CLI 사용 시: tailwindcss → @tailwindcss/cli
    (Vite 플러그인 사용 시 해당 없음)
```

---

## 6. v3 vs v4 핵심 변경 비교

| 항목 | Tailwind v3 | Tailwind v4 |
|------|-------------|-------------|
| 설정 파일 | `tailwind.config.js` | `app.css @theme {}` |
| 진입점 | `@tailwind base/components/utilities` | `@import "tailwindcss"` |
| 테마 커스텀 | `theme.extend` (JS 객체) | `@theme` 블록 (CSS 변수) |
| content 경로 | `content: [...]` 명시 필요 | 자동 감지 |
| JIT 모드 | 별도 옵션 (`mode: 'jit'`) | 항상 활성 (기본값) |
| Vite 통합 | PostCSS 경유 | `@tailwindcss/vite` 플러그인 |
| ring 기본값 | `3px`, `blue-500` | `1px`, `currentColor` |
| border 기본색 | `gray-200` | `currentColor` |
| 테마 값 형식 | JS (hex, rgb 등) | CSS 변수 (oklch 권장) |
