# 안티패턴 & AI 코드리뷰 가이드

AI가 생성한 Svelte 코드를 리뷰하고, 흔한 안티패턴을 식별하여 수정하는 가이드.

---

## 1. AI 코드리뷰 프롬프트 템플릿

아래 텍스트를 복사하여 Claude에 붙여넣으면 Svelte 5 모범 사례 관점에서 코드리뷰를 받을 수 있다.

```
아래 Svelte 코드를 Svelte 5 모범 사례 관점에서 리뷰해 주세요.
체크할 항목:
1. Svelte 4 레거시 문법 (on:click, export let, store, use:action 등)
2. $effect로 상태 동기화 (→ $derived로 대체해야 함)
3. props 의존 값을 $derived 없이 선언
4. 리액티브 객체/클래스 인스턴스 구조분해
5. .ts 파일에서 rune 사용 (.svelte.ts 필요)
6. SSR 위험한 모듈 레벨 상태
7. $effect에 async 직접 사용
8. Action 옵션을 객체로 직접 전달 (함수 래핑 필요)
9. Context vs 모듈 상태 선택의 적절성
10. 0 falsy 버그 (|| 대신 ?? 사용)
11. $bindable 없이 자식이 부모 상태 변경
12. 일반 Map/Set을 $state로 래핑 (SvelteMap/SvelteSet 필요)

이슈 발견 시: 문제 설명 + 수정 코드를 제시해 주세요.
```

---

## 2. CLAUDE.md 추가용 Svelte 코딩 규칙

아래 텍스트를 `CLAUDE.md`에 추가하면 AI가 Svelte 5 코드를 작성할 때 모범 사례를 따른다.

```
## Svelte 5 코딩 규칙

### 반응성
- props 의존 값은 반드시 $derived로 선언 (스크립트는 한 번만 실행됨)
- 상태→상태 동기화에 $effect 사용 금지 → $derived 또는 getter/setter로 해결
- $effect는 외부 시스템 연동(DOM 라이브러리, WebSocket, analytics) 전용
- $effect에 async 직접 사용 금지 → 별도 async 함수를 effect 내에서 호출
- 리액티브 객체/클래스 인스턴스는 절대 구조분해 금지

### 파일 규칙
- rune ($state, $derived, $effect) 사용 시 .svelte.ts 확장자 필수
- Svelte 5 문법만 사용 (on:click → onclick, export let → $props(), slot → snippet)

### 상태 관리
- 전역 상태: $state 객체 export (기본) / 클래스 싱글톤 (복잡한 로직)
- SSR 환경: 모듈 레벨 상태 대신 Context API 사용
- Date/Map/Set: SvelteDate, SvelteMap, SvelteSet 사용

### 컴포넌트
- 자식이 부모 상태 변경 시 $bindable 필수
- DOM 동작 부착: {@attach} 권장 (5.29+), use: Action은 레거시
- 기본값 처리: || 대신 ?? 사용 (0 falsy 버그 방지)
- rest props로 네이티브 속성 전달 (HTMLButtonAttributes 등)

### SvelteKit
- 데이터 로딩: +page.server.ts의 load 함수 사용
- 폼 처리: Form Actions + use:enhance (progressive enhancement)
- 인증: hooks.server.ts에서 event.locals로 전파
```

---

## 3. 안티패턴 카탈로그 (Bad → Good)

### AP-01: `$effect`로 상태 동기화 → `$derived`

```svelte
<!-- ❌ Bad -->
<script>
  let { count } = $props()
  let doubled = $state(0)
  $effect(() => { doubled = count * 2 })
</script>
```

`$effect`로 상태를 다른 상태에 동기화하면 불필요한 비동기 지연이 발생하고, 복잡한 경우 무한루프 위험이 있다.

```svelte
<!-- ✅ Good -->
<script>
  let { count } = $props()
  let doubled = $derived(count * 2)
</script>
```

---

### AP-02: props 의존 파생값을 일반 변수로 선언

```svelte
<!-- ❌ Bad -->
<script>
  let { type } = $props()
  let color = type === 'danger' ? 'red' : 'green'  // 초기값에 고정됨
</script>
```

Svelte 스크립트는 한 번만 실행된다. props가 변경되어도 재계산되지 않는다.

```svelte
<!-- ✅ Good -->
<script>
  let { type } = $props()
  let color = $derived(type === 'danger' ? 'red' : 'green')
</script>
```

---

### AP-03: 리액티브 객체 구조분해

```ts
// ❌ Bad — 값 복사 → 반응성 끊김
const svc = new ReactiveService()
const { loading, error } = svc
```

구조분해하면 getter와의 연결이 끊기고 초기값이 복사된다.

```ts
// ✅ Good — 항상 인스턴스를 통해 접근
const svc = new ReactiveService()
svc.loading   // getter 호출 → 반응성 유지
svc.error
```

`$state` 객체, 팩토리 반환값, 클래스 인스턴스 모두 동일한 규칙.

---

### AP-04: Svelte 4 레거시 이벤트 핸들러

```svelte
<!-- ❌ Bad -->
<button on:click={() => count++}>클릭</button>
```

```svelte
<!-- ✅ Good -->
<button onclick={() => count++}>클릭</button>
```

---

### AP-05: `export let` → `$props()`

```svelte
<!-- ❌ Bad (Svelte 4) -->
<script>
  export let label = ''
  export let count = 0
</script>
```

