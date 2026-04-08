# Svelte 내장 리액티브 클래스

Svelte는 자주 쓰이는 패턴을 위한 **내장 리액티브 클래스**를 제공한다.
직접 `$state` + `$effect`를 조합할 필요 없이 import해서 바로 사용.

---

## 핵심 원리: 왜 내장 클래스가 필요한가?

### 일반 객체의 문제

```ts
let date = new Date()

// Date 메서드로 값을 변경해도 Svelte가 감지하지 못한다
setInterval(() => date.setSeconds(date.getSeconds() + 1), 1000)
// → UI 갱신 안 됨!
```

```
$state는 Proxy로 프로퍼티 변경을 감지한다:
  user.name = 'kim'     // ✅ Proxy set trap → 감지됨

그런데 Date의 setSeconds()는 왜 안 되는가?

Date, Map, Set 같은 내장 객체는 "내부 슬롯(internal slot)"에 데이터를 저장한다.
  - Date → [[DateValue]] 슬롯에 타임스탬프 저장
  - Map  → [[MapData]] 슬롯에 엔트리 저장

setSeconds()는 프로퍼티를 바꾸는 게 아니라 내부 슬롯을 직접 변경한다.
Proxy의 set trap은 프로퍼티 변경만 가로채므로, 내부 슬롯 변경은 감지 불가.

let date = $state(new Date())
date.setSeconds(...)    // ❌ 내부 슬롯 변경 → Proxy가 가로챌 수 없음
date.someProperty = 1   // ✅ 프로퍼티 변경 → Proxy set trap 동작
date = new Date()       // ✅ 재할당 → 감지됨
```

### 해결: 매번 새 객체를 재할당

```ts
let date = $state(new Date())

$effect(() => {
  const timer = setInterval(() => {
    date = new Date()  // 새 객체 재할당 → 반응성 동작
  }, 1000)
  return () => clearInterval(timer)
})
```

작동하지만 매번 `$effect` + `setInterval` + 클린업을 직접 작성해야 한다.

### 내장 리액티브 클래스 = 이 보일러플레이트를 캡슐화

```ts
import { SvelteDate } from 'svelte/reactivity'

const date = new SvelteDate()  // 끝! 메서드 호출 시 자동 반응
```

---

## SvelteDate

`Date`를 확장한 리액티브 버전. `setSeconds()` 등 메서드 호출 시 UI 자동 갱신.

```ts
import { SvelteDate } from 'svelte/reactivity'

const date = new SvelteDate()

// 메서드 호출만으로 반응성 동작 — 재할당 불필요
setInterval(() => date.setSeconds(date.getSeconds() + 1), 1000)
```

```svelte
<p>{date.getHours()}:{date.getMinutes()}:{date.getSeconds()}</p>
<!-- 매초 자동 갱신됨 -->
```

### 일반 Date vs SvelteDate

```
일반 Date + $state:
  date.setSeconds(...)  → 내부 변경 → UI 갱신 안 됨 ❌
  date = new Date()     → 재할당 → UI 갱신 ✅ (하지만 매번 새 객체 생성)

SvelteDate:
  date.setSeconds(...)  → 내부적으로 반응성 트리거 → UI 갱신 ✅
  setter 메서드 호출만으로 충분
```

---

## SvelteMap, SvelteSet

`Map`과 `Set`의 리액티브 버전. `.set()`, `.delete()` 등 메서드 호출 시 반응.

```ts
import { SvelteMap, SvelteSet } from 'svelte/reactivity'

const map = new SvelteMap()
map.set('key', 'value')  // → 반응성 동작

const set = new SvelteSet()
set.add('item')          // → 반응성 동작
```

일반 `Map`/`Set`은 `$state`로 감싸도 `.set()`, `.add()` 같은 메서드 호출을 감지하지 못한다.
(`$state`의 Proxy는 일반 객체/배열의 프로퍼티 변경만 추적)

---

## SvelteURL, SvelteURLSearchParams

URL 조작 시 반응성이 필요할 때 사용.

```ts
import { SvelteURL } from 'svelte/reactivity'

const url = new SvelteURL('https://example.com/path')

// 프로퍼티 변경 시 반응
url.protocol = 'http:'     // → href 자동 갱신
url.pathname = '/new-path'  // → href 자동 갱신
```

