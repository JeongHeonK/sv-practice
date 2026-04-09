# 05-hooks.md 최적화 Spec

## 개요

- **목적**: 1170줄의 05-hooks.md를 다른 sveltekit 문서(616~886줄)와 일관된 분량으로 최적화
- **대상**: SvelteKit 학습 중인 React/Next.js 경험자
- **문제 정의**: 실습 섹션 중복, 잘못된 패턴 시퀀스 분량 과다, React 비교표 과잉, 에러 페이지 섹션이 02-core.md와 겹침

---

## 요구사항

### 구조 결정

| 항목 | 결정 | 근거 |
|------|------|------|
| 파일 구조 | 2파일로 분리 | 1170줄 과분량, 기본/고급 개념 규모 분리 |
| 05-hooks.md | handle + locals + handleError + 에러페이지 링크 | 가장 먼저 배워야 할 핵심 3개 |
| 06-advanced-hooks.md | resolve 옵션 + handleFetch + reroute + transport | 고급/특수 케이스 |
| 목표 분량 | 05: ~300~350줄, 06: ~250~300줄 | 합계 약 600줄 (기존 1170줄의 50%) |

### 기능 요구사항

#### P0: 제거 대상
- **실습 섹션**: Section 2(handle 실습), Section 3(locals 실습) — 개념 설명 코드와 중복
- **잘못된 패턴 시퀀스**: `잘못된 → 차선악(await parent) → 올바른` 3단계 → 올바른 패턴만 유지
- **React/Next.js 비교표 전체**: Section 1, 2, 3에 있는 모든 비교표 제거
- **전체 흐름 정리 다이어그램** (Section 3 말미): 제거
- **filterSerializedResponseHeaders 앞부분**: SSR fetch 캡처 원리 설명(~40줄) 제거 → 코드 + 표만
- **에러 페이지 섹션**: 02-core.md로 링크 대체

#### P0: 유지 대상
- **handle 요청 흐름 다이어그램**: Client → handle(①②③) → 응답 ASCII
- **handle 3가지 패턴 코드**: resolve 전 / resolve 후 / resolve 우회
- **isDataRequest 표**: SSR vs 클라이언트 내비게이션 구분
- **sequence 소개 1줄**: 코드 없이 `sequence(handleAuth, handleLogging)` 1줄만
- **locals vs 쿠키 비교표**: 생존 범위 / 저장 위치 / 용도 / 보안 비교
- **올바른 인증 패턴 코드**: handle에서 cookies → locals.user 주입
- **경로 보호 전략 A vs B**: 중앙 집중(handle) vs 분산(개별 페이지 load)
- **locals TypeScript 타입 정의**: app.d.ts App.Locals 인터페이스
- **handleError 서버 + 클라이언트 코드**: 각각 분리하여 유지, init 훅 간단 언급
- **예상된 vs 예상치 못한 오류 비교표**

#### P1: 에러 페이지 관련 후크 특유 내용 (05에 남김)
- **레이아웃 오류 특수 동작**: `(marketing)/+layout.server.ts` 오류 → 같은 레벨 `+error.svelte` 안 쓰임 → 상위 레벨 사용
- **src/error.html 정적 폴백은 제거** (너무 엣지케이스)

#### P1: 06-advanced-hooks.md 구성
- **resolve 옵션**: transformPageChunk 상세(lang 예제) + preload 표 + filterSerializedResponseHeaders 코드 + 표
- **handleFetch**: 두 사례 모두 유지 (내부 URL 교체 + 크로스 오리진 쿠키 전달)
- **reroute**: reroute vs redirect 비교표 + 다국어 라우팅 예제
- **transport**: 핵심 압축 (Rect 클래스 코드 + encode/decode 선언 + 흐름 다이어그램)
- **말미 전체 훅 정리 표**: 06 말미로 이동 (현재 05 말미에 있음)

---

## 기술 설계

### 05-hooks.md 재편 후 구조

```
# SvelteKit Hooks

## 1. Hooks 개요          (~25줄)
   - 3종류 표 (hooks.server / hooks.client / hooks.ts)
   - 파일 위치 트리
   - ~~React 비교표~~ 제거

## 2. handle 훅           (~70줄)
   - 요청 흐름 다이어그램 (ASCII)
   - 기본 동작 (event / resolve / 반환값 설명)
   - 3가지 패턴 코드 (resolve 전 / 후 / 우회)
   - isDataRequest 표
   - sequence 소개 1줄
   - ~~실습 섹션~~ 제거
   - ~~handle vs middleware 비교표~~ 제거

## 3. event.locals        (~100줄)
   - locals 개념 다이어그램 (요청 생존 범위)
   - locals vs 쿠키 비교표
   - ~~잘못된 패턴~~ 제거
   - ~~차선악(await parent)~~ 제거
   - 올바른 패턴 코드 (handle에서 주입)
   - 경로 보호 전략 A vs B (둘 다 유지)
   - TypeScript 타입 정의 (app.d.ts)
   - ~~실습 섹션~~ 제거
   - ~~전체 흐름 정리 다이어그램~~ 제거
   - ~~React 비교표~~ 제거

## 4. +error.svelte 페이지 (~15줄)
   - > 기본 구조는 02-core.md Section 7 참고 링크
   - 레이아웃 오류 특수 동작 (같은 레벨 +error.svelte 건너뜀)
   - ~~오류 탐색 순서 전체~~ 제거
   - ~~src/error.html 정적 폴백~~ 제거

## 5. handleError 훅      (~80줄)
   - 공유 훅 개념 표
   - 서버 handleError 코드 (로깅 + Sentry 주석)
   - 클라이언트 handleError 코드
   - 발생 시점 설명 (SSR vs 클라이언트 내비게이션)
   - init 훅 간단 언급 (코드 포함)
   - 예상된 vs 예상치 못한 오류 비교표
```

