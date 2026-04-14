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
DOM commit                         $effect.pre ← React의 useLayoutEffect와 동일
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
>
> **$effect 대안 체크리스트**:
> - 상태 → 상태 동기화? → `$derived` 사용
> - 외부 DOM 라이브러리 연동? → `{@attach ...}` 사용 (Svelte 5.29+)
> - 사용자 인터랙션 반응? → 이벤트 핸들러 사용
> - 외부 값 구독 (WebSocket 등)? → `createSubscriber` 사용
> - 디버깅 로깅? → `$inspect` 또는 `$inspect.trace()` 사용
> - `if (browser) { ... }` 래핑? → 불필요 ($effect는 서버에서 실행되지 않음)

---

## $effect.tracking -- 추적 컨텍스트 감지

현재 코드가 **추적 컨텍스트(tracking context)** 안에서 실행 중인지 `boolean`으로 반환한다.

### 추적 컨텍스트란?

반응형 값을 읽으면 해당 값의 변경을 **추적(구독)하는 곳**. 즉, 값이 바뀌면 다시 실행되는 곳이다.

| 위치 | `$effect.tracking()` | 이유 |
|--|--|--|
| `<script>` 최상위 | `false` | 컴포넌트 setup은 1회만 실행, 재실행 안 됨 |
| `$effect` 내부 | `true` | 의존성 변경 시 재실행됨 |
| 템플릿 (`{expression}`) | `true` | 값 변경 시 DOM 업데이트 필요 |
| 이벤트 핸들러 (`onclick`) | `false` | 사용자 액션으로 실행, 추적 불필요 |

```svelte
<script>
  console.log('setup:', $effect.tracking())       // false

  $effect(() => {
    console.log('effect:', $effect.tracking())     // true
  })
</script>

<p>template: {$effect.tracking()}</p>              <!-- true -->
```

### React 비유

React에는 직접 대응하는 API가 없다. 굳이 비유하면:

- **추적 컨텍스트** ≈ React 컴포넌트의 렌더 페이즈 (JSX 반환하는 본문)
- **비추적 컨텍스트** ≈ 이벤트 핸들러, setTimeout 콜백

### 용도

주로 **라이브러리 작성자**를 위한 고급 기능. 추적 컨텍스트 여부에 따라 동작을 분기할 때 사용한다.

```js
// 라이브러리 내부 예시: 추적 컨텍스트일 때만 구독 생성
function createSmartValue() {
  let value = $state(0)

  return {
    get current() {
      if ($effect.tracking()) {
        // effect/템플릿에서 읽힘 → 구독 설정
        subscribe()
      }
      return value
    }
  }
}
```

> 일반 애플리케이션 코드에서는 거의 사용할 일이 없다. "이런 게 있다" 정도로 알아두면 충분.

---

## $inspect.trace — effect 재실행 원인 추적

`$effect` 또는 `$derived.by` 내부의 **첫 번째 줄**에 `$inspect.trace(label)`을 추가하면, 어떤 의존성 변경이 해당 블록을 트리거했는지 콘솔에 출력한다.

```js
$effect(() => {
  $inspect.trace('my-effect')  // 첫 번째 줄에 작성
  canvas.draw(data)
})
```

반응성 디버깅에 유용 — 무언가 너무 자주 재실행되거나 예상치 못한 트리거가 있을 때 사용한다.
