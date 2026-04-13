---
title: WebRTC 빌드 시스템 (gn, ninja)
date: 2026-04-13 00:00:00 +0900
categories: [Blog, WebRTC]
tags: [Tech, WebRTC, gn, ninja]
pin: true
---

## WebRTC 라이브러리를 어떻게 가져오는가?

빌드 시스템(GN)과 소스(C++) 두 축이 있는데, **둘 다 "라이브러리 전체를 묶은 단일 객체"는 없다.** WebRTC는 수백 개의 세분화된 GN 타겟이고, 예제는 필요한 것만 골라 `deps`에 나열한다.

### 빌드 진입: `src/examples/BUILD.gn`

```python
rtc_executable("peerconnection_client") {
  sources = [ "peerconnection/client/conductor.cc", ... ]
  deps = [
    "../api:peer_connection_interface",           # 핵심 공개 API
    "../api:create_modular_peer_connection_factory",
    "../api:enable_media",                        # 코덱 레지스트리
    "../p2p:port_allocator",                      # ICE
    "../rtc_base:threading",
    ...                                           # 수십 개
  ]
  if (is_linux) {
    sources += [ "peerconnection/client/linux/main.cc", ... ]
  }
}
```

### deps는 는 뭐지?

#### 선 요약

`deps`는 "이 타겟과 나 사이의 모든 관계(헤더+순서+링크+권한)를 열어달라"는 선언. 링크는 그중 하나.

#### (1) 헤더를 찾을 수 있게 해줌 (컴파일 타임)

`deps`에 넣으면 `#include "api/peer_connection_interface.h"`가 컴파일된다. 빼면 **컴파일 자체가 실패.** 링크 이전 문제.

#### (2) 빌드 순서 강제

ninja 그래프에 "이 라이브러리가 먼저 빌드되어야 내가 컴파일됨"이란 edge를 추가.

#### (3) 링크

최종 실행파일에 `.a`/`.o`를 포함.

#### (4) 가시성 검증

`gn check`가 **"deps에 없는 타겟의 헤더를 몰래 include하지 않았나"** 를 검사. WebRTC가 수많은 세분화된 타겟으로 쪼개진 이유가 이것 — 레이어 경계를 빌드 시스템이 강제로 지켜줌.

#### 전이적 확장

A가 B를 deps, B가 C를 `public_deps`하면 **A는 C의 헤더도 자동으로 쓸 수 있다.** `peerconnection_client`가 수십 개만 적어도 실제로는 수백 개 라이브러리 위에서 빌드되는 이유.

```bash
gn desc out/Debug //examples:peerconnection_client deps        # 직접 deps
gn desc out/Debug //examples:peerconnection_client deps --all  # 전이적 전체
```

---

## GN = Generate Ninja

Google이 Chromium용으로 만든 **메타빌드 시스템**. CMake나 Bazel과 비슷한 위치.

- 입력: `BUILD.gn` (선언형 DSL, Python 유사 문법)
- 출력: **ninja 빌드 파일** (`build.ninja`)
- GN 자체는 컴파일하지 않음. "뭘 어떻게 빌드할지"만 기술

## ninja = 실행 엔진

- `build.ninja`만 보고 실행. 조건문도 변수도 거의 몰라도 됨
- 타임스탬프/해시 비교로 **변경된 부분만** 재빌드
- 파서가 극도로 단순해서 수만 개 타겟 스캔이 서브초 단위

### 핵심 명령

```bash
gn gen out/Debug --args='...'      # BUILD.gn → build.ninja 변환
gn desc out/Debug //path:target    # 타겟 정보 조회
gn check out/Debug //path:target   # include 규칙 검증
gn format BUILD.gn                 # 포맷팅

ninja -C out/Debug peerconnection_client
```

`BUILD.gn`을 수정하면 ninja가 실행 직전에 **자동으로 `gn gen`을 재호출**한다. 사용자는 `ninja`만 치면 됨.

---

## GN과 ninja의 역할 차이

두 도구는 **다른 시간대**에 산다.

