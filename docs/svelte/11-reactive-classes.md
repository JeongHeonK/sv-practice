# 리액티브 클래스 — 로직 캡슐화와 재사용

Svelte 5에서 로직을 재사용 가능한 단위로 캡슐화하는 패턴.

---

## 팩토리 함수로 로직 추출 — 반응성 유지의 핵심

여러 컴포넌트에서 동일한 상태 로직을 쓸 때, 함수로 추출할 수 있다.

```ts
// counter.svelte.ts
export function createCounter() {
  let count = $state(0)

  return {
    get value() { return count },   // ✅ getter — 접근할 때마다 실행
    increment: () => count++,
    reset: () => count = 0,
  }
}
```

```svelte
<script>
  import { createCounter } from './counter.svelte.ts'
  const counter = createCounter()  // 컴포넌트마다 독립 인스턴스
</script>

<h2>{counter.value}</h2>
<button onclick={counter.increment}>+1</button>
<button onclick={counter.reset}>리셋</button>
```

### 치명적 함정: 값을 직접 반환하면 반응성이 끊긴다

```ts
// 작성한 코드
return { value: count }

// 컴파일 결과
return { value: $.get(count) }  // → { value: 0 }
```

```
$state(0)은 내부적으로 "신호 객체"를 생성한다.
count를 읽을 때마다 $.get(count) 호출로 컴파일된다.

객체 프로퍼티에 직접 넣으면:
  → 객체 생성 시점에 $.get(count) 한 번 호출
  → 반환값 0이 value에 고정
  → 이후 count가 바뀌어도 value는 0

getter로 감싸면:
  → 접근할 때마다 $.get(count) 호출
  → Svelte가 "이건 반응형 의존성"으로 인식
  → 변경 시 다시 읽어서 UI 갱신
```

### 구조분해도 같은 원리

```ts
const counter = createCounter()

const { value } = counter  // ❌ getter 호출 → 0 복사 → 연결 끊김
counter.value               // ✅ 매번 getter 실행 → 반응성 유지
```

**규칙: 반응형 객체(팩토리 반환값, 클래스 인스턴스)는 구조분해하지 않는다.**

---

## 팩토리 함수의 한계 → 클래스가 더 나은 이유

팩토리 함수는 getter를 **직접 작성**해야 한다. 빠뜨리면 반응성이 끊긴다.

클래스는 컴파일러가 `$state` 프로퍼티에 getter/setter를 **자동 생성**한다.

```ts
// 팩토리 함수 — getter를 수동으로 만들어야 함
function createCounter() {
  let count = $state(0)
  return {
    get value() { return count },  // 빠뜨리면 반응성 끊김
    increment: () => count++,
  }
}

// 클래스 — 컴파일러가 자동 처리
class Counter {
  value = $state(0)                // 컴파일러가 getter/setter 생성
  increment() { this.value++ }
}
```

### 컴파일 결과 비교

```
팩토리 함수:
  return { value: count }           → return { value: $.get(count) }
  return { get value() { return count } }  → 개발자가 직접 감싸야 함

클래스:
  value = $state(0)
    → 컴파일러가 자동으로:
       get value() { return $.get(this.#value) }
       set value(v) { $.set(this.#value, v) }
    → 개발자는 평범한 프로퍼티처럼 쓰면 됨
```

**결론: 재사용 로직이 단순하면 팩토리 함수, 상태가 여러 개이거나 복잡하면 클래스.**

---

## 패턴 4: 리액티브 클래스 — $state + getter/setter + #private

### 구조

```ts
class ReactiveService {
  // ── 공개 상태: 컴포넌트에서 bind/표시 ──
  loading = $state(true)
  error: string | undefined = $state(undefined)

  // ── 비공개 상태 + 커스텀 getter/setter ──
  #source = $state(0)

  get source() { return this.#source }
  set source(v: number) {
    this.#source = v < 0 ? 0 : v     // 유효성 검사
    this.#target = this.#compute()    // 연쇄 갱신
  }

  // ── 비공개 메서드 ──
  #compute() { /* ... */ }
  async #fetchData() { /* ... */ }

  // ── 생성자: $effect 등록 ──
  constructor() {
    $effect(() => {
      this.#fetchData()               // 의존하는 상태 변경 시 재실행
    })
  }
}
```

