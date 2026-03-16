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

> **구조 설명**
> - 각 `.h/.c` 쌍은 하나의 역할만 담당. C 언어에서 파일을 역할별로 나누는 것이 관례
> - `main.c` : 게임 루프만 담당. 비즈니스 로직은 각 모듈에 위임
> - `board.h/.c` : 데이터 구조(Apple, Board)와 그 조작 함수
> - `input.h/.c` : 마우스 입력 처리
> - `render.h/.c` : 화면 그리기. 여기에 `CELL_SIZE`, `BOARD_X`, `BOARD_Y` 레이아웃 상수가 정의됨
> - `timer.h/.c` : 타이머 및 올 클리어 판정
> - `game.h/.c` : 위 모듈들을 연결하는 중심. `Game` 구조체와 상태 전환 로직

---

## 데이터 구조 설계

### 사과 하나

```c
typedef struct {
    int value;      // 1~9
    bool removed;   // 제거 여부
} Apple;
```

> **코드 설명**
> - `int value` : 사과에 적힌 숫자. 1~9 범위
> - `bool removed` : 사과를 실제로 배열에서 삭제하지 않고, 이 플래그로 "제거됨"을 표시. 렌더링과 합산 모두 이 값을 보고 무시할지 결정

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

> **코드 설명**
> - `Apple grid[ROWS][COLS]` : 2차원 배열. `grid[행][열]`로 접근. 총 10×17 = 170개의 Apple
> - `int score` : Board에 포함되어 있지만, Level 04에서 Game 구조체로 이동됨. 최종 설계에서는 `Game.score`를 사용

### 드래그 선택 박스

```c
typedef struct {
    Vector2 start;   // 드래그 시작 좌표 (픽셀)
    Vector2 end;     // 드래그 현재 좌표 (픽셀)
    bool active;     // 드래그 중 여부
} DragBox;
```

> **코드 설명**
> - `Vector2` : Raylib가 정의한 `{float x, float y}` 구조체. 픽셀 좌표를 표현
> - `bool active` : 드래그 중이 아닐 때 박스를 그리거나 합산하지 않기 위한 플래그
> - `start`와 `end`를 분리 보관하는 이유: 드래그 방향(좌→우, 우→좌 등)을 후처리(`drag_to_rect`)에서 정규화하기 위해

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

> **코드 설명**
> - `typedef enum { ... } GameState` : 열거형. 게임이 어느 상태에 있는지를 이름 있는 정수로 표현. `STATE_MENU=0`, `STATE_PLAYING=1`, `STATE_GAMEOVER=2`로 내부 저장됨
> - **상태 머신(State Machine)**: main.c의 `switch(game.state)`에서 각 상태에 맞는 로직만 실행. 상태 간 전환은 명확한 조건(SPACE 누름, 시간 종료, 올 클리어)에서만 발생
> - `float timeLeft` : 초 단위. 매 프레임 `GetFrameTime()`(이전 프레임 경과 시간)만큼 감소
> - `float clearTime` : 게임오버 화면에서 "X초 만에 클리어" 표시용. `120.0f - timeLeft`로 계산

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

> **코드 설명**
> - `do { ... } while(조건)` : 한 번은 반드시 실행하고, 조건이 맞을 때까지 반복. 조건을 만족하는 배치가 나올 때까지 170개 숫자를 계속 새로 생성
> - `rand() % 9 + 1` : 0~8 범위를 1~9로 이동. `rand()` 사용 전 `srand(time(NULL))` 호출 필수
> - **Fisher-Yates 셔플**: 뒤에서부터 앞으로 이동하며 무작위 위치와 교환. 모든 순열이 동일한 확률로 나오는 것이 수학적으로 보장된 완전 셔플 알고리즘
> - `values[r * COLS + c]` : 1차원 배열을 2차원 격자로 읽기. r행 c열 = `r * 17 + c` 번 인덱스

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

> **코드 설명**
> - `(x - BOARD_X) / CELL_SIZE` : 게임판 시작점(BOARD_X)으로부터의 거리를 셀 크기로 나누면 열 인덱스가 나온다. 예: x=128 → (128-20)/54 = 2.0 → 열 2
> - `(int)` 캐스팅: 소수점을 버려 정수 인덱스를 얻는다. 이 방식은 경계에 주의해야 한다. 실제 구현(board_sum_in_rect)에서는 셀 중심 좌표와 비교하는 방식을 채택

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

> **코드 설명**
> - `fminf(a, b)` / `fmaxf(a, b)` : float용 최솟값/최댓값 함수 (`<math.h>` 필요). 정수용은 `min/max`이지만 C 표준에는 없어서 직접 구현하거나 삼항 연산자를 쓰는 경우도 많다
> - `fmaxf(0, pixel_to_col(x0))` : 열 인덱스가 0 미만이 되지 않도록 클램핑(경계 제한). 게임판 밖으로 드래그했을 때 배열 범위를 벗어나지 않도록 보호
> - `fminf(COLS - 1, ...)` : 열 인덱스가 16(최대)을 넘지 않도록 클램핑
> - **주의**: 이 설계는 dev_level05에서 "셀 중심 기반" 방식으로 변경됨. 위 코드는 개념 이해용

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

> **코드 설명**
> - `if (!game->drag.active) return;` : 방어 코드. 드래그 중이 아닌데 호출됐을 때 즉시 종료
> - 합산과 제거를 **같은 범위 순회 로직**으로 처리하는 것이 핵심. 로직이 다르면 "합은 10인데 다른 사과 제거" 버그 발생

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

