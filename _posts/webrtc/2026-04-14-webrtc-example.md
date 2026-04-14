---
title: webrtc sample 분석 (conductor)
date: 2026-04-14 00:00:00 +0900
categories: [Blog, WebRTC]
tags: [Tech, WebRTC, sample, conductor]
pin: true
---

## conductor의 상속

```cpp
class Conductor : public webrtc::PeerConnectionObserver,
                  public webrtc::CreateSessionDescriptionObserver,
                  public PeerConnectionClientObserver,
                  public MainWndCallback {
```

### 1. `webrtc::PeerConnectionObserver`

- **정의:** `src/api/peer_connection_interface.h`
- **역할:** WebRTC PeerConnection의 상태 변화를 감지하는 옵저버

| 메서드 | 역할 |
|--------|------|
| `OnSignalingChange()` | 시그널링 상태(stable, have-offer 등) 변경 시 호출 |
| `OnAddStream()` | 원격 피어의 미디어 스트림 추가 |
| `OnRemoveStream()` | 원격 피어의 미디어 스트림 제거 |
| `OnDataChannel()` | 원격 피어가 데이터 채널을 열었을 때 |
| `OnRenegotiationNeeded()` | 재협상이 필요할 때 (ICE restart 등) |
| `OnNegotiationNeededEvent()` | spec-compliant 재협상 이벤트 (Operations Chain이 비어있을 때만) |
| `OnIceConnectionChange()` | ICE 연결 상태 변경 (legacy) |
| `OnStandardizedIceConnectionChange()` | ICE 연결 상태 변경 (표준) |
| `OnConnectionChange()` | PeerConnection 전체 상태 변경 |
| `OnIceGatheringChange()` | ICE candidate 수집 상태 변경 |
| `OnIceCandidate()` | 새 ICE candidate가 수집되었을 때 |
| `OnIceCandidateError()` | ICE candidate 수집 실패 |
| `OnIceCandidateRemoved()` | ICE candidate 제거 |
| `OnIceConnectionReceivingChange()` | ICE 연결 수신 상태 변경 |
| `OnIceSelectedCandidatePairChanged()` | 선택된 ICE candidate 쌍 변경 |
| `OnAddTrack()` | 리시버와 트랙 생성 시 (하위 호환) |
| `OnTrack()` | 원격 트랜시버로부터 미디어 수신 시작 (Unified Plan) |
| `OnRemoveTrack()` | 트랙 제거 시 |
| `OnInterestingUsage()` | 흥미로운 사용 패턴 감지 시 |

---

### 2. `webrtc::CreateSessionDescriptionObserver`

- **정의:** `src/api/jsep.h`
- **역할:** SDP(Session Description Protocol) Offer/Answer 생성 결과를 비동기로 수신
- **참고:** `RefCountInterface`를 상속하여 참조 카운팅됨

| 메서드 | 역할 |
|--------|------|
| `OnSuccess(SessionDescriptionInterface* desc)` | SDP 생성 성공, 생성된 SDP 전달 |
| `OnFailure(RTCError error)` | SDP 생성 실패, 에러 정보 전달 |

`CreateOffer()` / `CreateAnswer()` 호출 시 이 옵저버를 전달하면 비동기로 결과를 받습니다.

---

### 3. `PeerConnectionClientObserver`

- **정의:** `src/examples/peerconnection/client/peer_connection_client.h`
- **역할:** 시그널링 서버와의 통신 이벤트를 수신 (예제 앱 전용)

| 메서드 | 역할 |
|--------|------|
| `OnSignedIn()` | 시그널링 서버에 로그인 성공 |
| `OnDisconnected()` | 서버 연결 해제 |
| `OnPeerConnected(id, name)` | 새 피어가 접속 |
| `OnPeerDisconnected(peer_id)` | 피어가 연결 종료 |
| `OnMessageFromPeer(peer_id, message)` | 피어로부터 메시지(SDP/ICE candidate) 수신 |
| `OnMessageSent(err)` | 메시지 전송 완료/실패 |
| `OnServerConnectionFailure()` | 서버 연결 실패 |

---

### 4. `MainWndCallback`

- **정의:** `src/examples/peerconnection/client/main_wnd.h`
- **역할:** UI 윈도우에서 발생하는 사용자 액션을 처리 (예제 앱 전용)