### 핵심 포인트

**클래스 프로퍼티 = 반응형 상태:**
```
loading = $state(true)
  → Svelte 컴파일러가 내부적으로 getter/setter 생성
  → 외부에서 instance.loading 접근 시 의존성 자동 등록
```

**$effect는 생성자에서 등록 가능:**
```ts
constructor() {
  $effect(() => { this.#fetchData() })
  // 단, 인스턴스가 컴포넌트 초기화 시점에 생성되어야 함
}
```

---

## #private vs public setter 선택

```
this.target = x       →  set target(x) 호출 → 부수효과 실행
this.#target = x      →  비공개 필드 직접 저장 → setter 우회

판단 기준: "이 시점에서 setter의 부수효과가 필요한가?"
  │
  ├─ YES (사용자 입력) → this.target = v   (공개 setter)
  └─ NO  (내부 연쇄)  → this.#target = v  (비공개 필드)
```

---

## setter 체인 시각화

```
set source(v)                          set baseCurrency(v)
  → #source = validate(v)               → #baseCurrency = v
  → #target = #compute()                → #fetchRates()
                                             → 응답 후: #target = #compute()
                                               (# 직접 갱신 — setter 우회!)

set target(v)   ← 사용자 입력 시에만
  → #target = v
  → #computeReverse()  ← 역계산
```

```
만약 내부 연쇄에서 this.target (공개 setter)를 썼다면:
  → #computeReverse() 호출 → source가 불필요하게 역계산
  → 부동소수점 오차 → source 미세하게 변경됨
```

---

## 치명적 함정: 구조분해는 반응성을 끊는다

```ts
const svc = new ReactiveService()

// ❌ 구조분해 → 반응성 끊김!
const { loading, error } = svc
// loading = true (초기값 복사) → 이후 svc.loading이 false가 되어도 갱신 안 됨

// ✅ 항상 인스턴스를 통해 접근
svc.loading   // → getter 호출 → 의존성 추적 유지
```

### 왜?

```
svc.loading 접근 시:
  ┌──────────┐    getter 호출    ┌──────────────┐
  │ 템플릿    │ ──────────────→  │ $state 내부   │  의존성 등록 → 변경 시 UI 갱신
  └──────────┘                   └──────────────┘

구조분해 시:
  const { loading } = svc
  loading = true  ← boolean 값 복사. getter와의 연결 끊김.
  이후 내부에서 loading이 false로 바뀌어도 이 변수는 여전히 true.

React 비유: const { current } = useRef(0)  → ref와 연결 끊김
```

**규칙: 리액티브 클래스 인스턴스는 절대 구조분해하지 않는다.**

---

## 패턴 5: .svelte.ts 파일 확장자

리액티브 클래스를 컴포넌트 밖으로 꺼내 재사용하려면 `.svelte.ts` 확장자가 필요하다.

```
❌ my-service.ts           → $state, $derived, $effect 컴파일 안 됨
✅ my-service.svelte.ts    → Svelte 컴파일러가 rune 처리
```

```
src/lib/utils/
  my-service.svelte.ts     ← 클래스 정의
src/routes/
  +page.svelte             ← 사용처 A
src/lib/components/
  MyWidget.svelte           ← 사용처 B
```

각 `new ReactiveService()`는 독립적인 인스턴스 — 상태 공유 없음.

---

## 패턴 6: 모듈 레벨 공유 상태 (싱글톤)

함수/클래스 **밖**에 `$state`를 선언하면 모든 컴포넌트가 **같은 상태**를 공유한다.

### 독립 인스턴스 vs 공유 상태

```
독립 (팩토리 함수 / 클래스):
  컴포넌트 A: const counter = new Counter()  → 상태 A
  컴포넌트 B: const counter = new Counter()  → 상태 B
  → 각자 독립된 상태

공유 (모듈 레벨):
  counter.svelte.ts에서 $state 직접 선언
  컴포넌트 A: import counter  → 같은 상태
  컴포넌트 B: import counter  → 같은 상태
  → 하나의 상태를 여러 컴포넌트가 공유
```

