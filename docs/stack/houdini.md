# Houdini — SvelteKit GraphQL 클라이언트

> 대상: GraphQL 경험이 없는 React 개발자. GraphQL 기초부터 시작한다.

---

## 1. GraphQL 기초 (Houdini 이전에)

### REST vs GraphQL

REST API는 서버가 정해준 형태의 데이터를 받는다. 예를 들어 `/api/users/1`을 호출하면
서버가 준비한 필드를 모두 받는다. 필요 없는 필드도 포함되고, 반대로 관련 데이터를
얻으려면 `/api/users/1/posts` 같은 추가 요청이 필요하다.

GraphQL은 **클라이언트가 원하는 필드를 선언하면 서버가 그것만 돌려준다**. 엔드포인트는
단 하나(`/graphql`)이고 요청 자체가 "쿼리 문서"다.

```
REST                        GraphQL
/users/1           →        query { user(id: "1") { id name } }
/users/1/posts     →        query { user(id: "1") { posts { title } } }
```

두 번 요청하던 것을 한 번에 처리할 수 있다.

---

### Query, Mutation, Fragment

| 용어 | 의미 | React 비유 |
|------|------|-----------|
| Query | 데이터 읽기 | GET 요청 |
| Mutation | 데이터 쓰기/변경 | POST/PUT/DELETE |
| Fragment | 재사용 가능한 필드 조각 | 공통 props 인터페이스 |

**기본 쿼리 문법**

```graphql
query GetUser {
  user(id: "1") {
    id
    name
    email
  }
}
```

`GetUser`는 쿼리 이름(선택 사항이지만 권장). 중괄호 안이 "원하는 필드 목록"이다.

**뮤테이션 문법**

```graphql
mutation CreatePost($title: String!, $body: String!) {
  createPost(title: $title, body: $body) {
    id
    title
  }
}
```

`$title`은 변수. `String!`은 "null 불가 String" 타입이다.

---

### Schema 타입 개념

GraphQL 서버는 **Schema**를 통해 어떤 타입이 존재하는지 선언한다.

```graphql
type User {
  id: ID!
  name: String!
  email: String
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  author: User!
}
```

클라이언트는 Schema에 정의된 필드만 요청할 수 있다. 타입스크립트의 인터페이스와
유사하다고 보면 된다.

---

## 2. Houdini란?

Houdini는 SvelteKit에 특화된 GraphQL 클라이언트다.
Apollo Client나 URQL처럼 런타임 라이브러리이기도 하지만, 핵심 차이는 **컴파일 타임**에
동작한다는 점이다.

```
일반 GraphQL 클라이언트 (Apollo 등)
코드 작성 → 별도 codegen 실행 → 타입 생성 → 런타임에 타입 사용

Houdini
코드 작성 → 빌드 시 자동으로 타입 + 스토어 생성 → 런타임에 타입 사용
```

**핵심 장점 세 가지**

1. **보일러플레이트 제로** — GraphQL 쿼리를 작성하면 TypeScript 타입이 자동 생성된다.
   `interface UserData { ... }` 같은 코드를 손으로 쓸 필요가 없다.

2. **SvelteKit `load` 함수 자동 연결** — `@load` 디렉티브 하나로 쿼리가 SvelteKit의
   서버사이드 `load` 함수와 자동으로 통합된다.

3. **정규화 캐시 내장** — 같은 ID의 객체는 캐시의 한 곳에만 저장되어, 뮤테이션 후
   수동 리페치 없이 UI가 자동 갱신된다.

---

## 3. 기본 쿼리 — @load 디렉티브

SvelteKit의 라우트(`+page.svelte`)에서 쿼리를 작성하는 가장 단순한 패턴이다.

**파일 구조**

```
src/routes/users/
  +page.svelte    ← 컴포넌트
  +page.gql       ← 쿼리 정의 (또는 svelte 파일 내 inline)
```

**+page.gql**

```graphql
query UserList @load {
  users {
    id
    name
    email
  }
}
```

`@load`를 붙이면 Houdini가 SvelteKit의 `load` 함수를 **자동 생성**한다.
`+page.server.ts`에 `load`를 직접 작성할 필요가 없다.