```svelte
<p>현재 URL: {url.href}</p>
<input bind:value={url.pathname} />
<input bind:value={url.protocol} />
<!-- 입력 변경 → url.href 자동 업데이트 -->
```

---

## MediaQuery

CSS 미디어 쿼리의 결과를 반응형 값으로 제공.

```ts
import { MediaQuery } from 'svelte/reactivity'

const large = new MediaQuery('min-width: 800px')
```

```svelte
{#if large.current}
  <p>큰 화면</p>
{:else}
  <p>작은 화면</p>
{/if}
```

`large.current`가 화면 크기 변경 시 자동 업데이트.

> CSS로 처리할 수 있는 건 CSS로 하는 것이 우선. JS에서 분기가 필요한 경우(컴포넌트 자체를 바꾸는 등)에만 사용.

---

## svelte/motion — Tween, Spring

애니메이션용 리액티브 클래스. 값을 시간에 따라 부드럽게 전환.

### Tween (이징 기반 애니메이션)

```ts
import { Tween } from 'svelte/motion'

let progress = new Tween(0, {
  duration: 400,    // ms
  easing: cubicOut  // 이징 함수
})

// target을 설정하면 current가 시간에 따라 변화
progress.target = 1
```

```svelte
<progress value={progress.current} />
<!-- 0에서 1로 부드럽게 애니메이션 -->
```

```
target = 1 설정 시:
  current: 0 → 0.1 → 0.3 → 0.6 → 0.85 → 0.95 → 1.0
            ├────── duration(400ms) 동안 이징 곡선 ──────┤
```

### Spring (물리 기반 애니메이션)

```ts
import { Spring } from 'svelte/motion'

let coords = new Spring({ x: 0, y: 0 }, {
  stiffness: 0.1,  // 강성 (높을수록 빠름)
  damping: 0.5     // 감쇠 (낮을수록 바운스)
})

// target 설정 → 스프링 물리로 current가 변화
coords.target = { x: 100, y: 200 }
```

```svelte
<div style="left: {coords.current.x}px; top: {coords.current.y}px" />
```

### Tween vs Spring

```
┌──────────┬─────────────────────┬─────────────────────┐
│          │ Tween               │ Spring              │
├──────────┼─────────────────────┼─────────────────────┤
│ 방식     │ 시간 기반 (duration) │ 물리 기반 (stiffness)│
│ 느낌     │ 예측 가능한 전환     │ 자연스러운 바운스    │
│ 옵션     │ duration, easing    │ stiffness, damping  │
│ 적합한 곳│ 프로그레스 바, 페이드 │ 드래그, 좌표 이동    │
└──────────┴─────────────────────┴─────────────────────┘
```

---

## 내장 리액티브 클래스 공통 패턴

### target과 current

Tween, Spring은 **target/current 패턴**을 사용:

```
target = 원하는 최종값 (즉시 설정)
current = 현재 보간된 값 (시간에 따라 변화, 읽기 전용)
```

### 모든 내장 클래스의 공통점

```
1. import해서 new로 인스턴스 생성
2. 메서드 호출이나 프로퍼티 변경 → 내부적으로 반응성 트리거
3. $state + $effect 보일러플레이트를 캡슐화
4. 구조분해 금지 (리액티브 클래스 규칙 동일)
```

---

## createSubscriber -- 외부 이벤트를 반응성에 연결

`svelte/reactivity`에서 제공. 외부 이벤트(스크롤, WebSocket 등)를 Svelte 반응성 시스템에 연결하는 함수.

### 문제: 브라우저 API는 반응적이지 않다

```svelte
<h1>{window.scrollY}</h1>
<!-- 스크롤해도 0 고정 — window.scrollY는 일반 브라우저 API, 반응성 없음 -->
```

`$effect`로 해결할 수 있지만:

```js
let scrollY = $state(window.scrollY)

$effect(() => {
  const handler = () => scrollY = window.scrollY
  window.addEventListener('scroll', handler)
  return () => window.removeEventListener('scroll', handler)
})
```

**문제점**: 다른 컴포넌트에서도 scrollY가 필요하면 동일한 코드를 반복해야 하고, 구독이 중복된다.

### createSubscriber 기본 구조

```js
import { createSubscriber } from 'svelte/reactivity'
import { on } from 'svelte/events'

const sub = createSubscriber((update) => {
  // ① start: 첫 구독자가 생길 때 1회 실행
  const off = on(window, 'scroll', update)  // 스크롤 시 update() 호출

  return () => {
    // ② cleanup: 모든 구독자가 사라질 때 실행
    off()
  }
})
```

