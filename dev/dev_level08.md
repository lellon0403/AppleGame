# Level 08 — 메뉴 화면

## 목표
게임 시작 전 타이틀/메뉴 화면을 추가한다.
스페이스바를 누르면 게임이 시작되고, 게임오버에서 R키로 재시작할 때도 이 흐름을 따른다.

## 완료 기준
- 실행 시 메뉴 화면이 먼저 보인다
- 스페이스바를 누르면 게임이 시작된다
- 게임오버에서 R키를 누르면 메뉴로 돌아오거나 바로 재시작된다 (선택)

---

## 1단계 — STATE_MENU 흐름 설계

```
프로그램 시작
    → STATE_MENU  (타이틀 화면)
        → SPACE 누름
            → game_init() 호출
            → STATE_PLAYING
                → 120초 경과 또는 올 클리어
                    → STATE_GAMEOVER
                        → R 누름
                            → game_init() 호출
                            → STATE_PLAYING  (또는 STATE_MENU로 돌아가기)
```

> **흐름 설명**
> - 이것이 **상태 머신(State Machine)** 패턴. 게임이 특정 상태에 있을 때는 그에 맞는 로직만 실행되고, 조건이 충족되면 다른 상태로 전환
> - 각 상태는 이전에 선언한 `GameState` 열거형 값(STATE_MENU, STATE_PLAYING, STATE_GAMEOVER)으로 표현
> - 이 패턴의 장점: 메뉴에서 실수로 타이머가 돌거나, 게임오버 화면에서 드래그가 되는 등의 버그를 상태 체크만으로 방지

---

## 2단계 — game.c 수정

`game_init`이 STATE_MENU 에서 시작하도록 변경:

```c
/* game.c */
void game_init(Game *game)
{
    board_init(&game->board);
    game->drag.active = false;
    game->timeLeft    = 120.0f;
    game->score       = 0;
    game->state       = STATE_MENU;    /* ← PLAYING 에서 MENU 로 변경 */
    game->thinColor   = false;
    game->allCleared  = false;
    game->clearTime   = 0.0f;
}
```

> **코드 설명**
> - `STATE_MENU`로 초기화: 이제 프로그램 시작 시 바로 게임이 시작되지 않고 메뉴 화면이 먼저 표시된다
> - R키로 재시작해도 `game_init`을 호출하면 메뉴로 돌아가도록 할 수 있고, 아래 `game_start`를 따로 만들어 바로 PLAYING으로 전환할 수도 있다

---

## 3단계 — render.c에 메뉴 화면 렌더링 추가

`render.h`에 선언 추가:

```c
/* render.h 에 추가 */
void draw_menu(void);
```

`render.c`에 구현 추가:

```c
/* render.c 에 추가 */
void draw_menu(void)
{
    ClearBackground((Color){ 20, 20, 30, 255 });

    /* 타이틀 */
    const char *title = "Fruit Box";
    int title_size = 72;
    int title_w    = MeasureText(title, title_size);
    DrawText(title,
             1010 / 2 - title_w / 2,
             150,
             title_size,
             (Color){ 255, 80, 80, 255 });

    /* 부제목 */
    const char *subtitle = "- 사과 게임 -";
    int sub_size = 28;
    int sub_w    = MeasureText(subtitle, sub_size);
    DrawText(subtitle,
             1010 / 2 - sub_w / 2,
             240,
             sub_size,
             LIGHTGRAY);

    /* 규칙 설명 */
    const char *rule1 = "드래그로 합이 10인 사과를 선택하세요";
    const char *rule2 = "2분 안에 최대한 많이 제거하세요";
    int rule_size = 20;
    DrawText(rule1,
             1010 / 2 - MeasureText(rule1, rule_size) / 2,
             330, rule_size, GRAY);
    DrawText(rule2,
             1010 / 2 - MeasureText(rule2, rule_size) / 2,
             360, rule_size, GRAY);

    /* 시작 안내 — 깜빡임 효과 */
    double t = GetTime();
    if ((int)(t * 2) % 2 == 0) {   /* 0.5초 주기로 깜빡 */
        const char *start = "[ SPACE ] 시작";
        int start_size = 26;
        int start_w    = MeasureText(start, start_size);
        DrawText(start,
                 1010 / 2 - start_w / 2,
                 450,
                 start_size,
                 WHITE);
    }

    /* 조작 안내 */
    DrawText("T : 연한 색 토글", 20, 570, 16, DARKGRAY);
}
```

> **코드 설명**
> - `ClearBackground`를 draw_menu 안에서 호출: 메뉴 화면은 게임 화면(어두운 회색)과 다른 배경색(어두운 남색)을 쓰기 때문. main.c에서 따로 ClearBackground를 호출하지 않아도 된다
> - `const char *title = "Fruit Box"` : 문자열 리터럴을 포인터로 받는다. `char title[] = "Fruit Box"`와 동작은 비슷하지만, 포인터 방식이 MeasureText 같은 함수에 바로 전달하기 편하다
> - `1010 / 2 - title_w / 2` : 수평 중앙 정렬 공식. 창 너비의 절반 - 텍스트 너비의 절반 = 텍스트의 시작 x 좌표
> - `double t = GetTime()` : 프로그램 실행 이후 경과 시간(초)을 반환. `double`을 쓰는 이유 — float은 약 7자리 유효숫자, double은 약 15자리. 오래 실행될수록 초 단위 정밀도가 중요해진다
> - `(int)(t * 2) % 2 == 0` : 깜빡임 구현. `t * 2`는 1초에 2씩 증가. `(int)`로 소수점을 버리면 0, 0, 1, 1, 2, 2... 패턴이 된다. `% 2 == 0`이면 짝수 구간(보임), 홀수 구간(안 보임)으로 0.5초 주기 깜빡임

