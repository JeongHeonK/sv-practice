# 명령형 라이브러리를 선언적 컴포넌트로 래핑 — Context 활용

Context API(14번 참조)의 핵심 활용 사례: Konva 같은 명령형 JS 라이브러리를 Svelte 컴포넌트 계층으로 변환한다.

## 목표: 명령형 → 선언적

```
명령형 (Konva 직접 사용):
  const stage = new Konva.Stage({ container, width, height })
  const layer = new Konva.Layer()
  const rect = new Konva.Rect({ x: 10, y: 10, width: 100, fill: 'red' })
  layer.add(rect)
  stage.add(layer)

선언적 (Svelte 컴포넌트):
  <Stage width={500} height={500}>
    <Layer>
      <Rect x={10} y={10} width={100} fill="red" />
    </Layer>
  </Stage>

→ 자식 컴포넌트가 부모 인스턴스를 알아야 함
→ Layer는 어떤 Stage에 추가해야 하는지, Rect는 어떤 Layer에 추가해야 하는지
→ Context로 해결
```

## 명령형 기본 코드: bind:this + $effect

먼저 명령형 방식으로 Konva를 초기화하는 기본 코드.

> **주의**: 아래 초기 예제는 `$effect` 안에서 Stage를 생성한다. 이는 학습용이며, prop 변경 시 전체 재생성되는 문제가 있다. 이후 구현 5에서 `onMount`로 개선한다.

```svelte
<script lang="ts">
  import Konva from 'konva'

  let container: HTMLDivElement

  $effect(() => {
    // $effect 안에서 실행 → DOM 마운트 보장
    const stage = new Konva.Stage({
      container,      // bind:this로 참조한 DOM 요소
      width: 500,
      height: 500,
    })

    const layer = new Konva.Layer()

    const rect = new Konva.Rect({
      x: 10, y: 10,
      width: 100, height: 50,
      fill: 'cornflowerblue',
      stroke: 'black',
      strokeWidth: 2,
    })

    layer.add(rect)     // 레이어에 도형 추가
    stage.add(layer)    // 스테이지에 레이어 추가
  })
</script>

<div bind:this={container}></div>
```

```
bind:this — DOM 요소 참조:
  let container: HTMLDivElement     ← 변수 선언
  <div bind:this={container}>      ← DOM 마운트 후 참조 할당

  $effect 안에서 사용해야 하는 이유:
    스크립트 최상위에서는 container가 아직 undefined
    $effect는 DOM 업데이트 후 실행 → container가 확실히 존재
```

## SvelteKit SSR 주의

```
문제:
  SvelteKit은 서버에서도 모듈을 import한다.
  Konva는 내부적으로 Canvas API를 사용 → 서버에 Canvas가 없으면 에러.

해결 방법:
  1. canvas 패키지 설치: pnpm add canvas
     → 서버에서 빈 캔버스를 그려 에러 방지
  2. 또는 dynamic import로 클라이언트에서만 로드:
     const Konva = (await import('konva')).default
```

## Context가 필요한 이유

```
명령형 코드의 계층 구조:

  Stage (컨테이너)
    └─ Layer
         └─ Rect, Circle, ...

각 계층이 부모 인스턴스를 알아야 한다:
  Layer → "나는 어떤 Stage에 추가되어야 하나?"
  Rect  → "나는 어떤 Layer에 추가되어야 하나?"

prop drilling:
  <Stage>에서 stage 인스턴스를 <Layer>에 prop으로 전달?
  <Layer>에서 layer 인스턴스를 <Rect>에 prop으로 전달?
  → 중간 컴포넌트가 많아지면 번거롭고 유연성 없음

Context:
  <Stage> → setContext('stage', stageInstance)
  <Layer> → getContext('stage') → stage.add(layer)
            setContext('layer', layerInstance)
  <Rect>  → getContext('layer') → layer.add(rect)
  → 계층 깊이와 무관하게 부모 인스턴스 접근 가능
```

> 대부분의 인기 라이브러리(Leaflet, Konva, D3 등)에는 이미 Svelte/React 래퍼가 존재한다. 하지만 래퍼가 없는 라이브러리를 직접 래핑해야 할 때 이 패턴을 사용한다.

## 구현 1: Stage 컴포넌트

