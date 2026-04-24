---
title: webrtc protocol 정리
date: 2026-04-24 00:00:00 +0900
categories: [Blog, WebRTC, protocol]
tags: [Tech, WebRTC, protocol]
pin: true
---

## 1. 프로토콜 개요

### ICE (Interactive Connectivity Establishment)

NAT/방화벽 환경에서 두 피어 간의 최적 경로를 찾기 위한 프레임워크 (RFC 8445).

**세 가지 후보 유형**
- **Host Candidate**: 로컬 네트워크 인터페이스의 주소
- **Server Reflexive Candidate (srflx)**: STUN 서버를 통해 얻은 공인 IP 주소
- **Relayed Candidate (relay)**: TURN 서버를 경유하는 주소

수집된 후보들은 SDP를 통해 교환되며, 각 후보 쌍에 대해 STUN Binding Request로 연결성 검사를 수행합니다. 우선순위는 일반적으로 **Host > srflx > relay** 순.

### RTP (Real-time Transport Protocol)

실시간 오디오/비디오 데이터를 전송하기 위한 프로토콜 (RFC 3550). UDP 위에서 동작하며 신뢰성보다 지연(latency) 최소화를 우선시합니다.

**RTP 헤더 주요 필드**
- Payload Type (코덱 식별)
- Sequence Number (패킷 순서, 손실 감지)
- Timestamp (재생 동기화)
- SSRC (동기화 소스 식별자)

WebRTC에서는 보안을 위해 **SRTP (Secure RTP)** 를 사용.

### RTCP (RTP Control Protocol)

RTP 세션의 품질을 모니터링하고 제어 정보를 교환하는 프로토콜. RTP와 쌍을 이루어 동작.

**주요 리포트 타입**
- Sender Report (SR) — 송신자 통계
- Receiver Report (RR) — 수신자 통계 (손실률, 지터, 지연)
- SDES, BYE, APP
- RTCP Feedback: **NACK** (재전송), **PLI/FIR** (키프레임 요청), **REMB/TMMBR** (대역폭 제어)

WebRTC에서는 `rtcp-mux`로 RTP와 동일 포트 공유, 보안 적용 시 **SRTCP**.

### DTLS (Datagram Transport Layer Security)

UDP 기반 통신에 TLS와 동일한 수준의 보안을 제공 (RFC 6347/9147).

**두 가지 역할**
1. DataChannel의 암호화 전송 계층 (SCTP over DTLS)
2. **DTLS-SRTP** (RFC 5764): SRTP 키 협상

WebRTC의 DTLS는 **자가 서명 인증서**를 사용하며, 인증서의 지문(fingerprint)을 SDP의 `a=fingerprint` 속성으로 교환해 중간자 공격을 방지.

---

## 2. 연결 수립 절차 다이어그램

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│  Peer A     │    │  Signaling   │    │ STUN / TURN │    │  Peer B     │
│  (Caller)   │    │   Server     │    │   Server    │    │  (Callee)   │
└──────┬──────┘    └──────┬───────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ①    │   Signaling : SDP Offer / Answer 교환                    │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │ createOffer()    │                   │                   │
       │─────┐            │                   │                   │
       │<────┘            │                   │                   │
       │ SDP Offer        │                   │                   │
       │─────────────────>│                   │                   │
       │                  │  Forward Offer    │                   │
       │                  │──────────────────────────────────────>│
       │                  │   SDP Answer      │                   │
       │                  │<──────────────────────────────────────│
       │ Forward Answer   │                   │                   │
       │<─────────────────│                   │                   │
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ②    │   ICE : 후보 수집 & 연결성 검사                          │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │ [host 후보 수집] │                   │  [host 후보 수집] │
       │ STUN Binding Request                 │                   │
       │─────────────────────────────────────>│                   │
       │        Binding Response (srflx 주소) │                   │
       │<─────────────────────────────────────│                   │
       │                  │                   │  STUN Binding Req │
       │                  │                   │<──────────────────│
       │                  │                   │  Response (srflx) │
       │                  │                   │──────────────────>│
       │ ICE candidates (trickle, via signaling)                  │
       │─────────────────>│──────────────────────────────────────>│
       │ STUN Binding Request (연결성 검사, USE-CANDIDATE)        │
       │─────────────────────────────────────────────────────────>│
       │      Binding Response → 후보 쌍 nominated                │
       │<─────────────────────────────────────────────────────────│
       │    ✓ ICE connected : 경로 확정 (host / srflx / relay)    │
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ③    │   DTLS Handshake : 암호화 세션 수립 (UDP 위)             │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │ ClientHello                                              │
       │─────────────────────────────────────────────────────────>│
       │                   HelloVerifyRequest (cookie)            │
       │<─────────────────────────────────────────────────────────│
       │ ClientHello + cookie                                     │
       │─────────────────────────────────────────────────────────>│
       │      ServerHello, Certificate, ServerHelloDone           │
       │<─────────────────────────────────────────────────────────│
       │ ClientKeyExchange, ChangeCipherSpec, Finished            │
       │─────────────────────────────────────────────────────────>│
       │              ChangeCipherSpec, Finished                  │
       │<─────────────────────────────────────────────────────────│
       │  ✓ fingerprint 검증 + SRTP keying material 추출          │
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ④    │   Media Flow : SRTP / SRTCP (rtcp-mux, 단일 포트)        │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │ SRTP : 암호화된 오디오/비디오 (seq, ts, SSRC)            │
       │═════════════════════════════════════════════════════════>│
       │            SRTP : 반대 방향 미디어 스트림                │
       │<═════════════════════════════════════════════════════════│
       │       SRTCP : Receiver Report (loss, jitter, RTT)        │
       │<─────────────────────────────────────────────────────────│
       │ SRTCP : Sender Report + SDES                             │
       │─────────────────────────────────────────────────────────>│
       │   RTCP FB : NACK / PLI / REMB (재전송·키프레임·대역폭)   │
       │<─────────────────────────────────────────────────────────│
       │ SCTP over DTLS : DataChannel 메시지 (선택)               │
       │<────────────────────────────────────────────────────────>│
       │  • ICE consent freshness : 30초마다 STUN Binding 재확인  │
       │  • ICE restart : 네트워크 변경 시 ufrag/pwd 재협상       │
       ▼                  ▼                   ▼                   ▼

