# 시도 기록 (PPT용)

"확인해 줘"를 할 때마다 그 시점에 작성되어 있던 코드, 결과, 원인을 순서대로 적는다.
LOG.md가 커밋 단위의 요약이라면, 이 파일은 그 사이의 시행착오를 그대로 남긴 것이다.

- 줄 번호는 그 시점의 mm.c 기준이다.
- #14~#19 구간은 시도별 코드가 남아 있지 않아 커밋된 결과와 LOG.md의 메모만 적었다.

## 한눈에 보는 변경 내역

| 구분 | 대상 | 내용 | 커밋 |
| --- | --- | --- | --- |
| 생성 | 매크로 (WSIZE, DSIZE, CHUNKSIZE, MAX, PACK, GET, PUT, GET_SIZE, GET_ALLOC, HDRP, FTRP, NEXT_BLKP, PREV_BLKP) | #14 | 464e06e |
| 생성 | 전역 변수 `heap_listp`, 함수 원형들 | #15 | 464e06e |
| 수정 | `mm_init` (naive → 프롤로그/에필로그 생성) | #15 | 464e06e |
| 생성 | `extend_heap` | #16 | 464e06e |
| 수정 | `mm_free` (아무것도 안 함 → 헤더/풋터 변경 + coalesce) | #17 | 464e06e |
| 생성 | `coalesce` | #18 | 464e06e |
| 수정 | `mm_malloc` (brk 증가 → find_fit/place/extend_heap) | #19 | 464e06e |
| 생성 | `find_fit` (크기만 비교하는 초안) | #20 | 464e06e |
| 수정 | `find_fit` 조건에 `!GET_ALLOC(HDRP(bp))` 추가 | #20 | e2f9afe |
| 삭제 | 중복된 `find_fit` 원형 한 줄 | #20 | e2f9afe |
| 생성 | `place` | #23 | e2f9afe |
| 수정 | `mm_realloc`의 copySize 계산 두 줄 | #24 | 018989f |
| 생성 | 전역 변수 `find_heap` | #21 | 작업 중 |
| 수정 | `mm_init`에 `find_heap = heap_listp;` 추가 | #21 | 작업 중 |
| 수정 | `find_fit` (first fit → next fit) | #21 | 작업 중 |

남아 있는 예전(naive) 흔적: 매크로 `ALIGNMENT`, `ALIGN`, `SIZE_T_SIZE`(미사용), 파일 맨 위 주석, `mm_malloc`/`mm_free` 주석, 팀 정보.

## 점수 변화

| 시점 | 통과 | 점수 |
| --- | --- | --- |
| find_fit, place 완성 (e2f9afe) | 9/11 | 점수 없음 (realloc 2개 실패) |
| mm_realloc 수정 (018989f) | 11/11 | 44 (util) + 8 (thru) = 53/100 |
| next fit 작업 중 | - | 아래 기록 참고 |

---

## 1. find_fit (#20, first fit)

### 시도 1 — 크기만 비교
```c
for (bp = heap_listp; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
    if (GET_SIZE(HDRP(bp)) >= asize) {
        return bp;
    }
}
return NULL;
```
- 결과: 컴파일은 되지만 place가 없어 링크 에러로 실행 불가.
- 문제: 이미 할당된 블록도 크기만 맞으면 고른다.
- 덧붙임: 고쳤다고 생각하고 확인을 요청했지만 파일이 저장되지 않아 같은 코드가 두 번 확인됐다.

### 시도 2 — 할당 여부 확인 추가 (완성)
```c
if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize) {
```
- 결과: place까지 작성한 뒤 9/11 통과.

## 2. place (#23)

### 시도 1 — 한 번에 완성
```c
size_t csize = GET_SIZE(HDRP(bp));

if ((csize - asize) >= (2*DSIZE)) {
    PUT(HDRP(bp), PACK(asize, 1));
    PUT(FTRP(bp), PACK(asize, 1));
    bp = NEXT_BLKP(bp);
    PUT(HDRP(bp), PACK(csize-asize, 0));
    PUT(FTRP(bp), PACK(csize-asize, 0));
}
else {
    PUT(HDRP(bp), PACK(csize, 1));
    PUT(FTRP(bp), PACK(csize, 1));
}
```
- 결과: make 성공, 첫 mdriver 실행. 0~8번 통과, 9·10번(realloc) 실패.

## 3. mm_realloc (#24)

고친 곳은 copySize를 구하는 부분뿐이다. 나머지 줄은 처음 받은 코드 그대로다.

### 시작 상태 — 처음 받은 naive 코드
```c
copySize = *(size_t *)((char *)oldptr - SIZE_T_SIZE);
```
- 결과: `mm_realloc did not preserve the data from old block` (9, 10번 실패)
- 원인: 크기를 8바이트 앞에서 읽는다. 우리 블록의 헤더는 4바이트 앞에 있다.

### 시도 1 — SIZE_T_SIZE를 DSIZE로 교체
```c
copySize = *(size_t *)((char *)oldptr - DSIZE);
```
- 생각: 헤더 4 + 풋터 4 = 8바이트를 빼면 된다.
- 문제: 이 숫자는 "빼는 값"이 아니라 "몇 바이트 앞을 읽을지"다. 8바이트 앞은 앞 블록의 풋터다.
- 배운 점: 읽기와 빼기는 서로 다른 두 단계다.

### 시도 2 — 두 줄로 나눔
```c
copySize = *(size_t *)((char *)WSIZE - SIZE_T_SIZE);
copySize-DSIZE;
```
- 결과: 경고 `statement with no effect`, 실행 시 segmentation fault.
- 문제 1: `oldptr` 자리에 숫자 `WSIZE`를 넣어 "어느 블록인지"가 사라졌다. 주소 4 - 8을 읽는다.
- 문제 2: `copySize-DSIZE;`는 계산만 하고 결과를 버린다.

### 시도 3 — HDRP를 넣어 봄
```c
copySize = *(size_t *)((char *)oldptr - SIZE_T_SIZE(HDRP));
    copySize-DSIZE;
```
- 결과: 컴파일 에러 `'HDRP' undeclared`, `called object is not a function`.
- 문제: `HDRP`는 `HDRP(블록)`처럼 괄호 안에 블록을 넣어야 한다. `SIZE_T_SIZE`는 숫자라서 괄호를 붙일 수 없다.

### 시도 4 — GET_SIZE(HDRP(bp)) 등장
```c
copySize = *(size_t *)((char *)oldptr - GET_SIZE(HDRP(bp));
    copySize-DSIZE;
```
- 결과: 컴파일 에러 (괄호가 닫히지 않음, `bp` 없음).
- 문제: 예전 포인터 계산이 앞에 남아 있다. 이 함수에는 `bp`라는 변수가 없다.

### 시도 5 — 완성
```c
copySize = GET_SIZE(HDRP(ptr));
    copySize = copySize-DSIZE;
```
- 결과: 11/11 통과, 44 (util) + 8 (thru) = 53/100.
- 남은 점: `ptr == NULL`, `size == 0`인 경우는 다루지 않는다.

## 4. next fit (#21) — 작업 중

목표: 매번 첫 칸부터 찾던 것을, 지난번에 멈춘 자리부터 찾게 해서 thru 점수를 올린다.
시작 점수: 44 (util) + 8 (thru) = 53/100.

### 4-1. 위치를 기억할 변수 만들기

#### 시도 1 — PUT으로 값을 쓰려 함
```c
static char *find_haep;
...
    heap_listp += (2*WSIZE);
    PUT(find_heap, 0);
    PUT(find_haep + (heap_listp(2*WSIZE), PACK(DSIZE, 1));
```
- 결과: 컴파일 에러.
- 문제 1: 선언은 `find_haep`, 사용은 `find_heap`으로 철자가 다르다.
- 문제 2: `heap_listp(...)`는 변수를 함수처럼 부른 것이고 괄호도 닫히지 않았다.
- 문제 3: `PUT`은 힙 안에 값을 써넣는 도구다. 여기서 필요한 것은 변수에 위치를 기억시키는 것이다.

