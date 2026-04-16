---
title: PeerConnectionObserver
date: 2026-04-16 00:00:00 +0900
categories: [Blog, WebRTC, signaling]
tags: [Tech, WebRTC, sample, signaling, recv-sdp]
pin: true
---

# PeerConnectionObserver 동작 정리

WebRTC 체크아웃 소스(`src/api`, `src/pc`) 기준으로 `PeerConnectionObserver`가 **어디에 등록되고**, **`OnIceCandidate` 등 OnXXX 이벤트가 어떻게 호출되는지**를 정리한다.

---

## 1. 역할

`PeerConnectionObserver`는 **애플리케이션이 구현하는 콜백 인터페이스**로, `PeerConnection` 내부 상태 변화·ICE·미디어·데이터 채널 등을 앱에 알린다.

공개 API(`PeerConnectionInterface`)에는 **모든 `PeerConnectionObserver` 콜백이 시그널링 스레드에서 호출된다**는 설명이 있다.

---

## 2. 인터페이스 개요

`api/peer_connection_interface.h`에 정의된다. 예:

- `OnSignalingChange` — 시그널링 상태 변경
- `OnIceGatheringChange` — ICE 수집 상태 변경
- `OnIceCandidate` — 로컬 ICE 후보가 수집됨
- `OnIceCandidateError`, `OnIceCandidateRemoved` 등
- `OnDataChannel`, `OnAddTrack` / `OnTrack`, `OnRemoveTrack` 등

`OnIceCandidate`는 **순수 가상**이므로 구현 클래스에서 반드시 오버라이드한다.

---

## 3. 등록 경로

### 3.1 `PeerConnectionDependencies`로 포인터 전달

`PeerConnection` 생성 시 **`PeerConnectionDependencies`**에 `PeerConnectionObserver*`를 넣는다.

```cpp
webrtc::PeerConnectionDependencies pc_dependencies(observer_pointer);
auto error_or_pc = peer_connection_factory_->CreatePeerConnectionOrError(
    config, std::move(pc_dependencies));
```

`PeerConnectionDependencies` 생성자는 `observer` 멤버에 포인터를 저장한다 (`api/peer_connection_interface.cc`).

### 3.2 팩토리에서 검증

`PeerConnectionFactory::CreatePeerConnectionOrError`는 `dependencies.observer == nullptr`이면 `RTCError`로 생성을 거절한다 (`pc/peer_connection_factory.cc`).

### 3.3 `PeerConnection`이 raw 포인터로 보관

`PeerConnection` 생성자에서 `observer_(dependencies.observer)`로 멤버에 저장한다 (`pc/peer_connection.cc`).

- **소유권은 없음** — 앱이 `PeerConnectionObserver` 구현체의 수명을 관리한다.
- **`Close()`가 끝난 뒤**에는 observer를 더 이상 사용하지 않으며, 구현에서는 `observer_ = nullptr`로 끊는다.
- 공개 API 주석: `Close()` 완료 후에는 observer 객체를 안전하게 파괴할 수 있다.

### 3.4 예제: `Conductor`

`examples/peerconnection/client/conductor.cc`에서는 `PeerConnectionDependencies(this)`로 **자기 자신(`Conductor`)**을 넘긴다. `Conductor`는 `PeerConnectionObserver`를 상속한다.

---

## 4. 콜백 디스패치: `RunWithObserver`

내부적으로 대부분의 관찰자 알림은 `PeerConnection::RunWithObserver`를 통해 전달된다 (`pc/peer_connection.cc`).

동작 요약:

- **`RTC_DCHECK_RUN_ON(signaling_thread())`** — 시그널링 스레드에서만 실행.
- **`RTC_DCHECK(observer_)`** — 유효한 observer가 있을 때만 사용.
- **`std::move(task)(observer_)`** — 람다 안에서 `observer_->OnXXX(...)`를 **동기적으로** 직접 호출.

