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

### 네이티브 이벤트 전달

rest props spread를 사용하면 `onclick` 등 네이티브 이벤트가 자동으로 내부 요소에 전달된다.
`HTMLButtonAttributes`에 이벤트 타입이 포함되어 있어 자동완성도 동작한다.

```svelte
<!-- 사용하는 쪽 — 별도 작업 없이 바로 전달됨 -->
<Button onclick={() => alert('clicked')}>전송</Button>
```

### 커스텀 이벤트 — 함수 prop

Svelte 5에서 커스텀 이벤트는 그냥 함수 prop이다. `createEventDispatcher` 불필요.

```svelte
<!-- Button.svelte -->
<script lang="ts">
  type Props = HTMLButtonAttributes & {
    left?: Snippet<[boolean]>
    children: Snippet
    onleftmouseover?: () => void  // 커스텀 이벤트 = 함수 prop
  }

  let { left, children, onleftmouseover, ...restProps }: Props = $props()
</script>

<button {...restProps}>
  {#if left}
    <div
      role="presentation"
      onmouseenter={() => {
        isLeftHovered = true
        onleftmouseover?.()  <!-- 옵셔널이므로 ?. 로 호출 -->
      }}
    >
      {@render left(isLeftHovered)}
    </div>
  {/if}
  {@render children()}
</button>
```

```svelte
<!-- 사용하는 쪽 -->
<Button onleftmouseover={() => console.log('left hovered')}>검색</Button>
```

| | Svelte 4 | Svelte 5 |
|--|--|--|
| 커스텀 이벤트 | `createEventDispatcher()` + `dispatch('event')` | 함수 prop |
| 사용 | `on:event={handler}` | `onevent={handler}` |

### 이벤트 버블링과 캡처

이벤트는 기본적으로 안쪽 → 바깥쪽(버블링)으로 전파된다.

```svelte
<div onclick={() => console.log('div')}>
  <button onclick={(e) => {
    console.log('button')
    e.stopPropagation()  // div까지 전파 차단
  }}>
    클릭
  </button>
</div>
<!-- 출력: button -->
```

캡처는 바깥쪽 → 안쪽 순서. Svelte에서는 이벤트명에 `capture` 붙이면 된다.

```svelte
<!-- 일반 JS: addEventListener('click', fn, { capture: true }) -->
<!-- Svelte: onclickcapture -->
<div onclickcapture={(e) => {
  console.log('div 먼저')
  e.stopPropagation()  // button 이벤트 실행 안 됨
}}>
  <button onclick={() => console.log('button')}>클릭</button>
</div>
<!-- 출력: div 먼저 -->
```

### 이벤트 위임 (Event Delegation)

Svelte는 성능을 위해 이벤트 리스너를 해당 요소가 아닌 **루트 요소**에 등록한다.
DevTools에서 버튼을 검사하면 버튼에 직접 리스너가 없고 루트에 붙어있는 걸 볼 수 있다.
개발 시에는 신경 쓸 필요 없다.

---

## 외부 노출 — bind:this + export const

컴포넌트 내부 HTML 요소에 프로그래밍 방식으로 접근할 때 사용한다.
`bind:this`로 내부 요소 참조를 잡고, `export const`로 외부에 함수를 노출한다.

```svelte
<!-- Button.svelte -->
<script lang="ts">
  let button = $state<HTMLButtonElement>()

  // 외부에 노출할 함수
  export const focus = () => button?.focus()
  export const getButton = () => button  // 요소 자체를 반환
</script>

<button bind:this={button} {...restProps}>
  {@render children()}
</button>
```

```svelte
<!-- 사용하는 쪽 -->
<script lang="ts">
  import Button from './Button.svelte'

  let btn: ReturnType<typeof Button>  // 컴포넌트 타입 — export한 함수 자동완성 가능

  $effect(() => {
    btn.focus()               // export한 함수 호출
    btn.getButton()?.click()  // 요소 자체를 받아 DOM 메서드 직접 호출
  })
</script>

<Button bind:this={btn}>클릭</Button>
```

> `bind:this`를 컴포넌트에 사용하면 해당 컴포넌트가 `export`한 값/함수에 접근할 수 있다.
> HTML 요소에 `bind:this`를 사용하면 DOM 요소 자체를 참조한다.

---

## 동적 태그 — svelte:element

`href` 유무에 따라 `<button>` 또는 `<a>`를 렌더링하는 패턴.

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { HTMLButtonAttributes, HTMLAnchorAttributes } from 'svelte/elements'

  // href 있으면 앵커 속성, 없으면 버튼 속성
  type Props =
    | (HTMLAnchorAttributes & { href: string })
    | (HTMLButtonAttributes & { href?: never })

  let { href, children, ...restProps }: Props = $props()

  let tag = $derived(href ? 'a' : 'button')
  let el: HTMLButtonElement | HTMLAnchorElement
</script>

<svelte:element this={tag} bind:this={el} {...restProps}>
  {@render children()}
</svelte:element>
```

```svelte
<!-- 사용하는 쪽 -->
<Button>클릭</Button>                        <!-- <button> 렌더링 -->
<Button href="https://svelte.dev">이동</Button>  <!-- <a> 렌더링 -->
```

`href?: never` — href 없을 때 버튼 속성만 허용, `href: string` — href 있을 때 앵커 속성만 허용.
union 타입으로 분기해서 TypeScript가 상황에 맞는 속성을 정확히 추론한다.

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