| 메서드 | 역할 |
|--------|------|
| `StartLogin(server, port)` | 사용자가 로그인 버튼을 눌렀을 때 |
| `DisconnectFromServer()` | 서버 연결 해제 요청 |
| `ConnectToPeer(peer_id)` | 특정 피어와 연결 시도 |
| `DisconnectFromCurrentPeer()` | 현재 피어와 연결 해제 |
| `UIThreadCallback(msg_id, data)` | UI 스레드에서 비동기 콜백 실행 |
| `Close()` | 앱 종료 |

---

### 아키텍처 다이어그램

```
┌─────────────┐     PeerConnectionClientObserver     ┌──────────────────┐
│  Signaling  │ ──────────────────────────────────── │                  │
│   Server    │  OnSignedIn, OnMessageFromPeer, ...  │                  │
└─────────────┘                                      │                  │
                                                     │    Conductor     │
┌─────────────┐     PeerConnectionObserver           │   (Controller)   │
│   WebRTC    │ ──────────────────────────────────── │                  │
│   Engine    │  OnIceCandidate, OnTrack, ...        │                  │
│             │                                      │                  │
│  (Create)   │  CreateSessionDescriptionObserver    │                  │
│  Offer/Ans  │ ──────────────────────────────────── │                  │
└─────────────┘  OnSuccess, OnFailure                │                  │
                                                     │                  │
┌─────────────┐     MainWndCallback                  │                  │
│     UI      │ ──────────────────────────────────── │                  │
│   Window    │  StartLogin, ConnectToPeer, ...      │                  │
└─────────────┘                                      └──────────────────┘
```

---

## conductor의 역할

`Conductor`는 세 계층 사이의 **중재자(Mediator)** 역할을 합니다.

| 계층 | 대표 객체 | 하는 일 |
|------|-----------|---------|
| UI | `MainWindow` | 화면 표시, 사용자 입력 |
| 시그널링 | `PeerConnectionClient` | 서버와 HTTP 통신 |
| 미디어 | `PeerConnection` | WebRTC 연결, 음성/영상 처리 |

이 세 가지는 서로를 직접 알지 못합니다. Conductor가 중간에서 연결합니다.

---

## 스레드 구조

| 스레드 | 이름 | 누가 만드는가 | 어디서 |
|--------|------|---------------|--------|
| 시그널링 스레드 | (이름 없음) | **Conductor가 직접** | `conductor.cc:177` |
| 네트워크 스레드 | `pc_network_thread` | **WebRTC가 자동** | `connection_context.cc:43-48` |
| 워커 스레드 | `pc_worker_thread` | **WebRTC가 자동** | `connection_context.cc:108-110` |

Conductor는 시그널링 스레드만 만들어서 `PeerConnectionFactoryDependencies`에 넘기고,
나머지 2개는 WebRTC 내부(`ConnectionContext`)에서 `nullptr`일 경우 자동 생성합니다.

---

## 콜백 수신 메커니즘

WebRTC로부터의 응답은 **콜백 패턴**으로 처리됩니다.

1. `CreatePeerConnection()` 시 `PeerConnectionDependencies(this)`로 옵저버 등록
2. WebRTC 내부 스레드(네트워크)에서 이벤트 발생
3. `PostTask()`로 시그널링 스레드 메시지 큐에 작업 전달
4. 시그널링 스레드가 `RunWithObserver()`를 통해 Conductor의 콜백 호출

```
[네트워크 스레드]                    [시그널링 스레드]                [UI 스레드 (메인)]
     │                                    │                              │
ICE Agent가                               │                              │
candidate 발견                            │                              │
     │                                    │                              │
     ├── PostTask() ──────────────────→   │                              │
     │   (메시지 큐에 넣음)               │                              │
                                          │                              │
                              PeerConnection::                           │
                              OnIceCandidate()                           │
                                          │                              │
                              RunWithObserver()                          │
                                          │                              │
                              Conductor::                                │
                              OnIceCandidate()                           │
                                          │                              │
                              SendMessage()                              │
                                          │                              │
                                          ├── QueueUIThreadCallback() ─→ │
                                                                         │
                                                          UIThreadCallback()
                                                          실제 메시지 전송
```

---

## 시나리오: A가 B에게 영상통화를 건다

### Step 1: UI → Conductor (사용자가 B를 클릭)

```
[MainWindow]  "B 클릭됨!"
     │
     └──→  Conductor::ConnectToPeer(peer_id=3)
```

Conductor 판단: "통화를 걸고 싶구나 → PeerConnection 만들고 → Offer를 생성하자"

