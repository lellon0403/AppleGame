# Level 07 — 타이머 + 게임오버

## 목표
2분(120초) 카운트다운 타이머를 구현하고, 시간이 끝나면 게임오버 화면으로 전환한다.
올 클리어(사과 전부 제거) 시에도 게임오버 화면으로 전환하고 소요 시간을 표시한다.

## 완료 기준
- 우측에 세로 타이머 게이지가 표시된다
- 남은 시간이 숫자로 표시된다
- 120초 후 게임오버 화면이 표시된다
- 올 클리어 시 즉시 게임오버 화면으로 전환되고 "ALL CLEAR!" 메시지가 표시된다
- R키를 누르면 게임이 재시작된다

---

## 1단계 — timer.h / timer.c 생성

```c
/* timer.h */
#ifndef TIMER_H
#define TIMER_H

#include "game.h"

/* 타이머 업데이트 (매 프레임 호출) */
void timer_update(Game *game);

/* 올 클리어 여부 체크 */
void check_all_cleared(Game *game);

#endif /* TIMER_H */
```

```c
/* timer.c */
#include "timer.h"
#include "board.h"
#include "raylib.h"

void timer_update(Game *game)
{
    if (game->state != STATE_PLAYING) return;

    game->timeLeft -= GetFrameTime();   /* 이전 프레임 경과 시간만큼 감소 */

    if (game->timeLeft <= 0.0f) {
        game->timeLeft = 0.0f;
        game->state    = STATE_GAMEOVER;
    }
}

void check_all_cleared(Game *game)
{
    if (game->state != STATE_PLAYING) return;

    /* 제거되지 않은 사과가 하나라도 있으면 미클리어 */
    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            if (!game->board.grid[r][c].removed) return;
        }
    }

    /* 전부 제거됨 → 올 클리어 */
    game->allCleared = true;
    game->clearTime  = 120.0f - game->timeLeft;
    game->state      = STATE_GAMEOVER;
}
```

> **코드 설명**
> - `if (game->state != STATE_PLAYING) return;` : 메뉴 화면이나 게임오버 상태에서는 타이머가 동작하지 않도록 보호. 이 줄이 없으면 게임오버 화면에서도 타이머가 계속 줄어든다
> - `GetFrameTime()` : Raylib 함수. **이전 프레임을 처리하는 데 걸린 실제 시간(초)**를 반환. `SetTargetFPS(60)`이면 약 0.0167초. 이 값을 빼면 프레임 수와 무관하게 실제 시간 기반으로 카운트다운된다. `game->timeLeft -= 1`처럼 고정값을 빼면 60FPS에서는 60배 빠르게 줄어드는 버그 발생
> - `game->timeLeft = 0.0f` : 0 이하가 되면 강제로 0으로 고정. 음수가 되지 않게 보호
> - **check_all_cleared 로직**: 모든 170개 사과를 이중 for문으로 순회. `removed`가 false인 사과가 하나라도 있으면 즉시 `return`(미클리어). 함수 끝까지 도달한다는 것 = 전부 removed = 올 클리어
> - `game->clearTime = 120.0f - game->timeLeft` : 소요 시간 = 전체 시간 - 남은 시간. 게임오버 화면에서 "X초 만에 클리어"로 표시

---

## 2단계 — render.c에 타이머 + 게임오버 화면 추가

`render.h`에 선언 추가:

```c
/* render.h 에 추가 */
void draw_timer_bar(float time_left, float time_total);
void draw_gameover(const Game *game);
```

`render.c`에 구현 추가:

```c
/* render.c 에 추가 */
#include "game.h"
#include <stdio.h>

/* 우측 세로 타이머 게이지 */
void draw_timer_bar(float time_left, float time_total)
{
    /* 게이지 배경 (회색) */
    int bar_x      = 960;
    int bar_y      = BOARD_Y;
    int bar_width  = 22;
    int bar_height = ROWS * CELL_SIZE;   /* 540px */

    DrawRectangle(bar_x, bar_y, bar_width, bar_height,
                  (Color){ 60, 60, 60, 255 });

    /* 게이지 채우기 */
    float ratio  = time_left / time_total;
    if (ratio < 0.0f) ratio = 0.0f;
    int   filled = (int)(bar_height * ratio);

    /* 시간에 따라 색상 변경: 충분 → 초록, 절반 → 노랑, 위험 → 빨강 */
    Color bar_color;
    if (ratio > 0.5f)       bar_color = GREEN;
    else if (ratio > 0.25f) bar_color = YELLOW;
    else                    bar_color = RED;

    DrawRectangle(bar_x,
                  bar_y + (bar_height - filled),
                  bar_width,
                  filled,
                  bar_color);

    /* 게이지 테두리 */
    DrawRectangleLines(bar_x, bar_y, bar_width, bar_height, GRAY);

    /* 남은 시간 숫자 표시 */
    char buf[8];
    sprintf(buf, "%d", (int)time_left + 1);   /* 올림 표시 */
    int tw = MeasureText(buf, 16);
    DrawText(buf,
             bar_x + bar_width / 2 - tw / 2,
             bar_y + bar_height + 6,
             16, WHITE);
}

/* 게임오버 / 올 클리어 화면 */
void draw_gameover(const Game *game)
{
    /* 반투명 오버레이 */
    DrawRectangle(0, 0, 1010, 600, (Color){ 0, 0, 0, 170 });

    if (game->allCleared) {
        /* 올 클리어 */
        DrawText("ALL CLEAR!",
                 1010 / 2 - MeasureText("ALL CLEAR!", 60) / 2,
                 160, 60, GOLD);

        char time_buf[48];
        sprintf(time_buf, "클리어 시간: %.2f 초", game->clearTime);
        DrawText(time_buf,
                 1010 / 2 - MeasureText(time_buf, 28) / 2,
                 250, 28, YELLOW);
    } else {
        /* 시간 초과 */
        DrawText("TIME OVER",
                 1010 / 2 - MeasureText("TIME OVER", 60) / 2,
                 160, 60, RED);
    }

    /* 점수 */
    char score_buf[32];
    sprintf(score_buf, "SCORE: %d / 170", game->score);
    DrawText(score_buf,
             1010 / 2 - MeasureText(score_buf, 32) / 2,
             310, 32, WHITE);

    /* 재시작 안내 */
    DrawText("[ R ] 재시작",
             1010 / 2 - MeasureText("[ R ] 재시작", 24) / 2,
             400, 24, LIGHTGRAY);
}
```

> **코드 설명**
> - `float ratio = time_left / time_total` : 0.0(시간 없음) ~ 1.0(가득 참) 비율 계산. `time_total`은 120.0f로 전달
> - `if (ratio < 0.0f) ratio = 0.0f;` : 혹시 음수가 되는 경우 방어. 게이지 크기를 음수로 그릴 수 없다
> - `int filled = (int)(bar_height * ratio)` : 비율에 따른 픽셀 높이. ratio=1.0이면 540px, ratio=0.5이면 270px
> - `bar_y + (bar_height - filled)` : 게이지가 **아래에서 위로** 줄어드는 효과. 전체 게이지 높이에서 채워진 높이를 빼면 "빈 공간 높이"가 되고, 그만큼 y를 내려서 시작
> - `ratio > 0.5f` → 초록, `ratio > 0.25f` → 노랑, 그 외 → 빨강: 시간 경과에 따라 시각적으로 긴박감 전달
> - `(int)time_left + 1` : 남은 시간을 올림으로 표시. 59.9초면 60으로 표시. 0.0초가 되기 직전에 "0"이 표시되지 않도록 함
> - `DrawRectangle(0, 0, 1010, 600, (Color){ 0, 0, 0, 170 })` : 화면 전체를 알파 170(약 67% 불투명)의 검정으로 덮어 반투명 오버레이 효과. 게임판이 흐릿하게 비쳐 보인다
> - `1010 / 2 - MeasureText("ALL CLEAR!", 60) / 2` : 창 너비의 중앙에서 텍스트 너비의 절반을 빼면 정중앙 x 좌표