---

## 4단계 — main.c 수정

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
    game_init(&game);   /* → STATE_MENU 로 시작 */

    while (!WindowShouldClose())
    {
        /* ── 업데이트 ── */
        switch (game.state)
        {
            case STATE_MENU:
                if (IsKeyPressed(KEY_SPACE)) {
                    board_init(&game.board);   /* 새 판 생성 */
                    game.drag.active = false;
                    game.timeLeft    = 120.0f;
                    game.score       = 0;
                    game.allCleared  = false;
                    game.clearTime   = 0.0f;
                    game.state       = STATE_PLAYING;
                }
                break;

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
                    /* 메뉴로 돌아가지 않고 바로 재시작 */
                    board_init(&game.board);
                    game.drag.active = false;
                    game.timeLeft    = 120.0f;
                    game.score       = 0;
                    game.allCleared  = false;
                    game.clearTime   = 0.0f;
                    game.state       = STATE_PLAYING;
                }
                if (IsKeyPressed(KEY_M)) {
                    /* M키로 메뉴로 돌아가기 (선택 기능) */
                    game_init(&game);
                }
                break;
        }

        /* 실시간 합산 */
        int current_sum = 0;
        if (game.state == STATE_PLAYING && game.drag.active) {
            Rectangle rect = drag_to_rect(&game.drag);
            current_sum = board_sum_in_rect(&game.board, rect,
                                            BOARD_X, BOARD_Y, CELL_SIZE);
        }

        /* ── 렌더링 ── */
        BeginDrawing();

            if (game.state == STATE_MENU) {
                draw_menu();
            } else {
                ClearBackground((Color){ 30, 30, 30, 255 });
                draw_board(&game.board, game.thinColor);
                draw_drag_box(&game.drag, current_sum);
                draw_score(game.score);
                draw_timer_bar(game.timeLeft, 120.0f);

                if (game.state == STATE_GAMEOVER) {
                    draw_gameover(&game);
                }
            }

        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

> **코드 설명**
> - `case STATE_MENU:` 블록 : SPACE를 누르면 게임 데이터를 초기화하고 STATE_PLAYING으로 전환. `board_init`으로 새 사과 배치를 생성
> - `case STATE_GAMEOVER:` : R키는 바로 재시작, M키는 메뉴로 복귀. 두 가지 선택지를 제공
> - **렌더링 분기**: STATE_MENU일 때는 `draw_menu()`만 호출 (내부에서 ClearBackground 포함). 그 외 상태에서는 게임 화면을 그리고, 추가로 STATE_GAMEOVER일 때만 오버레이를 그린다
> - 메뉴 → 플레이 전환 시 초기화 코드가 중복된다. 이 중복을 없애는 방법이 아래 `game_start` 함수

> `draw_menu` 안에서 `ClearBackground`를 직접 호출하므로 메뉴 상태에서는
> `BeginDrawing()` 바로 뒤에 `draw_menu()`만 호출하면 된다.

---

## 5단계 — 빌드 및 확인

`F5` 실행 후:

1. 실행 즉시 타이틀 화면이 보인다
2. "[ SPACE ] 시작" 텍스트가 0.5초 간격으로 깜빡인다
3. SPACE를 누르면 게임판이 나타나고 타이머가 시작된다
4. 게임오버 후 R키 → 바로 새 게임 시작
5. 게임오버 후 M키 → 메뉴로 복귀

---

## 재시작 중복 코드 정리 (선택 최적화)

`STATE_MENU → PLAYING`과 `STATE_GAMEOVER → PLAYING` 에서 같은 초기화 코드가 반복된다.
이를 별도 함수로 분리하면 깔끔하다:

```c
/* game.h 에 추가 */
void game_start(Game *game);   /* 판만 새로 만들고 PLAYING으로 전환 */

/* game.c 에 추가 */
void game_start(Game *game)
{
    board_init(&game->board);
    game->drag.active = false;
    game->timeLeft    = 120.0f;
    game->score       = 0;
    game->allCleared  = false;
    game->clearTime   = 0.0f;
    game->state       = STATE_PLAYING;
}
```

> **코드 설명**
> - `game_init`과 `game_start`를 분리하는 이유: `game_init`은 오디오, thinColor 등 프로그램 전체 초기화. `game_start`는 새 판 시작만. R키로 재시작할 때 오디오를 다시 초기화할 필요는 없다
> - `thinColor`를 초기화하지 않는 것도 의도적: 연한 색 모드는 플레이어 설정이므로 재시작해도 유지

main.c에서 `board_init(...); ... game.state = STATE_PLAYING;` 블록을 `game_start(&game);` 한 줄로 교체.

---

## 자주 발생하는 오류

| 현상 | 원인 | 해결 |
|------|------|------|
| 메뉴에서 게임판이 비쳐 보임 | draw_menu에서 ClearBackground 누락 | draw_menu 첫 줄에 ClearBackground 호출 |
| SPACE를 눌러도 게임 시작 안됨 | state 분기 누락 | switch(STATE_MENU) 블록 확인 |
| 깜빡임 없이 고정 표시됨 | GetTime() 조건 오류 | `(int)(t * 2) % 2 == 0` 조건 확인 |

---

## 다음 단계
Level 08 완료 후 → [dev_level09.md] 연한 색 토글 버튼으로 진행
