# Level 03 — 사과 렌더링

## 목표
게임판의 170개 사과를 화면에 그린다.
사과는 원(Circle) + 숫자 텍스트 방식으로 표현한다 (이미지 없이).

## 완료 기준
- 17×10 격자에 사과 원이 빠짐없이 표시된다
- 각 사과 안에 1~9 숫자가 중앙 정렬되어 보인다
- 숫자마다 다른 색상이 적용된다

---

## 1단계 — 파일 추가

```
src/
├── main.c
├── board.h / board.c
├── render.h      ← 새로 생성
└── render.c      ← 새로 생성
```

---

## 2단계 — render.h 작성

```c
/* render.h */
#ifndef RENDER_H
#define RENDER_H

#include "raylib.h"
#include "board.h"
#include <stdbool.h>

/* 레이아웃 상수 */
#define CELL_SIZE  54      /* 사과 한 칸 크기 (px) */
#define BOARD_X    20      /* 게임판 좌측 여백 */
#define BOARD_Y    30      /* 게임판 상단 여백 */

/* 사과 색상 (숫자 1~9) */
Color apple_color(int value);

/* 사과 한 개 그리기 */
void draw_apple(int col, int row, const Apple *apple, bool thin_color);

/* 게임판 전체 그리기 */
void draw_board(const Board *board, bool thin_color);

#endif /* RENDER_H */
```

---

## 3단계 — render.c 작성

```c
/* render.c */
#include "render.h"

Color apple_color(int value)
{
    /* 1~9 각 숫자에 대응하는 색상 */
    Color palette[9] = {
        { 220,  50,  50, 255 },   /* 1 — 빨강   */
        { 230, 120,  30, 255 },   /* 2 — 주황   */
        { 200, 185,  20, 255 },   /* 3 — 노랑   */
        {  55, 170,  55, 255 },   /* 4 — 초록   */
        {  30, 155, 210, 255 },   /* 5 — 하늘   */
        {  50,  80, 200, 255 },   /* 6 — 파랑   */
        { 140,  60, 200, 255 },   /* 7 — 보라   */
        { 210,  80, 150, 255 },   /* 8 — 핑크   */
        { 150,  90,  50, 255 },   /* 9 — 갈색   */
    };
    return palette[value - 1];
}

void draw_apple(int col, int row, const Apple *apple, bool thin_color)
{
    if (apple->removed) return;   /* 제거된 사과는 그리지 않음 */

    /* 셀의 중앙 픽셀 좌표 계산 */
    float cx = BOARD_X + col * CELL_SIZE + CELL_SIZE / 2.0f;
    float cy = BOARD_Y + row * CELL_SIZE + CELL_SIZE / 2.0f;
    float radius = CELL_SIZE / 2.0f - 3.0f;   /* 셀 안쪽 여백 3px */

    /* 색상 결정 */
    Color base = apple_color(apple->value);
    if (thin_color) {
        /* 연한 색 모드: 채도/명도 낮춤 */
        base.r = (unsigned char)(base.r / 2 + 100);
        base.g = (unsigned char)(base.g / 2 + 100);
        base.b = (unsigned char)(base.b / 2 + 100);
    }

    /* 원 배경 + 테두리 */
    DrawCircleV((Vector2){ cx, cy }, radius, base);
    DrawCircleLinesV((Vector2){ cx, cy }, radius,
                     (Color){ 0, 0, 0, 60 });   /* 반투명 검정 테두리 */

    /* 숫자 텍스트 중앙 정렬 */
    char buf[2] = { (char)('0' + apple->value), '\0' };
    int  font_size = 20;
    int  tw = MeasureText(buf, font_size);
    DrawText(buf,
             (int)(cx - tw / 2.0f),
             (int)(cy - font_size / 2.0f),
             font_size, WHITE);
}

void draw_board(const Board *board, bool thin_color)
{
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            draw_apple(c, r, &board->grid[r][c], thin_color);
}
```

---

## 4단계 — main.c 수정

```c
/* main.c */
#include "raylib.h"
#include "board.h"
#include "render.h"
#include <stdlib.h>
#include <time.h>

int main(void)
{
    srand((unsigned int)time(NULL));

    Board board;
    board_init(&board);

    InitWindow(1010, 600, "Fruit Box");
    SetTargetFPS(60);

    while (!WindowShouldClose())
    {
        BeginDrawing();
            ClearBackground((Color){ 30, 30, 30, 255 });

            draw_board(&board, false);   /* 사과 전체 렌더링 */

            /* 임시 디버그 텍스트 */
            DrawText("Level 03 - 사과 렌더링", 20, 580, 14, GRAY);

        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

---

## 5단계 — 빌드 및 확인

1. `Ctrl+Shift+B` 빌드
2. `F5` 실행

### 정상 결과
```
┌─────────────────────────────────────────┐
│  ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ●      │
│  3 7 2 5 1 8 4 6 9 2 3 5 1 7 4 2 6      │
│  ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ● ●      │
│  ...                                    │
└─────────────────────────────────────────┘
```
- 색깔이 다른 원들이 17×10으로 배치된다
- 각 원 안에 숫자가 보인다

---

## 레이아웃 조정

사과가 화면을 벗어나거나 너무 빽빽한 경우 `render.h`의 상수를 수정한다.

| 상수 | 기본값 | 의미 |
|------|--------|------|
| `CELL_SIZE` | 54 | 사과 한 칸 크기. 줄이면 판이 작아짐 |
| `BOARD_X` | 20 | 좌측 여백 |
| `BOARD_Y` | 30 | 상단 여백 |

게임판 전체 너비: `17 × 54 = 918px`
게임판 전체 높이: `10 × 54 = 540px`
창 크기(1010×600) 기준으로 여유 공간: 우측 72px, 하단 30px

---

## 자주 발생하는 오류

| 오류 | 원인 | 해결 |
|------|------|------|
| 사과가 화면 밖으로 나감 | CELL_SIZE가 너무 큼 | CELL_SIZE를 48~50으로 줄여보기 |
| 숫자가 원 중앙에 없음 | 폰트 크기 대비 위치 계산 오류 | `cy - font_size / 2.0f` 값 조정 |
| 원이 겹침 | radius가 CELL_SIZE/2 초과 | `CELL_SIZE / 2.0f - 3.0f` 여백 확인 |

---

## 다음 단계
Level 03 완료 후 → [dev_level04.md] 마우스 드래그 박스 표시로 진행