예상 총 분량: **~290~310줄**

### 06-advanced-hooks.md 신규 구조

```
# SvelteKit 고급 훅

## 1. resolve 옵션         (~70줄)
   - transformPageChunk 상세 (app.html %lang% 예제)
   - preload 표 + 1줄 설명 (FOUT 관련)
   - filterSerializedResponseHeaders: 언제 쓰는지 + 코드 + 표
   - ~~앞부분 SSR 캡처 원리 설명~~ 제거

## 2. handleFetch 훅       (~60줄)
   - 기본 시그니처
   - 사례 1: 내부 URL 교체 (성능)
   - 사례 2: 크로스 오리진 쿠키 전달 (인증)
   - handle vs handleFetch 비교표 (유지 — React 비교 아님)

## 3. reroute 훅           (~50줄)
   - 기본 동작 코드
   - reroute vs redirect 비교표
   - 다국어 라우팅 예제 (translations 레코드)

## 4. transport 훅         (~60줄)
   - 문제 코드 (Rect 클래스, 직렬화 오류)
   - transport 선언 코드 (encode/decode)
   - 흐름 다이어그램 (server → encode → HTML → decode → 클라이언트)

## 말미: 전체 훅 정리 표   (~15줄)
   - 6개 훅 한눈에 보기 (파일/종류/설명)
```

예상 총 분량: **~255~265줄**

---

## 범위

### 포함 (In Scope)
- 05-hooks.md 기존 내용 재편 (Section 순서 조정 포함)
- 06-advanced-hooks.md 신규 파일 생성
- 02-core.md로의 크로스 레퍼런스 링크 삽입

### 제외 (Out of Scope)
- 02-core.md 수정 (이미 완료된 파일)
- src/error.html 정적 폴백 설명 (너무 엣지케이스)
- React/Next.js 비교표 (전면 제거)
- HTTP 캐시 헤더 등 외부 시스템

---

## 엣지케이스 & 에러 처리

- Section 번호 재편 시 기존 03-routing.md 등에서 "Section X" 링크가 있으면 업데이트
- `sequence()` 코드를 1줄로만 줄일 때, 실제 사용법이 불명확하지 않도록 짧은 설명 추가

---

## 의존성

- **02-core.md**: Section 7 에러 처리 — 05-hooks.md Section 4에서 링크 걸기
- **docs/svelte/**: runes 관련 연결 없음 (hooks는 runes 독립)

---

## 전체 작업 순서

### Phase 1: 05-hooks.md 재편
1. Section 1 개요: React 비교표 제거
2. Section 2 handle: 실습 섹션 제거, handle vs middleware 비교표 제거, sequence 1줄로 압축
3. Section 3 locals: 잘못된/차선악 패턴 제거, 실습 섹션 제거, 전체 흐름 다이어그램 제거, React 비교표 제거
4. Section 6 에러 페이지: 02-core.md 링크로 대체 + 레이아웃 오류 특수 동작만 남김 → Section 4로 이동
5. Section 7 handleError → Section 5로 이동 (번호 조정)
6. 전체 훅 정리 표: 제거 (06 파일 말미로 이동)

### Phase 2: 06-advanced-hooks.md 신규 생성
7. resolve 옵션 이동 (원 Section 4): filterSerializedResponseHeaders 원리 설명 제거
8. handleFetch 이동 (원 Section 5): 그대로 유지
9. reroute 이동 (원 Section 8 일부): 다국어 예제 포함
10. transport 이동 (원 Section 8 일부): 핵심 압축
11. 전체 훅 정리 표 말미에 배치

---

## 인터뷰 로그

| 질문 | 결정 | 근거 |
|------|------|------|
| 목표 분량 | 2파일 분리 | 1170줄 과분량 |
| 분리 기준 | 기본(handle/locals/handleError) vs 고급(resolve/handleFetch/reroute/transport) | 학습 순서와 사용 빈도 |
| 실습 섹션 | 전체 제거 | 개념 설명 코드와 중복 |
| 잘못된 패턴 시퀀스 | 올바른 패턴만 유지 | 분량 과다, 혼란 유발 |
| React 비교표 | 모두 제거 | 다른 docs와 일관성 |
| filterSerializedResponseHeaders | 원리 설명 제거, 코드+표만 | 실무에서 드문 케이스 |
| transport | 핵심만 압축 | 드문 케이스 |
| sequence | 1줄 소개만 | 중요하지만 코드는 불필요 |
| locals TypeScript 타입 | 유지 | 타입 안전 사용에 중요 |
| handleFetch 사례 | 둘 다 유지 | 각각 다른 실무 문제 |
| reroute | 다국어 예제 포함 | 실용성 |
| 에러 페이지 | 02-core 링크 + 레이아웃 오류 특수 동작만 | 중복 최소화, 후크 특유 내용만 |
| locals vs 쿠키 비교표 | 유지 | 직관적 이해에 필수 |
| 전체 흐름 다이어그램 | 제거 | 실습 섹션과 함께 삭제 |
| 전체 훅 정리 표 | 06 말미로 이동 | 고급 파일의 마무리로 적합 |
