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
