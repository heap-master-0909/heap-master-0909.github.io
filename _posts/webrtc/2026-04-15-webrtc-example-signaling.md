---
title: webrtc signaling 절차 분석
date: 2026-04-15 00:00:00 +0900
categories: [Blog, WebRTC, signaling]
tags: [Tech, WebRTC, sample, signaling]
pin: true
---

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│  Peer A     │    │  Signaling   │    │ STUN / TURN │    │  Peer B     │
│  (Caller)   │    │   Server     │    │   Server    │    │  (Callee)   │
└──────┬──────┘    └──────┬───────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ①    │   Signaling : SDP Offer / Answer 교환                    │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │                  │                   │                   │
       │ createOffer()    │                   │                   │
       │─────┐            │                   │                   │
       │<────┘            │                   │                   │
       │                  │                   │                   │
       │ SDP Offer        │                   │                   │
       │ (ufrag/pwd,      │                   │                   │
       │  fingerprint,    │                   │                   │
       │  codecs)         │                   │                   │
       │─────────────────>│                   │                   │
       │                  │  Forward Offer    │                   │
       │                  │──────────────────────────────────────>│
       │                  │                   │                   │
       │                  │   SDP Answer      │                   │
       │                  │<──────────────────────────────────────│
       │ Forward Answer   │                   │                   │
       │<─────────────────│                   │                   │
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ②    │   ICE : 후보 수집 & 연결성 검사                          │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │                  │                   │                   │
       │ [host 후보 수집] │                   │  [host 후보 수집] │
       │                  │                   │                   │
       │ STUN Binding Request                 │                   │
       │─────────────────────────────────────>│                   │
       │        Binding Response (srflx 주소) │                   │
       │<─────────────────────────────────────│                   │
       │                  │                   │  STUN Binding Req │
       │                  │                   │<──────────────────│
       │                  │                   │  Response (srflx) │
       │                  │                   │──────────────────>│
       │                  │                   │                   │
       │ ICE candidates (trickle, via signaling)                  │
       │─────────────────>│──────────────────────────────────────>│
       │                  │                   │                   │
       │ STUN Binding Request (연결성 검사, USE-CANDIDATE)        │
       │─────────────────────────────────────────────────────────>│
       │      Binding Response → 후보 쌍 nominated                │
       │<─────────────────────────────────────────────────────────│
       │                  │                   │                   │
       │    ✓ ICE connected : 경로 확정 (host / srflx / relay)    │
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ③    │   DTLS Handshake : 암호화 세션 수립 (UDP 위)             │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │                  │                   │                   │
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
       │                  │                   │                   │
       │  ✓ fingerprint 검증 + SRTP keying material 추출          │
       │    (DTLS-SRTP : RFC 5764)                                │
       │                  │                   │                   │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
  ④    │   Media Flow : SRTP / SRTCP (rtcp-mux, 단일 포트)        │
═══════╪══════════════════╪═══════════════════╪═══════════════════╪═══════
       │                  │                   │                   │
       │ SRTP : 암호화된 오디오/비디오 (seq, ts, SSRC)            │
       │═════════════════════════════════════════════════════════>│
       │            SRTP : 반대 방향 미디어 스트림                │
       │<═════════════════════════════════════════════════════════│
       │                  │                   │                   │
       │       SRTCP : Receiver Report (loss, jitter, RTT)        │
       │<─────────────────────────────────────────────────────────│
       │ SRTCP : Sender Report + SDES                             │
       │─────────────────────────────────────────────────────────>│
       │   RTCP FB : NACK / PLI / REMB (재전송·키프레임·대역폭)   │
       │<─────────────────────────────────────────────────────────│
       │                  │                   │                   │
       │ SCTP over DTLS : DataChannel 메시지 (선택)               │
       │<────────────────────────────────────────────────────────>│
       │                  │                   │                   │
       │  • ICE consent freshness : 30초마다 STUN Binding 재확인  │
       │  • ICE restart : 네트워크 변경 시 ufrag/pwd 재협상       │
       │                  │                   │                   │
       ▼                  ▼                   ▼                   ▼

범례 :
  ─────>  일반 메시지(요청/응답)
  ═════>  암호화된 미디어 스트림 (SRTP)
  ①②③④  연결 수립 단계
