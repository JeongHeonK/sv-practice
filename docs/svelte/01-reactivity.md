# 반응성 — React에서 Svelte로 멘탈 모델 전환

## 핵심 차이: "다시 실행" vs "연결"

React는 상태가 바뀌면 **컴포넌트 함수 전체를 다시 실행**하고 Virtual DOM을 비교한다.
Svelte는 상태가 바뀌면 **그 상태에 연결된 DOM만 직접 업데이트**한다.

```text
React:   setState → 컴포넌트 함수 재실행 → VDOM diff → DOM 패치
Svelte:  $state 변경 → 연결된 effect만 재실행 → DOM 직접 업데이트
```

| | React | Svelte |
| -- | -- | -- |
| 업데이트 단위 | 컴포넌트 전체 | 변경된 상태에 연결된 곳만 |
| Virtual DOM | 있음 | 없음 |
| 의존성 추적 | `useMemo`, `useCallback`으로 수동 명시 | 자동 추적 |
| 런타임 | React 라이브러리 필요 | 컴파일된 바닐라 JS만 실행 |

> **멘탈 모델 전환**: React에서는 "렌더링 함수가 다시 호출된다"고 생각한다. Svelte에서는 "상태와 DOM 사이에 전선이 연결되어 있다"고 생각하면 된다.

---

## Svelte는 컴파일러다

`.svelte` 파일을 빌드 시점에 바닐라 JS/CSS로 변환한다. 브라우저는 Svelte를 모른다.

```text
.svelte  →  컴파일러  →  .js/.css  →  브라우저 실행
```

---

## 3가지 핵심 개념

```text
$state    →  값이 바뀌면 알림을 보내는 소스
$derived  →  소스가 바뀌면 자동 재계산되는 값
effect    →  소스가 바뀌면 자동 재실행되는 부작용 (DOM 업데이트 등)
```

React로 비유하면:

```text
$state   ≈  useState (하지만 setter 함수 없이 직접 변경)
$derived ≈  useMemo  (하지만 의존성 배열 없이 자동 추적)
effect   ≈  useEffect (하지만 의존성 배열 없이 자동 추적)
```

---

## 의존성 자동 추적 원리

**핵심 트릭**: `$state` 값을 **읽는 순간** "지금 누가 실행 중인지" 체크해서 의존성을 등록한다.

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

> **React와 차이**: React는 `useMemo(fn, [dep1, dep2])`처럼 의존성을 수동으로 나열한다. Svelte는 코드를 실행하면서 실제로 읽은 값만 자동으로 추적한다. 의존성 배열이 없다.

---

## 동적 의존성

매 실행마다 의존성을 **다시 수집**하므로, 조건문에 따라 의존성 트리가 바뀔 수 있다.

```js
$effect(() => {
  console.log(userId || fullName)
})
```

```text
userId가 있을 때: userId만 추적 (fullName은 읽히지 않으니까)
userId가 없을 때: userId + fullName 모두 추적
```

React의 `useEffect`는 의존성 배열이 고정이지만, Svelte는 실행 경로에 따라 자동으로 달라진다.

---

## Lazy evaluation

아무 effect도 읽지 않는 `$derived`는 재계산 자체를 건너뛴다. 사용되지 않는 계산은 비용이 0이다.
