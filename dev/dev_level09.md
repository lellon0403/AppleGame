# Level 09 — 연한 색 토글

## 목표
원작의 「薄い色(연한 색)」 기능을 구현한다.
T키 또는 화면 버튼을 클릭하면 사과 색이 연해져 눈 피로도를 낮춘다.

## 완료 기준
- T키를 누를 때마다 연한 색 ↔ 기본 색이 전환된다
- 화면에 토글 버튼이 표시된다
- 버튼 클릭으로도 전환된다
- 연한 색 모드에서는 사과 색이 확연히 연해진다

---

## 1단계 — 연한 색 모드는 이미 준비됨

Level 03에서 `draw_apple`에 `bool thin_color` 파라미터를 이미 추가했다.
`game.thinColor` 필드도 Level 04에서 Game 구조체에 포함시켰다.

이번 단계에서 할 일:
1. T키 입력으로 `game.thinColor` 토글
2. 화면에 버튼 UI 추가 + 클릭 처리

---

## 2단계 — main.c에 T키 처리 추가

`STATE_PLAYING` 업데이트 블록에 추가:

```c
case STATE_PLAYING:
    drag_update(&game.drag);

    /* 연한 색 토글 (T키) */
    if (IsKeyPressed(KEY_T)) {
        game.thinColor = !game.thinColor;
    }

    if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) {
        game_try_remove(&game);
    }

    timer_update(&game);
    check_all_cleared(&game);
    break;
```

---

## 3단계 — render.c에 토글 버튼 추가

`render.h`에 선언 추가:

```c
/* render.h 에 추가 */

/* 연한 색 토글 버튼
 * 반환값: 버튼이 클릭됐으면 true */
bool draw_thin_button(bool thin_color);
```

`render.c`에 구현 추가:

```c
/* render.c 에 추가 */
#include <stdbool.h>

bool draw_thin_button(bool thin_color)
{
    /* 버튼 위치 — 게임판 아래 점수 옆 */
    Rectangle btn = {
        .x      = BOARD_X + 160,
        .y      = BOARD_Y + ROWS * CELL_SIZE + 4,
        .width  = 100,
        .height = 26
    };

    /* 마우스 호버 및 클릭 감지 */
    Vector2 mouse = GetMousePosition();
    bool hovered  = CheckCollisionPointRec(mouse, btn);
    bool clicked  = hovered && IsMouseButtonReleased(MOUSE_LEFT_BUTTON);

    /* 버튼 배경 */
    Color bg_color;
    if (thin_color)    bg_color = (Color){ 180, 180, 120, 255 };  /* 활성 */
    else if (hovered)  bg_color = (Color){  80,  80,  80, 255 };  /* 호버 */
    else               bg_color = (Color){  50,  50,  50, 255 };  /* 기본 */

    DrawRectangleRec(btn, bg_color);
    DrawRectangleLinesEx(btn, 1.5f, GRAY);

    /* 버튼 텍스트 */
    const char *label = thin_color ? "색: 연함 ON" : "색: 연함 OFF";
    int label_w = MeasureText(label, 14);
    DrawText(label,
             (int)(btn.x + btn.width  / 2 - label_w / 2),
             (int)(btn.y + btn.height / 2 - 7),
             14,
             thin_color ? BLACK : WHITE);

    return clicked;
}
```

---

## 4단계 — main.c 렌더링 블록에 버튼 추가

```c
/* main.c 렌더링 블록 (BeginDrawing 안) */

if (game.state == STATE_PLAYING || game.state == STATE_GAMEOVER) {
    ClearBackground((Color){ 30, 30, 30, 255 });
    draw_board(&game.board, game.thinColor);
    draw_drag_box(&game.drag, current_sum);
    draw_score(game.score);
    draw_timer_bar(game.timeLeft, 120.0f);

    /* 연한 색 버튼 — 게임오버 화면에서도 표시 */
    if (draw_thin_button(game.thinColor)) {
        game.thinColor = !game.thinColor;
    }

    if (game.state == STATE_GAMEOVER) {
        draw_gameover(&game);
    }
}
```

> **주의**: `draw_thin_button`이 `IsMouseButtonReleased`를 내부적으로 감지하므로
> 게임 로직의 `game_try_remove`와 충돌할 수 있다.
> 버튼 클릭 시에는 사과 제거가 동시에 실행되지 않도록 아래 처리를 추가한다.

---

## 5단계 — 버튼 클릭과 사과 제거 충돌 방지

버튼 영역을 클릭하면 사과 제거가 실행되지 않도록 한다.

방법: `game_try_remove`를 호출하기 전에 버튼 영역인지 확인

`game.h`에 버튼 영역 반환 함수 추가:

```c
/* render.h 에 추가 */
Rectangle get_thin_button_rect(void);
```

```c
/* render.c 에 추가 */
Rectangle get_thin_button_rect(void)
{
    return (Rectangle){
        BOARD_X + 160,
        BOARD_Y + ROWS * CELL_SIZE + 4,
        100,
        26
    };
}
```

`main.c`에서 사과 제거 전 버튼 영역 체크:

```c
if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) {
    Vector2 mouse = GetMousePosition();
    Rectangle btn = get_thin_button_rect();

    /* 버튼 위에서 뗀 경우 → 사과 제거 생략 */
    if (!CheckCollisionPointRec(mouse, btn)) {
        game_try_remove(&game);
    }
}
```

---

## 6단계 — 빌드 및 확인

`F5` 실행 후:

1. T키를 누를 때마다 사과 색이 진하게 ↔ 연하게 바뀐다
2. 화면 하단에 토글 버튼이 표시된다
3. 버튼에 마우스를 올리면 색이 변한다 (호버)
4. 버튼을 클릭하면 색이 전환된다
5. 버튼 클릭 시 사과가 제거되지 않는다

---

## 연한 색 시각 효과 미세조정

`draw_apple`의 연한 색 처리 부분:

```c
if (thin_color) {
    base.r = (unsigned char)(base.r / 2 + 100);
    base.g = (unsigned char)(base.g / 2 + 100);
    base.b = (unsigned char)(base.b / 2 + 100);
}
```

너무 연하다면 `+ 100` 값을 줄이고,
너무 진하다면 값을 늘린다.

또는 알파값을 줄이는 방법:

```c
if (thin_color) base.a = 130;   /* 0~255, 낮을수록 투명 */
```

---

## 자주 발생하는 오류

| 현상 | 원인 | 해결 |
|------|------|------|
| 버튼 클릭 시 사과도 제거됨 | 충돌 방지 코드 누락 | 5단계의 CheckCollisionPointRec 체크 추가 |
| T키가 STATE_GAMEOVER에서도 반응 | state 분기 미처리 | T키 처리를 STATE_PLAYING 블록 안에만 넣기 |
| 버튼이 게임판 위에 표시됨 | y 좌표 계산 오류 | `BOARD_Y + ROWS * CELL_SIZE` 재확인 |

---

## 다음 단계
Level 09 완료 후 → [dev_level10.md] BGM / 효과음 (선택)으로 진행
