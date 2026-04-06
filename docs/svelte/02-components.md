# 컴포넌트 & 템플릿 API

컴포넌트 구조, Props, 이벤트, Snippets, 템플릿 문법, 스타일링, Actions/Attachments, 특수 요소, `<script module>`까지.

---

## 1. 파일 = 컴포넌트

`.svelte` 파일 자체가 컴포넌트다. 별도 export나 함수 정의가 필요 없다.

```svelte
<!-- Button.svelte -->
<button>click</button>
```

```svelte
<script>
  import Button from './Button.svelte'
</script>
<Button />
```

---

## 2. Props: `$props()`, 기본값, rest props

`$props()` rune으로 props를 선언한다.

```svelte
<script lang="ts">
  let { label, count = 0 } = $props()
</script>

<button>{label}: {count}</button>
```

### 네이티브 속성 전달 — rest props

`disabled`, `aria-*` 등은 자동으로 전달되지 않는다. rest props + spread로 해결.

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte'
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

## 3. 이벤트: `onclick` 속성 방식

Svelte 5에서는 `on:` 디렉티브 대신 **속성 형태**로 이벤트를 바인딩한다.

```svelte
<button onclick={() => count++}>클릭</button>
<button {onclick}>축약형</button>

<!-- ❌ 레거시: on:click -->
```

---

## 4. 커스텀 이벤트 = 함수 prop

함수를 prop으로 넘긴다. `createEventDispatcher`는 레거시.

```svelte
<!-- Child.svelte -->
<script lang="ts">
  let { onsubmit }: { onsubmit?: (value: string) => void } = $props()
</script>
<button onclick={() => onsubmit?.('hello')}>전송</button>
```

```svelte
<!-- Parent.svelte -->
<Child onsubmit={(v) => console.log(v)} />
```

---

## 5. `$bindable` — 양방향 바인딩

자식이 부모 상태를 직접 변경할 수 있게 한다.

```svelte
<!-- Child.svelte -->
<script lang="ts">
  let { value = $bindable('') } = $props()
</script>
<input bind:value />
```

```svelte
<!-- Parent.svelte -->
<Child bind:value={myValue} />
```

`$bindable` 없이 부모 소유 상태를 자식이 변경하면 **ownership 경고**가 발생한다.

**콜백 vs `$bindable`**: 부모가 업데이트 로직을 제어하려면 콜백, 자식이 내부적으로 관리하면 `$bindable`.

---

## 6. `export const` — 부모에서 자식 메서드 호출

```svelte
<!-- Child.svelte -->
<script lang="ts">
  let el = $state<HTMLInputElement>()
  export const focus = () => el?.focus()
</script>
<input bind:this={el} />
```

```svelte
<!-- Parent.svelte -->
<script lang="ts">
  let child: ReturnType<typeof Child>
</script>
<Child bind:this={child} />
<button onclick={() => child.focus()}>포커스</button>
```

---

## 7. Snippets & `{@render}` — children, render props 대체

### children snippet

태그 사이 콘텐츠가 자동으로 `children` snippet이 된다.

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte'
  let { children }: { children: Snippet } = $props()
</script>

<button>{@render children()}</button>
```

```svelte
<Button>클릭</Button>  <!-- "클릭"이 children으로 전달 -->
```

### snippet props (render props 대체)

snippet을 prop으로 전달하여 컴포넌트 내부 상태를 외부에 노출할 수 있다.

```svelte
<!-- Button.svelte -->
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
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div onmouseenter={() => hovered = true} onmouseleave={() => hovered = false}>
      {@render left(hovered)}
    </div>
  {/if}
  {@render children()}