먼저 컴포넌트 구조를 잡는다. barrel export용 인덱스 파일과 Stage 컴포넌트.

```
src/lib/components/konva/
  index.ts          ← barrel export
  Stage.svelte      ← Stage 래퍼
  (Layer.svelte)    ← 다음 단계
  (Rect.svelte)     ← 다음 단계
```

```ts
// index.ts — barrel export
export { default as Stage } from './Stage.svelte'
export { default as Layer } from './Layer.svelte'
// export { default as Rect } from './Rect.svelte'
```

### Context 파일: konva-context.ts

Stage 인스턴스를 자식에게 전달하기 위한 Context 캡슐화.

```ts
// konva-context.ts
import { setContext, getContext } from 'svelte'
import Konva from 'konva'

// ── Stage Context ──
const stageKey = Symbol('konva-stage')

export function setStageContext(getStage: () => Konva.Stage | undefined) {
  setContext(stageKey, getStage)   // getter 함수 전달 (비동기 초기화 대응)
}

export function getStageContext(): Konva.Stage {
  const getStage = getContext<() => Konva.Stage | undefined>(stageKey)
  if (!getStage) throw new Error('Layer는 Stage의 자식이어야 합니다')

  const stage = getStage()
  if (!stage) throw new Error('Stage가 아직 초기화되지 않았습니다')

  return stage
}

// ── Layer Context ──
const layerKey = Symbol('konva-layer')

export function setLayerContext(layer: Konva.Layer) {
  setContext(layerKey, layer)       // 인스턴스 직접 전달 (동기 생성이므로 getter 불필요)
}

export function getLayerContext(): Konva.Layer {
  const layer = getContext<Konva.Layer>(layerKey)
  if (!layer) throw new Error('도형은 Layer의 자식이어야 합니다')

  return layer
}
```

```
Stage vs Layer — Context에 전달하는 방식이 다른 이유:

  Stage Context: getter 함수 전달   → setStageContext(() => stage)
  Layer Context: 인스턴스 직접 전달  → setLayerContext(node)

  Stage는 $effect 안에서 생성 (DOM 필요) → setContext 시점에 undefined
    → 함수로 감싸서 나중에 호출 시 참조
  Layer는 스크립트 최상위에서 즉시 생성 → setContext 시점에 이미 존재
    → 인스턴스를 직접 전달해도 됨
```

```
왜 Stage 인스턴스 대신 "함수"를 전달하는가?

  setContext는 스크립트 최상위에서만 호출 가능.
  하지만 Stage 인스턴스는 $effect 안에서 생성됨 (DOM 마운트 후).

  시간 순서:
    1. <script> 실행 → stage = undefined
    2. setContext(key, stage) → undefined가 Context에 저장됨
    3. $effect → stage = new Konva.Stage(...)  ← 이미 늦음!

  해결: 함수를 전달하면 호출 시점에 stage를 참조
    setContext(key, () => stage)  → 함수 자체는 즉시 저장 가능
    자식에서 getStage() 호출 시 → 그 시점의 stage 반환
```

### Stage 컴포넌트 (Context 연결 버전)

```svelte
<!-- Stage.svelte -->
<script lang="ts">
  import Konva from 'konva'
  import type { Snippet } from 'svelte'
  import { setStageContext } from './konva-context'

  let { children, ...props }: { children?: Snippet } & Konva.StageConfig = $props()

  let container: HTMLDivElement
  let stage: Konva.Stage | undefined = $state()
  let ready = $state(false)

  // Context에 getter 함수 전달 — 스크립트 최상위에서 호출
  setStageContext(() => stage)

  $effect(() => {
    stage = new Konva.Stage({ container, ...props })
    ready = true   // 자식 렌더링 허용

    return () => stage?.destroy()
  })
</script>

<div bind:this={container}>
  {#if ready && children}
    {@render children()}
  {/if}
</div>
```

