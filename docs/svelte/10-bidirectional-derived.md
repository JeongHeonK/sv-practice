# 양방향 파생 패턴 — $derived 한계와 해결책

Svelte 5에서 파생값을 사용자가 수정해야 할 때 어떻게 설계하는가?

---

## 패턴 1: $derived 오버라이드 (Svelte 5.25+)

Svelte 5.25부터 `$derived`에 직접 할당할 수 있다. 의존성이 변경되면 다시 원래 계산값으로 돌아간다.

```svelte
<script>
  let { post, like } = $props()

  let likes = $derived(post.likes)  // 서버 데이터에서 파생

  async function onclick() {
    likes += 1           // ← 즉시 오버라이드 (낙관적 UI)
    try {
      await like()       // 서버에 반영
    } catch {
      likes -= 1         // 실패 시 롤백
    }
    // post.likes가 서버에서 갱신되면 → $derived가 다시 계산
  }
</script>

<button {onclick}>🧡 {likes}</button>
```

```
$derived 오버라이드 동작:

  post.likes = 10 (서버 데이터)
    → likes = $derived(post.likes) = 10

  사용자 클릭 → likes += 1 → likes = 11 (임시 오버라이드)
    → post.likes는 여전히 10

  서버 응답 → post.likes = 11
    → $derived 재계산 → likes = 11 (오버라이드 해제, 서버값으로 복귀)
```

### 오버라이드가 적합한 경우 vs 아닌 경우

```
적합:                               부적합:
─────                              ─────
낙관적 UI (좋아요, 토글)             양방향 변환 (USD ↔ KRW)
임시 편집 후 서버 확인                역계산이 필요한 경우
사용자 입력 → 서버 반영 대기          두 입력이 서로 파생 관계
```

양방향 변환처럼 **역계산이 필요한 경우**에는 오버라이드만으로 부족하다. 아래 패턴들이 필요.

> 5.25 이전에는 `$derived`가 완전한 읽기 전용이었다.

---

## $derived가 해결하지 못하는 경우

```
A ($state) ──계산──→ B ($derived)

bind:value는 양방향이 필요:
  1. 상태 → input (읽기)    ✅ $derived 가능
  2. input → 상태 (쓰기)    → $derived 오버라이드는 가능하지만, 역계산은 안 됨

예: 환율 변환기
  USD 입력 → KRW = USD × 1300   ✅ $derived
  KRW 입력 → USD = KRW / 1300   ❌ $derived 오버라이드로는 역계산 불가

필요한 것: A ←──→ B   어느 쪽이든 입력 시 상대방을 역계산
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

Svelte 5에서 getter/setter 객체와 `bind:value`를 조합하여 유효성 검사를 삽입할 수 있다.

```svelte
<script>
  let amount = $state(1)

  // getter/setter 객체로 유효성 검사 삽입
  let validated = {
    get value() { return amount },
    set value(v: number) { amount = Math.max(0, v) }
  }
</script>

<!-- 기본 양방향 바인딩 -->
<input type="number" bind:value={amount} />

<!-- getter/setter로 유효성 검사 삽입 -->
<input type="number" bind:value={validated.value} />
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

// Svelte: getter/setter 객체 + bind:value
<input type="number" bind:value={validated.value} />
```

---

## 판단 트리: "파생값을 사용자가 수정해야 한다"

```
"파생값인데 사용자가 수정할 수도 있어야 한다"
  │
  ├─ 역계산 불필요? (낙관적 UI 등)
  │   └─ $derived 오버라이드 (5.25+) → ✅ 가장 간단
  │
  ├─ 역계산 필요? (양방향 변환)
  │   ├─ $effect 두 개로 동기화?     → ❌ 핑퐁 + 부동소수점 오차
  │   ├─ 이벤트 콜백 (oninput)?     → ✅ 안전하지만 코드 많음
  │   ├─ getter/setter 객체?        → ✅ bind 가능, 핑퐁 없음
  │   └─ 리액티브 클래스?           → ✅ 캡슐화 + 재사용까지
```
