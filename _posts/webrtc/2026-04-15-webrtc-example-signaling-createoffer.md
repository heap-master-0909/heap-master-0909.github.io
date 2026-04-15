---
title: webrtc sample 분석 (signaling-createoffer)
date: 2026-04-15 00:00:00 +0900
categories: [Blog, WebRTC, signaling]
tags: [Tech, WebRTC, sample, signaling, createoffer]
pin: true
---

# 1. createOffer()

```cpp
void SdpOfferAnswerHandler::CreateOffer(
    CreateSessionDescriptionObserver* observer,
    const PeerConnectionInterface::RTCOfferAnswerOptions& options) {
  RTC_LOG_THREAD_BLOCK_COUNT();
  RTC_DCHECK_RUN_ON(signaling_thread());
  // Chain this operation. If asynchronous operations are pending on the chain,
  // this operation will be queued to be invoked, otherwise the contents of the
  // lambda will execute immediately.
  operations_chain_->ChainOperation(
      [this_weak_ptr = weak_ptr_factory_.GetWeakPtr(),
       observer_refptr =
           scoped_refptr<CreateSessionDescriptionObserver>(observer),
       options](std::function<void()> operations_chain_callback) {
        // Abort early if `this_weak_ptr` is no longer valid.
        if (!this_weak_ptr) {
          observer_refptr->OnFailure(
              RTCError(RTCErrorType::INTERNAL_ERROR,
                       "CreateOffer failed because the session was shut down"));
          operations_chain_callback();
          return;
        }
        // The operation completes asynchronously when the wrapper is invoked.
        auto observer_wrapper =
            make_ref_counted<CreateSessionDescriptionObserverOperationWrapper>(
                std::move(observer_refptr),
                std::move(operations_chain_callback));
        this_weak_ptr->DoCreateOffer(options, observer_wrapper);
      });
}
```

## 함수 시그니처

```cpp
void SdpOfferAnswerHandler::CreateOffer(
    CreateSessionDescriptionObserver* observer,
    const PeerConnectionInterface::RTCOfferAnswerOptions& options);
```

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `observer` | `CreateSessionDescriptionObserver*` | SDP Offer 생성 결과를 비동기로 받을 콜백 객체. 성공 시 `OnSuccess(SessionDescriptionInterface*)`, 실패 시 `OnFailure(RTCError)` 호출 |
| `options` | `RTCOfferAnswerOptions` | Offer 생성 옵션 (ICE restart 여부, 오디오/비디오 수신 여부 등) |

---

## 스레드 안전성 보장

```cpp
RTC_LOG_THREAD_BLOCK_COUNT();
RTC_DCHECK_RUN_ON(signaling_thread());
```

- **`RTC_DCHECK_RUN_ON`**: 이 함수가 반드시 **시그널링 스레드**에서만 호출되도록 디버그 시 검증한다. WebRTC에서 SDP 관련 작업은 모두 시그널링 스레드에서 수행되어야 한다.
- **`RTC_LOG_THREAD_BLOCK_COUNT`**: 스레드 블로킹 횟수를 로깅하여 성능 문제를 추적한다.

---

## Operations Chain (작업 체인)

```cpp
operations_chain_->ChainOperation(...)
```

WebRTC의 `PeerConnection`은 여러 비동기 작업(CreateOffer, CreateAnswer, SetLocalDescription, SetRemoteDescription 등)이 **순서대로** 실행되어야 한다. `operations_chain_`은 이를 보장하는 **직렬화(serialization) 메커니즘**이다.

- 체인에 이미 대기 중인 작업이 있으면 → **큐에 추가**되어 순서를 기다림
- 대기 중인 작업이 없으면 → 람다가 **즉시 실행**됨

---

## 람다 캡처 목록

```cpp
[this_weak_ptr = weak_ptr_factory_.GetWeakPtr(),
 observer_refptr = scoped_refptr<CreateSessionDescriptionObserver>(observer),
 options]
```

