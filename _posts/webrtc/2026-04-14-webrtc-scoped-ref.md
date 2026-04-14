---
title: scoped_refptr ref count 관리법
date: 2026-04-14 00:00:00 +0900
categories: [Blog, WebRTC]
tags: [Tech, WebRTC, scoped_refptr]
pin: true
---

## 생성 절차

```cpp
auto conductor = webrtc::make_ref_counted<Conductor>(env, &client, &wnd);
```

```cpp
template <
    typename T,
    typename... Args,
    typename std::enable_if<std::is_convertible_v<T*, RefCountInterface*> &&
                                std::is_abstract_v<T>,
                            T>::type* = nullptr>
absl_nonnull scoped_refptr<T> make_ref_counted(Args&&... args) {
  return scoped_refptr<T>(new RefCountedObject<T>(std::forward<Args>(args)...));
}
```

```cpp
explicit scoped_refptr(T* absl_nullable p) : ptr_(p) {
if (ptr_)
    ptr_->AddRef();
}
```

### 헷갈리쥬? 좀 더 자세히 설명

```cpp
// 결국은 여기가 헷갈린다.
absl_nonnull scoped_refptr<T> make_ref_counted(Args&&... args) {
  return scoped_refptr<T>(new RefCountedObject<T>(std::forward<Args>(args)...));
}

// new RefCountedObject<T>(std::forward<Args>(args)...) 요걸 먼저 뜯어보자
```

```cpp
new RefCountedObject<T>(std::forward<Args>(args)...)

// T가 Conductor이고, args가 env, &client, &wnd이므로:

new RefCountedObject<Conductor>(env, &client, &wnd)

// new이므로 힙에 객체를 만들고, 그 결과는 RefCountedObject<Conductor>* — raw 포인터입니다.
```

```cpp
// 여기까지 하면,
absl_nonnull scoped_refptr<T> make_ref_counted(Args&&... args) {
  return scoped_refptr<T>(/* raw pointer */);
}
// 이런형태겠지?
```

```cpp
// scoped_refptr내에서 raw pointer는 이런식으로 관리
explicit scoped_refptr(T* absl_nullable p) : ptr_(p) {
if (ptr_)
    ptr_->AddRef();
}

template <class T>
class ABSL_NULLABILITY_COMPATIBLE scoped_refptr {
 public:
  using element_type = T;

  scoped_refptr() : ptr_(nullptr) {}
    ~scoped_refptr() {
    if (ptr_)
    // 소멸자에 Release존재
      ptr_->Release();

  // ...

 protected:
  T* ptr_;
};
```

1. `new RefCountedObject<Conductor>(...)` → 힙에 객체 생성, 내부 ref_count_ = 0
2. `scoped_refptr(T* p)` 생성자 진입 → `ptr_->AddRef()` 호출 → ref_count_ = 1
3. conductor가 포인터를 들고 있고, ref count = 1

---

## RefCountInterface 상속의 기준

사실 이건 주석에 적혀있다

```cpp
// ref_count.h
// Interfaces where refcounting is part of the public api should
// inherit this abstract interface. The implementation of these
// methods is usually provided by the RefCountedObject template class,
// applied as a leaf in the inheritance tree.
```

### 정리

* 상속해야 하는 경우: 
    * **공개 API 인터페이스**
    * 여러 곳에서 공유 참조되고, 참조 카운팅이 public API의 일부인 인터페이스 클래스
        * 예: `CreateSessionDescriptionObserver, SetSessionDescriptionObserver, AudioSourceInterface, VideoTrackSourceInterface` 등
    * 이런 인터페이스를 통해 포인터를 넘기고 받는 쪽에서 scoped_refptr로 수명을 관리해야 하므로, AddRef()/Release()가 public contract에 포함되어야 한다
* 상속하지 않는 경우: 
    * **내부 구현 클래스 또는 단일 소유 객체**
    * 소유권이 명확하고 std::unique_ptr로 충분한 경우
    * 혹은 자체적으로 AddRef()/Release()를 직접 구현하는 경우 (이 경우 make_ref_counted의 두 번째 오버로드가 사용됨)
    * 혹은 참조 카운팅이 필요 없는 일반 객체 (세 번째 오버로드에서 FinalRefCountedObject로 감쌈)

---

## 공개 API Interface?

api내의 객체는 공개 api interface이다.
