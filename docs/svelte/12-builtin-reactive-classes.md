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
Date 객체의 setSeconds()는 내부 값을 변경(mutate)하지만,
$state는 "재할당"을 감지하는 것이지 내부 변경을 감지하지 않는다.

let date = $state(new Date())
date.setSeconds(...)    // ❌ 내부 변경 → $state가 감지 못함
date = new Date()       // ✅ 재할당 → $state가 감지
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

let date = new SvelteDate()  // 끝! 메서드 호출 시 자동 반응
```

---

## SvelteDate

`Date`를 확장한 리액티브 버전. `setSeconds()` 등 메서드 호출 시 UI 자동 갱신.

```ts
import { SvelteDate } from 'svelte/reactivity'

let date = new SvelteDate()

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

let map = new SvelteMap()
map.set('key', 'value')  // → 반응성 동작

let set = new SvelteSet()
set.add('item')          // → 반응성 동작
```

일반 `Map`/`Set`은 `$state`로 감싸도 `.set()`, `.add()` 같은 메서드 호출을 감지하지 못한다.
(`$state`의 Proxy는 일반 객체/배열의 프로퍼티 변경만 추적)

---

## SvelteURL, SvelteURLSearchParams

URL 조작 시 반응성이 필요할 때 사용.

```ts
import { SvelteURL } from 'svelte/reactivity'

let url = new SvelteURL('https://example.com/path')

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

## 전체 요약

| 클래스 | 출처 | 용도 |
|--------|------|------|
| `SvelteDate` | `svelte/reactivity` | 날짜/시간 실시간 갱신 |
| `SvelteMap` | `svelte/reactivity` | 리액티브 Map |
| `SvelteSet` | `svelte/reactivity` | 리액티브 Set |
| `SvelteURL` | `svelte/reactivity` | URL 조작 반응형 |
| `SvelteURLSearchParams` | `svelte/reactivity` | URL 파라미터 반응형 |
| `MediaQuery` | `svelte/reactivity` | 미디어 쿼리 결과 반응형 |
| `Tween` | `svelte/motion` | 이징 애니메이션 |
| `Spring` | `svelte/motion` | 스프링 물리 애니메이션 |

**핵심: 일반 JS 내장 객체(Date, Map, Set, URL)의 메서드 호출은 `$state`가 감지할 수 없다.
리액티브 버전을 쓰면 메서드 호출만으로 UI가 갱신된다.**
