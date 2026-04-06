# 상태 관리 아키텍처

Effect 라이프사이클, 양방향 파생, 로직 재사용(팩토리/클래스), 전역 상태, Context API, 내장 리액티브 클래스까지.

---

## 1. Effect Lifecycle 타이밍 다이어그램

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

핵심: **$effect.pre → DOM 업데이트 → $effect** 순서. 재실행 시 teardown이 먼저 호출된다.

---

## 2. `onMount` vs `$effect` 비교표

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

### `$effect.tracking()` — 추적 컨텍스트 감지

현재 코드가 추적 컨텍스트 안에서 실행 중인지 반환한다. 주로 라이브러리 작성자용.

| 위치 | 결과 | 이유 |
|--|--|--|
| `<script>` 최상위 | `false` | setup은 1회만 실행 |
| `$effect` 내부 | `true` | 의존성 변경 시 재실행 |
| 템플릿 | `true` | DOM 업데이트 필요 |
| 이벤트 핸들러 | `false` | 추적 불필요 |

---

## 3. 양방향 파생 패턴

### `$derived` 오버라이드 (5.25+, 낙관적 UI)

```js
let serverLikes = $state(10)
let likes = $derived(serverLikes)

function onLike() {
  likes = serverLikes + 1  // 즉시 오버라이드
  fetch('/api/like', { method: 'POST' })
    .then(() => serverLikes++)  // 서버 확인 → $derived 자동 복귀
}
```

적합: 낙관적 UI, 임시 편집. **부적합**: 양방향 변환(USD ↔ KRW) 같은 역계산.

### 안티패턴: `$effect` 두 개로 핑퐁

```ts
let a = $state(100)
let b = $state(0)

$effect(() => { untrack(() => { b = a * RATIO }) })   // a → b
$effect(() => { untrack(() => { a = b / RATIO }) })   // b → a
```

`$effect`는 "누가 바꿨는지" 모른다. 한쪽이 바뀌면 반대쪽도 바뀌고, 부동소수점 오차로 핑퐁이 발생한다.

### getter/setter 객체로 쓰기 가능한 파생 구현

```ts
let source = $state(100)

let target = {
  get value() { return source * RATIO },        // 읽기: 정방향
  set value(v: number) { source = v / RATIO }   // 쓰기: 역방향
}
```

```svelte
<input type="number" bind:value={source} />
<input type="number" bind:value={target.value} />
```

setter는 "사용자 입력 시에만" 호출되므로 핑퐁이 발생하지 않는다.

### 입력 유효성 검사

```svelte
<script>
  let amount = $state(1)

  let validated = {
    get value() { return amount },
    set value(v: number) { amount = Math.max(0, v) }
  }
</script>

<input type="number" bind:value={validated.value} />
```

---

## 4. 로직 재사용: 팩토리 함수 → 클래스 진화

### 팩토리 함수

```ts
// counter.svelte.ts
export function createCounter() {
  let count = $state(0)

  return {
    get value() { return count },   // ✅ getter 필수
    increment: () => count++,
    reset: () => count = 0,
  }
}
```

### 치명적 함정: 값 직접 반환 vs getter

```ts
return { value: count }          // ❌ 생성 시점의 값(0)이 고정됨
return { get value() { return count } }  // ✅ 접근 시마다 최신값
```

`$state`는 내부적으로 "신호 객체"를 생성한다. 프로퍼티에 직접 넣으면 생성 시점에 값이 복사되어 반응성이 끊긴다. getter로 감싸면 접근할 때마다 최신값을 읽는다.

### 클래스가 더 나은 이유 — 컴파일러 자동 getter/setter

```ts
class Counter {
  value = $state(0)                // 컴파일러가 getter/setter 자동 생성
  increment() { this.value++ }
  reset() { this.value = 0 }
}
```

팩토리 함수는 getter를 수동으로 만들어야 하지만, 클래스는 컴파일러가 `$state` 필드에 getter/setter를 자동 생성한다. 빠뜨릴 위험이 없다.

**결론**: 단순하면 팩토리 함수, 상태가 여러 개이거나 복잡하면 클래스.

### 리액티브 클래스 패턴

```ts
class ReactiveService {
  loading = $state(true)
  error: string | undefined = $state(undefined)

  #source = $state(0)
  get source() { return this.#source }
  set source(v: number) {
    this.#source = v < 0 ? 0 : v
    this.#target = this.#compute()
  }

  items = $state<Todo[]>([])
  doneCount = $derived(this.items.filter(i => i.done).length)
}
```

