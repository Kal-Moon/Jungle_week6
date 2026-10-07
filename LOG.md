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

## 2026-10-07

### 완성한 함수
- #20 find_fit (first fit 방식)
- #23 place

### 남은 것
- #24 realloc (지금 들어 있는 것은 예전 naive 코드)

### mdriver 결과
make가 에러와 경고 없이 끝났고, 처음으로 mdriver를 실행했다.

`./mdriver -V` : 11개 중 9개 통과, 2개 실패

| 번호 | 트레이스 | valid | util | Kops |
| --- | --- | --- | --- | --- |
| 0 | amptjp | yes | 99% | 181 |
| 1 | cccp | yes | 99% | 149 |
| 2 | cp-decl | yes | 99% | 167 |
| 3 | expr | yes | 100% | 201 |
| 4 | coalescing | yes | 66% | 21214 |
| 5 | random | yes | 92% | 179 |
| 6 | random2 | yes | 92% | 205 |
| 7 | binary | yes | 55% | 24 |
| 8 | binary2 | yes | 51% | 40 |
| 9 | realloc | no | - | - |
| 10 | realloc2 | no | - | - |

- 9, 10번 에러: `mm_realloc did not preserve the data from old block`
- 실패한 트레이스가 있어서 전체 점수(Perf index)는 아직 나오지 않는다.

짧은 트레이스
- short1-bal.rep : valid yes, util 66%, 40 (util) + 40 (thru) = 80/100
- short2-bal.rep : valid yes, util 89%, 54 (util) + 40 (thru) = 94/100

### 막혔던 점과 해결 방법
| 막혔던 점 | 해결 방법 |
| --- | --- |
| find_fit이 블록 크기만 비교해서, 이미 할당된 블록도 고를 수 있었다 | 조건에 `!GET_ALLOC(HDRP(bp))`를 더해 "비어 있고 크기도 충분한" 블록만 고르게 했다 |
| 고쳤다고 생각했는데 make 결과가 그대로였다 | 에디터에서 저장이 안 된 상태였다. 파일 저장 시각을 확인하고 Ctrl+S 후 다시 make |
| place가 없어서 `undefined reference to 'place'` 링크 에러 | 원형만 있고 본체가 없었다. place 본체를 작성했다 |
| find_fit 원형이 두 번 적혀 있었다 | 중복된 한 줄을 지웠다 |

### 완성한 함수 (추가)
- #24 mm_realloc

### mdriver 결과 (mm_realloc 수정 후)
make가 에러와 경고 없이 끝났고, 11개 트레이스가 모두 통과해 처음으로 전체 점수가 나왔다.

| 번호 | 트레이스 | valid | util | Kops |
| --- | --- | --- | --- | --- |
| 0 | amptjp | yes | 99% | 450 |
| 1 | cccp | yes | 99% | 731 |
| 2 | cp-decl | yes | 99% | 445 |
| 3 | expr | yes | 100% | 535 |
| 4 | coalescing | yes | 66% | 123818 |
| 5 | random | yes | 92% | 428 |
| 6 | random2 | yes | 92% | 452 |
| 7 | binary | yes | 55% | 44 |
| 8 | binary2 | yes | 51% | 57 |
| 9 | realloc | yes | 27% | 106 |
| 10 | realloc2 | yes | 34% | 4217 |
| 합계 | | | 74% | 125 |

- Perf index = 44 (util) + 8 (thru) = 53/100
- short1-bal.rep : 40 (util) + 40 (thru) = 80/100
- short2-bal.rep : 54 (util) + 40 (thru) = 94/100
- Kops(속도)는 실행할 때마다 조금씩 달라진다.

