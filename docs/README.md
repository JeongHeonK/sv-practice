# 프로젝트 온보딩 인덱스

React → Svelte 전환을 위한 7일 커리큘럼 인덱스.

---

## 프로젝트 철학

### 제로 UI (Zero-JS First)

JS 없이도 form POST → Action → redirect 흐름만으로 핵심 기능이 동작한다.
JS가 있을 때는 `use:enhance`를 추가해 페이지 새로고침 없이 UX를 강화한다 — 이것이 Progressive Enhancement다.

방송 운영 AI 제품은 서버 부하가 급변하고 네트워크가 불안정한 환경에서도 동작해야 한다.
JS에 의존하지 않는 form-first 아키텍처는 그 요구에 가장 직접적으로 대응한다.

---

## 기술 스택 의존성 다이어그램

```
┌─────────────────────────────────────────────────┐
│                   SvelteKit                     │
│  라우팅 / load / Form Actions / Hooks / 배포    │
│                                                 │
│   ┌──────────────────────────────────────────┐  │
│   │              Svelte 5                    │  │
│   │  컴파일러 / $state / $derived / Snippets │  │
│   └──────────────────────────────────────────┘  │
│                                                 │
│   ┌─────────────┐    ┌────────────────────────┐ │
│   │   Houdini   │    │   Tailwind CSS v4      │ │
│   │  GraphQL    │    │   + bits-ui            │ │
│   │  (load 통합)│    │   (UI 컴포넌트 레이어) │ │
│   └─────────────┘    └────────────────────────┘ │
│                                                 │
│   ┌──────────────────────────────────────────┐  │
│   │          lingui (i18n)                   │  │
│   │   메시지 카탈로그 / 런타임 번역          │  │
│   └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

- Svelte 5는 컴파일러 기반 반응성 레이어. SvelteKit은 그 위의 풀스택 프레임워크.
- Houdini는 SvelteKit의 `load` 함수에 통합되어 GraphQL 쿼리를 자동 실행·캐시한다.
- bits-ui는 headless 컴포넌트를, Tailwind v4는 스타일을 담당한다.
- lingui는 런타임에 locale 메시지를 주입하며 어느 레이어와도 독립적이다.

---

## 7일 커리큘럼

### Day 1 — Svelte 반응성 & 컴포넌트 기초

| 문서 | 학습 목표 |
|------|----------|
| [svelte/01-reactivity.md](svelte/01-reactivity.md) | `$state` / `$derived` / `$effect` 동작 원리와 올바른 선택 기준을 익힌다. |
| [svelte/02-components.md](svelte/02-components.md) | Props, Snippets, Actions로 재사용 가능한 컴포넌트를 구성한다. |

### Day 2 — Svelte 상태 아키텍처

| 문서 | 학습 목표 |
|------|----------|
| [svelte/03-state-architecture.md](svelte/03-state-architecture.md) | 전역 상태 3가지 방법, Context API, 내장 반응성 클래스를 판단 기준과 함께 이해한다. |
| [svelte/04-performance.md](svelte/04-performance.md) | `$state.raw`, 지연 로딩, 가상화 등 성능 최적화 패턴을 파악한다. _(작성 예정)_ |

### Day 3 — SvelteKit 기초

| 문서 | 학습 목표 |
|------|----------|
| [sveltekit/01-project-structure.md](sveltekit/01-project-structure.md) | 디렉터리 관례와 파일 역할을 파악해 새 파일의 위치를 직관적으로 결정한다. |
| [sveltekit/02-core.md](sveltekit/02-core.md) | Hooks, Request / Response, 핵심 API 흐름을 이해한다. |
| [sveltekit/03-routing.md](sveltekit/03-routing.md) | 파일 기반 라우팅, 레이아웃, 에러 페이지 구조를 습득한다. |
| [sveltekit/04-loading-data.md](sveltekit/04-loading-data.md) | `load` 함수 유형과 Form Actions의 제로 UI 패턴을 익힌다. |

### Day 4 — SvelteKit 고급

| 문서 | 학습 목표 |
|------|----------|
| [sveltekit/05-advanced-patterns.md](sveltekit/05-advanced-patterns.md) | 인증 가드, 중첩 레이아웃, 스트리밍 등 실무 패턴을 다룬다. |
| [sveltekit/06-advanced-hooks.md](sveltekit/06-advanced-hooks.md) | `handle`, `handleFetch`, `handleError` 조합 전략을 이해한다. |
| [sveltekit/07-load-invalidation.md](sveltekit/07-load-invalidation.md) | `invalidate` / `invalidateAll` 전략으로 데이터 동기화를 제어한다. |

### Day 5 — Houdini GraphQL

| 문서 | 학습 목표 |
|------|----------|
| [stack/houdini.md](stack/houdini.md) | SvelteKit load에 Houdini를 통합하고, 캐시 정책과 낙관적 업데이트를 적용한다. _(작성 예정)_ |

### Day 6 — Tailwind v4 + bits-ui

| 문서 | 학습 목표 |
|------|----------|
| [stack/tailwind.md](stack/tailwind.md) | v4의 CSS-first 설정 방식과 디자인 토큰 관리 방법을 익힌다. _(작성 예정)_ |
| [stack/bits-ui.md](stack/bits-ui.md) | headless 컴포넌트의 합성 패턴과 접근성 기본값을 파악한다. _(작성 예정)_ |

### Day 7 — 성능 + 코드 리뷰 + 국제화 (여유 시)

| 문서 | 학습 목표 |
|------|----------|
| [svelte/05-code-review-guide.md](svelte/05-code-review-guide.md) | 안티패턴 15개를 숙지해 AI 생성 코드를 리뷰할 수 있는 수준에 도달한다. |
| [sveltekit/08-env-server-modules.md](sveltekit/08-env-server-modules.md) | 환경 변수 분류와 서버 전용 모듈 경계를 이해한다. |
| [sveltekit/09-build-deploy.md](sveltekit/09-build-deploy.md) | 어댑터 선택과 배포 파이프라인 구성을 파악한다. |
| [stack/svelte-i18n-lingui.md](stack/svelte-i18n-lingui.md) | lingui 메시지 카탈로그와 런타임 locale 전환 패턴을 익힌다. _(작성 예정)_ |

---

## 빠른 참조 (Quick Reference)

| 기술 | 핵심 패턴 | 문서 |
|------|----------|------|
| Svelte 5 반응성 | 파생값은 `$derived`, 부수효과는 `$effect`, 원시값은 `$state` | [svelte/01-reactivity.md](svelte/01-reactivity.md) |
| Svelte 컴포넌트 | Props는 `$props()`, 양방향 바인딩은 `$bindable`, 슬롯 대체는 `Snippets` | [svelte/02-components.md](svelte/02-components.md) |
| 전역 상태 | 인스턴스 공유 → Context, 앱 전역 → 모듈 싱글턴, 복잡 로직 → 클래스 | [svelte/03-state-architecture.md](svelte/03-state-architecture.md) |
| SvelteKit 라우팅 | `+page`, `+layout`, `+server`, `(group)`, `[[optional]]` 파일 관례 | [sveltekit/03-routing.md](sveltekit/03-routing.md) |
| 데이터 로딩 | 공개 데이터 → `+page.ts`, 인증 필요 → `+page.server.ts`, 뮤테이션 → Actions | [sveltekit/04-loading-data.md](sveltekit/04-loading-data.md) |
| Form Actions | `<form method="POST">` + Action + `use:enhance` = 제로 UI + Progressive Enhancement | [sveltekit/04-loading-data.md](sveltekit/04-loading-data.md) |
| Hooks | 인증은 `handle`, 외부 fetch 수정은 `handleFetch`, 에러 리포팅은 `handleError` | [sveltekit/06-advanced-hooks.md](sveltekit/06-advanced-hooks.md) |
| 환경 변수 | 공개 → `$env/static/public`, 서버 전용 → `$env/static/private` | [sveltekit/08-env-server-modules.md](sveltekit/08-env-server-modules.md) |
| Houdini | `load`에서 `graphql` 태그 함수 호출 → 캐시 자동 관리 | [stack/houdini.md](stack/houdini.md) |
| Tailwind v4 | `@theme` 블록에서 CSS 변수로 토큰 정의, `@apply` 최소화 | [stack/tailwind.md](stack/tailwind.md) |
| bits-ui | `<Primitive.button>` 합성으로 스타일만 교체, ARIA는 bits-ui가 관리 | [stack/bits-ui.md](stack/bits-ui.md) |
| lingui | `msg` 태그로 메시지 추출 → `.po` 파일 번역 → 런타임 `i18n.activate(locale)` | [stack/svelte-i18n-lingui.md](stack/svelte-i18n-lingui.md) |

---

_작성 예정 표시된 문서는 해당 Day 학습 전까지 생성된다._
