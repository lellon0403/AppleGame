# Level 05 — 합산 로직 + 박스 색상 변경

## 목표
드래그 박스 안에 있는 사과들의 합을 실시간으로 계산하고,
합이 정확히 10이면 박스 색을 빨간색으로 바꾼다.

## 완료 기준
- 드래그 중 박스 안 사과 합이 계속 계산된다
- 합 == 10이면 박스가 빨간색으로 변한다
- 합 ≠ 10이면 박스가 흰색이다
- 이미 제거된 사과는 합산에서 제외된다

---

## 1단계 — board.c에 합산 함수 추가

`board.h`에 선언 추가:

```c
/* board.h 에 추가 */
#include "raylib.h"   /* Rectangle 타입 사용 */

/*
 * 드래그 박스(픽셀 Rectangle) 안에 있는 사과들의 합을 반환
 * board_x, board_y: 게임판 좌측상단 픽셀 좌표
 * cell_size: 셀 하나의 픽셀 크기
 */
int board_sum_in_rect(const Board *board,
                      Rectangle rect,
                      int board_x, int board_y, int cell_size);
```

`board.c`에 구현 추가:

```c
/* board.c 에 추가 */
int board_sum_in_rect(const Board *board,
                      Rectangle rect,
                      int board_x, int board_y, int cell_size)
{
    /*
     * 픽셀 좌표 rect를 격자 인덱스 범위로 변환
     * 사과 셀의 중심이 rect 안에 있으면 포함된 것으로 판단
     */

    /* rect의 픽셀 경계 */
    float rx0 = rect.x;
    float ry0 = rect.y;
    float rx1 = rect.x + rect.width;
    float ry1 = rect.y + rect.height;

    int sum = 0;
    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            if (board->grid[r][c].removed) continue;

            /* 이 셀의 중심 픽셀 좌표 */
            float cx = board_x + c * cell_size + cell_size / 2.0f;
            float cy = board_y + r * cell_size + cell_size / 2.0f;

            /* 중심이 rect 안에 있으면 합산 */
            if (cx >= rx0 && cx <= rx1 && cy >= ry0 && cy <= ry1) {
                sum += board->grid[r][c].value;
            }
        }
    }
    return sum;
}
```

> **설계 결정**: 원작 게임은 사과 셀의 "임의의 픽셀이 조금이라도 겹치면" 포함되는 방식이다.
> 그러나 "셀 중심이 박스 안에 있을 때만 포함"하는 방식이 구현이 단순하고 오조작이 적어 여기서는 이 방식을 채택한다.

---

## 2단계 — main.c에서 합산 호출 + draw_drag_box에 sum 전달

```c
/* main.c — 게임 루프 부분 */

while (!WindowShouldClose())
{
    /* ── 업데이트 ── */
    drag_update(&game.drag);

    /* 드래그 중일 때 실시간 합산 */
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
        draw_drag_box(&game.drag, current_sum);   /* sum 전달 */

        /* 디버그: 현재 합 표시 */
        char debug[32];
        sprintf(debug, "SUM: %d", current_sum);
        DrawText(debug, 20, 570, 18, YELLOW);

    EndDrawing();
}
```

`main.c`에 `#include <stdio.h>` 추가 (sprintf 사용)

---

## 3단계 — draw_drag_box 재확인

Level 04에서 작성한 `draw_drag_box`는 이미 `sum` 파라미터를 받아 색상을 바꾼다.

```c
/* render.c — draw_drag_box (Level 04에서 작성, 수정 불필요) */
void draw_drag_box(const DragBox *drag, int sum)
{
    if (!drag->active) return;

    Rectangle rect = drag_to_rect(drag);
    Color border_color = (sum == 10) ? RED : WHITE;   /* ← 이 부분이 핵심 */

    DrawRectangleRec(rect, Fade(border_color, 0.12f));
    DrawRectangleLinesEx(rect, 2.0f, border_color);
}
```

Level 04에서 `sum`을 항상 0으로 넘겼다면, 이제 실제 합산 값을 넘기므로 자동으로 동작한다.

---

## 4단계 — 빌드 및 확인

`F5`로 실행 후:

1. 합이 10이 되는 조합을 드래그한다
   - 예: `1+9`, `2+8`, `3+7`, `4+6`, `5+5`
   - 예: `1+2+7`, `2+3+5`
2. 합이 10이 되는 순간 박스가 **빨간색**으로 바뀌는지 확인
3. 다른 사과를 추가해 합이 10을 넘으면 **다시 흰색**으로 돌아오는지 확인
4. 화면 하단 "SUM: X" 텍스트로 실시간 합산값 확인

---

## 셀 포함 판단 방식 비교

| 방식 | 설명 | 장점 | 단점 |
|------|------|------|------|
| 셀 중심 포함 | 셀 중심(cx, cy)이 rect 안에 있으면 포함 | 구현 단순, 오조작 적음 | 박스 가장자리에서 직관과 다를 수 있음 |
| 셀 픽셀 겹침 | 셀 영역이 조금이라도 rect와 겹치면 포함 | 원작에 더 가까움 | 구현 복잡, 1픽셀 오조작 발생 가능 |

셀 중심 방식 코드는 위에 작성되어 있다.
셀 픽셀 겹침 방식으로 바꾸고 싶다면 조건을 아래로 교체:

```c
/* 셀 영역과 드래그 박스가 겹치는지 확인 (픽셀 겹침 방식) */
float sx0 = board_x + c * cell_size;
float sy0 = board_y + r * cell_size;
float sx1 = sx0 + cell_size;
float sy1 = sy0 + cell_size;

if (sx1 > rx0 && sx0 < rx1 && sy1 > ry0 && sy0 < ry1) {
    sum += board->grid[r][c].value;
}
```

---

## 자주 발생하는 오류

| 현상 | 원인 | 해결 |
|------|------|------|
| 합이 항상 0 | board_sum_in_rect 미호출 | drag.active 체크 + 함수 호출 확인 |
| 합이 이상하게 큼 | 제거된 사과 포함 | `removed` 체크 확인 |
| 박스 색이 안 바뀜 | draw_drag_box에 sum=0 고정 | current_sum 전달 확인 |
| BOARD_X/BOARD_Y 매개변수 오류 | render.h와 값 불일치 | `#include "render.h"` 후 상수 사용 |

---

## 다음 단계
Level 05 완료 후 → [dev_level06.md] 사과 제거 + 점수로 진행
