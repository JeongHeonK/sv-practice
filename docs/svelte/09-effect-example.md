# $effect 실전 — setInterval 패턴

`$effect` + 클린업의 핵심 패턴을 익히는 예제.

---

## 기본: 자동 카운터

```svelte
<script lang="ts">
  let count = $state(0)

  $effect(() => {
    const interval = setInterval(() => count++, 1000)
    return () => clearInterval(interval)  // 클린업 필수
  })
</script>

<h1>{count}</h1>
```

클린업 없으면 effect 재실행 시 이전 interval이 남아 count가 가속된다.

> **주의**: 이 예제에서 `count++`는 `$effect` 내부에서 상태를 변경한다. setInterval 콜백이 비동기로 실행되어 무한루프는 발생하지 않지만, 일반적으로 **`$effect` 안에서 상태를 변경하는 것은 안티패턴**이다. 타이머/구독처럼 외부 이벤트에 의한 변경만 예외적으로 허용된다.

---

## 의존성에 의한 재생성

`frequency`를 의존성으로 사용하면 값이 바뀔 때 interval이 자동으로 재생성된다.

```svelte
<script lang="ts">
  let count = $state(0)
  let frequency = $state(1000)

  $effect(() => {
    const interval = setInterval(() => count++, frequency)
    //                                          ↑ 동기 구간에서 읽힘 → 의존성 등록
    return () => clearInterval(interval)
  })
</script>

<button onclick={() => frequency *= 2}>느리게</button>
<button onclick={() => frequency /= 2}>빠르게</button>
```

### 왜 count는 의존성이 안 되나?

`setInterval` 콜백은 **비동기**로 실행된다. effect의 동기 실행 구간이 끝난 후 호출되므로 의존성으로 등록되지 않는다.

```text
effect 실행 (동기) → frequency 읽기 → 의존성 등록 ✓ → effect 완료

--- 1초 후 ---
setInterval 콜백 (비동기) → count++ → 의존성 등록 ✗ (스택 비어있음)
```

---

## 일시정지/재생

```svelte
<script lang="ts">
  let count = $state(0)
  let frequency = $state(1000)
  let paused = $state(false)

  $effect(() => {
    if (paused) return  // paused 읽음 → 의존성 등록

    const interval = setInterval(() => count++, frequency)
    return () => clearInterval(interval)
  })
</script>
```

- `paused = true` → effect 재실행 → `return` → 클린업으로 interval 제거
- `paused = false` → effect 재실행 → 새 interval 생성

---

## effect보다 이벤트 핸들러가 나은 경우

위 예제처럼 버튼으로 직접 제어하는 경우, effect 없이 이벤트 핸들러가 더 명확하다.

```js
let intervalId: ReturnType<typeof setInterval>

function start() {
  intervalId = setInterval(() => count++, frequency)
}
function pause() {
  clearInterval(intervalId)
}
function reset() {
  clearInterval(intervalId)
  count = 0
  start()
}
```

| | `$effect` | 이벤트 핸들러 |
| -- | -- | -- |
| interval 생명주기 | 의존성 변경에 묶임 | 직접 제어 |
| 적합한 경우 | 상태 변화에 자동 반응 | 사용자 액션에 직접 반응 |

**"$effect를 쓰기 전에 이벤트 핸들러로 해결되지 않는지 먼저 생각하자."**

---

## $effect vs 이벤트 콜백 — 데이터 요청 트리거

상태 변경 시 API를 다시 호출해야 할 때 두 가지 접근.

### $effect (의존성 기반 자동 재요청)

```ts
$effect(() => {
  fetchData()  // 내부에서 참조하는 상태 변경 시 자동 재실행
})
```

### 이벤트 콜백 (명시적 트리거)

```svelte
<select bind:value={category} onchange={fetchData}>
```

### 비교

```
┌──────────────┬─────────────────────┬──────────────────────┐
│              │ $effect             │ 이벤트 콜백           │
├──────────────┼─────────────────────┼──────────────────────┤
│ 트리거       │ 의존성 변경 시 자동  │ 사용자 액션 시에만    │
│ 초기 실행    │ ✅ 자동 (마운트 시)  │ ❌ 수동 호출 필요     │
│ 제어력       │ 낮음 (자동)         │ 높음 (명시적)         │
│ 적합한 상황  │ "X 바뀌면 항상 fetch"│ "이 액션에서만 fetch" │
└──────────────┴─────────────────────┴──────────────────────┘
```

### $effect + async 주의

```ts
// ❌ $effect에 직접 async 불가 (반환값이 Promise가 되어 클린업 함수로 쓸 수 없음)
$effect(async () => { ... })

// ✅ 별도 async 함수를 effect에서 호출
async function fetchData() { ... }
$effect(() => { fetchData() })
```