#### 시도 2 — 완성
```c
static char *find_heap;
...
    heap_listp += (2*WSIZE);
    find_heap = heap_listp;
```
- 결과: 11/11 통과, 53/100. 아직 아무도 이 변수를 쓰지 않아 점수는 그대로다.

### 4-2. find_fit이 그 변수를 쓰게 하기

#### 시도 1 — 시작 위치만 바꿈
```c
for (bp = find_heap; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
    if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize) {
        GET_SIZE(HDRP(bp));
        return bp;
    }
}
```
- 결과: 경고 `statement with no effect`, 52/100으로 그대로.
- 문제: `GET_SIZE(HDRP(bp));`는 읽기만 한다. `find_heap`이 한 번도 바뀌지 않아 여전히 첫 칸부터 찾는다.

#### 시도 2 — 변수를 함수처럼 부름
```c
find_heap = find_heap(bp);
```
- 결과: 컴파일 에러 `called object 'find_heap' is not a function`.

#### 시도 3 — 위치 기억 성공
```c
find_heap = bp;
return bp;
```
- 결과: 빌드 성공. 전체 실행은 segmentation fault.
- 트레이스별: binary 73/100 (33 util + **40 thru**), short1 80, short2 94 통과.
  random·realloc은 `Ran out of memory`, amptjp는 `Payload overlaps another payload`, cccp·coalescing은 강제 종료.
- 의미: 속도가 오른다는 것은 확인됐다. 대신 숨어 있던 문제 두 가지가 드러났다.
  - 문제 A: 멈춘 자리부터 끝까지만 찾고 포기한다. 앞쪽 빈칸을 다시 쓰지 못해 힙만 늘어난다.
  - 문제 B: 기억해 둔 칸이 coalesce로 앞 칸과 합쳐지면, `find_heap`이 큰 칸의 한가운데를 가리킨다.

### 4-3. 문제 A — 앞쪽 구간도 찾기

#### 시도 1 — 두 번째 for 문 추가, 부등호를 뒤집음
```c
for (bp = find_heap; GET_SIZE(HDRP(bp)) < 0; bp = NEXT_BLKP(bp)) {
    if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) <= asize) {
        return bp;
    }
}
```
- 결과: 빌드 성공, 실행 결과는 4-2 시도 3과 완전히 같음.
- 문제 1: 시작이 `find_heap`이라 이미 본 구간을 다시 본다.
- 문제 2: 크기는 음수가 될 수 없어 `< 0`은 항상 거짓이다. 이 for 문은 한 번도 돌지 않는다.
- 문제 3: `<= asize`는 필요한 것보다 작은 칸을 고른다.
- 문제 4: 찾았을 때 `find_heap = bp;`가 없다.

#### 시도 2 — 조건은 되돌리고 시작 위치를 find_fit으로
```c
for (bp = find_fit; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp)) {
    if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize) {
        find_heap = bp;
        return bp;
    }
}
```
- 결과: 경고 `assignment to 'char *' from incompatible pointer type`. 13개 트레이스 전부 segmentation fault (통과하던 short1, short2, binary도 실패).
- 나아진 점: 안쪽 `if`와 `find_heap = bp;`는 맞게 고쳐졌다.
- 문제 1: `find_fit`은 변수가 아니라 지금 이 함수의 이름이다. 힙의 칸이 아니라 함수의 코드가 있는 주소에서 찾기 시작한다.
- 문제 2: 멈추는 조건이 여전히 "힙 끝까지"다. `find_heap`에서 멈춰야 한다.

#### 시도 3 — 시작 위치는 고쳤지만 멈추는 조건을 지움
```c
for (bp = heap_listp; bp = NEXT_BLKP(bp)) {
    if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize) {
        find_heap = bp;
        return bp;
    }
}
```
- 결과: 컴파일 에러 `expected ';' before ')' token` (208번째 줄). 실행 파일이 새로 만들어지지 않음.
- 나아진 점: 시작 위치가 `heap_listp`로 맞게 고쳐졌다.
- 문제: for 문의 괄호 안은 `시작; 계속할 조건; 다음으로 이동` 세 칸인데, 가운데 칸(조건)을 통째로 지워 두 칸만 남았다. 컴파일러는 `bp = NEXT_BLKP(bp)`를 조건으로 읽는다.

#### 시도 4 — 조건을 다시 넣었지만 크기와 위치를 비교함
```c
for (bp = heap_listp; GET_SIZE(HDRP(bp)) < find_heap; bp = NEXT_BLKP(bp)) {
    if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize) {
        find_heap = bp;
        return bp;
    }
}
```
- 결과: 경고 `comparison between pointer and integer` (208번째 줄). 실행하면 끝나지 않고 멈춰 있음(무한 반복). 전체 실행과 short1, short2, amptjp, cccp, cp-decl, expr, coalescing이 모두 시간 제한에 걸려 중단됨(나머지는 돌리지 않음).
- 나아진 점: for 문의 세 칸이 다시 갖춰졌고, `<`와 `find_heap`을 쓴 방향도 맞다.
- 문제: 왼쪽이 `GET_SIZE(HDRP(bp))`, 즉 칸의 **크기**(몇십~몇천)다. 오른쪽 `find_heap`은 **위치**(주소, 수십억 단위의 숫자)다. 크기는 언제나 주소보다 작아서 조건이 항상 참이 된다.
- 왜 끝나지 않는가: 힙 끝 표시는 크기가 0이다. 거기서 `NEXT_BLKP(bp)`는 bp + 0, 즉 제자리다. 조건은 계속 참이라 같은 자리에서 영원히 돈다.

#### 시도 5 — 비교의 좌우를 바꿈
```c
for (bp = heap_listp; find_heap < GET_SIZE(HDRP(bp)); bp = NEXT_BLKP(bp)) {
```
- 결과: 같은 경고 `comparison between pointer and integer`. 무한 반복은 사라졌지만 4-2 시도 3과 같은 결과로 돌아감.
  - 통과: short1 80, short2 94, binary 73 (33 util + 40 thru), binary2 71 (31 util + 40 thru)
  - `Ran out of memory`: random, random2, realloc, realloc2
  - `Payload overlaps another payload`: amptjp
  - 강제 종료: cccp, cp-decl, expr, coalescing
- 문제: 여전히 위치(`find_heap`)와 크기(`GET_SIZE(...)`)를 비교한다. 이번에는 "주소 < 크기"라서 항상 거짓이 되고, 두 번째 for 문이 한 번도 돌지 않는다. 그래서 앞쪽 빈칸을 여전히 못 쓴다(문제 A 그대로).
- 배운 점: 좌우를 바꾸는 것으로는 해결되지 않는다. 비교하는 두 값의 종류가 같아야 한다(위치는 위치와).

#### 시도 6 — GET_SIZE를 빼고 heap_listp(bp)를 넣음
```c
for (bp = heap_listp; find_heap < heap_listp(bp); bp = NEXT_BLKP(bp)) {
```
- 결과: 컴파일 에러 `called object 'heap_listp' is not a function or function pointer` (208번째 줄). 실행 파일이 새로 만들어지지 않음.
- 나아진 점: 크기(`GET_SIZE`)를 비교에서 뺐다. 이제 위치끼리 비교하려는 모양이다.
- 문제 1: `heap_listp(bp)`는 변수를 함수처럼 부른 것이다. 괄호를 붙일 수 있는 것은 함수와 `HDRP(bp)` 같은 매크로뿐이다.
- 문제 2: 비교 대상이 `heap_listp`(첫 칸)이다. 첫 칸의 위치는 변하지 않으므로, 비교해야 하는 것은 한 칸씩 움직이는 `bp`다.
- 문제 3: 방향이 `find_heap < ...`이다. "bp가 find_heap보다 앞에 있는 동안"이려면 작은 쪽이 bp여야 한다.

