# Effect 실행 타이밍과 수명 주기

---

## $effect — 사용 전 확인

`$effect`는 탈출구(escape hatch)다. 먼저 대안을 고려한다.

| 상황 | 대안 |
| -- | -- |
| 상태 → 상태 동기화 | `$derived` |
| 버튼 클릭 부수 작업 | 이벤트 핸들러 |
| 외부 DOM 라이브러리 연동 | `{@attach ...}` |
| 외부 값 구독 (WebSocket 등) | `createSubscriber` |
| 반응성 디버깅 | `$inspect` |

또한 effect 안에서 `if (browser) {...}` 로 감싸면 안 된다.
effect는 서버에서 실행되지 않으므로 그런 조건문이 필요 없다.

```js
// ✗ 불필요
$effect(() => {
  if (browser) {
    doSomething()
  }
})

// ✓ 그냥 바로 사용
$effect(() => {
  doSomething()
})
```

---

## $effect 실행 타이밍

`$effect`는 **DOM 업데이트가 적용된 후** 실행된다.

```text
상태 변경
    ↓
DOM 업데이트 적용
    ↓
$effect 실행   ← 이 시점에 DOM에 접근하면 최신 DOM
```

- 컴포넌트 마운트 시: DOM이 마운트된 후 처음 실행
- 의존성 변경 시: DOM이 업데이트된 후 재실행

---

## bind:this — DOM 요소 참조

React의 `useRef`에 해당. `bind:this`로 DOM 요소 참조를 변수에 바인딩한다.

```svelte
<script lang="ts">
  let historyEl: HTMLParagraphElement

  $effect(() => {
    history.length  // 의존성 등록
    console.log(historyEl.innerText)  // 최신 DOM 값
  })
</script>

<p bind:this={historyEl}>{history}</p>
```

`$effect` 안에서 DOM에 접근하면 항상 최신 상태가 보장된다.

---

## $effect.pre — DOM 업데이트 전 실행

```text
상태 변경
    ↓
$effect.pre 실행   ← DOM 업데이트 전
    ↓
DOM 업데이트 적용
    ↓
$effect 실행       ← DOM 업데이트 후
```

```js
$effect.pre(() => {
  history.length  // 의존성
  console.log(historyEl?.innerText)  // DOM 업데이트 전 → 이전 값
})
```

- 처음 실행 시 DOM이 아직 없으므로 `?.` 옵셔널 체이닝 필수
- DOM 업데이트 전 측정(레이아웃 계산 등)이 필요할 때 사용

---

## tick() — DOM 업데이트 완료 대기

`tick`은 **보류 중인 DOM 업데이트가 완료되면 resolve되는 Promise**를 반환한다.

```js
import { tick } from 'svelte'

$effect.pre(() => {
  history.length

  // DOM 업데이트 전 작업
  console.log(historyEl?.innerText)  // 이전 DOM

  tick().then(() => {
    // DOM 업데이트 완료 후 작업
    console.log(historyEl.innerText)  // 최신 DOM
  })
})
```

`$effect.pre` 안에서 DOM 업데이트 전후 작업을 모두 처리해야 할 때 유용하다.

### $effect vs tick 타이밍 비교

| | 실행 시점 |
| -- | -- |
| `$effect.pre` | DOM 업데이트 전 |
| `$effect` | DOM 업데이트 후 |
| `tick().then(...)` | DOM 업데이트 후 (`$effect`와 동일) |

---

## 클린업 함수

`$effect` / `$effect.pre` 안에서 함수를 반환하면 **effect 재실행 직전** 또는
**컴포넌트 파괴 시** 실행된다.

```js
$effect(() => {
  const timer = setInterval(() => { ... }, 1000)

  return () => {
    clearInterval(timer)  // effect 재실행 전 or 컴포넌트 파괴 시 정리
  }
})
```

활용 사례:

- 이벤트 리스너 추가 / 제거
- WebSocket 열기 / 닫기
- 타이머 시작 / 취소

---

## onMount / onDestroy — 수명 주기 함수

`$effect`와 달리 **딱 한 번만 실행**된다.

```js
import { onMount, onDestroy } from 'svelte'

onMount(() => {
  console.log('컴포넌트 마운트')

  return () => {
    console.log('컴포넌트 파괴')  // 클린업
  }
})

onDestroy(() => {
  console.log('컴포넌트 파괴')
})
```

### $effect vs onMount 비교

| | `$effect` | `onMount` |
| -- | -- | -- |
| 실행 횟수 | 의존성 변경 시마다 | 마운트 시 1회 |
| 재실행 | 의존성 변경 시 | 없음 |
| 클린업 | 재실행 전 + 파괴 시 | 파괴 시만 |
| 실행 타이밍 | DOM 업데이트 후 | DOM 마운트 후 |

의존성이 있어서 재실행이 필요하면 `$effect`, 마운트 시 1회만 실행하면 `onMount`.
