# Load 무효화 (Invalidation)

load 함수의 재실행 조건과 수동 무효화 메커니즘. `depends()`, `invalidate()`, `invalidateAll()` 사용 방법.

---

## 1. Load 무효화란

SvelteKit의 load 함수는 **자동으로** 다음 상황에서 재실행된다.

- 참조한 `params` 값이 변경된 경우 (예: `/blog/[id]`의 `id` 변경)
- 참조한 `url.pathname`, `url.search` 등이 변경된 경우
- `await parent()`를 호출했고 부모 load가 재실행된 경우
- `fetch(url)`로 요청한 URL이 무효화된 경우 (universal load만)

자동 재실행이 **발생하지 않는** 경우:

- 외부 데이터가 변경되었지만 URL이나 params는 그대로인 경우
- 다른 컴포넌트의 사용자 액션으로 서버 데이터가 바뀐 경우
- 주기적으로 최신 데이터를 유지해야 하는 경우

이런 시나리오에서는 **수동 무효화**가 필요하다.

```text
자동 재실행 트리거:
  params 변경  ─┐
  url 변경     ─┼─→  load 재실행
  부모 load    ─┘

수동 재실행 트리거:
  invalidate(key)     ─┐
  invalidateAll()     ─┴─→  load 재실행
```

> 재실행 시 컴포넌트는 **다시 생성되지 않는다** — `data` prop만 업데이트된다. 컴포넌트 내부 상태(입력 필드 값 등)는 그대로 유지된다.

---

## 2. `depends()` — 커스텀 키 등록

load 함수 안에서 `depends()`를 호출하면, 이 load가 해당 키에 **의존한다고 등록**된다. 나중에 `invalidate(key)`를 호출하면 이 load가 재실행된다.

```typescript
// +page.ts
import type { PageLoad } from './$types'

export const load: PageLoad = async ({ fetch, depends }) => {
  depends('app:messages')  // 커스텀 키 등록

  const messages = await fetch('/api/messages').then(r => r.json())
  return { messages }
}
```

### 키 명명 규칙

커스텀 키는 **소문자 알파벳 + 콜론** 접두사 형식(`[a-z]:`)이어야 한다.

```text
올바른 예:
  app:messages
  data:user-profile
  cache:products

잘못된 예:
  messages          <- 접두사 없음
  APP:messages      <- 대문자 불가
  /api/messages     <- URL 형식은 depends()가 아닌 fetch()로 자동 등록
```

### 여러 키 등록

한 load 함수에 여러 키를 등록할 수 있다.

```typescript
export const load: PageLoad = async ({ fetch, depends }) => {
  depends('app:messages')
  depends('app:user')

  // 두 키 중 하나만 invalidate되어도 이 load가 재실행됨
  const [messages, user] = await Promise.all([
    fetch('/api/messages').then(r => r.json()),
    fetch('/api/user').then(r => r.json()),
  ])
  return { messages, user }
}
```

> universal load(`+page.ts`)에서 `fetch(url)`을 호출하면 해당 URL이 자동으로 의존성으로 등록된다. server load(`+page.server.ts`)는 **자동 등록되지 않는다** — 민감한 URL이 클라이언트에 노출될 수 있기 때문이다. server load에서 수동 무효화가 필요하면 반드시 `depends()`로 커스텀 키를 등록해야 한다.

---

## 3. `invalidate()` — 특정 load만 재실행

`$app/navigation`에서 가져온 `invalidate()`는 **해당 키 또는 URL에 의존하는 load만** 재실행한다.

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import { invalidate } from '$app/navigation'
  import type { PageProps } from './$types'

  let { data }: PageProps = $props()

  async function refresh() {
    await invalidate('app:messages')
  }
</script>

