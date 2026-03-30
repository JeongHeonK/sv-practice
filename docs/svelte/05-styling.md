# 스타일링 Props & class

## CSS 스코핑과 :global

### 기본 동작 — 컴포넌트 스코프

Svelte `<style>`의 모든 CSS는 해당 컴포넌트 내 요소에만 적용된다.
컴파일 시 요소에 해시 클래스(예: `svelte-xyz`)가 자동 추가되는 방식으로 동작한다.

```svelte
<!-- Button.svelte -->
<style>
  button { background: red; }  /* 이 컴포넌트의 button에만 적용 */
</style>
```

### 타겟팅 불가능한 경우

아래 요소들은 이 컴포넌트에 직접 존재하지 않으므로 일반 CSS로 타겟팅할 수 없다:

- **자식 컴포넌트 내부 HTML** — `<Button>`을 사용하는 페이지에서 Button 내부의 `<button>`을 타겟팅 불가
- **외부 라이브러리 컴포넌트** — lucide-svelte 아이콘 등이 렌더링하는 SVG
- **`{@html}`로 삽입된 HTML** — 서버에서 내려온 HTML 문자열 내부 태그

### `:global` 선택자

스코프를 해제하여 컴포넌트 외부 요소까지 타겟팅한다.

```svelte
<style>
  /* 인라인 방식 — 특정 선택자만 전역 */
  .wrapper :global(button) { background: blue; }

  /* 블록 방식 — 블록 내 모든 선택자가 전역 */
  :global {
    body { background: #111; }
  }

  /* 3rd party 아이콘 SVG 타겟팅 */
  .left-content :global(svg) { stroke: red; }

  /* {@html}로 삽입된 HTML 내부 p 태그 */
  .wrapper :global(p) { color: white; }
</style>
```

### CSS 키프레임 전역 선언

키프레임도 기본적으로 컴포넌트 스코프다. 전역으로 쓰려면 `-global-` 접두사 사용.

```svelte
<style>
  /* 컴포넌트 스코프 — 이 컴포넌트 내에서만 사용 가능 */
  @keyframes move { ... }

  /* 전역 — 어디서든 'move'라는 이름으로 사용 가능 */
  @keyframes -global-move { ... }
  /* 컴파일 후 → @keyframes move (접두사 제거됨) */
</style>
```

### 언제 써야 하나

| 상황 | 권장 방법 |
|------|-----------|
| 자식 컴포넌트 스타일 변경 | props 또는 CSS 변수 |
| 제어 불가능한 3rd party 컴포넌트 | `:global` |
| `{@html}` 내부 요소 타겟팅 | `:global` |
| 전역 요소(`body`, `html`) | `:global` |

> `:global`은 스코프 보호를 완전히 해제하므로 꼭 필요한 경우에만 최소 범위로 사용한다.
> 부모 스코프 선택자(`.wrapper :global(button)`)와 조합해 범위를 제한하는 것이 좋다.

---

## SCSS 사용

`lang="scss"` 추가 후 `sass-embedded` 설치만 하면 별도 설정 없이 동작한다.

```bash
pnpm add -D sass-embedded
```

```svelte
<style lang="scss">
  button {
    /* 중첩 문법 사용 가능 */
    .left-content { ... }
    .right-content { ... }
  }
</style>
```

---

## Props 기본값

선택적 props에 `=` 로 기본값을 지정하면 전달하지 않아도 항상 값이 존재한다.

```ts
let { size = 'small', shadow = false, children }: Props = $props()
```

```svelte
<!-- shadow만 넘기면 true로 인식 -->
<Button size="large" shadow>확인</Button>
```

---

## 조건부 클래스 (Svelte 5.16+)

`class` 속성이 **객체와 배열**을 직접 받을 수 있게 됐다. `class:` 디렉티브는 deprecated.

### 객체 방식

키 = 클래스명, 값 = 조건. 조건이 truthy일 때만 클래스 추가.

