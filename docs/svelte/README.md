# Svelte 5 온보딩 가이드

Svelte 5 + SvelteKit 실무 투입을 위한 온보딩 문서.
**목표**: 순서대로 읽고 나면 AI가 생성한 Svelte 코드를 리뷰하고 수정할 수 있는 수준이 된다.

---

## 읽는 순서

| 순서 | 파일 | 주제 | 소요 시간 |
|:---:|------|------|:---------:|
| 1 | [01-reactivity.md](01-reactivity.md) | 컴파일러, `$state`, `$derived`, `$effect`, 의존성 추적 | ~20분 |
| 2 | [02-components.md](02-components.md) | Props, 이벤트, Snippets, 스타일링, Actions, 특수 요소 | ~30분 |
| 3 | [03-state-architecture.md](03-state-architecture.md) | 라이프사이클, 전역 상태, Context API, 내장 클래스 | ~30분 |
| 4 | [04-sveltekit.md](04-sveltekit.md) | 라우팅, load, Form Actions, 인증 가드, 에러 핸들링 | ~25분 |
| 5 | [05-code-review-guide.md](05-code-review-guide.md) | 안티패턴 15개 + AI 코드리뷰 프롬프트 | ~20분 |

---

## 주제별 퀵 네비게이션

### "이게 뭔지 모르겠다"

| 주제 | 어디를 볼까 |
|------|-----------|
| `$state`, `$derived`, `$effect` | [01-reactivity.md](01-reactivity.md) |
| `$props()`, `$bindable` | [02-components.md](02-components.md) §2, §5 |
| `{#each}`, `{#await}`, `{@const}` | [02-components.md](02-components.md) §8 |
| Snippets, `{@render}` | [02-components.md](02-components.md) §7 |
| Actions (`use:`), Attachments (`{@attach}`) | [02-components.md](02-components.md) §11 |
| `<svelte:boundary>`, `<svelte:head>` | [02-components.md](02-components.md) §12 |
| `<script module>` | [02-components.md](02-components.md) §13 |
| `onMount` vs `$effect` | [03-state-architecture.md](03-state-architecture.md) §2 |
| Context API, `createContext` | [03-state-architecture.md](03-state-architecture.md) §9 |
| `SvelteMap`, `SvelteDate`, `MediaQuery` | [03-state-architecture.md](03-state-architecture.md) §10 |
| Form Actions, `use:enhance` | [04-sveltekit.md](04-sveltekit.md) §6 |
| `hooks.server.ts`, 인증 가드 | [04-sveltekit.md](04-sveltekit.md) §5 |

### "어떤 걸 써야 하지?"

| 판단이 필요한 상황 | 어디를 볼까 |
|-------------------|-----------|
| `$derived` vs `$effect` | [01-reactivity.md](01-reactivity.md) §4 (체크리스트) |
| `$state` vs `$state.raw` | [01-reactivity.md](01-reactivity.md) §2 |
| 팩토리 함수 vs 클래스 | [03-state-architecture.md](03-state-architecture.md) §4 |
| 모듈 상태 vs Context API | [03-state-architecture.md](03-state-architecture.md) §8, §9 |
| 전역 상태 3가지 방법 | [03-state-architecture.md](03-state-architecture.md) §6 |
| Action vs Attachment | [02-components.md](02-components.md) §11 |
| server load vs universal load | [04-sveltekit.md](04-sveltekit.md) §3 |

---

## 온보딩 완료 기준

05번(안티패턴)의 15개 항목을 모두 이해하고, AI가 생성한 코드에서 해당 패턴을 식별할 수 있으면 **코드리뷰 가능 수준**이다.

---

## 기존 학습 노트

기존 18개 파일은 [archive/](archive/) 폴더에 보관되어 있다. 세부 내용이나 추가 예제가 필요하면 참조할 수 있다.