**+page.svelte**

```svelte
<script lang="ts">
  import type { PageData } from './$houdini'

  let { data }: { data: PageData } = $props()

  const UserList = $derived(data.UserList)
</script>

{#each $UserList.data.users as user}
  <p>{user.name} — {user.email}</p>
{/each}
```

`PageData`는 `./$houdini`에서 자동 생성된 타입이다. `data`를 구조 분해하면
`UserList` 스토어를 얻는다. `$UserList.data`로 실제 쿼리 결과에 접근한다.

**inline 방식 (컴포넌트 파일 안에 쿼리 작성)**

라우트가 아닌 일반 컴포넌트에서는 `@load` 디렉티브와 함께 inline으로 작성한다.

```svelte
<script lang="ts">
  import { graphql } from '$houdini'

  const info = graphql(`
    query UserInfo @load {
      viewer {
        firstName
      }
    }
  `)
</script>

<p>{$info.data.viewer.firstName}</p>
```

---

## 4. Colocated Fragment 패턴 (핵심)

Fragment는 GraphQL의 "재사용 가능한 필드 조각"이다. Houdini에서는 이를 컴포넌트와
함께 배치(colocate)하는 패턴을 적극 권장한다.

### 왜 Fragment를 컴포넌트에 붙이는가?

React에서 부모가 자식에게 props를 전달할 때, 자식이 어떤 필드를 필요로 하는지
부모가 알아야 한다. 자식 컴포넌트가 늘어나면 부모 쿼리가 비대해진다.

Fragment 패턴은 **자식 컴포넌트가 자기 데이터 요구사항을 직접 선언**한다.
부모는 그 Fragment를 "사용하겠다"고 spread만 하면 된다.

### fragment() 함수 사용법

> 중요: `graphql()` 단독으로는 Fragment를 정의할 수 없다. 반드시 `fragment(prop, graphql(...))` 쌍으로 호출해야 반응형 스토어가 반환된다.

```svelte
<!-- MessageItem.svelte -->
<script lang="ts">
  import { fragment, graphql } from '$houdini'
  import type { MessageItem_message } from '$houdini'

  let { message }: { message: MessageItem_message } = $props()

  const data = $derived(fragment(message, graphql(`
    fragment MessageItem_message on Message {
      id
      content
      createdAt
      author {
        name
        avatar
      }
    }
  `)))
</script>

<article>
  <img src={$data.author.avatar} alt={$data.author.name} />
  <p>{$data.content}</p>
  <time>{$data.createdAt}</time>
</article>
```

**동작 원리**

```
컴파일 타임:
  Houdini가 fragment 이름 "MessageItem_message"를 발견
      → MessageItem_message 타입 자동 생성
      → $houdini 모듈에 export

런타임:
  부모가 message prop으로 데이터 조각을 전달
  fragment(message, graphql(...)) 호출
      → 반응형 Svelte store 반환
  $data 로 store를 구독하여 렌더링
```

**네이밍 컨벤션** — Fragment 이름은 `컴포넌트명_타입명` 형식을 따른다.
`MessageItem_message`는 `MessageItem` 컴포넌트가 `Message` 타입의 Fragment임을 나타낸다.
Houdini 컴파일러가 이 이름을 기반으로 타입을 생성한다.

---

## 5. Fragment Composition — 쿼리 조합

자식 컴포넌트가 Fragment를 선언하면, 부모 쿼리는 그 Fragment를 spread(`...`)해서
하나의 쿼리로 조합한다.

**부모 쿼리 (+page.gql)**

```graphql
query MessageList @load {
  messages {
    # MessageItem.svelte 의 Fragment를 여기서 spread
    ...MessageItem_message
  }
}
```

**부모 컴포넌트 (+page.svelte)**

```svelte
<script lang="ts">
  import type { PageData } from './$houdini'
  import MessageItem from '$lib/MessageItem.svelte'

  let { data }: { data: PageData } = $props()
  const MessageList = $derived(data.MessageList)
</script>

{#each $MessageList.data.messages as message}
  <MessageItem {message} />
{/each}
```

