# 리액티브 클래스 — 로직 캡슐화와 재사용

Svelte 5에서 로직을 재사용 가능한 단위로 캡슐화하는 패턴.

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
| 리액티브 클래스 | 로직 캡슐화 + 재사용 | **구조분해 금지** |
| `.svelte.ts` | 컴포넌트 외부에서 rune 사용 | 일반 `.ts`에서는 rune 불가 |
| `#private` field | 내부 연쇄에서 setter 우회 | 부수효과 필요 시 공개 setter 사용 |
| getter/setter in class | 유효성 검사 + 연쇄 갱신 | 외부 접근 = getter, 내부 저장 = #field |
