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