```
ready 플래그가 필요한 이유:

  $effect 실행 전: stage = undefined
    → children 렌더링 → Layer가 getStageContext() 호출
    → stage가 undefined → "Cannot read properties of undefined" 에러!

  해결: ready = false로 시작 → $effect에서 stage 생성 후 ready = true
    → ready가 true일 때만 children 렌더링
    → Layer가 실행될 때 stage가 확실히 존재

  시간 순서:
    1. <script> → stage = undefined, ready = false
    2. setStageContext(() => stage) → getter 함수 저장
    3. 템플릿 렌더링 → ready = false → children 렌더링 안 함
    4. $effect → stage = new Konva.Stage(...), ready = true
    5. 템플릿 재렌더링 → ready = true → children 렌더링
    6. Layer 실행 → getStageContext() → stage 반환 ✅
```

## 구현 2: Layer 컴포넌트

```svelte
<!-- Layer.svelte -->
<script lang="ts">
  import Konva from 'konva'
  import type { Snippet } from 'svelte'
  import { getStageContext, setLayerContext } from './konva-context'
  import { onDestroy } from 'svelte'

  let { children, ...props }: { children?: Snippet } & Konva.LayerConfig = $props()

  // Context에서 부모 Stage 가져오기
  const stage = getStageContext()

  // 레이어 생성 & Stage에 추가
  const node = new Konva.Layer(props)
  stage.add(node)

  // 자식(도형)이 이 레이어에 접근할 수 있도록 Context 설정
  setLayerContext(node)

  // 클린업: 컴포넌트 제거 시 레이어 파괴
  onDestroy(() => node.destroy())
</script>

{#if children}
  {@render children()}
{/if}
```

## 구현 3: Rect 컴포넌트

```svelte
<!-- Rect.svelte -->
<script lang="ts">
  import Konva from 'konva'
  import { getLayerContext } from './konva-context'
  import { onDestroy } from 'svelte'

  // 자식 없음 — children 불필요
  let props: Konva.RectConfig = $props()

  // Context에서 부모 Layer 가져오기
  const layer = getLayerContext()

  // 도형 생성 & Layer에 추가
  const node = new Konva.Rect(props)
  layer.add(node)

  onDestroy(() => node.destroy())
</script>
```

```ts
// index.ts — 최종 barrel export
export { default as Stage } from './Stage.svelte'
export { default as Layer } from './Layer.svelte'
export { default as Rect } from './Rect.svelte'
```

```svelte
<!-- +page.svelte — 최종 사용 -->
<script>
  import { Stage, Layer, Rect } from '$lib/components/konva'
</script>

<Stage width={500} height={500}>
  <Layer>
    <Rect
      x={20} y={40}
      width={200} height={100}
      fill="purple"
      stroke="black"
      strokeWidth={2}
      draggable
    />
  </Layer>
</Stage>
```

```
전체 Context 흐름:

  <Stage>
    setStageContext(() => stage)     ← getter 함수 저장
    $effect → stage 생성 → ready = true → children 렌더링
    │
    └─ <Layer>
         getStageContext()           ← stage 인스턴스 획득
         node = new Konva.Layer()
         stage.add(node)
         setLayerContext(node)       ← 레이어 인스턴스 저장
         │
         └─ <Rect>
              getLayerContext()      ← layer 인스턴스 획득
              node = new Konva.Rect(props)
              layer.add(node)       ← 도형 추가 완료!

  Rect는 자식이 없으므로:
    - children prop 없음
    - 템플릿 없음 (순수 로직 컴포넌트)
    - Context를 설정하지 않음 (소비만 함)
```

```
Layer vs Stage — 초기화 시점의 차이:

  Stage:
    DOM 요소(container)가 필요 → $effect 안에서 생성
    → ready 플래그로 자식 렌더링 지연

  Layer:
    DOM 요소 불필요 → 스크립트 최상위에서 바로 생성 가능
    → new Konva.Layer(props) 즉시 실행
    → stage.add(node) 즉시 실행

  클린업 방식도 다름:
    Stage: $effect return → effect 재실행/컴포넌트 파괴 시 호출
    Layer: onDestroy → 컴포넌트 파괴 시에만 호출
           ($effect 안에서 생성하지 않았으므로 onDestroy 사용)
```

## 구현 4: 이벤트 처리

Konva 노드는 `node.on('eventName', callback)`으로 이벤트를 등록한다. 이를 Svelte prop으로 자연스럽게 사용하려면 타입 정의 + 자동 등록 로직이 필요하다.

### 이벤트 타입 정의

