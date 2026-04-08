# $bindable — 양방향 바인딩

---

## 핵심 개념: 데이터 흐름 방향

Svelte에서 props는 기본적으로 **단방향** — 부모 → 자식으로만 흐른다.

```
일반 props:
  부모 → 자식     (단방향)
  자식 → 부모     ❌ 불가

$bindable:
  부모 ↔ 자식     (양방향)
```

`$bindable`은 자식이 받은 prop을 **변경하면 부모의 원본 상태도 같이 바뀌게** 해주는 rune이다.

---

## React 비교로 이해하기

React에서 input 값을 부모-자식 간 공유하려면:

```tsx
// React — 콜백으로 올려보내기 (lifting state up)
function Parent() {
  const [name, setName] = useState('');
  return <Child value={name} onChange={setName} />;
}

function Child({ value, onChange }) {
  return <input value={value} onChange={(e) => onChange(e.target.value)} />;
}
```

Svelte에서 같은 동작:

```svelte
<!-- Parent.svelte -->
<script>
  import Child from './Child.svelte'
  let name = $state('')
</script>

<Child bind:value={name} />
<p>부모가 보는 값: {name}</p>
```

```svelte
<!-- Child.svelte -->
<script>
  let { value = $bindable('') } = $props()
</script>

<input bind:value />
```

React의 `value` + `onChange` 두 개가 Svelte에서는 `bind:value` 하나로 끝난다.

---

## 단계별 이해

### 1단계: $bindable 없이 — 단방향

```svelte
<!-- Child.svelte -->
<script>
  let { count } = $props()
</script>

<!-- 자식이 이 값을 바꾸면? -->
<button onclick={() => count++}>+1</button>
<p>자식: {count}</p>
```

```svelte
<!-- Parent.svelte -->
<script>
  import Child from './Child.svelte'
  let count = $state(0)
</script>

<Child {count} />
<p>부모: {count}</p>
```

```
결과:
  자식에서 버튼 클릭 → 자식의 count만 바뀜
  부모의 count는 그대로 0
  → ⚠️ ownership 경고 발생 (부모 소유 상태를 자식이 변경)
```

### 2단계: $bindable 추가 — 양방향

```svelte
<!-- Child.svelte -->
<script>
  let { count = $bindable(0) } = $props()
</script>

<button onclick={() => count++}>+1</button>
<p>자식: {count}</p>
```

```svelte
<!-- Parent.svelte -->
<script>
  import Child from './Child.svelte'
  let count = $state(0)
</script>

<Child bind:count />   <!-- bind: 추가 -->
<p>부모: {count}</p>
```

```
결과:
  자식에서 버튼 클릭 → count++ → 부모의 count도 같이 변경
  부모: 1, 자식: 1 (동기화됨)
```

---

## 동작 원리

`$bindable` + `bind:`가 만나면 **부모와 자식이 같은 상태를 참조**한다.

```
$bindable 없이:
  부모 count (0)  →  복사  →  자식 count (0)
  자식이 바꿔도 부모 영향 없음 (별개의 값)

$bindable + bind:
  부모 count (0)  ←→  자식 count
  같은 메모리를 참조. 한쪽이 바꾸면 양쪽 다 변경
```

---

## bind: 없이 $bindable 쓰면?

`$bindable`로 선언해도 **부모가 `bind:`를 안 쓰면 단방향**이다.

```svelte
<!-- 부모가 bind: 안 쓴 경우 -->
<Child count={5} />   <!-- 그냥 prop 전달 — 단방향 -->
<Child bind:count />  <!-- bind: 사용 — 양방향 -->
```

`$bindable`은 "양방향 가능"이라고 **허용**하는 것이지, **강제**하는 것이 아니다. 부모가 `bind:`를 써야 실제로 양방향이 된다.

---

## $bindable 기본값

```ts
// 기본값 없음 — bind:도 없고 prop도 안 넘기면 undefined
let { value = $bindable() } = $props()

// 기본값 있음 — bind:도 없고 prop도 안 넘기면 'fallback' 사용
let { value = $bindable('fallback') } = $props()
```

---

## 실전 예시: 모달 열기/닫기

가장 자연스러운 $bindable 사용처 — 부모가 열고, 자식(모달)이 스스로 닫아야 할 때.

```svelte
<!-- Modal.svelte -->
<script>
  let { open = $bindable(false) } = $props()
</script>

{#if open}
  <div class="overlay">
    <div class="modal">
      <slot />
      <button onclick={() => open = false}>닫기</button>
    </div>
  </div>
{/if}
```

```svelte
<!-- Page.svelte -->
<script>
  import Modal from './Modal.svelte'
  let showModal = $state(false)
</script>

<button onclick={() => showModal = true}>모달 열기</button>

<Modal bind:open={showModal}>
  <p>모달 내용</p>
</Modal>
```

```
흐름:
  부모: showModal = true  → 모달 열림
  모달: open = false      → showModal도 false로 → 모달 닫힘

콜백 방식이었다면:
  <Modal open={showModal} onClose={() => showModal = false}>
  → 부모가 onClose 콜백을 일일이 전달해야 함
```

---

## 콜백 vs $bindable — 언제 어떤 걸 쓰나

| 기준 | 콜백 (onChange) | $bindable (bind:) |
|---|---|---|
| 부모가 변경 로직을 제어 | 검증, 포맷팅, 조건부 거부 등 | 불가 |
| 단순 값 동기화 | 보일러플레이트 많음 | 깔끔 |
| 디버깅 | 콜백에서 추적 가능 | 어디서 바뀌었는지 추적 어려움 |
| 적합한 곳 | 폼 검증, 필터링 | input 래퍼, 모달 open/close, 탭 index |

```svelte
<!-- 콜백이 맞는 경우 — 부모가 값을 검증/변환해야 할 때 -->
<SearchInput
  value={query}
  onChange={(v) => {
    if (v.length > 100) return  // 100자 초과 거부
    query = v.trim()            // 공백 제거 후 저장
  }}
/>

<!-- $bindable이 맞는 경우 — 단순 동기화 -->
<Modal bind:open={showModal} />
<ColorPicker bind:color={selectedColor} />
<Tabs bind:activeIndex={currentTab} />
```

---

## 주의사항

```
1. 남용 금지
   양방향 바인딩을 많이 쓰면 데이터 흐름이 복잡해진다.
   "어디서 이 값이 바뀌는 거지?" → 추적이 어려워짐.
   Svelte 공식 문서도 "sparingly and carefully" 권고.

2. ownership 경고
   $bindable 없이 부모 소유 상태를 자식이 변경하면 경고 발생.
   → $bindable + bind:를 쓰거나, 콜백 방식으로 변경.

3. Proxy 객체 mutation
   $bindable로 전달된 객체는 자식에서 mutation 가능.
   let { user = $bindable() } = $props()
   user.name = 'new'  // 부모의 user.name도 변경됨
   → 객체 전체가 양방향이 되므로 의도치 않은 변경 주의.
```