### 막혔던 점과 해결 방법 (mm_realloc)
| 막혔던 점 | 해결 방법 |
| --- | --- |
| `mm_realloc did not preserve the data from old block` | 처음 받은 naive 코드가 블록 크기를 `SIZE_T_SIZE`(8바이트) 앞에서 읽고 있었다. 우리 블록은 크기를 헤더(4바이트 앞)에 저장하므로 읽는 위치가 틀렸다 |
| `SIZE_T_SIZE`를 `DSIZE`로만 바꿈 | 그 숫자는 "빼는 값"이 아니라 "얼마나 앞을 읽을지"였다. 8바이트 앞은 앞 블록의 풋터다. 읽기와 빼기를 두 단계로 나눠야 했다 |
| `oldptr` 자리에 `WSIZE`를 넣어서 segmentation fault | "어느 블록인지"가 사라져 엉뚱한 주소를 읽었다 |
| `copySize-DSIZE;` → `statement with no effect` 경고 | 계산만 하고 결과를 버리는 줄이었다. `copySize = copySize - DSIZE;`처럼 다시 대입해야 한다 |
| `HDRP`를 괄호 없이 쓰고, `SIZE_T_SIZE(HDRP)`처럼 숫자를 함수처럼 부름 | `HDRP(bp)`는 블록을 괄호 안에 넣어야 하는 매크로다 |
| `GET_SIZE(HDRP(bp))` 앞에 예전 포인터 계산이 남아 괄호가 안 닫힘, `bp` 변수도 없음 | 예전 방식을 통째로 지우고 `mm_free` 첫 줄과 같은 모양으로, 이 함수의 변수 `ptr`을 넣었다 |

### 완성한 함수 (추가)
- #21 find_fit을 next fit 방식으로 변경 (전역 변수 `find_heap`, `mm_init`, `find_fit`, `coalesce` 수정)

### mdriver 결과 (next fit 적용 후)
make가 에러와 경고 없이 끝났고, 11개 트레이스가 모두 통과했다.

| 번호 | 트레이스 | valid | util | Kops |
| --- | --- | --- | --- | --- |
| 0 | amptjp | yes | 91% | 2407 |
| 1 | cccp | yes | 92% | 3344 |
| 2 | cp-decl | yes | 95% | 1560 |
| 3 | expr | yes | 97% | 924 |
| 4 | coalescing | yes | 66% | 113654 |
| 5 | random | yes | 91% | 791 |
| 6 | random2 | yes | 89% | 890 |
| 7 | binary | yes | 55% | 480 |
| 8 | binary2 | yes | 51% | 1441 |
| 9 | realloc | yes | 27% | 98 |
| 10 | realloc2 | yes | 45% | 3555 |
| 합계 | | | 73% | 513 |

- Perf index = 44 (util) + 34 (thru) = 78/100 (세 번 돌린 결과: 78, 74, 76. thru가 30~34점 사이에서 달라진다)
- short1-bal.rep : 40 (util) + 40 (thru) = 80/100
- short2-bal.rep : 54 (util) + 40 (thru) = 94/100

### 전후 비교 (first fit → next fit)
| | first fit | next fit |
| --- | --- | --- |
| util 점수 | 44 | 44 |
| thru 점수 | 8 | 30~34 |
| Perf index | 53/100 | 74~78/100 |
| 전체 util | 74% | 73% |
| 전체 처리량 | 125 Kops | 513 Kops |

### 막혔던 점과 해결 방법 (next fit)
시도별 코드는 ATTEMPTS.md에 있다.

| 막혔던 점 | 해결 방법 |
| --- | --- |
| 위치를 기억시키려고 `PUT(find_heap, 0)`을 씀 | `PUT`은 힙 안에 값을 쓰는 도구다. 변수에 기억시킬 때는 `find_heap = heap_listp;`처럼 `=`를 쓴다 |
| `find_heap(bp)`, `heap_listp(bp)`처럼 변수에 괄호를 붙임 | 괄호는 함수와 매크로에만 붙인다. 변수는 이름만 쓴다 |
| 멈춘 자리부터 끝까지만 찾아서 `Ran out of memory` | 못 찾으면 첫 칸부터 `find_heap` 직전까지 한 번 더 찾는 두 번째 for 문을 추가했다 |
| 두 번째 for 문 조건에서 크기와 주소를 비교 (`GET_SIZE(...) < find_heap`) → 무한 반복 | 위치는 위치와 비교한다: `bp < find_heap` |
| `Payload overlaps another payload`, segmentation fault | coalesce로 합쳐진 칸의 한가운데를 `find_heap`이 가리키고 있었다. 합친 뒤 `find_heap`이 그 칸 안쪽에 있으면 칸의 시작(`bp`)으로 옮긴다 |
| coalesce의 if 조건에서 부등호 방향이 반대 | 숫자를 넣어 확인했다 (bp=100, find_heap=150, 다음 칸=200) |

