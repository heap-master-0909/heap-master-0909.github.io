---
title: mediasoup build & test 방법
date: 2026-04-17 00:00:00 +0900
categories: [Etc, mediasoup]
tags: [Tech, Etc, mediasoup]
pin: true
---

## 1. Node.js 설치 (nvm)

```bash
# nvm 설치 (sudo 불필요)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# 쉘 재로드
source ~/.zshrc

# Node.js 22 설치 (mediasoup-demo 요구사항 >=22)
nvm install 22
nvm use 22
```

---

## 2. mediasoup-demo 클론 및 빌드

```bash
cd ~/git
git clone https://github.com/versatica/mediasoup-demo.git

# 서버 의존성 설치
cd mediasoup-demo/server
npm install

# 앱 의존성 설치 (peer dependency 충돌로 --legacy-peer-deps 필요)
cd ../app
npm install --legacy-peer-deps
```

---

## 3. TLS 인증서 생성

mediasoup은 WebRTC 특성상 HTTPS 필수. 로컬 테스트용 자체 서명 인증서를 생성한다.

```bash
mkdir -p ~/git/mediasoup-demo/server/certs

openssl req -x509 -newkey rsa:2048 \
  -keyout ~/git/mediasoup-demo/server/certs/key.pem \
  -out ~/git/mediasoup-demo/server/certs/fullchain.pem \
  -days 365 -nodes -subj "/CN=localhost"
```

---

## 4. 서버 설정 (config.mjs)

```bash
cp ~/git/mediasoup-demo/server/config.example.mjs \
   ~/git/mediasoup-demo/server/config.mjs
```

`config.mjs`에서 수정할 항목:

```js
// TLS 인증서 경로
tls: {
    cert: '/Users/admin/git/mediasoup-demo/server/certs/fullchain.pem',
    key:  '/Users/admin/git/mediasoup-demo/server/certs/key.pem',
},

// 디버깅 시 worker 1개로 줄이기 (기본값: os.cpus().length)
numWorkers: 1,
```

> **numWorkers를 1로 줄이는 이유:** 기본값은 CPU 코어 수(M3 기준 12개)만큼 worker 프로세스를 생성한다.
> 디버거를 잘못된 worker에 attach하면 브레이크포인트가 걸리지 않으므로, 디버깅 시에는 1개로 설정한다.

---

## 5. 서버 및 앱 실행

```bash
# 터미널 1 - 서버
export NVM_DIR="$HOME/.nvm" && . "$NVM_DIR/nvm.sh"
cd ~/git/mediasoup-demo/server
DEBUG="mediasoup-demo-server*" node bin/mediasoup-demo-server

# 터미널 2 - 프론트엔드
export NVM_DIR="$HOME/.nvm" && . "$NVM_DIR/nvm.sh"
cd ~/git/mediasoup-demo/app
npm start
```

브라우저에서 **`https://localhost:5555`** 접속.
자체 서명 인증서 경고 → "고급 → 계속 진행" 클릭.

---

## 6. C++ Worker 디버그 빌드

mediasoup worker는 Node.js가 별도 C++ 프로세스로 spawn한다.
C++ 레벨 디버깅을 위해 디버그 심볼 포함 빌드가 필요하다.

```bash
# 빌드 도구 설치
pip3 install meson ninja invoke

# Debug 빌드
cd ~/git/mediasoup-demo/server/node_modules/mediasoup/worker
MEDIASOUP_BUILDTYPE=Debug python3 -m invoke -r . mediasoup-worker

# 결과물 확인
ls -lh out/Debug/mediasoup-worker
```

### Debug 바이너리로 서버 실행

```bash
cd ~/git/mediasoup-demo/server
MEDIASOUP_WORKER_BIN=./node_modules/mediasoup/worker/out/Debug/mediasoup-worker \
DEBUG="mediasoup-demo-server*" \
node bin/mediasoup-demo-server
```

---

## 7. CLion에서 C++ 디버깅

### 프로젝트 열기

`compile_commands.json`을 프로젝트로 import해야 소스 파일이 타겟으로 인식된다.

```
File → Open →
~/git/mediasoup-demo/server/node_modules/mediasoup/worker/out/Debug/build/compile_commands.json
→ "Open as Project" 선택
```

> 일반 디렉토리로 열면 "This file does not belong to any project target" 오류가 뜨며 브레이크포인트가 동작하지 않는다.

### 프로세스 Attach

1. 서버를 Debug 바이너리로 먼저 실행
2. CLion → `Run → Attach to Process`
3. `mediasoup-worker` 선택 (numWorkers=1이면 하나만 뜸)
4. **브라우저에서 접속 전에** attach 완료해야 함

### 브레이크포인트 추천 위치

| 파일 | 위치 | 트리거 시점 |
|------|------|------------|
| `src/RTC/Router.cpp` | `Router::Router()` | 브라우저가 방 생성 시 |
| `src/RTC/WebRtcTransport.cpp` | `WebRtcTransport::WebRtcTransport()` | WebRTC transport 생성 시 |
| `src/RTC/Producer.cpp` | `Producer::ReceiveRtpPacket()` | 카메라/마이크 미디어 수신 시 |

---

## 참고: mediasoup 아키텍처

```
브라우저
  ↕ HTTPS / WebSocket (Signaling)
Node.js 서버 (mediasoup-demo-server)  ← JavaScript 레벨
  ↕ IPC (Unix socket)
mediasoup-worker (C++ 프로세스)        ← C++ 레벨 (CLion으로 디버깅)
  ↕ UDP/TCP
브라우저 (RTP/SRTP 미디어)
```

WebRTC 내부 구현(`deps/libwebrtc`)은 정적 라이브러리로 번들되어 소스가 없다.
내부까지 디버깅하려면 [WebRTC 소스](https://webrtc.googlesource.com/src)를 별도로 빌드해야 한다 (`depot_tools` 필요).
