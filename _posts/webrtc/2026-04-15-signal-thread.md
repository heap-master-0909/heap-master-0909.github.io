---
title: signal thread 분석
date: 2026-04-15 00:00:00 +0900
categories: [Blog, WebRTC]
tags: [Tech, WebRTC, signal-thread]
pin: true
---

## signal thread 등록 절차

```cpp
bool Conductor::InitializePeerConnection() {
  RTC_DCHECK(!peer_connection_factory_);
  RTC_DCHECK(!peer_connection_);

  if (!signaling_thread_) {
    signaling_thread_ = webrtc::Thread::CreateWithSocketServer();
    signaling_thread_->Start();
  }
```

```cpp
std::unique_ptr<Thread> Thread::CreateWithSocketServer() {
  return std::unique_ptr<Thread>(new Thread(CreateDefaultSocketServer()));
}
```

```cpp
std::unique_ptr<SocketServer> CreateDefaultSocketServer() {
  return std::unique_ptr<SocketServer>(new PhysicalSocketServer);
}
```

```cpp
bool Thread::Start() {
  RTC_DCHECK(!IsRunning());

  if (IsRunning())
    return false;

  Restart();  // reset IsQuitting() if the thread is being restarted

  // Make sure that ThreadManager is created on the main thread before
  // we start a new thread.
  ThreadManager::Instance();

  owned_ = true;

#if defined(WEBRTC_WIN)
  thread_ = CreateThread(nullptr, 0, PreRun, this, 0, &thread_id_);
  if (!thread_) {
    return false;
  }
```

## signal thread인지 확인 방법

### 1단계: TLS 슬롯 할당 (생성자)

ThreadManager는 싱글톤이고, 생성자에서 OS에 TLS 슬롯을 하나 할당

```cpp
// Windows:
// thread.cc Lines 234-235
#if defined(WEBRTC_WIN)
ThreadManager::ThreadManager() : key_(TlsAlloc()) {}
// TlsAlloc()은 Windows OS에게 "TLS 슬롯 하나 줘"라고 요청하는 Win32 API입니다. 반환값은 슬롯의 인덱스(정수)이며, key_ 멤버에 저장
```

핵심은 key_는 프로세스 전체에서 하나이지만, 이 키를 통해 접근하는 값은 스레드마다 다르다는 것

### 2단계: TLS에 값 쓰기 (SetCurrentThreadInternal)

```cpp
// Windows:
// thread.cc Lines 241-243
void ThreadManager::SetCurrentThreadInternal(Thread* thread) {
  TlsSetValue(key_, thread);
}
```

### 3단계: TLS에서 값 읽기 (CurrentThread)

```cpp
// Windows:
// thread.cc Lines 237-239
Thread* ThreadManager::CurrentThread() {
  return static_cast<Thread*>(TlsGetValue(key_));
}
```

TlsGetValue(key_)는 "이 함수를 호출한 OS 스레드"의 TLS 슬롯 key_번에 저장된 값을 반환합니다. 스레드 A에서 호출하면 스레드 A가 저장한 값, 스레드 B에서 호출하면 스레드 B가 저장한 값이 나옵니다.

### 4단계: 공개 API (SetCurrentThread)

외부에서 호출하는 SetCurrentThread는 단순 TLS 저장 전에 부가 작업을 수행합니다:

```cpp
// thread.cc Lines 246-268
void ThreadManager::SetCurrentThread(Thread* thread) {
#if RTC_DLOG_IS_ON
  if (CurrentThread() && thread) {
    RTC_DLOG(LS_ERROR) << "SetCurrentThread: Overwriting an existing value?";
  }
#endif  // RTC_DLOG_IS_ON
  if (thread) {
    thread->EnsureIsCurrentTaskQueue();
  } else {
    Thread* current = CurrentThread();
    if (current) {
      current->ClearCurrentTaskQueue();
    }
  }
  SetCurrentThreadInternal(thread);
}
```