범례 :
  ─────>  일반 메시지(요청/응답)
  ═════>  암호화된 미디어 스트림 (SRTP)
  ①②③④  연결 수립 단계
```

---

## 3. ① Signaling — SDP Offer / Answer

### 단계 1-1. `createOffer()` — 로컬에서 SDP 생성

{% raw %}
```cpp
#include "api/peer_connection_interface.h"
#include "api/create_peerconnection_factory.h"

// PeerConnectionFactory 생성 (프로세스당 1개)
auto pcf = webrtc::CreatePeerConnectionFactory(
    /*network_thread=*/network_thread_.get(),
    /*worker_thread=*/worker_thread_.get(),
    /*signaling_thread=*/signaling_thread_.get(),
    /*default_adm=*/nullptr,
    webrtc::CreateBuiltinAudioEncoderFactory(),
    webrtc::CreateBuiltinAudioDecoderFactory(),
    webrtc::CreateBuiltinVideoEncoderFactory(),
    webrtc::CreateBuiltinVideoDecoderFactory(),
    /*audio_mixer=*/nullptr,
    /*audio_processing=*/nullptr);

// RTCConfiguration: STUN/TURN 서버, ICE 정책
webrtc::PeerConnectionInterface::RTCConfiguration cfg;
cfg.servers.push_back({.urls = {"stun:stun.l.google.com:19302"}});
cfg.servers.push_back({
    .urls = {"turn:turn.example.com:3478?transport=udp"},
    .username = "alice", .password = "s3cret"});
cfg.sdp_semantics = webrtc::SdpSemantics::kUnifiedPlan;
cfg.bundle_policy  = webrtc::PeerConnectionInterface::kBundlePolicyMaxBundle;
cfg.rtcp_mux_policy = webrtc::PeerConnectionInterface::kRtcpMuxPolicyRequire;

auto pc = pcf->CreatePeerConnectionOrError(
    cfg, webrtc::PeerConnectionDependencies(&observer)).MoveValue();

// 미디어 트랙 추가
pc->AddTransceiver(cricket::MEDIA_TYPE_AUDIO);
pc->AddTransceiver(cricket::MEDIA_TYPE_VIDEO);

