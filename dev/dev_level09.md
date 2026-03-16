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

> **코드 설명**
> - `game.thinColor = !game.thinColor` : `!`(NOT 연산자)로 bool 값을 반전. `true`이면 `false`로, `false`이면 `true`로 전환. 이것이 **토글(toggle)** 패턴의 가장 간결한 구현
> - `IsKeyPressed(KEY_T)` : T키를 방금 누른 첫 프레임에만 true. `IsKeyDown`을 쓰면 누르는 동안 매 프레임 토글되어 빠르게 깜빡이는 버그 발생

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

> **코드 설명**
> - `Rectangle btn = { .x = ..., .y = ..., ... }` : C99 지정 초기화자(designated initializer). 필드 이름을 명시해서 가독성을 높임. `{ BOARD_X + 160, ..., 100, 26 }`처럼 순서만으로도 가능하지만 이 방식이 더 명확하다
> - `BOARD_Y + ROWS * CELL_SIZE + 4` : 게임판 바닥(점수 표시 위치)보다 4px 아래. `draw_score`와 같은 y 범위에 나란히 배치
> - `CheckCollisionPointRec(mouse, btn)` : Raylib 함수. 점(마우스 좌표)이 사각형(버튼 영역) 안에 있으면 `true`. 마우스 호버 감지에 사용
> - `bool clicked = hovered && IsMouseButtonReleased(...)` : 두 조건을 AND로 연결. 버튼 위에 있을 때 마우스를 뗐을 때만 `clicked = true`
> - **3단계 상태 분기**: `thin_color`(활성) > `hovered`(호버) > 기본 순서로 조건 체크. if-else if-else 패턴으로 우선순위를 표현
> - `thin_color ? BLACK : WHITE` : 활성(연한 색 ON) 상태일 때는 밝은 배경에 검정 텍스트, 비활성일 때는 어두운 배경에 흰색 텍스트로 대비를 높임
> - **반환값 bool**: 이 함수는 그리기뿐만 아니라 클릭 여부를 반환한다. main.c에서 이 값을 받아 thinColor를 토글하는 데 사용

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

> **코드 설명**
> - `if (draw_thin_button(game.thinColor))` : 함수 호출 결과를 if 조건으로 직접 사용. 버튼이 클릭되면(`true`) thinColor를 토글. Level 07까지는 `draw_score`, `draw_timer_bar`만 있었는데 여기서 버튼 그리기를 추가
> - 게임오버 상태에서도 버튼을 표시(`|| game.state == STATE_GAMEOVER`): 결과 화면에서도 연한 색 모드를 켜고 끌 수 있게

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

> **코드 설명**
> - **충돌 문제**: `IsMouseButtonReleased`는 draw_thin_button 안에서도, game_try_remove 전에도 모두 감지된다. 버튼 위에서 클릭을 떼면 두 가지가 동시에 실행된다. 버튼 클릭 = thinColor 토글, 사과 제거 시도 = 두 번 발생
> - `!CheckCollisionPointRec(mouse, btn)` : 마우스가 버튼 영역 **밖**에 있을 때만 game_try_remove 호출. 버튼 영역 안이면 사과 제거를 건너뜀
> - `get_thin_button_rect()` : 버튼의 Rectangle 위치를 함수로 분리. draw_thin_button과 main.c 두 곳에서 같은 좌표를 사용하는데, 함수 하나로 통일하면 위치를 바꿀 때 한 곳만 수정하면 된다

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

> **코드 설명**
> - `base.r / 2 + 100` : R 채널을 절반으로 줄이고 100을 더한다. 원래 220이었다면 220/2 + 100 = 210이 되어 비슷하게 밝지만, 원래 50이었다면 50/2 + 100 = 125가 되어 훨씬 밝아진다 → 전체적으로 색들이 비슷한 밝기(파스텔톤)로 수렴
> - `base.a = 130` : 투명도를 낮추는 방법. 130/255 ≈ 51% 불투명 → 배경색이 비쳐 보이면서 연해 보임. 이 방법은 배경색(어두운 회색)에 따라 결과가 달라진다

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
