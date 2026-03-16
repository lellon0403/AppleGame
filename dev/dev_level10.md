# Level 10 — BGM / 효과음 (선택)

## 목표
배경음악(BGM)과 효과음을 추가한다.
Raylib의 오디오 시스템을 사용하므로 외부 라이브러리 추가 없이 구현 가능하다.

## 완료 기준
- 게임 중 BGM이 반복 재생된다
- 사과 제거 시 효과음이 재생된다
- 게임오버 시 BGM이 멈추고 종료 효과음이 재생된다
- BGM 없이도 게임이 정상 동작한다 (오디오 파일 없으면 조용히 무시)

---

## 오디오 파일 준비

### 지원 포맷
Raylib은 `.wav`, `.mp3`, `.ogg`, `.flac` 형식을 지원한다.

### 무료 사운드 소스
- https://freesound.org (효과음, 키워드: `pop`, `chime`, `game over`)
- https://opengameart.org (BGM, 키워드: `puzzle bgm loop`)

### 파일 배치
```
assets/
└── sounds/
    ├── bgm.ogg        ← 루프 배경음악
    ├── pop.wav        ← 사과 제거 효과음
    └── gameover.wav   ← 게임오버 효과음
```

---

## 1단계 — 프로젝트 속성에 오디오 라이브러리 추가

Raylib의 오디오는 `InitAudioDevice()`를 호출하면 자동으로 활성화된다.
별도 라이브러리 추가는 필요 없다.

---

## 2단계 — audio.h / audio.c 생성

```c
/* audio.h */
#ifndef AUDIO_H
#define AUDIO_H

#include "raylib.h"
#include <stdbool.h>

typedef struct {
    Music bgm;         /* 루프 재생할 배경음악 */
    Sound pop;         /* 사과 제거 효과음 */
    Sound gameover;    /* 게임오버 효과음 */
    bool  loaded;      /* 오디오 로딩 성공 여부 */
} AudioAssets;

void audio_init(AudioAssets *audio);
void audio_update(AudioAssets *audio);   /* 매 프레임 호출 (BGM 스트리밍용) */
void audio_play_bgm(AudioAssets *audio);
void audio_stop_bgm(AudioAssets *audio);
void audio_play_pop(AudioAssets *audio);
void audio_play_gameover(AudioAssets *audio);
void audio_unload(AudioAssets *audio);

#endif /* AUDIO_H */
```

> **코드 설명**
> - `Music bgm` vs `Sound pop` : Raylib에서 두 타입의 차이가 중요하다
>   - `Sound` : 파일 전체를 **메모리에 한 번에 올려** 재생. 짧은 효과음에 적합. 로딩 후 즉시 재생 가능
>   - `Music` : 파일을 **스트리밍(조금씩 읽으며)** 재생. 긴 BGM 파일을 메모리에 다 올리지 않아도 되어 효율적. 대신 매 프레임 `UpdateMusicStream()`을 호출해야 끊기지 않는다
> - `bool loaded` : 오디오 파일이 없어도 게임이 크래시되지 않도록 하는 안전장치. 파일을 못 찾으면 `loaded = false`로 두고, 모든 오디오 함수에서 이 플래그를 먼저 확인한다

```c
/* audio.c */
#include "audio.h"
#include <stdio.h>

void audio_init(AudioAssets *audio)
{
    InitAudioDevice();
    audio->loaded = false;

    /* 파일이 없으면 조용히 건너뜀 */
    if (!FileExists("assets/sounds/bgm.ogg")   ||
        !FileExists("assets/sounds/pop.wav")    ||
        !FileExists("assets/sounds/gameover.wav"))
    {
        printf("[오디오] 사운드 파일이 없습니다. 무음으로 진행합니다.\n");
        return;
    }

    audio->bgm      = LoadMusicStream("assets/sounds/bgm.ogg");
    audio->pop      = LoadSound("assets/sounds/pop.wav");
    audio->gameover = LoadSound("assets/sounds/gameover.wav");

    /* BGM 볼륨 조정 */
    SetMusicVolume(audio->bgm, 0.5f);
    SetSoundVolume(audio->pop, 0.8f);
    SetSoundVolume(audio->gameover, 0.9f);

    audio->loaded = true;
}

void audio_update(AudioAssets *audio)
{
    /* Music 타입은 스트리밍 방식이므로 매 프레임 업데이트 필요 */
    if (audio->loaded) UpdateMusicStream(audio->bgm);
}

void audio_play_bgm(AudioAssets *audio)
{
    if (!audio->loaded) return;
    PlayMusicStream(audio->bgm);
}

void audio_stop_bgm(AudioAssets *audio)
{
    if (!audio->loaded) return;
    StopMusicStream(audio->bgm);
}

void audio_play_pop(AudioAssets *audio)
{
    if (!audio->loaded) return;
    PlaySound(audio->pop);
}

void audio_play_gameover(AudioAssets *audio)
{
    if (!audio->loaded) return;
    PlaySound(audio->gameover);
}

void audio_unload(AudioAssets *audio)
{
    if (!audio->loaded) {
        CloseAudioDevice();
        return;
    }
    UnloadMusicStream(audio->bgm);
    UnloadSound(audio->pop);
    UnloadSound(audio->gameover);
    CloseAudioDevice();
}
```

