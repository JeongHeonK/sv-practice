# Context API 실전 — 트리 범위 상태 공유

모듈 레벨 상태(11번 참조)와 달리, Context는 **컴포넌트 트리 범위**로 상태를 제한한다. SSR에서도 안전하고, 같은 컴포넌트를 여러 그룹에서 독립적으로 사용할 수 있다.

---

모듈 레벨 상태는 **앱 전체**에서 공유된다. 같은 컴포넌트를 여러 곳에서 쓰되, **그룹별로 독립된 상태**가 필요하면 Context를 사용한다.

## 문제: 모듈 상태는 모든 인스턴스에서 공유됨

```svelte
<!-- ClickToCount.svelte — 모듈 상태를 사용 -->
<script>
  import { count } from './counter.svelte.ts'
</script>
<button onclick={() => count.value++}>{count.value}</button>
```

```svelte
<!-- App.svelte -->
<ClickToCount />  <!-- ┐ -->
<ClickToCount />  <!-- ┘ 모두 같은 count를 공유 — 하나 클릭하면 전부 변경 -->

<ClickToCount />  <!-- ┐ -->
<ClickToCount />  <!-- ┘ 이것도 같은 count — 그룹 분리 불가능 -->
```

## 해결: 부모 컴포넌트에서 Context로 상태 제공

```svelte
<!-- Counter.svelte — 부모 컴포넌트 -->
<script lang="ts">
  import { setContext, type Snippet } from 'svelte'

  let { children }: { children: Snippet } = $props()

  // $state 객체를 context로 설정 → 반응형 상태 공유
  let count = $state({ value: 0 })

  function increment() { count.value++ }
  function reset() { count.value = 0 }

  setContext<CounterContext>('count', { count, increment, reset })
</script>

{@render children()}
```

```svelte
<!-- ClickToCount.svelte — 자식 컴포넌트 -->
<script lang="ts">
  import { getContext } from 'svelte'

  const counter = getContext<CounterContext>('count')
</script>

<button onclick={counter.increment}>{counter.count.value}</button>
<button onclick={counter.reset}>리셋</button>
```

```svelte
<!-- App.svelte -->
<script>
  import Counter from './Counter.svelte'
  import ClickToCount from './ClickToCount.svelte'
</script>

<Counter>           <!-- 그룹 A: 독립된 상태 -->
  <ClickToCount />
  <ClickToCount />  <!-- A 내부에서 동기화 -->
</Counter>

<Counter>           <!-- 그룹 B: 독립된 상태 -->
  <ClickToCount />
  <ClickToCount />  <!-- B 내부에서 동기화, A와 독립 -->
</Counter>
```

```
동작 원리:

  Counter A → setContext('count', $state({ value: 0 }))
    ├─ ClickToCount → getContext('count') → A의 상태
    └─ ClickToCount → getContext('count') → A의 상태  ← 동기화됨

  Counter B → setContext('count', $state({ value: 0 }))  ← 별개의 $state
    ├─ ClickToCount → getContext('count') → B의 상태
    └─ ClickToCount → getContext('count') → B의 상태  ← 동기화됨

  각 Counter가 독립된 $state를 만들어 context로 제공
  → 같은 Counter 안의 자식끼리만 상태 공유
```

## 핵심: Context 자체는 반응형이 아니다

```ts
// ❌ 단순 값 전달 — 반응성 없음
setContext('count', 0)

// 자식에서:
let count = getContext('count')  // 0 (값 복사)
count++                          // 로컬 변수만 변경, 다른 자식에 영향 없음
```

```
Context에 원시값(숫자, 문자열)을 넣으면:
  → 자식이 getContext로 받는 건 "값의 복사본"
  → 자식에서 변경해도 다른 자식에 전달되지 않음
  → 해당 컴포넌트 로컬 상태처럼만 동작

반응형으로 만들려면:
  → $state 객체(Proxy)를 전달  ← 가장 간단
  → getter/setter 객체를 전달  ← 원시값 공유 시
  → 클래스 인스턴스를 전달     ← 복잡한 로직 시
```

## 대안: getter/setter 객체로 Context에 전달

원시값을 공유하거나 setter에 로직이 필요하면 getter/setter 패턴을 사용한다.

```svelte
<!-- Counter.svelte — getter/setter 버전 -->
<script lang="ts">
  import { setContext, type Snippet } from 'svelte'

  let { children }: { children: Snippet } = $props()

  let count = $state(0)  // 원시값

  setContext('count', {
    get value() { return count },     // 읽기: 반응성 유지
    set value(v: number) { count = v }, // 쓰기: 유효성 검사 삽입 가능
    increment: () => count++,
    reset: () => count = 0,
  })
</script>

{@render children()}
```

```svelte
<!-- ClickToCount.svelte -->
<script lang="ts">
  import { getContext } from 'svelte'

  const counter = getContext<{ value: number; increment: () => void; reset: () => void }>('count')
</script>

<button onclick={counter.increment}>{counter.value}</button>
```

