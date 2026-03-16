# Level 02 — 게임판 초기화 + 터미널 출력

## 목표
게임판(17×10) 데이터 구조를 만들고, 사과 숫자를 랜덤 배치한 후 콘솔에 출력해 확인한다.
화면 렌더링 전에 로직이 올바른지 먼저 검증하는 단계다.

## 완료 기준
- 프로그램 실행 시 콘솔에 17×10 숫자 격자가 출력된다
- 격자의 모든 숫자 합이 10의 배수다
- 9의 개수가 1의 개수보다 많지 않다

---

## 1단계 — 파일 구조 만들기

솔루션 탐색기에서 아래 파일들을 추가한다.
(우클릭 → 추가 → 새 항목)

```
src/
├── main.c       ← Level 01에서 만든 파일 (수정)
├── board.h      ← 새로 생성
└── board.c      ← 새로 생성
```

> **구조 설명**
> - `board.h` : 데이터 구조(구조체)와 함수 선언만 담는 헤더 파일. 다른 .c 파일들이 `#include "board.h"`로 가져다 쓴다
> - `board.c` : board.h에서 선언한 함수들의 실제 구현이 담기는 파일
> - 이렇게 h/c를 분리하는 이유: 게임이 커지면 main.c 하나에 모든 코드를 넣기 어려워진다. 역할별로 파일을 나눠 관리하기 위함

---

## 2단계 — board.h 작성

```c
/* board.h */
#ifndef BOARD_H
#define BOARD_H

#include <stdbool.h>

#define COLS         17
#define ROWS         10
#define TOTAL_APPLES (COLS * ROWS)   /* 170 */

/* 사과 하나 */
typedef struct {
    int  value;    /* 숫자 1~9 */
    bool removed;  /* 제거 여부 */
} Apple;

/* 게임판 전체 */
typedef struct {
    Apple grid[ROWS][COLS];
    int   score;
} Board;

/* 함수 선언 */
void board_init(Board *board);
void board_print(const Board *board);   /* 디버그용 콘솔 출력 */
int  board_remaining(const Board *board);

#endif /* BOARD_H */
```

> **코드 설명**
> - `#ifndef BOARD_H` / `#define BOARD_H` / `#endif` : **헤더 가드**. 이 헤더를 여러 .c 파일에서 include하거나 같은 파일에서 실수로 두 번 include해도 내용이 중복 적용되지 않게 막는 안전장치. C언어의 관례적인 패턴
> - `#include <stdbool.h>` : C99에서 `bool`, `true`, `false`를 사용하려면 이 헤더가 필요. C++과 달리 C에는 bool이 기본 내장되어 있지 않다
> - `#define COLS 17` : 매크로 상수. 컴파일 전에 `COLS`를 만나면 모두 `17`로 치환. `const int`와 달리 메모리를 차지하지 않음. 나중에 게임판 크기를 바꾸고 싶을 때 이 한 줄만 수정하면 된다
> - `#define TOTAL_APPLES (COLS * ROWS)` : 다른 매크로를 사용해 계산. 괄호로 감싸는 것이 안전 — 매크로는 단순 치환이므로 예상치 못한 연산자 우선순위 문제가 생길 수 있다
> - `typedef struct { ... } Apple;` : 구조체에 이름표를 붙인다. 이후 `struct Apple` 대신 `Apple`만으로 사용 가능. C++과 달리 C에서는 typedef 없이는 항상 `struct` 키워드를 붙여야 한다
> - `bool removed;` : 사과가 제거됐는지 나타내는 플래그. 실제로 배열에서 지우는 대신 이 값을 `true`로 바꾸기만 한다 — 배열 요소를 삭제하고 앞으로 당기는 것보다 훨씬 간단
> - `Apple grid[ROWS][COLS]` : 2차원 배열. `grid[행][열]`로 접근. `grid[0][0]`이 좌상단, `grid[9][16]`이 우하단
> - 함수 선언(`.h`)과 구현(`.c`) 분리: 다른 파일에서 이 함수를 쓸 때 `.h`만 include하면 된다. 구현 세부사항을 몰라도 사용 가능

