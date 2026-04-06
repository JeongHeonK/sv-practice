# Actions & Attachments — DOM에 커스텀 동작 부착

DOM 요소에 커스텀 동작을 부착하는 메커니즘. 네이티브에 없는 이벤트 추가, 서드파티 라이브러리 초기화 등에 활용한다.

> **Svelte 5.29+ 권장**: 새 코드에서는 `{@attach}`(Attachment)를 우선 사용한다. 기본 반응형이고 컴포넌트에도 적용 가능하다. Action(`use:`)은 레거시 호환이나 Attachment로 대체할 수 없는 경우에 사용한다.

## 핵심 개념

```
Action = DOM 요소가 마운트될 때 실행되는 함수
- 첫 번째 인수: 적용된 DOM 노드
- 두 번째 인수: 전달된 옵션
- $effect 반환 함수로 cleanup(파괴 시 정리) 처리
```

## 기본 사용법: longpress 예제

### 1. Action 함수 정의

```svelte
<!-- src/lib/actions/longpress.svelte.ts -->
<script lang="ts" module>
  // .svelte.ts 확장자 → $effect 등 rune 사용 가능

  export function longpress(node: HTMLButtonElement, options: { duration: number }) {
    $effect(() => {
      const duration = options.duration ?? 1000
      console.log('마운트', duration)

      return () => {
        console.log('파괴됨')
      }
    })
  }
</script>
```

> **파일 확장자**: `$effect`를 포함하므로 `.svelte.ts` 확장자 필요

### 2. Action 적용 (`use:` 디렉티브)

```svelte
<script lang="ts">
  import { longpress } from '$lib/actions/longpress.svelte'
</script>

<!-- use:함수명={옵션} -->
<button use:longpress={{ duration: 3000 }}>
  길게 누르기
</button>
```

```
use:longpress              → 옵션 없이 사용
use:longpress={{ duration: 3000 }}  → 옵션 객체 전달
```

## TypeScript 타이핑

`Action` 제네릭으로 3가지를 지정한다:

```ts
import type { Action } from 'svelte/action'

// Action<노드타입, 옵션타입, 커스텀이벤트맵>
export const longpress: Action<
  HTMLButtonElement,                          // 1. 허용할 DOM 요소 타입
  { duration?: number },                      // 2. 옵션 타입
  { onlongpress: (e: CustomEvent) => void }   // 3. 커스텀 이벤트 맵
> = (node, options) => {
  // ...
}
```

### 타입이 제공하는 안전장치

```svelte
<!-- ✅ 정상 -->
<button use:longpress={{ duration: 3000 }} onlongpress={() => console.log('길게 눌림')}>

<!-- ❌ 타입 에러: div에는 사용 불가 (HTMLButtonElement로 제한) -->
<div use:longpress={{ duration: 3000 }}>

<!-- ❌ 타입 에러: 옵션에 없는 프로퍼티 -->
<button use:longpress={{ durationn: 3000 }}>

<!-- ✅ 커스텀 이벤트 자동완성 제공 -->
<button use:longpress onlongpress={...}>
```

## Action 라이프사이클

```
컴포넌트 마운트 → action 함수 실행 → $effect 실행 (마운트)
                                         ↓
옵션(반응형 값) 변경 → $effect 재실행     ↓
                                         ↓
컴포넌트 파괴 → $effect return 함수 실행 (cleanup)
```

### 주의: action 함수 자체는 1회만 실행

```
longpress(node, options) ← 마운트 시 1회만 호출
  └─ $effect(() => { ... }) ← 의존성 변경 시 재실행
```

옵션을 변경해도 `longpress` 함수가 다시 호출되지 않는다. **내부 `$effect`가 반응형 값을 추적해야** 옵션 변경에 반응할 수 있다.

## 반응형 옵션 전달 패턴

### 문제: 일반 객체 전달 시 반응성 상실

```svelte
<script>
  let duration = $state(3000)
</script>

<!-- ❌ 객체 생성 시점의 값(3000)만 전달됨, 이후 변경 감지 불가 -->
<button use:longpress={{ duration }}>
```

`{ duration }`은 객체 생성 시점에 `duration`의 **값**을 복사하므로, 이후 `duration`이 바뀌어도 action 내부 `$effect`가 추적하지 못한다.

### 해결 1: 함수로 감싸기 (권장)

```svelte
<!-- 객체를 반환하는 함수를 전달 -->
<button use:longpress={() => ({ duration })}>
```

```ts
// action 내부
export function longpress(
  node: HTMLButtonElement,
  getOptions: () => { duration: number }  // 함수 타입으로 변경
) {
  $effect(() => {
    const options = getOptions()  // ← $effect 내부에서 호출해야 추적됨!
    const duration = options.duration ?? 1000
    console.log('duration:', duration)

    return () => { /* cleanup */ }
  })
}
```

