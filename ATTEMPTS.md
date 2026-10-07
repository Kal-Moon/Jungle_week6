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