```

---

# 실제로 오가는 데이터를 살펴보자

## 시그널링 단계 — JSON/SDP 데이터

### Alice가 시그널 서버에 로그인

```
Alice ──────────────────────────────▶ Signal Server
GET /sign_in?peer_name=alice HTTP/1.1
Host: localhost:8888
```

```
Alice ◀────────────────────────────── Signal Server
HTTP/1.1 200 OK
Pragma: 1                              ← Alice의 peer_id
Content-Length: 8
alice,1,1
```

### Bob도 로그인

```
Bob ◀─────────────────────────────── Signal Server
HTTP/1.1 200 OK
Pragma: 2                              ← Bob의 peer_id
alice,1,1
bob,2,1
```

### Alice가 Offer 생성 (createOffer) — 네트워크 나가기 전 로컬 데이터

```
Alice [로컬 생성]
┌──────────────────────────────────────────────────────────┐
│ SDP offer (실제 내용)                                      │
├──────────────────────────────────────────────────────────┤
│ v=0                                                       │
│ o=- 4611731400430051336 2 IN IP4 127.0.0.1                │
│ s=-                                                       │
│ t=0 0                                                     │
│ a=group:BUNDLE 0 1                                        │
│ a=msid-semantic: WMS stream_label                         │
│                                                           │
│ m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105     │
│ c=IN IP4 0.0.0.0                                          │
│ a=rtcp:9 IN IP4 0.0.0.0                                   │
│ a=ice-ufrag:F7gI                          ← ICE 자격     │
│ a=ice-pwd:x9cml/YzichV2+XlhiMu8g          ← ICE 비밀번호 │
│ a=fingerprint:sha-256 AB:CD:EF:12:...:9F  ← DTLS 지문    │
│ a=setup:actpass                            ← DTLS 역할   │
│ a=mid:0                                                   │
│ a=sendrecv                                                │
│ a=rtcp-mux                                                │
│ a=rtpmap:111 opus/48000/2                                 │
│ a=fmtp:111 minptime=10;useinbandfec=1                     │
│ a=ssrc:1001 cname:local_stream                            │
│                                                           │
│ m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102       │
│ c=IN IP4 0.0.0.0                                          │
│ a=ice-ufrag:F7gI                                          │
│ a=ice-pwd:x9cml/YzichV2+XlhiMu8g                          │
│ a=fingerprint:sha-256 AB:CD:EF:12:...:9F                  │
│ a=setup:actpass                                           │
│ a=mid:1                                                   │
│ a=sendrecv                                                │
│ a=rtcp-mux                                                │
│ a=rtpmap:96 VP8/90000                                     │
│ a=rtpmap:98 VP9/90000                                     │
│ a=ssrc:2001 cname:local_stream                            │
└──────────────────────────────────────────────────────────┘
```

### Alice → Signal Server → Bob (JSON으로 감싸서)

```
Alice ─────────────────────────▶ Signal Server ─────────────▶ Bob
POST /message?peer_id=1&to=2 HTTP/1.1
Content-Type: text/plain
Content-Length: ...

{
  "type": "offer",
  "sdp": "v=0\r\no=- 4611...\r\ns=-\r\nt=0 0\r\n...\r\na=setup:actpass\r\n..."
}
```

### Bob의 Answer

```
Bob ─────────────────────────▶ Signal Server ─────────────▶ Alice
{
  "type": "answer",
  "sdp": "v=0\r\no=- 7234... IN IP4 127.0.0.1\r\n
          ...
          a=ice-ufrag:B3kM\r\n                 ← Bob의 ICE ufrag
          a=ice-pwd:q7Hn/ZzmPpQ4+YnpijNv9h\r\n ← Bob의 ICE pwd
          a=fingerprint:sha-256 12:34:56:...\r\n ← Bob의 지문
          a=setup:active\r\n                    ← Bob이 DTLS Client 확정
          ..."
}
```

### ICE candidate trickle (양방향)

```
Alice ─────────────────────────▶ Signal Server ─────────────▶ Bob
{
  "sdpMid": "0",
  "sdpMLineIndex": 0,
  "candidate": "candidate:842163049 1 udp 1677729535 192.168.0.10 57842 typ host generation 0 ufrag F7gI network-id 1"
}
```

candidate 문자열을 풀어보면:

```
candidate:842163049              ← foundation (후보 식별자)
1                                 ← component (1=RTP, 2=RTCP)
udp                              ← 프로토콜
1677729535                       ← 우선순위
192.168.0.10                     ← IP (호스트 머신)
57842                            ← UDP 포트
typ host                         ← 후보 종류 (host/srflx/relay)
generation 0
ufrag F7gI                       ← 어느 ICE 세션 소속인지
network-id 1
```

그 외 추가로 trickle되는 것들:

```
{"candidate":"candidate:... udp ... 203.0.113.5 61234 typ srflx raddr 192.168.0.10 rport 57842 ..."}
                                                                  ↑
                                                       STUN으로 알아낸 공인 IP

