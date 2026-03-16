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

> **구조 설명**
> - 화면에 그리는 기능을 `render.h / render.c`로 분리. board.c가 "데이터 담당"이라면 render.c는 "화면 출력 담당"
> - 이렇게 역할별로 파일을 나누면 나중에 렌더링 방식을 바꿀 때 render.c만 수정하면 된다

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

> **코드 설명**
> - `#define CELL_SIZE 54` : 사과 한 칸의 픽셀 크기. 17열 × 54px = 918px, 10행 × 54px = 540px. 창 크기 1010×600 안에 딱 맞는 값
> - `BOARD_X`, `BOARD_Y` : 게임판이 창의 좌측 상단 모서리에서 얼마나 떨어져 있는지. 여백(패딩) 역할
> - `bool thin_color` : 연한 색 모드 여부. 원작의 「薄い色」 기능 (Level 09에서 완성). 지금은 `false`로 고정해서 쓴다
> - `Color` : Raylib가 정의한 `{unsigned char r, g, b, a}` 구조체

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

> **코드 설명**
> - `palette[value - 1]` : 배열 인덱스는 0부터 시작하는데 value는 1~9이므로, `value - 1`로 0~8 인덱스로 맞춘다. value가 1이면 `palette[0]` (빨강), 9이면 `palette[8]` (갈색)
> - `float cx = BOARD_X + col * CELL_SIZE + CELL_SIZE / 2.0f` : 셀의 **중심** 픽셀 좌표 계산. `BOARD_X`(게임판 시작점) + `col * CELL_SIZE`(해당 열의 시작점) + `CELL_SIZE / 2.0f`(셀 중앙까지)
> - `CELL_SIZE / 2.0f - 3.0f` : 반지름. `2.0f`처럼 f를 붙이는 이유 — 정수 나눗셈을 방지. `54 / 2`는 정수 27이지만, `54 / 2.0f`는 실수 27.0. 여기서는 결과가 같지만 소수점이 있는 경우 차이가 남
> - `(unsigned char)(base.r / 2 + 100)` : R,G,B 값을 반으로 줄이고 100을 더해서 연한(파스텔) 색으로 만든다. `unsigned char`로 캐스팅해 0~255 범위를 보장
> - `DrawCircleV((Vector2){ cx, cy }, radius, base)` : `Vector2`는 Raylib가 정의한 `{float x, float y}` 구조체. `(Vector2){ cx, cy }`는 즉석에서 구조체를 만드는 C99 복합 리터럴 문법
> - `(Color){ 0, 0, 0, 60 }` : 검정색인데 알파(투명도)가 60 → 약 24% 불투명. 반투명 테두리 효과
> - `char buf[2] = { (char)('0' + apple->value), '\0' }` : 정수 `3`을 문자 `'3'`으로 변환하는 트릭. `'0'`의 ASCII 값은 48, `3`을 더하면 51 = `'3'`의 ASCII. `'\0'`은 문자열 끝 표시 (null terminator)
> - `MeasureText(buf, font_size)` : 이 텍스트를 그렸을 때의 픽셀 **너비**를 반환. 반으로 나눠서 빼면 텍스트를 중앙 정렬할 수 있다
> - `draw_board` : 이중 for문으로 전체 10×17 격자를 순회. **주의**: 바깥쪽이 행(r), 안쪽이 열(c). `draw_apple`에 넘길 때는 `(c, r)` 순서로 넘긴다 (함수 인터페이스가 col, row 순)

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

> **코드 설명**
> - `#include "render.h"` 추가: 이제 `draw_board` 함수를 사용할 수 있다. render.h가 내부적으로 `#include "board.h"`를 하므로, main.c에서 board.h를 따로 include하지 않아도 되지만 명시적으로 포함하는 것이 더 명확하다
> - `draw_board(&board, false)` : 두 번째 인자 `false`는 연한 색 모드 비활성화. Level 09에서 `game.thinColor` 값으로 교체할 예정
> - `board_init`을 `InitWindow` **이전**에 호출: 창이 열리기 전에 데이터를 준비. 반드시 이 순서일 필요는 없지만 논리적으로 자연스러움

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