> **코드 설명**
> - `InitAudioDevice()` : Raylib 오디오 시스템을 초기화. `InitWindow()` 이후에 호출해야 한다. 프로그램 종료 전 `CloseAudioDevice()`로 정리
> - `FileExists("assets/sounds/bgm.ogg")` : Raylib 함수. 파일이 실제로 존재하면 `true`. 없는 파일을 `LoadMusicStream`으로 로드하면 오류가 나므로 미리 확인
> - `||` 조건: 세 파일 중 하나라도 없으면 오디오 없이 진행. 일부만 있는 상황을 단순하게 처리
> - `SetMusicVolume(audio->bgm, 0.5f)` : 0.0(무음) ~ 1.0(최대 볼륨). BGM을 0.5f로 낮추면 효과음이 묻히지 않는다
> - `audio->loaded = true` : 모든 로딩이 성공한 후에만 true. 이후 모든 오디오 함수의 첫 줄에서 이 값을 확인하므로, 로딩 실패 시 어떤 함수도 실제 오디오 처리를 하지 않는다
> - `UpdateMusicStream(audio->bgm)` : BGM이 끊기지 않고 계속 재생되려면 매 프레임 이 함수를 호출해야 한다. `Sound`는 이런 처리가 필요 없다. main.c의 게임 루프 최상단에서 호출
> - `audio_unload` : `loaded`가 false일 때도 `CloseAudioDevice()`는 호출해야 한다. 파일 로딩 실패 시에도 `InitAudioDevice()`는 이미 호출되었기 때문

---

## 3단계 — Game 구조체에 AudioAssets 추가

`game.h` 수정:

```c
/* game.h */
#include "audio.h"   /* 추가 */

typedef struct {
    Board       board;
    DragBox     drag;
    float       timeLeft;
    int         score;
    GameState   state;
    bool        thinColor;
    bool        allCleared;
    float       clearTime;
    AudioAssets audio;       /* 추가 */
    bool        gameoverSoundPlayed;  /* 게임오버 효과음 중복 방지 */
} Game;
```

`game.c`의 `game_init`에 초기화 추가:

```c
void game_init(Game *game)
{
    board_init(&game->board);
    game->drag.active          = false;
    game->timeLeft             = 120.0f;
    game->score                = 0;
    game->state                = STATE_MENU;
    game->thinColor            = false;
    game->allCleared           = false;
    game->clearTime            = 0.0f;
    game->gameoverSoundPlayed  = false;
    /* audio는 main에서 별도로 audio_init 호출 — game_init마다 재초기화 금지 */
}
```

> **코드 설명**
> - `AudioAssets audio` : Game 구조체 안에 오디오 데이터를 포함. 이제 `game.audio`로 모든 오디오 상태에 접근 가능
> - `bool gameoverSoundPlayed` : 게임오버 효과음 **중복 재생 방지**용 플래그. 게임오버 상태가 되는 순간 한 번만 재생해야 하는데, main.c 루프가 계속 돌면서 같은 조건이 매 프레임 충족되어 효과음이 반복되는 버그를 막는다
> - `/* audio는 main에서 별도로 audio_init 호출 */` : `game_init`이 R키 재시작에서도 호출되는데, 그때마다 `audio_init`을 다시 호출하면 오디오 장치를 다시 열고 파일을 다시 로드하는 비효율이 생긴다. 오디오는 최초 1회만 초기화

---

## 4단계 — main.c에 오디오 연동

```c
/* main.c 전체 */
#include "raylib.h"
#include "game.h"
#include "render.h"
#include "input.h"
#include "timer.h"
#include "audio.h"
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
    audio_init(&game.audio);   /* 창 초기화 후 오디오 초기화 */

    bool prev_state_was_gameover = false;

    while (!WindowShouldClose())
    {
        /* ── 오디오 스트림 업데이트 (매 프레임) ── */
        audio_update(&game.audio);

        /* ── 업데이트 ── */
        switch (game.state)
        {
            case STATE_MENU:
                if (IsKeyPressed(KEY_SPACE)) {
                    game_start(&game);
                    audio_play_bgm(&game.audio);   /* BGM 시작 */
                }
                break;

            case STATE_PLAYING:
                drag_update(&game.drag);

                if (IsKeyPressed(KEY_T)) {
                    game.thinColor = !game.thinColor;
                }

                if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) {
                    Vector2 mouse = GetMousePosition();
                    Rectangle btn = get_thin_button_rect();

                    if (!CheckCollisionPointRec(mouse, btn)) {
                        int before = game.score;
                        game_try_remove(&game);
                        if (game.score > before) {
                            audio_play_pop(&game.audio);   /* 제거 효과음 */
                        }
                    }
                }

                timer_update(&game);
                check_all_cleared(&game);

                /* 방금 GAMEOVER로 전환됐으면 BGM 정지 + 효과음 */
                if (game.state == STATE_GAMEOVER && !game.gameoverSoundPlayed) {
                    audio_stop_bgm(&game.audio);
                    audio_play_gameover(&game.audio);
                    game.gameoverSoundPlayed = true;
                }
                break;

            case STATE_GAMEOVER:
                if (IsKeyPressed(KEY_R)) {
                    game_start(&game);
                    game.gameoverSoundPlayed = false;
                    audio_play_bgm(&game.audio);
                }
                if (IsKeyPressed(KEY_M)) {
                    game_init(&game);
                    audio_stop_bgm(&game.audio);
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

                if (draw_thin_button(game.thinColor)) {
                    game.thinColor = !game.thinColor;
                }

                if (game.state == STATE_GAMEOVER) {
                    draw_gameover(&game);
                }
            }

        EndDrawing();
    }

    audio_unload(&game.audio);
    CloseWindow();
    return 0;
}
```