### 동작 흐름

```
추적 컨텍스트(템플릿/effect)에서 sub() 호출
  │
  ├─ 첫 구독자 → start 함수 실행 (이벤트 리스너 등록)
  │
  ├─ 이벤트 발생 → update() 호출 → 추적 컨텍스트 재실행
  │                                  → 그 안의 브라우저 API 재평가
  │
  └─ 마지막 구독자 제거 → cleanup 실행 (이벤트 리스너 해제)
```

### 사용 예시

```svelte
<script>
  import { createSubscriber } from 'svelte/reactivity'
  import { on } from 'svelte/events'
  import { browser } from '$app/environment'

  const sub = createSubscriber((update) => {
    const off = on(window, 'scroll', update)
    return () => off()
  })
</script>

<!-- sub()을 호출해야 이 템플릿이 스크롤 시 재실행됨 -->
<h1>{sub(), window.scrollY}</h1>
```

`sub()`이 추적 컨텍스트 안에서 호출되면 → `update()` 호출 시마다 해당 컨텍스트가 재실행 → `window.scrollY`가 재평가되어 최신 값 표시.

### 핵심: 자동 구독자 수 관리

```
구독자 A 등장 → start 실행 (1회)
구독자 B 등장 → start 재실행 안 함 (이미 실행됨)
구독자 B 제거 → cleanup 안 함 (A가 아직 사용 중)
구독자 A 제거 → cleanup 실행 (모든 구독자 사라짐)
구독자 C 등장 → start 다시 실행
```

여러 컴포넌트에서 `sub()`을 호출해도 이벤트 리스너는 **1개만** 등록된다. 모든 구독자가 사라져야 정리된다. → 중복 구독 방지 + 메모리 누수 방지.

### React 비유

React에는 직접 대응이 없다. 굳이 비유하면 `useSyncExternalStore`와 유사:

```
useSyncExternalStore(subscribe, getSnapshot)
  - subscribe: 외부 스토어 변경 시 콜백 호출
  - getSnapshot: 현재 값 반환

createSubscriber(start)
  - start: 구독 시작 + update 콜백 제공
  - 값 자체는 저장하지 않음 → 브라우저 API에서 직접 읽음
```

차이점: `useSyncExternalStore`는 값을 스냅샷으로 관리하지만, `createSubscriber`는 **값을 저장하지 않고** 추적 컨텍스트의 재실행만 트리거한다. 실제 값은 `window.scrollY` 같은 원본에서 직접 읽는다.

### 내부 구현 원리 (소스 코드 분석)

`createSubscriber`의 핵심 트릭은 **숨겨진 `version` 상태**다:

```js
// createSubscriber 내부 (단순화)
function createSubscriber(start) {
  let subscribers = 0
  let version = $state(0)   // ← 핵심: 숨겨진 반응형 상태
  let stop = null

  return function sub() {
    // ① 추적 컨텍스트인지 확인
    if ($effect.tracking()) {
      version   // ← 읽기만 해도 이 컨텍스트의 의존성으로 등록됨

      // ② 첫 구독자일 때만 start 실행
      if (subscribers === 0) {
        stop = start(() => {
          version++   // ← update() = version 증가 → 구독자 재실행
        })
      }
      subscribers++

      // ③ 구독 해제 시 (effect 파괴 시)
      // subscribers-- → 0이 되면 stop() 호출
    }
  }
}
```

```
왜 이게 작동하는가?

1. sub()을 템플릿에서 호출
   → $effect.tracking() === true
   → version을 읽음 → 이 템플릿이 version에 의존하게 됨

2. 스크롤 발생 → update() 호출 → version++
   → version이 변경됨 → 의존하는 템플릿 재실행
   → 템플릿 안의 window.scrollY도 재평가 → 최신 값 표시

3. $effect.tracking() === false인 곳에서 호출하면?
   → 아무것도 안 함. 구독도 안 하고 이벤트 리스너도 안 붙임.
```

**`$effect.tracking()`의 실전 용도가 바로 이것** — 추적 컨텍스트일 때만 구독을 설정하고, 아닐 때는 불필요한 리스너를 피한다.

### 재사용 가능한 클래스로 캡슐화