### 직접 export가 안 되는 이유

```ts
// counter.svelte.ts
let count = $state(0)

export default count
// ❌ 값(0)이 복사되어 export → 반응성 끊김

export { count }
// ❌ import한 쪽에서 count++ 시도 → "import에 할당 불가" (JS 규칙)
```

```
export default count는 컴파일 시:
  export default $.get(count)  → 0이 export됨 → 반응성 없음

import { count } from '...'
count++  → JS 모듈의 import binding은 읽기 전용 → 에러
```

### 해결: getter/setter 객체로 내보내기

```ts
// counter.svelte.ts
let count = $state(0)

export default {
  get value() { return count },      // 접근 시 $.get(count) 실행 → 반응성 유지
  set value(v) { count = v },        // 외부에서 값 변경 가능
  increment: () => count++,
  reset: () => count = 0,
}
```

```svelte
<!-- 컴포넌트 A, B 모두 동일 -->
<script>
  import counter from './counter.svelte.ts'
</script>

<p>{counter.value}</p>
<button onclick={counter.increment}>+1</button>
<button onclick={counter.reset}>리셋</button>
<!-- A에서 +1 → B에서도 즉시 반영 -->
```

### 더 간단한 방법: $state 객체를 직접 export

`$state`에 **객체**를 넘기면 Proxy로 감싸진다. Proxy가 getter/setter를 자동 처리하므로 수동으로 작성할 필요 없다.

```ts
// counter.svelte.ts
export const count = $state({ value: 0 })

// 끝! getter/setter, 클래스, 팩토리 함수 전부 불필요
```

```svelte
<script>
  import { count } from './counter.svelte.ts'
</script>

<p>{count.value}</p>
<button onclick={() => count.value++}>+1</button>
<button onclick={() => count.value = 0}>리셋</button>
```

```
왜 되는가?

let count = $state(0)      → 원시값. export 시 $.get(count) = 0 복사. 반응성 끊김.
const count = $state({...}) → 객체. Proxy로 감싸짐. export해도 Proxy 참조 유지.
                               count.value 접근 → Proxy get trap → 의존성 등록 ✅
                               count.value = 1  → Proxy set trap → 반응성 트리거 ✅
```

조작 함수를 별도로 export할 수도 있다:

```ts
export const count = $state({ value: 0 })
export function increment() { count.value++ }
export function reset() { count.value = 0 }
```

### 클래스로도 동일하게 가능

```ts
// counter.svelte.ts
class Counter {
  value = $state(0)
  increment() { this.value++ }
  reset() { this.value = 0 }
}

export default new Counter()  // 인스턴스를 export → 싱글톤
```

```
클래스 export 비교:
  export default Counter        → 컴포넌트에서 new → 독립 인스턴스
  export default new Counter()  → 인스턴스를 export → 공유 싱글톤
```

### 공유 상태에 $effect 사용하기 — $effect.root

컴포넌트 안에서 공유 상태에 `$effect`를 걸면, 해당 컴포넌트를 **사용한 횟수만큼** effect가 생성된다.

```svelte
<!-- ClickCount.svelte — 3번 사용하면 effect도 3개 -->
<script>
  import { count } from './counter.svelte.ts'

  $effect(() => {
    console.log(count.value)  // 컴포넌트 인스턴스마다 실행됨
  })
</script>
```

```
<ClickCount />  → effect 1개
<ClickCount />  → effect 1개
<ClickCount />  → effect 1개

count.value++ → 콘솔에 3번 출력!
```

상태 변경 시 **한 번만** 실행되는 effect가 필요하면 모듈 레벨에 정의한다. 단, 컴포넌트 밖에서는 `$effect`를 직접 쓸 수 없고 `$effect.root`로 감싸야 한다.