// Offer 생성 시작
class OfferCB : public webrtc::CreateSessionDescriptionObserver {
  void OnSuccess(webrtc::SessionDescriptionInterface* desc) override {
    std::string sdp; desc->ToString(&sdp);
    pc_->SetLocalDescription(new rtc::RefCountedObject<SetLocalCB>(), desc);
    signaling_->Send({{"type","offer"},{"sdp",sdp}});
  }
};
pc->CreateOffer(new rtc::RefCountedObject<OfferCB>(), {});
```
{% endraw %}

### 단계 1-2. SDP Offer (A → Signaling Server)

WebSocket 위에 JSON으로 실려나가는 실제 페이로드:

```json
{
  "type": "offer",
  "sdp": "v=0\r\no=- 4611731400430051336 2 IN IP4 127.0.0.1\r\ns=-\r\nt=0 0\r\na=group:BUNDLE 0 1\r\na=extmap-allow-mixed\r\na=msid-semantic: WMS mstream0\r\nm=audio 9 UDP/TLS/RTP/SAVPF 111 103 104\r\nc=IN IP4 0.0.0.0\r\na=rtcp:9 IN IP4 0.0.0.0\r\na=ice-ufrag:F7gI\r\na=ice-pwd:x9cml/YzichV2+XlhiMu8g\r\na=ice-options:trickle\r\na=fingerprint:sha-256 AA:BB:CC:DD:EE:FF:...\r\na=setup:actpass\r\na=mid:0\r\na=sendrecv\r\na=rtcp-mux\r\na=rtpmap:111 opus/48000/2\r\na=fmtp:111 minptime=10;useinbandfec=1\r\na=rtcp-fb:111 transport-cc\r\nm=video 9 UDP/TLS/RTP/SAVPF 96 97\r\nc=IN IP4 0.0.0.0\r\na=ice-ufrag:F7gI\r\na=ice-pwd:x9cml/YzichV2+XlhiMu8g\r\na=fingerprint:sha-256 AA:BB:CC:DD:EE:FF:...\r\na=setup:actpass\r\na=mid:1\r\na=sendrecv\r\na=rtcp-mux\r\na=rtpmap:96 VP8/90000\r\na=rtcp-fb:96 nack\r\na=rtcp-fb:96 nack pli\r\na=rtcp-fb:96 goog-remb\r\na=rtcp-fb:96 transport-cc\r\n"
}
```

**주요 SDP 속성**

| 속성 | 의미 |
|------|------|
| `a=group:BUNDLE 0 1` | 오디오(mid=0), 비디오(mid=1)를 단일 전송 채널로 묶음 |
| `a=ice-ufrag` / `a=ice-pwd` | ICE 자격증명 (STUN MESSAGE-INTEGRITY 계산) |
| `a=fingerprint:sha-256 ...` | DTLS 인증서 지문 (MITM 방지) |
| `a=setup:actpass` | DTLS 역할 협상 (active/passive/actpass) |
| `a=rtcp-mux` | RTP/RTCP 포트 공유 |
| `a=rtcp-fb` | 지원하는 RTCP 피드백 유형 |
| `UDP/TLS/RTP/SAVPF` | SRTP + Secure RTCP with Feedback |

### 단계 1-3. Forward Offer (Signaling Server → B)

시그널링 서버는 라우팅만 담당:

```cpp
// 서버 측 (C++ websocket)
server_.OnMessage([&](Connection* sender, const json& msg) {
  if (msg["type"] == "offer") {
    auto peer = room_.Other(sender);
    peer->Send(msg);  // 그대로 전달
  }
});
```

### 단계 1-4. Peer B가 Offer 처리 → `CreateAnswer`

```cpp
signaling_->OnMessage = [&](const json& msg) {
  if (msg["type"] == "offer") {
    webrtc::SdpParseError err;
    auto remote = webrtc::CreateSessionDescription(
        webrtc::SdpType::kOffer, msg["sdp"], &err);
    pc_->SetRemoteDescription(std::move(remote),
        rtc::make_ref_counted<SetRemoteCB>());
    pc_->CreateAnswer(new rtc::RefCountedObject<AnswerCB>(), {});
  }
};
```

`SetRemoteDescription` 내부 처리:
- Offer의 `a=setup:actpass`를 보고 B는 `active` (DTLS client) 역할 결정
- Offer의 `fingerprint`를 저장 (DTLS 인증서 검증에 사용)
- `ice-ufrag` / `ice-pwd`를 저장 (STUN MESSAGE-INTEGRITY 계산/검증)
- ICE agent를 `controlled` 역할로 설정

### 단계 1-5. SDP Answer (B → A)

```
a=setup:active        ← Offer의 actpass에 대한 응답으로 active 선택
a=ice-ufrag:k3Pq      ← B의 ICE 자격증명 (A와 다름)
a=ice-pwd:...
a=fingerprint:sha-256 11:22:33:44:...   ← B의 인증서 지문
```

---

## 4. ② ICE — 후보 수집 및 연결성 검사

### 단계 2-1. Host candidate 수집

`SetLocalDescription` 성공 시 ICE agent 자동 시작:

```cpp
// cricket::BasicPortAllocator::GetNetworks()
// 로컬 NIC 목록 열거 (Linux: getifaddrs, Windows: GetAdaptersAddresses)
std::vector<rtc::Network*> networks;
network_manager_->GetNetworks(&networks);

for (auto* net : networks) {
  // 각 인터페이스에 UDP 소켓 바인드 → Host candidate 생성
  auto port = cricket::UDPPort::Create(
      network_thread_, socket_factory_, net,
      /*min_port=*/0, /*max_port=*/0,
      ice_ufrag_, ice_pwd_);

  port->SignalCandidateReady.connect(this, &Allocator::OnCandidate);
  port->PrepareAddress();  // getsockname → candidate 생성
}
```

생성된 Host candidate 예:

```
candidate:1 1 udp 2113667327 192.168.1.10 54321 typ host generation 0
           ▲ ▲   ▲                                 ▲
      foundation│ priority                     type=host
            component(1=RTP)
```

### 단계 2-2. STUN Binding Request (A → STUN server)

```cpp
cricket::StunRequest* req = new cricket::StunBindingRequest(this);
req->SetType(STUN_BINDING_REQUEST);
req->SendMessage();
```

와이어 위 실제 UDP 페이로드 (20바이트 헤더만):

```
Offset  Bytes           Meaning
------  --------------  ----------------------------------------
0-1     00 01           Type: Binding Request
2-3     00 00           Length: 0 (attributes 없음)
4-7     21 12 A4 42     Magic Cookie (고정값)
8-19    B7 E7 A7 01     Transaction ID (96 bits, 랜덤)
        BC 34 D6 86
        FA 87 DF AE
