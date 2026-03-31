# 양방향 파생 패턴 — $derived 한계와 해결책

Svelte 5에서 파생값을 사용자가 수정해야 할 때 어떻게 설계하는가?

---

## 패턴 1: $derived는 읽기 전용

### 문제

```
A ($state) ──계산──→ B ($derived)
                          ↑
                     쓸 수 없음!

bind:value는 양방향이 필요:
  1. 상태 → input (읽기)    ✅ $derived 가능
  2. input → 상태 (쓰기)    ❌ $derived 불가

React 비유: useMemo 결과에 setValue를 호출할 수 없는 것과 동일.
```

```svelte
<!-- ❌ 에러: $derived 값에 bind 불가 -->
<input bind:value={derivedValue} />

<!-- 단방향 표시만 가능 -->
<input value={derivedValue} />
```

### 상태 설계 딜레마

```
현재:     A ($state) ──→ B ($derived)     단방향만 가능
필요한 것: A ←──→ B                       어느 쪽이든 입력 가능
```

---

## 안티패턴: $effect로 상태↔상태 동기화 (핑퐁)

$derived 대신 B도 $state로 만들고 $effect 두 개로 양방향 계산을 시도하면?

```ts
let a: number = $state(100)
let b: number = $state(0)

// effect 1: a → b
$effect(() => {
  untrack(() => { b = a * RATIO })
})

// effect 2: b → a (역계산)
$effect(() => {
  untrack(() => { a = b / RATIO })
})
```

### untrack이 필요한 이유

```
effect 1은 a에 의존 → b를 계산
effect 2는 b에 의존 → a를 계산

untrack 없이 쓰면:
  b 변경 → effect 2 실행 → a 변경 → effect 1 실행 → 무한루프!

untrack으로 "쓰기"를 의존성에서 제외해야 순환을 끊을 수 있다.
```

### 핑퐁 발생 과정

```
사용자가 a = 9 입력:

  ① a 변경 (9)
  ② effect 1 → b = 9 × 0.85 = 7.65
  ③ b 변경됨 → effect 2 → a = 7.65 / 0.85
  ④ JS 부동소수점 → a = 8.999999...

┌────────┐ effect1 ┌────────┐ effect2 ┌──────────┐
│ a = 9  │ ──→     │b = 7.65│ ──→     │a = 8.99  │  ← 원치 않는 역계산
└────────┘         └────────┘         └──────────┘
```

### 근본 원인

```
$effect는 "누가 바꿨는지" 모른다.
  - 의존성이 바뀌면 무조건 실행
  - "사용자 입력" vs "다른 effect가 바꾼 것" 구분 불가
  → 한쪽이 바뀌면 반대쪽도 바뀌고, 다시 원래쪽도 바뀌는 핑퐁

React에서도 동일:
  useEffect A: a → setB(...)
  useEffect B: b → setA(...)
  → 같은 핑퐁 (React도 이 패턴은 안티패턴)
```

### $effect 용도 판단 기준

```
$effect 적합:                    $effect 부적합:
──────────────                   ──────────────
외부 DOM 조작                     상태 → 상태 동기화 ❌
WebSocket 구독                    파생값 계산 ❌
서드파티 라이브러리 연동            양방향 변환 ❌
```

---

## 패턴 2: getter/setter 객체로 "쓰기 가능한 파생" 구현

JS 객체의 getter/setter를 활용하면 파생 + 쓰기를 한 곳에서 처리할 수 있다.

### 핵심 아이디어

```
target을 $state도 $derived도 아닌 "getter/setter 객체"로 정의:

  get → source에서 target을 계산  (파생 읽기)
  set → 입력값으로 source를 역계산 (쓰기 시 역변환)
```

### 구현

```ts
let source = $state(100)

let target = {
  get value() {
    return source * RATIO           // 읽기: 정방향 계산
  },
  set value(v: number) {
    source = v / RATIO              // 쓰기: 역방향 계산
  }
}
```

```svelte
<input type="number" bind:value={source} />
<input type="number" bind:value={target.value} />
```

### 동작 흐름

```
[정방향] source 변경:
  source = 100 → target.value (getter) → 100 × RATIO → 표시
  ✅ 계산 1회

[역방향] target input 변경:
  target.value = 200 (setter) → source = 200 / RATIO
    → source 변경 → target.value (getter) 재계산 → 원래 값 복원
  ✅ 계산 2회, 핑퐁 없음
```

### $effect와의 결정적 차이

```
$effect 방식:                       getter/setter 방식:
──────────────                      ────────────────────
target 변경                          target input 변경
  → effect2: source 계산              → setter: source 계산
  → source 변경                       → source 변경
  → effect1: target 재계산             → getter: target 재계산
  → target 변경                        (끝! setter 다시 안 불림)
  → effect2: source 재계산...
  → 핑퐁 계속!                       핑퐁 없음!

핵심: setter는 "사용자 입력 시에만" 호출.
      getter 반환값이 input에 표시되는 건 setter를 트리거하지 않음.
```

