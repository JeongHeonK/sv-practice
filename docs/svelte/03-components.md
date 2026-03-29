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

---

## 외부 노출

```svelte
<script>
  // 외부에서 접근 가능한 값/함수 노출
  export const reset = () => { ... }
</script>
```