> **코드 설명**
> - `IsMouseButtonPressed` : 클릭 첫 프레임만. 드래그 시작점 기록
> - `IsMouseButtonDown` : 누르는 동안 매 프레임. 끝점을 현재 마우스 위치로 갱신
> - `IsMouseButtonReleased` : 놓는 첫 프레임만. 사과 제거 시도
> - 실제 구현(Level 06)에서는 `released` 처리를 main.c로 이동 — `game_try_remove`가 `drag.active = false`를 내부에서 처리하기 때문

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

> **코드 설명**
> - `GetFrameTime()` : 이전 프레임이 처리된 실제 시간(초). 60FPS에서 약 0.0167초. 이 값을 누적해서 빼면 프레임 수와 무관하게 정확한 초 단위 카운트다운이 된다
> - `game->timeLeft = 0.0f` : 음수 방지. 게임오버 화면에서 남은 시간이 음수로 표시되지 않도록

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

> **렌더링 순서 설명**
> - 나중에 그릴수록 화면 앞쪽에 표시된다. 드래그 박스가 사과 위에 보이려면 `draw_board` 이후에 `draw_drag_box`를 호출해야 한다
> - 게임오버 오버레이가 가장 마지막에 그려져야 모든 것을 덮는다

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

> **코드 설명**
> - `CELL_SIZE / 2.0f` : 셀의 중심 좌표. col * CELL_SIZE는 셀 왼쪽 끝, 여기에 반 칸을 더하면 중심
> - `Fade(base, 0.5f)` : Raylib 함수. 색상의 알파(투명도)를 50%로 줄여 연한 색 효과. dev_level03에서는 RGB 직접 조정 방식을 사용하는데, Fade 방식과 다른 결과가 나올 수 있다
> - `char buf[2] = { '0' + apple->value, '\0' }` : 정수 → 문자 변환. '0'(48) + 3 = '3'(51)

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

> **코드 설명**
> - `Texture2D` : Raylib가 정의한 GPU 텍스처 타입. `LoadTexture`로 PNG 파일을 GPU 메모리에 올린다
> - `LoadTexture` / `UnloadTexture` : 게임 시작 시 1회 로드, 종료 시 1회 해제. 매 프레임 로드하면 메모리 누수와 심각한 성능 저하
> - `assets_unload`를 `CloseWindow()` 전에 호출해야 GPU 메모리 해제가 정상 처리됨

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

> **코드 설명**
> - `Color tint` : **색조(tint)**. 이미지의 각 픽셀 색상에 tint 색을 곱해서 색을 바꾼다. 흰색 이미지라면 tint 색 그대로 표시됨. 사과 이미지 1장으로 9가지 색을 표현하는 핵심
> - `Rectangle src` : 원본 이미지에서 잘라낼 영역. 전체 이미지를 쓰므로 `{0, 0, width, height}`
> - `Rectangle dest` : 화면에 그릴 목적지 영역. `{x, y, CELL_SIZE, CELL_SIZE}`로 이미지를 54×54로 스케일
> - `DrawTexturePro(tex, src, dest, origin, rotation, tint)` : 원본 영역을 목적지 영역에 그리는 함수. `origin`은 회전 기준점(좌상단), `rotation`은 각도(0=회전 없음)

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

> **코드 설명**
> - `palette[value - 1]` : value가 1~9이고 배열 인덱스는 0~8이므로 -1 보정
> - `Color` 는 `{R, G, B, A}` 각 0~255. 255가 최대(완전 불투명)

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

> **코드 설명**
> - **생명주기(lifecycle)**: GPU 메모리를 할당(Load)하면 반드시 해제(Unload)해야 한다. `CloseWindow()` 이후에 해제하면 이미 그래픽 시스템이 종료되어 문제가 생길 수 있으므로 그 전에 호출

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

> **코드 설명**
> - `ratio = timeLeft / 120.0f` : 남은 시간을 0.0~1.0 비율로 정규화
> - `20 + (400 - barH)` : 게이지가 아래에서 위로 줄어드는 효과. `barH`가 줄어들수록 y 시작점이 내려가고 높이도 줄어든다

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

> **코드 설명**
> - `Game game = {0}` : 구조체 전체를 0으로 초기화. C에서 구조체를 선언만 하면 쓰레기값이 들어있으므로, game_init 호출 전 안전하게 초기화
> - `switch(game.state)` : 상태 머신의 핵심. 각 case는 해당 상태에서만 실행되는 로직
> - `render(&game)` : 실제 구현에서는 여러 draw_ 함수 호출로 분리됨. 이 계획서에서는 개념 설명을 위해 단순화

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

> **코드 설명**
> - **조기 탈출(early return)** 패턴: 제거되지 않은 사과가 하나라도 있으면 즉시 `return`. 함수 끝까지 도달 = 전부 제거됨 = 올 클리어. 170개를 전부 체크하는 것보다 빠른 평균 성능
> - `clearTime = 120.0f - game->timeLeft` : 소요 시간 계산

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

> **레이아웃 설명**
> - 게임판: 17×54 = 918px 너비, 10×54 = 540px 높이. BOARD_X=20 좌측 여백
> - 우측 게이지 바: 960~982px 범위 (22px 너비)
> - 하단 여백: 게임판 아래(570px~600px)에 점수와 버튼 배치

---

## 주의사항 및 팁

- `rand()` 사용 전 `srand((unsigned)time(NULL))` 호출
- `GetFrameTime()`은 `SetTargetFPS(60)` 이후에 정확히 동작함
- 드래그 박스가 게임판 경계 밖으로 나가는 경우 클램핑 필수
- 이미 `removed == true`인 사과는 합산 및 렌더링에서 제외
- 올 클리어 보너스 점수는 원작에 없으므로 시간 기록으로 대체