```
$state 객체 vs getter/setter — Context에서의 선택:

  $state({ value: 0 })          getter/setter 객체
  ─────────────────────          ──────────────────
  Proxy가 자동 처리               수동으로 작성
  프로퍼티 변경으로 반응성         getter 접근으로 반응성
  코드 간결                      setter에 유효성 검사 가능
  ✅ 기본 선택                    원시값 공유 시 필요
```

> 클래스 인스턴스도 동일하게 `setContext`에 전달할 수 있다. `$state` 필드의 getter/setter가 자동 생성되므로 가장 안전한 방법이다.

## hasContext — 컨텍스트 없으면 로컬 상태로 폴백

컴포넌트가 **단독으로도** 사용되고, **Context 안에서도** 사용될 수 있다면 `hasContext`로 분기한다.

```svelte
<!-- ClickToCount.svelte -->
<script lang="ts">
  import { getContext, hasContext } from 'svelte'

  // Context가 있는지 확인 — 반드시 스크립트 최상위에서 호출
  const hasCountContext = hasContext('count')

  // 로컬 상태 생성 함수 — Context와 동일한 인터페이스
  function createLocalState() {
    let count = $state(0)
    return {
      get value() { return count },
      increment: () => count++,
      reset: () => count = 0,
    }
  }

  // Context가 있으면 공유 상태, 없으면 로컬 상태
  const counter = hasCountContext
    ? getContext<{ value: number; increment: () => void; reset: () => void }>('count')
    : createLocalState()
</script>

<button onclick={counter.increment}>{counter.value}</button>
<button onclick={counter.reset}>리셋</button>
```

```svelte
<!-- App.svelte -->
<ClickToCount />       <!-- 단독 사용: 로컬 상태 (독립) -->

<Counter>              <!-- Counter 안: 공유 상태 -->
  <ClickToCount />
  <ClickToCount />     <!-- 이 둘은 동기화됨 -->
</Counter>

<Counter>
  <ClickToCount />
  <ClickToCount />     <!-- 이 둘도 동기화됨, 위 그룹과는 독립 -->
</Counter>
```

```
hasContext 동작:

  ClickToCount (단독)
    → hasContext('count') = false
    → createLocalState() → 로컬 $state 사용
    → 다른 컴포넌트와 상태 공유 없음

  Counter > ClickToCount
    → hasContext('count') = true
    → getContext('count') → 부모의 $state 사용
    → 같은 Counter 안의 자식끼리 동기화

핵심: 로컬 상태와 공유 상태의 인터페이스를 동일하게 유지
  → counter.value, counter.increment, counter.reset
  → 마크업 코드를 분기할 필요 없음
```

## Context API 제약사항

```
1. setContext / getContext / hasContext는 스크립트 최상위에서만 호출 가능
   ✅ <script> 최상위
   ❌ 함수 내부, 콜백 내부, $effect 내부

2. getAllContexts() — 부모가 설정한 모든 컨텍스트를 Map으로 반환
   const allCtx = getAllContexts()
   // Map { 'count' => { value: 0, ... }, 'theme' => { ... } }
   // Context가 없으면 빈 Map

3. Context는 "한 방향"
   → 부모 → 자식으로만 전달 (React와 동일)
   → 자식이 부모에게 context로 값을 올릴 수 없음
```

## Context 캡슐화 — 별도 파일로 추출

위 예제에는 반복과 취약점이 있다:

```
문제 1: 타입 중복
  Counter.svelte   → setContext<CounterContext>('count', ...)
  ClickToCount.svelte → getContext<CounterContext>('count')
  → 같은 타입을 두 곳에서 정의

문제 2: 하드코딩된 문자열 키
  setContext('count', ...)
  getContext('count')
  → 오타 시 런타임 에러. 여러 Context가 같은 키를 쓰면 충돌.

문제 3: 상태 생성 로직 중복
  Counter.svelte      → { count, increment, reset } 객체 생성
  createLocalState()  → 동일한 구조의 객체 생성
  → 인터페이스가 같은 코드를 두 번 작성
```

### 해결: 전용 파일에 캡슐화

```ts
// lib/context/counter-context.svelte.ts

import { setContext, getContext, hasContext } from 'svelte'

// 1. 타입을 한 곳에서 정의
type CounterContext = {
  value: number
  increment: () => void
  reset: () => void
}

// 2. Symbol 키 — 고유하므로 충돌 불가
const key = Symbol('counter')

// 3. 상태 생성 함수 — Context용/로컬용 공통
export function createCounterState(initial = 0) {
  let count = $state(initial)
  return {
    get value() { return count },
    increment: () => count++,
    reset: () => count = 0,
  }
}

// 4. 래퍼 함수 — 키와 타입을 내부에서 처리
export function setCounterContext(ctx: CounterContext) {
  setContext(key, ctx)
}

export function getCounterContext(): CounterContext {
  return getContext<CounterContext>(key)
}

export function hasCounterContext(): boolean {
  return hasContext(key)
}
```

### 사용: 반복 제거된 컴포넌트