| 캡처 변수 | 캡처 방식 | 이유 |
|---|---|---|
| `this_weak_ptr` | `weak_ptr` | 비동기 실행 시점에 `SdpOfferAnswerHandler`가 이미 파괴되었을 수 있으므로, **dangling pointer** 방지 |
| `observer_refptr` | `scoped_refptr` (참조 카운팅) | observer의 수명을 람다가 살아있는 동안 보장 |
| `options` | 값 복사 | 경량 구조체이므로 값 복사로 충분 |

---

## 생존 검사 (Liveness Check)

```cpp
if (!this_weak_ptr) {
  observer_refptr->OnFailure(
      RTCError(RTCErrorType::INTERNAL_ERROR,
               "CreateOffer failed because the session was shut down"));
  operations_chain_callback();
  return;
}
```

큐에서 대기하다가 실행될 때, `SdpOfferAnswerHandler`가 이미 소멸되었다면:

1. observer에게 **실패를 알린다** (`OnFailure`)
2. **`operations_chain_callback()`을 반드시 호출**하여 체인의 다음 작업이 실행될 수 있도록 한다.
   - 이를 호출하지 않으면 체인 전체가 영구적으로 멈추게 된다.

---

## Observer Wrapper 패턴

```cpp
auto observer_wrapper =
    make_ref_counted<CreateSessionDescriptionObserverOperationWrapper>(
        std::move(observer_refptr),
        std::move(operations_chain_callback));
this_weak_ptr->DoCreateOffer(options, observer_wrapper);
```

`CreateSessionDescriptionObserverOperationWrapper`는 두 가지 역할을 결합한다:

1. **원래 observer에게 결과 전달**: `DoCreateOffer`가 완료되면 `OnSuccess`/`OnFailure`를 원래 observer에게 전달
2. **체인 콜백 자동 호출**: 결과 전달 후 `operations_chain_callback()`을 자동으로 호출하여, 다음 대기 중인 작업이 시작되도록 함

이 래퍼 덕분에 `DoCreateOffer` 내부에서는 체인 관리를 신경 쓸 필요 없이, 순수하게 SDP Offer 생성 로직에만 집중할 수 있다.

---

## 전체 실행 흐름

```
CreateOffer 호출
  │
  ├─ 시그널링 스레드 검증
  │
  └─ operations_chain에 작업 등록
       │
       ├─ (이전 작업이 있으면 완료 대기)
       │
       └─ 실행 시점에 this 생존 확인
            │
            ├─ [소멸됨] → OnFailure 호출 + operations_chain_callback()
            │
            └─ [생존함] → observer를 래핑 → DoCreateOffer 실행
                              │
                              └─ 완료 시: 원래 observer에 결과 전달
                                        + operations_chain_callback()
```

# 1.1 DoCreateOffer

## 호출 관계

```
PeerConnection::CreateOffer()
  └─ SdpOfferAnswerHandler::CreateOffer()          (L1716)
       └─ operations_chain_->ChainOperation(...)
            └─ SdpOfferAnswerHandler::DoCreateOffer()  (L2780) ← 이 함수
                 └─ webrtc_session_desc_factory_->CreateOffer()
```

---

## 함수 시그니처

```cpp
void SdpOfferAnswerHandler::DoCreateOffer(
    const PeerConnectionInterface::RTCOfferAnswerOptions& options,
    scoped_refptr<CreateSessionDescriptionObserver> observer);
```

| 파라미터 | 설명 |
|---|---|
| `options` | ICE restart, offer_to_receive_audio/video 등 Offer 생성 옵션 |
| `observer` | 실제로는 `CreateSessionDescriptionObserverOperationWrapper`로 래핑된 observer. 결과 전달 + 체인 콜백을 모두 처리한다. |

---

## 실행 흐름 상세

### 1단계: 기본 검증

#### (a) Observer null 체크

```cpp
if (!observer) {
  RTC_LOG(LS_ERROR) << "CreateOffer - observer is NULL.";
  return;
}
```

observer가 없으면 결과를 전달할 곳이 없으므로 즉시 리턴한다.

#### (b) PeerConnection 닫힘 체크