```
[BUILD.gn 수정]
      ↓
  gn gen out/Debug      ← "구성 단계" · 느림 · 가끔 실행
      ↓
  out/Debug/build.ninja
      ↓
  ninja -C out/Debug    ← "실행 단계" · 빠름 · 매번 실행
      ↓
  .o, .a, 실행파일
```

### GN이 하는 일 (구성 단계)

- **조건 분기 해석**: `if (is_linux)`, `if (is_debug)` 같은 분기를 여기서 평가
- **`args.gn` 읽기**: `is_debug=true` 같은 플래그를 보고 어떤 타겟을 켤지 결정
- **toolchain 결정**: clang 경로, 컴파일 플래그, sysroot 확정
- **의존성 그래프 구축**: 수만 개 타겟 연결
- **템플릿 확장**: `rtc_library` 같은 매크로를 저수준 규칙으로 풀어냄
- **가시성/정합성 검사**
- **출력**: 평탄화된 ninja 파일. 더 이상 분기도 변수도 없음

### ninja가 하는 일 (실행 단계)

- 조건문 해석 못 함, 변수 치환 거의 없음
- 입력 파일의 타임스탬프/해시 비교 → 바뀌었는지 판단
- 바뀐 것의 다운스트림만 재빌드, 병렬로
- 명령 한 줄 한 줄은 GN이 이미 확정해준 것. ninja는 그냥 실행

### 왜 이렇게 나눴나

| 문제 | Make 혼자일 때 | GN + ninja |
|------|----------------|------------|
| 조건 분기 많은 거대 빌드 | Makefile 파싱만 수 초 | 조건은 `gn gen` 때 1회 해석 |
| 증분 빌드 정확성 | 헤더 의존성 추적 미묘함 | GN이 모든 전이 의존성을 펼쳐 ninja에 명시 |
| 병렬성 | `-j` 되지만 스케줄러가 덜 정교 | ninja가 I/O·프로세스 기동 자체가 빠름 |
| 플랫폼 추상화 | `$(if ...)` 범벅 | BUILD.gn이 깔끔한 DSL로 흡수 |

---

## 컴파일러 경로는 코드 어디에 있나?

실제 소스 3곳에 분산돼 있다.

### 1. `src/build/config/clang/clang.gni:30` — 절대경로 한 줄

```python
default_clang_base_path = "//third_party/llvm-build/Release+Asserts"
```

여기가 **clang 위치를 실제로 글자로 적어둔 곳.**

### 2. `src/build/toolchain/gcc_toolchain.gni:918,928-929` — 경로 조립

```python
_path = "$clang_base_path/bin"
...
cc  = "${prefix}/clang"
cxx = "${prefix}/clang++"
```

최종 실행파일 경로 완성: `third_party/llvm-build/Release+Asserts/bin/clang++`

이 `.gni` 파일은 `clang_toolchain(...)` **템플릿**을 정의한다.

### 3. `src/build/toolchain/linux/BUILD.gn` — 템플릿을 실제 호출

```python
import("//build/toolchain/gcc_toolchain.gni")

clang_toolchain("clang_x64") { ... }
clang_toolchain("clang_arm64") { ... }
```

### 호출 체인

```
clang.gni:30 (default_clang_base_path)
        ↓
gcc_toolchain.gni:928 (cxx = ".../bin/clang++")
        ↓
linux/BUILD.gn (clang_toolchain("clang_x64") 호출)
        ↓  (gn gen)
out/Debug/toolchain.ninja (명령어로 박힘)
        ↓  (ninja)
실제 clang++ 프로세스 기동
```

"컴파일러 지정"은 결국 `cc`/`cxx`라는 GN 변수에 문자열을 담는 일. Clang이든 gcc든 MSVC든 같은 메커니즘.

---

## Google 번들 clang vs 시스템 clang

**빌드에 쓰이는 건 Google이 소스트리에 넣어둔 번들 clang이다. 시스템에 깔린 clang이 아니다.**

### 증거

| 항목 | Google 번들 | 시스템 설치분 |
|---|---|---|
| 경로 | `src/third_party/llvm-build/Release+Asserts/bin/clang++` | `/usr/bin/clang++` |
| 버전 | clang 23.0.0git (LLVM main 스냅샷) | Ubuntu clang 18.1.3 |
| 빌드 주체 | Google이 Chromium fork에서 빌드 | Ubuntu 패키지 관리자 |