#### 시도 7 — 부등호 방향만 바꿈
```c
for (bp = heap_listp; find_heap > heap_listp(bp); bp = NEXT_BLKP(bp)) {
```
- 결과: 같은 컴파일 에러 `called object 'heap_listp' is not a function or function pointer`.
- 나아진 점: 방향은 맞아졌다(`find_heap`이 큰 쪽).
- 문제: `heap_listp(bp)`가 그대로다. 변수에 괄호를 붙일 수 없고, 비교해야 하는 것은 움직이지 않는 `heap_listp`가 아니라 한 칸씩 움직이는 `bp`다.
- 여기서 조건의 답을 안내받음: `bp < find_heap` ("지금 살펴보는 칸이 지난번에 멈춘 칸보다 앞에 있는 동안").

#### 시도 8 — 문제 A 해결
```c
for (bp = heap_listp; bp < find_heap; bp = NEXT_BLKP(bp)) {
    if (!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize) {
        find_heap = bp;
        return bp;
    }
}
```
- 결과: make 에러·경고 없음. `Ran out of memory`가 모두 사라짐. 전체 실행(`./mdriver -V`)은 아직 강제 종료.
- 트레이스를 하나씩 따로 돌린 결과 (13개 중 9개 통과):

| 트레이스 | 결과 |
| --- | --- |
| short1 | 40 (util) + 40 (thru) = 80 |
| short2 | 54 + 40 = 94 |
| coalescing | 40 + 40 = 80 |
| random | 55 + 40 = 95 |
| random2 | 54 + 31 = 84 |
| binary | 33 + 28 = 61 |
| binary2 | 31 + 40 = 71 |
| realloc | 16 + 5 = 21 |
| realloc2 | 27 + 40 = 67 |
| amptjp, cp-decl, expr | `Payload overlaps another payload` |
| cccp | 강제 종료 |

- 의미: 앞쪽 빈칸을 다시 쓰게 되어 메모리 부족이 없어졌다. 남은 실패는 모두 문제 B(coalesce로 합쳐진 칸의 한가운데를 `find_heap`이 가리키는 경우)다.
- 주의: 통과한 트레이스도 문제 B가 우연히 드러나지 않았을 뿐일 수 있다.

### 4-4. 문제 B — coalesce 뒤에 find_heap 바로잡기

#### 시도 1 — return bp 앞에 if 추가
```c
    if (find_heap < bp && > NEXT_BLKP(bp)) {
        find_heap = bp;
    }

    return bp;
```
- 결과: 컴파일 에러 `expected expression before '>' token`, 경고 `comparison of distinct pointer types lacks a cast` (196번째 줄). 실행 파일이 새로 만들어지지 않음.
- 맞은 점: 위치(return 바로 앞), `if`와 `&&`를 쓴 구조, 안쪽의 `find_heap = bp;`.
- 고민했던 점: 조건이 위치 비교인지, find_fit에서 쓴 `!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize`인지. → 위치 비교가 맞다. find_fit의 조건은 "이 칸을 내줘도 되는가"를 묻는 것이고, 여기서는 "find_heap이 합쳐진 칸 안쪽에 있는가"를 묻는다. coalesce에는 `asize`도 없다.
- 문제 1: `&&` 뒤가 `> NEXT_BLKP(bp)`로 시작한다. 무엇을 비교하는지(왼쪽)가 빠졌다. `&&`의 양쪽은 각각 완전한 비교여야 한다.
- 문제 2: 부등호 방향이 둘 다 반대다. 지금은 "bp보다 앞이고 다음 칸보다 뒤"인데, 그런 위치는 없다.
- 문제 3 (경고): `find_heap`은 `char *`, 이 함수의 `bp`는 `void *`라서 종류가 다른 포인터를 비교한다는 경고가 난다.

#### 시도 2 — && 뒤는 완성했지만 앞쪽에 !GET_ALLOC을 넣음
```c
    if (find_heap < !GET_ALLOC(HDRP(bp)) && find_heap > NEXT_BLKP(bp)) {
        find_heap = bp;
    }
```
- 결과: 컴파일은 됨. 경고 `comparison between pointer and integer` (196번째 줄). 실행 결과는 그대로(전체 실행 강제 종료, amptjp·cp-decl·expr 겹침, cccp 강제 종료).
- 나아진 점: `&&` 뒤가 `find_heap > NEXT_BLKP(bp)`로 완전한 비교가 됐다.
- 문제 1: 앞쪽 비교의 오른쪽이 `!GET_ALLOC(HDRP(bp))`다. 이건 "비어 있으면 1, 아니면 0"이라는 숫자다. 위치(`find_heap`)가 0이나 1보다 작을 수는 없으니 항상 거짓이다.
- 문제 2: 뒤쪽 비교의 방향이 반대다. "find_heap이 다음 칸보다 뒤"는 합쳐진 칸의 바깥이다.
- 결과적으로 if 안이 한 번도 실행되지 않아, 고치기 전과 똑같이 동작한다.

#### 시도 3 — 비교 대상은 맞췄지만 방향이 둘 다 반대
```c
    if (find_heap < (char *)bp && find_heap > NEXT_BLKP(bp)) {
        find_heap = bp;
    }
```
- 결과: make 에러·경고 없음. 실행 결과는 그대로(전체 실행 강제 종료, amptjp·cp-decl·expr 겹침, cccp 강제 종료).
- 나아진 점: 비교 대상이 `bp`와 `NEXT_BLKP(bp)`로 맞아졌고, `(char *)`를 붙여 경고도 사라졌다.
- 문제: 부등호가 둘 다 반대다. 지금 조건은 "find_heap이 bp보다 앞이고, 동시에 다음 칸보다 뒤"다. bp는 다음 칸보다 앞에 있으므로 두 조건을 동시에 만족하는 위치는 없다. if 안이 한 번도 실행되지 않는다.
- 배운 점: 컴파일러는 문법만 본다. 경고가 없어도 조건의 뜻이 틀리면 조용히 아무 일도 하지 않는다.

#### 시도 4 — 완성
```c
    if (find_heap > (char *)bp && find_heap < NEXT_BLKP(bp)) {
        find_heap = bp;
    }
```
- 결과: make 에러·경고 없음. 11/11 통과. **44 (util) + 34 (thru) = 78/100**.
- 고친 것: 부등호 두 개의 방향. "find_heap이 bp보다 뒤이고 다음 칸보다 앞" = 합쳐진 칸의 안쪽.

### next fit 전후 비교

| 번호 | 트레이스 | first fit util | next fit util | first fit Kops | next fit Kops |
| --- | --- | --- | --- | --- | --- |
| 0 | amptjp | 99% | 91% | 450 | 2407 |
| 1 | cccp | 99% | 92% | 731 | 3344 |
| 2 | cp-decl | 99% | 95% | 445 | 1560 |
| 3 | expr | 100% | 97% | 535 | 924 |
| 4 | coalescing | 66% | 66% | 123818 | 113654 |
| 5 | random | 92% | 91% | 428 | 791 |
| 6 | random2 | 92% | 89% | 452 | 890 |
| 7 | binary | 55% | 55% | 44 | 480 |
| 8 | binary2 | 51% | 51% | 57 | 1441 |
| 9 | realloc | 27% | 27% | 106 | 98 |
| 10 | realloc2 | 34% | 45% | 4217 | 3555 |
| 합계 | | 74% | 73% | 125 | 513 |

| | first fit (018989f) | next fit |
| --- | --- | --- |
| util 점수 | 44 | 44 |
| thru 점수 | 8 | 34 |
| Perf index | 53/100 | 78/100 |

- 속도: 전체 처리량이 125 → 513 Kops로 약 4배. binary 트레이스는 10~25배 빨라졌다.
- 효율: 0~3, 5, 6번 트레이스의 util이 1~8%p 내려갔다(빈칸을 앞에서부터 채우지 않아 조각이 퍼짐). 평균은 74% → 73%로 util 점수는 44점 그대로다.
- 9번(realloc)은 여전히 느리고(98 Kops) util도 27%다. thru가 40점 만점에 못 미치는 주된 이유다.
- Kops는 실행할 때마다 달라진다.

