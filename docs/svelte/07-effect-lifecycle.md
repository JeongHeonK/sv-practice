# Effect 수명 주기

---

## $effect 실행 타이밍

React의 `useEffect`와 마찬가지로 **DOM 업데이트 후** 실행된다.

```text
상태 변경 → DOM 업데이트 → $effect 실행 (최신 DOM 접근 가능)
```

### bind:this — DOM 참조

React의 `useRef`에 해당. `$effect` 안에서 접근하면 항상 최신 DOM.

```svelte
<script>
  let el: HTMLParagraphElement

  $effect(() => {
    console.log(el.innerText)  // DOM 업데이트 후 → 최신 값
  })
</script>

<p bind:this={el}>{text}</p>
```

---

## $effect.pre — DOM 업데이트 전 실행

레이아웃 측정 등 DOM이 변경되기 전 작업이 필요할 때 사용. 흔하지 않다.

```text
상태 변경 → $effect.pre (이전 DOM) → DOM 업데이트 → $effect (최신 DOM)
```

처음 실행 시 DOM이 아직 없으므로 `?.` 옵셔널 체이닝 필수.

---

## 클린업

`$effect` 안에서 함수를 반환하면 **재실행 전** 또는 **컴포넌트 파괴 시** 실행된다. React useEffect 클린업과 동일.

```js
$effect(() => {
  const timer = setInterval(() => { ... }, 1000)
  return () => clearInterval(timer)
})
```

---

## onMount — 마운트 시 1회 실행

`$effect`는 의존성이 바뀔 때마다 재실행되지만, `onMount`는 딱 한 번만 실행된다.

```js
import { onMount } from 'svelte'

onMount(() => {
  console.log('마운트됨')
  return () => console.log('파괴됨')  // 클린업
})
```

| | `$effect` | `onMount` |
| -- | -- | -- |
| 실행 횟수 | 의존성 변경 시마다 | 마운트 시 1회 |
| 클린업 | 재실행 전 + 파괴 시 | 파괴 시만 |

> **주의**: `$effect`나 `onMount` 안에서 `if (browser) {...}` 감싸지 말 것. 둘 다 서버에서 실행되지 않으므로 불필요하다.