```

### 단계 2-3. STUN Binding Response (STUN server → A)

공인 주소를 XOR-MAPPED-ADDRESS로 응답:

```
00 01 01 | Type: Binding Success Response (0x0101)
00 0C    | Length: 12 bytes
21 12 A4 42 B7 E7 A7 01 BC 34 D6 86 FA 87 DF AE   | Cookie + TxID

Attribute: XOR-MAPPED-ADDRESS (0x0020)
00 20    | Type
00 08    | Length
00 01    | Reserved + Family (IPv4)
A1 47    | XOR'd port (= port ^ 0x2112 = 54321)
DF D7 6D B7  | XOR'd IPv4 (= 203.0.113.5 ^ 0x2112A442)
```

libwebrtc 처리:

```cpp
void StunPort::OnStunBindingRequestSucceeded(
    int rtt_ms, const rtc::SocketAddress& stun_server,
    const rtc::SocketAddress& stun_reflected_addr) {
  // srflx 후보 생성
  AddAddress(stun_reflected_addr, socket_->GetLocalAddress(),
             stun_server, UDP_PROTOCOL_NAME, "", "",
             STUN_PORT_TYPE, ICE_TYPE_PREFERENCE_SRFLX,
             /*generation=*/0, /*url=*/"stun:stun.l.google.com:19302",
             /*is_final=*/true);
}
```

결과 srflx candidate:

```
candidate:2 1 udp 1677729535 203.0.113.5 54321
          typ srflx raddr 192.168.1.10 rport 54321
```

### 단계 2-6. ICE candidates trickle 전송

수집된 후보는 즉시 `OnIceCandidate`로 올라오고 시그널링으로 전달:

```cpp
void PCObserver::OnIceCandidate(
    const webrtc::IceCandidateInterface* c) override {
  std::string s; c->ToString(&s);
  signaling_->Send({
      {"type", "candidate"},
      {"candidate", {
          {"candidate", "candidate:" + s},
          {"sdpMid", c->sdp_mid()},
          {"sdpMLineIndex", c->sdp_mline_index()}
      }}
  });
}
```

실제 전송되는 JSON:

```json
{
  "type": "candidate",
  "candidate": {
    "candidate": "candidate:2 1 udp 1677729535 203.0.113.5 54321 typ srflx raddr 192.168.1.10 rport 54321 generation 0 ufrag F7gI",
    "sdpMid": "0",
    "sdpMLineIndex": 0
  }
}
```

수신 측 처리:

```cpp
auto cand = webrtc::CreateIceCandidate(
    msg["sdpMid"], msg["sdpMLineIndex"],
    msg["candidate"]["candidate"], &err);
pc->AddIceCandidate(std::move(cand),
    [](webrtc::RTCError e) { /* ... */ });
```

### 단계 2-7. P2P STUN Binding (연결성 검사)

이제 **피어 간 직접 STUN**을 보냄. Host가 STUN 서버로 보낸 것과 달리 `USERNAME`과 `MESSAGE-INTEGRITY`가 포함됨:

```cpp
// cricket::Connection::Ping() 내부
auto req = std::make_unique<ConnectionRequest>(this);
auto* msg = req->mutable_msg();
msg->SetType(STUN_BINDING_REQUEST);

// ICE-CONTROLLING / ICE-CONTROLLED
if (port_->GetIceRole() == ICEROLE_CONTROLLING) {
  msg->AddAttribute(std::make_unique<StunUInt64Attribute>(
      STUN_ATTR_ICE_CONTROLLING, tiebreaker_));
  // nominated 쌍이면
  msg->AddAttribute(std::make_unique<StunByteStringAttribute>(
      STUN_ATTR_USE_CANDIDATE));
}

// PRIORITY
msg->AddAttribute(std::make_unique<StunUInt32Attribute>(
    STUN_ATTR_PRIORITY, prflx_priority));

// USERNAME = "remote_ufrag:local_ufrag"
msg->AddAttribute(std::make_unique<StunByteStringAttribute>(
    STUN_ATTR_USERNAME, remote_ufrag_ + ":" + local_ufrag_));

// MESSAGE-INTEGRITY (HMAC-SHA1 with remote_pwd)
msg->AddMessageIntegrity(remote_pwd_);
msg->AddFingerprint();
```

와이어 페이로드:

```
00 01 00 58           Binding Request, len=88
21 12 A4 42 [TxID]    Cookie + Transaction ID

