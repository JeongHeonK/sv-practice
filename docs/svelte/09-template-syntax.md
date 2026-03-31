# 템플릿 문법

---

## {#each} — 배열 반복

React의 `array.map()`에 해당. **항상 key를 지정**한다.

```svelte
<ul>
  {#each notifications as notification (notification.id)}
    <li>{notification.title}</li>
  {/each}
</ul>
```

> key가 없으면 인덱스 기반으로 DOM을 재사용해 항목 삭제/이동 시 잘못된 요소가 남는다. React의 `key` prop과 같은 이유. **인덱스를 key로 쓰면 안 된다.**

### 인덱스 접근

```svelte
{#each notifications as notification, index (notification.id)}
  <li>{index}: {notification.title}</li>
{/each}
```

### 구조분해

```svelte
{#each notifications as { id, title, body } (id)}
  <li>
    <h5>{title}</h5>
    <p>{body}</p>
  </li>
{/each}
```

### {:else} — 빈 배열

```svelte
{#each notifications as n (n.id)}
  <li>{n.title}</li>
{:else}
  <p>알림이 없습니다.</p>
{/each}
```

---

## {@const} — 템플릿 내 상수

루프/if/snippet 안에서 중복 계산을 피할 때 사용한다.

```svelte
{#each notifications as { title, date } (title)}
  {@const dateObj = new Date(date)}
  <li>
    <h5>{title}</h5>
    <time datetime={dateObj.toISOString()}>{dateObj.toLocaleDateString()}</time>
  </li>
{/each}
```

`{#each}`, `{#if}`, `{#snippet}` 등의 **직접 자식**이어야 한다.

---

## 배열 뮤테이션

React에서는 `setItems([...items, newItem])`처럼 새 배열을 만들어야 한다. Svelte `$state` 배열은 Proxy이므로 **직접 변경**해도 반응성이 동작한다.

```js
let notifications = $state(generateNotifications(10))

notifications.splice(index, 1)     // ✓ 요소 제거
notifications.push(newItem)        // ✓ 요소 추가
notifications[0].title = 'new'     // ✓ 중첩 프로퍼티 수정
```

### key 없는 {#each}의 함정

key가 없으면 `splice(0, 1)`로 첫 요소를 제거해도 **화면에서는 마지막 요소가 사라지는** 것처럼 보인다.

```text
배열:  [A, B, C]  →  splice(0,1)  →  [B, C]

key 없음: 인덱스 기준 재사용 → 0번에 B, 1번에 C 덮어씀 → 2번(C 노드) 제거
key 있음: 항목 기준 매핑 → A 노드 정확히 제거
```

React에서 `key`를 빼먹으면 생기는 문제와 정확히 같다.
