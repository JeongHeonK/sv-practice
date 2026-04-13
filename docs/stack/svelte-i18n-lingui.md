# svelte-i18n-lingui

> **Day 7+ 이후 학습 권장** — 실무 투입 초기에는 불필요.
> 온보딩 1주일 완료 후 실제 국제화 작업이 필요할 때 읽을 것.

---

## 1. Lingui란?

JavaScript/TypeScript 생태계 범용 i18n 라이브러리. React에 종속되지 않아 SvelteKit에도 적용 가능하다.

**핵심 특징:**

- **ICU 메시지 포맷** — 복수형, 성별, 조건 분기를 하나의 메시지 문자열로 표현
- **컴파일 타임 처리** — `t` 매크로가 빌드 시 최적화된 i18n 호출로 변환됨. 런타임 파싱 비용 없음
- **타입 안전** — 메시지 ID와 변수가 TypeScript 타입으로 보호됨
- **자동 추출** — `lingui extract` 명령 하나로 코드에서 번역 대상 메시지를 수집

---

## 2. ICU 메시지 포맷

단순 문자열부터 복수형까지 하나의 포맷으로 처리한다.

```
# 단순 메시지
"안녕하세요, {name}님"

# 복수형
"{count, plural, one {메시지 # 개} other {메시지 # 개}}"

# 성별
"{gender, select, male {그가} female {그녀가} other {그들이}} 작성했습니다"
```

react-i18next의 `count` 인터폴레이션 방식보다 표현력이 높고, 번역가 도구(Crowdin, Lokalise 등)와 호환성이 좋다.

---

## 3. SvelteKit 통합 설정

### 설치

```bash
pnpm add @lingui/core
pnpm add -D @lingui/cli @lingui/babel-plugin-lingui-macro
```

Svelte에서는 `@lingui/react` 대신 `@lingui/core`를 사용한다.

### `lingui.config.js`

```javascript
import { defineConfig } from '@lingui/cli'

export default defineConfig({
  sourceLocale: 'ko',
  locales: ['ko', 'en', 'ja'],
  catalogs: [
    {
      path: '<rootDir>/src/locales/{locale}/messages',
      include: ['src'],
    },
  ],
})
```

### i18n 싱글톤 초기화 (`src/lib/i18n.ts`)

```typescript
import { i18n } from '@lingui/core'

export const defaultLocale = 'ko'

export async function loadLocale(locale: string) {
  const { messages } = await import(`../locales/${locale}/messages`)
  i18n.load(locale, messages)
  i18n.activate(locale)
}
```

### `+layout.svelte`에서 locale 설정

```svelte
<script lang="ts">
  import { i18n } from '@lingui/core'
  import { loadLocale } from '$lib/i18n'

  // 앱 초기화 시 기본 locale 활성화
  loadLocale('ko')
</script>

<slot />
```

SvelteKit SSR 환경에서는 `+layout.server.ts`에서 `Accept-Language` 헤더를 파싱해 locale을 결정하고, `+layout.svelte`로 전달하는 패턴을 사용한다.

---

## 4. 기본 사용 패턴

### 단순 문자열

```svelte
<script lang="ts">
  import { t } from '@lingui/core/macro'
</script>

<h1>{t`안녕하세요`}</h1>
<p>{t`환영합니다, ${name}님`}</p>
```

`t` 매크로는 빌드 시 아래처럼 변환된다.

```javascript
// 빌드 후 생성된 코드 (직접 작성하지 않음)
import { i18n } from '@lingui/core'
i18n._({ id: 'mY42CM', message: '안녕하세요' })
```

### 복수형

```svelte
<script lang="ts">
  import { plural } from '@lingui/core/macro'

  let count = $state(0)
</script>

<p>
  {plural(count, {
    one: '메시지 # 개',
    other: '메시지 # 개',
  })}
</p>
```

### 커스텀 ID 지정

메시지 ID를 명시하면 리팩토링 후에도 기존 번역이 유지된다.

```typescript
import { t } from '@lingui/core/macro'

const message = t({
  id: 'greeting.welcome',
  message: `환영합니다, ${name}님`,
})
```

---

## 5. 메시지 추출 → 번역 → 빌드 사이클

```
코드 작성 (t 매크로 사용)
        ↓
lingui extract        ← 코드 스캔, .po 파일 생성/업데이트
        ↓
.po 파일 번역         ← 번역가 작업 (또는 직접 편집)
        ↓
lingui compile        ← .po → 최적화된 .js 메시지 카탈로그
        ↓
프로덕션 빌드
```

```bash
# 메시지 추출 (새 문자열 추가 시)
pnpm lingui extract

# 번역 완료 후 컴파일
pnpm lingui compile

# package.json scripts 등록 권장
```

추출된 `.po` 파일 예시:

```po
#: src/routes/+page.svelte:5
msgid "안녕하세요"
msgstr "Hello"

#: src/routes/+page.svelte:6
msgid "환영합니다, {name}님"
msgstr "Welcome, {name}"
```

---

## 6. 동적 locale 전환

```typescript
import { loadLocale } from '$lib/i18n'

async function switchLocale(locale: string) {
  await loadLocale(locale)
  // Svelte 반응성에 의해 t 매크로 출력이 자동 갱신됨
}
```

초기 번들에는 기본 locale만 포함되고, 나머지는 동적 import로 지연 로딩된다.

---

## React react-i18next vs lingui

| 항목 | react-i18next | lingui + Svelte |
|------|--------------|----------------|
| 패키지 | `react-i18next`, `i18next` | `@lingui/core`, `@lingui/cli` |
| 번역 훅/매크로 | `useTranslation()` 훅 | `t` 매크로 (빌드 타임) |
| 복수형 처리 | `count` 인터폴레이션 | ICU 포맷 (`plural`) |
| 메시지 추출 | `i18next-parser` (별도 설정) | `lingui extract` (내장) |
| 런타임 파싱 | 있음 | 없음 (컴파일 타임 변환) |
| 번역 파일 포맷 | JSON | `.po` / `.json` 선택 가능 |
| Provider 필요 | `<I18nextProvider>` | 없음 (싱글톤 직접 사용) |
