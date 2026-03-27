---
title: WebRTC 기술 로드맵
date: 2026-03-22 00:00:00 +0900
categories: [Blog, WebRTC]
tags: [Tech, WebRTC, RoadMap]
pin: true
---

## Phase 1: 네트워크 & 실시간 통신 기초 

실시간 미디어 전송의 토대가 되는 네트워크 지식을 먼저 잡는다.

| 주제 | 핵심 내용 | 학습 자료 |
|------|-----------|-----------|
| **UDP / TCP 차이** | 왜 실시간 통신에 UDP를 쓰는지, Head-of-line blocking 문제 | Stevens의 *Unix Network Programming* |
| **RTP / RTCP** | 미디어 패킷 구조, Sequence Number, Timestamp, SSRC, RTCP SR/RR | RFC 3550 정독 |
| **SRTP** | 미디어 암호화, DTLS-SRTP 핸드셰이크 | RFC 3711 |
| **NAT Traversal** | STUN / TURN / ICE 프로토콜, Candidate Gathering, Connectivity Check | RFC 5389, 5766, 8445 |
| **SDP** | Offer/Answer 모델, 미디어 라인 구조, codec negotiation | RFC 4566, 3264 |

### 실습

- C++로 간단한 UDP 소켓 서버/클라이언트를 만들어 RTP 패킷을 직접 파싱해보기

---

## Phase 2: WebRTC 핵심 이해

### 2-1. WebRTC 아키텍처 전체 그림

- **Signaling** → **ICE** → **DTLS** → **SRTP** → 미디어 전송 흐름
- P2P vs SFU vs MCU 토폴로지 차이와 각각의 트레이드오프
- **SFU(Selective Forwarding Unit)** — 대규모 서비스는 거의 확실히 SFU 기반

### 2-2. libwebrtc (Google WebRTC Native)

- Google의 WebRTC 네이티브 라이브러리 빌드 및 구조 파악
- `PeerConnection`, `MediaStream`, `RtpTransceiver` 등 핵심 API
- 소스코드 읽기: `webrtc/pc/`, `webrtc/media/`, `webrtc/modules/rtp_rtcp/`

### 2-3. 오픈소스 미디어 서버 분석

| 프로젝트 | 언어 | 특징 |
|----------|------|------|
| **mediasoup** | C++ (worker) + Node.js (signaling) |  |
| **Janus Gateway** | C | 플러그인 아키텍처, SFU/MCU 모두 지원 |
| **Pion** | Go | 가볍고 읽기 쉬움, WebRTC 개념 학습에 좋음 |

> **핵심 추천**: `mediasoup`을 깊이 분석할 것. 

---

## Phase 3: 오디오/비디오 미디어 기초 

### 3-1. 오디오

| 주제 | 내용 |
|------|------|
| **코덱** | Opus (필수), G.711, AAC — 특히 Opus의 동작 원리와 파라미터 |
| **에코 제거 (AEC)** | Acoustic Echo Cancellation 원리 |
| **노이즈 억제 (NS)** | RNNoise 등 ML 기반 접근법 |
| **자동 이득 제어 (AGC)** | 볼륨 자동 조절 |
| **Jitter Buffer** | 네트워크 지터 보상, adaptive jitter buffer |

### 3-2. 비디오

| 주제 | 내용 |
|------|------|
| **코덱** | VP8, VP9, H.264, AV1 — 인코딩/디코딩 파이프라인 |
| **Simulcast / SVC** | 다양한 수신자 대역폭 대응 전략 |
| **대역폭 추정 (BWE)** | GCC (Google Congestion Control), REMB, Transport-CC |
| **FEC / NACK** | 패킷 손실 복구 전략 |

---

## Phase 4: 미디어 서버 개발 역량

### 4-1. SFU 서버 직접 구현해보기

단계적으로 구현:

1. **DTLS 핸드셰이크** 처리 (OpenSSL / BoringSSL)
2. **SRTP 패킷 복호화/암호화**
3. **RTP 라우팅** — 1:N 포워딩 로직
4. **ICE candidate** 처리
5. **Simulcast** 수신 후 선택적 포워딩
6. **RTCP** 피드백 처리 (NACK, PLI, REMB)

### 4-2. 대용량 서버 시스템 설계

- **이벤트 루프 기반 설계**: epoll / io_uring (Linux), libuv
- **멀티스레딩**: 미디어 파이프라인의 lock-free 설계
- **수평 확장**: 서버 간 미디어 릴레이, Cascaded SFU
- **모니터링**: Prometheus + Grafana로 서버 메트릭 수집

### 4-3. Node.js 연동

- Node.js 기초 + Express / Fastify
- WebSocket 기반 시그널링 서버 구현
- C++ ↔ Node.js 연동 (N-API / node-addon-api)
- mediasoup 아키텍처 참고: Node.js가 시그널링, C++이 미디어 처리

---

## Phase 5: 통화 품질 지표 & 운영

### 핵심 품질 지표 (QoS / QoE)

| 지표 | 의미 | 기준값 |
|------|------|--------|
| **RTT** | 왕복 지연 | < 300ms |
| **Jitter** | 패킷 도착 시간 변동 | < 30ms |
| **Packet Loss Rate** | 패킷 손실률 | < 1% |
| **MOS** | 주관적 통화 품질 (Mean Opinion Score) | > 3.5 / 5.0 |
| **비트레이트** | 인코딩/전송 비트레이트 변화 추이 | 적응적 |
| **프레임레이트** | 영상 통화 FPS | > 24fps |

### 품질 분석 도구

- **WebRTC internals** (`chrome://webrtc-internals`)
- `getStats()` API를 통한 실시간 지표 수집
- SRTP 패킷 캡처 후 분석 (Wireshark, tcpdump)
- 자체 Admin / 모니터링 대시보드 설계

---

## Phase 6: 실전 프로젝트 (지속)

### 추천 포트폴리오 프로젝트

1. **미니 SFU 서버** — C++로 2~4인 영상통화 SFU 구현
   - DTLS + SRTP + RTP 포워딩 + RTCP 피드백
2. **시그널링 서버** — Node.js + WebSocket
3. **품질 대시보드** — getStats() 데이터 수집 → 시각화
4. **부하 테스트** — 동시 N개 세션 시뮬레이션