`createSubscriber` + 브라우저 API를 클래스로 감싸면 어디서든 재사용 가능:

```ts
// $lib/utils/scroll-y.svelte.ts  (.svelte.ts 확장자 필수!)
import { createSubscriber } from 'svelte/reactivity'
import { on } from 'svelte/events'

class ScrollY {
  #subscribe

  constructor() {
    this.#subscribe = createSubscriber((update) => {
      const off = on(window, 'scroll', update)
      return () => off()
    })
  }

  get current() {
    this.#subscribe()                    // ← 추적 컨텍스트면 구독 등록
    return window.scrollY ?? 0           // ← 실제 값은 브라우저 API에서 직접
  }
}

export default new ScrollY()  // 싱글턴 인스턴스 export
```

```svelte
<!-- 사용하는 컴포넌트 -->
<script>
  import scrollY from '$lib/utils/scroll-y.svelte'
</script>

<h1>{scrollY.current}</h1>  <!-- 스크롤 시 자동 갱신 -->
```

```
패턴 정리:

class 내부:
  #subscribe = createSubscriber(...)   ← 구독 로직 캡슐화

  get current() {
    this.#subscribe()    ← getter 안에서 구독 함수 호출 (추적 컨텍스트 감지)
    return 실제_값       ← 브라우저 API에서 직접 읽기
  }

export default new Class()  ← 싱글턴으로 export → 여러 컴포넌트에서 공유
```

추적 컨텍스트 밖에서 `scrollY.current`를 읽으면? → `#subscribe()` 내부에서 `$effect.tracking() === false` → 구독 안 함, 값만 반환.

### svelte/reactivity/window — 이미 만들어진 것들

위에서 직접 만든 ScrollY 클래스를 Svelte가 이미 제공한다:

```ts
import { scrollY, scrollX, innerWidth, innerHeight } from 'svelte/reactivity/window'
```

```svelte
<h1>{scrollY.current}</h1>  <!-- 직접 만든 것과 동일하게 동작 -->
```

내부 구현은 `ReactiveValue` 클래스를 사용하며, 우리가 만든 패턴과 동일:

```
ReactiveValue 클래스 (Svelte 소스):
  - constructor(fn, onsubscribe)
    - fn: 값을 반환하는 함수 (= () => window.scrollY)
    - onsubscribe: 구독 시 실행할 함수 (= createSubscriber에 전달하는 start)
  - get current():
    - this.#subscribe() 호출
    - fn()으로 값 반환

scrollY = new ReactiveValue(
  () => window.scrollY,                    // 값 반환
  (update) => on(window, 'scroll', update) // 스크롤 시 update
)
```

> **실무에서는 `svelte/reactivity/window`를 사용하면 된다.** `createSubscriber`를 직접 쓸 일은 커스텀 외부 이벤트(WebSocket, IntersectionObserver 등)를 연결할 때 정도.

---

## 전체 요약

| 클래스 | 출처 | 용도 |
|--------|------|------|
| `SvelteDate` | `svelte/reactivity` | 날짜/시간 실시간 갱신 |
| `SvelteMap` | `svelte/reactivity` | 리액티브 Map |
| `SvelteSet` | `svelte/reactivity` | 리액티브 Set |
| `SvelteURL` | `svelte/reactivity` | URL 조작 반응형 |
| `SvelteURLSearchParams` | `svelte/reactivity` | URL 파라미터 반응형 |
| `MediaQuery` | `svelte/reactivity` | 미디어 쿼리 결과 반응형 |
| `createSubscriber` | `svelte/reactivity` | 외부 이벤트 → 반응성 연결 |
| `Tween` | `svelte/motion` | 이징 애니메이션 |
| `Spring` | `svelte/motion` | 스프링 물리 애니메이션 |

**핵심: 일반 JS 내장 객체(Date, Map, Set, URL)의 메서드 호출은 `$state`가 감지할 수 없다.
리액티브 버전을 쓰면 메서드 호출만으로 UI가 갱신된다.**

> **Svelte 4 store와의 비교**: Svelte 4에서는 `writable`, `derived` 같은 store로 반응성을 관리했다. Svelte 5에서는 `$state` 필드를 가진 클래스(내장 리액티브 클래스 포함)가 store를 대체한다. 클래스 방식이 타입 안전하고, 일반 JS 문법으로 작동하며, 컴파일러가 getter/setter를 자동 생성하여 실수를 줄인다.
