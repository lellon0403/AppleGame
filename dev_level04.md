# Level 04 — 마우스 드래그 박스 표시

## 목표
마우스 왼쪽 버튼을 누른 채 드래그하면 화면에 반투명 사각형 박스가 그려진다.
아직 사과 제거는 하지 않으며, 박스가 시각적으로 올바르게 표시되는 것만 확인한다.

## 완료 기준
- 마우스 클릭+드래그 시 사각형이 그려진다
- 드래그를 놓으면 박스가 사라진다
- 어느 방향으로 드래그해도 박스가 올바르게 그려진다 (우→좌, 아래→위 포함)

---

## 1단계 — 파일 추가

```
src/
├── main.c
├── board.h / board.c
├── render.h / render.c
├── input.h      ← 새로 생성
└── input.c      ← 새로 생성
```

---

## 2단계 — input.h 작성

```c
/* input.h */
#ifndef INPUT_H
#define INPUT_H

#include "raylib.h"
#include <stdbool.h>

/* 드래그 상태 */
typedef struct {
    Vector2 start;    /* 드래그 시작 좌표 */
    Vector2 end;      /* 드래그 현재 끝 좌표 */
    bool    active;   /* 현재 드래그 중이면 true */
} DragBox;

/* 드래그 상태 업데이트 (매 프레임 호출) */
void drag_update(DragBox *drag);

/* 드래그 박스를 정규화된 Rectangle로 반환 (좌상단 기준) */
Rectangle drag_to_rect(const DragBox *drag);

#endif /* INPUT_H */
```

---

## 3단계 — input.c 작성

```c
/* input.c */
#include "input.h"

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

    if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) {
        drag->active = false;
    }
}

Rectangle drag_to_rect(const DragBox *drag)
{
    /* start와 end 중 어느 쪽이 더 작은지 모르므로 정규화 */
    float x = drag->start.x < drag->end.x ? drag->start.x : drag->end.x;
    float y = drag->start.y < drag->end.y ? drag->start.y : drag->end.y;
    float w = drag->start.x < drag->end.x
              ? drag->end.x - drag->start.x
              : drag->start.x - drag->end.x;
    float h = drag->start.y < drag->end.y
              ? drag->end.y - drag->start.y
              : drag->start.y - drag->end.y;

    return (Rectangle){ x, y, w, h };
}
```

---

## 4단계 — render.c에 드래그 박스 렌더링 추가

`render.h`에 함수 선언 추가:

```c
/* render.h 에 추가 */
#include "input.h"

/* 드래그 박스 그리기 */
void draw_drag_box(const DragBox *drag, int sum);
```

`render.c`에 함수 구현 추가:

```c
/* render.c 에 추가 */
void draw_drag_box(const DragBox *drag, int sum)
{
    if (!drag->active) return;

    Rectangle rect = drag_to_rect(drag);

    /* 합이 10이면 빨간색, 아니면 흰색 */
    Color border_color = (sum == 10) ? RED : WHITE;

    /* 반투명 내부 채우기 */
    DrawRectangleRec(rect, Fade(border_color, 0.12f));

    /* 테두리 */
    DrawRectangleLinesEx(rect, 2.0f, border_color);
}
```

---

## 5단계 — game.h / game.c 신규 생성

게임 전체 상태를 하나의 구조체로 관리하기 위해 game.h를 도입한다.

```c
/* game.h */
#ifndef GAME_H
#define GAME_H

#include "board.h"
#include "input.h"
#include <stdbool.h>

typedef enum {
    STATE_MENU,
    STATE_PLAYING,
    STATE_GAMEOVER
} GameState;

typedef struct {
    Board     board;
    DragBox   drag;
    float     timeLeft;    /* 남은 시간(초) */
    int       score;
    GameState state;
    bool      thinColor;   /* 연한 색 모드 */
    bool      allCleared;
    float     clearTime;
} Game;

void game_init(Game *game);

#endif /* GAME_H */
```

```c
/* game.c */
#include "game.h"
#include <stdlib.h>
#include <time.h>

void game_init(Game *game)
{
    board_init(&game->board);
    game->drag.active = false;
    game->timeLeft    = 120.0f;
    game->score       = 0;
    game->state       = STATE_PLAYING;
    game->thinColor   = false;
    game->allCleared  = false;
    game->clearTime   = 0.0f;
}
```

---

## 6단계 — main.c 수정

```c
/* main.c */
#include "raylib.h"
#include "game.h"
#include "render.h"
#include "input.h"
#include <stdlib.h>
#include <time.h>

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
        drag_update(&game.drag);

        /* ── 렌더링 ── */
        BeginDrawing();
            ClearBackground((Color){ 30, 30, 30, 255 });

            draw_board(&game.board, game.thinColor);
            draw_drag_box(&game.drag, 0);   /* sum은 다음 단계에서 계산 */

            DrawText("Level 04 - 드래그 박스", 20, 580, 14, GRAY);
        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

---

## 7단계 — 빌드 및 확인

`F5`로 실행 후:
1. 게임판 위 아무 곳이나 클릭+드래그 → 흰색 반투명 사각형이 그려진다
2. 마우스 버튼을 놓으면 박스가 사라진다
3. 오른쪽→왼쪽, 아래→위 방향으로 드래그해도 박스가 제대로 그려진다

---

## 드래그 방향별 동작 확인

| 드래그 방향 | start | end | 기대 동작 |
|------------|-------|-----|----------|
| 좌→우, 위→아래 | (100,100) | (300,200) | 정상 |
| 우→좌, 위→아래 | (300,100) | (100,200) | `drag_to_rect`가 보정해야 함 |
| 좌→우, 아래→위 | (100,200) | (300,100) | `drag_to_rect`가 보정해야 함 |
| 우→좌, 아래→위 | (300,200) | (100,100) | `drag_to_rect`가 보정해야 함 |

`drag_to_rect`에서 `fminf` / 너비·높이 절댓값 계산이 이를 처리한다.

---

## 자주 발생하는 오류

| 현상 | 원인 | 해결 |
|------|------|------|
| 드래그 시 박스가 뒤집힘 | drag_to_rect 정규화 누락 | input.c의 x, y, w, h 계산 재확인 |
| 클릭만 해도 박스가 남음 | released 처리 누락 | `IsMouseButtonReleased` 코드 확인 |
| 박스가 사과 위에 안 보임 | 렌더링 순서 문제 | `draw_drag_box`를 `draw_board` 이후에 호출 |

---

## 다음 단계
Level 04 완료 후 → [dev_level05.md] 합산 로직 + 박스 색상 변경으로 진행