## 5. 명시적 가용 리스트 (#22) — 작업 중

`mm.c`(implicit + next fit, 커밋 9d07c80)를 `mm-explicit.c`로 복제해 따로 구현한다. 줄 번호는 `mm-explicit.c` 기준이다.
시작 점수: `mm.c`와 같은 코드라 44 (util) + 30~34 (thru) = 74~78/100.

설계 메모 (구현 전에 정리한 것)
- 빈칸끼리만 따로 줄을 세운다. 각 빈칸은 데이터 영역에 "앞 빈칸 주소"(bp 자리)와 "뒤 빈칸 주소"(bp + 4)를 적어 둔다.
- 처음 생각: 앞 빈칸은 풋터, 뒤 빈칸은 다음 헤더로 찾으면 된다 → 그건 바로 옆 칸을 찾는 방법(implicit)이다. 다음 빈칸은 멀리 떨어져 있을 수 있어 주소를 직접 적어야 한다.
- 처음 생각: 최소 블록이 24바이트는 되어야 한다 → 32비트에서는 주소가 4바이트라 헤더 4 + 앞 4 + 뒤 4 + 풋터 4 = 16으로 그대로다. 24는 64비트일 때다.
- 처음 생각: pred, succ를 find_fit의 지역 변수로 선언한다 → 변수가 아니라 빈칸마다 힙 안에 적혀 있는 값이다. 헤더처럼 매크로로 자리를 찾는다.
- 줄이 바뀌는 곳: 넣기는 mm_free, extend_heap, place(쪼개고 남은 부분), 빼기는 place(할당), coalesce(합칠 때).

### 5-1. 앞/뒤 빈칸 주소 매크로

#### 시도 1 — PREV_BLKP, NEXT_BLKP 모양을 따라 함
```c
#define pred(bp)    ((char *)(bp)) - GET_SIZE(((char *)(bp) - PREV_BLKP))
#define sicc(bp)    ((char *)(bp)) + GET_SIZE(((char *)(bp) + NEXT_BLKP))
```
- 결과: 컴파일 에러·경고 없음, 11/11 통과, 44 (util) + 25 (thru) = 69/100. 다만 이 매크로를 아직 아무 데서도 쓰지 않아서 검사되지 않은 것이다. 매크로는 쓰이는 순간에야 컴파일된다.
- 문제 1: `PREV_BLKP`, `NEXT_BLKP`는 `PREV_BLKP(bp)`처럼 괄호 안에 칸을 넣어야 하는 매크로다. 쓰는 순간 에러가 난다.
- 문제 2: `GET_SIZE`로 크기를 읽어 그만큼 건너뛰는 것은 "바로 옆 칸"으로 가는 계산이다. 여기서 필요한 것은 같은 칸 안에서 주소가 적힌 자리(bp, bp + 4)다. 크기는 필요 없다.
- 사소한 점: `sicc`는 succ(successor)의 오타로 보인다.

#### 시도 2 — HDRP, FTRP를 빼고 더함
```c
#define pred(bp)    ((char *)(bp)) - HDRP(bp)
#define sicc(bp)    ((char *)(bp)) + FTRP(bp)
```
- 결과: 컴파일 에러·경고 없음, 11/11 통과 (44 util + 10 thru = 54/100, thru는 측정 오차). 여전히 매크로를 쓰는 곳이 없어 검사되지 않은 상태.
- 나아진 점: `GET_SIZE`와 옆 칸 매크로를 뺐고, 괄호 안에 `bp`를 넣었다.
- 문제: `HDRP(bp)`, `FTRP(bp)`는 "거리"가 아니라 그 자체로 "위치(주소)"다. 위치에서 위치를 빼면 두 지점 사이의 거리(숫자)가 나오고, 위치에 위치를 더하는 것은 C에서 허용되지 않는다. 필요한 것은 bp에서 고정된 거리(0, WSIZE)만큼 떨어진 자리다.
- 여기서 답을 안내받음:
```c
#define PRED(bp)    ((char *)(bp))
#define SUCC(bp)    ((char *)(bp) + WSIZE)
```

#### 시도 3 — 완성 (답을 반영)
```c
#define PRED(bp)    ((char *)(bp))
#define SUCC(bp)    ((char *)(bp) + WSIZE)
```
- 결과: 아래 5-2와 함께 확인. 에러·경고 없음.

### 5-2. 줄의 맨 앞을 기억하는 전역 변수

#### 시도 1 — 한 번에 완성
```c
static char *free_listp;
...
    free_listp = NULL;      /* mm_init 안, extend_heap을 부르기 전 */
```
- 결과: 에러·경고 없음, 11/11 통과, 44 (util) + 29 (thru) = 73/100. 아직 매크로와 변수를 쓰는 곳이 없어 동작은 next fit 그대로다.
- 정한 것: 이름은 `heap_listp`와 짝을 맞춰 `free_listp`. 처음 값은 "빈칸이 하나도 없다"는 뜻의 `NULL`. extend_heap이 첫 빈칸을 줄에 넣게 되므로 그 전에 초기화한다.

### 5-3. "줄에 넣기" 함수 (insert_free)

구현 전 생각 (질문에 대한 처음 답 → 결론)
- bp의 "뒤" 자리에 적을 값: `SUCC` → 그건 자리 이름이다. 적을 값은 원래 맨 앞이던 칸의 위치(`free_listp`).
- bp의 "앞" 자리에 적을 값: `PRED+WSIZE` → 맨 앞이라 앞에 아무도 없으니 `NULL`.
- free_listp의 새 값: 빈 리스트 → 넣은 뒤에는 `bp`가 맨 앞이다.
- 줄이 비어 있을 때: `continue`로 지나간다 → `continue`는 반복문 안에서만 쓴다. "있을 때만 한다"는 `if`다.
- 배운 점: 자리(`SUCC(bp)`)와 그 자리에 적는 값은 다르다.

#### 시도 1 — 요청해서 코드를 안내받음
```c
static void insert_free(void *bp);      /* 원형 */

static void insert_free(void *bp)
{
    PUT(SUCC(bp), (unsigned int)free_listp);
    PUT(PRED(bp), (unsigned int)NULL);
    if (free_listp != NULL) {
        PUT(PRED(free_listp), (unsigned int)bp);
    }
    free_listp = bp;
}
```
- 이 함수는 직접 쓰지 않고 안내받은 코드를 넣었다.
- 순서가 중요하다: `free_listp = bp;`가 마지막이어야 원래 맨 앞 칸의 위치를 잃지 않는다.

### 5-4. "줄에서 빼기"

#### 시도 1 — insert_free 안에 이어서 씀
```c
    free_listp = bp;

    if (PRED(bp) < bp; bp < SUCC(bp)){
        PUT(PRED(bp) + SUCC(bp));
    }
}
```
- 결과: 컴파일 에러 (`macro "PUT" requires 2 arguments, but only 1 given` 등, 253~254번째 줄). 실행 파일이 만들어지지 않음.
- 문제 1: 빼기는 넣기와 반대되는 일이라 별도의 함수여야 한다. insert_free 안에 두면 넣자마자 빼게 된다.
- 문제 2: `if`의 괄호 안에는 조건 하나만 들어간다. 세미콜론으로 나누는 것은 `for`다. 두 조건을 잇는 것은 `&&`다.
- 문제 3: `PUT`은 `PUT(자리, 값)` 두 개가 필요하다. `PRED(bp) + SUCC(bp)`는 자리 두 개를 더한 것이라 뜻이 없다.
- 문제 4: `PRED(bp) < bp`는 자리끼리의 비교다(`PRED(bp)`는 bp와 같은 주소). 앞 칸이 있는지 알려면 그 자리에 적힌 값을 `GET`으로 읽어 `NULL`과 비교해야 한다.