<button onclick={refresh}>새로고침</button>
<ul>
  {#each data.messages as msg}
    <li>{msg.text}</li>
  {/each}
</ul>
```

### 세 가지 호출 방식

```typescript
import { invalidate } from '$app/navigation'

// 1. 커스텀 키로 무효화 — depends()로 등록한 키와 매칭
await invalidate('app:messages')

// 2. URL 문자열로 무효화 — 해당 URL을 fetch한 load가 재실행됨
await invalidate('/api/messages')

// 3. URL 패턴 함수로 무효화 — 조건에 맞는 URL 의존성 전체를 무효화
await invalidate(url => url.pathname.startsWith('/api/'))
```

방식 3은 여러 API 엔드포인트에 걸친 load를 한 번에 재실행할 때 유용하다.

---

## 4. `invalidateAll()` — 모든 load 재실행

현재 페이지에서 활성화된 **모든** load 함수를 재실행한다.

```typescript
import { invalidateAll } from '$app/navigation'

await invalidateAll()
```

### invalidate vs invalidateAll 선택 기준

```text
invalidate(key) 사용 시나리오:
  - 특정 데이터만 변경됨 (예: 메시지 목록)
  - 레이아웃 load는 그대로 두고 페이지 load만 갱신
  - 성능 최적화가 필요한 경우

invalidateAll() 사용 시나리오:
  - 어떤 load가 영향받는지 명확하지 않은 경우
  - 사용자 인증 상태 변경 (로그인/로그아웃)
  - 전체 페이지 데이터를 일괄 갱신해야 할 때
```

`invalidateAll()`은 편리하지만 **불필요한 네트워크 요청**을 유발할 수 있다. 레이아웃 load까지 포함하여 모든 load가 재실행되므로, 영향 범위를 알고 있다면 `invalidate()`를 우선 사용하는 것이 좋다.

---

## 5. Form Actions + 무효화 통합 패턴

`use:enhance`를 사용한 Form Action은 제출 성공 후 **자동으로 현재 페이지의 load를 재실행**한다. 별도로 `invalidate`를 호출할 필요가 없다.

```svelte
<form method="POST" use:enhance>
  <input name="text" />
  <button>메시지 전송</button>
</form>
<!-- 전송 성공 시 현재 페이지 load 자동 재실행 -->
```

### 다른 페이지의 캐시를 무효화해야 할 때

Form Action 완료 후 **다른 경로의 데이터**도 함께 갱신해야 한다면 `enhance`의 콜백에서 수동으로 처리한다.

```svelte
<script lang="ts">
  import { enhance } from '$app/forms'
  import { invalidate } from '$app/navigation'
</script>

<form
  method="POST"
  use:enhance={() => {
    return async ({ update }) => {
      await update()               // 현재 페이지 load 재실행 (기본 동작)
      await invalidate('app:cart') // 다른 컴포넌트의 장바구니 데이터도 갱신
    }
  }}
>
  <button>구매</button>
</form>
```

커스텀 fetch + `deserialize`를 쓰는 경우에는 `invalidateAll()`을 직접 호출한다.

```typescript
if (result.type === 'success') {
  await invalidateAll()
}
```

---

## 6. 실전 패턴: 폴링 대신 무효화

주기적으로 데이터를 갱신할 때 `setInterval` + 직접 fetch 대신, `invalidate()`를 이용하면 load 함수의 캐싱·에러 처리 로직을 그대로 재활용할 수 있다.

```svelte
<script lang="ts">
  import { onMount } from 'svelte'
  import { invalidate } from '$app/navigation'
  import type { PageProps } from './$types'

  let { data }: PageProps = $props()

  onMount(() => {
    const timer = setInterval(() => {
      invalidate('app:messages')  // load 재실행 → data.messages 자동 업데이트
    }, 5000)

    return () => clearInterval(timer)
  })
</script>
```

```text
기존 폴링 방식:
  setInterval → fetch → 상태 수동 업데이트 → 에러 처리 직접

무효화 방식:
  setInterval → invalidate() → load 재실행 → data prop 자동 업데이트
                                └── 에러 처리, 캐싱 → load 함수가 담당
```

> 실시간 요구 사항이 높다면 WebSocket이나 SSE(Server-Sent Events)를 사용하되, 그 이벤트 핸들러 안에서 `invalidate()`를 호출하는 패턴도 유효하다.

---

## React Query와 비교

| 개념 | React Query | SvelteKit |
|--|--|--|
| 데이터 페칭 | `useQuery` | load 함수 |
| 의존성 선언 | `queryKey` 배열 | `depends(key)` |
| 특정 쿼리 무효화 | `queryClient.invalidateQueries({ queryKey })` | `invalidate(key)` |
| 전체 무효화 | `queryClient.invalidateQueries()` | `invalidateAll()` |
| 자동 재실행 조건 | 윈도우 포커스, staleTime 만료 | URL/params 변경, 상위 load 변경 |
| 폴링 | `refetchInterval` 옵션 | `setInterval` + `invalidate()` |
| 뮤테이션 후 갱신 | `onSuccess: () => queryClient.invalidateQueries(...)` | Form Action 자동 재실행 또는 `enhance` 콜백 |
