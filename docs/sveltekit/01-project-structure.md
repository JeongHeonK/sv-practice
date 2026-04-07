# 프로젝트 폴더 구조

SvelteKit 프로젝트의 디렉토리 레이아웃과 주요 설정 파일.

---

## 1. 루트 설정 파일

```text
프로젝트 루트/
├── svelte.config.js          ← SvelteKit 설정 (어댑터, 별칭 등)
├── vite.config.ts            ← Vite 빌드 설정 + 플러그인
├── tsconfig.json             ← TypeScript 설정 (건드릴 일 거의 없음)
├── eslint.config.js          ← 린팅 규칙
├── .prettierrc               ← 포매팅 규칙
├── package.json / pnpm-lock.yaml
└── static/                   ← 정적 에셋 (favicon 등, 빌드 시 그대로 복사)
```

---

## 2. `src/` 핵심 구조

```text
src/
├── app.html                  ← HTML 셸 (모든 페이지의 기본 템플릿)
├── app.css                   ← 글로벌 스타일
├── app.d.ts                  ← TypeScript 앱 타입 선언
├── lib/                      ← 공유 코드 ($lib 별칭으로 접근)
│   └── components/           ← 재사용 컴포넌트
├── routes/                   ← 파일 기반 라우팅
│   ├── +layout.svelte        ← 루트 레이아웃
│   └── +page.svelte          ← 홈페이지 (/)
└── .svelte-kit/              ← 자동 생성 (빌드/타입), 삭제해도 재생성됨
```

> 라우팅 상세 → 03-routing.md

---

## 3. `app.html` — 플레이스홀더

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%sveltekit.assets%/favicon.png" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    %sveltekit.head%
  </head>
  <body>
    <div>%sveltekit.body%</div>
  </body>
</html>
```

| 플레이스홀더 | 빌드 시 대체 내용 |
|--|--|
| `%sveltekit.assets%` | `static/` 폴더 경로 |
| `%sveltekit.head%` | `<svelte:head>` 등에서 주입한 스크립트, 스타일, SEO 메타 |
| `%sveltekit.body%` | 현재 라우트의 페이지 콘텐츠 |

---

## 4. `$lib` 별칭과 커스텀 별칭

`$lib`은 `src/lib/`을 가리키는 **기본 제공 별칭**이다. 깊은 상대 경로 대신 사용:

```svelte
<!-- ❌ 상대 경로 -->
<script>
  import Button from '../../../lib/components/Button.svelte'
</script>

<!-- ✅ $lib 별칭 -->
<script>
  import Button from '$lib/components/Button.svelte'
</script>
```

**커스텀 별칭**도 `svelte.config.js`에서 설정 가능:

```js
// svelte.config.js
const config = {
  kit: {
    alias: {
      $components: 'src/lib/components/*'
    }
  }
}
```

```svelte
<!-- 커스텀 별칭 사용 -->
<script>
  import Button from '$components/Button.svelte'
</script>
```

> 별칭 변경 후 **개발 서버 재시작** 필요.

---

## 5. 환경 변수 (`$env`)

SvelteKit은 환경 변수를 4개 모듈로 제공한다.

| | Static (빌드 시 치환) | Dynamic (런타임 읽기) |
|--|--|--|
| **Private** (서버 전용) | `$env/static/private` | `$env/dynamic/private` |
| **Public** (클라이언트 노출 가능) | `$env/static/public` | `$env/dynamic/public` |

`PUBLIC_` 접두사 규칙: public 모듈에 노출하려면 변수명이 `PUBLIC_`으로 시작해야 한다.

```ts
// +page.server.ts (서버 전용)
import { DATABASE_URL } from '$env/static/private'

// +page.svelte (클라이언트 노출 가능)
import { PUBLIC_API_URL } from '$env/static/public'
```

> `.env` 파일은 개발 환경에서 자동 로드된다. 프로덕션 환경 변수는 배포 플랫폼(Vercel, Node 등)에서 설정한다.

**실무 팁**: 대부분의 경우 `static` 모듈이면 충분하다. `dynamic`은 빌드 없이 런타임에 변수를 바꿔야 하는 경우(예: 컨테이너 환경)에 사용한다.

---

## 6. `.svelte-kit/` 폴더

- `dev`나 `build` 시 자동 생성되는 내부 폴더
- 생성된 TypeScript 타입(`$types` 등)이 포함되어 있어 타입 안전성에 기여
- **삭제해도 안전** — 서버 재시작 시 재생성됨
- `.gitignore`에 기본 포함

---

## 7. 프로젝트 구조 철학

SvelteKit은 의도적으로 아키텍처 패턴을 처방하지 않는다 (FSD, Atomic Design, Clean Architecture 등 무관). 프레임워크가 제공하는 구조적 규약은 3가지뿐이다:

- `routes/` — 파일 기반 라우팅 (URL 구조)
- `lib/` — 공유 코드 (`$lib` 별칭)
- `lib/server/` — 서버 전용 코드 (클라이언트에 절대 노출되지 않음)

실무에서 자주 쓰이는 `lib/` 하위 구조 예시:

```text
src/lib/
├── components/     ← UI 컴포넌트
├── server/         ← 서버 전용 (DB, auth 등)
├── utils/          ← 유틸리티 함수
└── types/          ← 타입 정의
```

아키텍처 패턴(FSD, Atomic 등)은 SvelteKit과 무관하다. `lib/` 내부 구조는 팀 컨벤션에 맞게 자유롭게 구성한다.

---

## React/Next.js 비교

| 개념 | Next.js | SvelteKit |
|--|--|--|
| HTML 셸 | `app/layout.tsx` (Root Layout) | `src/app.html` |
| 정적 에셋 | `public/` | `static/` |
| 공유 코드 별칭 | `@/` (tsconfig paths) | `$lib` (기본 제공) |
| 파일 라우팅 | `app/` 디렉토리 | `src/routes/` |
| 자동 생성 폴더 | `.next/` | `.svelte-kit/` |
| 설정 파일 | `next.config.js` | `svelte.config.js` |