#### 시도 2 — 별도 함수로 분리, 앞쪽 조건만 시도
```c
static void remove_free(void *bp)
{
     if ((char *)PRED < bp) {
        PUT(PRED(bp) = free_listp
    }
}
```
- 결과: 컴파일 에러 (`'PUT' undeclared`, `expected ';' at end of input` 등, 257~258번째 줄). 실행 파일이 만들어지지 않음.
- 나아진 점: `insert_free`에서 꺼내 별도 함수 `remove_free`로 만들었다. 한 번에 하나(앞쪽)만 시도한 것도 좋은 접근이다.
- 문제 1: `PRED`를 괄호 없이 썼다. `PRED(bp)`처럼 칸을 넣어야 한다.
- 문제 2: 조건이 위치의 크기 비교(`<`)다. 알고 싶은 것은 "앞 칸이 있는가"이고, 그건 `bp`의 "앞" 자리에 적힌 값이 `NULL`인지로 판단한다.
- 문제 3: `PUT(PRED(bp) = free_listp`는 `PUT(자리, 값)`의 쉼표 자리에 `=`를 썼고, 닫는 괄호와 세미콜론이 없다.
- 문제 4: 고치는 대상이 `bp`의 "앞" 자리다. `bp`는 줄에서 빠지는 칸이라 고칠 필요가 없다. 고쳐야 하는 것은 앞 칸의 "뒤" 자리다.
- 빠진 것: `remove_free`의 원형이 파일 위쪽에 없다.

#### 시도 3 — 지역 변수와 if/else 틀을 만듦
```c
static void remove_free(void *bp)
{
    char *prev;
    char *next;

     if (bp != NULL) {
        GET_SIZE(PRED(next));
    }
    else{
        PUT(SUCC(insert_free), (unsigned int)bp);
    }
}
```
- 결과: 컴파일은 됨. 경고 `statement with no effect`(261번째 줄), `unused variable 'prev'`. 함수를 아직 부르지 않아 점수는 그대로(44 util + 24 thru = 68/100).
- 나아진 점: 지역 변수 `prev`, `next` 선언, `if { } else { }` 틀, `!= NULL` 비교, `PUT(자리, 값)` 문법이 모두 갖춰졌다.
- 문제 1: `prev`와 `next`를 선언만 하고 값을 넣지 않았다. 값이 없는 변수를 `PRED(next)`에 쓰면 엉뚱한 주소를 읽는다.
- 문제 2: 조건이 `bp != NULL`이다. `bp`는 빼려는 칸 자신이라 항상 있다. 물어야 하는 것은 앞 칸(`prev`)이 있는가다.
- 문제 3: `GET_SIZE(PRED(next));`는 읽기만 하고 버린다. 또 `GET_SIZE`는 아래 3비트를 지우는 크기 전용 매크로라 주소를 읽을 때는 `GET`을 쓴다.
- 문제 4: `SUCC(insert_free)`의 `insert_free`는 함수 이름이다(find_fit 때 `bp = find_fit`과 같은 실수). 칸의 위치가 들어 있는 변수를 넣어야 한다.
- 문제 5: if와 else의 내용이 서로 바뀐 모양이다. `PUT`으로 이웃 칸을 고치는 것은 앞 칸이 "있을 때"다.
- 빠진 것: `remove_free`의 원형이 아직 없다.

#### 시도 4 — 안내받은 줄을 변형해 넣음
```c
static void remove_free(void *bp)
{
    char *prev;
    char *next;

     if (free_listp != NULL) {
        next = (char *)PUT(PRED(bp));
    }
    else{
        bp = free_listp;
    }
}
```
- 결과: 컴파일 에러 (`'PUT' undeclared`, 261번째 줄 — `PUT`에 값 하나만 줌). 실행 파일이 만들어지지 않음.
- 문제 1: 값을 꺼내는 줄이 `if` 안에 들어갔고, `GET` 대신 `PUT`을 썼다. `PUT`은 쓰기, `GET`은 읽기다. 또 `PRED`(앞)에서 읽은 것을 `next`(뒤)에 담았다.
- 문제 2: 조건이 `free_listp != NULL`이다. 줄에 칸이 있는지를 묻는 것이고, 빼는 중이라면 항상 참이다. 물어야 하는 것은 `prev != NULL`이다.
- 문제 3: `bp = free_listp;`는 좌우가 반대다. 바뀌어야 하는 것은 `free_listp`이고, 새 값은 `bp`가 아니라 `next`다.
- 여기서 답을 안내받음 (아래 시도 5).

#### 시도 5 — 답을 안내받음
```c
static void remove_free(void *bp);      /* 원형 */

static void remove_free(void *bp)
{
    char *prev = (char *)GET(PRED(bp));
    char *next = (char *)GET(SUCC(bp));

    if (prev != NULL) {
        PUT(SUCC(prev), (unsigned int)next);
    }
    else {
        free_listp = next;
    }
    if (next != NULL) {
        PUT(PRED(next), (unsigned int)prev);
    }
}
```
- 이 함수는 직접 완성하지 못하고 안내받은 코드를 넣었다.
- 정리: ① 앞/뒤 칸의 위치를 먼저 꺼낸다. ② 앞 칸이 있으면 앞 칸의 "뒤"를 next로, 없으면(bp가 맨 앞) free_listp를 next로. ③ 뒤 칸이 있으면 뒤 칸의 "앞"을 prev로.

#### 시도 6 — 안내받은 코드를 반영
- 결과: 에러 없음. 경고는 `'insert_free' defined but not used`, `'remove_free' defined but not used` 2개(아직 부르는 곳이 없어서 정상). 11/11 통과, 44 (util) + 33 (thru) = 76/100.
- 상태: 매크로(PRED, SUCC), 전역 변수(free_listp), insert_free, remove_free까지 준비됨. 동작은 아직 next fit 그대로다.

### 5-5. place와 coalesce에서 넣기/빼기 부르기

구현 전 생각
- 넣기는 `PUT(FTRP(bp), PACK(csize-asize, 0));` 뒤 → 맞다. 남은 조각의 헤더·풋터가 다 적힌 뒤여야 한다.
- 빼기는 `bp = NEXT_BLKP(bp);` 이후 → 그 줄 이후의 `bp`는 남은 조각(줄에 들어간 적 없는 칸)이다. 빼야 하는 것은 원래 `bp`이고, else 쪽(통째로 주는 경우)에서도 빠져야 한다.

#### 시도 1 — place 맨 끝에 remove_free
```c
static void place(void *bp, size_t asize)
{
    size_t csize = GET_SIZE(HDRP(bp));

    if ((csize - asize) >= (2*DSIZE)) {
        PUT(HDRP(bp), PACK(asize, 0));
        PUT(FTRP(bp), PACK(asize, 0));
        bp = NEXT_BLKP(bp);

        PUT(HDRP(bp), PACK(csize-asize, 0));
        PUT(FTRP(bp), PACK(csize-asize, 0));
    }
    else {
        PUT(HDRP(bp), PACK(csize, 1));
        PUT(FTRP(bp), PACK(csize, 1));
    }
    remove_free(bp);
}
```
- 결과: 컴파일 에러 없음(경고 `'insert_free' defined but not used`). 실행하면 segmentation fault.
- 나아진 점: `remove_free(bp)`를 한 번만 써서 if/else 두 갈래를 모두 처리하려 했다.
- 문제 1: 위치가 맨 끝이다. if 쪽을 지나면 `bp`가 이미 남은 조각으로 바뀌어 있어, 줄에 없는 칸을 빼게 된다.
- 문제 2: 233~234번째 줄의 `PACK(asize, 1)`이 `PACK(asize, 0)`으로 바뀌었다. 할당한 칸에 "빈칸" 표시를 하게 된다(원래 코드는 1이었다).
- 문제 3: 남은 조각을 줄에 넣는 `insert_free`가 아직 없다.
- 참고: 이 단계에서는 place를 다 맞게 고쳐도 강제 종료된다. coalesce가 아직 빈칸을 줄에 넣지 않아서, 줄에 들어간 적 없는 칸을 빼게 되기 때문이다. place와 coalesce를 둘 다 고친 뒤에야 통과한다.

