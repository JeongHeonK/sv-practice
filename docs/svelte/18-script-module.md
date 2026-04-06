# `<script module>` — 컴포넌트 인스턴스 간 코드 공유

## Svelte의 데이터 공유 방법 총정리

```
방법 1: 모듈 레벨 상태 (svelte.ts 파일에서 export)
──────────────────────────────────────────────────
→ 앱 전체 어디서든 import해서 사용
→ 서버에서 안전하지 않음 (모든 요청이 같은 인스턴스 공유)
→ 11-reactive-classes.md 패턴 6 참조

방법 2: Context API (setContext / getContext)
──────────────────────────────────────────────────
→ 특정 컴포넌트 트리 내에서만 공유
→ 부모가 정의, 자식만 접근 가능 — 외부 컴포넌트는 접근 불가
→ 서버에서도 안전 (요청별 컴포넌트 트리가 독립)
→ 14-context-api.md 참조

방법 3: <script module> (이 문서)
──────────────────────────────────────────────────
→ 동일한 컴포넌트의 모든 인스턴스 간 코드 공유
→ 한 번만 실행됨
```

## 핵심 개념

Svelte 컴포넌트의 스크립트 태그는 두 종류가 있다:

```svelte
<!-- 인스턴스 스크립트: 컴포넌트 인스턴스마다 실행 -->
<script lang="ts">
  let count = $state(0)  // 각 인스턴스가 자신만의 count를 가짐
</script>

<!-- 모듈 스크립트: 한 번만 실행, 모든 인스턴스가 공유 -->
<script lang="ts" module>
  let shared = $state(0)  // 모든 인스턴스가 같은 shared를 참조
</script>
```

```
<Counter />  ← 인스턴스 1: 자신만의 count, 공유 shared
<Counter />  ← 인스턴스 2: 자신만의 count, 공유 shared
<Counter />  ← 인스턴스 3: 자신만의 count, 공유 shared

인스턴스 스크립트: 3번 실행 (각 인스턴스마다)
모듈 스크립트: 1번만 실행 (최초 import 시)
```

## `<script>` vs `<script module>`

| 항목 | `<script>` | `<script module>` |
|------|------------|---------------------|
| 실행 횟수 | 인스턴스마다 | 1번만 |
| 상태 범위 | 각 인스턴스 독립 | 모든 인스턴스 공유 |
| `$props` 접근 | 가능 | 불가 |
| `$effect` 사용 | 가능 | 가능 (주의 필요) |
| `export` | props로 노출 | 일반 JS export (외부 import 가능) |
| 용도 | 인스턴스별 로직 | 공유 상태, 유틸, 타입 export |

## 변수 접근 방향

```
<script module>                    <script>
  let shared = $state(0)    ──→     // ✅ 모듈 변수에 접근 가능
                                    shared++
                             ✘
  // ❌ 인스턴스 변수에      ←──     let { name } = $props()
  //    접근 불가                    let local = $state('')
```

```
모듈 → 인스턴스: 접근 불가 (어떤 인스턴스인지 알 수 없음)
인스턴스 → 모듈: 접근 + 수정 가능
```

## 예제 1: 인스턴스 개수 추적

```svelte
<!-- Button.svelte -->
<script lang="ts" module>
  let buttonCount = $state(0)

  // 외부에서 import 가능한 함수
  export function getButtonCount() {
    return buttonCount
  }
</script>

<script lang="ts">
  import { onMount } from 'svelte'

  // 각 인스턴스가 마운트/파괴될 때 공유 카운트 업데이트
  $effect(() => {
    buttonCount++
    return () => { buttonCount-- }
  })
</script>

<!-- 모든 인스턴스에서 동일한 값 표시 -->
<p>현재 버튼 수: {buttonCount}</p>
<button><slot /></button>
```

