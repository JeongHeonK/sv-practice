# Snippets — React의 children/render props 대체

React에서 `children`, `render props`로 하던 것을 Svelte에서는 **Snippet**으로 한다.

---

## 기본 사용

```svelte
{#snippet greeting(name: string)}
  <p>hello {name}!</p>
{/snippet}

{@render greeting('world')}
```

---

## React children → Svelte children snippet

```tsx
// React
function Button({ children }) {
  return <button>{children}</button>
}
<Button>클릭</Button>
```

```svelte
<!-- Svelte — 태그 사이 콘텐츠가 자동으로 children snippet이 됨 -->
<script lang="ts">
  import type { Snippet } from 'svelte'
  let { children }: { children: Snippet } = $props()
</script>

<button>{@render children()}</button>
```

```svelte
<Button>클릭</Button>  <!-- "클릭"이 children으로 전달 -->
```

---

## React render props → Svelte snippet props

React에서 컴포넌트에 렌더 함수를 넘기는 패턴 → Svelte에서는 snippet을 prop으로 전달.

```tsx
// React render props
<Button left={() => <SearchIcon />}>검색</Button>
```

```svelte
<!-- Svelte — snippet을 prop으로 전달 -->
<Button>
  {#snippet left()}
    <Search />
  {/snippet}
  검색
</Button>
```

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte'

  let { left, children }: {
    left?: Snippet
    children: Snippet
  } = $props()
</script>

<button>
  {#if left}{@render left()}{/if}
  {@render children()}
</button>
```

선택적 snippet은 `{#if}`로 존재 여부를 확인한 후 렌더링해야 한다.

---

## Snippet에 인수 전달 — 내부 상태 공유

컴포넌트가 내부 상태를 snippet 인수로 넘겨 외부에서 활용할 수 있다. React의 render props 패턴과 동일한 목적.

```svelte
<!-- Button.svelte — 내부 hover 상태를 snippet에 전달 -->
<script lang="ts">
  import type { Snippet } from 'svelte'

  let { left, children }: {
    left?: Snippet<[boolean]>   // boolean 인수 1개
    children: Snippet
  } = $props()

  let hovered = $state(false)
</script>

<button>
  {#if left}
    <div onmouseenter={() => hovered = true} onmouseleave={() => hovered = false}>
      {@render left(hovered)}
    </div>
  {/if}
  {@render children()}
</button>
```

```svelte
<!-- 사용 측 — hover 상태를 받아 활용 -->
<Button>
  {#snippet left(hovered)}
    {#if hovered}<SearchX />{:else}<Search />{/if}
  {/snippet}
  검색
</Button>
```

`children`에 인수를 전달하려면 명시적 snippet 정의가 필요하다:

```svelte
<Button>
  {#snippet children(hovered)}
    검색 {hovered ? '(hover)' : ''}
  {/snippet}
</Button>
```