{"candidate":"candidate:... udp ... 198.51.100.7 50000 typ relay raddr ... rport ..."}
                                                                  ↑
                                                       TURN 서버 중계 주소
```

---

## ICE 단계 — STUN Binding 패킷 바이트

### Alice → Bob: STUN Binding Request

이제 UDP 직접 통신 시작. 시그널 서버 안 거침.

```
Alice (192.168.0.10:57842) ────────UDP────────▶ Bob (192.168.0.20:61234)

┌──────────────────────────────────────────────────────────┐
│ STUN Binding Request (RFC 5389)                           │
├──────────────────────────────────────────────────────────┤
│ 0x00 0x01                        ← Message Type: Request  │
│ 0x00 0x58                        ← 메시지 길이            │
│ 0x21 0x12 0xA4 0x42              ← Magic Cookie (고정)    │
│ [12 bytes transaction ID]                                 │
│                                                           │
│ ── Attributes ──                                          │
│ USERNAME: "B3kM:F7gI"            ← BobUfrag:AliceUfrag    │
│ PRIORITY: 0x6E7F1EFF                                      │
│ ICE-CONTROLLING: 0x8B4A2...      ← "내가 통제자"          │
│ USE-CANDIDATE (필요시)                                    │
│ MESSAGE-INTEGRITY: [HMAC-SHA1]   ← Bob의 ice-pwd로 서명   │
│ FINGERPRINT: [CRC32]                                      │
└──────────────────────────────────────────────────────────┘
```

### Bob → Alice: STUN Binding Response

```
Bob (192.168.0.20:61234) ────────UDP────────▶ Alice (192.168.0.10:57842)

┌──────────────────────────────────────────────────────────┐
│ STUN Binding Success Response                             │
├──────────────────────────────────────────────────────────┤
│ 0x01 0x01                        ← Response               │
│ [같은 transaction ID]                                     │
│                                                           │
│ XOR-MAPPED-ADDRESS: 192.168.0.10:57842                    │
│                      ↑                                    │
│                "네가 나한테 보낸 출발지 주소 이거였어"      │
│ MESSAGE-INTEGRITY: [HMAC-SHA1]                            │
│ FINGERPRINT: [CRC32]                                      │
└──────────────────────────────────────────────────────────┘
```

반대 방향(Bob→Alice 체크)도 동일하게 오가면 → 이 candidate pair "writable", 이후 이 5-tuple로 모든 패킷 전송.

---

## DTLS 핸드셰이크 — 같은 UDP에 DTLS 레코드가 흐름

### 여기를 이해하기 위해선 DTLS의 이해가 필요

#### DTLS란?

DTLS(Datagram Transport Layer Security)는 TLS를 UDP 같은 비신뢰성 데이터그램 위에서 동작하도록 적응시킨 프로토콜입니다. 
TLS는 TCP의 순서 보장과 재전송에 의존하는데 UDP는 패킷 유실·순서 뒤바뀜·중복이 그대로 노출되기 때문에, DTLS가 직접 시퀀스 번호, 재전송 타이머, 메시지 프래그먼테이션, 쿠키 교환까지 전부 처리합니다. 
WebRTC에서는 미디어 스트림의 SRTP 키 교환(DTLS-SRTP)과 DataChannel 암호화에 사용됩니다.

#### 왜 쿠키가 필요한가 — UDP의 DoS 취약점

UDP는 TCP와 달리 3-way handshake가 없어서 **송신자 IP를 쉽게 위조**할 수 있습니다. 
만약 서버가 ClientHello를 받자마자 바로 세션 상태(메모리·크립토 컨텍스트)를 할당한다면, 공격자는 위조 IP로 ClientHello를 대량으로 뿌리는 것만으로 서버 자원을 고갈시킬 수 있습니다. 
더 나쁜 건 서버 응답(ServerHello + Certificate)이 ClientHello보다 훨씬 커서, 위조한 피해자 IP 쪽으로 증폭 공격(amplification attack)까지 가능해진다는 점입니다.

쿠키(HelloVerifyRequest) 동작 방식
핵심 아이디어는 "클라이언트가 정말로 그 IP에 있는지 먼저 확인한 뒤에 리소스를 할당한다" 입니다.

```
Client                                  Server
     │                                       │
     │  ① ClientHello (쿠키 없음)            │
     │──────────────────────────────────────▶│
     │                                       │──┐
     │                                       │  │  상태 저장 없이
     │                                       │  │  cookie = HMAC(비밀키, IP ‖ ...)
     │                                       │◀─┘
     │                                       │
     │  ② HelloVerifyRequest + 쿠키          │
     │◀──────────────────────────────────────│
     │                                       │
     │  ③ ClientHello + 쿠키 (재전송)        │
     │──────────────────────────────────────▶│
     │                                       │──┐
     │                                       │  │  쿠키 검증 성공
     │                                       │  │  세션 할당 (여기서 처음!)
     │                                       │◀─┘
     │                                       │
     │  ④ ServerHello, Certificate, ...      │
     │◀──────────────────────────────────────│
     │                 ...                   │
     ▼                                       ▼
