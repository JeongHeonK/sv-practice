# Svelte 온보딩 문서 최적화 Spec

## 개요

- **목적**: 기존 18개 학습 노트를 "신입이 바로 AI와 함께 Svelte 코드를 짜고 코드리뷰할 수 있는" 온보딩 문서로 재구성
- **대상 사용자**: React 경험자 신입 개발자
- **문제 정의**: 현재 문서는 강의 노트 스타일로 파편화되어 있어, 실무 투입 시 "어디서 무엇을 찾는지", "AI가 생성한 코드의 문제를 어떻게 잡는지"에 대한 가이드가 없음

---

## 요구사항

### 기능 요구사항 (P0)

| 우선순위 | 요구사항 |
|---------|---------|
| P0 | 신입이 순서대로 읽고 Svelte 5 실무 투입 가능 |
| P0 | AI(Claude)가 생성한 코드에서 안티패턴 식별 및 수정 가능 |
| P0 | 아키텍처 판단 가능 (언제 Context vs 모듈 상태, $derived vs $effect 등) |
| P0 | 18개 파일 → 5~7개로 재구성, 기존 파일은 archive/ 이동 |
| P1 | README.md 신입 첫날 가이드 + 읽는 순서 |
| P1 | AI 코드리뷰 프롬프트 템플릿 (복사해서 Claude에 넣는 형태) |
| P1 | SvelteKit 확장 (Form Actions, 네비게이션 가드, 낙관적 UI, 에러 핸들링) |
| P2 | CLAUDE.md에 넣을 수 있는 Svelte 코딩 규칙 텍스트 |

### 비기능 요구사항

- **스타일**: React 비교 제거, Svelte 독립적 설명
- **코드 예시**: 실무에 쓸 수 있는 컴포넌트 수준 (로그인 폼, 데이터 테이블 등)
- **안티패턴**: 코드리뷰 수준의 리뷰어가 될 수 있도록 Bad→Good 예시로

---

## 기술 설계

### 파일 구조

```
docs/svelte/
├── README.md                       ← 신입 첫날 가이드 + 읽는 순서 [신규]
├── 01-reactivity.md                ← 반응성 원리 [리라이트]
├── 02-components.md                ← 컴포넌트 & 템플릿 API [신규 통합]
├── 03-state-architecture.md        ← 상태 관리 아키텍처 [신규 통합]
├── 04-sveltekit.md                 ← SvelteKit [대폭 확장]
├── 05-code-review-guide.md         ← 안티패턴 & AI 코드리뷰 가이드 [신규]
└── archive/                        ← 기존 18개 파일 이동
    ├── 01-reactivity.md
    ├── 02-runes.md
    └── ... (기존 파일 전부)
```

> 내용이 많으면 02 또는 03을 분리해 최대 7개까지 허용

---

### 각 파일 설계

#### README.md
- 이 문서들이 왜 존재하는지 (온보딩 목적)
- **읽는 순서**: 01 → 02 → 03 → 04 → 05 (안티패턴은 마지막)
- **주제별 퀵 네비게이션**: "$bindable이 뭔지 모르겠다 → 02번", "전역상태 어떻게 → 03번" 등
- 온보딩 완료 기준 (안티패턴 파일 다 이해하면 코드리뷰 가능 수준)

---

#### 01-reactivity.md — 반응성 원리

**출처**: 01, 02, 03번 통합

**섹션 구성**:
1. Svelte는 컴파일러다 (DOM 직접 업데이트, VDOM 없음)
2. `$state` — 읽기/쓰기 반응형 값
   - Proxy 기반 깊은 반응성
   - `$state.raw` — 재할당 전용 (선택 기준)
   - 구조분해 시 반응성 끊김 (경고)
3. `$derived` — 파생값
   - `$derived.by` — 복잡한 표현식
   - props 의존 값은 반드시 `$derived` (핵심 규칙)
4. `$effect` — 부작용 (탈출구)
   - 쓰기 전 체크리스트
   - 무한루프 주의 + untrack
   - cleanup (return 함수)
   - `$effect.pre`
5. 의존성 자동 추적 원리 (동적 의존성, Lazy evaluation)
6. 디버깅: `$inspect`, `$inspect.trace()`, `$state.snapshot`

---

#### 02-components.md — 컴포넌트 & 템플릿 API

**출처**: 04, 05, 06, 07, 16, 17, 18번 통합