```svelte
<!-- 사용처 -->
<script>
  import Button from './Button.svelte'
  import { getButtonCount } from './Button.svelte'  // named export

  let show = $state(true)
</script>

<label><input type="checkbox" bind:checked={show} /> 3번째 버튼 표시</label>

<Button>1번</Button>
<Button>2번</Button>
{#if show}
  <Button>3번</Button>
{/if}

<!-- 모듈에서 export한 함수 사용 -->
<button onclick={() => alert(getButtonCount())}>버튼 수 확인</button>
```

```
초기 상태 (show = true):
  Button 1: buttonCount = 3
  Button 2: buttonCount = 3
  Button 3: buttonCount = 3    ← 모두 같은 값 (공유 상태)

show = false (3번째 파괴):
  Button 1: buttonCount = 2
  Button 2: buttonCount = 2    ← $effect cleanup에서 감소
```

### $state vs 일반 변수 — 판단 기준

```
$state가 필요한 경우:
  → 템플릿에서 표시 ({buttonCount})
  → $effect에서 읽기
  → $derived의 의존성
  즉, 변경 시 UI나 다른 반응형 값이 갱신되어야 할 때

일반 변수(let)로 충분한 경우:
  → 함수 호출로만 접근 (getButtonCount())
  → 내부 로직에서만 사용
  → UI 갱신과 무관한 추적용 데이터

  let buttonCount = $state(0)  // 템플릿에서 {buttonCount} 표시 → $state 필요
  let buttonCount = 0          // getButtonCount()로만 읽기 → 일반 변수 충분
```

> **규칙**: `$state`는 반응형이 필요한 변수에만 사용한다. 단순히 모듈 스크립트에 있다는 이유로 모든 변수를 `$state`로 선언할 필요 없다.

## 예제 2: AudioPlayer — 하나만 재생

```svelte
<!-- AudioPlayer.svelte -->
<script lang="ts" module>
  // 모든 AudioPlayer 인스턴스가 공유
  // → 하나만 재생 중이도록 보장하는 용도
  let currentlyPlaying = $state<HTMLAudioElement | null>(null)
</script>

<script lang="ts">
  let { src }: { src: string } = $props()
  let audio: HTMLAudioElement

  function play() {
    // 다른 인스턴스가 재생 중이면 먼저 정지
    if (currentlyPlaying && currentlyPlaying !== audio) {
      currentlyPlaying.pause()
    }
    audio.play()
    currentlyPlaying = audio
  }
</script>

<audio bind:this={audio} {src}></audio>
<button onclick={play}>재생</button>
```

```
AudioPlayer 3개 렌더링:
  <AudioPlayer src="a.mp3" />  ← 재생 클릭 → currentlyPlaying = audio A
  <AudioPlayer src="b.mp3" />  ← 재생 클릭 → A 정지, currentlyPlaying = audio B
  <AudioPlayer src="c.mp3" />  ← 재생 클릭 → B 정지, currentlyPlaying = audio C

→ currentlyPlaying은 모듈 레벨이므로 모든 인스턴스가 같은 변수 참조
→ 하나를 재생하면 나머지는 자동 정지
```

## 예제 3: VideoPlayer — Set으로 인스턴스 추적 + 전역 제어

모든 VideoPlayer 인스턴스의 video 요소를 `Set`으로 추적하고, 외부에서 모두 재생/일시정지하거나 한 번에 하나만 재생되도록 제어하는 패턴.

### Set 선택 이유

```
배열 vs Set:
  배열: 중복 가능, 삭제 시 indexOf + splice 필요
  Set:  중복 불가, add/delete가 O(1), 삭제가 간편

→ 컴포넌트 인스턴스 추적에는 Set이 적합
```

### VideoPlayer 컴포넌트