```svelte
<!-- Counter.svelte -->
<script lang="ts">
  import { type Snippet } from 'svelte'
  import { createCounterState, setCounterContext } from '$lib/context/counter-context.svelte.ts'

  let { children, initialCount = 0 }: { children: Snippet; initialCount?: number } = $props()

  // initialCount는 초기값으로만 사용 — prop 변경에 반응할 필요 없음
  setCounterContext(createCounterState(initialCount))
  // 타입 지정 불필요 — setCounterContext가 내부에서 처리
  // 키 지정 불필요 — Symbol이 내부에서 처리
</script>

{@render children()}
```

```svelte
<!-- ClickToCount.svelte -->
<script lang="ts">
  import {
    getCounterContext,
    hasCounterContext,
    createCounterState
  } from '$lib/context/counter-context.svelte.ts'

  const counter = hasCounterContext()
    ? getCounterContext()    // 타입 자동 추론
    : createCounterState()  // 동일한 인터페이스
</script>

<button onclick={counter.increment}>{counter.value}</button>
<button onclick={counter.reset}>리셋</button>
```

```svelte
<!-- App.svelte -->
<Counter initialCount={4}>
  <ClickToCount />   <!-- 초기값 4에서 시작, 공유 상태 -->
  <ClickToCount />
</Counter>

<ClickToCount />     <!-- 단독 사용: 0에서 시작, 로컬 상태 -->
```

### Symbol 키가 문자열보다 나은 이유

```
문자열 키:
  setContext('count', ...)   // 컴포넌트 A
  setContext('count', ...)   // 컴포넌트 B → 키 충돌!

Symbol 키:
  const keyA = Symbol('count')  // 컴포넌트 A 전용
  const keyB = Symbol('count')  // 컴포넌트 B 전용
  keyA === keyB  // false — 설명이 같아도 다른 Symbol

  Symbol은 생성할 때마다 고유한 값이 된다.
  같은 설명 문자열을 넘겨도 === 비교 시 false.
  → 여러 Context가 있어도 키 충돌 불가능
```

### 리팩토링 전후 비교

```
Before:                              After:
──────                               ─────
Counter.svelte                       counter-context.svelte.ts
  타입 정의 (중복)                     타입 정의 (1곳)
  키 'count' (하드코딩)                Symbol 키 (고유)
  상태 생성 로직                       createCounterState() (공유)
                                      setCounterContext() (래퍼)
ClickToCount.svelte                   getCounterContext() (래퍼)
  타입 정의 (중복)                     hasCounterContext() (래퍼)
  키 'count' (하드코딩)
  createLocalState() (중복)          Counter.svelte
                                      setCounterContext(createCounterState())

                                     ClickToCount.svelte
                                      hasCounterContext() ? get... : create...
```

---

## 핵심 정리

| 패턴 | 용도 | 주의점 |
|------|------|--------|
| Context API | SSR 안전 / 트리 범위 공유 | 반응형 값 전달 필수, 스크립트 최상위에서만 호출 |
| `hasContext` 폴백 | 단독/Context 양쪽 사용 가능 | 로컬 상태와 동일한 인터페이스 유지 |
| Context 캡슐화 파일 | 타입·키·상태 생성 일원화 | Symbol 키로 충돌 방지, `.svelte.ts` 필요 |
| `$state` 객체 전달 | Context에서 반응형 공유 (기본 선택) | Proxy 전달, 프로퍼티 변경으로 반응성 |
| getter/setter 전달 | 원시값 공유 / setter 유효성 검사 | 수동 작성 필요, setter에 로직 삽입 가능 |

> **중요 — Svelte 5.40+ 권장 패턴**: `createContext`를 사용하면 타입 안전성이 보장되고 키 충돌을 원천 차단할 수 있다. 새 코드에서는 `createContext`를 기본으로 사용하고, `setContext/getContext`는 원리 학습용 또는 레거시 코드에서만 사용한다. `createContext` 사용법은 [11-reactive-classes.md](11-reactive-classes.md)의 Context API 섹션 참조.

### createContext 방식으로 위 캡슐화를 더 간결하게 (5.40+)

```ts
// lib/context/counter-context.svelte.ts (createContext 버전)
import { createContext } from 'svelte'

interface CounterContext {
  value: number
  increment: () => void
  reset: () => void
}

// createContext가 타입·키를 모두 처리 — Symbol 키 수동 생성 불필요
export const [getCounterContext, setCounterContext] = createContext<CounterContext>()

export function createCounterState(initial = 0) {
  let count = $state(initial)
  return {
    get value() { return count },
    increment: () => count++,
    reset: () => count = 0,
  }
}
```

```svelte
<!-- Counter.svelte (createContext 버전) -->
<script lang="ts">
  import { type Snippet } from 'svelte'
  import { setCounterContext, createCounterState } from '$lib/context/counter-context.svelte.ts'

  let { children, initialCount = 0 }: { children: Snippet; initialCount?: number } = $props()
  setCounterContext(createCounterState(initialCount))
</script>

{@render children()}
```

> 아래 예제들은 `setContext/getContext`를 직접 사용하여 원리를 이해하기 위한 것이다.