```ts
// events.ts
import Konva from 'konva'

// Konva가 지원하는 이벤트 목록
export const events = [
  'mousedown', 'mouseup', 'mouseover', 'mouseout', 'mouseenter',
  'mouseleave', 'mousemove', 'click', 'dblclick',
  'touchstart', 'touchend', 'touchmove', 'tap', 'dbltap',
  'dragstart', 'dragmove', 'dragend',
  'wheel', 'contextmenu',
  'pointerdown', 'pointerup', 'pointermove', 'pointerclick',
] as const

// Svelte prop용 이벤트 훅 타입 — on 접두사 + 소문자 이벤트명
// 예: ondragend, onclick, onmouseover ...
export type KonvaEventHooks = {
  [K in (typeof events)[number] as `on${K}`]?: (e: Konva.KonvaEventObject<Event>) => void
}
```

```
Konva 이벤트 → Svelte prop 매핑:

  Konva:  node.on('dragend', (e) => { ... })
  Svelte: <Rect ondragend={(e) => { ... }} />

  이벤트명 앞에 'on' 접두사를 붙여 Svelte 이벤트 prop 규칙을 따른다.
  타입 정의에 KonvaEventHooks를 추가하면 자동완성이 제공된다.
```

### 이벤트 자동 등록 함수

모든 컴포넌트에서 이벤트 등록 로직을 반복하지 않기 위해 함수로 추출한다.

```ts
// events.ts (계속)
export function registerEvents(
  node: Konva.Node,
  props: KonvaEventHooks,
) {
  for (const eventName of events) {
    const callback = (props as Record<string, unknown>)[`on${eventName}`]
    if (callback && typeof callback === 'function') {
      node.on(eventName, (e: Konva.KonvaEventObject<Event>) => {
        callback(e)
      })
    }
  }
}
```

```
동작 원리:

  events 배열을 순회하며:
    1. props에서 'on' + 이벤트명에 해당하는 prop을 찾음
       예: eventName = 'dragend' → props['ondragend']
    2. 해당 prop이 함수이면 node.on()으로 이벤트 등록
    3. 해당 prop이 없으면 무시

  모든 컴포넌트(Stage, Layer, Rect)에서 동일하게 사용 가능
```

### 컴포넌트에 이벤트 적용

각 컴포넌트의 prop 타입에 `KonvaEventHooks`를 추가하고, 노드 생성 후 `registerEvents`를 호출한다.

```svelte
<!-- Rect.svelte (이벤트 지원 버전) -->
<script lang="ts">
  import Konva from 'konva'
  import { getLayerContext } from './konva-context'
  import { type KonvaEventHooks, registerEvents } from './events'
  import { onDestroy } from 'svelte'

  // KonvaEventHooks를 타입에 추가 → ondragend, onclick 등 자동완성
  let props: Konva.RectConfig & KonvaEventHooks = $props()

  const layer = getLayerContext()

  const node = new Konva.Rect(props)
  layer.add(node)
  registerEvents(node, props)   // 이벤트 자동 등록

  onDestroy(() => node.destroy())
</script>
```

```svelte
<!-- Layer.svelte (이벤트 지원 버전) -->
<script lang="ts">
  import Konva from 'konva'
  import type { Snippet } from 'svelte'
  import { getStageContext, setLayerContext } from './konva-context'
  import { type KonvaEventHooks, registerEvents } from './events'
  import { onDestroy } from 'svelte'

  let { children, ...props }: { children?: Snippet } & Konva.LayerConfig & KonvaEventHooks = $props()

  const stage = getStageContext()
  const node = new Konva.Layer(props)
  stage.add(node)
  setLayerContext(node)
  registerEvents(node, props)   // 이벤트 자동 등록

  onDestroy(() => node.destroy())
</script>

{#if children}
  {@render children()}
{/if}
```

```svelte
<!-- Stage.svelte (이벤트 지원 버전) — $effect 안에서 등록 -->
<script lang="ts">
  import Konva from 'konva'
  import type { Snippet } from 'svelte'
  import { setStageContext } from './konva-context'
  import { type KonvaEventHooks, registerEvents } from './events'

  let { children, ...props }:
    { children?: Snippet } & Konva.StageConfig & KonvaEventHooks = $props()

  let container: HTMLDivElement
  let stage: Konva.Stage | undefined = $state()
  let ready = $state(false)

  setStageContext(() => stage)

  $effect(() => {
    stage = new Konva.Stage({ container, ...props })
    registerEvents(stage, props)  // Stage는 $effect 안에서 등록
    ready = true

    return () => stage?.destroy()
  })
</script>

<div bind:this={container}>
  {#if ready && children}
    {@render children()}
  {/if}
</div>
```

