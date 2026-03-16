# Level 01 — Raylib 세팅 + 창 띄우기

## 목표
Visual Studio에서 Raylib를 연결하고, 빈 창이 정상적으로 열리는 것을 확인한다.

## 완료 기준
- 검은 배경의 빈 창이 뜬다
- 창 제목이 "Fruit Box"로 표시된다
- ESC 또는 X 버튼으로 종료된다

---

## 1단계 — Raylib 다운로드

1. 브라우저에서 https://github.com/raysan5/raylib/releases 접속
2. 최신 릴리즈에서 `raylib-5.x_win64_msvc16.zip` 파일 다운로드
   - 파일명 예시: `raylib-5.0_win64_msvc16.zip`
3. 압축 해제 → 원하는 폴더에 저장 (예: `C:\libs\raylib\`)

압축 해제 후 폴더 구조 확인:
```
C:\libs\raylib\
├── include\
│   └── raylib.h       ← 헤더 파일
├── lib\
│   └── raylib.lib     ← 링크 라이브러리
└── ...
```

> **구조 설명**
> - `include\raylib.h` : **함수 선언**이 들어있는 헤더 파일. `#include "raylib.h"`를 쓰면 이 파일을 읽는다. 레시피 목록처럼 "이런 함수가 있다"는 정보만 담김
> - `lib\raylib.lib` : **함수의 실제 구현(기계어)**이 들어있는 라이브러리 파일. 컴파일 마지막 단계(링크)에서 우리 코드와 합쳐진다
> - 이 두 파일의 위치를 Visual Studio에 알려주는 것이 3단계의 핵심

---

## 2단계 — Visual Studio 프로젝트 생성

1. Visual Studio 실행
2. **새 프로젝트 만들기** → **빈 프로젝트(C++)** 선택
   - 이름: `fruit_box`
   - 위치: `C:\jin_c\fruit_box\` (또는 원하는 경로)
3. 프로젝트 생성 완료 후 **솔루션 탐색기** 확인

---

## 3단계 — 프로젝트 속성 설정

메뉴 → **프로젝트** → **fruit_box 속성** 열기
상단 구성: **모든 구성**, 플랫폼: **x64** 로 설정 후 아래 항목 적용

### 3-1. 추가 포함 디렉터리 (헤더 경로)
```
구성 속성 → C/C++ → 일반 → 추가 포함 디렉터리
```
값 추가:
```
C:\libs\raylib\include
```

### 3-2. 추가 라이브러리 디렉터리
```
구성 속성 → 링커 → 일반 → 추가 라이브러리 디렉터리
```
값 추가:
```
C:\libs\raylib\lib
```

### 3-3. 추가 종속성 (링크할 라이브러리)
```
구성 속성 → 링커 → 입력 → 추가 종속성
```
맨 앞에 추가 (기존 항목 유지):
```
raylib.lib;winmm.lib;gdi32.lib;opengl32.lib;
```

> **설정 설명**
> - **추가 포함 디렉터리** : `#include "raylib.h"` 를 만났을 때 어느 폴더에서 찾을지 알려주는 설정. 설정 안 하면 `cannot open source file "raylib.h"` 오류 발생
> - **추가 라이브러리 디렉터리** : `.lib` 파일이 있는 폴더 위치. 설정 안 하면 링크 오류 발생
> - **추가 종속성** : 실제로 링크할 `.lib` 파일 이름 목록
>   - `raylib.lib` : Raylib 본체
>   - `winmm.lib` : Windows 멀티미디어 (오디오 등) — Raylib가 내부적으로 사용
>   - `gdi32.lib` : Windows 그래픽 장치 인터페이스 — Raylib가 내부적으로 사용
>   - `opengl32.lib` : OpenGL 렌더링 엔진 — Raylib가 내부적으로 사용

### 3-4. 서브시스템 설정 (콘솔 창 없애기, 선택)
```
구성 속성 → 링커 → 시스템 → 서브시스템
```
값: `콘솔(/SUBSYSTEM:CONSOLE)` 유지 (개발 중에는 콘솔 출력이 편함)

**적용** → **확인** 클릭

---

## 4단계 — 소스 파일 생성

솔루션 탐색기 → **소스 파일** 우클릭 → **추가** → **새 항목**
파일명: `main.c` (`.cpp` 가 아닌 `.c` 로 생성)

> **주의**: Visual Studio는 기본적으로 `.c` 파일도 C++로 컴파일할 수 있다.
> C로 컴파일하려면 파일 속성 → **C/C++ → 고급 → 컴파일 방식** → **C 코드로 컴파일(/TC)** 설정

---

## 5단계 — main.c 코드 작성

아래 코드를 `main.c`에 그대로 입력한다.

```c
#include "raylib.h"

int main(void)
{
    /* 창 초기화 */
    InitWindow(1010, 600, "Fruit Box");
    SetTargetFPS(60);

    /* 게임 루프 */
    while (!WindowShouldClose())
    {
        BeginDrawing();
            ClearBackground((Color){ 30, 30, 30, 255 });   /* 어두운 회색 배경 */
            DrawText("Fruit Box - Level 01 OK", 20, 20, 24, WHITE);
        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

> **코드 설명**
> - `#include "raylib.h"` : 꺾쇠(`<stdio.h>`)가 아닌 따옴표(`"raylib.h"`)를 쓴 이유 — 표준 라이브러리가 아니라 우리가 직접 경로를 알려준 헤더이기 때문. Visual Studio가 3단계에서 설정한 포함 디렉터리에서 찾는다
> - `InitWindow(1010, 600, "Fruit Box")` : OS에 1010×600 픽셀 창을 만들어달라고 요청. 이 함수를 호출하기 전에는 그리기 함수를 쓸 수 없다
> - `SetTargetFPS(60)` : 게임 루프가 1초에 최대 60번 돌도록 속도를 제한. 이보다 빠르게 돌아도 Raylib가 자동으로 잠깐 대기시켜 속도를 맞춰준다
> - `while (!WindowShouldClose())` : ESC 키를 누르거나 창의 X 버튼을 누르면 `WindowShouldClose()`가 `true`를 반환 → `!`(NOT)로 뒤집어서 루프를 탈출한다. **이것이 게임 루프의 핵심 구조**
> - `BeginDrawing()` / `EndDrawing()` : 이 두 함수 사이에서만 화면에 그림을 그릴 수 있다. **더블 버퍼링** 방식으로 화면 깜빡임을 방지 — `BeginDrawing`에서 뒷면 버퍼에 그린 후, `EndDrawing`에서 앞면과 교체
> - `ClearBackground((Color){ 30, 30, 30, 255 })` : 매 프레임 화면 전체를 지정한 색으로 덮어씌워 이전 프레임의 그림을 지운다. `Color`는 `{R, G, B, A}` 구조체이고, 숫자는 0~255 범위. 255는 완전 불투명
> - `DrawText("...", 20, 20, 24, WHITE)` : 좌측에서 20px, 상단에서 20px 위치에 폰트 크기 24로 텍스트를 그린다. `WHITE`는 Raylib가 미리 정의해둔 색상 상수
> - `CloseWindow()` : 창을 닫고 Raylib 내부 메모리를 모두 해제. 프로그램 종료 전 반드시 호출해야 메모리 누수를 막을 수 있다

---

## 6단계 — 빌드 및 실행

1. 메뉴 → **빌드** → **솔루션 빌드** (단축키: `Ctrl+Shift+B`)
2. 오류 없이 빌드 성공 확인
3. `F5` 또는 **디버그 → 디버깅 시작** 으로 실행

### 정상 결과
- 1010×600 크기의 창이 열린다
- 어두운 배경에 흰색으로 "Fruit Box - Level 01 OK" 텍스트가 보인다
- ESC를 누르거나 X 버튼을 누르면 창이 닫힌다

---

## 자주 발생하는 오류

| 오류 메시지 | 원인 | 해결 |
|------------|------|------|
| `cannot open source file "raylib.h"` | 헤더 경로 미설정 | 3-1 단계 재확인 |
| `LINK : fatal error LNK1181: raylib.lib 열 수 없음` | 라이브러리 경로 미설정 | 3-2 단계 재확인 |
| `LNK2019: 외부 기호 확인 안 됨 _winmm` | winmm.lib 누락 | 3-3 단계에 winmm.lib 추가 |
| 창은 뜨지만 바로 꺼짐 | 런타임 DLL 누락 | raylib DLL이 필요한 동적 빌드의 경우 — 정적 빌드(raylib.lib) 사용 중이면 DLL 불필요 |

---

## 다음 단계
Level 01 완료 후 → [dev_level02.md] 게임판 초기화 + 터미널 출력으로 진행