```
핵심: $effect 내부에서 getOptions() 호출
→ 함수 실행 시 duration($state)을 읽음
→ $effect가 duration을 의존성으로 추적
→ duration 변경 시 $effect 재실행
```

> **주의**: `getOptions()`를 `$effect` 외부에서 호출하면 의존성 추적이 되지 않는다.

### 해결 2: $state 프록시 객체 전달

```svelte
<script>
  let options = $state({ duration: 3000 })
</script>

<button use:longpress={options}>
```

```ts
export function longpress(node: HTMLButtonElement, options: { duration: number }) {
  $effect(() => {
    // options는 $state 프록시 → .duration 접근 시 추적됨
    const duration = options.duration ?? 1000

    return () => { /* cleanup */ }
  })
}
```

> 프록시 객체 자체를 전달하면 프로퍼티 접근이 추적되지만, 타입이 복잡해질 수 있다.

## 실전 구현: longpress 커스텀 이벤트 디스패치

### 커스텀 이벤트 디스패치 방법

```ts
// JS 네이티브 API로 커스텀 이벤트 발송
node.dispatchEvent(new CustomEvent('longpress'))
```

### 전체 구현

```ts
// src/lib/actions/longpress.svelte.ts
import type { Action } from 'svelte/action'

export const longpress: Action<
  HTMLButtonElement,
  () => { duration?: number },
  { onlongpress: (e: CustomEvent) => void }
> = (node, getOptions) => {
  $effect(() => {
    const options = getOptions()  // $effect 내부에서 호출 → 반응형 추적
    const duration = options.duration !== undefined
      ? options.duration
      : 1000

    let timer: number

    function handleMouseDown() {
      timer = setTimeout(() => {
        node.dispatchEvent(new CustomEvent('longpress'))
      }, duration)
    }

    function handleMouseUp() {
      clearTimeout(timer)
    }

    node.addEventListener('mousedown', handleMouseDown)
    node.addEventListener('mouseup', handleMouseUp)

    return () => {
      // cleanup: 타이머 정리 + 이벤트 리스너 제거 (메모리 누수 방지)
      clearTimeout(timer)
      node.removeEventListener('mousedown', handleMouseDown)
      node.removeEventListener('mouseup', handleMouseUp)
    }
  })
}
```

### 사용 예시

```svelte
<script lang="ts">
  import { longpress } from '$lib/actions/longpress.svelte'

  let duration = $state(3000)
  let show = $state(true)
</script>

<label>
  <input type="checkbox" bind:checked={show} /> 버튼 표시
</label>
<label>
  duration: {duration}ms
  <input type="range" min={0} max={5000} bind:value={duration} />
</label>

{#if show}
  <button
    use:longpress={() => ({ duration })}
    onlongpress={() => console.log('길게 눌림!')}
  >
    {duration}ms 동안 누르기
  </button>
{/if}
```

### 동작 흐름

```
mousedown → setTimeout(duration) 시작
  ├─ duration 경과 전 mouseup → clearTimeout → 이벤트 발생 안 함
  └─ duration 경과 → dispatchEvent('longpress') → onlongpress 핸들러 실행

duration 변경 → $effect 재실행 → 기존 리스너 cleanup → 새 duration으로 재등록
버튼 파괴 → $effect cleanup → 타이머 정리 + 리스너 제거
```

### 주의: `0`은 falsy — `undefined` 체크 필요

```ts
// ❌ duration이 0이면 falsy → 의도치 않게 1000으로 폴백
const duration = options.duration ?? 1000       // 0 ?? 1000 = 0 (OK, nullish)
const duration = options.duration || 1000       // 0 || 1000 = 1000 (Bug!)
const duration = options.duration ? options.duration : 1000  // 0은 falsy → 1000 (Bug!)

// ✅ 명시적 undefined 체크
const duration = options.duration !== undefined ? options.duration : 1000
```

```
?? (nullish coalescing): null, undefined만 폴백 → 0은 유효값으로 통과
|| (logical or): 모든 falsy(0, '', false, null, undefined) 폴백
삼항 조건: 조건부에서 0은 false → 폴백
```

## 실전 예제 2: 서드파티 라이브러리 래핑 (Tippy.js)

Action으로 서드파티 라이브러리 초기화를 캡슐화하면 `bind:this` + `$effect` 보일러플레이트를 제거할 수 있다.

### Before: Action 없이 직접 사용