버전이 5개 릴리스나 차이 난다. **완전히 다른 바이너리.**

### 왜 번들하나 — 재현성(Hermetic Build)

1. **Chromium 전용 플러그인 포함**: `find-bad-constructs` 같은 정적 분석 플러그인 내장
2. **LLVM main 스냅샷**: 정식 릴리스가 아니라 특정 커밋(`8a0be0bc...`)에서 빌드
3. **모든 개발자가 동일 결과**: 배포판 달라도 바이트 단위 동일 바이너리
4. **Sanitizer 런타임 동봉**: ASan/TSan/UBSan 전용 런타임 내장

### sysroot도 번들

`/usr/include`, `/usr/lib`도 안 쓴다. `build/linux/debian_*_sysroot/`의 Debian sysroot를 쓴다. Ubuntu 22든 24든 Fedora든 결과가 동일.

### fetch 16GB 안에 들어있는 것

- 컴파일러 (clang, lld)
- 표준 라이브러리 (libc++, libc++abi)
- sysroot (libc, 리눅스 헤더)
- 모든 서드파티 (abseil, BoringSSL, libvpx, opus, ...)

호스트 머신은 사실상 커널과 `bash`/`python3`만 제공하면 된다. 이게 **hermetic build** 철학.

---

## gdb란?

**GNU Debugger. 실행 중인 프로그램을 멈추고 들여다보는 도구.**

### 하는 일

- 중단점(breakpoint): 특정 줄/함수에서 프로그램 정지
- 변수 조회: 정지 상태에서 변수·메모리·레지스터 값 확인
- 스텝 실행: 한 줄씩, 함수 안으로/밖으로
- 콜 스택 보기: 지금 이 함수가 누구한테서 불렸나
- 크래시 분석: 코어 덤프 사후 분석
- 실행 중 프로세스에 붙기(attach)

### 왜 Debug 빌드가 필요한가

| 플래그 | 효과 | gdb에서 뭐가 달라지나 |
|---|---|---|
| `symbol_level=2` | 디버그 심볼 삽입 | 소스 라인에 중단점 가능, 변수 이름으로 조회 |
| `is_debug=true` | 최적화 off (`-O0`) | 변수가 레지스터로 사라지지 않음 |

Release 빌드로 gdb를 붙이면: `"No symbols loaded"`, `<optimized out>`, 줄 번호 뒤죽박죽.

### 기본 사용 예

```bash
gdb ./out/Debug/peerconnection_client
(gdb) break Conductor::InitializePeerConnection
(gdb) run
(gdb) bt                # 콜 스택
(gdb) print *peer_connection_
(gdb) step              # 함수 안으로
(gdb) next              # 함수 건너뜀
(gdb) continue
```

### VS Code와의 관계

`.vscode/launch.json`의 `"MIMode": "gdb"` — VS Code 디버거는 **gdb 위에 씌운 GUI**. 내부적으로 gdb 프로세스를 띄우고 MI(Machine Interface) 프로토콜로 명령 주고받음. F5 누르면 VS Code가 뒤에서 gdb 조종.

### 경쟁자

| 도구 | 특징 |
|---|---|
| gdb | Linux 표준, GNU, 거의 모든 곳 |
| lldb | LLVM 쪽, macOS 기본 |
| WinDbg / VS | Windows용 |

### 서버 개발자 실용 팁

- **프로덕션 크래시**: `objcopy --only-keep-debug`로 심볼만 별도 파일로 빼두면 Release 바이너리의 크래시도 추적 가능
- **코어 덤프**: `ulimit -c unlimited` → `gdb ./binary core`로 죽기 직전 상태 분석
- **동시성 버그**: `thread apply all bt` — 모든 스레드 스택 한 번에. 데드락 추적 필수

---

## Google 번들 gdb는 왜 없나?

**gdb 자체는 번들 안 한다. 시스템 gdb 그대로 쓴다.** 하지만 gdb용 **플러그인·스크립트는 제공**한다.

### `src/tools/gdb/`

