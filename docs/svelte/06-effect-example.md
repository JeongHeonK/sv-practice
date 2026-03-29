# $effect 실전 예제 — 카운터

`setInterval` + 클린업으로 `$effect` 핵심 패턴을 익히는 예제.

---

## 기본 구조

```svelte
<script lang="ts">
  let count = $state(0)

  $effect(() => {
    const interval = setInterval(() => {
      count++
    }, 1000)

    return () => clearInterval(interval)  // 클린업 필수
  })
</script>

<h1>{count}</h1>
```

클린업 없으면 effect 재실행 시 이전 interval이 남아 interval이 누적되고
count가 1초마다 여러 번 증가한다.

---

## 동적 주기 — frequency 상태를 의존성으로

```svelte
<script lang="ts">
  let count = $state(0)
  let frequency = $state(1000)

  $effect(() => {
    const interval = setInterval(() => {
      count++  // ← setInterval 콜백 내부 = 비동기 → 의존성 미등록
    }, frequency)  // ← frequency는 동기적으로 읽힘 → 의존성 등록

    return () => clearInterval(interval)
  })
</script>

<button onclick={() => frequency *= 2}>느리게</button>
<button onclick={() => frequency /= 2}>빠르게</button>
```

### 비동기 콜백 내부는 의존성 추적 안 됨

```text
effect 실행 (동기 구간)
  ↓
frequency 읽기 → 스택 확인 → 의존성 등록 ✓
  ↓
setInterval 등록 (콜백은 나중에 실행)
  ↓
effect 완료 → 스택 pop

--- 1초 후 ---
setInterval 콜백 실행 (비동기)
  → 스택 비어있음
  → count 읽기/쓰기 → 의존성 등록 안됨 ✗
```

`setInterval` 콜백 안에서 `count`를 읽어도 의존성으로 등록되지 않는다.
effect가 `count` 변경 시 재실행되지 않으므로 interval이 한 번만 생성된다.

---

## 일시정지 / 재생

`paused` 상태를 effect 의존성으로 만들어 effect 재실행을 트리거한다.

```svelte
<script lang="ts">
  let count = $state(0)
  let frequency = $state(1000)
  let paused = $state(false)

  $effect(() => {
    if (paused) return  // paused가 의존성으로 등록됨

    const interval = setInterval(() => {
      count++
    }, frequency)

    return () => clearInterval(interval)
  })
</script>

<button onclick={() => paused = !paused}>
  {paused ? '재생' : '일시정지'}
</button>
```

- 일시정지 클릭 → `paused = true` → effect 재실행 → 클린업 실행 → interval 제거
- 재생 클릭 → `paused = false` → effect 재실행 → 새 interval 생성

---

## 리셋 — effect 강제 재실행 트릭

리셋 시 interval을 새로 만들고 싶다면 effect를 강제로 재실행해야 한다.
의존성 값을 바꿔야 재실행되므로, 임시값 → 원래값 순으로 변경하는 트릭을 사용한다.

```js
function reset() {
  count = 0

  // effect 강제 재실행: 임시값으로 변경 → 클린업 실행 → 원래값 복구 → 재생성
  const original = frequency
  frequency = 0           // 값 변경 → effect 클린업 실행
  frequency = original    // 원래값 복구 → effect 재실행 → interval 새로 생성
}
```

다소 hacky한 방법이다. 더 깔끔한 대안은 아래를 참고.

---

## 결론 — effect vs 이벤트 핸들러

이 카운터 예제처럼 버튼 클릭으로 interval을 직접 제어해야 하는 경우,
effect 없이 이벤트 핸들러에서 직접 처리하는 게 더 명확하다.

```js
// effect 없이 직접 제어
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

| | `$effect` 방식 | 이벤트 핸들러 방식 |
| -- | -- | -- |
| interval 생명주기 | 의존성 변경에 묶임 | 직접 제어 |
| 강제 재실행 | 임시값 트릭 필요 | 그냥 함수 호출 |
| 적합한 경우 | 외부 라이브러리 연동 | 버튼으로 직접 제어 |

**effect를 쓰려 할 때 "이벤트 핸들러로 해결 안 되나?" 먼저 고민할 것.**