```cpp
void Conductor::ConnectToPeer(int peer_id) {
    if (InitializePeerConnection()) {
        peer_id_ = peer_id;
        peer_connection_->CreateOffer(this, ...);
    }
}
```

### Step 2: WebRTC → Conductor (Offer 생성 완료)

```
[WebRTC 엔진]  "Offer 만들었어!"
     │
     └──→  Conductor::OnSuccess(desc)
```

Conductor 판단: "Offer를 받았으니 → Local에 설정하고 → JSON으로 만들어서 → 시그널링 서버로 보내자"

```cpp
void Conductor::OnSuccess(SessionDescriptionInterface* desc) {
    peer_connection_->SetLocalDescription(..., desc);
    // SDP를 JSON으로 직렬화
    SendMessage(Json::writeString(factory, jmessage));
}
```

### Step 3: Conductor → 시그널링 서버

```
[Conductor]  "이 Offer를 B에게 전달해줘"
     │
     └──→  PeerConnectionClient::SendToPeer(peer_id=3, sdp_json)
```

`PeerConnectionClient`가 HTTP로 시그널링 서버에 전송합니다.

### Step 4: B쪽에서 Answer 도착

```
[PeerConnectionClient]  "B에게서 메시지 왔어!"
     │
     └──→  Conductor::OnMessageFromPeer(peer_id=3, answer_json)
```

Conductor 판단: "Answer가 왔구나 → WebRTC에 RemoteDescription으로 설정하자"

```cpp
void Conductor::OnMessageFromPeer(int peer_id, const std::string& message) {
    // JSON 파싱 → SDP 객체 생성
    peer_connection_->SetRemoteDescription(..., session_description.release());
    // type이 offer였다면 CreateAnswer()도 호출
}
```

### Step 5: ICE Candidate 교환 (자동)

**보내기**: WebRTC가 candidate 발견 → Conductor → 시그널링 서버 → B

```
[WebRTC 엔진]  "ICE candidate 찾았어!"
     │
     └──→  Conductor::OnIceCandidate(candidate)
                │
                └──→  SendMessage(json) → PeerConnectionClient → 서버 → B
```

**받기**: B의 candidate 도착 → 시그널링 서버 → Conductor → WebRTC

```
[PeerConnectionClient]  "B의 ICE candidate 왔어!"
     │
     └──→  Conductor::OnMessageFromPeer(...)
                │
                └──→  peer_connection_->AddIceCandidate(...)
```

### Step 6: 연결 완료, 원격 영상 수신

```
[WebRTC 엔진]  "B의 영상 트랙 도착!"
     │
     └──→  Conductor::OnAddTrack(receiver)
                │
                └──→  QueueUIThreadCallback(NEW_TRACK_ADDED, track)
                           │
                           └──→  UIThreadCallback(NEW_TRACK_ADDED)
                                      │
                                      └──→  main_wnd_->StartRemoteRenderer(video_track)
```

Conductor 판단: "영상 트랙이 왔구나 → UI 스레드에서 렌더러를 시작하자"

---

## 전체 흐름 다이어그램

```
  [UI: MainWindow]          [Conductor]           [시그널링: PeerConnectionClient]    [WebRTC 엔진]
       │                        │                            │                            │
  B 클릭 ──────────────→  ConnectToPeer()                    │                            │
       │                        │ ──── InitializePeerConnection() ──────────────→  Factory/PC 생성
       │                        │ ──── CreateOffer(this) ───────────────────────→  Offer 생성 시작
       │                        │                            │                            │
       │                   OnSuccess(desc) ←─────────────────────────────────── Offer 완성!
       │                        │                            │                            │
       │                        │ ── SendMessage(json) ──→ SendToPeer() ──→ 서버 ──→ B    │
       │                        │                            │                            │
       │                   OnMessageFromPeer() ←── B의 Answer 도착                        │
       │                        │ ── SetRemoteDescription() ────────────────────→  적용    │
       │                        │                            │                            │
       │                   OnIceCandidate() ←───────────────────────────────── candidate!
       │                        │ ── SendMessage() ────→ SendToPeer() ──→ 서버 ──→ B      │
       │                        │                            │                            │
       │                   OnAddTrack() ←─────────────────────────────────── B 영상 도착!
  StartRemoteRenderer() ←──────│                            │                            │
       │                        │                            │                            │
    B 영상 표시!                 │                            │                            │
```