### 완성한 함수 (추가)
- #22 명시적 가용 리스트 (`mm-explicit.c`에 따로 구현. `mm.c`는 implicit + next fit 그대로)
  - 추가: 매크로 `PRED`, `SUCC`, 전역 변수 `free_listp`, 함수 `insert_free`, `remove_free`
  - 수정: `mm_init`, `coalesce`, `place`, `find_fit`
  - 삭제: next fit용 `find_heap`
  - 방식: 반납된 칸을 줄의 맨 앞에 넣는 LIFO, 찾기는 first fit

### mdriver 결과 (명시적 리스트, mm-explicit.c)
Makefile은 `mm.c`만 빌드하므로 `mm-explicit.c`는 따로 빌드해서 확인했다. 에러와 경고 없이 빌드됐고 11개 트레이스가 모두 통과했다.

| 번호 | 트레이스 | valid | util | Kops |
| --- | --- | --- | --- | --- |
| 0 | amptjp | yes | 89% | 19139 |
| 1 | cccp | yes | 92% | 30666 |
| 2 | cp-decl | yes | 94% | 15844 |
| 3 | expr | yes | 96% | 11384 |
| 4 | coalescing | yes | 66% | 75235 |
| 5 | random | yes | 88% | 4262 |
| 6 | random2 | yes | 85% | 6286 |
| 7 | binary | yes | 55% | 1759 |
| 8 | binary2 | yes | 51% | 3800 |
| 9 | realloc | yes | 26% | 72 |
| 10 | realloc2 | yes | 34% | 2875 |
| 합계 | | | 71% | 506 |

- Perf index = 42 (util) + 34 (thru) = 76/100 (세 번 돌린 결과: 76, 82, 76. thru가 34~40점 사이에서 달라진다)
- short1-bal.rep : 40 (util) + 40 (thru) = 80/100
- short2-bal.rep : 54 (util) + 40 (thru) = 94/100

### 세 방식 비교
| | first fit | next fit | explicit |
| --- | --- | --- | --- |
| util 점수 | 44 | 44 | 42 |
| thru 점수 | 8 | 30~34 | 34~40 |
| Perf index | 53 | 74~78 | 76~82 |
| 전체 util | 74% | 73% | 71% |

- 0~8번 트레이스는 next fit보다 3~8배 빨라졌다.
- 전체 처리량은 next fit과 비슷하다(506 vs 513 Kops). 전체 시간의 약 90%를 9번(realloc) 트레이스가 쓰기 때문이다. 병목은 find_fit이 아니라 mm_realloc이다.

### 막혔던 점과 해결 방법 (명시적 리스트)
시도별 코드와, 직접 쓴 부분/안내받은 부분의 구분은 ATTEMPTS.md 5절에 있다.

| 막혔던 점 | 해결 방법 |
| --- | --- |
| 앞/뒤 빈칸을 풋터와 다음 헤더로 찾으려 함 | 그건 바로 옆 칸이다. 다음 빈칸은 멀리 있을 수 있어서, 빈칸의 데이터 영역에 주소를 직접 적는다 |
| 최소 블록이 24바이트는 되어야 한다고 생각함 | 32비트에서는 주소가 4바이트라 헤더 4 + 앞 4 + 뒤 4 + 풋터 4 = 16으로 그대로다 |
| pred/succ를 지역 변수로 선언하려 함 | 변수가 아니라 빈칸마다 힙 안에 적혀 있는 값이다. 헤더처럼 매크로(`PRED`, `SUCC`)로 자리를 찾는다 |
| 자리(`SUCC(bp)`)와 그 자리에 적는 값을 혼동 | 자리는 `SUCC(bp)`, 값 읽기는 `(char *)GET(SUCC(bp))`, 값 쓰기는 `PUT(SUCC(bp), 값)` |
| place에서 `bp = NEXT_BLKP(bp)` 뒤에 remove_free를 둠 | 그 줄 뒤의 bp는 남은 조각이다. 원래 칸을 빼야 하므로 함수 맨 위에서 뺀다 |
| coalesce에서 NEXT_BLKP/PREV_BLKP를 지워야 하는지 고민 | 합치기는 옆 칸끼리만 하므로 그대로 둔다. 호출만 추가한다 |
| `remove_free;`, `NEXT_BLKP(bp) = NULL;`처럼 호출이 아닌 문장을 씀 | 함수는 `이름(넘길 값);`으로 부른다 |
| place만 고치고 실행해서 segmentation fault | coalesce가 줄에 넣지 않으면 줄에 없는 칸을 빼게 된다. 넣기와 빼기는 함께 맞춰야 한다 |
