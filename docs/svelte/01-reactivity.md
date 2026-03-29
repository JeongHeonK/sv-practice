# 신호 기반 반응성 (Signal-based Reactivity)

## Svelte vs React

| | React | Svelte |
| -- | -- | -- |
| 방식 | 컴포넌트 함수 전체 재실행 후 Virtual DOM 비교 | 의존성 연결된 곳만 직접 DOM 업데이트 |
| Virtual DOM | 있음 | 없음 |
| 런타임 | 브라우저에 React 라이브러리 필요 | 컴파일 결과물(바닐라 JS)만 있으면 됨 |
| 의존성 추적 | `useMemo`, `useCallback` 등으로 직접 명시 | 자동 추적 |

Svelte는 **컴파일러**다. `.svelte` 파일을 바닐라 JS/CSS로 변환하고, 브라우저는 그 결과물만 실행한다.

```text
개발 시          빌드 시            런타임
.svelte   →  컴파일러  →  .js/.css  →  브라우저에서 실행
(Svelte 문법)           (바닐라 JS)    (Svelte 런타임 없음)
```

---

## 3가지 핵심 개념

```text
$state    → 읽기/쓰기 가능한 소스 (신호)
$derived  → 소스 기반 계산값 (읽기 전용)
effect    → DOM 업데이트 등 부작용 실행
```

---

## 의존성 자동 추적 원리 — 파생 스택

**핵심 트릭**: `$state` 값을 읽을 때 "지금 누가 실행 중인지" 체크한다.

```text
파생 스택 = 현재 실행 중인 effect/derived를 담는 임시 통
```

### h1 effect는 어디서?

Svelte 컴파일러가 템플릿을 자동으로 effect로 변환한다.

```svelte
<!-- 내가 작성한 코드 -->
<h1>Welcome {fullName}</h1>
```

```js
// Svelte가 내부적으로 생성 (개념적)
effect(() => {
  h1.textContent = 'Welcome ' + fullName
})
```

### 의존성 추적 동작 순서

```text
① h1 effect 실행 → 스택에 push → [h1 effect]

② fullName 읽기 → $derived이므로 계산 함수 실행
   → 스택에 push → [h1 effect, fullName]

③ fullName 내부에서 firstName 읽기
   → $state가 스택 확인 → "맨 위: fullName"
   → firstName → fullName 의존성 등록

④ fullName 내부에서 lastName 읽기
   → lastName → fullName 의존성 등록

⑤ fullName 계산 완료 → 스택에서 pop → [h1 effect]

⑥ h1 effect에서 fullName 읽기 완료
   → $derived가 스택 확인 → "맨 위: h1 effect"
   → fullName → h1 effect 의존성 등록

⑦ h1 effect 완료 → 스택에서 pop → []
```

**결과 의존성 트리:**

```text
firstName ──→ fullName ──→ h1 effect
lastName  ──→ fullName
```

---

## 상태 변경 시 동작

```text
firstName 변경
    ↓
fullName → 재계산 필요 표시
    ↓
fullName 재계산 → 값이 달라짐?
    ├── 같음  → h1 effect 실행 안함 (최적화)
    └── 다름  → h1 effect 실행 → DOM 업데이트
```

---

## 동적 의존성 트리

조건문에 따라 런타임에 의존성 트리가 바뀐다.

```js
// userId가 있으면 fullName 코드에 도달하지 않음
$effect(() => {
  console.log(userId || fullName)
})
```

```text
userId 없을 때:              userId 있을 때:

firstName → fullName         fullName → 아무도 안 씀 (실행 안함)
lastName  → fullName
fullName  → effect           userId → effect
userId    → effect
```

**아무 effect도 안 쓰는 `$derived`는 재계산 자체를 건너뜀** — Lazy evaluation.