0024 0004 6EFFFFFE    PRIORITY = 1862270462
0025 0000             USE-CANDIDATE (nomination 시)
802A 0008 [8bytes]    ICE-CONTROLLING tiebreaker
0006 0009             USERNAME len=9
"k3Pq:F7gI" 00 00 00  "remote_ufrag:local_ufrag" + padding
0008 0014 [20 bytes]  MESSAGE-INTEGRITY (HMAC-SHA1)
8028 0004 [CRC32]     FINGERPRINT
```

### 단계 2-8. Binding Response → 후보 쌍 nominated

수신자는 `MESSAGE-INTEGRITY`를 `local_pwd`로 검증 후 응답:

```cpp
void StunServerSocket::OnBindingRequest(StunMessage* msg, ...) {
  if (!msg->ValidateMessageIntegrity(ice_pwd_)) {
    SendErrorResponse(msg, STUN_ERROR_UNAUTHORIZED);
    return;
  }
  StunMessage response(STUN_BINDING_RESPONSE, msg->transaction_id());
  response.AddAttribute(MakeXorMappedAddr(remote_addr));
  response.AddMessageIntegrity(ice_pwd_);
  response.AddFingerprint();
  SendStun(response);
}
```

양방향 성공 시:

```cpp
void P2PTransportChannel::SwitchSelectedConnection(
    Connection* conn, IceSwitchReason reason) {
  selected_connection_ = conn;
  SignalReadyToSend(this);
  SetReceiving(true);
  SetWritable(true);
  // IceConnectionState → kIceConnectionConnected
}
```

---

## 5. ③ DTLS Handshake

ICE가 writable이 되면 DTLS가 같은 소켓에서 시작:

```cpp
// cricket::DtlsTransport::OnWritableState()
bool DtlsTransport::MaybeStartDtls() {
  if (!ice_->writable()) return false;
  if (dtls_state_ == DTLS_TRANSPORT_NEW) {
    dtls_->StartSSL();       // ClientHello 트리거 (active 측만)
    set_dtls_state(DTLS_TRANSPORT_CONNECTING);
  }
  return true;
}
```

### 단계 3-1. ClientHello (A → B)

DTLS 레코드 + 핸드셰이크 페이로드:

```
== DTLS Record ==
16              Content Type: Handshake
FE FD           Version: DTLS 1.2
00 00           Epoch: 0
00 00 00 00 00 00   Sequence Number: 0
00 C4           Length: 196

== Handshake ==
01              Type: ClientHello (1)
00 00 B8        Length: 184
00 00           Message Sequence: 0
00 00 00        Fragment Offset: 0
00 00 B8        Fragment Length: 184

FE FD           client_version: DTLS 1.2
[32 bytes]      Random (gmt_unix_time + 28 random)
00              SessionID length: 0
00              Cookie length: 0 (첫 번째 HELLO)

00 08           CipherSuites length
  13 02         TLS_AES_256_GCM_SHA384
  13 03         TLS_CHACHA20_POLY1305_SHA256
  C0 2B         TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  C0 2C         TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384

01 00           Compression: null

Extensions:
  000A supported_groups: x25519, secp256r1
  000B ec_point_formats
  000D signature_algorithms: ecdsa_secp256r1_sha256
  000E use_srtp:                         ← DTLS-SRTP 프로파일 협상
    0004 0001 0002   profiles: AEAD_AES_128_GCM, AES128_CM_SHA1_80
    00              mki length
  0017 extended_master_secret
```

### 단계 3-2. HelloVerifyRequest (B → A)

DoS 방지를 위해 cookie를 돌려줌:

```
16 FE FD 00 00 00 00 00 00 00 00  Record (Handshake, epoch=0, seq=0)
00 23 Length=35

03                   Type: HelloVerifyRequest (3)
00 00 17             Length: 23
00 00                Message Seq: 0
00 00 00             Fragment offset: 0
00 00 17             Fragment len: 23

FE FD                server_version
14                   cookie length: 20
[20 bytes cookie]    (서버가 HMAC으로 생성)
```

### 단계 3-3. ClientHello + cookie (A → B)

동일 ClientHello에 cookie 추가:

```
01 00 00 CC  ... (ClientHello with cookie 필드 채움)
14 [20 bytes cookie]
```

### 단계 3-4. ServerHello / Certificate / ServerHelloDone

한 UDP 패킷에 여러 핸드셰이크 메시지가 fragment되어 들어감:

```
== ServerHello ==
02 00 00 46   Type=2, len=70
FE FD [Random 32B] [SessionID]
C0 2B         선택된 CipherSuite
00            Compression null
Extensions: use_srtp (선택된 프로파일 0x0001), ems, ...

== Certificate ==
0B 00 04 12 ... (X.509 DER 인코딩, 자가 서명)
  Issuer: CN=WebRTC
  Subject: CN=WebRTC
  Public Key: EC secp256r1
  Signature: ECDSA-SHA256

== ServerKeyExchange (ECDHE) ==
0C ... curve=x25519, pubkey, signature

== CertificateRequest ==       ← 상호 인증 (WebRTC는 항상 요구)
0D ...

== ServerHelloDone ==
0E 00 00 00
```

libwebrtc는 BoringSSL 기반이라 직접 코드는 없지만 설정은:

```cpp
// rtc::OpenSSLStreamAdapter
SSL_CTX* ctx = SSL_CTX_new(DTLS_method());
SSL_CTX_set_min_proto_version(ctx, DTLS1_2_VERSION);
SSL_CTX_set_max_proto_version(ctx, DTLS1_2_VERSION);

// 자가 서명 인증서 등록
SSL_CTX_use_certificate(ctx, cert_->x509());
SSL_CTX_use_PrivateKey(ctx, cert_->evp_pkey());