이중 등록 감지: 이미 다른 Thread*가 등록되어 있는데 또 쓰려고 하면 경고 로그
TaskQueue 연동: thread를 설정할 때 EnsureIsCurrentTaskQueue(), 해제할 때 ClearCurrentTaskQueue() 호출. Thread는 TaskQueue이기도 하므로 TaskQueue::Current()도 함께 관리
실제 TLS 저장: 최종적으로 SetCurrentThreadInternal(thread) 호출

```
전체 흐름 요약
프로세스 시작
  └─ ThreadManager 싱글톤 생성
       └─ TlsAlloc() → key_ = 7 (예시) ← OS가 슬롯 번호 하나 할당
스레드 A 생성 (PreRun 진입)
  └─ SetCurrentThread(threadA)
       └─ TlsSetValue(7, threadA)  ← 스레드 A의 7번 슬롯에 threadA 저장
스레드 B 생성 (PreRun 진입)
  └─ SetCurrentThread(threadB)
       └─ TlsSetValue(7, threadB)  ← 스레드 B의 7번 슬롯에 threadB 저장
```

이후 어디서든:
  스레드 A에서 Thread::Current() → TlsGetValue(7) → threadA
  스레드 B에서 Thread::Current() → TlsGetValue(7) → threadB
같은 키(7)를 사용하지만 OS가 스레드별로 독립된 저장 공간을 보장해주기 때문에, 각 스레드가 자기 자신의 Thread*만 정확하게 꺼내올 수 있는 구조입니다.

---

## 다른 스레드가 태스크를 넣는 방법

다른 스레드(예: 메인 스레드)가 시그널링 스레드에 작업을 요청하는 방식은 두 가지입니다:

### 방법 A: PostTask (비동기 — "맡기고 돌아감")

```cpp
// thread.cc Lines 491-506
void Thread::PostTaskImpl(absl::AnyInvocable<void() &&> task, ...) {
  if (IsQuitting()) return;
  {
    MutexLock lock(&mutex_);
    messages_.push_back(std::move(task));  // 큐에 넣고
    WakeUpSocketServer();                  // 잠자는 스레드를 깨움
  }
}
```

### 방법 B: BlockingCall (동기 — "끝날 때까지 기다림")

```cpp
// thread.cc Lines 759-791
void Thread::BlockingCallImpl(FunctionView<void()> functor, ...) {
  // ...
  Event done;
  absl::Cleanup cleanup = [&done] { done.Set(); };
  PostTask([functor, cleanup = std::move(cleanup)] { functor(); });
  done.Wait(Event::kForever);  // 태스크가 실행 완료될 때까지 호출자가 블록
}
```

내부적으로 PostTask로 태스크를 넣고, Event로 완료를 기다립니다.

전체 흐름을 시퀀스로 보면

```
메인 스레드                         시그널링 스레드
─────────                         ──────────────
signaling_thread_->Start()   ──→  [OS 스레드 생성]
                                   PreRun()
                                   ├─ SetCurrentThread(this)
                                   └─ Run()
                                       └─ ProcessMessages(kForever)
                                           └─ Get() → 큐 비었음 → 잠듦 💤
signaling_thread_->PostTask(  ──→  messages_에 태스크 추가
  []{ CreateOffer(); }             WakeUpSocketServer() → 깨어남! ⏰
)                                  Get() → 태스크 꺼냄
                                   Dispatch() → CreateOffer() 실행
                                   Get() → 큐 비었음 → 다시 잠듦 💤
signaling_thread_->PostTask(  ──→  또 깨어남 → 실행 → 잠듦
  []{ SetRemoteDesc(); }
)
                                   ... 반복 ...
signaling_thread_->Quit()    ──→  stop_ = 1
                                   Get() → IsQuitting() → nullptr 반환
                                   ProcessMessages() 종료
                                   Run() 리턴 → 스레드 종료
```