---

## 3단계 — board.c 작성

```c
/* board.c */
#include "board.h"
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

/* ───── 내부 헬퍼 ───── */

/* values 배열에서 특정 숫자의 개수를 센다 */
static int count_value(const int *values, int target)
{
    int cnt = 0;
    for (int i = 0; i < TOTAL_APPLES; i++)
        if (values[i] == target) cnt++;
    return cnt;
}

/* Fisher-Yates 셔플 */
static void shuffle(int *arr, int n)
{
    for (int i = n - 1; i > 0; i--) {
        int j   = rand() % (i + 1);
        int tmp = arr[i];
        arr[i]  = arr[j];
        arr[j]  = tmp;
    }
}

/* ───── 공개 함수 ───── */

void board_init(Board *board)
{
    int values[TOTAL_APPLES];
    int sum;

    /* 조건을 만족하는 배치가 나올 때까지 반복 생성 */
    do {
        sum = 0;
        for (int i = 0; i < TOTAL_APPLES; i++) {
            values[i] = (rand() % 9) + 1;   /* 1~9 */
            sum      += values[i];
        }
    } while (
        sum % 10 != 0                          /* 합이 10의 배수가 아님 */
        || count_value(values, 9) > count_value(values, 1)  /* 9 > 1 이면 클리어 불가 */
    );

    shuffle(values, TOTAL_APPLES);

    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            board->grid[r][c].value   = values[r * COLS + c];
            board->grid[r][c].removed = false;
        }
    }
    board->score = 0;
}

void board_print(const Board *board)
{
    int total = 0;
    int cnt1 = 0, cnt9 = 0;

    printf("===== 게임판 =====\n");
    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            int v = board->grid[r][c].value;
            printf("%d ", v);
            total += v;
            if (v == 1) cnt1++;
            if (v == 9) cnt9++;
        }
        printf("\n");
    }
    printf("==================\n");
    printf("합계: %d  (10의 배수: %s)\n", total, total % 10 == 0 ? "OK" : "NG");
    printf("1의 개수: %d  9의 개수: %d  (클리어 가능: %s)\n",
           cnt1, cnt9, cnt9 <= cnt1 ? "OK" : "NG");
}

int board_remaining(const Board *board)
{
    int cnt = 0;
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            if (!board->grid[r][c].removed) cnt++;
    return cnt;
}
```

> **코드 설명**
> - `static int count_value(...)` : `static`을 함수 앞에 붙이면 이 함수가 **board.c 파일 내부에서만** 쓸 수 있다. 외부에서 실수로 호출할 수 없게 보호. board.h에 선언하지 않은 이유도 같다
> - `rand() % 9 + 1` : `rand()`는 0~32767 범위의 정수 난수를 반환. `% 9`로 0~8 범위로 좁히고, `+1`로 1~9 범위로 이동. 이 패턴은 C에서 범위 난수를 만드는 기본 방법
> - `do { ... } while (조건)` : 일반 `while`과 달리, **무조건 한 번 실행한 후** 조건을 체크. 조건이 거짓이면 다시 반복. 여기서는 조건이 맞을 때까지 계속 새 숫자 배치를 만들어낸다. 보통 수십 번 안에 조건을 만족하는 배치가 나온다
> - `sum % 10 != 0` : 합을 10으로 나눈 나머지가 0이 아니면 재생성. `%`는 나머지 연산자. 이것이 원작 게임의 핵심 규칙 (research.md 참고)
> - `count_value(values, 9) > count_value(values, 1)` : 9는 1과만 짝을 이룰 수 있어서 9가 1보다 많으면 이론적으로 올 클리어 불가. research.md 4.1절 참고
> - **Fisher-Yates 셔플**: `i`를 끝에서 앞으로 이동하며, 0~i 범위 중 무작위 위치 `j`를 골라 `arr[i]`와 교환. 이 알고리즘은 모든 순열이 동일한 확률로 나오는 것이 수학적으로 증명됨. 단순히 여러 번 랜덤 교환하는 방식보다 훨씬 균등한 셔플
> - `values[r * COLS + c]` : 1차원 배열을 2차원 격자로 변환. r번째 행의 c번째 열 = `r × 열개수 + c`. 예: 3행 5열 = `3 * 17 + 5 = 56`번 인덱스
> - `board->grid[r][c].value` : board가 포인터이므로 `.` 대신 `->` 사용. `board->grid`는 `(*board).grid`와 동일