// DTLS-SRTP 프로파일
SSL_CTX_set_tlsext_use_srtp(ctx,
    "SRTP_AEAD_AES_128_GCM:SRTP_AES128_CM_SHA1_80");

// 상호 인증 요구, 검증은 콜백으로 (fingerprint 비교)
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT,
                   &SSLVerifyCallback);
```

### 단계 3-5. ClientKeyExchange + CCS + Finished (A → B)

```
== Certificate ==            ← A도 자신의 인증서 전송
0B 00 ...

== ClientKeyExchange ==
10 00 00 24 [ECDHE pubkey 33B]

== CertificateVerify ==
0F ... (handshake hash에 대한 서명)

== ChangeCipherSpec (별도 record) ==
Record: Type=20 (ChangeCipherSpec), epoch=0
14 FE FD 00 00 ... 00 01 01

== Finished (encrypted, epoch=1) ==
Record: Type=22, epoch=1, seq=0
16 FE FD 00 01 ... [AEAD ciphertext]
Plaintext 후: 14 00 00 0C [verify_data 12B]
```

### 단계 3-6. ChangeCipherSpec + Finished (B → A)

동일한 구조로 B도 응답. 이 시점에 양방향 암호화 세션 수립.

### 단계 3-7. Fingerprint 검증 + SRTP 키 추출

```cpp
// 1) Fingerprint 검증 (SDP에 있던 지문과 실제 인증서 비교)
bool OpenSSLStreamAdapter::SSLVerifyCallback(SSL* ssl, uint8_t* out_alert) {
  X509* cert = SSL_get_peer_certificate(ssl);
  uint8_t digest[EVP_MAX_MD_SIZE];
  unsigned digest_len;
  X509_digest(cert, EVP_sha256(), digest, &digest_len);

  if (memcmp(digest, expected_fingerprint_, digest_len) != 0) {
    RTC_LOG(LS_ERROR) << "DTLS fingerprint mismatch!";
    return false;   // MITM → 핸드셰이크 실패
  }
  return true;
}

// 2) SRTP keying material 추출 (RFC 5705)
uint8_t material[60];   // 2*(key=16 + salt=14) for AES128_CM_SHA1_80
SSL_export_keying_material(
    ssl_, material, 60,
    "EXTRACTOR-dtls_srtp", 19,
    nullptr, 0, 0);

// 분할
const uint8_t* client_key  = material + 0;
const uint8_t* server_key  = material + 16;
const uint8_t* client_salt = material + 32;
const uint8_t* server_salt = material + 46;

// SRTP 세션 초기화
SrtpSession send, recv;
send.SetSend(SRTP_AES128_CM_SHA1_80, client_key, 16 + 14, {});
recv.SetRecv(SRTP_AES128_CM_SHA1_80, server_key, 16 + 14, {});
```

---

## 6. ④ Media Flow — SRTP / SRTCP / DataChannel

### 단계 4-1. SRTP 송신 (A → B)

Opus 인코더 출력 → RTP 헤더 + SRTP 암호화:

```cpp
// webrtc::RTPSenderAudio::SendAudio
void RTPSenderAudio::SendAudio(AudioFrameType type, int8_t pt,
                               uint32_t rtp_ts, const uint8_t* data,
                               size_t size) {
  auto packet = rtp_sender_->AllocatePacket();
  packet->SetPayloadType(pt);          // 111 (Opus)
  packet->SetTimestamp(rtp_ts);
  packet->SetSequenceNumber(seq_++);
  packet->SetSsrc(ssrc_);
  packet->SetMarker(type == kAudioFrameSpeech && last_was_silence_);

  // RFC 8285 확장
  packet->SetExtension<TransportSequenceNumber>(tcc_seq_++);
  packet->SetExtension<AudioLevel>(voice_activity, level_dbov);

  auto* payload = packet->AllocatePayload(size);
  memcpy(payload, data, size);

  rtp_sender_->SendPacket(std::move(packet));
}

// 이후 SrtpTransport::SendRtpPacket
bool SrtpTransport::SendRtpPacket(rtc::CopyOnWriteBuffer* packet, ...) {
  uint8_t* data = packet->MutableData();
  int in_len = packet->size();
  int out_len;
  size_t max_len = packet->capacity();

  if (!send_session_->ProtectRtp(data, in_len, max_len, &out_len)) {
    return false;
  }
  packet->SetSize(out_len);
  return network_->SendPacket(data, out_len, options);
}
```

**RTP 헤더 포맷** (RFC 3550):

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│V=2│P│X│  CC   │M│    PT     │       Sequence Number          │
├───┴─┴─┴───────┴─┴───────────┴────────────────────────────────┤
│                          Timestamp                           │
├──────────────────────────────────────────────────────────────┤
│                Synchronization Source (SSRC)                 │
└──────────────────────────────────────────────────────────────┘
```

실제 패킷 (Opus 오디오, SRTP 암호화 전):

