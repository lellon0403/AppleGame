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

> **코드 설명**
> - `Vector2` : Raylib가 정의한 `{float x, float y}` 구조체. 픽셀 좌표를 표현할 때 사용
> - `bool active` : 드래그 중인지 나타내는 플래그. 마우스를 누르면 `true`, 손을 떼면 `false`. 이 값이 `false`일 때는 박스를 그리지 않는다
> - `drag_to_rect` 를 별도 함수로 분리한 이유: 드래그는 어느 방향으로든 할 수 있어서, start가 end보다 오른쪽이나 아래에 있을 수 있다. 이를 항상 "좌상단 기준" 사각형으로 정규화하는 작업이 필요하기 때문

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

> **코드 설명**
> - `IsMouseButtonPressed` : 버튼을 **방금 누른 첫 번째 프레임**에만 `true`. 누르는 동안 계속 `true`가 아님
> - `IsMouseButtonDown` : 버튼을 **누르고 있는 동안 매 프레임** `true`. 드래그 중 좌표를 업데이트할 때 사용
> - `IsMouseButtonReleased` : 버튼에서 **손을 뗀 첫 번째 프레임**에만 `true`. Level 06에서 이 시점에 사과 제거를 시도한다
> - `GetMousePosition()` : 현재 마우스 커서의 픽셀 좌표를 `Vector2`로 반환. 창 좌측 상단이 (0, 0)
> - `drag->end = drag->start` (Pressed 시점): 드래그 시작 직후에는 start와 end가 같은 위치. 이렇게 하지 않으면 이전 드래그의 end 위치가 남아있다
> - **drag_to_rect 정규화**: 드래그를 오른쪽→왼쪽, 또는 아래→위로 하면 start.x > end.x 상황이 생긴다. `Rectangle`은 항상 좌상단 좌표와 양수 너비/높이가 필요하므로, 삼항 연산자(`? :`)로 작은 값을 x, y로 선택하고 두 값의 차이(절댓값)를 너비, 높이로 계산
> - `(Rectangle){ x, y, w, h }` : C99 복합 리터럴. 구조체를 즉석에서 값으로 만들어 반환

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

> **코드 설명**
> - `if (!drag->active) return;` : 드래그 중이 아니면 즉시 함수 종료. 박스를 그릴 필요가 없다
> - `(sum == 10) ? RED : WHITE` : 삼항 연산자. 합이 10이면 빨간색, 아니면 흰색. Level 04에서는 sum을 항상 0으로 넘기므로 항상 흰색. Level 05에서 실제 합산값을 넘긴다
> - `Fade(border_color, 0.12f)` : Raylib 함수. 색상의 알파값을 0.0(투명)~1.0(불투명) 비율로 조정. 0.12f = 12% 불투명 → 반투명 내부 채우기 효과
> - `DrawRectangleRec(rect, color)` : Rectangle 구조체를 받아 사각형을 채워 그린다
> - `DrawRectangleLinesEx(rect, 2.0f, color)` : Rectangle 구조체를 받아 두께 2px의 테두리만 그린다

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

> **코드 설명**
> - `typedef enum { STATE_MENU, STATE_PLAYING, STATE_GAMEOVER } GameState;` : **열거형**. 게임 상태에 이름을 붙인다. 내부적으로 STATE_MENU=0, STATE_PLAYING=1, STATE_GAMEOVER=2로 저장되지만, 코드에서 숫자 대신 이름을 쓸 수 있어 가독성이 좋다
> - `float timeLeft` : 초(秒) 단위 남은 시간. `120.0f`로 시작해서 매 프레임 `GetFrameTime()`만큼 감소 (Level 07)
> - 모든 게임 데이터를 `Game` 하나에 묶는 이유: 여러 함수 사이에 데이터를 전달할 때 인자가 많아지는 것을 방지. `game_try_remove(&game)` 한 줄로 모든 상태에 접근 가능

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

> **코드 설명**
> - `game->board` : Game 구조체 안에 Board가 들어있다. `game_init`에서 Board 초기화를 포함해서 처리
> - `game->drag.active = false` : 드래그 상태를 명시적으로 초기화. C에서 지역 변수는 초기화하지 않으면 쓰레기값이 들어있을 수 있다
> - `120.0f` : float 리터럴. `120`은 int, `120.0f`는 float. `timeLeft`가 float이므로 타입을 맞춰준다
> - `STATE_PLAYING` : Level 08에서 메뉴 화면을 추가할 때 `STATE_MENU`로 변경

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

> **코드 설명**
> - 게임 루프의 구조: **업데이트(로직) → 렌더링** 순서. 매 프레임 이 두 단계를 반복
> - `drag_update(&game.drag)` : `game.drag`의 주소를 넘겨 드래그 상태를 갱신. `BeginDrawing()` **전**에 호출해야 입력을 먼저 처리하고 그린다
> - `draw_drag_box(&game.drag, 0)` : sum을 0으로 고정해서 아직 항상 흰색 박스. Level 05에서 실제 합산값으로 교체

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
