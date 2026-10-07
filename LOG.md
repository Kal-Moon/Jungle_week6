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

## 2026-10-07 (점수 개선 준비)

### 한 일
- 점수 개선을 시작하기 전에, GitHub에 올라간 기준 버전(`018989f`, 묵시적 리스트 + first fit)의 함수와 점수를 `presentation/` 폴더에 저장했다.
- 완성한 함수: 없음 (mm.c 변경 없음)

### mdriver 결과 (같은 코드, 클라우드 환경에서 3회)
- util 74%, 속도 214~227 Kops → Perf index 59~60/100
- 내 컴퓨터 결과(53/100)와 공간 점수(44)는 같고, 속도 점수만 8 → 14~15로 다르다.

### 막혔던 점과 해결 방법
| 막혔던 점 | 해결 방법 |
| --- | --- |
| 클라우드에서 `bits/libc-header-start.h: No such file or directory`로 빌드 실패 | `-m32`(32비트) 빌드에 필요한 `gcc-multilib`이 없었다. 설치 후 빌드 성공 |
| 같은 코드인데 점수가 53점과 59점으로 다르게 나옴 | 속도는 컴퓨터 성능에 따라 달라진다. 개선 전/후는 같은 컴퓨터에서 잰 숫자끼리 비교한다 |