```svelte
<script lang="ts">
  import tippy from 'tippy.js'
  import 'tippy.js/dist/tippy.css'

  let button: HTMLButtonElement

  $effect(() => {
    const tooltip = tippy(button, { content: '툴팁 텍스트' })
    return () => tooltip.destroy()
  })
</script>

<button bind:this={button}>Hover me</button>
```

문제점: 요소마다 `bind:this` + `$effect` + destroy를 반복해야 한다.

### After: Action으로 캡슐화

```ts
// src/lib/actions/tooltip.svelte.ts
import tippy, { type Props } from 'tippy.js'
import 'tippy.js/dist/tippy.css'
import 'tippy.js/animations/scale.css'    // 선택: 애니메이션 CSS
import type { Action } from 'svelte/action'

export const tooltip: Action<
  HTMLElement,                   // 모든 HTML 요소에 적용 가능
  () => Partial<Props>           // 옵션을 반환하는 함수 (반응형)
> = (node, getOptions) => {
  // 인스턴스 1회 생성 ($effect 외부)
  const tip = tippy(node, getOptions())

  // 옵션 변경 감지용 별도 $effect
  $effect(() => {
    const options = getOptions()  // 반응형 추적
    tip.setProps(options)         // 기존 인스턴스 업데이트 (파괴/재생성 X)
  })

  // DOM 제거 시 인스턴스 정리
  $effect(() => {
    return () => tip.destroy()
  })
}
```

```svelte
<script lang="ts">
  import { tooltip } from '$lib/actions/tooltip.svelte'

  let content = $state('안녕하세요!')
</script>

<input bind:value={content} />

<!-- 한 줄로 툴팁 적용 -->
<button use:tooltip={() => ({ content, animation: 'scale' })}>
  Hover me
</button>
```

### 최적화: 인스턴스 재생성 vs 업데이트

```
방법 1: $effect 안에서 생성 + destroy
─────────────────────────────────────
$effect(() => {
  const options = getOptions()
  const tip = tippy(node, options)
  return () => tip.destroy()       // 옵션 변경마다 파괴 → 재생성
})
→ 단순하지만, 변경마다 인스턴스 파괴/재생성 비용 발생

방법 2: $effect 밖에서 생성 + setProps로 업데이트 (권장)
─────────────────────────────────────
const tip = tippy(node, getOptions())   // 1회 생성

$effect(() => {
  tip.setProps(getOptions())            // 옵션 변경 시 업데이트만
})

$effect(() => {
  return () => tip.destroy()            // DOM 제거 시에만 파괴
})
→ 라이브러리가 setProps 같은 업데이트 API를 제공하면 이 방식이 효율적
```

```
핵심 판단 기준:
라이브러리에 update/setProps API가 있는가?
  ├─ Yes → 인스턴스 1회 생성 + $effect로 업데이트
  └─ No  → $effect 안에서 생성/파괴 반복
```

---

## Attachments: 권장 방식 (Svelte 5.29+)

Action의 개선된 버전. **기본 반응형** + **컴포넌트에도 적용 가능**.

> **새 코드에서는 `{@attach}`를 기본으로 사용한다.** Action은 deprecated되지 않았지만, Attachment가 반응성·적용 범위·코드 간결성 모든 면에서 우월하다. Action은 `{@attach}`로 대체할 수 없는 레거시 코드나 특수한 경우에만 사용한다.

### Action vs Attachment

```
                        Action (use:)           Attachment ({@attach})
─────────────────────── ─────────────────────── ───────────────────────
적용 대상               HTML 요소만              HTML 요소 + Svelte 컴포넌트
반응성                  기본 비반응형             기본 반응형 ($effect로 실행)
                        (함수 래핑 필요)
옵션 전달               use:fn={options}         {@attach fn(options)}
인라인 정의             불가                     가능
컴포넌트 전달           불가                     props spread로 전달 가능
```

### 기본 문법

```svelte
<script>
  import type { Attachment } from 'svelte'

  // Attachment = 요소를 받아 cleanup 함수를 반환하는 함수
  const myAttachment: Attachment = (element) => {
    console.log('마운트', element)

    return () => {
      console.log('정리')  // 마운트 해제 또는 재실행 시
    }
  }
</script>

<!-- {@attach 함수명} -->
<button {@attach myAttachment}>Click</button>
```

```
핵심: Attachment 함수는 $effect 안에서 실행됨
→ 내부에서 참조하는 $state가 변경되면 자동으로 재실행
→ Action처럼 수동으로 $effect를 만들 필요 없음
```

### Tippy.js를 Attachment로 래핑

#### 옵션 없는 기본형

