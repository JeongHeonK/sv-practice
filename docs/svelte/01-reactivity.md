# 반응성 원리

Svelte 5의 반응성 시스템 전체를 다룬다. 컴파일러 동작 원리부터 `$state`, `$derived`, `$effect`, 디버깅 도구까지.

---

## 1. Svelte는 컴파일러다

`.svelte` 파일을 빌드 시점에 바닐라 JS/CSS로 변환한다. 브라우저에 런타임 프레임워크가 로드되지 않는다.

```text
.svelte  →  컴파일러  →  .js/.css  →  브라우저 실행
```

상태가 바뀌면 Virtual DOM 비교 없이 **해당 상태에 연결된 DOM만 직접 업데이트**한다.

```text
$state 변경 → 연결된 effect만 재실행 → DOM 직접 업데이트
```

| 특성 | 설명 |
|-----|------|
| 업데이트 단위 | 변경된 상태에 연결된 곳만 |
| Virtual DOM | 없음 |
| 의존성 추적 | 자동 (수동 명시 불필요) |
| 런타임 | 컴파일된 바닐라 JS만 실행 |

> **멘탈 모델**: "상태와 DOM 사이에 전선이 연결되어 있다"고 생각하면 된다. 상태가 바뀌면 전선을 따라 연결된 DOM만 업데이트된다.

---

## 2. `$state` — 읽기/쓰기 반응형 값

읽기/쓰기 가능한 반응형 값. setter 함수 없이 **직접 변경**한다.

```js
let count = $state(0)
count++  // 직접 변경 → 반응성 동작
```

### 객체/배열 — 깊은 반응성 (Proxy)

`$state`에 객체/배열을 넘기면 내부적으로 `Proxy`로 감싼다. 중첩 프로퍼티까지 자동 추적된다.

```js
let user = $state({ name: 'jh', age: 20 })
user.name = 'kim'  // Proxy set 트랩 → 의존성 재실행
```

**세분화된 반응성**: 의존성은 객체 전체가 아니라 **프로퍼티 단위**로 등록된다.

```svelte
<script>
  let user = $state({ name: 'Park', age: 25 });
</script>

<!-- name만 의존 → name 변경 시에만 업데이트 -->
<p>{user.name}</p>

<!-- age만 의존 → age 변경 시에만 업데이트 -->
<p>{user.age}</p>
```

`user.name = 'Kim'`을 실행하면 첫 번째 `<p>`만 업데이트된다. 두 번째 `<p>`는 건드리지 않는다.

배열도 동일하게 동작한다:

```svelte
<script>
  let items = $state([
    { text: 'A', done: false },
    { text: 'B', done: false },
    { text: 'C', done: false },
  ]);
</script>

{#each items as item}
  <span class={{ done: item.done }}>{item.text}</span>
{/each}
```

`items[1].done = true` → **B 항목만** 업데이트. A, C는 무시. `useMemo`나 `memo` 같은 수동 최적화가 불필요하다.

### `$state.raw` — 재할당 전용

Proxy 오버헤드 없이 **재할당만** 감지한다. API 응답 같은 큰 읽기 전용 객체에 적합.

```js
let data = $state.raw(null)

async function load() {
  data = await fetch('/api').then(r => r.json())  // ✓ 재할당 → 감지
  // data.items.push(...)  // ✗ 뮤테이션은 감지 안됨
}
```

**선택 기준**: 프로퍼티를 직접 변경하는가?
- **YES** → `$state` (Proxy, 세분화된 반응성)
- **NO** → `$state.raw` (재할당 전용, 가벼움)

### 구조분해 시 반응성 끊김 (주의)

```js
let user = $state({ name: 'jh' })

const { name } = user  // ❌ name은 그냥 string → 반응성 없음
user.name               // ✅ 직접 접근해야 추적됨
```

구조분해하면 Proxy에서 값이 복사되어 일반 변수가 된다. Proxy와의 연결이 끊기므로 이후 변경이 감지되지 않는다.

```text
user.name 접근 시:
  → Proxy get trap → 의존성 등록 → 이후 변경 시 UI 갱신 ✅

구조분해 시:
  const { name } = user
  → Proxy get trap → 'jh' 복사 → name은 일반 string
  → 이후 user.name 변경 시 name은 여전히 'jh' ❌
```

---

## 3. `$derived` — 파생값

다른 상태로부터 계산된 값. **의존성 배열 없이 자동 추적**된다.

```js
let first = $state('길동')
let last = $state('홍')
let fullName = $derived(last + ' ' + first)  // "홍 길동"
```

- 의존 상태 변경 시 자동 재계산
- **동기적 실행** — 상태 변경 직후 즉시 최신값
- 순수 함수여야 함

### `$derived.by` — 복잡한 표현식

`$derived`는 표현식(expression)을 받는다. 로직이 복잡하면 `$derived.by`로 함수를 넘긴다.

```js
let result = $derived.by(() => {
  const items = list.filter(x => x.active)
  return items.map(x => x.value).join(', ')
})
```

### `$derived` 오버라이드 (5.25+)

직접 할당으로 임시 오버라이드 가능. 의존성 변경 시 원래 계산값으로 복귀한다. 낙관적 UI에 유용.

```js
let serverLikes = $state(10)
let likes = $derived(serverLikes)

function onLike() {
  likes = serverLikes + 1  // 즉시 11 표시 (오버라이드)
  fetch('/api/like', { method: 'POST' })
    .then(() => serverLikes++)  // 서버 확인 후 원본 업데이트 → $derived 자동 복귀
}
```

