# Special Elements (특수 요소)

DOM 요소나 Svelte 컴포넌트 외에, 브라우저 객체(window, document, body)나 `<head>`에 선언적으로 접근할 수 있는 특수 요소들.

## `<svelte:element>` — 동적 HTML 요소 렌더링

`this` prop으로 렌더링할 HTML 태그를 동적으로 결정한다.

```svelte
<!-- Button.svelte — href 유무에 따라 <a> 또는 <button> 렌더링 -->
<svelte:element this={href ? 'a' : 'button'} {href} {...rest}>
  {@render children?.()}
</svelte:element>
```

> 이전 섹션(04-components)에서 다룬 내용. 여기서는 다른 특수 요소에 집중한다.

## `<svelte:window>` — Window 객체 접근

### 이벤트 바인딩

```svelte
<!-- 컴포넌트 최상위에 위치해야 함 (블록/요소 내부 불가) -->
<svelte:window onscroll={(e) => console.log(e)} />

<!-- 스크롤 확인용 큰 요소 -->
<div style="height: 3000px"></div>
```

```
$effect에서 수동으로 하는 것과 동일:
  $effect(() => {
    window.addEventListener('scroll', handler)
    return () => window.removeEventListener('scroll', handler)
  })

→ <svelte:window>가 이벤트 등록/해제를 자동 처리
```

### 값 바인딩

`scrollX`, `scrollY`, `innerWidth`, `innerHeight`, `outerWidth`, `outerHeight` 등을 바인딩할 수 있다.

```svelte
<script>
  let scrollY = $state(0)
</script>

<svelte:window bind:scrollY />

<div style="position: fixed; top: 10px; left: 10px;">
  scrollY: {scrollY}
</div>
```

### 더 쉬운 방법: `svelte/reactivity/window`

Svelte 5에서는 상태를 만들고 바인딩하는 대신, 반응형 모듈에서 직접 가져올 수 있다.

```svelte
<script>
  import { scrollY } from 'svelte/reactivity/window'
</script>

<!-- $state + bind 없이 바로 사용 -->
<p>scrollY: {scrollY.current}</p>
```

```
<svelte:window bind:scrollY>    → 상태 생성 + 바인딩 필요
scrollY from reactivity/window  → import만으로 사용 (더 간편)
```

## `<svelte:document>` — Document 객체 접근

`<svelte:window>`와 유사하지만, **document에만 존재하는 이벤트/프로퍼티**에 접근할 때 사용.

```svelte
<script>
  let visibilityState = $state<DocumentVisibilityState | null>(null)
</script>

<!-- 최상위에 위치해야 함 -->
<svelte:document
  bind:visibilityState
  onvisibilitychange={() => console.log(document.visibilityState)}
/>

<p>현재 상태: {visibilityState}</p>
```

```
탭 전환 시:
  다른 탭으로 이동 → visibilityState = 'hidden'
  다시 돌아옴      → visibilityState = 'visible'

사용 예: 탭 비활성 시 애니메이션 정지, 데이터 폴링 중단 등
```

### window vs document 이벤트 선택 기준

```
window에 있는 이벤트   → <svelte:window>    (scroll, resize, keydown 등)
document에만 있는 이벤트 → <svelte:document>  (visibilitychange 등)
```

## `<svelte:body>` — Body 요소 접근

`<body>`에만 존재하는 이벤트를 바인딩할 때 사용.

```svelte
<svelte:body onmouseenter={() => console.log('마우스 진입')} />
```

```
window에 없지만 body에 있는 이벤트 예시:
  mouseenter, mouseleave 등

사용 빈도는 낮지만 필요한 경우가 있으므로 알아두면 좋다.
```

## `<svelte:head>` — Head 태그에 삽입

`<head>` 태그에 `<title>`, `<meta>`, `<link>` 등을 동적으로 삽입한다. **SvelteKit으로 풀 앱을 만들 때 필수적**.

```svelte
<svelte:head>
  <title>페이지 제목</title>
  <meta name="description" content="페이지 설명" />
  <meta property="og:title" content="OG 제목" />
</svelte:head>
```

### 동적 SEO 태그

```svelte
<script>
  let { data } = $props()  // SvelteKit에서 페이지 데이터
</script>

<svelte:head>
  <title>{data.title} | My App</title>
  <meta name="description" content={data.description} />
</svelte:head>
```

```
SvelteKit에서 페이지별 동적 SEO:
  /blog/hello → <title>Hello World | My App</title>
  /about      → <title>About | My App</title>

→ 각 페이지 컴포넌트에 <svelte:head>를 넣고
  페이지 데이터에 따라 동적으로 태그 삽입
```

## `<svelte:boundary>` — 에러 경계