```ts
// src/lib/actions/tooltip.svelte.ts
import tippy from 'tippy.js'
import 'tippy.js/dist/tippy.css'
import type { Attachment } from 'svelte'

export const tp2: Attachment = (element) => {
  const tooltip = tippy(element, { content: '하드코딩 텍스트' })

  return () => tooltip.destroy()
}
```

```svelte
<button {@attach tp2}>Hover me</button>
```

#### 옵션을 받는 형태: 함수가 Attachment를 반환

```ts
import tippy, { type Props } from 'tippy.js'
import 'tippy.js/dist/tippy.css'
import 'tippy.js/animations/scale.css'
import type { Attachment } from 'svelte'

// tp2는 Attachment가 아니라, Attachment를 "반환하는 함수"
export const tp2 = (options: Partial<Props>): Attachment => {
  return (element) => {
    const tooltip = tippy(element, options)
    return () => tooltip.destroy()
  }
}
```

```svelte
<script lang="ts">
  import { tp2 } from '$lib/actions/tooltip.svelte'

  let content = $state('안녕하세요!')
</script>

<input bind:value={content} />

<!-- 기본 반응형: content 변경 시 자동 재실행 -->
<button {@attach tp2({ content, animation: 'scale' })}>
  Hover me
</button>
```

```
Action 방식:   use:tooltip={() => ({ content })}   ← 함수 래핑 필요
Attachment:    {@attach tp2({ content })}           ← 직접 전달, 자동 반응형
```

### 컴포넌트에 Attachment 적용

Action으로는 불가능했던 핵심 기능: **Svelte 컴포넌트에 Attachment를 전달**할 수 있다.

```svelte
<!-- Button.svelte -->
<script lang="ts">
  import type { HTMLButtonAttributes } from 'svelte/elements'

  let { children, ...rest }: HTMLButtonAttributes = $props()
</script>

<!-- ...rest로 spread → Attachment도 함께 전달됨 -->
<button {...rest}>
  {@render children?.()}
</button>
```

```svelte
<!-- 사용처 -->
<script>
  import Button from './Button.svelte'
  import { tp2 } from '$lib/actions/tooltip.svelte'
</script>

<!-- ✅ 컴포넌트에 Attachment 적용 가능 -->
<Button {@attach tp2({ content: '컴포넌트 툴팁' })}>
  버튼 3
</Button>
```

```
동작 원리:
1. Button 컴포넌트가 {@attach tp2(...)}를 props로 받음
2. {...rest}로 내부 <button> 요소에 spread
3. Attachment가 최종 HTML 요소에 적용됨

→ Action은 HTML 요소에만 직접 사용 가능했으므로
  컴포넌트 → 내부 요소로 전달하는 패턴이 불가능했음
```

### 인라인 Attachment

별도 함수 정의 없이 직접 인라인으로 작성 가능 (Action에서는 불가).

```svelte
<div {@attach (element) => {
  console.log('마운트:', element)
  return () => console.log('정리')
}}>
  인라인 attachment
</div>
```

---

## React 비교

```
React                              Svelte Action          Svelte Attachment
─────────────────────────         ──────────────────────  ──────────────────────
useRef + useEffect                 use:action              {@attach fn}
  ref.current로 DOM 접근              node 인수로 직접 접근     element 인수로 직접 접근
  useEffect cleanup으로 정리          $effect return으로 정리   return 함수로 정리
  커스텀 훅으로 추상화                 action 함수로 재사용      attachment 함수로 재사용
  forwardRef로 전달                  컴포넌트 전달 불가         props spread로 전달
```

## 선택 가이드

```
"DOM에 커스텀 동작을 부착해야 한다"
  │
  ├─ Svelte 5.29+ 사용 중?
  │   └─ {@attach} 사용 (권장) ✅
  │
  ├─ 5.29 미만 or 레거시 코드 유지보수?
  │   └─ use: Action 사용
  │
  └─ 기존 Action을 마이그레이션?
      └─ {@attach}로 전환 권장 (반응성·범위 개선)
```

## 정리

| 항목 | Action (`use:`) | Attachment (`{@attach}`) |
|------|-----------------|--------------------------|
| **권장 여부** | 레거시 호환 | **신규 코드 권장** ✅ |
| 적용 대상 | HTML 요소만 | HTML 요소 + 컴포넌트 |
| 반응성 | 수동 (함수 래핑) | 자동 ($effect로 실행) |
| 옵션 전달 | `use:fn={opts}` | `{@attach fn(opts)}` |
| 인라인 | 불가 | 가능 |
| 타이핑 | `Action<Node, Opts, Events>` | `Attachment` |
| cleanup | `$effect` return | return 함수 |
| 최소 버전 | - | Svelte 5.29+ |
