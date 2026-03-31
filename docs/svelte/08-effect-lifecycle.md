# Effect Lifecycle

---

## 전체 수명 주기 타이밍 다이어그램

```text
[Mount]
  │
  ├─ 컴포넌트 setup 코드 실행 (동기)
  ├─ DOM 생성
  ├─ onMount 콜백 실행 (1회)
  ├─ $effect.pre 최초 실행
  ├─ $effect 최초 실행
  │
[State Change]
  │
  ├─ 1. 변경 배치 (같은 microtask 내 변경은 한 번에 처리)
  ├─ 2. $effect.pre 실행 ← 이전 DOM 상태 접근 가능
  ├─ 3. DOM 업데이트
  ├─ 4. $effect 실행   ← 최신 DOM 상태 접근 가능
  │
[Destroy]
  │
  ├─ $effect teardown (return 함수)
  └─ onMount teardown (return 함수)
```

핵심: **$effect.pre -> DOM 업데이트 -> $effect** 순서. 재실행 시 teardown이 먼저 호출된다.

---

## React useEffect vs Svelte $effect 타이밍 비교

```text
React                              Svelte
─────────────────────────          ─────────────────────────
setState()                         $state 변경
  │                                  │
  ▼                                  ▼
render (VDOM 생성)                 변경 배치 (microtask)
  │                                  │
  ▼                                  ▼
DOM commit                         $effect.pre ← React에 없음
  │                                  │
  ▼                                  ▼
useEffect cleanup                  DOM 업데이트 (세분화된 갱신)
  │                                  │
  ▼                                  ▼
useEffect 실행                     teardown → $effect 실행
```

| | React `useEffect` | Svelte `$effect` |
|--|--|--|
| 의존성 | deps 배열 수동 선언 | 동기 읽기 자동 추적 |
| 배치 | 18+ auto batching | microtask 배치 (항상) |
| DOM 전 실행 | `useLayoutEffect` | `$effect.pre` |
| 조건부 의존성 | deps 배열 고정 | 마지막 실행 경로 기준 재추적 |

---

## $effect.pre -- DOM 업데이트 전 실행

레이아웃 측정 등 DOM 변경 전 작업이 필요할 때. React의 `useLayoutEffect`와 유사하나, DOM 업데이트 **전**에 실행되는 점이 다르다.

처음 실행 시 DOM이 아직 없으므로 `?.` 옵셔널 체이닝 또는 early return 필수.

---

## Cleanup (teardown)

`$effect` 안에서 함수를 반환하면 **재실행 직전** + **컴포넌트 파괴 시** 호출된다.

```js
$effect(() => {
  const timer = setInterval(() => { /* ... */ }, 1000)
  return () => clearInterval(timer)  // teardown
})
```

---

## onMount vs $effect

```js
import { onMount } from 'svelte'
onMount(() => {
  console.log('마운트됨')
  return () => console.log('파괴됨')
})
```

| | `$effect` | `onMount` |
|--|--|--|
| 실행 횟수 | 의존성 변경 시마다 | 마운트 시 1회 |
| teardown 시점 | 재실행 전 + 파괴 시 | 파괴 시만 |
| SSR | 실행 안 됨 | 실행 안 됨 |

> **원칙**: 상태 동기화에 `$effect`를 쓰지 말 것. `$derived`로 해결 가능한지 먼저 확인한다. `$effect`는 외부 시스템 연동(DOM 조작, 네트워크, analytics)용 escape hatch다.