구현 전 생각 (coalesce)
- 처음 생각: 명시적 리스트니까 coalesce에서 `NEXT_BLKP`, `PREV_BLKP`를 지우면 되나 → 지우지 않는다. 합치기는 창고에서 바로 붙어 있는 칸끼리만 할 수 있어서 옆 칸을 찾는 매크로가 그대로 필요하다. 줄의 앞/뒤(`PRED`, `SUCC`)는 찾기용이다. 기존 줄은 그대로 두고 호출 6줄만 추가한다.

#### 시도 2 — 자리는 모두 맞고, 호출 대신 다른 문장을 씀
place
```c
    size_t csize = GET_SIZE(HDRP(bp));
    remove_free(bp);

    if ((csize - asize) >= (2*DSIZE)) {
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);

        PUT(HDRP(bp), PACK(csize-asize, 0));
        PUT(FTRP(bp), PACK(csize-asize, 0));

        insert_free;
    }
```
coalesce
```c
    if (prev_alloc && next_alloc) {            /* Case 1 */
        bp;
        return bp;
    }
    else if (prev_alloc && !next_alloc) {      /* Case 2 */
        NEXT_BLKP(bp) = NULL;
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        ...
    }
    else if (!prev_alloc && next_alloc) {      /* Case 3 */
         PREV_BLKP(bp) = NULL;
        ...
    }
    else {                                     /* Case 4 */
        PREV_BLKP(bp) = NULL;
        NEXT_BLKP(bp) = NULL;
        ...
    }
    if (find_heap > (char *)bp && find_heap < NEXT_BLKP(bp)) {
        find_heap = bp;
    }
    bp;
    return bp;
```
- 결과: 컴파일 에러 4개 `lvalue required as left operand of assignment` (183, 190, 198, 199번째 줄), 경고 `statement with no effect` (178, 210, 246번째 줄). 실행 파일이 만들어지지 않음.
- 맞은 점: place의 `remove_free(bp);` 위치(함수 맨 위)와 `PACK(asize, 1)` 복구. coalesce에 추가한 여섯 줄의 **위치**가 전부 맞다(Case 1의 return 앞, Case 2~4의 size 계산 앞, 맨 끝 return 앞).
- 문제 1: 넣어야 할 자리에 `bp;`, `insert_free;`만 적었다. 함수를 부르려면 `이름(넘길 값);` 모양이어야 한다.
- 문제 2: 빼야 할 자리에 `NEXT_BLKP(bp) = NULL;`을 적었다. `NEXT_BLKP(bp)`는 계산 결과(위치)이지 값을 넣을 수 있는 상자가 아니라서 `=`의 왼쪽에 올 수 없다. "줄에서 뺀다"는 `NULL`을 넣는 것이 아니라 `remove_free`를 부르는 것이다.

#### 시도 3 — 함수 이름만 적음
```c
    if (prev_alloc && next_alloc) {            /* Case 1 */
        insert_free;
        return bp;
    }
    else if (prev_alloc && !next_alloc) {      /* Case 2 */
        remove_free;
        ...
    }
    else if (!prev_alloc && next_alloc) {      /* Case 3 */
        remove_free;
        ...
    }
    else {                                     /* Case 4 */
        remove_free;
        remove_free;
        ...
    }
    ...
    insert_free;
    return bp;
```
(place의 `insert_free;`도 그대로)
- 결과: 컴파일 에러는 없지만 경고 `statement with no effect`가 7개(178, 183, 190, 198, 199, 211, 247번째 줄). 실행하면 segmentation fault.
- 나아진 점: 각 자리에 맞는 함수를 골랐다(넣을 곳에 insert_free, 뺄 곳에 remove_free).
- 문제: 괄호와 넘길 값이 없다. `remove_free;`는 함수를 부르는 것이 아니라 이름만 적은 것이라 아무 일도 하지 않는다. 그래서 줄에 아무것도 들어가지 않고, place의 `remove_free(bp);`가 줄에 없는 칸을 빼려다 죽는다.
- 들었던 의문: Case 4에서 remove_free를 두 번 쓸 필요가 있나 → 있다. 괄호 안이 비어서 같은 줄처럼 보였을 뿐, 빼야 하는 칸이 둘(앞 칸, 뒤 칸)이라 호출도 두 번이다.

#### 시도 4 — 대부분 호출로 바꿈, 세 줄이 남음
```c
178:        insert_free(bp);
183:        remove_free(NEXT_BLKP(bp));
190:        remove_free(PREV_BLKP(bp));
198:        remove_free;(PREV_BLKP(bp))
199:        remove_free;(NEXT_BLKP(bp))
211:    insert_free(bp);
237:    remove_free(bp);            /* place */
247:        insert_free;            /* place */
```
- 결과: 컴파일 에러 `expected ';' before ...` (198, 199번째 줄), 경고 `statement with no effect` (247번째 줄). 실행 파일이 만들어지지 않음.
- 맞은 점: 178, 183, 190, 211번째 줄은 올바른 호출이 됐다. 198, 199번째 줄도 넘길 칸(앞 칸, 뒤 칸)은 맞게 골랐다.
- 문제 1: 198, 199번째 줄에서 세미콜론이 이름 바로 뒤에 있다(`remove_free;(...)`). 세미콜론은 문장의 끝이라 `remove_free;`에서 문장이 끝나 버린다. 괄호 뒤로 옮겨야 한다.
- 문제 2: place의 247번째 줄이 아직 `insert_free;`다.

#### 시도 5 — 완성
```c
/* coalesce */
178:        insert_free(bp);                /* Case 1: return 앞 */
183:        remove_free(NEXT_BLKP(bp));     /* Case 2 */
190:        remove_free(PREV_BLKP(bp));     /* Case 3 */
198:        remove_free(PREV_BLKP(bp));     /* Case 4 */
199:        remove_free(NEXT_BLKP(bp));     /* Case 4 */
211:    insert_free(bp);                    /* 맨 끝 return 앞 */
/* place */
237:    remove_free(bp);                    /* 함수 맨 위 */
247:        insert_free(bp);                /* 남은 조각의 헤더·풋터를 쓴 뒤 */
```
- 결과: 에러·경고 없음. 11/11 통과, 44 (util) + 18 (thru) = 62/100. short1 80, short2 94.
- 의미: 빈칸 줄이 실제로 관리되기 시작했다(넣기 3곳, 빼기 5곳). 다만 find_fit은 아직 옛 방식(next fit, 옆 칸으로 건너가기)이라 줄을 쓰지 않는다. 그래서 util은 그대로이고, thru는 줄 관리 비용만 늘어 next fit 때보다 조금 낮게 나올 수 있다(측정 오차도 큼).

### 5-6. find_fit이 빈칸 줄만 따라가게 하기

#### 시도 1 — for 문의 세 칸을 바꿔 봄
```c
    for (bp = insert_free; free_listp(bp) > 0; bp = NEXT_BLKP(bp)) {
        if (GET_SIZE(HDRP(bp)) >= asize) {
            find_heap = bp;
            return bp;
        }
    }
    for (bp = heap_listp; bp < find_heap; bp = NEXT_BLKP(bp)) {
        if (GET_SIZE(HDRP(bp)) >= asize) {
            find_heap = bp;
            return bp;
        }
    }
    return NULL;
```
- 결과: 컴파일 에러 `called object 'free_listp' is not a function or function pointer`, 경고 `assignment to 'char *' from incompatible pointer type` (219번째 줄). 실행 파일이 만들어지지 않음.
- 맞은 점: `if`에서 `!GET_ALLOC(HDRP(bp))`를 뺐다. 줄에는 빈칸만 있으니 확인할 필요가 없다.
- 문제 1 (시작): `bp = insert_free`는 함수 이름을 넣은 것이다(next fit 때 `bp = find_fit`과 같은 실수). 줄의 맨 앞 칸의 위치는 변수 `free_listp`에 있다.
- 문제 2 (조건): `free_listp(bp) > 0`은 변수를 함수처럼 불렀다. 줄의 끝은 "다음 칸이 없음", 즉 `bp`가 `NULL`이 되는 때다.
- 문제 3 (이동): `bp = NEXT_BLKP(bp)`는 여전히 옆 칸으로 간다. 줄의 뒤 칸으로 가야 한다.
- 문제 4: 두 번째 for 문과 `find_heap = bp;`가 남아 있다. next fit용이라 지워야 한다.
- 들었던 의문: find_heap은 전부 지우는 것인가 → 그렇다. `mm-explicit.c`에서만 지운다(`mm.c`는 next fit 버전으로 남긴다).

