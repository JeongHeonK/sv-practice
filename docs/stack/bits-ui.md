# bits-ui

## 1. bits-ui란?

**Headless UI 라이브러리** — 동작(interaction)과 접근성(accessibility)만 제공하고, 스타일은 전혀 강제하지 않는다.

- Svelte 5 완전 호환 (Runes 기반 내부 구현)
- React 생태계의 Radix UI와 같은 철학, Svelte 용으로 재작성
- 모든 컴포넌트가 TypeScript 타입을 내장 제공
- WAI-ARIA 스펙 준수 — ARIA 속성, 키보드 내비게이션, Focus trap이 기본 내장

> "스타일은 내 것, 동작과 접근성은 라이브러리 것"

---

## 2. Tailwind v4 조합 패턴

### `class` prop 직접 오버라이드

bits-ui의 모든 컴포넌트는 `class` prop을 받는다. Tailwind 유틸리티 클래스를 그대로 전달하면 된다.

```svelte
<Dialog.Trigger class="bg-primary text-white px-4 py-2 rounded-lg hover:bg-primary/90">
  열기
</Dialog.Trigger>

<Dialog.Overlay class="fixed inset-0 bg-black/50 backdrop-blur-sm" />

<Dialog.Content class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2
  bg-white dark:bg-zinc-900 p-6 rounded-xl shadow-xl w-full max-w-md">
  <!-- ... -->
</Dialog.Content>
```

### `child` 스니펫 패턴 (커스텀 엘리먼트 교체)

기본 렌더 엘리먼트를 완전히 교체하고 싶을 때 사용한다. `asChild`는 제거되었고 Svelte 5의 `child` 스니펫으로 대체됐다.

```svelte
<Dialog.Trigger>
  {#snippet child({ props })}
    <MyButton {...props}>열기</MyButton>
  {/snippet}
</Dialog.Trigger>
```

`props`를 반드시 스프레드해야 ARIA 속성과 이벤트 핸들러가 연결된다.

---

## 3. 핵심 컴포넌트 예시

### Dialog (모달)

```svelte
<script lang="ts">
  import { Dialog } from 'bits-ui'

  let isOpen = $state(false)
</script>

<Dialog.Root bind:open={isOpen}>
  <Dialog.Trigger class="px-4 py-2 bg-blue-600 text-white rounded-lg">
    열기
  </Dialog.Trigger>

  <Dialog.Portal>
    <Dialog.Overlay class="fixed inset-0 bg-black/50" />
    <Dialog.Content
      class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2
             bg-white p-6 rounded-xl shadow-xl max-w-md w-full"
    >
      <Dialog.Title class="text-lg font-semibold">제목</Dialog.Title>
      <Dialog.Description class="text-sm text-gray-600 mt-1">설명</Dialog.Description>

      <div class="mt-4">
        <!-- 콘텐츠 -->
      </div>

      <Dialog.Close class="absolute top-4 right-4 text-gray-400 hover:text-gray-600">
        ✕
      </Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

핵심 구조: `Root → Trigger / Portal → (Overlay + Content → Title, Description, Close)`

`Dialog.Portal`은 컨텐츠를 `<body>` 하단에 렌더링해 z-index 충돌을 방지한다.

### Combobox (검색 가능한 셀렉트)

```svelte
<script lang="ts">
  import { Combobox } from 'bits-ui'
</script>

<Combobox.Root>
  <Combobox.Input placeholder="검색..." />
  <Combobox.Trigger />
  <Combobox.Portal>
    <Combobox.Content>
      <Combobox.Group>
        <Combobox.GroupHeading>옵션</Combobox.GroupHeading>
        <Combobox.Item value="apple">사과</Combobox.Item>
        <Combobox.Item value="banana">바나나</Combobox.Item>
      </Combobox.Group>
    </Combobox.Content>
  </Combobox.Portal>
</Combobox.Root>
```

### Svelte 트랜지션 적용

애니메이션이 필요하면 `forceMount` + `child` 스니펫을 조합한다.

```svelte
<script lang="ts">
  import { Dialog } from 'bits-ui'
  import { fly, fade } from 'svelte/transition'
</script>

<Dialog.Root>
  <Dialog.Portal>
    <Dialog.Overlay forceMount>
      {#snippet child({ props, open })}
        {#if open}
          <div {...props} transition:fade={{ duration: 150 }} />
        {/if}
      {/snippet}
    </Dialog.Overlay>

    <Dialog.Content forceMount>
      {#snippet child({ props, open })}
        {#if open}
          <div {...props} transition:fly={{ y: 10, duration: 200 }}>
            <!-- 콘텐츠 -->
          </div>
        {/if}
      {/snippet}
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

`forceMount`는 컴포넌트를 DOM에 유지시켜 Svelte 트랜지션이 동작할 수 있게 한다.

---

## 4. 접근성 내장 기능

bits-ui가 자동으로 처리하는 것들:

| 기능 | 동작 |
|------|------|
| ARIA 속성 | `role`, `aria-expanded`, `aria-controls` 등 자동 주입 |
| 키보드 내비게이션 | `Enter`, `Space`, `Escape`, 방향키 처리 |
| Focus trap | 모달 열림 시 포커스를 컨텐츠 내부로 가둠 |
| Focus 복원 | 모달 닫힘 시 트리거로 포커스 복귀 |
| 스크롤 잠금 | Dialog 열림 시 body 스크롤 잠금 |

별도 구현 없이 WAI-ARIA 기준을 충족한다.

---

## 5. 설치

```bash
pnpm add bits-ui
```

Tailwind v4 프로젝트에서는 추가 설정 없이 바로 `class` prop으로 스타일링 가능하다.

---

## React Radix UI vs bits-ui

| 항목 | Radix UI (React) | bits-ui (Svelte) |
|------|-----------------|-----------------|
| 패키지 | `@radix-ui/react-*` (컴포넌트별) | `bits-ui` (단일 패키지) |
| 스타일 prop | `className` | `class` |
| 커스텀 엘리먼트 | `asChild` prop | `child` 스니펫 |
| 컴포넌트 패턴 | `<Dialog.Root>` | `<Dialog.Root>` |
| 트랜지션 | CSS / framer-motion | Svelte 트랜지션 내장 |
| 상태 관리 | `useState` | `$state` (Runes) |
| 포털 | `<Dialog.Portal>` | `<Dialog.Portal>` |