```

자세히 보면, 서버는 ① 단계에서 ClientHello를 받아도 어떤 세션 상태도 만들지 않고 대신 쿠키를 계산만 합니다. 계산식은 대략

```
cookie = HMAC(서버_비밀키, 클라이언트_IP ‖ ClientHello_파라미터)
```

형태이고, 서버 비밀키는 서버 메모리에만 존재하며 클라이언트별로 따로 저장하는 값은 없기 때문에 완전히 stateless합니다. 
클라이언트가 그 쿠키를 포함해서 ClientHello를 재전송하면(③), 서버는 받은 IP와 파라미터로 HMAC을 다시 계산해서 쿠키 값을 비교합니다. 
일치할 때에만 그제서야 세션을 할당하고 ServerHello → Certificate → ClientKeyExchange → Finished 같은 정상 핸드셰이크를 이어갑니다.

### Bob → Alice: ClientHello (첫 번째, cookie 없음)

```
┌──────────────────────────────────────────────────────────┐
│ DTLS Record Header                                        │
├──────────────────────────────────────────────────────────┤
│ ContentType: 22 (Handshake)      ← 첫 바이트 = 22        │
│ Version: 0xFEFD (DTLS 1.2)                                │
│ Epoch: 0                                                  │
│ Sequence: 0                                               │
│ Length: 0x00BE                                            │
├──────────────────────────────────────────────────────────┤
│ Handshake: ClientHello                                    │
│   Version: DTLS 1.2                                       │
│   Random: [32 bytes]                                      │
│   SessionID: empty                                        │
│   Cookie: empty                           ← 아직 없음    │
│   CipherSuites:                                           │
│     - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256             │
│     - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256               │
│     - ...                                                 │
│   Extensions:                                             │
│     ┌─── use_srtp (RFC 5764) ─────────────────┐          │
│     │ SRTPProtectionProfiles:                 │          │
│     │   - SRTP_AEAD_AES_128_GCM  (0x0007)     │          │
│     │   - SRTP_AEAD_AES_256_GCM  (0x0008)     │          │
│     │   - SRTP_AES128_CM_HMAC_SHA1_80 (0x0001)│          │
│     └─────────────────────────────────────────┘          │
│     supported_groups, signature_algorithms, ...           │
└──────────────────────────────────────────────────────────┘
```

### Alice → Bob: HelloVerifyRequest

```
┌──────────────────────────────────────────────────────────┐
│ HelloVerifyRequest                                        │
│   Version: DTLS 1.2                                       │
│   Cookie: [20 bytes 랜덤]   ← DoS 방지용 챌린지          │
└──────────────────────────────────────────────────────────┘
```

### Bob → Alice: ClientHello (cookie 포함, 재전송)

```
ClientHello
   ...
   Cookie: [20 bytes Alice가 준 값]  ← 포함
   ...
