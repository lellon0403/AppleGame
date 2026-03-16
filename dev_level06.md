# Level 06 — 사과 제거 + 점수

## 목표
드래그를 놓을 때 박스 안 합이 정확히 10이면 해당 사과들을 제거하고 점수를 올린다.
제거된 사과는 화면에서 사라지고 점수가 화면에 표시된다.

## 완료 기준
- 합 10인 상태로 드래그를 놓으면 사과가 사라진다
- 사과 1개 제거 = 1점
- 점수가 화면에 실시간으로 표시된다
- 합이 10이 아니면 아무 일도 일어나지 않는다

---

## 1단계 — board.c에 제거 함수 추가

`board.h`에 선언 추가:

```c
/* board.h 에 추가 */

/*
 * 드래그 박스 안 사과를 제거하고 제거된 개수를 반환
 * 합이 10인 경우에만 호출해야 함
 */
int board_remove_in_rect(Board *board,
                         Rectangle rect,
                         int board_x, int board_y, int cell_size);
```

`board.c`에 구현 추가:

```c
/* board.c 에 추가 */
int board_remove_in_rect(Board *board,
                         Rectangle rect,
                         int board_x, int board_y, int cell_size)
{
    float rx0 = rect.x;
    float ry0 = rect.y;
    float rx1 = rect.x + rect.width;
    float ry1 = rect.y + rect.height;

    int removed_count = 0;

    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            if (board->grid[r][c].removed) continue;

            float cx = board_x + c * cell_size + cell_size / 2.0f;
            float cy = board_y + r * cell_size + cell_size / 2.0f;

            if (cx >= rx0 && cx <= rx1 && cy >= ry0 && cy <= ry1) {
                board->grid[r][c].removed = true;
                removed_count++;
            }
        }
    }
    return removed_count;
}
```

> `board_sum_in_rect`와 셀 포함 판단 로직이 완전히 동일해야 한다.
> 두 함수에서 다른 방식을 쓰면 "합은 10인데 제거가 다른 사과에 적용"되는 버그 발생.

---

## 2단계 — game.c에 try_remove 함수 추가

`game.h`에 선언 추가:

```c
/* game.h 에 추가 */
#include "render.h"   /* BOARD_X, BOARD_Y, CELL_SIZE 사용 */

/*
 * 드래그 해제 시 호출
 * 합이 10이면 제거 + 점수 획득
 */
void game_try_remove(Game *game);
```

`game.c`에 구현 추가:

```c
/* game.c 에 추가 */
#include "render.h"   /* 상수 */

void game_try_remove(Game *game)
{
    if (!game->drag.active) return;

    Rectangle rect = drag_to_rect(&game->drag);

    /* 합산 먼저 확인 */
    int sum = board_sum_in_rect(&game->board, rect,
                                BOARD_X, BOARD_Y, CELL_SIZE);

    if (sum == 10) {
        int removed = board_remove_in_rect(&game->board, rect,
                                           BOARD_X, BOARD_Y, CELL_SIZE);
        game->score += removed;
    }

    /* 드래그 종료 */
    game->drag.active = false;
}
```

---

## 3단계 — input.c 수정

지금까지 `drag_update`가 `IsMouseButtonReleased`일 때 `drag->active = false`를 직접 처리했다.
그런데 이제 "해제 시점"에 game_try_remove를 먼저 실행하고 active를 false로 바꿔야 한다.

`input.c`에서 released 처리를 제거한다:

```c
/* input.c 수정 */
void drag_update(DragBox *drag)
{
    if (IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
        drag->start  = GetMousePosition();
        drag->end    = drag->start;
        drag->active = true;
    }

    if (drag->active && IsMouseButtonDown(MOUSE_LEFT_BUTTON)) {
        drag->end = GetMousePosition();
    }

    /* Released 처리는 main.c에서 game_try_remove 호출 후 처리 */
}
```

---

## 4단계 — main.c 수정

```c
/* main.c */
#include "raylib.h"
#include "game.h"
#include "render.h"
#include "input.h"
#include <stdlib.h>
#include <time.h>
#include <stdio.h>

int main(void)
{
    srand((unsigned int)time(NULL));

    InitWindow(1010, 600, "Fruit Box");
    SetTargetFPS(60);

    Game game;
    game_init(&game);

    while (!WindowShouldClose())
    {
        /* ── 입력 업데이트 ── */
        drag_update(&game.drag);

        /* 드래그 해제 시 제거 시도 */
        if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) {
            game_try_remove(&game);
        }

        /* 실시간 합산 */
        int current_sum = 0;
        if (game.drag.active) {
            Rectangle rect = drag_to_rect(&game.drag);
            current_sum = board_sum_in_rect(&game.board, rect,
                                            BOARD_X, BOARD_Y, CELL_SIZE);
        }

        /* ── 렌더링 ── */
        BeginDrawing();
            ClearBackground((Color){ 30, 30, 30, 255 });

            draw_board(&game.board, game.thinColor);
            draw_drag_box(&game.drag, current_sum);
            draw_score(&game.score);

        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

---

## 5단계 — render.c에 점수 표시 추가

`render.h`에 선언 추가:

```c
/* render.h 에 추가 */
void draw_score(int score);
```

`render.c`에 구현 추가:

```c
/* render.c 에 추가 */
void draw_score(int score)
{
    char buf[32];
    sprintf(buf, "SCORE: %d", score);
    DrawText(buf, BOARD_X, BOARD_Y + ROWS * CELL_SIZE + 8, 22, WHITE);
}
```

`render.c`에 `#include <stdio.h>` 추가 (sprintf 사용)

---

## 6단계 — 빌드 및 확인

`F5` 실행 후:

1. 합이 10이 되는 조합을 드래그 (예: `1`과 `9`)
2. 박스가 빨간색으로 변하는지 확인
3. 마우스를 놓으면 두 사과가 **사라지는지** 확인
4. 하단 점수가 **+2** 올라가는지 확인
5. 합이 10이 아닌 상태로 놓으면 **아무것도 안 사라지는지** 확인

---

## 전체 데이터 흐름 정리

```
[마우스 누름]
    → drag.start = 현재 마우스 위치
    → drag.active = true

[드래그 중]
    → drag.end = 현재 마우스 위치
    → board_sum_in_rect() 로 실시간 합 계산
    → draw_drag_box(sum) 으로 색상 변경

[마우스 놓음]
    → game_try_remove() 호출
        → board_sum_in_rect() 로 합 재확인
        → 합 == 10이면 board_remove_in_rect() 호출
        → score += 제거된 사과 수
    → drag.active = false
```

---

## 자주 발생하는 오류

| 현상 | 원인 | 해결 |
|------|------|------|
| 사과가 안 사라짐 | game_try_remove 미호출 | IsMouseButtonReleased 블록 확인 |
| 합이 10인데 제거 안됨 | sum/remove 판단 기준 불일치 | board_sum_in_rect와 board_remove_in_rect의 cx,cy 조건 동일한지 확인 |
| 점수가 1씩 올라감 | removed_count 미사용 | score += removed (개수만큼) |
| 제거 후 drag.active가 true | game_try_remove에서 active=false 누락 | game.drag.active = false 확인 |

---

## 다음 단계
Level 06 완료 후 → [dev_level07.md] 타이머 + 게임오버로 진행