### 사용 예시

```svelte
<Stage width={500} height={500} onclick={(e) => console.log('stage clicked', e)}>
  <Layer onclick={(e) => console.log('layer clicked', e)}>
    <Rect
      x={20} y={40}
      width={200} height={100}
      fill="purple"
      stroke="black"
      strokeWidth={2}
      draggable
      ondragend={(e) => console.log('drag ended', e)}
      ondblclick={(e) => console.log('double clicked', e)}
    />
  </Layer>
</Stage>
```

## 구현 5: Prop 변경 시 노드 업데이트 (반응성)

현재까지의 컴포넌트는 **초기 prop만 반영**한다. prop이 바뀌어도 Konva 노드는 업데이트되지 않는다.

### 문제: $effect에서 Stage를 생성하면 prop 변경 시 재생성된다

```
문제 1: Stage가 $effect 안에서 생성됨
  → props가 의존성으로 등록됨
  → prop 변경(width, height 등) → $effect 재실행
  → stage.destroy() → new Konva.Stage()
  → 기존 Layer/Rect가 모두 사라짐!

  예: 창 크기에 반응하는 width/height를 전달하면
    import { innerWidth, innerHeight } from 'svelte/reactivity/window'
    <Stage width={innerWidth.current} height={innerHeight.current / 2}>
    → 창 리사이즈 → $effect 재실행 → Stage 재생성 → 모든 도형 소실

문제 2: Rect/Layer의 prop이 변경되어도 반영 안 됨
  let x = $state(50)
  <Rect x={x} ... />
  x = 100  → Konva.Rect는 여전히 x=50 (초기값에 고정)
```

### 해결: onMount + $effect 분리

Stage 생성은 **한 번만** (`onMount`), prop 업데이트는 **각 prop별 $effect**로 분리한다.

```svelte
<!-- Stage.svelte (최종 버전) -->
<script lang="ts">
  import Konva from 'konva'
  import type { Snippet } from 'svelte'
  import { onMount } from 'svelte'
  import { setStageContext } from './konva-context'
  import { type KonvaEventHooks, registerEvents } from './events'

  let { children, ...props }:
    { children?: Snippet } & Konva.StageConfig & KonvaEventHooks = $props()

  let container: HTMLDivElement
  let stage: Konva.Stage | undefined = $state()
  let ready = $state(false)

  setStageContext(() => stage)

  // ① onMount — Stage 생성은 한 번만
  onMount(() => {
    stage = new Konva.Stage({ container, ...props })
    registerEvents(stage, props)
    ready = true

    return () => stage?.destroy()  // onMount의 return → 컴포넌트 파괴 시 실행
  })

  // ② 각 prop별 $effect — 변경 시 해당 속성만 업데이트
  const propKeys = Object.keys(props).filter(k => !k.startsWith('on'))
  for (const key of propKeys) {
    $effect(() => {
      stage?.setAttr(key, (props as Record<string, unknown>)[key])
    })
  }
</script>

<div bind:this={container}>
  {#if ready && children}
    {@render children()}
  {/if}
</div>
```

```
onMount vs $effect — Stage 생성에 왜 onMount인가:

  $effect:
    → 의존성 변경 시 재실행
    → props를 사용 → props가 의존성
    → prop 변경 → Stage 재생성 → Layer/Rect 소실!

  onMount:
    → 컴포넌트 마운트 시 딱 한 번만 실행
    → 의존성 추적 없음
    → prop 변경해도 Stage는 그대로 유지

  prop 업데이트는 별도 $effect로:
    for (const key of propKeys) {
      $effect(() => {
        stage?.setAttr(key, props[key])  // 해당 prop만 읽음 → 해당 prop만 의존성
      })
    }
    → width 변경 → width $effect만 재실행 → stage.setAttr('width', newValue)
    → height 변경 → height $effect만 재실행

  이벤트 prop(on...)은 필터링:
    Object.keys(props).filter(k => !k.startsWith('on'))
    → onclick, ondragend 등은 $effect 불필요 (이미 registerEvents로 등록됨)
```