**섹션 구성**:
1. 파일 = 컴포넌트 (.svelte 파일 구조)
2. Props: `$props()`, 기본값, rest props
3. 이벤트: `onclick` 속성 방식 (레거시 `on:click` 대비)
4. 커스텀 이벤트 = 함수 prop
5. `$bindable` — 양방향 바인딩
6. `export const` — 부모에서 자식 메서드 호출
7. Snippets & `{@render}` — `children`, render props 대체
8. 템플릿 문법: `{#each}` (key 필수), `{#if}`, `{#await}`, `{@const}`
9. 배열 뮤테이션 (직접 변경 가능)
10. 스타일링: CSS 스코프, `:global`, 조건부 class (배열/객체), CSS 변수
11. Actions (`use:`) & Attachments (`{@attach}`) — DOM 동작 부착
    - 반응형 옵션 전달 패턴 (함수 래핑)
    - `{@attach}` 권장 (Svelte 5.29+)
12. 특수 요소: `<svelte:window>`, `<svelte:head>`, `<svelte:boundary>`, `<svelte:element>`
    - DOM/미디어 바인딩 (`bind:offsetWidth`, `bind:paused` 등)
13. `<script module>` — 인스턴스 간 코드 공유
    - `$state` vs 일반 변수 판단 기준
    - named export 패턴

**실무 예시**: 재사용 가능한 Button 컴포넌트 (rest props + snippet + attachment)

---

#### 03-state-architecture.md — 상태 관리 아키텍처

**출처**: 08, 09, 10, 11, 12, 14, 15번 통합

**섹션 구성**:
1. Effect Lifecycle 전체 타이밍 다이어그램 (Mount → State Change → Destroy)
2. `onMount` vs `$effect` 비교표
3. 양방향 파생 패턴
   - `$derived` 오버라이드 (5.25+, 낙관적 UI)
   - 안티패턴: `$effect` 두 개로 핑퐁
   - getter/setter 객체로 쓰기 가능한 파생 구현
4. 로직 재사용: 팩토리 함수 → 클래스 진화
   - 치명적 함정: 값 직접 반환 vs getter
   - 클래스가 더 나은 이유 (컴파일러 자동 getter/setter)
5. `.svelte.ts` 파일 확장자 (rune 사용을 위한 필수 조건)
6. 전역 상태 3가지 방법
   - `$state` 객체 (기본 선택)
   - 팩토리 함수 (원시값 공유)
   - 클래스 싱글톤 (복잡한 로직)
7. `$effect.root` — 컴포넌트 밖 effect
8. SSR 안전성 경고 (모듈 상태 vs Context API)
9. Context API
   - `createContext` (5.40+, 권장)
   - `setContext`/`getContext`/`hasContext`
   - 반응형 값 전달 (원시값 ❌, `$state` 객체 ✅)
   - `hasContext` 폴백 패턴
   - Context 캡슐화 파일 (Symbol 키, 타입 중복 제거)
10. 내장 리액티브 클래스
    - `SvelteDate`, `SvelteMap`, `SvelteSet`, `SvelteURL`
    - `MediaQuery`, `Tween`, `Spring`
    - `createSubscriber` — 외부 이벤트 반응성 연결
11. 언제 무엇을 선택할 것인가 (판단 트리)
    - 컴포넌트 로컬 상태 / 전역 / SSR 안전 / 트리 범위

**실무 예시**: 데이터 테이블 컴포넌트 (정렬 + 페이지네이션 + 로딩 상태를 클래스로 캡슐화)

---

#### 04-sveltekit.md — SvelteKit (대폭 확장)

**출처**: 13번 기반 + 신규 내용 추가

**섹션 구성**:
1. 파일 컨벤션 & 실행 환경 맵 (표)
2. 데이터 플로우 시각화 (hooks → layout → page)
3. `load` 함수: server vs universal 비교
4. `+page.svelte`에서 `$props()` 로 data 수신
5. `hooks.server.ts` — 네비게이션 가드
   - `event.locals`로 인증 상태 전파
   - 미인증 시 `redirect()`
   - 레이아웃 단위 권한 분리
6. Form Actions
   - 기본 `<form action>` + `+page.server.ts` `actions`
   - `use:enhance` — progressive enhancement
   - Named actions (`?/login`, `?/register`)
   - 성공 후 `invalidate` / `invalidateAll`로 데이터 갱신
   - 낙관적 UI: `use:enhance` 콜백에서 UI 선업데이트 + 실패 시 롤백
7. 에러 핸들링
   - `+error.svelte` — 경로별 에러 UI
   - `handleError` — 로깅 vs 사용자 메시지 분리
   - `error()` helper로 typed error throw
8. 서버 전용 API: `+server.ts`

**실무 예시**: 로그인 폼 (Form Action + use:enhance + 낙관적 UI + 에러 처리 전체 흐름)

---

#### 05-code-review-guide.md — 안티패턴 & AI 코드리뷰 가이드

**섹션 구성**:

##### 1. AI 코드리뷰 프롬프트 템플릿
복사해서 Claude에 넣을 수 있는 텍스트:
```
아래 Svelte 코드를 Svelte 5 모범 사례 관점에서 리뷰해 주세요.
체크할 항목:
1. Svelte 4 레거시 문법 (on:click, export let, store, use:action 등)
2. $effect로 상태 동기화 (→ $derived로 대체해야 함)
3. props 의존 값을 $derived 없이 선언
4. 리액티브 객체/클래스 인스턴스 구조분해
5. .ts 파일에서 rune 사용 (.svelte.ts 필요)
6. SSR 위험한 모듈 레벨 상태
7. $effect에 async 직접 사용
8. Action 옵션을 객체로 직접 전달 (함수 래핑 필요)
9. Context vs 모듈 상태 선택의 적절성

이슈 발견 시: 문제 설명 + 수정 코드를 제시해 주세요.
```

##### 2. CLAUDE.md 추가용 Svelte 코딩 규칙
복사해서 CLAUDE.md에 넣을 수 있는 텍스트 블록

##### 3. 안티패턴 카탈로그 (Bad → Good)

각 항목: 제목 / ❌ 버그 코드 / 설명 / ✅ 수정 코드

- **AP-01**: $effect로 상태 동기화 → $derived
- **AP-02**: props 의존 파생값을 일반 변수로 선언
- **AP-03**: 리액티브 객체 구조분해
- **AP-04**: Svelte 4 레거시 이벤트 핸들러 (`on:click` → `onclick`)
- **AP-05**: `export let` → `$props()`
- **AP-06**: `writable` store → `$state` 클래스
- **AP-07**: `.ts` 파일에서 rune 사용 (`.svelte.ts` 필요)
- **AP-08**: 모듈 레벨 상태를 SSR에서 사용 (→ Context API)
- **AP-09**: `$effect(async () => ...)` (→ 별도 async 함수 + `$effect(() => { fn() })`)
- **AP-10**: Action 옵션 객체 직접 전달 (→ `() => ({ ... })` 함수 래핑)
- **AP-11**: `||` 로 기본값 처리 시 `0` falsy 버그 (→ `?? ` 또는 명시적 `!== undefined`)
- **AP-12**: `$effect` 핑퐁 (양방향 상태 동기화) → getter/setter 객체
- **AP-13**: Context에 원시값 전달 (반응성 없음 → `$state` 객체 전달)
- **AP-14**: `$bindable` 없이 자식이 부모 상태 직접 변경 (ownership 경고)
- **AP-15**: `SvelteMap`/`SvelteSet` 대신 일반 Map/Set을 `$state`로 래핑

---

## 범위

### 포함 (In Scope)
- 기존 18개 파일 내용 전체 흡수 + 재구성
- SvelteKit Form Actions, 네비게이션 가드, 낙관적 UI, 에러 핸들링 확장
- README.md (신입 첫날 가이드)
- AI 코드리뷰 프롬프트 템플릿 + CLAUDE.md용 텍스트
- archive/ 폴더로 기존 파일 이동

### 제외 (Out of Scope)
- better-auth / drizzle 특정 스택 연동
- TypeScript 심화 (타입 정의 등)
- 테스트 작성 (vitest)
- 배포 (adapter 설정)
- 기존 .svelte 소스 파일 수정

---

## 엣지케이스 & 주의사항

- **분량 초과**: 02-components.md 또는 03-state-architecture.md가 너무 길면 분리 허용 (최대 7개)
- **코드 예시 품질**: 실무 예시(로그인 폼, 데이터 테이블)는 실제로 동작하는 코드여야 함
- **Svelte 버전 명시**: `createContext`(5.40+), `$derived` 오버라이드(5.25+), `{@attach}`(5.29+) 등 버전 태그 유지
- **archive/ 이동**: 기존 링크가 끊길 수 있으나 온보딩 최적화 목적이므로 허용

---

## 의존성

- Svelte 5 (5.40+ 일부 기능)
- SvelteKit
- TypeScript (코드 예시는 `lang="ts"` 기준)

---

## 인터뷰 로그

| 결정 사항 | 근거 |
|---------|------|
| 18개 → 5개 파일 (최대 7개) | 카테고리별 통합으로 탐색 용이성 향상 |
| React 비교 제거 | Svelte 독립적 사고를 위해 (오히려 혼란 유발) |
| 안티패턴 전용 파일(05번) | 코드리뷰 능력 = 안티패턴 인식 능력, 별도 집중 필요 |
| 아키텍처 판단 수준의 리뷰 | Context vs 모듈, $derived vs $effect 등 설계 레벨 |
| SvelteKit 대폭 확장 | Form Actions + 낙관적 UI + 가드 + 에러 핸들링 실무 핵심 |
| 기존 파일 archive/ 이동 | 레거시 참조 가능성 유지 + 온보딩 디렉토리 청결 |
| AI 프롬프트 템플릿 포함 | "AI가 짠 코드를 신입이 리뷰" 시나리오 직접 지원 |
| 실무 예시 (로그인 폼, 데이터 테이블) | 개념 예시보다 즉시 활용 가능한 코드로 |