```
Byte 0   1    2  3   4 5 6 7     8 9 A B    C D E F    10~
┌────┬────┬─────┬─────────┬──────────┬─────────┬──────────────┐
│ 80 │ EF │ 04 D2│ 00 01 E2 40│ 12 34 56 78│ [Ext]│[Opus payload]│
└────┴────┴─────┴─────────┴──────────┴─────────┴──────────────┘
 V=2  M=1  seq=  timestamp  SSRC       ext       encrypted body
 P=0  PT=  1234  =123456    =0x12345678
      111
```

SRTP 암호화 후 (AES-128-CM + HMAC-SHA1 10B):

```
┌────┬────┬─────┬─────────┬──────────┬──────────────┬──────────┐
│ 80 │ EF │ 04 D2│ 00 01 E2 40│ 12 34 56 78│[encrypted payload]│[auth]│
└────┴────┴─────┴─────────┴──────────┴──────────────┴──────────┘
 header는 그대로                         AES-CM으로 암호화      HMAC-
 (인증 범위에만 포함)                                           SHA1 10B
```

### 단계 4-2. SRTP 수신 (B → A)

```cpp
bool SrtpTransport::OnRtpPacketReceived(
    rtc::CopyOnWriteBuffer packet, int64_t packet_time_us) {
  int out_len;
  if (!recv_session_->UnprotectRtp(
          packet.MutableData(), packet.size(), &out_len)) {
    RTC_LOG(LS_WARNING) << "SRTP auth failed";
    return false;
  }
  packet.SetSize(out_len);

  RtpPacketReceived parsed_packet(&extensions_, packet_time_us);
  if (!parsed_packet.Parse(packet)) return false;

  demuxer_.OnRtpPacket(parsed_packet);
  return true;
}
```

### 단계 4-3. SRTCP Receiver Report (B → A)

매 RTCP 간격(기본 ~5초, 대역폭 비례)마다 자동 송신:

```cpp
webrtc::rtcp::ReceiverReport rr;
rr.SetSenderSsrc(local_ssrc_);

webrtc::rtcp::ReportBlock block;
block.SetMediaSsrc(remote_ssrc_);
block.SetFractionLost(fraction_lost);       // 8bit: 손실율 * 256
block.SetCumulativeLost(cumulative_lost);   // 24bit
block.SetExtHighestSeqNum(ext_highest_seq);
block.SetJitter(jitter);                    // RFC 3550 A.8
block.SetLastSr(last_sr_timestamp);         // NTP middle 32bit
block.SetDelayLastSr(delay_since_last_sr);  // 1/65536 초 단위
rr.AddReportBlock(block);

auto buffer = rr.Build();
srtp_send_.ProtectRtcp(buffer.data(), buffer.size(), ...);
transport_->SendRtcp(buffer);
```

와이어 포맷:

```
80 C9 00 07             V=2, P=0, RC=1, PT=201 (RR), len=7
12 34 56 78             Sender SSRC
─────────────────────── Report Block 1 ───────────────────────
AB CD EF 01             SSRC of source (송신자)
05                      Fraction lost: 5/256 ≈ 2%
00 00 23                Cumulative lost: 35
00 00 10 00             Extended highest seq number: 4096
00 00 00 1A             Interarrival jitter: 26
12 34 56 78             Last SR timestamp
00 00 10 00             Delay since last SR
```

### 단계 4-4. SRTCP Sender Report + SDES (A → B)

```
80 C8 00 0C                  SR (PT=200), len=12
12 34 56 78                  Sender SSRC
[8B NTP timestamp]           송신 시각
[4B RTP timestamp]           동일 시점의 RTP ts
[4B packet count]            누적 송신 패킷 수
[4B octet count]             누적 송신 바이트 수

─────────────────────── Compound ───────────────────────
81 CA 00 06                  SDES (PT=202), source count=1
12 34 56 78                  SSRC
01 10                        CNAME (type=1), length=16
"user@host-uuid"             canonical name
00 00                        END + padding
```

### 단계 4-5. RTCP Feedback — NACK / PLI / REMB

**NACK** (패킷 손실 감지 시 즉시):

```cpp
webrtc::rtcp::Nack nack;
nack.SetSenderSsrc(local_ssrc_);
nack.SetMediaSsrc(remote_ssrc_);
nack.SetPacketIds({1234, 1235, 1237, 1242});
```

와이어:

```
81 CD 00 03          V=2, FMT=1 (Generic NACK), PT=205, len=3
AA AA AA AA          Sender SSRC
12 34 56 78          Media SSRC
─── FCI ───
04 D2                PID = 1234
00 05                BLP = 0000 0000 0000 0101
                     → 1234, 1235 (+1), 1237 (+3) 손실
```

**PLI** (키프레임 요청):

```
81 CE 00 02          FMT=1 (PLI), PT=206 (PSFB), len=2
AA AA AA AA          Sender SSRC
12 34 56 78          Media SSRC
                     (FCI 없음)
```

**REMB** (대역폭 추정):

```
8F CE 00 05          FMT=15 (AFB), PT=206
AA AA AA AA          Sender SSRC
00 00 00 00          Media SSRC = 0 (REMB는 0)
52 45 4D 42          "REMB"
01 12 23 45          NumSSRCs=1, BR Exp=4, Mantissa=0x223345
                     → 2.2 Mbps
12 34 56 78          Constraining SSRC
```