부모는 `message` 객체 전체를 자식에게 넘긴다. 자식은 자기 Fragment에 선언된 필드만
`$data`로 접근한다. **부모가 자식의 내부 필드를 알 필요가 없다.**

### N+1 문제 방지

```
전통적 방식:
  목록 쿼리 1회 → 각 항목마다 상세 쿼리 N회 = 1+N 요청

Fragment Composition:
  부모 쿼리 안에 자식 Fragment가 포함 → 단 1회 요청으로 모든 필드 수집
```

Houdini 컴파일러가 모든 Fragment를 수집해서 단일 쿼리로 합치기 때문에 N+1이 발생하지
않는다.

---

## 6. Mutation + 캐시 자동 갱신

### @list 디렉티브로 목록 등록

캐시가 특정 배열을 "관리 목록"으로 추적하려면 `@list` 디렉티브로 이름을 붙인다.

```graphql
query AllMessages @load {
  messages @list(name: "Message_list") {
    id
    ...MessageItem_message
  }
}
```

### 뮤테이션에서 목록 조작

목록에 항목을 추가할 때 `ListName_insert` Fragment를 사용한다.

```graphql
mutation AddMessage($content: String!) {
  addMessage(content: $content) {
    ...Message_list_insert
  }
}
```

Houdini가 뮤테이션 응답을 받으면 **자동으로** `Message_list` 캐시 배열 앞에 새 항목을
삽입한다. `refetch` 호출이 필요 없다.

**세 가지 목록 조작 패턴**

| Fragment 이름 | 동작 |
|--------------|------|
| `ListName_insert` | 목록에 항목 추가 |
| `ListName_remove` | 목록에서 항목 제거 |
| `ListName_toggle` | 목록에 있으면 제거, 없으면 추가 |

### @ItemType_delete 디렉티브

항목을 완전히 삭제할 때는 삭제된 ID에 `@TypeName_delete` 디렉티브를 붙인다.
해당 ID를 참조하는 **모든 목록**에서 자동 제거된다.

```graphql
mutation DeleteMessage($id: ID!) {
  deleteMessage(id: $id) {
    deletedId @Message_delete
  }
}
```

### 컴포넌트에서 뮤테이션 호출

```svelte
<script lang="ts">
  import { graphql } from '$houdini'

  const addMessage = graphql(`
    mutation AddMessage($content: String!) {
      addMessage(content: $content) {
        ...Message_list_insert
      }
    }
  `)

  async function handleSubmit(content: string) {
    await addMessage.mutate({ content })
    // UI는 자동으로 갱신됨 — 별도 refetch 불필요
  }
</script>
```

---

## 7. Normalized Cache

### ID 기반 정규화

Houdini 캐시는 데이터를 "테이블"처럼 저장한다.

```
일반 캐시 (목록별 저장):
  messages_page1: [{ id: "1", content: "Hello" }, ...]
  messages_page2: [{ id: "1", content: "Hello" }, ...]  ← 중복!

정규화 캐시 (ID 기반 저장):
  Message:1  → { id: "1", content: "Hello" }
  Message:2  → { id: "2", content: "World" }
  messages_page1 → [ref:Message:1, ref:Message:2, ...]
  messages_page2 → [ref:Message:1, ...]               ← 참조만
```

`Message:1` 항목이 업데이트되면 이 ID를 참조하는 **모든 쿼리가 동시에 갱신**된다.
화면 여러 곳에 같은 유저 정보가 있어도 하나만 수정하면 전체가 동기화된다.

### @cache 정책

쿼리마다 캐시 동작을 다르게 설정할 수 있다.

```graphql
query FrequentlyUpdated @cache(policy: NetworkOnly) {
  liveScores {
    score
  }
}

query RarelyChanged @cache(policy: CacheOnly) {
  siteConfig {
    theme
  }
}
```

| 정책 | 동작 |
|------|------|
| `CacheAndNetwork` | 캐시를 먼저 보여주고 네트워크 응답으로 갱신 (기본값) |
| `NetworkOnly` | 항상 네트워크 요청 |
| `CacheOnly` | 캐시만 사용, 네트워크 요청 없음 |
| `NoCache` | 캐시 미사용, 응답도 저장 안 함 |