### props 의존 값은 반드시 `$derived` (핵심 규칙)

Svelte에서 `<script>` 블록은 **한 번만 실행**된다. props가 변경되어도 스크립트 최상위 코드는 재실행되지 않는다.

```js
let { type } = $props()

let color = $derived(type === 'danger' ? 'red' : 'green')  // ✅ type 변경 시 재계산
let color = type === 'danger' ? 'red' : 'green'            // ❌ 초기값에 고정됨
```

> 이것이 Svelte에서 가장 흔한 실수다. props에 의존하는 모든 계산은 `$derived`로 감싸야 한다.

---

## 4. `$effect` — 부작용 (탈출구)

상태 변화에 반응해 부작용을 실행한다. **탈출구이므로 대안을 먼저 고려할 것.**

```js
$effect(() => {
  canvas.draw(data)  // 외부 라이브러리 연동
})
```

### 쓰기 전 체크리스트

```text
상태 → 상태 동기화?            →  $derived
사용자 인터랙션 반응?           →  이벤트 핸들러
외부 DOM 라이브러리 연동?       →  {@attach ...}
외부 값 구독 (WebSocket 등)?   →  createSubscriber
디버깅 로깅?                   →  $inspect
```

**위 대안이 모두 안 될 때만 `$effect`를 사용한다.**

### 상태 동기화에 `$effect` 쓰지 말 것

```js
// ❌ effect로 상태 동기화
let doubled = $state(0)
$effect(() => { doubled = num * 2 })

// ✅ $derived로 해결
let doubled = $derived(num * 2)
```

### 무한루프 주의 + `untrack`

`$effect` 안에서 읽고 쓰는 상태가 같으면 무한루프가 발생한다.

```js
// ❌ 무한루프
$effect(() => {
  count++  // count 읽기 + 쓰기 → effect 재실행 → 반복
})

// ✅ untrack으로 해결
import { untrack } from 'svelte'
$effect(() => {
  trigger              // 추적됨
  untrack(() => count++) // 추적 안함
})
```

### 클린업 (return 함수)

```js
$effect(() => {
  const timer = setInterval(() => tick(), 1000)
  return () => clearInterval(timer)  // effect 재실행 전 or 컴포넌트 파괴 시 호출
})
```

### `$effect.pre`

DOM 업데이트 **전에** 실행된다. 스크롤 위치 저장처럼 DOM 변경 전에 값을 읽어야 할 때 사용.

```js
$effect.pre(() => {
  savedScrollTop = container.scrollTop
})
```

### `$effect` 안에서 async 사용 금지

```js
// ❌ 클린업 함수를 반환할 수 없음
$effect(async () => {
  const data = await fetch(url)
})

// ✅ 별도 async 함수 호출
$effect(() => {
  async function fetchData() {
    const res = await fetch(url)
    // ...
  }
  fetchData()
})
```

> `$effect`의 콜백은 동기 함수여야 한다. async 함수는 Promise를 반환하므로 클린업 패턴이 동작하지 않는다.

---

## 5. 의존성 자동 추적 원리

**핵심**: `$state` 값을 **읽는 순간** "지금 누가 실행 중인지" 체크해서 의존성을 등록한다.

컴파일러가 템플릿을 자동으로 effect로 변환한다:

```svelte
<!-- 내가 쓴 코드 -->
<h1>{fullName}</h1>
```

```js
// 컴파일러가 내부적으로 생성 (개념적)
effect(() => {
  h1.textContent = fullName
})
```

### 추적 과정

```text
① effect 실행 → fullName 읽기
② fullName은 $derived → firstName, lastName 읽기
③ 각 $state가 "fullName이 나를 읽었다" 기록
④ fullName이 "h1 effect가 나를 읽었다" 기록

결과:
firstName ──→ fullName ──→ h1 effect (DOM 업데이트)
lastName  ──→ fullName
```

### 상태 변경 시

```text
firstName 변경
  → fullName 재계산 → 값이 달라졌으면 → h1 effect 재실행 → DOM 업데이트
                    → 값이 같으면   → h1 effect 스킵 (최적화)
```

### 동적 의존성

매 실행마다 의존성을 **다시 수집**하므로, 조건문에 따라 의존성이 바뀔 수 있다.

```js
$effect(() => {
  console.log(userId || fullName)
})
```

```text
userId가 있을 때: userId만 추적 (fullName은 읽히지 않으니까)
userId가 없을 때: userId + fullName 모두 추적
```

### Lazy evaluation

아무 effect도 읽지 않는 `$derived`는 재계산 자체를 건너뛴다. 사용되지 않는 계산은 비용이 0이다.

---

## 6. 디버깅 도구

### `$inspect` — 상태 변경 자동 로깅

개발 빌드에서만 동작. 중첩 프로퍼티 변경도 감지한다.

```js
$inspect(object)                       // 변경 시 console.log
$inspect(object).with(console.trace)   // 스택 트레이스 포함
```

### `$inspect.trace()` — effect 재실행 원인 추적

```js
$effect(() => {
  $inspect.trace()  // 어떤 상태 변경이 이 effect를 트리거했는지 출력
  // ...
})
```

### `$state.snapshot` — 프록시 벗기기

외부 라이브러리에 `$state` 객체를 넘길 때 Proxy 문제가 생기면 사용.

```js
console.log($state.snapshot(user))  // 순수 JS 객체
```