> **코드 설명**
> - `audio_init(&game.audio)` : `InitWindow()` 이후, 게임 루프 시작 전에 딱 한 번 호출. `game_init`과 별도로 초기화
> - `audio_update(&game.audio)` : 게임 루프 최상단에서 매 프레임 호출. switch 블록보다 앞에 두어야 어느 상태에서든 BGM이 끊기지 않는다
> - `int before = game.score; game_try_remove(&game); if (game.score > before)` : 사과 제거 전후 점수를 비교해서 실제로 제거됐을 때만 효과음 재생. `game_try_remove`가 반환값이 없으므로 이 방식으로 제거 여부를 감지
> - `if (game.state == STATE_GAMEOVER && !game.gameoverSoundPlayed)` : `STATE_PLAYING` 케이스 안에서, `timer_update`나 `check_all_cleared`로 인해 방금 게임오버로 전환됐을 때를 감지. `!gameoverSoundPlayed`로 한 번만 실행
> - `game.gameoverSoundPlayed = false` (R키 재시작 시): 새 게임 시작 전에 플래그를 리셋해야 다음 게임오버 때 효과음이 다시 재생된다
> - `audio_unload(&game.audio)` : `CloseWindow()` **전**에 오디오를 먼저 정리. 순서를 바꾸면 오디오 장치가 닫힌 후에 메모리를 해제하려 해서 오류 발생 가능

---

## 5단계 — 빌드 및 확인

`F5` 실행 후:

### 오디오 파일이 있는 경우
1. 스페이스로 게임 시작 → BGM이 재생된다
2. 합 10 조합 제거 → "pop" 효과음이 난다
3. 시간 종료 또는 올 클리어 → BGM이 멈추고 게임오버 효과음이 난다
4. R키 재시작 → BGM이 다시 시작된다

### 오디오 파일이 없는 경우
- 콘솔에 "[오디오] 사운드 파일이 없습니다." 출력
- 게임은 무음으로 정상 동작한다

---

## BGM 루프 설정

Raylib의 `Music`은 기본적으로 루프 재생된다.
루프를 끄려면:

```c
audio->bgm.looping = false;
```

---

## 볼륨 조절 팁

| 함수 | 대상 | 범위 |
|------|------|------|
| `SetMusicVolume(music, v)` | BGM | 0.0 ~ 1.0 |
| `SetSoundVolume(sound, v)` | 효과음 | 0.0 ~ 1.0 |
| `SetMasterVolume(v)` | 전체 | 0.0 ~ 1.0 |

---

## 자주 발생하는 오류

| 현상 | 원인 | 해결 |
|------|------|------|
| BGM이 한 번만 재생됨 | UpdateMusicStream 미호출 | audio_update를 매 프레임 호출 확인 |
| 효과음이 안 남 | audio.loaded가 false | 파일 경로 확인, assets/sounds/ 폴더 위치 확인 |
| 게임오버 효과음이 반복 재생됨 | gameoverSoundPlayed 플래그 누락 | 5단계의 플래그 확인 |
| 파일 경로 오류 | 실행 파일 기준 상대 경로 | .exe 위치 기준으로 assets/ 폴더가 있어야 함 |

### 실행 파일 기준 경로 문제 해결
Visual Studio에서 `F5` 실행 시 작업 디렉터리는 기본적으로 `.vcxproj` 파일이 있는 폴더다.
`assets/` 폴더를 `.vcxproj` 옆에 두거나,
프로젝트 속성 → 디버깅 → 작업 디렉터리를 명시적으로 설정한다.

---

## 전체 완성

Level 10까지 완료하면 다음 기능이 모두 구현된다:

| 기능 | 구현 단계 |
|------|----------|
| 게임판 17×10 랜덤 생성 | Level 02 |
| 사과 렌더링 (원 + 숫자) | Level 03 |
| 마우스 드래그 박스 | Level 04 |
| 합산 로직 + 빨간 박스 | Level 05 |
| 사과 제거 + 점수 | Level 06 |
| 2분 타이머 + 게임오버 | Level 07 |
| 메뉴 화면 + 재시작 | Level 08 |
| 연한 색 토글 | Level 09 |
| BGM + 효과음 | Level 10 |