```svelte
<!-- ✅ Good (Svelte 5) -->
<script>
  let { label = '', count = 0 } = $props()
</script>
```

---

### AP-06: `writable` store → `$state` 클래스

```ts
// ❌ Bad (Svelte 4 store)
import { writable, derived } from 'svelte/store'
const count = writable(0)
const doubled = derived(count, $c => $c * 2)
```

```ts
// ✅ Good (Svelte 5 클래스)
class Counter {
  count = $state(0)
  doubled = $derived(this.count * 2)
  increment() { this.count++ }
}
```

---

### AP-07: `.ts` 파일에서 rune 사용

```ts
// ❌ Bad — counter.ts
let count = $state(0)  // 컴파일 안 됨
```

```ts
// ✅ Good — counter.svelte.ts
let count = $state(0)  // Svelte 컴파일러가 처리
```

---

### AP-08: 모듈 레벨 상태를 SSR에서 사용

```ts
// ❌ Bad — 서버에서 모든 요청이 같은 상태를 공유
// user-store.svelte.ts
export const user = $state({ name: '', role: '' })
```

서버에서는 모든 요청이 같은 모듈 인스턴스를 공유하므로 사용자 간 데이터가 유출될 수 있다.

```ts
// ✅ Good — Context API로 요청별 격리
// user-context.svelte.ts
import { createContext } from 'svelte'

interface User { name: string; role: string }
export const [getUser, setUser] = createContext<User>()
```

---

### AP-09: `$effect(async () => ...)`

```ts
// ❌ Bad — Promise 반환 → 클린업 패턴 동작 안 함
$effect(async () => {
  const data = await fetch(url).then(r => r.json())
  items = data
})
```

```ts
// ✅ Good — 별도 async 함수 호출
$effect(() => {
  async function fetchData() {
    const data = await fetch(url).then(r => r.json())
    items = data
  }
  fetchData()
})
```

---

### AP-10: Action 옵션 객체 직접 전달

```svelte
<!-- ❌ Bad — 생성 시점의 값만 복사, 변경 감지 불가 -->
<button use:longpress={{ duration }}>

<!-- ✅ Good — 함수로 감싸서 반응형 추적 -->
<button use:longpress={() => ({ duration })}>
```

`$effect` 내부에서 함수를 호출해야 의존성이 추적된다.

> Svelte 5.29+에서는 `{@attach}` 사용 시 자동 반응형이므로 이 문제가 발생하지 않는다.

---

### AP-11: `||`로 기본값 처리 시 `0` falsy 버그

```ts
// ❌ Bad — duration이 0이면 1000으로 폴백
const duration = options.duration || 1000       // 0 || 1000 = 1000 (Bug!)

// ✅ Good — nullish coalescing
const duration = options.duration ?? 1000       // 0 ?? 1000 = 0 (OK)

// ✅ Good — 명시적 undefined 체크
const duration = options.duration !== undefined ? options.duration : 1000
```

`??`는 `null`과 `undefined`만 폴백. `||`는 모든 falsy(`0`, `''`, `false`)를 폴백.

---

### AP-12: `$effect` 핑퐁 (양방향 상태 동기화)

```ts
// ❌ Bad — 두 effect가 서로 트리거 → 부동소수점 오차로 핑퐁
$effect(() => { untrack(() => { b = a * RATIO }) })
$effect(() => { untrack(() => { a = b / RATIO }) })
```

```ts
// ✅ Good — getter/setter 객체
let target = {
  get value() { return source * RATIO },
  set value(v) { source = v / RATIO }
}
```

setter는 사용자 입력 시에만 호출되므로 핑퐁이 발생하지 않는다.

---

### AP-13: Context에 원시값 전달 (반응성 없음)

```ts
// ❌ Bad — 값 복사, 반응성 없음
setContext('count', 0)
```

```ts
// ✅ Good — $state 객체 전달
let count = $state({ value: 0 })
setContext('count', count)

// ✅ Better — createContext (5.40+)
export const [getCount, setCount] = createContext<{ value: number }>()
```

---

### AP-14: `$bindable` 없이 자식이 부모 상태 직접 변경

```svelte
<!-- ❌ Bad — ownership 경고 발생 -->
<!-- Child.svelte -->
<script>
  let { value } = $props()
</script>
<input bind:value />  <!-- 부모 소유 상태를 변경 → 경고 -->
```

```svelte
<!-- ✅ Good -->
<!-- Child.svelte -->
<script>
  let { value = $bindable('') } = $props()
</script>
<input bind:value />
```

```svelte
<!-- Parent.svelte -->
<Child bind:value={myValue} />
```

---

### AP-15: `SvelteMap`/`SvelteSet` 대신 일반 Map/Set을 `$state`로 래핑

```ts
// ❌ Bad — Map의 .set()은 내부 슬롯 변경 → Proxy 감지 불가
let map = $state(new Map())
map.set('key', 'value')  // UI 갱신 안 됨
```

```ts
// ✅ Good
import { SvelteMap } from 'svelte/reactivity'

let map = new SvelteMap()
map.set('key', 'value')  // UI 자동 갱신
```

Date, Map, Set, URL은 내부 슬롯에 데이터를 저장하므로 `$state`의 Proxy가 메서드 호출을 감지할 수 없다. Svelte가 제공하는 리액티브 버전을 사용해야 한다.