**규칙: 리액티브 객체(팩토리 반환값, 클래스 인스턴스)는 절대 구조분해하지 않는다.**

```ts
const { loading } = svc  // ❌ 값 복사 → 반응성 끊김
svc.loading               // ✅ getter 호출 → 반응성 유지
```

---

## 5. `.svelte.ts` 파일 확장자

컴포넌트 밖에서 `$state`, `$derived`, `$effect` 등 rune을 사용하려면 `.svelte.ts` 확장자가 필수.

```
❌ my-service.ts           → rune 컴파일 안 됨
✅ my-service.svelte.ts    → Svelte 컴파일러가 rune 처리
```

---

## 6. 전역 상태 3가지 방법

### 방법 1: `$state` 객체 (기본 선택)

```ts
// counter.svelte.ts
export const counter = $state({ count: 0 })
export function increment() { counter.count++ }
```

`$state`에 객체를 넘기면 Proxy로 감싸져 프로퍼티 접근이 곧 반응성. 공식 문서 권장 방식.

### 방법 2: 팩토리 함수 (원시값 공유)

```ts
let count = $state(0)

export default {
  get value() { return count },
  set value(v) { count = v },
  increment: () => count++,
}
```

원시값(`number`, `string`)은 Proxy 불가하므로 getter/setter로 감싸야 한다.

### 방법 3: 클래스 싱글톤 (복잡한 로직)

```ts
class Counter {
  count = $state(0)
  increment() { this.count++ }
  reset() { this.count = 0 }
}
export default new Counter()  // 인스턴스 export → 싱글톤
```

### 선택 가이드

```
"전역 상태가 필요하다"
  │
  ├─ 상태가 단순? → $state 객체 ✅ 기본 선택
  ├─ 원시값 공유 or setter에 로직? → 팩토리 함수
  ├─ 복잡한 로직 캡슐화? → 클래스 싱글톤
  └─ SSR + 사용자별 데이터? → Context API (아래 참조)
```

---

## 7. `$effect.root` — 컴포넌트 밖 effect

모듈 레벨에서 `$effect`를 직접 쓰면 "effect_orphan" 에러가 발생한다. `$effect.root`로 감싸야 한다.

```ts
// counter.svelte.ts
export const count = $state({ value: 0 })

$effect.root(() => {
  $effect(() => {
    localStorage.setItem('count', JSON.stringify(count))
  })
})
```

컴포넌트 안에서는 Svelte가 자동으로 루트 effect를 생성하므로 `$effect`를 바로 쓸 수 있다. 모듈 레벨에는 컴포넌트가 없으므로 직접 루트를 만들어야 한다.

---

## 8. SSR 안전성 경고 (모듈 상태 vs Context API)

```
모듈 레벨 상태는 서버에서 위험하다.

클라이언트: 브라우저 탭마다 독립된 모듈 인스턴스 → 문제 없음
서버(SSR):  하나의 프로세스가 모든 요청 처리 → 모듈 상태가 사용자 간 공유됨!

  사용자 A → count.value = 5
  사용자 B → count.value가 5로 보임 ← 데이터 유출!

해결: SSR 안전이 필요하면 → Context API (요청별 독립된 트리)
```

---

## 9. Context API

### `createContext` (5.40+, 권장)

```ts
// lib/context/counter.svelte.ts
import { createContext } from 'svelte'

interface CounterContext {
  value: number
  increment: () => void
  reset: () => void
}

export const [getCounter, setCounter] = createContext<CounterContext>()

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
<!-- Parent.svelte -->
<script lang="ts">
  import { setCounter, createCounterState } from '$lib/context/counter.svelte.ts'
  setCounter(createCounterState())
</script>
{@render children()}
```

```svelte
<!-- Child.svelte -->
<script lang="ts">
  import { getCounter } from '$lib/context/counter.svelte.ts'
  const counter = getCounter()
</script>
<button onclick={counter.increment}>{counter.value}</button>
```

### `setContext` / `getContext` / `hasContext`

5.40 이전 또는 수동 키 관리가 필요할 때:

```ts
import { setContext, getContext, hasContext } from 'svelte'

const key = Symbol('counter')  // Symbol로 충돌 방지

export function setCounterContext(ctx: CounterContext) {
  setContext(key, ctx)
}
export function getCounterContext(): CounterContext {
  return getContext<CounterContext>(key)
}
```

