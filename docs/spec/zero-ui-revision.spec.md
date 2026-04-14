# 제로 UI 개념 전면 수정 Spec

## 개요

- **목적**: 문서 전반에 잘못 정의된 "제로 UI" 개념을 실제 의미에 맞게 수정
- **문제 정의**: 현재 문서는 "제로 UI"를 "JavaScript 없이도 동작하는 Progressive Enhancement"로 잘못 정의하고 있음. 실제 의미는 **브라우저 UI 없이 Claude Code 같은 AI 대화창/CLI에서 핵심 기능을 직접 실행할 수 있는 API-first 설계 원칙**임.
- **맥락**: 공간 운영 AI 제품(온도·밝기 조절, 방문자 통계 파악 등)은 웹 브라우저 없이도 Claude Code 같은 대화창에서 실행 가능해야 한다.

---

## 요구사항

### 기능 요구사항 (P0)

1. **제로 UI 정의 재작성** — 3개 파일 모두에서 "JS 없이 form POST" 관련 설명 제거
2. **Progressive Enhancement 섹션 삭제** — `05-advanced-patterns.md` Section 4 전체 (no-js 패턴, use:enhance 비교, GraphQL 비교 포함) 삭제
3. **새 제로 UI 섹션 작성** — `05-advanced-patterns.md`에 "대화창 실행 가능 설계" 개념 중심 섹션 신규 작성 (코드 없이 ASCII 다이어그램만)
4. **Form Actions 섹션 추가** — `04-loading-data.md`에 Form Actions 섹션 신규 추가
5. **README 철학 섹션 재작성** — 대화창 실행 가능성 중심으로 재작성
6. **README Quick Reference 수정** — Form Actions 행에서 "제로 UI + Progressive Enhancement" 문구 완전 제거

### 비기능 요구사항

- 교육 문서 철학 준수: 개념 중심, 코드 최소화, ASCII 시각화
- 용어 "제로 UI"는 유지 (변경 없음)
- 섹션 삭제 후 05 파일의 번호를 재조정 (5→4, 6→5, 7→6)

---

## 기술 설계

### 아키텍처 결정 사항

| 질문 | 결정 | 근거 |
|------|------|------|
| "제로 UI" 용어 유지? | 유지 | 사용자 명시 확인 |
| PE 내용 처리 | 삭제 | PE는 별개 개념이며 제로 UI와 무관 |
| 04 파일 Form Actions 추가 | 추가 | 데이터 로딩(load) 문서에 뮤테이션(actions) 상호보완 필요 |
| 새 제로 UI 코드 예시 | ASCII 다이어그램만 | 문서 철학: 코드 최소화 |
| 05 섹션 번호 | 재조정 (5→4, 6→5, 7→6) | 4번 교체 후 연속성 유지 |

### 파일별 변경 상세

#### 1. `docs/README.md`

**변경 위치 1 — "제로 UI" 철학 섹션 (lines 9–15)**

- Before: "JS 없이도 form POST → Action → redirect 흐름만으로 핵심 기능이 동작한다..."
- After: 브라우저 화면 없이 Claude Code 같은 AI 대화창/CLI에서도 핵심 기능(온도·밝기 조절, 통계 파악 등)을 실행할 수 있어야 한다는 API-first 설계 원칙으로 재작성

**변경 위치 2 — Quick Reference 테이블 Form Actions 행 (line 117)**

- Before: `= 제로 UI + Progressive Enhancement`
- After: PE 언급 완전 제거, Form Actions의 서버 뮤테이션 역할만 설명

#### 2. `docs/sveltekit/05-advanced-patterns.md`

**변경 위치 1 — Section 4 전체 교체 (lines 116–218)**

- Before: "Progressive Enhancement — JS 유무에 따른 UI 분기" (JS 비활성화 감지, no-js 클래스 패턴, $derived 쓰기, use:enhance 비교, GraphQL 경계 포함)
- After: "제로 UI — 대화창 실행 가능 설계" (개념 정의 + 왜 중요한가 + Claude Code → API 호출 ASCII 다이어그램 + Form Actions와의 관계 한 줄)

**변경 위치 2 — 섹션 번호 재조정**

- 5. 병렬 실행 & 스트리밍 → 4
- 6. 네비게이션 진행률 표시 → 5
- 7. 코드 재실행 제어 → 6

#### 3. `docs/sveltekit/04-loading-data.md`

**변경 위치 — 새 섹션 추가**

- Form Actions 섹션을 새로 추가 (`+page.server.ts`의 `actions` export, `use:enhance`, ActionData 수신)
- Docs Philosophy 준수: 개념 설명 중심, 코드는 구조 예시 수준만
- 기존 섹션 번호와 충돌하지 않게 말미(React 비교 전)에 배치

---

## 범위

### 포함 (In Scope)

- `docs/README.md` 제로 UI 철학 섹션 재작성
- `docs/README.md` Quick Reference Form Actions 행 수정
- `docs/sveltekit/05-advanced-patterns.md` Section 4 교체 + 섹션 번호 재조정
- `docs/sveltekit/04-loading-data.md` Form Actions 섹션 추가

### 제외 (Out of Scope)

- `docs/sveltekit/07-load-invalidation.md` — 사용자가 직접 관리하는 파일, 수정 금지
- `docs/svelte/` 하위 파일 — 제로 UI 관련 내용 없음
- `docs/stack/` 하위 파일 — 제로 UI 관련 내용 없음
- MCP 서버 등록/설정 코드 — 범위 외

---

## 엣지케이스 & 주의사항

- `05-advanced-patterns.md`의 Section 4 삭제 시 React/Next.js 비교 테이블 맨 아래 행 "Progressive Enhancement" 행도 제거 또는 수정 필요
- `04-loading-data.md`에 Form Actions 섹션 추가 시 기존 섹션 번호(1~7)와 충돌하지 않게 8번으로 추가하거나, 기존 흐름에 자연스럽게 삽입

---

## 의존성

- 변경 대상 파일 3개는 서로 독립적으로 수정 가능
- README Quick Reference는 04 파일 변경 후 링크 일관성 확인 필요

---

## 인터뷰 로그 (주요 결정 요약)

| 질문 | 결정 | 근거 |
|------|------|------|
| 제로 UI 용어 유지? | 유지 | 용어 자체는 맞음, 정의만 틀렸음 |
| PE 내용 처리 | 완전 삭제 | 제로 UI와 개념 충돌, 별개 문서 주제 |
| 04에 Form Actions 추가? | 추가 | 데이터 로딩 문서에 뮤테이션 흐름 보완 필요 |
| 제로 UI = 무엇? | 대화창(Claude Code 등) 실행 가능 API-first 설계 | 공간 운영 AI: 온도·밝기·통계 등 대화창으로 조작 |
| 새 섹션 코드 수준 | ASCII 다이어그램만 | 교육 문서 철학: 코드 최소화 |
| README 철학 섹션 방향 | 대화창 실행 가능성 중심 재작성 | 기존 내용은 PE 설명으로 완전히 틀림 |
