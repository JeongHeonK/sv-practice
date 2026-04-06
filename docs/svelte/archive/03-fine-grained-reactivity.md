# 세분화된 반응성(Fine-grained Reactivity) — React와의 핵심 차이

React는 **컴포넌트 단위**로 리렌더링한다. Svelte는 **프로퍼티 단위**로 업데이트한다. 이 차이가 성능 최적화 전략을 근본적으로 바꾼다.

---

## React vs Svelte 업데이트 범위 비교

```
[ React ]                          [ Svelte ]
setState({ name: 'Kim' })         user.name = 'Kim'
         ↓                                 ↓
 컴포넌트 함수 전체 재실행            name을 읽는 바인딩만 업데이트
 → JSX 전체 재평가                  → age를 읽는 곳은 무시
 → Virtual DOM diff                → DOM 직접 업데이트 (no diffing)
 → memo 없으면 자식도 전부 재실행    → memo 불필요
```

---

## 원리: Proxy 기반 프로퍼티 추적

`$state`로 선언된 객체는 Proxy로 감싸진다. 각 프로퍼티의 **읽기/쓰기를 개별 추적**한다.

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

React에서는 `setUser({ ...user, name: 'Kim' })`이 **두 `<p>` 모두** 재평가를 유발한다.

---

## 배열도 동일하게 동작한다

```svelte
<script>
  let items = $state([
    { text: 'A', done: false },
    { text: 'B', done: false },
    { text: 'C', done: false },
  ]);
</script>

{#each items as item}
  <!-- item.done만 의존 → 해당 항목의 done이 바뀔 때만 업데이트 -->
  <span class:done={item.done}>{item.text}</span>
{/each}
```

`items[1].done = true` → **B 항목만** 업데이트. A, C는 무시.
React에서 같은 효과를 얻으려면 각 항목을 `React.memo`로 감싸야 한다.

---

## $derived — 의존성 자동 추적

```svelte
<script>
  let items = $state([{ done: false }, { done: true }, { done: false }]);
  let doneCount = $derived(items.filter(i => i.done).length);
</script>

<p>완료: {doneCount}</p>
```

`$derived`는 내부에서 **실제로 읽힌 프로퍼티만** 추적한다. `items[0].done`이 바뀌면 재계산되지만, 새로운 프로퍼티(예: `items[0].text`)를 추가해도 재계산되지 않는다.

> `$bindable` (양방향 바인딩)은 [04-components.md](04-components.md)에서 다룬다.
