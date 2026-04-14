---
title: webrtc sample 분석 (signaling)
date: 2026-04-15 00:00:00 +0900
categories: [Blog, WebRTC]
tags: [Tech, WebRTC, sample, signaling]
pin: true
---

```
●   Peer A (Caller)             Signaling Server             Peer B (Callee)
         │  1. createOffer()          │                            │
         │  2. setLocalDescription()  │                            │
         |                            │                            |                                                                                                     
         │──── SDP Offer 전송 ───────>│                            │                                                                                                     
         │                            │──── SDP Offer 전달 ───────>│
         │                            │                            │
         │                            │         3. setRemoteDescription(offer)
         │                            │         4. createAnswer()
         │                            │         5. setLocalDescription(answer)
         │                            │                            │
         │                            │<──── SDP Answer 전달 ──────│
         │<──── SDP Answer 수신 ──────│                            │
         │                            │                            │
         │  6. setRemoteDescription() │                            │
         │                            │                            │
         │  onicecandidate 발생       │              onicecandidate 발생
         │── ICE Candidate ──────────>│                            │
         │                            │── ICE Candidate ──────────>│
         │                            │                addIceCandidate()
         │                            │                            │
         │                            │<── ICE Candidate ─────────│
         │<── ICE Candidate ─────────│                            │
         │  addIceCandidate()         │                            │
         │                            │                            │
         │  ····· ICE 상태: new → checking → connected → completed ·····
         │                            │                            │
         │════ DTLS Handshake (암호화 키 교환) ═══════════════════>│
         │<═══════════════════════════════════════════════════════=│
         │                            │                            │
         │~~~ SRTP (Audio/Video) + SCTP (DataChannel) P2P 전송 ~~>│
         │<~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~│
         │                            │                            │
         ╰────────────────── P2P 연결 완료 ─────────────────────────╯

```