</button>
```

```svelte
<Button>
  {#snippet left(hovered)}
    {#if hovered}<SearchX />{:else}<Search />{/if}
  {/snippet}
  검색
</Button>
```

선택적 snippet은 `{#if}`로 존재 여부를 확인한 후 렌더링해야 한다.

`children`에 인수를 전달하려면 명시적 snippet 정의가 필요하다:

```svelte
<Button>
  {#snippet children(hovered)}
    검색 {hovered ? '(hover)' : ''}
  {/snippet}
</Button>
```

---

## 8. 템플릿 문법: `{#each}`, `{#if}`, `{#await}`, `{@const}`

### `{#each}` — key 필수

```svelte
{#each notifications as notification (notification.id)}
  <li>{notification.title}</li>
{/each}
```

key가 없으면 인덱스 기반으로 DOM을 재사용해 삭제/이동 시 잘못된 요소가 남는다. **인덱스를 key로 쓰면 안 된다.**

인덱스 접근, 구조분해, 빈 배열 처리:

```svelte
{#each notifications as { id, title, body }, index (id)}
  <li>{index}: {title}</li>
{:else}
  <p>알림이 없습니다.</p>
{/each}
```

### `{@const}` — 템플릿 내 상수

`{#each}`, `{#if}`, `{#snippet}` 등의 **직접 자식**이어야 한다.

```svelte
{#each notifications as { title, date } (title)}
  {@const dateObj = new Date(date)}
  <li>
    <time datetime={dateObj.toISOString()}>{dateObj.toLocaleDateString()}</time>
  </li>
{/each}
```

### `{#await}` — Promise 기반 비동기 UI

```svelte
{#await dataPromise}
  <p>로딩 중...</p>
{:then data}
  <p>{data.name}</p>
{:catch error}
  <p>에러: {error.message}</p>
{/await}
```

---

## 9. 배열 뮤테이션

`$state` 배열은 Proxy이므로 **직접 변경**해도 반응성이 동작한다.

```js
let items = $state(generateItems(10))

items.splice(index, 1)     // ✓ 요소 제거
items.push(newItem)        // ✓ 요소 추가
items[0].title = 'new'     // ✓ 중첩 프로퍼티 수정
```

---

## 10. 스타일링: CSS 스코프, `:global`, 조건부 class, CSS 변수

### CSS 스코프

`<style>` 블록은 자동으로 컴포넌트 스코프다. 컴파일 시 해시 클래스가 추가된다.

```svelte
<style>
  button { background: red; }  /* 이 컴포넌트의 button에만 적용 */
</style>
```

### `:global` — 스코프 해제

```svelte
<style>
  .wrapper :global(svg) { stroke: red; }

  :global {
    body { background: #111; }
  }
</style>
```

> `:global`은 부모 선택자와 조합해 범위를 제한한다.

### 조건부 class (Svelte 5.16+)

```svelte
<!-- 객체 방식 -->
<button class={{ active: isActive, shadow }}>

<!-- 배열 방식 -->
<button class={[isActive && 'active', shadow && 'shadow']}>
```

외부 클래스 수용:

```svelte
<script lang="ts">
  let { class: class_, ...rest }: Props = $props()
</script>

<button class={['base', class_]} {...rest}>
```

### CSS 변수로 자식 스타일 제어

```svelte
<!-- 부모 -->
<Button --color="red" />

<!-- Button.svelte -->
<style>
  button { color: var(--color, black); }
</style>
```

### SCSS

```bash
pnpm add -D sass-embedded
```

```svelte
<style lang="scss">
  button { .icon { ... } }
</style>
```

---

## 11. Actions (`use:`) & Attachments (`{@attach}`)

DOM 요소에 커스텀 동작을 부착하는 메커니즘. **Svelte 5.29+에서는 `{@attach}`를 기본으로 사용**한다.

### Action 기본 — `use:`

```ts
// src/lib/actions/longpress.svelte.ts
import type { Action } from 'svelte/action'

export const longpress: Action<
  HTMLButtonElement,
  () => { duration?: number },
  { onlongpress: (e: CustomEvent) => void }
> = (node, getOptions) => {
  $effect(() => {
    const options = getOptions()
    const duration = options.duration !== undefined ? options.duration : 1000

    let timer: number

    function handleMouseDown() {
      timer = setTimeout(() => {
        node.dispatchEvent(new CustomEvent('longpress'))
      }, duration)
    }

    function handleMouseUp() {
      clearTimeout(timer)
    }

    node.addEventListener('mousedown', handleMouseDown)
    node.addEventListener('mouseup', handleMouseUp)

    return () => {
      clearTimeout(timer)
      node.removeEventListener('mousedown', handleMouseDown)
      node.removeEventListener('mouseup', handleMouseUp)
    }
  })
}
```

```svelte
<button use:longpress={() => ({ duration })} onlongpress={() => console.log('길게 눌림!')}>
  누르기
</button>
```

### 반응형 옵션 전달 패턴

```svelte
<!-- ❌ 객체 직접 전달: 생성 시점 값만 복사, 변경 감지 불가 -->
<button use:longpress={{ duration }}>

<!-- ✅ 함수로 감싸기: $effect 내부에서 호출하여 반응형 추적 -->
<button use:longpress={() => ({ duration })}>
```

### Attachment — `{@attach}` (5.29+, 권장)

Action의 개선된 버전. **기본 반응형** + **컴포넌트에도 적용 가능**.

```svelte
<script>
  import type { Attachment } from 'svelte'

  const myAttachment: Attachment = (element) => {
    console.log('마운트', element)
    return () => console.log('정리')
  }
</script>

<button {@attach myAttachment}>Click</button>
```

옵션을 받는 형태 — Attachment를 반환하는 함수:

```ts
import tippy, { type Props } from 'tippy.js'
import type { Attachment } from 'svelte'

export const tooltip = (options: Partial<Props>): Attachment => {
  return (element) => {
    const tip = tippy(element, options)
    return () => tip.destroy()
  }
}
```

```svelte
<!-- 기본 반응형: content 변경 시 자동 재실행 -->
<button {@attach tooltip({ content, animation: 'scale' })}>Hover</button>
```

### Action vs Attachment 비교

| 항목 | Action (`use:`) | Attachment (`{@attach}`) |
|------|-----------------|--------------------------|
| 적용 대상 | HTML 요소만 | HTML 요소 + 컴포넌트 |
| 반응성 | 수동 (함수 래핑) | 자동 ($effect로 실행) |
| 인라인 | 불가 | 가능 |
| 컴포넌트 전달 | 불가 | props spread로 전달 가능 |
| 최소 버전 | - | Svelte 5.29+ |

> 새 코드에서는 `{@attach}`를 기본으로 사용한다. Action은 deprecated되지 않았지만, Attachment가 반응성·적용 범위·코드 간결성 모든 면에서 우월하다.

---

## 12. 특수 요소

### `<svelte:window>` — Window 이벤트/값 바인딩

```svelte
<svelte:window onkeydown={handleKey} bind:scrollY />
```

더 간편한 방법:

```svelte
<script>
  import { scrollY } from 'svelte/reactivity/window'
</script>
<p>scrollY: {scrollY.current}</p>
```

### `<svelte:head>` — 동적 SEO 태그

```svelte
<svelte:head>
  <title>{data.title} | My App</title>
  <meta name="description" content={data.description} />
</svelte:head>
```

### `<svelte:boundary>` — 에러 경계

```svelte
<svelte:boundary onerror={(error) => console.log(error)}>
  <BuggyComponent />

  {#snippet failed(error, reset)}
    <p>에러 발생: {error.message}</p>
    <button onclick={reset}>재시도</button>
  {/snippet}
</svelte:boundary>
```

포착됨: 렌더링 에러, `$effect` 내부 에러. **포착 안 됨: 이벤트 핸들러 내부 에러** (브라우저가 실행하므로 Svelte가 감싸지 못함).

### `<svelte:element>` — 동적 태그

```svelte
<svelte:element this={href ? 'a' : 'button'} {href} {...rest}>
  {@render children?.()}
</svelte:element>
```

### DOM/미디어 바인딩

```svelte
<!-- DOM 크기 (읽기 전용) -->
<div bind:offsetWidth={width} bind:offsetHeight={height} />

<!-- 미디어 (양방향) -->
<video {src} bind:duration bind:currentTime bind:paused bind:volume />
```

| 바인딩 | 방향 | 예시 |
|--------|------|------|
| `bind:value` | 양방향 | 입력 ↔ 상태 |
| `bind:this` | 단방향 | DOM → 변수 |
| `bind:scrollY` | 양방향 | 스크롤 ↔ 상태 |
| `bind:offsetWidth` | 읽기 전용 | DOM 크기 → 상태 |
| `bind:currentTime` | 양방향 | 미디어 탐색 |
| `bind:duration` | 읽기 전용 | 미디어 전체 길이 |

> `<svelte:window>`, `<svelte:document>`, `<svelte:body>`는 **컴포넌트 최상위**에 위치해야 한다. `<svelte:head>`, `<svelte:boundary>`는 어디서든 사용 가능.

---

## 13. `<script module>` — 인스턴스 간 코드 공유

```svelte
<!-- 인스턴스 스크립트: 인스턴스마다 실행 -->
<script lang="ts">
  let count = $state(0)
</script>

<!-- 모듈 스크립트: 한 번만 실행, 모든 인스턴스가 공유 -->
<script lang="ts" module>
  let shared = $state(0)
</script>
```

| 항목 | `<script>` | `<script module>` |
|------|------------|---------------------|
| 실행 횟수 | 인스턴스마다 | 1번만 |
| 상태 범위 | 각 인스턴스 독립 | 모든 인스턴스 공유 |
| `$props` 접근 | 가능 | 불가 |
| `export` | props로 노출 | 일반 JS export |

### `$state` vs 일반 변수 판단 기준

- **`$state` 필요**: 템플릿에서 표시, `$effect`에서 읽기, `$derived`의 의존성
- **일반 `let` 충분**: 함수 호출로만 접근, 내부 로직에서만 사용

### named export 패턴

```svelte
<script lang="ts" module>
  export type MyEvent = { id: string; timestamp: number }
  export const VERSION = '1.0.0'
  export function getCount() { return count }
</script>
```

```ts
import MyComponent, { type MyEvent, VERSION } from './MyComponent.svelte'
```

> **SSR 주의**: 모듈 레벨 상태는 서버에서 모든 요청이 공유한다. SSR 환경에서는 Context API를 사용해야 요청별 독립성이 보장된다.

---

## 실무 예시: 재사용 가능한 Button 컴포넌트

rest props + snippet + attachment를 조합한 실전 패턴.

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { Snippet, Attachment } from 'svelte'
  import type { HTMLButtonAttributes } from 'svelte/elements'

  type Props = HTMLButtonAttributes & {
    variant?: 'primary' | 'secondary' | 'danger'
    size?: 'sm' | 'md' | 'lg'
    left?: Snippet
    right?: Snippet
    children: Snippet
  }

  let {
    variant = 'primary',
    size = 'md',
    left,
    right,
    children,
    class: class_,
    ...restProps
  }: Props = $props()
</script>

<button
  class={['btn', `btn-${variant}`, `btn-${size}`, class_]}
  {...restProps}
>
  {#if left}<span class="btn-icon">{@render left()}</span>{/if}
  {@render children()}
  {#if right}<span class="btn-icon">{@render right()}</span>{/if}
</button>

<style>
  .btn {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    border: none;
    border-radius: 0.375rem;
    cursor: pointer;
    font-weight: 500;
  }
  .btn-primary { background: #3b82f6; color: white; }
  .btn-secondary { background: #6b7280; color: white; }
  .btn-danger { background: #ef4444; color: white; }
  .btn-sm { padding: 0.25rem 0.5rem; font-size: 0.875rem; }
  .btn-md { padding: 0.5rem 1rem; font-size: 1rem; }
  .btn-lg { padding: 0.75rem 1.5rem; font-size: 1.125rem; }
  .btn:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
```

```svelte
<!-- 사용 예시 -->
<Button variant="primary" onclick={handleSave} disabled={saving}>
  {#snippet left()}<SaveIcon />{/snippet}
  저장
</Button>

<Button variant="danger" size="sm">삭제</Button>

<!-- Attachment도 spread로 자동 전달 -->
<Button {@attach tooltip({ content: '저장합니다' })}>저장</Button>
```