### 반응형 값 전달 (원시값 ❌, `$state` 객체 ✅)

```ts
setContext('count', 0)                     // ❌ 값 복사 → 반응성 없음
setContext('count', $state({ value: 0 }))  // ✅ Proxy → 반응성 유지
```

Context에 원시값을 넣으면 자식이 받는 건 "값의 복사본"이다. `$state` 객체(Proxy)를 전달해야 프로퍼티 변경이 감지된다.

### `hasContext` 폴백 패턴

컴포넌트가 단독으로도, Context 안에서도 사용될 수 있을 때:

```svelte
<script lang="ts">
  import { getCounterContext, hasCounterContext, createCounterState } from '$lib/context/counter.svelte.ts'

  const counter = hasCounterContext()
    ? getCounterContext()     // 공유 상태
    : createCounterState()   // 로컬 상태 (동일 인터페이스)
</script>
```

### Context 제약사항

- `setContext` / `getContext` / `hasContext`는 **스크립트 최상위에서만** 호출 가능
- 부모 → 자식 한 방향만 전달 가능

---

## 10. 내장 리액티브 클래스

일반 JS 내장 객체(Date, Map, Set, URL)는 "내부 슬롯"에 데이터를 저장하므로 `$state`의 Proxy가 메서드 호출을 감지하지 못한다. Svelte가 리액티브 버전을 제공한다.

### `SvelteDate`, `SvelteMap`, `SvelteSet`, `SvelteURL`

```ts
import { SvelteDate, SvelteMap, SvelteSet, SvelteURL } from 'svelte/reactivity'

let date = new SvelteDate()
date.setSeconds(date.getSeconds() + 1)  // ✅ UI 자동 갱신

let map = new SvelteMap()
map.set('key', 'value')                 // ✅ 반응성 동작

let set = new SvelteSet()
set.add('item')                         // ✅ 반응성 동작

let url = new SvelteURL('https://example.com')
url.pathname = '/new'                   // ✅ href 자동 갱신
```

### `MediaQuery`

```ts
import { MediaQuery } from 'svelte/reactivity'

const large = new MediaQuery('min-width: 800px')
```

```svelte
{#if large.current}
  <p>큰 화면</p>
{:else}
  <p>작은 화면</p>
{/if}
```

### `Tween`, `Spring` (`svelte/motion`)

```ts
import { Tween, Spring } from 'svelte/motion'

let progress = new Tween(0, { duration: 400, easing: cubicOut })
progress.target = 1  // current가 0→1로 부드럽게 전환

let coords = new Spring({ x: 0, y: 0 }, { stiffness: 0.1, damping: 0.5 })
coords.target = { x: 100, y: 200 }  // 스프링 물리로 이동
```

| | Tween | Spring |
|--|--|--|
| 방식 | 시간 기반 (duration) | 물리 기반 (stiffness) |
| 느낌 | 예측 가능한 전환 | 자연스러운 바운스 |
| 적합 | 프로그레스 바, 페이드 | 드래그, 좌표 이동 |

### `createSubscriber` — 외부 이벤트 반응성 연결

브라우저 API(스크롤, WebSocket 등)를 Svelte 반응성에 연결한다.

```ts
import { createSubscriber } from 'svelte/reactivity'
import { on } from 'svelte/events'

const sub = createSubscriber((update) => {
  const off = on(window, 'scroll', update)
  return () => off()
})
```

```svelte
<!-- sub()을 호출하면 추적 컨텍스트에서 구독이 등록됨 -->
<h1>{sub(), window.scrollY}</h1>
```

내부적으로 숨겨진 `$state(version)`을 사용하여, `update()` 호출 시 version을 증가시키고 의존하는 컨텍스트를 재실행한다.

실무에서는 Svelte가 이미 만들어둔 것을 사용:

```ts
import { scrollY, innerWidth } from 'svelte/reactivity/window'
```

```svelte
<p>scrollY: {scrollY.current}</p>
```

---

## 11. 언제 무엇을 선택할 것인가 (판단 트리)

