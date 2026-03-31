# 스프레드시트 데모 — 세분화된 반응성 확인

`$state` 깊은 반응성이 실제로 **셀 단위**로만 업데이트되는 것을 확인하는 실습.

---

## 핵심 원리

`$state` 배열은 Proxy이므로 `data[row][col].value = 'hello'` 한 줄만 바꿔도 **그 프로퍼티에 의존하는 곳만** 재렌더링된다. 나머지 셀은 건드리지 않는다.

이것이 React와 근본적으로 다른 점이다. React에서는 `data` 객체가 바뀌면 해당 컴포넌트와 자식이 모두 리렌더링된다 (memo 없이는). Svelte는 프로퍼티 단위로 추적하므로 memo가 필요 없다.

---

## 데이터 구조

```ts
// 행의 배열 → 각 행은 셀의 배열
type Cell = { value?: string; backgroundColor?: string; color?: string }
const data: Cell[][] = [
  [{ value: 'hello' }, { value: '42' }],
  [{ value: '=SUM(A1,B1)' }],
]
```

빈 셀은 `undefined`. sparse 배열이므로 접근 시 존재 여부 체크 필요.

---

## 컴포넌트 핵심 패턴

### Props — $bindable 사용

Sheet 컴포넌트가 `data`를 직접 변경하므로 `$bindable` 필수.

```ts
let { data = $bindable([[]]) }: { data: Cell[][] } = $props()
```

부모에서는 `<Sheet bind:data={myData} />`로 사용.

### 행/열 수 — $derived

```js
let numRows = $derived(Math.max(data.length, 10))
let numCols = $derived.by(() => Math.max(...data.map(r => r.length), 10))
```

### 안전한 셀 수정 — sparse 배열 대응

```ts
function setCellProp(
  row: number, col: number,
  prop: 'value' | 'backgroundColor' | 'color',
  value: string
) {
  if (!data[row]) data[row] = []
  if (!data[row][col]) data[row][col] = { [prop]: value }
  else data[row][col][prop] = value
}
```

행/셀이 없으면 동적으로 생성. `$state` Proxy 덕분에 이 변경이 자동으로 반응성을 갖는다.

---

## 수식 처리

셀 값이 `=`으로 시작하면 수식. `=SUM(B3,B4)`, `=MULTIPLY(A1,B1)` 형태.

```ts
function parseValue(value: string | undefined): string | number {
  if (!value || !value.startsWith('=')) return value ?? ''

  const funcName = value.split('(')[0].substring(1)
  const args = /* 셀 주소 파싱 → 각 셀의 값 조회 (수식이면 재귀) */

  return args.reduce((prev, curr) => {
    if (funcName === 'SUM') return prev + curr
    if (funcName === 'MULTIPLY') return prev * curr
    return 0
  }, funcName === 'MULTIPLY' ? 1 : 0)
}
```

**세분화된 반응성 연결**: `parseValue`가 `data[row][col].value`를 읽으므로, `$derived` 안에서 호출하면 참조된 셀 값이 바뀔 때만 재계산된다.

---

## 반응성 확인 포인트

1. 셀 하나 수정 → DevTools Elements에서 해당 `<td>`만 깜빡임 확인
2. 수식 셀(`=SUM(A1,B1)`) → A1 값 변경 시 수식 셀만 재계산
3. 무관한 셀 변경 → 수식 셀 재계산 안 됨

이것이 `$state` Proxy + 세분화된 의존성 추적의 실제 동작이다.
