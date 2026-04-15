# GraphQL — 클라이언트 주도형 API 쿼리 언어

> 대상: REST API 경험자로 GraphQL을 처음 접하는 개발자.

---

## 1. 개요

GraphQL은 단일 엔드포인트(`/graphql`)에 쿼리를 보내 원하는 필드만 받는 API 규약이다.
서버가 Schema로 타입을 선언하고 클라이언트가 요청 구조를 직접 정의한다. 응답 구조가 쿼리와 1:1 대응되므로 over-fetch / under-fetch가 없다.

## 2. Schema & Type System

서버는 SDL(Schema Definition Language)로 타입을 선언한다.

```graphql
# Root Type — 클라이언트 진입점
type Query {
  hero(episode: Episode): Character
  human(id: ID!): Human
}
type Mutation {
  createReview(episode: Episode, review: ReviewInput!): Review
}

# Object Type
type Character {
  id:        ID!
  name:      String!
  appearsIn: [Episode!]!   # 배열 + 요소 모두 non-null
  friends:   [Character]
}

# Enum
enum Episode { NEWHOPE  EMPIRE  JEDI }

# Input Type — Mutation 인수 전용
input ReviewInput {
  stars:      Int!
  commentary: String
}
```

- `!` — Non-null: 해당 필드는 절대 `null`이 될 수 없다.
- 내장 Scalar: `Int` `Float` `String` `Boolean` `ID`. 커스텀 추가 시 `scalar DateTime`.

## 3. Query / Mutation

```graphql
# 기본 쿼리 — 중첩 필드 한 번에 요청
query GetUser {
  user(id: "1") {
    id
    name
    friends { name }
  }
}

# Mutation + Input Type
mutation CreateReview($review: ReviewInput!) {
  createReview(episode: JEDI, review: $review) {
    stars
    commentary
  }
}
```

Mutation도 반환 타입을 선언하므로 저장 후 결과를 바로 받는다.

## 4. Subscription

Subscription은 서버가 클라이언트로 실시간 이벤트를 푸시하는 오퍼레이션이다. WebSocket 등 지속 연결 위에서 동작한다.

```graphql
subscription NewMessages {
  newMessage(roomId: 123) { sender  text }
}
```

## 5. Variables

인라인 리터럴 대신 변수를 쓰면 쿼리 문자열을 재사용하고 파싱 캐시를 활용한다.

```graphql
query GetProfile($devicePicSize: Int, $userId: ID!) {
  user(id: $userId) {
    profilePic(size: $devicePicSize)
  }
}
```

`$변수명: 타입` 형태로 오퍼레이션 상단에 선언하고, 실제 값은 별도 `variables` JSON으로 전달한다. `!`를 붙이면 필수 변수다. 변수 타입은 **Input 계열만 가능** (Scalar · Enum · Input Object). Object Type은 쓸 수 없다.

## 6. Fragment

```graphql
# Named Fragment — 여러 쿼리에서 필드 집합 재사용
fragment UserFields on User {
  id  firstName  lastName
}
query { user(id: 1) { ...UserFields } }

# Inline Fragment — 타입별 분기 처리
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    ... on Droid { primaryFunction }
    ... on Human { homePlanet }
  }
}
```

`... on Type` 블록 안의 필드는 해당 타입일 때만 포함된다.

## 7. Error 처리

GraphQL은 오류가 있어도 HTTP 상태는 **200**이다. 응답 JSON의 `errors` 배열을 확인해야 한다.

```json
{
  "errors": [{
    "message": "Name could not be fetched.",
    "locations": [{ "line": 6, "column": 7 }],
    "path": ["hero", "friends", 1, "name"],
    "extensions": { "code": "CAN_NOT_FETCH" }
  }],
  "data": { "hero": { "name": "R2-D2" } }
}
```

`data`와 `errors`가 공존할 수 있다. 부분 성공 시 나머지 결과는 `data`에 담긴다.

## 8. Directive

`@include(if: Bool)` — `true`일 때만 필드 포함. `@skip(if: Bool)` — `true`이면 필드 제외.

```graphql
query MyQuery($show: Boolean!, $skip: Boolean!) {
  user(id: 1) {
    friends @include(if: $show) { name }
    bio     @skip(if: $skip)
  }
}
```

---

## 요약

| 개념         | 용도                    | 비고                           |
|--------------|-------------------------|--------------------------------|
| Schema / SDL | 서버 타입 정의          | Object · Scalar · Enum · Input |
| `!`          | Non-null 선언           | 필드·변수 모두 적용            |
| Query        | 데이터 읽기             | 중첩 필드 한 번에 요청         |
| Mutation     | 데이터 쓰기/변경        | Input Type 인수 활용           |
| Subscription | 실시간 이벤트 수신      | WebSocket 기반                 |
| Variables    | 쿼리 파라미터 분리      | 파싱 캐시 재사용               |
| Fragment     | 필드 집합 재사용        | Named / Inline                 |
| `errors[]`   | HTTP 200 안의 오류 배열 | `data`와 공존 가능             |
| Directive    | 조건부 필드 제어        | `@include` / `@skip`           |