---

## 패턴 3: bind:value getter/setter — 입력 유효성 검사

Svelte 5에서 `bind:value`에 getter/setter 함수를 직접 넘길 수 있다.

```svelte
<!-- 기본 양방향 바인딩 -->
<input type="number" bind:value={amount} />

<!-- getter/setter: 유효성 검사 삽입 -->
<input type="number" bind:value={
  () => amount,
  (v) => amount = Math.max(0, v)
} />
```

### 동작 흐름

```
┌──────────┐    getter      ┌──────────┐
│  $state  │ ──────────→    │  <input>  │   화면에 표시
│  amount  │                │           │
│          │ ←──────────    │           │   사용자 입력
└──────────┘    setter      └──────────┘

setter에서 유효성 검사:
  입력값 < 0  →  amount = 0 (강제 보정)
  입력값 >= 0 →  amount = 입력값
```

### React 비교

```jsx
// React: onChange 핸들러에서 유효성 검사
const [amount, setAmount] = useState(1)
<input
  value={amount}
  onChange={(e) => {
    const v = Number(e.target.value)
    setValue(v < 0 ? 0 : v)
  }}
/>

// Svelte: bind:value getter/setter
<input type="number" bind:value={
  () => amount,
  (v) => amount = v < 0 ? 0 : v
} />
```

---

## 패턴 6: {#await} 블록 — Promise 기반 비동기 UI

컴포넌트에서 Promise의 3가지 상태를 선언적으로 처리.

```ts
const dataPromise = fetch('/api/data').then(r => r.json())
```

```svelte
{#await dataPromise}
  <p>로딩 중...</p>
{:then data}
  <p>{data.name}</p>
{:catch error}
  <p>에러: {error.message}</p>
{/await}
```

```
Promise 생성
     │
     ├─ pending   → {#await promise} 블록 렌더링
     ├─ fulfilled → {:then value} 블록 렌더링
     └─ rejected  → {:catch error} 블록 렌더링
```

### React 비교

```jsx
// React: 3개 상태 + useEffect 직접 관리
const [data, setData] = useState(null)
const [loading, setLoading] = useState(true)
const [error, setError] = useState(null)

useEffect(() => {
  fetch(url).then(r => r.json())
    .then(d => { setData(d); setLoading(false) })
    .catch(e => { setError(e); setLoading(false) })
}, [])

// Svelte: Promise 하나 + {#await} → 상태 관리 불필요
```

---

## 패턴 7: $effect vs 이벤트 콜백 — 데이터 요청 트리거

상태 변경 시 API를 다시 호출해야 할 때 두 가지 접근.

### $effect (의존성 기반 자동 재요청)

```ts
$effect(() => {
  fetchData()  // 내부에서 참조하는 상태 변경 시 자동 재실행
})
```

### 이벤트 콜백 (명시적 트리거)

```svelte
<select bind:value={category} onchange={fetchData}>
```

### 비교

```
┌──────────────┬─────────────────────┬──────────────────────┐
│              │ $effect             │ 이벤트 콜백           │
├──────────────┼─────────────────────┼──────────────────────┤
│ 트리거       │ 의존성 변경 시 자동  │ 사용자 액션 시에만    │
│ 초기 실행    │ ✅ 자동 (마운트 시)  │ ❌ 수동 호출 필요     │
│ 제어력       │ 낮음 (자동)         │ 높음 (명시적)         │
│ 적합한 상황  │ "X 바뀌면 항상 fetch"│ "이 액션에서만 fetch" │
└──────────────┴─────────────────────┴──────────────────────┘
```

### $effect + async 주의

```ts
// ❌ $effect에 직접 async 불가 (반환값이 Promise가 되어 클린업 함수로 쓸 수 없음)
$effect(async () => { ... })

// ✅ 별도 async 함수를 effect에서 호출
async function fetchData() { ... }
$effect(() => { fetchData() })
```

---

## 판단 트리: "양방향 파생" 문제

```
"파생값인데 사용자가 수정할 수도 있어야 한다"
  │
  ├─ $derived만?              → ❌ 읽기 전용, bind 불가
  ├─ $effect 두 개로 동기화?   → ❌ 핑퐁 + 부동소수점 오차
  ├─ 이벤트 콜백 (oninput)?   → ✅ 안전하지만 코드 많음
  ├─ getter/setter 객체?      → ✅ bind 가능, 핑퐁 없음
  └─ 리액티브 클래스?         → ✅ 캡슐화 + 재사용까지
```
