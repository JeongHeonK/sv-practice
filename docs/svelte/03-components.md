# 컴포넌트

## .svelte 파일 = 컴포넌트

파일 자체가 컴포넌트다. 별도 export 없이 import하면 바로 사용 가능.

```jsx
// React — 함수 정의 + export 필요
export default function Button() {
  return <button>click</button>
}
```

```svelte
<!-- Svelte — 파일 자체가 컴포넌트 -->
<button>click</button>
```

```svelte
<!-- 사용할 때 -->
<script>
  import Button from './Button.svelte'
</script>

<Button />
```

---

## Props

```svelte
<script>
  // props 받기
  let { label, count = 0 } = $props()
</script>

<button>{label}: {count}</button>
```

props는 언제든 바뀔 수 있다. props에 의존하는 값은 `$derived`로 선언해야 한다.

```js
let { type } = $props()

// ✓ type이 바뀌면 color도 재계산됨
let color = $derived(type === 'danger' ? 'red' : 'green')

// ✗ type이 바뀌어도 color는 초기값 유지
let color = type === 'danger' ? 'red' : 'green'
```

---

## 이벤트 핸들러

`on:click` 같은 레거시 문법 대신 `onclick` 속성을 사용한다.

```svelte
<!-- ✓ Svelte 5 -->
<button onclick={() => count++}>click</button>

<!-- ✗ 레거시 (Svelte 4 방식) -->
<button on:click={() => count++}>click</button>
```

`window`, `document` 이벤트는 `<svelte:window>`, `<svelte:document>` 사용:

```svelte
<svelte:window onkeydown={handleKey} />
<svelte:document onvisibilitychange={handleVisibility} />
```

---

## Snippets — 재사용 가능한 마크업

컴포넌트 내 반복되는 마크업을 스니펫으로 분리한다.

```svelte
{#snippet greeting(name)}
  <p>hello {name}!</p>
{/snippet}

{@render greeting('world')}
{@render greeting('svelte')}
```

스니펫은 props로 전달할 수도 있다.

---

## 외부 노출

```svelte
<script>
  // 외부에서 접근 가능한 값/함수 노출
  export const reset = () => { ... }
</script>
```

---

## Svelte 5 — 레거시 문법 대체표

| 레거시 | Svelte 5 |
| -- | -- |
| `export let x` | `let { x } = $props()` |
| `$$props`, `$$restProps` | `$props()` |
| `on:click={...}` | `onclick={...}` |
| `<slot>` | `{#snippet}` + `{@render}` |
| `<svelte:component this={C}>` | `<C>` |
| `<svelte:self>` | `import Self from './...'` + `<Self>` |
| `use:action` | `{@attach ...}` |
| `class:active={bool}` | `class={[bool && 'active']}` |