```
"상태를 어떻게 관리할 것인가?"
  │
  ├─ 컴포넌트 로컬?
  │   └─ $state + $derived (기본)
  │
  ├─ 여러 컴포넌트에서 재사용?
  │   ├─ 독립 인스턴스 필요 → 팩토리 함수 / 클래스 (new 할 때마다 별개)
  │   └─ 전역 공유 필요 → 아래 분기
  │
  ├─ 전역 공유?
  │   ├─ 클라이언트 전용 → $state 객체 export (기본)
  │   ├─ 복잡한 로직 → 클래스 싱글톤
  │   └─ SSR 안전 필요 → Context API
  │
  └─ 트리 범위 공유? (모달 내부, 폼 그룹)
      └─ Context API + createContext (5.40+)
```

| 시나리오 | 추천 방법 |
|---------|----------|
| 컴포넌트 내부 상태 | `$state` + `$derived` |
| 로직 재사용 (독립) | 클래스 / 팩토리 함수 |
| 클라이언트 전역 (테마, 장바구니) | `$state` 객체 export |
| SSR + 사용자별 데이터 | Context API |
| 트리 범위 DI | Context API |
| 외부 이벤트 연결 | `createSubscriber` |
| Date/Map/Set 반응형 | 내장 리액티브 클래스 |

---

## 실무 예시: 데이터 테이블 (정렬 + 페이지네이션 + 로딩)

상태를 클래스로 캡슐화하여 여러 컴포넌트에서 재사용하는 패턴.

```ts
// lib/table-state.svelte.ts
export class TableState<T> {
  items = $state<T[]>([])
  loading = $state(false)
  error: string | undefined = $state(undefined)

  #sortKey = $state<keyof T | null>(null)
  #sortAsc = $state(true)
  #page = $state(1)
  #perPage: number

  constructor(perPage = 20) {
    this.#perPage = perPage
  }

  // 정렬
  get sortKey() { return this.#sortKey }
  get sortAsc() { return this.#sortAsc }

  sort(key: keyof T) {
    if (this.#sortKey === key) {
      this.#sortAsc = !this.#sortAsc
    } else {
      this.#sortKey = key
      this.#sortAsc = true
    }
  }

  sorted = $derived.by(() => {
    if (!this.#sortKey) return this.items
    const key = this.#sortKey
    const dir = this.#sortAsc ? 1 : -1
    return [...this.items].sort((a, b) => {
      const av = a[key], bv = b[key]
      if (av < bv) return -dir
      if (av > bv) return dir
      return 0
    })
  })

  // 페이지네이션
  get page() { return this.#page }
  get totalPages() { return Math.ceil(this.sorted.length / this.#perPage) }

  paged = $derived.by(() => {
    const start = (this.#page - 1) * this.#perPage
    return this.sorted.slice(start, start + this.#perPage)
  })

  goTo(page: number) {
    this.#page = Math.max(1, Math.min(page, this.totalPages))
  }

  // 데이터 로드
  async load(fetcher: () => Promise<T[]>) {
    this.loading = true
    this.error = undefined
    try {
      this.items = await fetcher()
      this.#page = 1
    } catch (e) {
      this.error = e instanceof Error ? e.message : '알 수 없는 오류'
    } finally {
      this.loading = false
    }
  }
}
```

```svelte
<!-- DataTable.svelte -->
<script lang="ts">
  import { TableState } from '$lib/table-state.svelte.ts'

  type User = { id: number; name: string; email: string }
  const table = new TableState<User>(10)

  import { onMount } from 'svelte'

  onMount(() => {
    table.load(() => fetch('/api/users').then(r => r.json()))
  })
</script>

{#if table.loading}
  <p>로딩 중...</p>
{:else if table.error}
  <p>에러: {table.error}</p>
{:else}
  <table>
    <thead>
      <tr>
        <th onclick={() => table.sort('name')}>
          이름 {table.sortKey === 'name' ? (table.sortAsc ? '▲' : '▼') : ''}
        </th>
        <th onclick={() => table.sort('email')}>
          이메일 {table.sortKey === 'email' ? (table.sortAsc ? '▲' : '▼') : ''}
        </th>
      </tr>
    </thead>
    <tbody>
      {#each table.paged as user (user.id)}
        <tr>
          <td>{user.name}</td>
          <td>{user.email}</td>
        </tr>
      {/each}
    </tbody>
  </table>

  <nav>
    <button onclick={() => table.goTo(table.page - 1)} disabled={table.page <= 1}>이전</button>
    <span>{table.page} / {table.totalPages}</span>
    <button onclick={() => table.goTo(table.page + 1)} disabled={table.page >= table.totalPages}>다음</button>
  </nav>
{/if}
```