```svelte
<!-- VideoPlayer.svelte -->
<script lang="ts" module>
  // 모든 인스턴스의 video 요소를 추적하는 공유 Set
  const allVideos = new Set<HTMLVideoElement>()

  // 외부에서 사용할 수 있도록 export
  export function getAllVideos(): Set<HTMLVideoElement> {
    return allVideos
  }

  export function playAll() {
    allVideos.forEach((video) => video.play())
  }

  export function pauseAll() {
    allVideos.forEach((video) => video.pause())
  }
</script>

<script lang="ts">
  let { src }: { src: string } = $props()

  let video: HTMLVideoElement

  // 마운트 시 Set에 추가, 파괴 시 제거
  $effect(() => {
    allVideos.add(video)
    return () => { allVideos.delete(video) }
  })

  // 한 번에 하나만 재생: 이 video가 재생되면 나머지 일시정지
  function onplay() {
    allVideos.forEach((_video) => {
      if (_video !== video) {
        _video.pause()
      }
    })
  }
</script>

<!-- svelte-ignore a11y_media_has_caption -->
<video {src} style="max-width: 100%;" bind:this={video} {onplay}></video>
```

### 사용처

```svelte
<script>
  import VideoPlayer from './VideoPlayer.svelte'
  import { playAll, pauseAll, getAllVideos } from './VideoPlayer.svelte'

  let showThird = $state(true)
</script>

<button onclick={playAll}>모두 재생</button>
<button onclick={pauseAll}>모두 일시정지</button>
<button onclick={() => console.log(getAllVideos())}>동영상 목록</button>

<label><input type="checkbox" bind:checked={showThird} /> 3번째 표시</label>

<VideoPlayer src="video1.mp4" />
<VideoPlayer src="video2.mp4" />
{#if showThird}
  <VideoPlayer src="video3.mp4" />
{/if}
```

### 동작 흐름

```
초기 (3개 마운트):
  allVideos = Set { video1, video2, video3 }

showThird = false:
  $effect cleanup → allVideos.delete(video3)
  allVideos = Set { video1, video2 }

video1 재생 클릭:
  onplay() → allVideos.forEach → video2.pause()
  → 한 번에 하나만 재생됨

pauseAll() 클릭:
  allVideos.forEach → video1.pause(), video2.pause()

주의: onplay에서 다른 video를 pause하므로
      playAll()은 마지막 video만 재생됨 (나머지는 onplay가 pause)
```

### 패턴 요약

```
1. <script module>에서 Set 정의 (공유 저장소)
2. <script>의 $effect에서 add/delete (인스턴스 라이프사이클 추적)
3. <script module>에서 제어 함수 export (외부에서 일괄 조작)
4. <script>에서 이벤트 핸들러로 상호 제어 (하나만 재생 등)
```

## `<script module>`에서 export (named export)

`<script module>`의 `export`는 **props가 아니라 일반 JS 모듈 export**다.

```svelte
<!-- MyComponent.svelte -->
<script lang="ts" module>
  export type MyEvent = { id: string; timestamp: number }
  export const COMPONENT_VERSION = '1.0.0'
</script>
```

```ts
// 다른 파일에서 import 가능
import MyComponent, { type MyEvent, COMPONENT_VERSION } from './MyComponent.svelte'
```

```
<script>에서 export       → Svelte props ($props로 정의하는 것이 Svelte 5 방식)
<script module>에서 export → 일반 JS export (타입, 상수, 유틸 등)
```

## 주의사항

```
1. 서버 안전성 ⚠️
   → 모듈 레벨 상태는 서버에서 모든 요청이 공유함 (방법 1과 동일한 문제)
   → SSR 환경에서는 요청 간 상태 오염 가능 (사용자 A의 데이터가 B에게 노출)
   → 브라우저에서도 SPA 내 모든 컴포넌트가 같은 인스턴스 공유 → 메모리 누수 주의
   → 서버 안전이 필요하면 Context API 사용 (요청별 독립된 트리)

2. $effect 사용 시
   → 모듈 스크립트에서 $effect를 쓰면 컴포넌트 라이프사이클과 무관하게 동작
   → cleanup이 필요하면 $effect.root 사용 (11번 문서 참조)

3. $state vs let 선택
   → 템플릿이나 $effect에서 읽히는 변수만 $state로 선언
   → 함수 호출로만 접근하는 내부 변수는 일반 let으로 충분
   → 불필요한 $state 사용은 Proxy 오버헤드를 유발
```
