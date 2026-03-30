# Snippets — 재사용 가능한 마크업

컴포넌트 내 반복되는 마크업을 스니펫으로 분리한다. 별도 파일 없이 컴포넌트 내부에서 정의하고 재사용 가능.

```svelte
{#snippet greeting(name: string)}
  <p>hello {name}!</p>
{/snippet}

{@render greeting('world')}
{@render greeting('svelte')}
```

## 인수 전달 (TypeScript)

```svelte
{#snippet sum(a: number, b: number)}
  <span>{a + b}</span>
{/snippet}

{@render sum(2, 2)}  <!-- 4 출력 -->
```

## 스니펫 중첩 (스코프 규칙)

스니펫 안에 또 다른 스니펫을 정의할 수 있다.
**내부에서 정의된 스니펫은 해당 스니펫 안에서만 사용 가능.**

```svelte
{#snippet sum(a: number, b: number)}
  {#snippet bold(x: any)}
    <b>{x}</b>          <!-- sum 내부에서만 사용 가능 -->
  {/snippet}
  <span>{@render bold(a + b)}</span>
{/snippet}

<!-- ✗ 오류: bold는 sum 스니펫 외부에서 접근 불가 -->
{@render bold('text')}
```

외부에서도 쓰려면 스니펫을 최상위 레벨로 이동한다.

---

## Props로 전달

스니펫을 컴포넌트 prop으로 전달하면 컴포넌트를 동적으로 커스터마이징할 수 있다.

### 기본 방식

```svelte
<!-- +page.svelte -->
<script>
  import Button from './Button.svelte'
  import { Search } from 'lucide-svelte'
</script>

{#snippet left()}
  <Search />
{/snippet}

<Button {left}>검색</Button>
```

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte'

  interface Props {
    left?: Snippet       // 선택적
    right?: Snippet      // 선택적
    children: Snippet    // 필수
  }

  let { left, right, children }: Props = $props()
</script>

<button>
  {#if left}
    <div class="left-content">{@render left()}</div>
  {/if}
  {@render children()}
  {#if right}
    <div class="right-content">{@render right()}</div>
  {/if}
</button>
```

선택적 스니펫은 반드시 `{#if}`로 존재 여부를 확인한 후 렌더링해야 타입 오류가 없다.

### 단축 문법 (컴포넌트 내부에 직접 정의)

prop 이름과 스니펫 이름이 같으면 컴포넌트 태그 안에서 바로 정의해도 자동으로 전달된다.

```svelte
<!-- ✓ 권장: 컴포넌트 내부에 스니펫 정의 -->
<Button>
  {#snippet left()}
    <Search />
  {/snippet}
  검색
</Button>

<!-- 위와 동일한 결과, 하지만 외부 정의 방식 -->
{#snippet left()}<Search />{/snippet}
<Button {left}>검색</Button>
```

### `children` — 내장 특수 스니펫

컴포넌트 태그 사이의 텍스트/마크업은 자동으로 `children` prop으로 전달된다.

```svelte
<Button>검색</Button>
<!-- "검색" 텍스트가 children 스니펫으로 전달됨 -->
```

```svelte
<!-- Button.svelte에서 사용 -->
{@render children()}
```

### 스니펫 Props에 인수 전달 — 컴포넌트 내부 상태 공유

컴포넌트가 내부 상태(hover 등)를 스니펫 인수로 넘겨 외부에서 활용할 수 있다.

**타입 지정:** `Snippet<[타입, ...]>` — 제네릭 배열로 인수 순서대로 명시

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte'

  interface Props {
    left?: Snippet<[boolean]>  // boolean 인수 1개를 받는 스니펫
    children: Snippet
  }

  let { left, children }: Props = $props()

  let isLeftHovered = $state(false)
</script>

<button>
  {#if left}
    <div
      class="left-content"
      role="presentation"
      onmouseenter={() => isLeftHovered = true}
      onmouseleave={() => isLeftHovered = false}
    >
      {@render left(isLeftHovered)}  <!-- 내부 상태를 인수로 전달 -->
    </div>
  {/if}
  {@render children()}
</button>
```

```svelte
<!-- +page.svelte — 인수로 hover 상태를 받아 활용 -->
<Button>
  {#snippet left(hovered)}
    {#if hovered}
      <SearchX />   <!-- 호버 시 다른 아이콘 -->
    {:else}
      <Search />
    {/if}
  {/snippet}
  검색
</Button>
```

**`children`에 인수를 전달하려면** 명시적 스니펫 정의가 필요하다.
태그 사이 텍스트는 인수를 받을 수 없기 때문.

```svelte
<!-- ✗ children이 인수를 받아야 할 때 — 텍스트 방식 불가 -->
<Button>검색</Button>

<!-- ✓ 명시적 snippet으로 정의해야 인수 접근 가능 -->
<Button>
  {#snippet children(hovered)}
    검색 {hovered}
  {/snippet}
</Button>
```
