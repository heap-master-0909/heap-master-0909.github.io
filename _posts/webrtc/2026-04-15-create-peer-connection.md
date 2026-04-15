---
title: CreatePeerConnection 분석
date: 2026-04-15 00:00:00 +0900
categories: [Blog, WebRTC]
tags: [Tech, WebRTC, CreatePeerConnection]
pin: true
---

```cpp
bool Conductor::CreatePeerConnection() {
  RTC_DCHECK(peer_connection_factory_);
  RTC_DCHECK(!peer_connection_);

  webrtc::PeerConnectionInterface::RTCConfiguration config;
  config.sdp_semantics = webrtc::SdpSemantics::kUnifiedPlan;
  webrtc::PeerConnectionInterface::IceServer server;
  server.uri = GetPeerConnectionString();
  server.username = GetTurnUserName();
  server.password = GetTurnPassword();
  config.servers.push_back(server);

  webrtc::PeerConnectionDependencies pc_dependencies(this);
  auto error_or_peer_connection =
      peer_connection_factory_->CreatePeerConnectionOrError(
          config, std::move(pc_dependencies));
  if (error_or_peer_connection.ok()) {
    peer_connection_ = std::move(error_or_peer_connection.value());
  }
  return peer_connection_ != nullptr;
}
```

> 요건 좀 복잡해서 이후에 하겠다.