```

### Alice → Bob: ServerHello + Certificate + 등등

여러 핸드셰이크 메시지가 하나의 UDP 데이터그램 안에 묶여 전송됩니다.

```
┌──────────────────────────────────────────────────────────┐
│ ServerHello                                               │
│   Version: DTLS 1.2                                       │
│   Random: [32 bytes]                                      │
│   CipherSuite: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256   │
│                ← Alice가 고른 것                          │
│   Extensions:                                             │
│     use_srtp:                                             │
│       SelectedProfile: SRTP_AEAD_AES_128_GCM ← 이거!     │
├──────────────────────────────────────────────────────────┤
│ Certificate                                               │
│   [DER-encoded X.509 cert 1개]                           │
│   Subject: CN=WebRTC                                      │
│   Issuer:  CN=WebRTC (self-signed)                        │
│   PublicKey: ECDSA P-256                                  │
│   NotBefore: 2026-04-17                                   │
│   NotAfter:  2026-05-17                                   │
│   Signature: [ECDSA 서명]                                │
├──────────────────────────────────────────────────────────┤
│ ServerKeyExchange                                         │
│   curve: secp256r1                                        │
│   ECDH public key: [65 bytes, Alice의 일시 공개키]        │
│   signature: ECDSA(above, Alice_cert_privkey)             │
├──────────────────────────────────────────────────────────┤
│ CertificateRequest                                        │
│   (양방향 인증)                                           │
├──────────────────────────────────────────────────────────┤
│ ServerHelloDone                                           │
└──────────────────────────────────────────────────────────┘
```

### Bob의 fingerprint 검증

```
Bob [로컬 계산]
  received_cert = [Alice가 보낸 DER cert]
  computed_fp = SHA-256(received_cert)
               = "AB:CD:EF:12:...:9F"

  sdp_fp = "AB:CD:EF:12:...:9F"   ← SDP offer의 a=fingerprint

  assert(computed_fp == sdp_fp)   ✔ OK
```

### Bob → Alice: Certificate, ClientKeyExchange, Finished 등

```
┌──────────────────────────────────────────────────────────┐
│ Certificate (Bob의 자가서명 cert)                         │
├──────────────────────────────────────────────────────────┤
│ ClientKeyExchange                                         │
│   ECDH public key: [65 bytes, Bob의 일시 공개키]          │
├──────────────────────────────────────────────────────────┤
│ CertificateVerify                                         │
│   [핸드셰이크 로그의 해시에 대한 Bob의 서명]              │
├──────────────────────────────────────────────────────────┤
│ [ChangeCipherSpec]              ← 여기부터 암호화 적용    │
├──────────────────────────────────────────────────────────┤
│ Finished (암호화됨)                                       │
│   verify_data = PRF(master_secret, "client finished",     │
│                     hash(handshake_log))                  │
└──────────────────────────────────────────────────────────┘
```

### 양쪽이 로컬에서 master_secret 계산

```
양쪽 모두:
  premaster_secret = ECDH(my_priv, peer_pub)
  master_secret = PRF(premaster, "master secret",
                      ClientHello.random || ServerHello.random)
                = [48 bytes, 양쪽 동일]
```

### Alice → Bob: Finished (마지막)

```
┌──────────────────────────────────────────────────────────┐
│ [ChangeCipherSpec]                                        │
│ Finished (암호화됨)                                       │
│   verify_data = PRF(master_secret, "server finished",     │
│                     hash(handshake_log))                  │
└──────────────────────────────────────────────────────────┘
```

---

## SRTP 키 유도 — 네트워크 통신 없음, 로컬 계산만

### 역시 이것도 srtp의 이해가 필요

우린 앞서 dtls를 통해 master_secret을 구했다
master_secret을 통해 srtp를 진행하게 된다

```
master_secret (DTLS의 뿌리 비밀, 공유됨)
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
  DTLS 레코드       TLS-Exporter       (다른 용도…)
  암호화용 키       label =
  (DTLS 채널         "EXTRACTOR-
   자체 보호)         dtls_srtp"
                         │
                         ▼
                 keying material (~60 B)
                         │
                    고정 순서로 쪼개기
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     master_key      master_salt    (양방향 분)
          │              │
          └──────┬───────┘
                 ▼
            SRTP KDF (AES-CM)
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
  session_    session_    session_
  enc_key     auth_key    salt
      │          │          │
      └──────────┴──────────┘
                 │
                 ▼
      실제 RTP 패킷 하나하나에 적용
      (암호화 + 무결성 태그 + IV)