### Layer, Rect에도 동일 패턴 적용

Layer와 Rect는 `onMount`가 아닌 스크립트 최상위에서 생성하므로, prop 업데이트 루프만 추가하면 된다.

```svelte
<!-- Rect.svelte (최종 버전) -->
<script lang="ts">
  import Konva from 'konva'
  import { getLayerContext } from './konva-context'
  import { type KonvaEventHooks, registerEvents } from './events'
  import { onDestroy } from 'svelte'

  let props: Konva.RectConfig & KonvaEventHooks = $props()

  const layer = getLayerContext()
  const node = new Konva.Rect(props)
  layer.add(node)
  registerEvents(node, props)

  // prop 변경 시 노드 속성 업데이트
  const propKeys = Object.keys(props).filter(k => !k.startsWith('on'))
  for (const key of propKeys) {
    $effect(() => {
      node.setAttr(key, (props as Record<string, unknown>)[key])
    })
  }

  onDestroy(() => node.destroy())
</script>
```

```svelte
<!-- Layer.svelte (최종 버전) -->
<script lang="ts">
  import Konva from 'konva'
  import type { Snippet } from 'svelte'
  import { getStageContext, setLayerContext } from './konva-context'
  import { type KonvaEventHooks, registerEvents } from './events'
  import { onDestroy } from 'svelte'

  let { children, ...props }: { children?: Snippet } & Konva.LayerConfig & KonvaEventHooks = $props()

  const stage = getStageContext()
  const node = new Konva.Layer(props)
  stage.add(node)
  setLayerContext(node)
  registerEvents(node, props)

  // prop 변경 시 노드 속성 업데이트
  const propKeys = Object.keys(props).filter(k => !k.startsWith('on'))
  for (const key of propKeys) {
    $effect(() => {
      node.setAttr(key, (props as Record<string, unknown>)[key])
    })
  }

  onDestroy(() => node.destroy())
</script>

{#if children}
  {@render children()}
{/if}
```

### 최종 사용 예시 — 동적 prop

```svelte
<script>
  import { Stage, Layer, Rect } from '$lib/components/konva'
  import { innerWidth, innerHeight } from 'svelte/reactivity/window'

  let x = $state(50)
  let y = $state(50)
  let fill = $state('#6633ff')
</script>

<Stage width={innerWidth.current} height={(innerHeight.current ?? 0) / 2}>
  <Layer>
    <Rect
      x={20} y={40}
      width={200} height={100}
      fill="purple"
      draggable
    />
    <Rect
      {x} {y} width={80} height={80}
      fill={fill}
      stroke="black" strokeWidth={2}
    />
  </Layer>
</Stage>

<input type="range" bind:value={x} min={0} max={400} />
<input type="range" bind:value={y} min={0} max={200} />
<input type="color" bind:value={fill} />
```

```
반응성 흐름:

  x = 50 → x = 150 (슬라이더)
    → Rect의 props.x 변경
    → x에 대한 $effect 재실행
    → node.setAttr('x', 150)
    → Konva가 캔버스에 다시 그림
    → 도형 이동 ✅

  innerWidth.current 변경 (창 리사이즈)
    → Stage의 props.width 변경
    → width에 대한 $effect 재실행
    → stage.setAttr('width', newWidth)
    → 캔버스 크기 조정 ✅
    → Layer/Rect는 그대로 유지 (Stage 재생성 없음) ✅
```

## 구현 6: $bindable로 양방향 바인딩 — 드래그 시 부모 상태 동기화

### 문제: 컴포넌트 내부에서 변경된 값이 부모에 반영되지 않음

```
부모 → 자식 (단방향):
  let x = $state(50)
  <Rect x={x} draggable />
  → 슬라이더로 x 변경 → Rect 이동 ✅
  → Rect 드래그 → x는 여전히 50 ❌

필요한 것: 양방향
  → 슬라이더로 x 변경 → Rect 이동 ✅
  → Rect 드래그 → x도 업데이트 ✅
  → bind:x={x} bind:y={y}
```

### 해결: $bindable + dragend 이벤트

