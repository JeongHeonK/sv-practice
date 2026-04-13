# 성능 최적화

Svelte 5의 성능 최적화 패턴을 다룬다. `{#key}` 블록, `$effect` 최소화, 코드 스플리팅, 세밀한 반응성까지.

---

## 1. `{#key}` 블록 — 강제 재마운트

### 언제 필요한가

컴포넌트는 기본적으로 내부 상태를 유지한다. props가 바뀌어도 컴포넌트가 재활용된다. key가 바뀔 때 컴포넌트를 **완전히 파괴하고 새로 생성**해야 하는 경우에 `{#key}`를 사용한다.

```text
일반적인 경우:
userId 변경 → 동일 컴포넌트 인스턴스 재활용 → 이전 상태 남아있음

{#key} 사용:
userId 변경 → 기존 컴포넌트 파괴 → 새 인스턴스 생성 → 완전 초기화
```

### 기본 문법

```svelte
{#key expression}
  <Component />
{/key}
```

expression 값이 바뀌면 내부 컴포넌트(또는 요소)가 파괴되고 재생성된다.

### 사용 예시

```svelte
<!-- userId가 바뀔 때 UserProfile을 완전히 새로 초기화 -->
{#key userId}
  <UserProfile id={userId} />
{/key}
```

내부에서 `onMount`로 데이터를 불러오거나, 자체 폼 상태를 가진 컴포넌트라면, `{#key}`를 감싸는 것만으로 "깨끗한 슬레이트" 효과를 얻는다.

### 트랜지션과 조합

```svelte
{#key currentPage}
  <div transition:fade>{currentPage}</div>
{/key}
```

값이 바뀔 때마다 트랜지션을 재생시킬 때도 유용하다.

### 주의사항

`{#key}`는 컴포넌트를 파괴하고 재생성하므로 비용이 있다. 단순히 props를 전달하는 것으로 해결되는 경우라면 `{#key}` 없이 `$derived`를 컴포넌트 내부에서 사용하는 것이 낫다.

---

## 2. 불필요한 `$effect` 방지

### `$effect`는 탈출구다

`$effect`는 Svelte 반응성 시스템 밖의 것(DOM 라이브러리, WebSocket, 타이머 등)과 연결할 때 사용한다. 순수한 상태-상태 동기화에는 사용하지 않는다.

### `$derived`로 대체해야 하는 경우

```js
// ❌ $effect로 상태를 동기화하면 안 됨
let doubled = $state(0)
$effect(() => {
  doubled = count * 2
})

// ✅ $derived로 해결
let doubled = $derived(count * 2)
```

`$effect`는 항상 한 프레임 지연 후 실행되지만 `$derived`는 즉시 동기적으로 반영된다. 상태→상태 변환은 항상 `$derived`가 더 빠르고 안전하다.

### 판단 기준표

```text
해결하려는 문제               →  올바른 도구
─────────────────────────────────────────────
A 상태 → B 상태 계산         →  $derived
이벤트에 반응                →  이벤트 핸들러
DOM 라이브러리 초기화/업데이트 →  {@attach ...}
WebSocket / EventEmitter 구독 →  createSubscriber
상태 변화 디버깅             →  $inspect
그 외 외부 부작용             →  $effect
```

### `$effect`가 적절한 경우

```js
// ✅ 외부 라이브러리 연동 — $effect가 맞다
$effect(() => {
  chartInstance.update(data)        // data 변경 시 차트 업데이트
  return () => chartInstance.destroy()
})
```

```js
// ✅ 타이머 — 외부 상태이므로 $effect가 맞다
$effect(() => {
  const id = setInterval(() => tick(), interval)
  return () => clearInterval(id)    // 클린업 필수
})
```

### 의존성 자동 추적

`$effect`와 `$derived` 모두 실행 중에 읽은 `$state` 값을 자동으로 의존성으로 등록한다. 수동으로 배열을 작성할 필요가 없다.

```text
$effect 실행
  → data 읽음 → "data가 변경되면 재실행" 등록
  → options.color 읽음 → "options.color가 변경되면 재실행" 등록

data만 바뀌면: $effect 재실행
options.color만 바뀌면: $effect 재실행
options.size가 바뀌면: 이 $effect와 무관 → 재실행 안함
```

`untrack`으로 특정 읽기를 추적에서 제외할 수 있다.

```js
import { untrack } from 'svelte'

$effect(() => {
  trigger             // 이것만 추적
  untrack(() => {
    sideData          // 읽지만 의존성 등록 안함
  })
})
```

---

## 3. 번들 크기 최적화

### `{#await}` + dynamic import — 가장 간단한 lazy loading

```svelte
{#await import('./HeavyChart.svelte') then { default: HeavyChart }}
  <HeavyChart {data} />
{/await}
```

컴포넌트가 실제로 렌더링될 때까지 번들을 로드하지 않는다. 로딩 상태가 필요하면:

```svelte
{#await import('./HeavyChart.svelte')}
  <p>로딩 중...</p>
{:then { default: HeavyChart }}
  <HeavyChart {data} />
{/await}
```

### 조건부 lazy loading

탭, 모달, 대형 에디터처럼 항상 렌더링되지 않는 컴포넌트에 효과적이다.

```svelte
<script>
  let showEditor = $state(false)
</script>

{#if showEditor}
  {#await import('./RichTextEditor.svelte') then { default: Editor }}
    <Editor />
  {/await}
{/if}
```

`showEditor`가 처음 `true`가 될 때 번들을 로드한다. 이후 `false`→`true` 전환에서는 이미 캐시된 모듈을 재사용한다.

### SvelteKit의 자동 코드 스플리팅

SvelteKit은 라우트 단위로 코드를 자동 분리한다. 별도 설정 없이 `/dashboard` 라우트는 대시보드 코드만, `/profile` 라우트는 프로필 코드만 로드된다.

```text
src/routes/
  +page.svelte          → 홈 번들
  dashboard/
    +page.svelte        → 대시보드 번들 (별도 청크)
  profile/
    +page.svelte        → 프로필 번들 (별도 청크)
```

공통 컴포넌트(`$lib/`)는 자동으로 공유 청크로 분리된다.

### `$state.raw` — 대용량 읽기 전용 데이터

API 응답처럼 프로퍼티를 직접 변경하지 않는 대형 객체에는 Proxy 오버헤드를 없앤다.

```js
let results = $state.raw([])   // 수천 개의 아이템도 Proxy 없이 관리

async function search(query) {
  results = await fetchResults(query)  // 재할당으로만 업데이트
}
```

---

## 4. 렌더링 최적화

### 세밀한 반응성 (Granular Reactivity)

Svelte는 컴포넌트 전체를 재렌더링하지 않는다. 특정 상태와 연결된 DOM 노드만 직접 업데이트한다.

```text
$state 변경
  → 해당 상태를 직접 읽는 DOM 표현식만 업데이트
  → 형제 표현식, 부모 컴포넌트 → 건드리지 않음
```

```svelte
<script>
  let user = $state({ name: 'Park', age: 25 })
</script>

<p>{user.name}</p>   <!-- name 변경 시에만 업데이트 -->
<p>{user.age}</p>    <!-- age 변경 시에만 업데이트 -->
```

`user.name = 'Kim'` → 첫 번째 `<p>`만 업데이트. 두 번째 `<p>`는 변경 없음. React의 `React.memo` 같은 수동 메모이제이션이 필요 없다.

### `$derived`의 push-pull 모델

Svelte는 상태 변경을 즉시 알리되(push), 실제 계산은 값이 읽힐 때까지 미룬다(pull).

```text
count 변경
  → large = $derived(count > 10) 에 알림 전송 (push)
  → large가 실제로 읽힐 때 재계산 (pull)
  → 이전 값과 동일하면 → 하위 DOM 업데이트 스킵
```

즉 count가 5 → 6으로 바뀌어도 `large`(count > 10)가 여전히 `false`라면 `large`에 의존하는 DOM은 업데이트되지 않는다.

### 긴 목록 최적화

#### keyed `{#each}`

목록 순서가 바뀌거나 항목이 삽입/삭제될 때, key가 없으면 인덱스 기반으로 DOM을 재활용해 잘못된 상태가 남을 수 있다.

```svelte
<!-- ❌ 인덱스 기반 재활용 → 상태 혼동 가능 -->
{#each items as item}
  <Row {item} />
{/each}

<!-- ✅ 고유 key로 DOM 노드와 데이터 1:1 연결 -->
{#each items as item (item.id)}
  <Row {item} />
{/each}
```

#### Virtual List 패턴

수천 개의 항목을 동시에 DOM에 렌더링하면 브라우저가 느려진다. 화면에 보이는 항목만 렌더링하는 virtual list를 사용한다.

```text
실제 데이터: 10,000개
  → 화면에 보이는 30~50개만 DOM 생성
  → 스크롤 시 위치 계산 후 필요한 항목만 교체
```

`svelte-virtual-list` 라이브러리를 활용하거나, `{#each visibleItems as item (item.id)}` 패턴으로 직접 구현한다.

---

## React 성능 최적화 비교

| 패턴 | React | Svelte 5 |
|------|-------|----------|
| 강제 재마운트 | `key` prop | `{#key}` 블록 |
| 파생값 메모이제이션 | `useMemo` | `$derived` |
| 함수 메모이제이션 | `useCallback` | 불필요 (컴파일러가 처리) |
| 부작용 | `useEffect` | `$effect` |
| 부작용 클린업 | `useEffect` return | `$effect` return |
| 지연 로딩 | `React.lazy` + `Suspense` | `{#await import(...)}` |
| 컴포넌트 메모이제이션 | `React.memo` | 불필요 (세밀한 반응성) |
| 대용량 읽기 전용 데이터 | `useRef` 또는 외부 변수 | `$state.raw` |