#### 시도 2 — 맞는 재료를 다른 칸에 넣음 (find_heap은 Claude가 삭제)
```c
static void *find_fit(size_t asize)
{
    char *bp;

    for (bp = (char *)GET(SUCC(bp)); free_listp > 0; bp = NEXT_BLKP(bp)) {
        if (GET_SIZE(HDRP(bp)) >= asize) {
            return bp;
        }
    }
    return NULL;
}
```
- `find_heap` 관련 줄은 요청에 따라 Claude가 지웠다: 전역 변수 선언, mm_init의 `find_heap = heap_listp;`, coalesce 끝의 `if (find_heap > ...)` 블록, find_fit의 `find_heap = bp;`와 두 번째 for 문 전체. 그 밖의 코드는 손대지 않았다.
- 결과: 컴파일 에러 없음. 경고 `'bp' is used uninitialized` (213번째 줄). 실행하면 segmentation fault (전체, short1, short2 모두).
- 나아진 점: 줄의 뒤 칸을 꺼내는 표현 `(char *)GET(SUCC(bp))`와 `free_listp`를 가져왔다. 필요한 재료는 다 나왔다.
- 문제 1 (시작): `(char *)GET(SUCC(bp))`가 시작 칸에 들어갔다. 이건 "다음으로 이동"에 쓸 표현이다. 시작 시점에는 `bp`에 아직 아무 값도 없어서(경고의 내용) 엉뚱한 주소를 읽는다. 시작은 `free_listp`다.
- 문제 2 (조건): `free_listp > 0`은 반복 중에 변하지 않는 값을 보고 있다. 한 칸씩 움직이는 것은 `bp`이고, 줄의 끝은 `bp`가 `NULL`이 되는 때다.
- 문제 3 (이동): `bp = NEXT_BLKP(bp)`는 여전히 옆 칸으로 간다. 시작 칸에 넣은 표현이 여기 와야 한다.

#### 시도 3 — 완성 (for 문 한 줄은 답을 안내받음)
```c
static void *find_fit(size_t asize)
{
    char *bp;

    for (bp = free_listp; bp != NULL; bp = (char *)GET(SUCC(bp))) {
        if (GET_SIZE(HDRP(bp)) >= asize) {
            return bp;
        }
    }
    return NULL;
}
```
- 결과: 에러·경고 없음. 11/11 통과. 세 번 실행: **76, 82, 76 /100** (util 42 + thru 34~40). short1 80, short2 94.
- for 문의 세 칸: 시작은 줄의 맨 앞(`free_listp`), 조건은 `NULL`이 아닌 동안, 이동은 줄의 뒤 칸(`SUCC`에 적힌 값).

### 세 방식의 find_fit 비교

| 방식 | 시작 | 계속할 조건 | 다음으로 이동 |
| --- | --- | --- | --- |
| first fit (implicit) | `heap_listp` | 크기가 0이 아닌 동안 | 옆 칸 `NEXT_BLKP(bp)` |
| next fit (implicit) | `find_heap` | 크기가 0이 아닌 동안 | 옆 칸 `NEXT_BLKP(bp)` |
| explicit (LIFO, first fit) | `free_listp` | `NULL`이 아닌 동안 | 줄의 뒤 칸 `(char *)GET(SUCC(bp))` |

### 세 방식의 결과 비교

| 번호 | 트레이스 | first fit util / Kops | next fit util / Kops | explicit util / Kops |
| --- | --- | --- | --- | --- |
| 0 | amptjp | 99% / 450 | 91% / 2407 | 89% / 19139 |
| 1 | cccp | 99% / 731 | 92% / 3344 | 92% / 30666 |
| 2 | cp-decl | 99% / 445 | 95% / 1560 | 94% / 15844 |
| 3 | expr | 100% / 535 | 97% / 924 | 96% / 11384 |
| 4 | coalescing | 66% / 123818 | 66% / 113654 | 66% / 75235 |
| 5 | random | 92% / 428 | 91% / 791 | 88% / 4262 |
| 6 | random2 | 92% / 452 | 89% / 890 | 85% / 6286 |
| 7 | binary | 55% / 44 | 55% / 480 | 55% / 1759 |
| 8 | binary2 | 51% / 57 | 51% / 1441 | 51% / 3800 |
| 9 | realloc | 27% / 106 | 27% / 98 | 26% / 72 |
| 10 | realloc2 | 34% / 4217 | 45% / 3555 | 34% / 2875 |
| 합계 | | 74% / 125 | 73% / 513 | 71% / 506 |

| | first fit (mm.c 018989f) | next fit (mm.c 9d07c80) | explicit (mm-explicit.c) |
| --- | --- | --- | --- |
| util 점수 | 44 | 44 | 42 |
| thru 점수 | 8 | 30~34 | 34~40 |
| Perf index | 53 | 74~78 | 76~82 |

- 0~8번 트레이스는 next fit보다 3~8배 빨라졌다(사용 중인 칸을 건너가지 않고 빈칸만 밟는다).
- 그런데 전체 처리량(506 Kops)은 next fit(513 Kops)과 비슷하다. 전체 0.222초 중 0.200초를 9번(realloc) 트레이스 하나가 쓰기 때문이다. 9번은 72 Kops로 세 방식 모두에서 느리다. 병목이 find_fit이 아니라 mm_realloc(매번 새로 할당하고 복사)에 있다는 뜻이다.
- util은 73% → 71%로 조금 더 내려갔다. 반납된 칸을 줄의 맨 앞에 넣는(LIFO) 방식이라, 주소 순서와 상관없이 가장 최근에 반납된 칸부터 쓰게 되어 조각이 더 퍼진다.

### 5-7. find_fit을 best fit으로 바꿔 보기 (2026-10-08, 로컬 실험 — 커밋 안 함)

"명시적 리스트가 best fit이 아니었나?"를 확인하다가, 지금 코드가 first fit임을 알고 best fit으로 바꿔 비교했다.
시간이 없어 `find_fit`은 요청에 따라 Claude가 작성했다. GitHub의 `mm-explicit.c`(368ae61)는 first fit 그대로다.

#### 시도 1 — 끝까지 돌며 가장 작은 칸을 기억 (Claude 작성)
```c
static void *find_fit(size_t asize)
{
    char *bp;
    char *best = NULL;
    size_t best_size = 0;

    for (bp = free_listp; bp != NULL; bp = (char *)GET(SUCC(bp))) {
        size_t size = GET_SIZE(HDRP(bp));

        if (size < asize)
            continue;
        if (size == asize)
            return bp;
        if (best == NULL || size < best_size) {
            best = bp;
            best_size = size;
        }
    }
    return best;
}
```
- 결과: 에러·경고 없음. 11/11 통과. 한 번 실행: **63/100** (util 45 + thru 18). short1 util 66%, short2 util 89%.
- first fit과 다른 점: 맞는 칸을 만나도 바로 돌려주지 않고 `best`에 기억한 뒤 계속 돈다. 크기가 정확히 같을 때만 바로 돌려준다.