```
src/tools/gdb/
├── gdbinit               ← gdb 시작 설정
├── gdb_chrome.py         ← Chromium/WebRTC 타입용 pretty-printer
├── absl_printers.py      ← abseil 타입용
└── ipcz_printers.py
```

### 왜 컴파일러와 디버거를 다르게 취급?

| 항목 | 번들? | 이유 |
|---|---|---|
| clang | O | 빌드 결과물이 비트 단위로 같아야 함 |
| lld | O | 빌드 결과물의 일부 |
| gdb | X | **관찰만** 하니 바이너리에 영향 없음 |
| Python | X | 실행 도구는 재현성 요구 약함 |

원칙: **"최종 산출물에 영향을 주는 도구만 번들한다."**

### Pretty-printer가 뭐길래

그냥 gdb로 WebRTC를 디버깅하면:

```
(gdb) print my_string
$1 = {static npos = ..., _M_dataplus = {... _M_p = 0x555555c8a040 "hello"}, ...}
```

pretty-printer 로드 후:

```
(gdb) print my_string
$1 = "hello"
```

`absl::optional`, `absl::Span`, `scoped_refptr`, `rtc::RefCountedObject` 같은 타입을 이쁘게 찍어준다.

### 활성화

```
source /home/th77/webrtc/src/tools/gdb/gdbinit
```

를 `~/.gdbinit`에 추가.

---

## lld란?

**LLVM Linker. clang과 한 세트인 링커.**

### 링커의 역할

컴파일은 `.cc` → `.o`까지다. `.o` 파일은:
- 서로의 함수 주소를 모름 (심볼 미결정)
- 라이브러리와 합쳐지지 않음
- 실행 가능한 형식이 아님

링커가 하는 일:
1. 수백~수만 개 `.o`/`.a`를 모아 **심볼 해결**
2. 섹션 병합 (모든 `.text` 합치고, `.data` 합치고...)
3. **재배치** (최종 주소 기준 모든 참조 재계산)
4. 동적 라이브러리 링크 정보 삽입
5. **최종 ELF/Mach-O/PE 바이너리 출력**

컴파일이 벽돌 굽기라면, 링크는 벽돌로 집 짓기.

### 링커 비교

| 링커 | 출신 | 특징 |
|---|---|---|
| ld (BFD) | GNU, 1980s | 표준, 느림 |
| gold | Google, 2008 | ELF 전용, 빠름 |
| **lld** | **LLVM, 2017~** | **플랫폼 독립(ELF/Mach-O/PE), 가장 빠름, 멀티스레드** |
| mold | 2021 | lld보다 더 빠름, 급부상 |

### 왜 WebRTC가 lld를 쓰나

Chromium급 규모면 링커가 병목. 과거 ld로는 링크만 몇 분 걸렸는데 lld는 수초~십여 초.

### 번들 위치

```
src/third_party/llvm-build/Release+Asserts/bin/
  ├── clang, clang++
  ├── lld              ← 범용
  ├── ld.lld           ← ELF(Linux)
  ├── ld64.lld         ← Mach-O(macOS)
  └── lld-link         ← PE(Windows)
```

하나의 lld 바이너리가 플랫폼별 이름으로 심볼릭 링크. clang이 링크 단계에서 `ld.lld`를 호출.

### ninja에서

```
command = ... clang++ ${ldflags} -o ...
```

`clang++`은 링커가 아니라 **링커를 호출하는 드라이버**. `${ldflags}` 안의 `-fuse-ld=lld`로 `ld.lld` 실행.

### lld의 기능

- **ThinLTO**: 파일 간 최적화(LTO)를 병렬/증분으로
- **Identical Code Folding (ICF)**: 바이트 동일한 함수 합쳐 바이너리 축소
- **크로스 컴파일**: Linux에서 Windows 바이너리 링크 가능
- **재현성**: 타임스탬프 삽입 등 비결정적 요소 없음

### 왜 gdb는 번들 안 하면서 lld는 번들하나

lld는 **빌드 결과물의 일부**라서 재현성에 직접 영향. gdb는 관찰 도구라 바이너리에 영향 없음. 그래서 lld까지 번들, gdb는 배제.
