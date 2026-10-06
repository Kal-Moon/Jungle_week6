# Malloc Lab 작업 기록

## 2026-10-06

### 완성한 함수
- #14 매크로
- #15 mm_init
- #16 extend_heap
- #17 mm_free
- #18 coalesce
- #19 mm_malloc

### 남은 것
- #20 find_fit
- #23 place

### mdriver 결과
- 아직 실행 불가 (통과한 트레이스, util, 점수 없음)
- 컴파일은 경고 없이 통과하지만, 링크 단계에서 에러가 나서 mdriver 실행 파일이 만들어지지 않는다.
  - `undefined reference to 'place'` : mm_malloc이 place를 부르는데 place 본체가 아직 없음
  - find_fit은 mm.c에 초안이 들어 있어 링크 에러에는 잡히지 않는다. 아직 검증 전이라 남은 것으로 둔다.

### 막혔던 점과 해결 방법
| 막혔던 점 | 해결 방법 |
| --- | --- |
| 함수 본체 첫 줄에 세미콜론을 붙여서 에러 | 세미콜론이 붙는 것은 원형이다. 원형은 파일 위쪽에 따로 한 줄로 적고, 본체에는 붙이지 않는다 |
| `heap_listp undeclared` | 전역 변수 `static char *heap_listp;` 선언이 빠져 있었다 |
| `implicit declaration` 경고 | 함수 원형을 사용 위치보다 위에 추가했다 |
| 저장하지 않고 make를 돌려서 수정 전 결과가 나옴 | make 전에 Ctrl+S로 저장한다 |
| mm_free 매개변수 이름이 ptr인데 책 코드는 bp | 이름을 맞췄다 |
