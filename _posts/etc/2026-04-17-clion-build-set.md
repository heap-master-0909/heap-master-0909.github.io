---
title: 개발 환경 세팅 (clion build setting on mac)
date: 2026-04-12 00:00:00 +0900
categories: [Etc, clion-build-setting]
tags: [Tech, Etc, clion-build-setting]
pin: true
---

# CLion에서 WebRTC 빌드 & 디버깅 설정

## 1. CLion 프로젝트 열기

WebRTC는 GN/Ninja 빌드 시스템을 사용하므로, CLion에 **Compilation Database** 방식으로 연동한다.

1. CLion 실행
2. `File > Open`
3. `/Users/admin/webrtc/src/out/Debug/compile_commands.json` 선택
4. **"Open as Project"** 클릭

> `compile_commands.json`이 없거나 오래됐다면 아래 명령으로 재생성한다.
> ```bash
> cd ~/webrtc/src
> export PATH="$HOME/depot_tools:$PATH"
> gn gen out/Debug --args='
>   target_os="mac"
>   target_cpu="arm64"
>   is_debug=true
>   symbol_level=2
>   enable_dsyms=true
>   is_component_build=false
>   rtc_include_tests=false
>   use_rtti=true
> '
> ```

---

## 2. Compilation Database 재로드

소스 파일을 새로 추가했거나 `compile_commands.json`을 재생성한 경우:

`Tools > Compilation Database > Reload Compilation Database Project`

---

## 3. Custom Build Target 설정

CLion이 직접 빌드하는 대신 **ninja를 호출**하도록 설정한다.

### 3-1. External Tool 등록

`Settings > Tools > External Tools > +`

| 항목 | 값 |
|------|-----|
| Name | `ninja-webrtc` |
| Program | `/Users/admin/depot_tools/ninja` |
| Arguments | `-C /Users/admin/webrtc/src/out/Debug loopback_dc` |
| Working directory | `/Users/admin/webrtc/src` |

> `loopback_dc` 부분을 빌드하려는 타겟 이름으로 교체한다.

### 3-2. Custom Build Target 등록

`Settings > Build, Execution, Deployment > Custom Build Targets > +`

| 항목 | 값 |
|------|-----|
| Name | `WebRTC Debug` |
| Build | `ninja-webrtc` (위에서 만든 External Tool) |
| Clean | (선택사항) |

---

## 4. Run/Debug Configuration 설정

`Run > Edit Configurations > + > Custom Build Application`

| 항목 | 값 |
|------|-----|
| Name | `loopback_dc` |
| Target | `WebRTC Debug` |
| Executable | `/Users/admin/webrtc/src/out/Debug/loopback_dc` |
| Program arguments | (없음) |
| Working directory | `/Users/admin/webrtc/src` |

---

## 5. 디버거 설정

`Settings > Build, Execution, Deployment > Toolchains`

- Debugger: **Bundled LLDB** 선택
- GDB는 Apple Silicon에서 동작하지 않으므로 반드시 LLDB를 선택한다.

---

## 6. 디버그 실행

`Shift + F9` 또는 `Run > Debug 'loopback_dc'`

---

## 7. 추천 브레이크포인트

| 파일 | 라인 | 의미 |
|------|------|------|
| `examples/loopback_dc/loopback_dc.cc` | `OnStateChange` 함수 | DataChannel 상태 변화 |
| `examples/loopback_dc/loopback_dc.cc` | `OnMessage` 함수 | 메시지 수신 확인 |
| `examples/loopback_dc/loopback_dc.cc` | `OnIceCandidate` 함수 | ICE 후보 내용 확인 |
| `examples/loopback_dc/loopback_dc.cc` | `CreateSdpObserver::OnSuccess` | SDP 내용 확인 |
| `pc/peer_connection.cc` | 관심 있는 라인 | WebRTC 내부 로직 추적 |

---

## 8. 주의사항

### PeerConnection API 스레드 규칙
WebRTC의 `PeerConnectionInterface` 메서드는 반드시 **signaling thread**에서 호출해야 한다.
그렇지 않으면 크래시가 발생한다.

```cpp
// 올바른 방법
signaling_thread->BlockingCall([&] {
    caller_pc->CreateOffer(...);
    caller_pc->SetLocalDescription(...);
});

// 잘못된 방법 (main thread에서 직접 호출)
caller_pc->CreateOffer(...);  // ← 크래시 위험
```

### DataChannel 생성 방식
`negotiated=true` + 동일한 `id`로 양쪽에서 생성하면 `OnDataChannel` 콜백 없이도 동작한다.

```cpp
webrtc::DataChannelInit dc_init;
dc_init.negotiated = true;
dc_init.id = 2;

auto caller_dc = caller_pc->CreateDataChannelOrError("chat", &dc_init);
auto callee_dc = callee_pc->CreateDataChannelOrError("chat", &dc_init);
```

### compile_commands.json이 최신이어야 한다
새 소스 파일 추가 후 CLion이 인식 못 하면 `gn gen`을 다시 실행하고
`Tools > Compilation Database > Reload Compilation Database Project`를 실행한다.