### 단계 4-6. SCTP over DTLS — DataChannel

DataChannel은 같은 DTLS 세션 위에 SCTP로 다중화:

```cpp
webrtc::DataChannelInit cfg;
cfg.ordered = true;
cfg.maxRetransmits = 3;
auto channel = pc->CreateDataChannelOrError("chat", &cfg).MoveValue();

channel->RegisterObserver(&obs);
channel->Send(webrtc::DataBuffer("hello"));
```

스택: `애플리케이션 데이터 → SCTP chunk → DTLS Application record → UDP`

SCTP DATA chunk 포맷:

```
== DTLS Application Record ==
17 FE FD 00 01 [seq] [len]
[encrypted: SCTP packet]

== Decrypted SCTP packet ==
Source Port:      5000
Destination Port: 5000
Verification Tag: 0x12345678
Checksum:         CRC32c

=== DATA Chunk ===
Type: 0 (DATA)
Flags: 0x03 (B|E: begin+end)
Length: 21
TSN: 1
Stream Id: 1
Stream Seq: 0
PPID: 51 (WebRTC string)
Data: "hello"
```

### 단계 4-7. ICE consent freshness (30초)

세션 중에도 경로 유효성을 주기 확인 (RFC 7675):

```cpp
void Connection::Ping(int64_t now) {
  if (now - last_ping_sent_ >= STUN_KEEPALIVE_INTERVAL) {   // 30s
    SendStunBindingRequest();
    last_ping_sent_ = now;
  }
  // 15초간 응답 없으면 consent 만료
  if (now - last_ping_response_received_ > CONSENT_TIMEOUT) {
    set_write_state(STATE_WRITE_TIMEOUT);
    SignalStateChange(this);   // → ICE failed
  }
}
```

### 단계 4-8. ICE Restart

네트워크 변경(Wi-Fi ↔ 4G) 시 새 ufrag/pwd로 재협상:

```cpp
// JavaScript-level: pc.createOffer({iceRestart: true})
// C++:
webrtc::PeerConnectionInterface::RTCOfferAnswerOptions opts;
opts.ice_restart = true;
pc->CreateOffer(new rtc::RefCountedObject<OfferCB>(), opts);
```

Restart 후 새 Offer의 SDP 차이:

```
a=ice-ufrag:zX9p      ← 이전 F7gI에서 변경
a=ice-pwd:newpwd...   ← 변경
a=ice-options:trickle
(fingerprint, setup은 동일 — DTLS 재협상 아님)
```

**핵심**: 기존 DTLS/SRTP 세션은 그대로 유지되고 전송 경로만 교체. 미디어 중단이 거의 없는 seamless fallback.

---

## 7. 전체 파이프라인 요약

모든 단계가 하나의 `PeerConnection` 객체 안에 캡슐화되어 실제 애플리케이션 코드는 다음처럼 압축됨:

```cpp
auto pcf = webrtc::CreatePeerConnectionFactory(/*...*/);
webrtc::PeerConnectionInterface::RTCConfiguration config;
config.servers.push_back({.urls = {"stun:stun.l.google.com:19302"}});
config.servers.push_back({
    .urls = {"turn:turn.example.com:3478"},
    .username = "user", .password = "pass"});

auto pc = pcf->CreatePeerConnectionOrError(
    config, webrtc::PeerConnectionDependencies(&observer)).value();

pc->AddTrack(audio_track, {"stream"});
pc->AddTrack(video_track, {"stream"});

// 이후 CreateOffer → ICE gather → DTLS → SRTP 전 과정이
// 라이브러리 내부에서 자동으로 진행되고, 결과만 콜백으로 올라옴
```

### 프로토콜 스택

```
┌─────────────────────────────────────────────────┐
│           Application (audio / video / data)    │
├───────────────────────────┬─────────────────────┤
│    SRTP / SRTCP           │  SCTP               │
│    (media + feedback)     │  (DataChannel)      │
├───────────────────────────┴─────────────────────┤
│               DTLS 1.2 (with DTLS-SRTP ext)     │
├─────────────────────────────────────────────────┤
│               ICE (STUN / TURN)                 │
├─────────────────────────────────────────────────┤
│                      UDP                        │
├─────────────────────────────────────────────────┤
│                      IP                         │
└─────────────────────────────────────────────────┘
```

### 핵심 설계 원칙

| 원칙 | 구현 |
|------|------|
| 경로 최적화 | ICE가 host→srflx→relay 순으로 최적 경로 선택 |
| E2E 암호화 | 모든 미디어/데이터는 DTLS 파생 키로 SRTP/SCTP 암호화 |
| MITM 방지 | SDP의 fingerprint와 DTLS 인증서 실제 지문 비교 |
| 단일 포트 | `rtcp-mux` + `BUNDLE`로 모든 스트림이 한 포트 공유 |
| 네트워크 적응 | NACK (손실 복구), PLI (키프레임), REMB (대역폭) |
| Seamless 전환 | ICE restart로 DTLS 세션 유지한 채 경로만 교체 |

---