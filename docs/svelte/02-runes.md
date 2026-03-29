# Runes

Svelte 5에서 반응성을 다루는 특수 문법. 컴파일러가 인식하는 키워드다.

---

## $state

읽기/쓰기 가능한 반응형 값.

```js
let count = $state(0)
let user = $state({ name: 'jh', age: 20 })

count++         // 변경 감지
user.name = 'kim' // 객체 프로퍼티도 감지
```

### 주의: 구조분해하면 반응성 끊김

```js
let user = $state({ name: 'jh' })

const { name } = user  // name은 그냥 string → 반응성 없음
user.name              // 직접 접근해야 추적됨
```

### $state.raw — 재할당 전용 큰 객체

`$state`는 객체/배열을 깊게 프록시한다 (깊은 반응성 = 오버헤드).
재할당만 하고 프로퍼티 뮤테이션은 안 하는 경우 `$state.raw`가 낫다.

```js
// API 응답 같은 큰 객체 — 통째로 교체만 함
let data = $state.raw(null)

async function load() {
  data = await fetch('/api/data').then(r => r.json())  // 재할당 → 감지
  // data.items.push(...)  // ✗ 뮤테이션은 감지 안됨
}
```

---

## $derived

다른 상태를 기반으로 계산된 값. 쓰기는 가능하지만 expression이 바뀌면 재계산으로 덮어써진다.

```js
let firstName = $state('jh')
let lastName  = $state('kim')
let fullName  = $derived(firstName + ' ' + lastName)
```

- 의존하는 상태가 변경되면 자동 재계산
- **동기적으로 실행** — 상태 변경 직후 즉시 최신값 반영
- 순수 함수여야 함 (부작용 금지)

### $derived.by — 복잡한 표현식

`$derived`는 표현식(expression)을 받는다. 표현식이 복잡하면 `$derived.by`로 함수를 넘긴다.

```js
// $derived — 단순 표현식
let fullName = $derived(firstName + ' ' + lastName)

// $derived.by — 복잡한 로직
let result = $derived.by(() => {
  const items = list.filter(x => x.active)
  return items.map(x => x.value).join(', ')
})
```

### bind:value에 $derived 쓰면 안되는 이유

```svelte
<input bind:value={fullName} />  <!-- 잘못된 사용 -->
```

- 타이핑하면 `fullName`에 값이 쓰이지만, 다음 reactive 업데이트 시 `$derived` expression 재계산으로 덮어써짐
- `bind:value`는 `$state`에만 사용할 것

---

## $effect

상태 변화에 반응해 부작용을 실행한다. DOM 직접 조작, 네트워크 요청 등.

```js
$effect(() => {
  canvas.draw(data)      // DOM 직접 조작
  fetch('/api/log', ...) // 네트워크 요청
})
```

### 실행 타이밍 — $derived와 차이

```js
let num = $state(5)
let doubled = $derived(num * 2)  // 동기 실행
let doubledState = $state()

$effect(() => {
  doubledState = num * 2  // 마이크로태스크 큐 → 나중에 실행
})

num = 10
console.log(doubled)      // 20 ✓ 즉시 반영
console.log(doubledState) // 10 ✗ 아직 이전 값
```

### 주의사항

#### 1. 상태 동기화에 $effect 쓰지 말 것

```js
// ✗ 잘못된 패턴
let doubled = $state()
$effect(() => { doubled = num * 2 })

// ✓ 올바른 패턴
let doubled = $derived(num * 2)
```

**2. 무한루프 주의** (React useEffect와 동일한 문제)

```js
let count = $state(0)

$effect(() => {
  count++  // count 읽고 → count 씀 → effect 재실행 → 무한루프
})
```

루프 없이 상태 읽으려면 `untrack` 사용:

```js
import { untrack } from 'svelte'

$effect(() => {
  console.log(trigger)           // 이건 추적
  untrack(() => count++)         // 이건 추적 안함 → 루프 없음
})
```

#### 3. 클린업 필수

```js
$effect(() => {
  const timer = setInterval(() => { ... }, 1000)
  return () => clearInterval(timer)  // 반드시 정리
})
```

### $derived vs $effect 정리

| | `$derived` | `$effect` |
| -- | -- | -- |
| 목적 | 값 계산 | 부작용 실행 |
| 순수 함수 | 필수 | 아님 |
| 실행 타이밍 | 동기 | 마이크로태스크 (비동기) |
| 반환값 | 계산된 값 | 없음 (클린업 함수 제외) |

**"상태 → 상태 동기화는 `$derived`, 그 외 부작용만 `$effect`"**

---

### untrack — 의존성 추적 제외

배열에 push할 때 배열을 읽으므로 의존성이 자동 등록된다. 의도치 않은 루프 발생 가능.

```js
let num = $state(5)
let history = $state<number[]>([])

// ✗ 무한루프 — history를 읽고(push) + 쓰므로 의존성 등록됨
$effect(() => {
  history.push(num)  // num 변경 → effect 실행 → history 변경 → effect 재실행 → ...
})
```

`untrack`으로 특정 코드를 의존성 추적에서 제외:

```js
import { untrack } from 'svelte'

$effect(() => {
  num  // num은 추적 (의존성 등록)
  untrack(() => {
    history.push(num)  // history는 추적 안함 → 루프 없음
  })
})
```

---

### $effect의 한계 — 같은 값이면 재실행 안됨

```js
// num이 5 → 5로 "변경"되면 effect 실행 안됨
// → history에 중복값이 누락될 수 있음
$effect(() => {
  num
  untrack(() => history.push(num))
})
```

이런 경우 **이벤트 핸들러에서 직접 처리하는 게 더 낫다**:

```js
// ✓ effect 없이 이벤트 핸들러에서 직접 처리
function generate() {
  num = Math.floor(Math.random() * 10)
  history.push(num)  // 값 변경 여부와 무관하게 항상 실행
}
```

이벤트 핸들러 방식의 장점:

- 동기적으로 실행 → 즉시 최신값 보장
- 값이 동일해도 항상 실행
- effect 특유의 타이밍 버그 없음

---

### $effect 사용 판단 기준

```text
$effect가 필요한 경우:
  - DOM 직접 조작 (canvas, 외부 라이브러리)
  - 구독/해제가 필요한 작업 (WebSocket, 이벤트 리스너)

$effect 없이 처리 가능한 경우:
  - 상태 → 상태 동기화  →  $derived 사용
  - 버튼 클릭 시 부수 작업  →  이벤트 핸들러에서 직접 처리
```

**effect를 쓰려 할 때 "이벤트 핸들러나 $derived로 해결 안되나?" 먼저 고민할 것.**

---

### $state 초기값에 반응형 값 사용 시 경고

```js
let num = $state(Math.random())  // 일반값 → 문제 없음

// ✗ 경고 발생 — $state 초기값에 다른 반응형 값 참조
let history = $state([num])  // num이 반응형이면 경고

// ✓ untrack으로 해결
import { untrack } from 'svelte'
let history = $state([untrack(() => num)])
```

초기값은 한 번만 사용되므로 추적할 의도가 없음을 `untrack`으로 명시.