---

## 3단계 — main.c 수정

```c
/* main.c */
#include "raylib.h"
#include "game.h"
#include "render.h"
#include "input.h"
#include "timer.h"
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
        /* ── 업데이트 ── */
        switch (game.state)
        {
            case STATE_PLAYING:
                drag_update(&game.drag);

                if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) {
                    game_try_remove(&game);
                }

                timer_update(&game);
                check_all_cleared(&game);
                break;

            case STATE_GAMEOVER:
                if (IsKeyPressed(KEY_R)) {
                    game_init(&game);
                }
                break;

            default:
                break;
        }

        /* 실시간 합산 (PLAYING 중에만) */
        int current_sum = 0;
        if (game.state == STATE_PLAYING && game.drag.active) {
            Rectangle rect = drag_to_rect(&game.drag);
            current_sum = board_sum_in_rect(&game.board, rect,
                                            BOARD_X, BOARD_Y, CELL_SIZE);
        }

        /* ── 렌더링 ── */
        BeginDrawing();
            ClearBackground((Color){ 30, 30, 30, 255 });

            draw_board(&game.board, game.thinColor);
            draw_drag_box(&game.drag, current_sum);
            draw_score(game.score);
            draw_timer_bar(game.timeLeft, 120.0f);

            if (game.state == STATE_GAMEOVER) {
                draw_gameover(&game);
            }

        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

> **코드 설명**
> - `switch (game.state)` : 상태별로 다른 업데이트 로직을 실행. if-else if 연속보다 가독성이 좋고 빠뜨린 상태를 `default`로 처리할 수 있다
> - `case STATE_PLAYING:` 블록에만 `drag_update`, `game_try_remove`, `timer_update`, `check_all_cleared`가 있다. 게임오버 상태에서는 이 함수들이 실행되지 않는다
> - `case STATE_GAMEOVER:` : R키만 감지. `IsKeyPressed(KEY_R)` 는 R을 방금 누른 첫 프레임에만 true
> - **렌더링은 switch 밖**: 게임판, 드래그 박스, 점수, 타이머는 항상 그리고, 게임오버 상태일 때 추가로 오버레이를 그리는 방식. STATE_PLAYING과 STATE_GAMEOVER 화면이 분리되지 않고 자연스럽게 겹쳐진다

---

## 4단계 — 빌드 및 확인

`F5` 실행 후:

### 타이머 확인
- 우측 세로 게이지가 서서히 줄어든다
- 초록 → 노랑(60초 이하) → 빨강(30초 이하) 으로 색이 바뀐다
- 게이지 아래 숫자가 카운트다운된다

### 게임오버 확인
- 120초가 지나면 반투명 검은 화면 위에 "TIME OVER"와 점수가 표시된다
- R키를 누르면 새 판이 시작된다

### 올 클리어 확인 (테스트용)
빠른 테스트를 위해 `game_init`에서 timeLeft를 크게 늘리거나,
보드를 아주 적은 사과로 만들어 올 클리어 후 화면 확인.

---

## 타이머 빠른 테스트 방법

개발 중 2분을 기다리지 않으려면 game.c의 game_init에서 테스트용으로:

```c
game->timeLeft = 5.0f;   /* 5초로 단축 */
```

확인 후 반드시 `120.0f`로 복원한다.

---

## 자주 발생하는 오류

| 현상 | 원인 | 해결 |
|------|------|------|
| 타이머가 게임 시작 전부터 줄어듦 | state 체크 누락 | timer_update에서 STATE_PLAYING 확인 |
| 올 클리어가 감지 안됨 | check_all_cleared 미호출 | main 루프에서 호출 확인 |
| 게임오버 후 드래그 계속됨 | state별 업데이트 분기 누락 | switch(game.state) 블록 확인 |
| draw_gameover에서 game 구조체 접근 불가 | 헤더 순환 포함 | render.h에서 game.h include 순서 확인 |

---

## 다음 단계
Level 07 완료 후 → [dev_level08.md] 메뉴 화면으로 진행