```svelte
<!-- Rect.svelte ($bindable 버전) -->
<script lang="ts">
  import Konva from 'konva'
  import { getLayerContext } from './konva-context'
  import { type KonvaEventHooks, registerEvents } from './events'
  import { onDestroy } from 'svelte'

  let {
    x = $bindable(0),       // ← $bindable로 양방향 바인딩 지원
    y = $bindable(0),
    staticProps = false,     // true면 드래그 이벤트 비활성화
    ...props
  }: Konva.RectConfig & KonvaEventHooks & { staticProps?: boolean } = $props()

  const layer = getLayerContext()
  const node = new Konva.Rect({ x, y, ...props })
  layer.add(node)
  registerEvents(node, props)

  // x, y는 구조분해했으므로 별도 $effect 필요
  $effect(() => { node.setAttr('x', x) })
  $effect(() => { node.setAttr('y', y) })

  // 나머지 prop 업데이트 (x, y 제외)
  const propKeys = Object.keys(props).filter(k => !k.startsWith('on'))
  for (const key of propKeys) {
    $effect(() => {
      node.setAttr(key, (props as Record<string, unknown>)[key])
    })
  }

  // 드래그 종료 시 부모 상태 업데이트 (bind 사용 시에만)
  // staticProps는 초기 설정 플래그 — 런타임에 변경하지 않으므로 초기값 캡처 OK
  if (!staticProps) {
    node.on('dragend', (e) => {
      x = e.currentTarget.attrs.x
      y = e.currentTarget.attrs.y
    })
  }

  onDestroy(() => node.destroy())
</script>
```

```
$bindable 핵심 포인트:

1. 구조분해 + $bindable:
   let { x = $bindable(0), y = $bindable(0), ...props } = $props()
   → x, y를 구조분해하면 rest props에서 빠짐
   → 별도 $effect로 node.setAttr('x', x) 필요
   → $bindable이므로 컴포넌트 내부에서 x = newValue 가능

2. 드래그 → 부모 상태 업데이트:
   node.on('dragend', (e) => {
     x = e.currentTarget.attrs.x   // $bindable이라 부모 상태도 변경됨
   })

3. staticProps 플래그:
   bind:x 없이 단방향으로만 쓸 때:
     <Rect x={50} y={50} staticProps />
     → dragend 이벤트 등록 안 함 (불필요한 함수 실행 방지)

   bind:x로 양방향 쓸 때:
     <Rect bind:x={x} bind:y={y} draggable />
     → dragend 이벤트 등록 → 드래그 시 부모 상태 동기화
```

### 사용

```svelte
<script>
  import { Stage, Layer, Rect } from '$lib/components/konva'
  let x = $state(50)
  let y = $state(50)
</script>

<!-- 양방향: 슬라이더 ↔ 드래그 동기화 -->
<Rect bind:x bind:y width={80} height={80} fill="purple" draggable />
<span>X: {x}</span> <input type="range" bind:value={x} min={0} max={400} />
<span>Y: {y}</span> <input type="range" bind:value={y} min={0} max={200} />

<!-- 단방향: 드래그 가능하지만 부모 상태와 동기화 안 함 -->
<Rect x={20} y={40} width={100} height={50} fill="coral" draggable staticProps />
```

```
양방향 흐름:

  슬라이더 → x = 150
    → $effect: node.setAttr('x', 150) → 도형 이동

  도형 드래그 → dragend
    → x = e.currentTarget.attrs.x (예: 250)
    → $bindable이므로 부모의 x도 250으로 변경
    → 슬라이더도 250으로 이동

  staticProps = true일 때:
    → dragend 이벤트 등록 안 함
    → 도형은 드래그 가능하지만 부모 상태는 변경 안 됨
```

## 구현 7: bind:this로 내부 노드 참조 노출

선언적 컴포넌트로 충분하지만, 엣지 케이스에서 Konva 노드에 직접 접근해야 할 수 있다. `bind:this`와 `export`를 조합하면 내부 노드를 부모에 노출할 수 있다.

### bind:this + export const/function

Svelte에서 `bind:this`로 컴포넌트를 바인딩하면, 해당 컴포넌트가 `export`한 멤버에 접근할 수 있다.

