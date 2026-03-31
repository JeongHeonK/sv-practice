# 스타일링

## CSS 스코핑 — CSS Modules 없이 자동 스코프

React에서는 CSS Modules나 styled-components로 스코프를 만든다. Svelte는 `<style>` 자체가 컴포넌트 스코프다.

```svelte
<style>
  button { background: red; }  /* 이 컴포넌트의 button에만 적용 */
</style>
```

컴파일 시 해시 클래스(`svelte-xyz`)가 자동 추가되어 다른 컴포넌트에 영향을 주지 않는다.

### :global — 스코프 해제

자식 컴포넌트, 3rd party 라이브러리, `{@html}` 내부 요소는 일반 CSS로 타겟팅할 수 없다.

```svelte
<style>
  /* 특정 선택자만 전역 */
  .wrapper :global(svg) { stroke: red; }

  /* 블록 전체 전역 */
  :global {
    body { background: #111; }
  }
</style>
```

> `:global`은 스코프 보호를 해제하므로 부모 선택자(`.wrapper :global(...)`)와 조합해 범위를 제한한다.

---

## 조건부 클래스 (Svelte 5.16+)

React에서 `clsx`/`classnames`로 하던 것을 Svelte는 네이티브로 지원한다.

```svelte
<!-- 객체 방식 -->
<button class={{ active: isActive, shadow }}>

<!-- 배열 방식 -->
<button class={[isActive && 'active', shadow && 'shadow']}>
```

`class:` 디렉티브는 여전히 동작하지만 신규 코드에서는 배열/객체 방식이 권장된다.

### 외부 클래스 수용

`class`는 예약어라 props에서 rename이 필요하다.

```svelte
<script lang="ts">
  let { class: class_, ...rest }: Props = $props()
</script>

<button class={['base', class_]} {...rest}>
```

---

## 자식 컴포넌트 스타일 제어

React에서 `className` prop으로 자식 스타일을 바꾸듯이, Svelte는 **CSS 변수**를 사용한다.

```svelte
<!-- 부모 -->
<Button --color="red" />

<!-- Button.svelte -->
<style>
  button { color: var(--color, black); }
</style>
```

prop 정의 없이 CSS 변수가 자동으로 전달된다. JS에서 접근 불가 — 순수 스타일 전용.

제어 불가능한 3rd party 컴포넌트만 `:global`로 오버라이드한다.

---

## rest props로 네이티브 속성 전달

```svelte
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements'
  type Props = HTMLButtonAttributes & { children: Snippet }
  let { children, ...restProps }: Props = $props()
</script>

<button {...restProps}>{@render children()}</button>
```

`disabled`, `aria-label`, `onclick` 등 모든 네이티브 속성이 자동 전달된다.

---

## SCSS 사용

`sass-embedded` 설치만 하면 별도 설정 없이 동작한다.

```bash
pnpm add -D sass-embedded
```

```svelte
<style lang="scss">
  button {
    .icon { ... }
  }
</style>
```