따라서 **별도 비동기 큐로 “나중에” 부르는 것이 아니라**, 해당 코드 경로가 시그널링 스레드에서 실행될 때 **즉시** 콜백이 호출된다는 점이 중요하다.

---

## 5. `OnIceCandidate`가 호출되기까지 (한 줄기)

1. ICE/트랜스포트 스택이 로컬 후보를 모은다.
2. `PeerConnection::OnTransportControllerCandidatesGathered`가 호출된다.
3. `IceCandidate` 객체를 만들고 `sdp_handler_->AddLocalIceCandidate`로 JSEP 상태와 맞춘 뒤, `PeerConnection::OnIceCandidate(std::move(candidate))`를 호출한다.
4. `PeerConnection::OnIceCandidate`는 (연결이 닫히지 않았다면) `RunWithObserver`로 **`observer_->OnIceCandidate(candidate.get())`** 를 호출한다.

참고: 소스에 **후보 수집 알림이 향후 네트워크 스레드에서 올 수 있다**는 TODO가 있어, 내부 진입 스레드는 바뀔 수 있으나, **관찰자에게 전달할 때는 시그널링 스레드 규칙 + `RunWithObserver` 패턴**이 맞춰질 것으로 예상할 수 있다.

---

## 6. 다른 `OnXXX` 이벤트 (개략)

| 영역 | 설명 |
|------|------|
| ICE 상태 | `SetIceConnectionState` 등에서 `RunWithObserver`로 `OnIceConnectionChange` 등 |
| ICE gathering | `OnIceGatheringChange`도 `RunWithObserver` |
| 세션/SDP | `pc/sdp_offer_answer.cc` 등에서 `pc_->RunWithObserver`로 시그널링 관련 콜백 |
| RTP/트랙 | `RtpTransmissionManager`가 생성 시 받은 `observer_`로 `OnAddTrack` / `OnRemoveTrack` 등 |
| DataChannel | `data_channel_controller.cc` 등에서 `RunWithObserver`로 `OnDataChannel` |

즉, **여러 서브시스템이 동일한 `observer_` raw 포인터로 이벤트를 보내며**, 공통적으로 **시그널링 스레드에서의 `RunWithObserver`** 또는 동일한 스레드 제약을 따른다.

---

## 7. 예제 앱(`Conductor`)과의 관계

| 항목 | 내용 |
|------|------|
| 등록 | `PeerConnectionDependencies(this)` → 팩토리가 `PeerConnection` 생성 시 `observer_`에 `Conductor*` 저장 |
| 수명 | `Conductor` 인스턴스는 해당 `PeerConnection`이 `Close()` 완료 전까지 유효해야 함 |
| 스레드 | `OnIceCandidate` 등은 시그널링 스레드에서 동기 호출 — UI/다른 스레드로 넘기려면 앱에서 큐잉 |
| 앱 로직 | 예: `Conductor::OnIceCandidate`에서 JSON 직렬화 후 시그널링 서버로 전송 (`conductor.cc`) |

---

## 8. 한 줄 요약

**`PeerConnectionObserver`는 `PeerConnectionDependencies`로 팩토리에 넘기고, `PeerConnection`이 raw 포인터 `observer_`로 유지하며, 내부 이벤트 발생 시 시그널링 스레드에서 `RunWithObserver`를 통해 `observer_->OnIceCandidate` 등을 동기 호출한다.**

---

## 참고 파일

- `src/api/peer_connection_interface.h` — `PeerConnectionObserver`, `PeerConnectionDependencies`, `signaling_thread()` 설명
- `src/api/peer_connection_interface.cc` — `PeerConnectionDependencies` 생성자
- `src/pc/peer_connection_factory.cc` — `CreatePeerConnectionOrError`, observer null 검사
- `src/pc/peer_connection.cc` — `observer_` 저장, `RunWithObserver`, `OnIceCandidate`, `Close()`에서 `observer_ = nullptr`
- `src/examples/peerconnection/client/conductor.h` / `conductor.cc` — 예제 구현체