```cpp
if (pc_->IsClosed()) {
  std::string error = "CreateOffer called when PeerConnection is closed.";
  pc_->message_handler()->PostCreateSessionDescriptionFailure(
      observer.get(), RTCError(RTCErrorType::INVALID_STATE, ...));
  return;
}
```

PeerConnection이 이미 `Close()`된 상태에서 Offer를 생성하려고 하면 `INVALID_STATE` 에러로 실패한다.

#### (c) 세션 에러 체크

```cpp
if (session_error() != SessionError::kNone) {
  std::string error_message = GetSessionErrorMsg();
  pc_->message_handler()->PostCreateSessionDescriptionFailure(
      observer.get(), RTCError(RTCErrorType::INTERNAL_ERROR, ...));
  return;
}
```

이전 SDP 협상 과정에서 세션 에러가 발생한 경우, PeerConnection이 **비정상 상태(inconsistent state)**일 수 있으므로 즉시 실패 처리한다.

#### (d) 옵션 유효성 체크

```cpp
if (!ValidateOfferAnswerOptions(options)) {
  pc_->message_handler()->PostCreateSessionDescriptionFailure(
      observer.get(), RTCError(RTCErrorType::INVALID_PARAMETER, ...));
  return;
}
```

`offer_to_receive_audio`와 `offer_to_receive_video` 값이 유효 범위(`kUndefined` ~ `kMaxOfferToReceiveMedia`)인지 검증한다.

> **참고**: 모든 실패 처리에서 `PostCreateSessionDescriptionFailure()`를 사용한다. 이는 observer 콜백을 **비동기로 Post**하여, 호출자에게 즉시 제어권을 돌려준 뒤 나중에 실패를 알린다.

---

### 2단계: 레거시 옵션 처리 (Unified Plan 전용)

```cpp
if (IsUnifiedPlan()) {
  RTCError error = HandleLegacyOfferOptions(options);
  if (!error.ok()) {
    pc_->message_handler()->PostCreateSessionDescriptionFailure(
        observer.get(), std::move(error));
    return;
  }
}
```

WebRTC spec 4.4.3.2 "Legacy configuration extensions"에 정의된 레거시 옵션을 처리한다.

`HandleLegacyOfferOptions()`의 동작:

| `offer_to_receive_*` 값 | 동작 |
|---|---|
| `0` | 해당 미디어 타입의 수신 방향을 제거 (recvonly → inactive, sendrecv → sendonly) |
| `1` | 해당 미디어 타입의 수신 transceiver가 없으면 하나 추가 |
| `> 1` | **지원하지 않음** → `UNSUPPORTED_PARAMETER` 에러 |

이 처리는 Unified Plan에서만 적용된다. Plan B에서는 `GetOptionsForPlanBOffer()`에서 별도로 처리한다.

---

### 3단계: MediaSessionOptions 구성

```cpp
MediaSessionOptions session_options;
GetOptionsForOffer(options, &session_options);
```

`RTCOfferAnswerOptions`를 실제 SDP 생성에 필요한 `MediaSessionOptions` 구조체로 변환한다.

`GetOptionsForOffer()`가 설정하는 주요 항목:

| 항목 | 설명 |
|---|---|
| 미디어 m= 라인 구성 | Unified Plan이면 transceiver 기반, Plan B이면 sender/receiver 기반으로 m= 섹션 결정 |
| ICE restart 플래그 | `options.ice_restart`이거나 새로운 ICE credential이 있으면 활성화 |
| ICE renomination | `configuration()->enable_ice_renomination` 설정 반영 |
| RTCP CNAME | 세션의 RTCP CNAME 설정 |
| 암호화 옵션 | SRTP, SDES 등 crypto 옵션 설정 |
| Pooled ICE credentials | PortAllocator에서 사전 할당된 ICE credential 가져옴 (네트워크 스레드 BlockingCall) |
| extmap-allow-mixed | RTP 확장 헤더 혼합 허용 여부 |
| SCTP 옵션 | 구식 SCTP SDP 문법 사용 여부, SNAP 프로토콜 활성화 여부 |