컴포넌트 렌더링/`$effect` 중 발생하는 에러를 포착하여 앱 전체가 깨지는 것을 방지한다. React의 Error Boundary와 동일한 개념.

### 기본 사용법

```svelte
<!-- BuggyComponent: 1초 후 에러 발생하는 예시 -->
<script lang="ts">
  let data: { text: string } | undefined = $state({ text: 'hello' })

  setTimeout(() => {
    data = undefined
  }, 1000)
</script>

<!-- data가 undefined가 되면 .text 접근 시 에러 -->
<p>{data.text ?? '대체 텍스트'}</p>
```

```svelte
<!-- 사용처 -->
<svelte:boundary onerror={(error) => console.log(error)}>
  <BuggyComponent />
</svelte:boundary>
```

```
에러 발생 시 동작:
1. <svelte:boundary> 내부 콘텐츠가 모두 제거됨
2. onerror 콜백이 호출됨 (Sentry 등 에러 리포팅에 활용)
3. failed 스니펫이 있으면 대체 콘텐츠 표시
```

### `failed` 스니펫으로 대체 UI 표시

```svelte
<svelte:boundary onerror={(error) => console.log(error)}>
  <BuggyComponent />

  {#snippet failed(error, reset)}
    <p>에러 발생: {error.message}</p>
    <button onclick={reset}>재시도</button>
  {/snippet}
</svelte:boundary>
```

```
failed 스니펫이 받는 인수:
  1. error — 발생한 에러 객체
  2. reset — 호출하면 boundary 내부 콘텐츠를 다시 생성 (재렌더링)

reset() 호출 → 컴포넌트 재생성 → 동일 버그면 다시 에러 발생
→ 간헐적 에러(네트워크 등)에는 유용, 코드 버그에는 근본 수정 필요
```

### 중요: 포착되는 에러 vs 포착되지 않는 에러

```
✅ 포착됨:
  - 렌더링 중 에러 (템플릿에서 undefined 접근 등)
  - $effect 내부에서 throw된 에러

❌ 포착 안 됨:
  - 이벤트 핸들러(콜백) 내부 에러 (onclick, oninput 등)
```

```svelte
<svelte:boundary onerror={(e) => console.log(e)}>
  <!-- ✅ 렌더링 에러 → boundary가 포착 -->
  <p>{data.text}</p>

  <!-- ✅ $effect 에러 → boundary가 포착 -->
  <!-- $effect(() => { throw new Error('effect error') }) -->

  <!-- ❌ 콜백 에러 → boundary가 포착하지 않음! -->
  <button onclick={() => console.log(data.text)}>
    클릭 시 에러 (포착 안 됨)
  </button>
</svelte:boundary>
```

```
왜 콜백 에러는 포착 안 되나?
→ 렌더링/$effect는 Svelte 런타임이 실행을 제어하므로 try-catch 가능
→ 이벤트 핸들러는 브라우저가 실행하므로 Svelte가 감싸지 못함
→ React Error Boundary도 동일한 제한
```

### React 비교

```
React                              Svelte
─────────────────────────         ─────────────────────────
class ErrorBoundary                <svelte:boundary>
  componentDidCatch(error)           onerror={(error) => ...}
  render() → fallback UI             {#snippet failed(error, reset)}
  key 변경으로 리셋                    reset() 함수 제공
  이벤트 핸들러 에러 포착 안 됨          동일 제한
```

## 보충: DOM 요소 차원 바인딩

코스 전반에서 다양한 `bind:` 사용법을 봤다 — `bind:value` (입력), `bind:this` (DOM 참조), `bind:scrollY` (window). 추가로 **모든 DOM 요소의 크기**도 바인딩할 수 있다.

```svelte
<script>
  let width = $state<number>()
  let height = $state<number>()
</script>

<textarea
  bind:offsetWidth={width}
  bind:offsetHeight={height}
></textarea>

<p>{width} x {height}</p>
```

### 바인딩 가능한 차원 속성

```
bind:clientWidth    — 패딩 포함, 보더/스크롤바 제외
bind:clientHeight
bind:offsetWidth    — 패딩 + 보더 + 스크롤바 포함
bind:offsetHeight
```

> 요소 크기가 변경되면 (드래그 리사이즈, 콘텐츠 변경 등) 바인딩된 상태가 자동 업데이트된다.

### 주의: 읽기 전용

```
바인딩된 상태를 코드에서 변경해도 DOM 요소의 실제 크기는 바뀌지 않는다.
→ DOM → 상태 (O) 단방향
→ 상태 → DOM (X)
```

## 보충: 미디어 요소 바인딩 (img / audio / video)

### 이미지: `bind:naturalWidth` / `bind:naturalHeight`