### 수동 리페치가 불필요한 이유

```
뮤테이션 실행
    → Houdini가 응답 데이터를 캐시에 반영
    → 영향받은 캐시 엔트리를 구독 중인 스토어에 알림
    → 스토어가 갱신되어 UI 자동 리렌더
```

이 흐름이 자동으로 일어나기 때문에 Apollo의 `cache.modify`나 수동 `refetch` 없이도
데이터 일관성이 유지된다.

---

## 8. Connection Pattern — 커서 기반 페이지네이션

### Connection 구조란?

GraphQL에서 리스트 페이지네이션은 `edges / node / pageInfo` 구조를 사용하는 것이 표준이다.
REST의 `?page=2` 방식(오프셋 기반)과 달리, 커서 기반은 "이 항목 이후부터" 방식으로 동작한다.

```
오프셋 기반 (REST):    ?page=2&limit=10
커서 기반 (GraphQL):   friends(first: 10, after: "cursor_abc")
```

오프셋 방식은 중간에 항목이 추가·삭제되면 같은 항목이 중복되거나 빠질 수 있다.
커서 방식은 특정 항목을 기준점으로 삼기 때문에 목록이 변해도 일관성이 보장된다.

### Connection 스키마 구조

```graphql
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

`edges`는 커서와 실제 데이터(`node`)를 함께 담는 래퍼 배열이다.
`pageInfo`는 다음/이전 페이지 존재 여부를 알려준다.

### @paginate 디렉티브

Houdini에서 커서 페이지네이션은 `@paginate` 디렉티브 하나로 처리한다.

```graphql
query UserList @load {
  friends(first: 10) @paginate {
    edges {
      node {
        id
        name
        email
      }
    }
  }
}
```

`@paginate`를 붙이면 Houdini가 쿼리 스토어에 자동으로 메서드를 추가한다.

```typescript
// 자동 생성되는 스토어 메서드
loadNextPage(pageCount?: number, after?: string): Promise<void>
loadPreviousPage(pageCount?: number, before?: string): Promise<void>
pageInfo: Readable<PageInfo>
```

### 컴포넌트 사용 패턴

```svelte
<script lang="ts">
  import type { PageData } from './$houdini'

  let { data }: { data: PageData } = $props()

  const UserList = $derived(data.UserList)
</script>

{#each $UserList.data.friends.edges as { node: user }}
  <p>{user.name} — {user.email}</p>
{/each}

{#if $UserList.pageInfo.hasNextPage}
  <button onclick={() => UserList.loadNextPage()}>
    더 보기
  </button>
{/if}
```

**동작 흐름**

```
초기 렌더링: friends(first: 10) 요청
    → 첫 10개 항목 + pageInfo.endCursor 수신

loadNextPage() 호출
    → friends(first: 10, after: endCursor) 요청
    → Houdini가 기존 목록에 자동 append
    → UI 자동 갱신 (수동 상태 관리 불필요)
```

Normalized Cache 덕분에 새로 로드된 항목은 캐시에 병합되고,
동일 항목이 다른 쿼리에 존재하면 그쪽도 함께 갱신된다.

---

## 9. React + Apollo vs SvelteKit + Houdini 비교

| 개념 | React + Apollo | SvelteKit + Houdini |
|------|---------------|---------------------|
| 쿼리 실행 | `useQuery` hook | `@load` 디렉티브 |
| 타입 생성 | `graphql-codegen` 별도 실행 | 컴파일 타임 자동 생성 |
| 쿼리-라우트 연결 | `getServerSideProps` 수동 작성 | `@load`로 자동 연결 |
| Fragment | 수동 타입 선언 필요 | `fragment()` + 자동 타입 |
| 뮤테이션 후 UI 갱신 | `cache.modify` 또는 `refetch` | `@list` + `_insert` 패턴 |
| 항목 삭제 후 목록 갱신 | `cache.evict` | `@TypeName_delete` 디렉티브 |
| 캐시 정책 제어 | `fetchPolicy` 옵션 | `@cache(policy:)` 디렉티브 |