---

## 4단계 — main.c 수정

Level 01의 main.c에 초기화 코드를 추가한다.

```c
/* main.c */
#include "raylib.h"
#include "board.h"
#include <stdlib.h>
#include <time.h>

int main(void)
{
    /* 난수 시드 설정 — 프로그램 시작 시 1회 */
    srand((unsigned int)time(NULL));

    /* 게임판 초기화 및 콘솔 출력 */
    Board board;
    board_init(&board);
    board_print(&board);   /* 콘솔에서 확인 */

    /* 창 초기화 */
    InitWindow(1010, 600, "Fruit Box");
    SetTargetFPS(60);

    while (!WindowShouldClose())
    {
        BeginDrawing();
            ClearBackground((Color){ 30, 30, 30, 255 });
            DrawText("Level 02 - 콘솔 출력 확인", 20, 20, 24, WHITE);
        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

> **코드 설명**
> - `srand((unsigned int)time(NULL))` : 난수 생성기의 **시드(씨앗값)**를 설정. `time(NULL)`은 현재 시간(1970년 이후 경과 초)을 반환하므로 실행할 때마다 다른 씨앗값이 된다. 이 줄 없이 `rand()`를 쓰면 매번 같은 순서의 숫자가 나온다. 프로그램 시작 시 단 한 번만 호출해야 한다
> - `Board board;` : Board 구조체 변수를 스택에 선언. 아직 초기화 안 됨 (쓰레기값)
> - `board_init(&board)` : board의 주소를 넘겨서 함수 안에서 내용을 채운다. C에서 함수가 구조체를 수정하려면 포인터(`&`)로 전달해야 한다
> - `board_print(&board)` : 창이 열리기 **전에** 콘솔에 출력. 로직이 맞는지 눈으로 확인하는 디버그 용도

---

## 5단계 — 빌드 및 확인

1. `Ctrl+Shift+B` 로 빌드
2. `F5` 로 실행
3. Visual Studio 하단 **출력** 탭 또는 별도 콘솔 창에서 아래와 같은 출력 확인:

```
===== 게임판 =====
3 7 2 5 1 8 4 6 9 2 3 5 1 7 4 2 6
1 4 9 3 6 2 8 5 1 7 3 2 4 6 9 1 5
...
==================
합계: 850  (10의 배수: OK)
1의 개수: 19  9의 개수: 18  (클리어 가능: OK)
```

> 숫자는 랜덤이므로 매번 다르게 나온다. 합계와 클리어 가능 여부만 OK인지 확인한다.

---

## 콘솔 출력이 안 보일 때

Visual Studio에서 `F5`로 실행하면 콘솔이 바로 닫히는 경우가 있다.

해결 방법 1 — `Ctrl+F5` (디버깅 없이 시작) 사용
해결 방법 2 — `board_print` 직후에 아래 코드 추가:
```c
system("pause");   /* 키 입력 대기 */
```
해결 방법 3 — 프로젝트 속성 → 링커 → 시스템 → 서브시스템을 **콘솔** 로 설정

---

## 자주 발생하는 오류

| 오류 | 원인 | 해결 |
|------|------|------|
| `board.h: No such file` | 파일 경로 문제 | 프로젝트에 board.h 추가됐는지 확인 |
| 합계가 항상 같음 | srand 누락 | `srand((unsigned)time(NULL))` 추가 |
| 무한 루프 (창이 안 뜸) | do-while 조건이 너무 까다로움 | count_value 로직 재확인 |

---

## 다음 단계
Level 02 완료 후 → [dev_level03.md] 사과 렌더링으로 진행