```ts
// counter.svelte.ts
export const count = $state({ value: 0 })

// ❌ $effect(() => { ... })
// → "effect_orphan" 에러. 컴포넌트 밖에는 루트 effect가 없음.

// ✅ $effect.root로 감싸기
$effect.root(() => {
  $effect(() => {
    console.log(count.value)  // 한 번만 실행
  })
})
```

```
왜 컴포넌트에서는 $effect가 바로 되는가?

컴포넌트 초기화 시 Svelte가 자동으로 루트 effect를 생성한다.
컴포넌트 안의 $effect는 이 루트 effect의 자식으로 등록된다.
컴포넌트가 파괴되면 루트 effect도 파괴 → 자식 effect 자동 정리.

모듈 레벨에는 컴포넌트가 없으므로 루트 effect가 없다.
$effect.root()로 직접 루트를 만들어줘야 한다.
```

### 실전: localStorage 영속화

```ts
// counter.svelte.ts
import { browser } from '$app/environment'

const saved = browser ? localStorage.getItem('count') : null
export const count = $state(saved ? JSON.parse(saved) : { value: 0 })

$effect.root(() => {
  $effect(() => {
    localStorage.setItem('count', JSON.stringify(count))
  })
})
```

```
주의 1: JSON.stringify(count)는 count 객체의 모든 프로퍼티를 읽는다.
  → 모든 프로퍼티가 의존성으로 등록됨
  → 어떤 프로퍼티가 바뀌든 effect 재실행

주의 2: localStorage는 브라우저 API — 서버에서 사용 불가.
  → SvelteKit은 서버에서도 모듈을 실행하므로 에러 발생
  → browser 체크로 분기 필요
  → $effect 안의 localStorage.setItem은 OK (effect는 클라이언트에서만 실행)
  → 모듈 최상위의 localStorage.getItem은 browser 체크 필수
```

### $state vs $state.raw 선택 — 공유 상태에서도 동일

```
API 응답처럼 통째로 교체하는 객체:
  export const user = $state.raw(null)  // Proxy 불필요
  user = await fetch('/api/me').then(r => r.json())  // 재할당으로 갱신

프로퍼티를 개별 변경하는 객체:
  export const count = $state({ value: 0 })  // Proxy 필요
  count.value++  // 프로퍼티 변경으로 갱신
```

---

### ⚠ SSR 안전성 경고

```
모듈 레벨 상태는 서버에서 위험하다.

클라이언트: 브라우저 탭마다 독립된 모듈 인스턴스 → 문제 없음
서버(SSR):  하나의 프로세스가 모든 요청 처리 → 모듈 레벨 상태가 사용자 간 공유됨!

  사용자 A 요청 → count.value = 5
  사용자 B 요청 → count.value가 5로 보임 ← 데이터 유출!

해결: 서버에서도 안전한 공유 상태가 필요하면 → Context API (setContext/getContext)
      Context는 컴포넌트 트리 범위로 스코프되어 요청 간 격리됨
```

---

## React 커스텀 훅 비교

```
React: useSomeService() → { value, setValue, loading, ... }
  - 훅은 컴포넌트 안에서만 호출 가능
  - 반환값을 구조분해해서 사용 (일반적)
  - 상태와 setter가 분리됨

Svelte: new ReactiveService() → svc.value, svc.loading
  - 클래스는 어디서든 인스턴스 생성 가능
  - 인스턴스를 통해 접근 (구조분해 금지!)
  - 상태와 로직이 하나의 객체에 캡슐화
```

---

## 핵심 정리

| 패턴 | 용도 | 주의점 |
|------|------|--------|
| 팩토리 함수 | 단순한 로직 추출 | getter 수동 작성 필요 |
| 리액티브 클래스 | 로직 캡슐화 + 재사용 | **구조분해 금지**, 컴파일러가 getter 자동 생성 |
| 모듈 레벨 상태 | 여러 컴포넌트 간 상태 공유 | 직접 export 불가, getter/setter 객체로 감싸기 |
| `.svelte.ts` | 컴포넌트 외부에서 rune 사용 | 일반 `.ts`에서는 rune 불가 |
| `#private` field | 내부 연쇄에서 setter 우회 | 부수효과 필요 시 공개 setter 사용 |