```

#### SRTP란?

SRTP(Secure Real-time Transport Protocol, RFC 3711) 는 RTP(Real-time Transport Protocol)에 암호화·무결성 보호를 덧붙인 프로토콜입니다. 
RTP는 WebRTC에서 오디오·비디오 스트림을 실어 나르는 프로토콜인데, 그 자체로는 평문이라 도청·변조가 가능합니다.
SRTP는 RTP 페이로드를 AES-CTR 같은 스트림 암호로 암호화하고 HMAC-SHA1로 무결성 태그를 붙여서 이걸 막습니다. RTCP(제어 메시지)에 대해서도 동일한 걸 해주는 SRTCP가 함께 있습니다.
WebRTC에서는 "DTLS-SRTP"(RFC 5764) 방식을 씁니다. 여기서 DTLS는 오직 키 협상용으로만 쓰이고, 실제 미디어는 DTLS가 아니라 SRTP로 암호화되어 흐릅니다. 
DTLS가 ECDHE로 안전하게 공유 비밀을 만들어주면, 그 비밀에서 SRTP 키를 뽑아내는 단계가 바로 질문하신 "SRTP 키 유도"입니다.

#### 키 유도가 "로컬 계산만"인 이유
DTLS 핸드셰이크가 끝나는 순간 클라이언트와 서버는 동일한 master_secret(TLS 키잉 재료)을 각자 갖고 있습니다. 
이 값은 네트워크를 건너간 적이 없고, ECDHE의 성질상 양쪽이 독립적으로 동일한 값을 계산해낸 것입니다. 
그 뒤 SRTP 키를 뽑는 과정은 순전히 함수 계산이라 추가적인 패킷 교환이 필요 없습니다 — 같은 입력에 같은 함수를 돌리면 같은 출력이 나온다는 성질만으로 양쪽이 같은 키를 갖게 됩니다.

```
DTLS 핸드셰이크 완료 직후:

  ┌──────────────────┐                  ┌──────────────────┐
  │      Client      │                  │      Server      │
  │                  │                  │                  │
  │  master_secret   │═════ 동일 ═════ │  master_secret   │
  └────────┬─────────┘                  └────────┬─────────┘
           │                                     │
           │  ① TLS-Exporter (RFC 5705)          │  ① TLS-Exporter
           │  label = "EXTRACTOR-dtls_srtp"      │  label = "EXTRACTOR-dtls_srtp"
           ▼                                     ▼
  ┌──────────────────┐                  ┌──────────────────┐
  │  keying material │═════ 동일 ═════ │  keying material │   ~60 B
  └────────┬─────────┘                  └────────┬─────────┘
           │  ② 고정된 순서로 쪼개기             │  ② 쪼개기
           ▼                                     ▼
     client_write_SRTP_master_key   (16 B)
     server_write_SRTP_master_key   (16 B)
     client_write_SRTP_master_salt  (14 B)
     server_write_SRTP_master_salt  (14 B)
           │                                     │
           │  ③ SRTP KDF (AES-CM 기반, RFC 3711) │  ③ SRTP KDF
           ▼                                     ▼
     session_encryption_key    ← 실제 RTP 페이로드 암호화용
     session_authentication_key ← HMAC-SHA1 무결성 태그용
     session_salt              ← IV 생성용
     (SRTCP용 세트도 동일 방식으로)

           ↑ 이 ①~③ 전부 "함수 호출"만 존재.
             어느 한 바이트도 네트워크로 오가지 않습니다.
```

---

## 미디어 송수신 — SRTP 패킷 실제 바이트

### Alice → Bob: SRTP 오디오 패킷 예시

```
┌──────────────────────────────────────────────────────────┐
│ UDP Datagram (평문으로 보이는 바이트)                      │
├──────────────────────────────────────────────────────────┤
│ ── RTP Header (12 bytes, 평문) ──                         │
│ 0x80                        ← V=2, P=0, X=0, CC=0         │
│ 0xEF                        ← M=1, PT=111 (Opus)          │
│ 0x12 0x34                   ← Sequence Number = 4660      │
│ 0x00 0x00 0x3B 0xC0         ← Timestamp = 15296           │
│ 0x00 0x00 0x03 0xE9         ← SSRC = 1001                 │
│                                                           │
│ ── Encrypted Payload (N bytes) ──                         │
│ 0x7A 0xE2 0x91 0xF4 0xC8 ...   ← AES-128-GCM 암호화된     │
│ ...                              Opus 프레임               │
│                                                           │
│ ── Auth Tag (16 bytes, AEAD) ──                           │
│ 0x3D 0x8E 0x1F ... 0x9C     ← GCM 인증 태그              │
└──────────────────────────────────────────────────────────┘

첫 바이트 0x80 = 128 → "RTP/SRTP 범위" (128~191) → Bob이 SRTP 핸들러로 분기
```