```svelte
<script>
  let imgWidth = $state<number>()
  let imgHeight = $state<number>()
</script>

<img
  src="https://picsum.photos/800/600"
  bind:naturalWidth={imgWidth}
  bind:naturalHeight={imgHeight}
/>

<p>원본 크기: {imgWidth} x {imgHeight}</p>
```

> `onload` 콜백으로 수동 처리하는 것보다 간편하다. 읽기 전용.

### audio / video: 풍부한 바인딩

커스텀 미디어 플레이어를 만들 때 핵심. 바인딩만으로 재생 상태를 완전히 제어할 수 있다.

```svelte
<!-- VideoPlayer.svelte -->
<script lang="ts">
  let { src }: { src: string } = $props()

  let duration = $state(0)
  let currentTime = $state(0)
  let paused = $state(true)
  let playbackRate = $state(1)
  let volume = $state(1)
  let muted = $state(false)
  let buffered = $state<{ start: number; end: number }[]>([])
</script>

<!-- svelte-ignore a11y_media_has_caption -->
<video
  {src}
  style="max-width: 100%;"
  bind:duration
  bind:currentTime
  bind:paused
  bind:playbackRate
  bind:volume
  bind:muted
  bind:buffered
></video>
```

> 상태 이름과 바인딩 프로퍼티 이름이 같으면 `bind:duration={duration}` 대신 `bind:duration`으로 축약 가능.

### 읽기/쓰기 vs 읽기 전용

```
읽기 + 쓰기 (양방향):
  bind:currentTime   — 탐색(seek) 가능
  bind:paused        — 재생/일시정지 제어
  bind:playbackRate  — 재생 속도 변경
  bind:volume        — 볼륨 조절
  bind:muted         — 음소거 토글

읽기 전용:
  bind:duration      — 전체 길이
  bind:buffered      — 버퍼링된 구간 [{start, end}, ...]
  bind:ended         — 재생 완료 여부
  bind:seeking       — 탐색 중 여부
  bind:videoWidth    — 비디오 원본 너비 (video만)
  bind:videoHeight   — 비디오 원본 높이 (video만)
```

### 커스텀 플레이어 활용 예시

```svelte
<!-- 재생/일시정지 버튼: paused 상태만 토글 -->
<button onclick={() => paused = !paused}>
  {paused ? '재생' : '일시정지'}
</button>

<!-- 탐색 바: currentTime을 range input에 바인딩 -->
<input type="range" min={0} max={duration} bind:value={currentTime} />

<!-- 볼륨 조절 -->
<input type="range" min={0} max={1} step={0.01} bind:value={volume} />

<!-- 재생 속도 -->
<select bind:value={playbackRate}>
  <option value={0.5}>0.5x</option>
  <option value={1}>1x</option>
  <option value={1.5}>1.5x</option>
  <option value={2}>2x</option>
</select>
```

```
핵심: bind로 상태를 동기화하면
→ 읽기: UI에 현재 상태 표시 (currentTime, duration 등)
→ 쓰기: UI 조작이 바로 미디어에 반영 (paused, volume 등)
→ 브라우저 기본 컨트롤 없이도 완전한 커스텀 플레이어 구현 가능
```

## 정리

| 특수 요소 | 대상 | 주요 용도 |
|-----------|------|-----------|
| `<svelte:element>` | 동적 HTML 태그 | `this`로 태그 결정 |
| `<svelte:window>` | `window` | scroll, resize, keydown 등 |
| `<svelte:document>` | `document` | visibilitychange 등 |
| `<svelte:body>` | `<body>` | mouseenter 등 |
| `<svelte:head>` | `<head>` | title, meta, SEO 태그 |
| `<svelte:boundary>` | 에러 경계 | onerror + failed 스니펫 |

| 바인딩 대상 | 방향 | 예시 |
|-------------|------|------|
| `bind:value` | 양방향 | 입력 ↔ 상태 |
| `bind:this` | 단방향 | DOM → 변수 |
| `bind:scrollY` | 양방향 | 스크롤 ↔ 상태 |
| `bind:offsetWidth` | 읽기 전용 | DOM 크기 → 상태 |
| `bind:naturalWidth` | 읽기 전용 | 이미지 원본 크기 |
| `bind:currentTime` | 양방향 | 미디어 탐색 |
| `bind:paused` | 양방향 | 재생/일시정지 |
| `bind:duration` | 읽기 전용 | 미디어 전체 길이 |

> `<svelte:window>`, `<svelte:document>`, `<svelte:body>`는 **컴포넌트 최상위**에 위치해야 한다. `<svelte:head>`, `<svelte:boundary>`는 어디서든 사용 가능.