```svelte
<button class={{ small: size === 'small', lg: size === 'lg', shadow }}>
<!--                                                           ↑ 변수명=클래스명이면 단축 가능 -->
```

### 배열 방식

falsy 값은 무시되고 truthy 문자열만 클래스로 추가된다.

```svelte
<button class={[
  size === 'small' && 'small',
  size === 'lg'    && 'lg',
  shadow           && 'shadow',
]}>
```

배열 안에 객체를 섞을 수도 있다 (자동으로 flat 처리).

```svelte
<button class={[
  size === 'small' && 'small',
  { shadow, test: true }
]}>
```

### 외부 클래스 수용 — `class` prop 받기

`class`는 예약어라 props에서 받을 때 rename이 필요하다.

```svelte
<script lang="ts">
  let { class: class_, size = 'small', shadow = false, children }: Props = $props()
</script>

<button class={[
  size === 'small' && 'small',
  size === 'lg'    && 'lg',
  shadow           && 'shadow',
  class_,           <!-- 외부에서 전달받은 클래스 추가 -->
]}>
```

```svelte
<!-- 사용 시 -->
<Button class={{ test: true }}>확인</Button>
```

> **참고:** `class:` 디렉티브는 여전히 동작하지만 신규 코드에서는 배열/객체 방식 사용 권장.

---

## 네이티브 HTML 속성 전달 — rest props 패턴

커스텀 컴포넌트는 `disabled`, `aria-*` 등 네이티브 속성을 자동으로 내부 요소에 전달하지 않는다.

```svelte
<!-- ✗ disabled가 실제 <button>에 전달되지 않음 -->
<Button disabled>전송</Button>
```

### 해결: HTMLButtonAttributes + rest props

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte'
  import type { HTMLButtonAttributes } from 'svelte/elements'

  // ✓ type 교차 타입 사용 — interface extends는 children 타입 충돌로 불가
  type Props = HTMLButtonAttributes & {
    left?: Snippet<[boolean]>
    children: Snippet
  }

  // 명시적으로 쓸 props만 구조분해, 나머지는 restProps로
  let { left, children, ...restProps }: Props = $props()

  let isLeftHovered = $state(false)
</script>

<!-- restProps를 네이티브 요소에 spread -->
<button {...restProps}>
  {#if left}
    <div
      role="presentation"
      onmouseenter={() => isLeftHovered = true}
      onmouseleave={() => isLeftHovered = false}
    >
      {@render left(isLeftHovered)}
    </div>
  {/if}
  {@render children()}
</button>
```

이렇게 하면 `disabled`, `aria-label`, `type`, `onclick` 등 모든 네이티브 버튼 속성이 자동으로 전달된다.

> **왜 `interface extends`가 안 되나?**
> `HTMLButtonAttributes`의 `children`은 React 호환 타입이고, 우리 `children`은 `Snippet` 타입이라 충돌.
> `type` 교차 타입(`&`)을 쓰면 우리 선언이 우선권을 가져 오류가 사라진다.

---

## CSS 변수를 prop처럼 전달

컴포넌트에 별도 prop 없이 CSS 변수를 직접 넘길 수 있다.

```svelte
<!-- 사용하는 쪽 — prop 정의 없이 바로 전달 -->
<Button --button-bgcolor="red" --button-color="green">클릭</Button>
<Button>클릭</Button>  <!-- 기본값 사용 -->
```

```svelte
<!-- Button.svelte — CSS에서 var()로 사용 -->
<style>
  button {
    background: var(--button-bgcolor, blue);  /* 두번째 인자 = 기본값 */
    color: var(--button-color, white);
  }
</style>
```

Svelte가 내부적으로 `display: contents` div로 감싸고 변수를 정의한다:

```html
<!-- 컴파일 결과 -->
<div style="display:contents; --button-bgcolor:red; --button-color:green">
  <button>클릭</button>
</div>
```

`$props()`로 받을 필요 없고 JS에서 접근 불가 — 순수 스타일 커스터마이징 전용이다.