---

### 4단계: Observer 래핑 및 SDP 생성 요청

```cpp
auto observer_wrapper =
    make_ref_counted<CreateDescriptionObserverWrapperWithCreationCallback>(
        [this](std::unique_ptr<SessionDescriptionInterface> desc) {
          RTC_DCHECK_RUN_ON(signaling_thread());
          last_created_offer_ = std::move(desc);
        },
        std::move(observer));
webrtc_session_desc_factory_->CreateOffer(observer_wrapper.get(), options,
                                          session_options);
```

#### `CreateDescriptionObserverWrapperWithCreationCallback`의 역할

observer를 한 번 더 래핑하여, SDP 생성 성공 시 **두 가지 일을 동시에** 처리한다:

**성공 시 (`OnSuccess`):**

1. 생성된 SDP를 `Clone()`하여 **원본은 `last_created_offer_`에 저장**
2. **복제본을 원래 observer에게 전달**

```
SDP 생성 완료
  │
  ├─ 원본 desc → last_created_offer_ (내부 캐시)
  │
  └─ Clone한 desc → observer->OnSuccess() (호출자에게 전달)
```

**실패 시 (`OnFailure`):**

1. `callback_`에 `nullptr` 전달 (`last_created_offer_` 갱신 없음)
2. 원래 observer에게 에러 전달

#### `last_created_offer_`를 저장하는 이유

`SetLocalDescription()` 호출 시, 전달된 SDP가 가장 최근에 생성된 Offer와 동일한지 비교하여 **SDP munging(변조)**을 감지하는 데 사용한다.

---

## 전체 실행 흐름도

```
DoCreateOffer(options, observer)
  │
  ├─ [검증 실패] observer가 null?
  │     └─ 로그 출력 후 리턴
  │
  ├─ [검증 실패] PeerConnection이 닫혔는가?
  │     └─ INVALID_STATE 에러 → 비동기 실패 통지
  │
  ├─ [검증 실패] 세션 에러가 있는가?
  │     └─ INTERNAL_ERROR → 비동기 실패 통지
  │
  ├─ [검증 실패] 옵션이 유효한가?
  │     └─ INVALID_PARAMETER → 비동기 실패 통지
  │
  ├─ [Unified Plan] 레거시 옵션 처리
  │     └─ offer_to_receive > 1이면 UNSUPPORTED_PARAMETER
  │
  ├─ GetOptionsForOffer() → MediaSessionOptions 구성
  │     ├─ m= 라인 결정 (Unified Plan / Plan B)
  │     ├─ ICE restart / renomination
  │     ├─ RTCP CNAME, crypto, extmap
  │     └─ Pooled ICE credentials (네트워크 스레드)
  │
  └─ webrtc_session_desc_factory_->CreateOffer()
        │
        └─ 완료 시:
             ├─ 원본 → last_created_offer_ 캐시
             └─ Clone → observer->OnSuccess() → 호출자에게 전달
                          → operations_chain_callback() → 체인 다음 작업
```

---

## Observer 래핑 체인 정리

`DoCreateOffer`에 도달하는 시점에서 observer는 이미 한 번 래핑된 상태이고, 여기서 한 번 더 래핑된다:

```
[원래 observer] (앱이 전달한 콜백)
  │
  └─ CreateSessionDescriptionObserverOperationWrapper  (CreateOffer에서 래핑)
       │  역할: 결과 전달 후 operations_chain_callback() 자동 호출
       │
       └─ CreateDescriptionObserverWrapperWithCreationCallback  (DoCreateOffer에서 래핑)
            │  역할: 성공 시 원본 SDP를 last_created_offer_에 캐시
            │
            └─ webrtc_session_desc_factory_->CreateOffer()가 최종 호출
```

결과가 돌아올 때는 **역순**으로 풀린다:

```
SDP 생성 완료
  → CreationCallback wrapper: 원본 캐시 + Clone 전달
    → OperationWrapper: 원래 observer에 전달 + 체인 콜백 호출
      → 앱의 OnSuccess()/OnFailure()
```
