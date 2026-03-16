# 사과 게임 (Fruit Box) — C 언어 구현 개발 계획서

## 개발 환경

| 항목 | 내용 |
|------|------|
| 언어 | C (C99 이상) |
| 컴파일러 | MSVC (Visual Studio 2022 권장) |
| 그래픽 라이브러리 | [Raylib 5.x](https://www.raylib.com/) |
| 플랫폼 | Windows |

### Raylib 설정 방법 (Visual Studio)
1. https://github.com/raysan5/raylib/releases 에서 `raylib-5.x_win64_msvc16.zip` 다운로드
2. 압축 해제 후 프로젝트에 `include/`, `lib/` 경로 추가
3. 프로젝트 속성 → 링커 → 추가 종속성에 `raylib.lib`, `winmm.lib` 추가

---

## 프로젝트 구조

```
fruit_box/
├── src/
│   ├── main.c          # 진입점, 게임 루프
│   ├── game.c / .h     # 게임 상태, 로직
│   ├── board.c / .h    # 게임판 (사과 배치, 제거)
│   ├── input.c / .h    # 마우스 드래그 처리
│   ├── render.c / .h   # 화면 렌더링
│   └── timer.c / .h    # 타이머 (2분 카운트다운)
└── assets/
    ├── images/
    │   └── apple.png   # 사과 이미지 (권장: 54×54px 또는 48×48px)
    └── sounds/         # (선택) BGM, 효과음
```

---

## 데이터 구조 설계

### 사과 하나

```c
typedef struct {
    int value;      // 1~9
    bool removed;   // 제거 여부
} Apple;
```

### 게임판

```c
#define COLS 17
#define ROWS 10
#define TOTAL_APPLES (COLS * ROWS)  // 170

typedef struct {
    Apple grid[ROWS][COLS];
    int score;
} Board;
```

### 드래그 선택 박스

```c
typedef struct {
    Vector2 start;   // 드래그 시작 좌표 (픽셀)
    Vector2 end;     // 드래그 현재 좌표 (픽셀)
    bool active;     // 드래그 중 여부
} DragBox;
```

### 게임 전체 상태

```c
typedef enum {
    STATE_MENU,
    STATE_PLAYING,
    STATE_GAMEOVER
} GameState;

typedef struct {
    Board board;
    DragBox drag;
    float timeLeft;   // 남은 시간 (초, 120.0f 시작)
    GameState state;
    bool allCleared;  // 올 클리어 여부
    float clearTime;  // 올 클리어 소요 시간
} Game;
```

---

## 게임판 초기화

### 사과 숫자 생성 규칙
- 1~9 숫자를 랜덤 배치
- **합이 10의 배수**가 되어야 함 (나무위키 확인 사항)
- **9의 개수 ≤ 1의 개수** 조건 만족 시에만 사용 (올 클리어 가능 보장)

```c
void board_init(Board *board) {
    int values[TOTAL_APPLES];
    int sum = 0;

    // 조건 만족할 때까지 재생성
    do {
        sum = 0;
        for (int i = 0; i < TOTAL_APPLES; i++) {
            values[i] = (rand() % 9) + 1;  // 1~9
            sum += values[i];
        }
    } while (sum % 10 != 0 || count9(values) > count1(values));

    // Fisher-Yates 셔플
    for (int i = TOTAL_APPLES - 1; i > 0; i--) {
        int j = rand() % (i + 1);
        int tmp = values[i]; values[i] = values[j]; values[j] = tmp;
    }

    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++) {
            board->grid[r][c].value   = values[r * COLS + c];
            board->grid[r][c].removed = false;
        }
    board->score = 0;
}
```

---

## 핵심 로직

### 드래그 박스 → 사과 셀 변환

픽셀 좌표의 드래그 박스가 어느 사과 셀과 겹치는지 판단한다.

```c
// 사과 셀 하나의 크기와 시작 오프셋
#define CELL_SIZE   54
#define BOARD_X     20
#define BOARD_Y     20

// 픽셀 좌표 → 격자 인덱스
int pixel_to_col(float x) { return (int)((x - BOARD_X) / CELL_SIZE); }
int pixel_to_row(float y) { return (int)((y - BOARD_Y) / CELL_SIZE); }
```

### 드래그 박스 내 사과 합산

```c
int sum_in_drag(Board *board, DragBox *drag) {
    // 드래그 박스의 픽셀 좌표 → 격자 범위
    float x0 = fminf(drag->start.x, drag->end.x);
    float y0 = fminf(drag->start.y, drag->end.y);
    float x1 = fmaxf(drag->start.x, drag->end.x);
    float y1 = fmaxf(drag->start.y, drag->end.y);

    int c0 = fmaxf(0, pixel_to_col(x0));
    int r0 = fmaxf(0, pixel_to_row(y0));
    int c1 = fminf(COLS - 1, pixel_to_col(x1));
    int r1 = fminf(ROWS - 1, pixel_to_row(y1));

    int sum = 0;
    for (int r = r0; r <= r1; r++)
        for (int c = c0; c <= c1; c++)
            if (!board->grid[r][c].removed)
                sum += board->grid[r][c].value;
    return sum;
}
```

### 사과 제거 및 점수 획득

```c
// 드래그 해제 시 호출
void try_remove(Game *game) {
    if (!game->drag.active) return;

    if (sum_in_drag(&game->board, &game->drag) == 10) {
        // 범위 내 사과 제거 + 점수 합산
        /* ... (sum_in_drag와 동일한 범위 순회) ... */
        game->board.grid[r][c].removed = true;
        game->board.score++;
    }
    game->drag.active = false;
}
```

---

## 입력 처리

```c
void input_update(Game *game) {
    if (IsMouseButtonPressed(MOUSE_LEFT_BUTTON)) {
        game->drag.start  = GetMousePosition();
        game->drag.end    = game->drag.start;
        game->drag.active = true;
    }
    if (game->drag.active && IsMouseButtonDown(MOUSE_LEFT_BUTTON)) {
        game->drag.end = GetMousePosition();
    }
    if (IsMouseButtonReleased(MOUSE_LEFT_BUTTON)) {
        try_remove(game);
    }
}
```

---

## 타이머

```c
void timer_update(Game *game) {
    if (game->state != STATE_PLAYING) return;
    game->timeLeft -= GetFrameTime();  // Raylib: 이전 프레임 경과 시간 반환
    if (game->timeLeft <= 0.0f) {
        game->timeLeft = 0.0f;
        game->state    = STATE_GAMEOVER;
    }
}
```

---

## 렌더링

### 렌더링 순서

```
BeginDrawing()
  └─ ClearBackground
  └─ draw_board()        // 사과 그리기
  └─ draw_drag_box()     // 드래그 중인 박스
  └─ draw_score()        // 점수 표시
  └─ draw_timer_bar()    // 타이머 게이지
  └─ (STATE_GAMEOVER 시) draw_result()
EndDrawing()
```

### 사과 이미지 방식 선택

사과를 표현하는 방법은 두 가지다. 프로젝트 초기에는 **방식 A**로 먼저 구현하고, 완성 후 **방식 B**로 교체하는 것을 추천한다.

#### 방식 A — 코드로 직접 그리기 (이미지 파일 불필요)

원 + 숫자 텍스트를 조합해 사과처럼 표현한다.

```c
void draw_apple(int col, int row, Apple *apple, bool thin_color) {
    if (apple->removed) return;

    float x = BOARD_X + col * CELL_SIZE + CELL_SIZE / 2.0f;
    float y = BOARD_Y + row * CELL_SIZE + CELL_SIZE / 2.0f;
    float r = CELL_SIZE / 2.0f - 3.0f;  // 셀 안쪽 여백 3px

    Color base = apple_color(apple->value);
    if (thin_color) base = Fade(base, 0.5f);  // 연한 색 모드

    DrawCircle((int)x, (int)y, r, base);
    DrawCircleLines((int)x, (int)y, r, Fade(BLACK, 0.3f));  // 테두리

    // 숫자 텍스트 중앙 정렬
    char buf[2] = { '0' + apple->value, '\0' };
    int  tw     = MeasureText(buf, 20);
    DrawText(buf, (int)x - tw / 2, (int)y - 10, 20, WHITE);
}
```

#### 방식 B — PNG 이미지 사용

원작처럼 실제 사과 이미지를 쓸 때. 이미지 1장(`apple.png`)을 숫자마다 색 조정해서 재사용한다.

##### 이미지 준비
- 크기: **54×54px** (CELL_SIZE와 동일)
- 형식: PNG (투명 배경 권장)
- 숫자는 코드에서 텍스트로 덮어 그림 — 이미지에 숫자를 포함하지 않아도 됨
- 무료 소스 예시: [Kenney.nl Assets](https://kenney.nl) 검색어: `fruit`

##### 텍스처 로딩 (게임 시작 시 1회)

```c
// render.h
typedef struct {
    Texture2D apple;   // 사과 이미지 1장
    Font      font;    // 숫자 폰트 (선택, 기본 폰트도 가능)
} Assets;

// render.c
Assets assets_load(void) {
    Assets a;
    a.apple = LoadTexture("assets/images/apple.png");
    // 텍스처가 CELL_SIZE와 다를 경우를 대비해 크기 확인
    // a.font = LoadFontEx("assets/fonts/bold.ttf", 22, NULL, 0);
    return a;
}

void assets_unload(Assets *a) {
    UnloadTexture(a->apple);
    // UnloadFont(a->font);
}
```

##### 이미지로 사과 한 개 그리기

```c
void draw_apple_texture(int col, int row, Apple *apple,
                        Texture2D tex, bool thin_color) {
    if (apple->removed) return;

    float x = BOARD_X + col * CELL_SIZE;
    float y = BOARD_Y + row * CELL_SIZE;

    // 숫자(1~9)에 따라 색조(tint) 변경 — 이미지 1장으로 9가지 색 표현
    Color tint = apple_color(apple->value);
    if (thin_color) tint = Fade(tint, 0.55f);

    // 이미지 크기를 CELL_SIZE에 맞게 스케일
    Rectangle src  = { 0, 0, (float)tex.width, (float)tex.height };
    Rectangle dest = { x, y, CELL_SIZE, CELL_SIZE };
    DrawTexturePro(tex, src, dest, (Vector2){0, 0}, 0.0f, tint);

    // 숫자 텍스트 중앙 오버레이
    char buf[2] = { '0' + apple->value, '\0' };
    int  tw     = MeasureText(buf, 20);
    DrawText(buf,
             (int)(x + CELL_SIZE / 2.0f - tw / 2.0f),
             (int)(y + CELL_SIZE / 2.0f - 10.0f),
             20, WHITE);
}
```

##### 색조(tint) 팔레트

```c
Color apple_color(int value) {
    // value 1~9 → 색상
    Color palette[9] = {
        { 220,  50,  50, 255 },  // 1 — 빨강
        { 230, 120,  30, 255 },  // 2 — 주황
        { 210, 190,  20, 255 },  // 3 — 노랑
        {  60, 180,  60, 255 },  // 4 — 초록
        {  30, 160, 210, 255 },  // 5 — 하늘
        {  50,  80, 200, 255 },  // 6 — 파랑
        { 140,  60, 200, 255 },  // 7 — 보라
        { 210,  80, 150, 255 },  // 8 — 핑크
        { 160, 100,  60, 255 },  // 9 — 갈색
    };
    return palette[value - 1];
}
```

##### Assets를 Game 구조체에 포함

```c
typedef struct {
    Board board;
    DragBox drag;
    float timeLeft;
    GameState state;
    bool allCleared;
    float clearTime;
    Assets assets;    // 추가
    bool thinColor;   // 연한 색 모드 토글
} Game;
```

##### 생명주기 관리

```c
// 시작 시
game->assets = assets_load();

// 종료 시 (CloseWindow 직전)
assets_unload(&game->assets);
```

### 드래그 박스 색상

```c
// 합 == 10 이면 빨간 테두리, 아니면 흰 테두리
Color box_color = (sum == 10) ? RED : WHITE;
DrawRectangleLinesEx(drag_rect, 2, box_color);
DrawRectangle(..., Fade(box_color, 0.15f));  // 반투명 내부
```

### 타이머 게이지 (우측 세로 바)

```c
float ratio = game->timeLeft / 120.0f;  // 0.0~1.0
int   barH  = (int)(400 * ratio);
DrawRectangle(980, 20 + (400 - barH), 20, barH, GREEN);
```

---

## 게임 루프 전체 흐름

```c
int main(void) {
    InitWindow(1010, 600, "Fruit Box");
    SetTargetFPS(60);

    Game game = {0};
    game_init(&game);  // 보드 초기화, state = STATE_MENU

    while (!WindowShouldClose()) {
        switch (game.state) {
            case STATE_MENU:
                if (IsKeyPressed(KEY_SPACE)) {
                    game_init(&game);
                    game.state = STATE_PLAYING;
                }
                break;
            case STATE_PLAYING:
                input_update(&game);
                timer_update(&game);
                check_all_cleared(&game);
                break;
            case STATE_GAMEOVER:
                if (IsKeyPressed(KEY_R)) game_init(&game);
                break;
        }

        BeginDrawing();
            render(&game);
        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

---

## 올 클리어 판정

```c
void check_all_cleared(Game *game) {
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            if (!game->board.grid[r][c].removed) return;

    game->allCleared = true;
    game->clearTime  = 120.0f - game->timeLeft;
    game->state      = STATE_GAMEOVER;
}
```

---

## 구현 단계 (추천 순서)

| 단계 | 작업 | 완료 기준 |
|------|------|-----------|
| 1 | Raylib 세팅 + 창 띄우기 | 빈 창이 뜨면 OK |
| 2 | 게임판 초기화 + 터미널 출력 | 17×10 숫자 배열 콘솔 확인 |
| 3 | 사과 렌더링 | 화면에 격자 표시 |
| 4 | 마우스 드래그 박스 표시 | 드래그 시 사각형 그려짐 |
| 5 | 합산 로직 + 색상 변경 | 합 10이면 박스 빨간색 |
| 6 | 사과 제거 + 점수 | 해제 시 사과 사라지고 점수 오름 |
| 7 | 타이머 + 게임오버 | 2분 후 화면 전환 |
| 8 | 메뉴 / 결과 화면 | R키로 재시작 |
| 9 | 연한 색 토글 버튼 | T키 또는 버튼 클릭으로 전환 |
| 10 | BGM / 효과음 (선택) | Raylib InitAudioDevice 활용 |

---

## 창 크기 레이아웃 (1010×600)

```
┌──────────────────────────────────────────┬──┐
│                                          │  │
│   게임판 (17×10, 셀 54px)                 │게│
│   918px × 540px                          │이│
│   BOARD_X=20, BOARD_Y=30                 │지│
│                                          │  │
│  점수: 000   [연한 색]                     │바│
└──────────────────────────────────────────┴──┘
 ←──────────────── 1010px ──────────────────→
```

---

## 주의사항 및 팁

- `rand()` 사용 전 `srand((unsigned)time(NULL))` 호출
- `GetFrameTime()`은 `SetTargetFPS(60)` 이후에 정확히 동작함
- 드래그 박스가 게임판 경계 밖으로 나가는 경우 클램핑 필수
- 이미 `removed == true`인 사과는 합산 및 렌더링에서 제외
- 올 클리어 보너스 점수는 원작에 없으므로 시간 기록으로 대체