```svelte
<!-- Rect.svelte — node를 export -->
<script lang="ts">
  // ... 기존 코드 ...
  const node = new Konva.Rect({ x, y, ...props })

  export { node }  // ← 외부에서 bind:this로 접근 가능
</script>
```

```svelte
<!-- Layer.svelte — node를 export -->
<script lang="ts">
  // ... 기존 코드 ...
  const node = new Konva.Layer(props)

  export { node }
</script>
```

```svelte
<!-- Stage.svelte — 함수로 export (비동기 초기화) -->
<script lang="ts">
  // ... 기존 코드 ...
  let stage: Konva.Stage | undefined = $state()

  // stage는 onMount에서 채워짐 → 초기에 undefined
  // 상수 대신 함수를 export하면 호출 시점에 참조 가능
  export function getNode() {
    return stage
  }
</script>
```

```
Stage가 함수를 export하는 이유:

  Rect/Layer:
    const node = new Konva.Rect(...)  ← 스크립트 최상위에서 즉시 생성
    export { node }                    ← 항상 유효한 인스턴스

  Stage:
    let stage = $state()              ← 초기에 undefined
    onMount(() => { stage = new Konva.Stage(...) })  ← 마운트 후 채워짐
    export { stage }                   ← bind:this 시점에 undefined일 수 있음!

    export function getNode() { return stage }
    → 호출 시점에 stage를 참조 → 마운트 이후라면 유효한 인스턴스 반환
```

### 사용

```svelte
<script>
  import { Stage, Layer, Rect } from '$lib/components/konva'
  import type { default as StageComponent } from '$lib/components/konva/Stage.svelte'
  import type { default as LayerComponent } from '$lib/components/konva/Layer.svelte'
  import type { default as RectComponent } from '$lib/components/konva/Rect.svelte'

  let stageRef: StageComponent
  let layerRef: LayerComponent
  let rectRef: RectComponent
</script>

<Stage bind:this={stageRef} width={500} height={300}>
  <Layer bind:this={layerRef}>
    <Rect bind:this={rectRef} x={50} y={50} width={100} height={80} fill="purple" />
  </Layer>
</Stage>

<button onclick={() => {
  // Rect/Layer — node에 직접 접근
  console.log(rectRef.node.getAttrs())    // { x: 50, y: 50, width: 100, ... }
  console.log(layerRef.node.getAttrs())

  // Stage — 함수 호출로 접근
  const stage = stageRef.getNode()
  console.log(stage?.getAttrs())          // { width: 500, height: 300, ... }

  // 명령형 조작도 가능
  rectRef.node.setAttr('x', 0)           // 도형을 x=0으로 이동
}}>
  정보 로그
</button>
```

```
bind:this로 접근 가능한 멤버:

  컴포넌트에서 export한 것만 접근 가능:
    export { node }           → ref.node
    export function getNode() → ref.getNode()

  export하지 않은 내부 변수/함수는 접근 불가.
  → 캡슐화 유지: 필요한 것만 선택적으로 노출

주의: 대부분의 경우 prop과 이벤트로 충분하다.
  bind:this + export는 Konva API를 직접 호출해야 하는 엣지 케이스용.
```

---

## 핵심 정리

| 구현 | 핵심 패턴 | 주의점 |
|------|----------|--------|
| Stage | `onMount` + Context 제공 (getter 함수) | ready 플래그로 자식 렌더링 지연 |
| Layer | 동기 생성 + Context 소비/제공 | `onDestroy`로 클린업 |
| Rect | 동기 생성 + Context 소비 | 자식 없음 (순수 로직 컴포넌트) |
| 이벤트 | `registerEvents` 공통 함수 | `on` 접두사 + 이벤트명으로 prop 매핑 |
| Prop 반응성 | prop별 `$effect` + `setAttr` | `onMount`로 생성, `$effect`로 업데이트 분리 |
| `$bindable` | x, y 양방향 바인딩 | 구조분해 시 별도 `$effect` 필요 |
| `bind:this` | `export { node }` / `export function` | Stage는 함수, Layer/Rect는 상수로 export |

> **$state.raw 고려**: Konva 노드 같은 대형 외부 객체를 `$state`로 관리할 때, 프로퍼티 변경이 아닌 재할당만 감지하면 되는 경우 `$state.raw`를 사용하면 Proxy 오버헤드를 줄일 수 있다.
