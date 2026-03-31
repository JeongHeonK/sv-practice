# Runes — 반응성 API

Svelte 5의 반응성 문법. 컴파일러가 인식하는 키워드다.

---

## $state — React의 useState

읽기/쓰기 가능한 반응형 값. **setter 함수 없이 직접 변경**한다.

```js
// React
const [count, setCount] = useState(0)
setCount(count + 1)

// Svelte
let count = $state(0)
count++  // 직접 변경 → 반응성 동작
```

### 객체/배열 — 깊은 반응성 (Proxy)

`$state`에 객체/배열을 넘기면 내부적으로 `Proxy`로 감싼다. 중첩 프로퍼티까지 자동 추적.

```js
let user = $state({ name: 'jh', age: 20 })
user.name = 'kim'  // → Proxy set 트랩 → 의존성 재실행
```

React에서는 `setUser({ ...user, name: 'kim' })`처럼 새 객체를 만들어야 하지만, Svelte는 직접 변경해도 감지된다.

**세분화된 반응성**: 의존성은 객체 전체가 아니라 **프로퍼티 단위**로 등록된다.

```js
let obj = $state({ name: 'jh', age: 20 })

$effect(() => { console.log(obj.name) })  // name에만 의존

obj.age = 30    // → 위 effect 재실행 안됨 (name 안 건드림)
obj.name = 'kim' // → 위 effect 재실행됨
```

### 구조분해하면 반응성 끊김

```js
let user = $state({ name: 'jh' })

const { name } = user  // name은 그냥 string → 반응성 없음
user.name               // 직접 접근해야 추적됨
```

### $state.raw — 재할당 전용

Proxy 오버헤드 없이 **재할당만** 감지. API 응답 같은 큰 객체에 적합.

```js
let data = $state.raw(null)

async function load() {
  data = await fetch('/api').then(r => r.json())  // ✓ 재할당 → 감지
  // data.items.push(...)  // ✗ 뮤테이션은 감지 안됨
}
```

**선택 기준**: 프로퍼티를 직접 변경하는가?
- YES → `$state` (Proxy, 세분화된 반응성)
- NO → `$state.raw` (재할당 전용, 가벼움)

---

## $derived — React의 useMemo

다른 상태로부터 계산된 값. **의존성 배열 없이 자동 추적**.

```js
// React
const fullName = useMemo(() => first + ' ' + last, [first, last])

// Svelte
let fullName = $derived(first + ' ' + last)
```

- 의존 상태 변경 시 자동 재계산
- **동기적 실행** — 상태 변경 직후 즉시 최신값
- 순수 함수여야 함

### $derived.by — 복잡한 표현식

`$derived`는 표현식(expression)을 받는다. 로직이 복잡하면 `$derived.by`로 함수를 넘긴다.

```js
let result = $derived.by(() => {
  const items = list.filter(x => x.active)
  return items.map(x => x.value).join(', ')
})
```

### props에 의존하는 값은 반드시 $derived

```js
let { type } = $props()

let color = $derived(type === 'danger' ? 'red' : 'green')  // ✓ type 변경 시 재계산
let color = type === 'danger' ? 'red' : 'green'            // ✗ 초기값에 고정됨
```

> React에서는 컴포넌트가 통째로 재실행되므로 `let color = ...`가 자연스럽다. Svelte에서는 스크립트가 **한 번만 실행**되므로 `$derived`로 반응성을 명시해야 한다. 이것이 가장 흔한 실수.

---

## $effect — React의 useEffect (하지만 다르다)

상태 변화에 반응해 부작용을 실행한다. **탈출구이므로 대안을 먼저 고려할 것.**

```js
$effect(() => {
  canvas.draw(data)  // 외부 라이브러리 연동
})
```

### React useEffect와 핵심 차이

| | React useEffect | Svelte $effect |
|--|--|--|
| 의존성 | 배열로 수동 명시 `[a, b]` | 자동 추적 |
| 실행 타이밍 | 렌더 후 (비동기) | DOM 업데이트 후 (비동기) |
| 값 계산 | useEffect 안에서 setState 패턴 | **$derived를 대신 써야 함** |

### $effect 쓰기 전 체크리스트

```text
상태 → 상태 동기화?            →  $derived
사용자 인터랙션 반응?           →  이벤트 핸들러
외부 DOM 라이브러리 연동?       →  {@attach ...}
외부 값 구독 (WebSocket 등)?   →  createSubscriber
디버깅 로깅?                   →  $inspect
```

**위 대안이 모두 안 될 때만 $effect를 사용한다.**

### 상태 동기화에 $effect 쓰지 말 것

```js
// ✗ React 습관 — useEffect에서 setState 하듯이
let doubled = $state()
$effect(() => { doubled = num * 2 })

// ✓ Svelte — $derived로 해결
let doubled = $derived(num * 2)
```

### 실행 타이밍

```js
let doubled = $derived(num * 2)      // 동기 — 즉시 최신값

$effect(() => { /* ... */ })         // 비동기 — DOM 업데이트 후 실행
```

### 무한루프 주의

```js
$effect(() => {
  count++  // count 읽기 + 쓰기 → effect 재실행 → 무한루프
})

// untrack으로 해결
import { untrack } from 'svelte'
$effect(() => {
  trigger              // 추적됨
  untrack(() => count++) // 추적 안함
})
```

### 클린업

```js
$effect(() => {
  const timer = setInterval(() => { ... }, 1000)
  return () => clearInterval(timer)  // effect 재실행 전 or 컴포넌트 파괴 시
})
```

React의 useEffect 클린업과 동일한 패턴이다.

---

## 디버깅 도구

### $inspect — 상태 변경 자동 로깅

개발 빌드에서만 동작. 중첩 프로퍼티 변경도 감지한다.

```js
$inspect(object)                       // 변경 시 console.log
$inspect(object).with(console.trace)   // 스택 트레이스 포함
```

### $inspect.trace() — effect 재실행 원인 추적

```js
$effect(() => {
  $inspect.trace()  // 어떤 상태 변경이 이 effect를 트리거했는지 출력
  // ...
})
```

### $state.snapshot — 프록시 벗기기

외부 라이브러리에 `$state` 객체를 넘길 때 Proxy 문제가 생기면 사용.

```js
console.log($state.snapshot(user))  // 순수 JS 객체
```
