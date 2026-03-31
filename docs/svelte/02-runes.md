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

### $state 내부 동작 — Proxy 기반 반응성

`$state`에 객체/배열을 넘기면 내부적으로 `Proxy`로 감싼다.

```js
let user = $state({ name: 'jh', age: 20 })
// 실제로는 new Proxy(target, handler) 생성

user.name = 'kim'     // → set 트랩 호출 → 의존성 재실행
console.log(user.age) // → get 트랩 호출 → 이 코드가 속한 effect/derived를 의존성으로 등록
```

- **get 트랩**: 프로퍼티를 읽을 때 현재 실행 중인 `$effect`/`$derived`를 의존성으로 등록
- **set 트랩**: 프로퍼티에 값을 쓸 때 등록된 의존성들을 재실행
- 중첩 객체도 프록시 체인으로 감싸서 깊은 반응성 제공

```js
let state = $state({ user: { address: { city: 'Seoul' } } })
state.user.address.city = 'Busan'  // 중첩 프록시 체인으로 감지됨
```

기본값(string, number, boolean)은 프록시 불가 → 변수 재할당만 감지.

### 세분화된 반응성 (Granular Reactivity)

프록시 덕분에 의존성은 **객체/배열 전체가 아닌 프로퍼티 단위**로 등록된다.

```js
let obj = $state({ name: 'jh', address: { city: 'Seoul' } })
let arr = $state([10, 20, 30])

// obj.name에만 의존하는 effect
$effect(() => { console.log(obj.name) })

obj.address.city = 'Busan'  // → 위 effect 재실행 안됨 (name 안 건드림)
obj.name = 'kim'            // → 위 effect 재실행됨

// arr[0]에만 의존하는 effect
$effect(() => { console.log(arr[0]) })

arr[1] = 99  // → 위 effect 재실행 안됨
arr[0] = 99  // → 위 effect 재실행됨
```

변경된 프로퍼티에 의존하는 effect/마크업만 재실행 → 불필요한 렌더링 없음.

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

배열에 push할 때 무한루프가 발생하는 이유는 `push` 내부 동작 때문이다.

`array.push(x)`는 내부적으로:
1. `array.length` **읽기** (get 트랩) → effect를 `length` 의존성으로 등록
2. `array[length] = x` 쓰기
3. `array.length` **쓰기** (set 트랩) → `length` 의존성인 effect 재실행

즉 push 하나가 `length`를 읽고 쓰므로, effect 안에서 push하면 자기 자신을 트리거한다.

```js
let num = $state(5)
let history = $state<number[]>([])

// ✗ 무한루프
// push → length 읽음(의존성 등록) → length 씀(effect 재실행) → push → ...
$effect(() => {
  history.push(num)
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

---

## 디버깅 유틸리티

### $state.snapshot — 프록시 없는 순수 객체

`$state` 객체는 Proxy라서 외부 API/라이브러리에 넘기면 문제가 될 수 있다.
`$state.snapshot()`으로 프록시를 벗긴 순수 JS 객체를 얻는다.

```js
let user = $state({ name: 'jh', address: { city: 'Seoul' } })

// ✗ Proxy 객체 — 외부 라이브러리에 넘기면 예상치 못한 동작
console.log(user)

// ✓ 순수 객체 스냅샷
console.log($state.snapshot(user))
```

### $inspect — 상태 변경 자동 로깅 (개발 전용)

`$effect`로 로깅하면 객체 전체에 의존해 세분화된 변경을 못 잡는다.
`$inspect`는 내부 프로퍼티 단위까지 추적해 변경될 때마다 자동으로 기록한다.

```js
// $effect 방식 — object 자체가 바뀌지 않으면 실행 안됨
$effect(() => { console.log(object) })  // object.name 변경 시 실행 안됨

// $inspect 방식 — 중첩 프로퍼티 변경도 감지
$inspect(object)  // object.name, object.address.city 등 변경 시 모두 로깅
```

- 개발 빌드에서만 실행, 프로덕션에서는 동작 안함
- 스냅샷(프록시 제거된 값)을 출력

#### .with() — 커스텀 로깅 함수

```js
// 기본: console.log
$inspect(object)

// 스택 트레이스 포함
$inspect(object).with(console.trace)

// 커스텀 처리
$inspect(object).with((type, value) => {
  if (type === 'update') sendToMonitoring(value)
})
```

### $inspect.trace() — effect 재실행 원인 추적

effect 안에서 사용하면 **어떤 상태 변경이 이 effect를 재실행시켰는지** 출력한다.

```js
$effect(() => {
  $inspect.trace()  // 재실행 원인이 된 상태를 콘솔에 강조 표시
  console.log(object.name)
  console.log(object.address.city)
})
// name 변경 시 → name이 원인임을 표시
// city 변경 시 → city가 원인임을 표시
```

### {@debug} — 마크업 중단점

마크업에서 값이 변경될 때마다 DevTools 중단점을 트리거한다.

```svelte
<!-- 특정 변수 변경 시 중단 -->
{@debug object, arr}

<!-- 인수 없이 — 컴포넌트 상태 변경 시마다 중단 -->
{@debug}
```

- DevTools가 열려 있을 때만 중단점 동작
- 콘솔에도 현재 값 출력
- 개발 시에만 사용, 커밋 전 제거할 것
