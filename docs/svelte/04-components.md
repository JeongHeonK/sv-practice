# 컴포넌트

## 파일 = 컴포넌트

React는 함수를 정의하고 export한다. Svelte는 `.svelte` 파일 자체가 컴포넌트다.

```svelte
<!-- Button.svelte — 이것만으로 컴포넌트 완성 -->
<button>click</button>
```

```svelte
<script>
  import Button from './Button.svelte'
</script>
<Button />
```

---

## Props — $props()

React의 함수 파라미터 대신 `$props()` rune을 사용한다.

```svelte
<script>
  // React:  function Button({ label, count = 0 }) { ... }
  // Svelte:
  let { label, count = 0 } = $props()
</script>

<button>{label}: {count}</button>
```

### 핵심 규칙: props 의존 값은 $derived

Svelte 스크립트는 **한 번만 실행**된다. React처럼 함수가 다시 호출되지 않는다.

```js
let { type } = $props()

let color = $derived(type === 'danger' ? 'red' : 'green')  // ✓ 재계산됨
let color = type === 'danger' ? 'red' : 'green'            // ✗ 초기값 고정
```

### 구조분해 후 파생값 버그

`$props()`에서 꺼낸 객체를 다시 구조분해하면 반응성이 끊긴다.

```js
// ✗ notification이 바뀌어도 title/body는 초기값
let { title, body } = $props<{ notification: Notification }>().notification

// ✓ 해결 — 객체로 받아서 직접 접근하거나 $derived
let { notification } = $props()
let title = $derived(notification.title)
```

---

## $bindable — 양방향 바인딩

React에서는 `onChange` 콜백으로 상태를 올려보낸다. Svelte는 `$bindable`로 자식이 부모 상태를 직접 변경할 수 있다.

```svelte
<!-- Child.svelte -->
<script>
  let { value = $bindable('') } = $props()
</script>
<input bind:value />
```

```svelte
<!-- Parent.svelte -->
<Child bind:value={myValue} />
```

`$bindable` 없이 부모 소유 상태를 자식이 변경하면 ownership 경고가 발생한다.

**콜백 vs $bindable**: 부모가 업데이트 로직을 제어하려면 콜백, 자식이 내부적으로 관리하면 `$bindable`.

---

## 이벤트 핸들러

`on:click` (Svelte 4)이 아니라 `onclick` (네이티브 속성)을 사용한다.

```svelte
<button onclick={() => count++}>click</button>

<!-- window/document 이벤트 -->
<svelte:window onkeydown={handleKey} />
```

### 커스텀 이벤트 = 함수 prop

React와 동일하게 함수를 prop으로 넘긴다. `createEventDispatcher`는 레거시.

```svelte
<!-- Child.svelte -->
<script lang="ts">
  let { onsubmit }: { onsubmit?: (value: string) => void } = $props()
</script>
<button onclick={() => onsubmit?.('hello')}>전송</button>
```

### 네이티브 속성 전달 — rest props

`disabled`, `aria-*` 등은 자동으로 전달되지 않는다. rest props + spread로 해결.

```svelte
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements'

  type Props = HTMLButtonAttributes & { children: Snippet }
  let { children, ...restProps }: Props = $props()
</script>

<button {...restProps}>
  {@render children()}
</button>
```

이제 `<Button disabled>`, `<Button onclick={...}>` 모두 내부 `<button>`에 전달된다.

---

## 외부 노출 — export const

자식 컴포넌트의 메서드를 부모에서 호출해야 할 때.

```svelte
<!-- Child.svelte -->
<script>
  let el = $state<HTMLInputElement>()
  export const focus = () => el?.focus()
</script>
<input bind:this={el} />
```

```svelte
<!-- Parent.svelte -->
<script>
  let child: ReturnType<typeof Child>
</script>
<Child bind:this={child} />
<button onclick={() => child.focus()}>포커스</button>
```

---

## 레거시 → Svelte 5 대체표

| 레거시 | Svelte 5 |
| -- | -- |
| `export let x` | `let { x } = $props()` |
| `on:click={...}` | `onclick={...}` |
| `<slot>` | `{#snippet}` + `{@render}` |
| `<svelte:component this={C}>` | `<C>` (동적 컴포넌트) |
| `use:action` | `{@attach ...}` |
| `class:active={bool}` | `class={[bool && 'active']}` |
| stores | `$state` 클래스 |