| 번호 | 트레이스 | explicit first fit util / Kops | explicit best fit util / Kops |
| --- | --- | --- | --- |
| 0 | amptjp | 89% / 19139 | 99% / 32025 |
| 1 | cccp | 92% / 30666 | 99% / 28922 |
| 2 | cp-decl | 94% / 15844 | 99% / 21648 |
| 3 | expr | 96% / 11384 | 100% / 18324 |
| 4 | coalescing | 66% / 75235 | 66% / 92249 |
| 5 | random | 88% / 4262 | 96% / 1198 |
| 6 | random2 | 85% / 6286 | 95% / 1322 |
| 7 | binary | 55% / 1759 | 55% / 319 |
| 8 | binary2 | 51% / 3800 | 51% / 139 |
| 9 | realloc | 26% / 72 | 31% / 73 |
| 10 | realloc2 | 34% / 2875 | 30% / 3048 |
| 합계 | | 71% / 506 | 75% / 267 |

| | explicit first fit (368ae61) | explicit best fit (로컬) |
| --- | --- | --- |
| util 점수 | 42 | 45 |
| thru 점수 | 34~40 | 18 |
| Perf index | 76~82 | 63 |

왜 점수가 내려갔나
- 얻은 것은 util 3점, 잃은 것은 thru 16~22점이다. 평균 util이 71% → 75%로 4%p 올랐는데, util 점수는 60점 × 평균 util이라 4%p는 약 3점에 그친다.
- 전체 시간이 0.222초 → 0.421초로 늘었다. 늘어난 0.2초가 거의 전부 7번(0.007초 → 0.038초)과 8번(0.006초 → 0.172초)이다.
- binary 트레이스는 작은 칸과 큰 칸을 번갈아 받은 뒤 한쪽만 반납해서, 합쳐지지 못한 빈칸이 줄에 수천 개 쌓인다. first fit은 맞는 칸을 만나면 멈추지만, best fit은 요청마다 그 줄을 끝까지 다 본다.
- 그런데 7번, 8번의 util은 55%, 51%로 그대로다. 이 트레이스의 낭비는 "어느 칸을 고르느냐"가 아니라 빈칸 사이에 사용 중인 칸이 끼어 합쳐지지 못하는 데서 오기 때문이다. 가장 느려진 곳에서 얻은 것이 없다.
- 정리: best fit은 util을 올리지만, 줄이 하나뿐이면 찾는 비용이 "빈칸 개수"만큼 든다. best fit이 손해 없이 쓰이려면 찾을 범위가 좁아야 한다(크기별로 줄을 나누는 segregated list).

### 5-8. 명시적 리스트에 next fit을 붙이면? (2026-10-08, 실험 — 프로젝트 파일에는 없음)

"명시적 리스트에 next fit으로 하면 어떻게 되나?"를 확인하려고 Claude가 `mm-explicit.c`의 복사본을 임시 폴더에 만들어 측정했다.
`mm-explicit.c` 자체에는 반영하지 않았다. 비교를 위해 커밋된 first fit(368ae61)도 같은 시점에 다시 돌렸다.

#### 시도 1 — 줄 안에서 "지난번에 멈춘 칸"을 기억 (Claude 작성)
```c
static char *rover;              /* 지난번에 멈춘 빈칸. mm_init에서 NULL로 초기화 */

static void *find_fit(size_t asize)
{
    char *bp;
    for (bp = rover; bp != NULL; bp = (char *)GET(SUCC(bp)))
        if (GET_SIZE(HDRP(bp)) >= asize) { rover = bp; return bp; }
    for (bp = free_listp; bp != rover; bp = (char *)GET(SUCC(bp)))
        if (GET_SIZE(HDRP(bp)) >= asize) { rover = bp; return bp; }
    return NULL;
}

/* remove_free 맨 앞에 추가: 기억한 칸이 줄에서 빠지면 그 뒤 칸으로 옮긴다 */
    if (rover == (char *)bp) rover = (char *)GET(SUCC(bp));
```
- 결과: 에러·경고 없음. 11/11 통과. 세 번 실행: **59, 59, 61 /100** (util 43 + thru 16~18).
- 같은 시점의 first fit: 71, 72, 77 /100 (util 42 + thru 29~34). 어제 기록(76~82)보다 조금 낮게 나왔다.
- implicit의 next fit(4절)과 구조는 같다: 기억한 자리부터 끝까지, 없으면 맨 앞부터 기억한 자리까지.

| 번호 | 트레이스 | explicit first fit util / Kops | explicit next fit util / Kops |
| --- | --- | --- | --- |
| 0 | amptjp | 89% / 7604~15764 | 94% / 20184~22550 |
| 1 | cccp | 92% / 19061~30175 | 93% / 29685~35208 |
| 2 | cp-decl | 94% / 9045~12506 | 95% / 19021~24087 |
| 3 | expr | 96% / 9313~12055 | 97% / 15215~25128 |
| 4 | coalescing | 66% / 53691~60555 | 66% / 46139~62718 |
| 5 | random | 88% / 2663~5861 | 90% / 3636~4483 |
| 6 | random2 | 85% / 4213~5130 | 88% / 3562~4169 |
| 7 | binary | 55% / 1189~1591 | 55% / 215~249 |
| 8 | binary2 | 51% / 3469~4246 | 51% / 124~159 |
| 9 | realloc | 26% / 61~74 | 25% / 57~69 |
| 10 | realloc2 | 34% / 2726~3350 | 34% / 3002~3268 |
| 합계 | | 71% / 432~513 | 72% / 236~266 |

명시적 리스트에서 세 가지 찾기 방식 비교

| | first fit (368ae61) | next fit (실험) | best fit (5-7, 로컬) |
| --- | --- | --- | --- |
| util 점수 | 42 | 43 | 45 |
| thru 점수 | 29~40 | 16~18 | 18 |
| Perf index | 71~82 | 59~61 | 63 |

왜 느려졌나
- implicit에서는 칸이 주소 순서라, 쪼개고 남은 빈칸이 "지난번에 멈춘 자리 바로 뒤"에 있었다. 그래서 next fit이 바로 맞는 칸을 찾았다.
- LIFO 명시적 리스트에서는 쪼개고 남은 빈칸이 줄의 맨 앞에 들어간다. 기억한 자리는 이미 줄의 중간으로 넘어가 있어서, 가장 쓸 만한 칸이 항상 등 뒤에 있다.
- 그래서 기억한 자리부터 줄 끝까지 다 보고, 맨 앞으로 돌아와서야 그 칸을 찾는다. 맞지 않는 빈칸이 수천 개 쌓이는 binary(7번, 8번)에서는 요청마다 줄을 거의 한 바퀴 돈다.
- LIFO에서 맨 앞부터 찾는 first fit은 "가장 최근에 생긴 빈칸부터 본다"는 뜻이라, next fit이 하려던 일을 이미 하고 있다.
- 정리: 측정한 세 방식 중 LIFO + first fit이 가장 높다. 찾기 방식의 좋고 나쁨은 리스트를 어떤 순서로 세우느냐에 따라 달라진다.
- 한계: 기억한 칸이 빠질 때 "그 뒤 칸으로 옮기는" 방식 하나만 실험했다.

### 명시적 리스트에서 직접 쓴 것과 안내받은 것

| 부분 | 누가 |
| --- | --- |
| 설계(자리, 최소 크기, 넣고 빼는 곳) | 질문에 답하며 정리. 처음 생각이 틀린 곳은 5절 설계 메모에 있음 |
| `PRED`, `SUCC` 매크로 | 두 번 시도 후 답을 안내받음 |
| `free_listp` 선언과 초기화 | 직접 작성 |
| `insert_free` | 코드를 안내받음 |
| `remove_free` | 네 번 시도 후 코드를 안내받음 |
| `place`, `coalesce`의 호출 8줄 | 직접 작성 (위치는 처음부터 맞았고, 호출 문법을 세 번 고침) |
| `find_fit`의 for 문 | 두 번 시도 후 한 줄을 안내받음 |
| `find_heap` 삭제 | 요청에 따라 Claude가 삭제 |
| `find_fit`의 best fit 버전 (5-7, 로컬 실험) | 요청에 따라 Claude가 작성 |
| 명시적 리스트의 next fit 실험 (5-8, 프로젝트 파일에는 없음) | Claude가 복사본에서 작성·측정 |
