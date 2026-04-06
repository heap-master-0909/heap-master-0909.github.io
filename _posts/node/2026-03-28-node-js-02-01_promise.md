---
title: 02-01. JS 문법 - promise
date: 2026-04-06 00:00:00 +0900
categories: [Node.JS, 문법]
tags: [Tech, Node.JS, 문법, Promise]
pin: true
---

## Promise

JavaScript는 **싱글 스레드** 언어이기 때문에 시간이 오래 걸리는 작업(네트워크 요청, 파일 읽기 등)을 동기적으로 처리하면 전체 코드가 멈춘다.
이를 해결하기 위해 **비동기(Asynchronous)** 처리 패턴이 발전해 왔다.

```
콜백(Callback) → Promise → async/await
```

`Promise`는 **"미래에 완료될 작업"**을 나타내는 객체다. 작업이 성공하면 `resolve`, 실패하면 `reject`를 호출한다.

### 3가지 상태

| 상태 | 설명 |
|------|------|
| **Pending** | 아직 결과가 나오지 않은 초기 상태 |
| **Fulfilled** | 작업이 성공적으로 완료됨 (`resolve` 호출) |
| **Rejected** | 작업이 실패함 (`reject` 호출) |

```
         ┌─ resolve(value) ──→ Fulfilled
Pending ─┤
         └─ reject(error)  ──→ Rejected
```

### resolve, reject

```javascript
function fetchUser(id) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (id > 0) {
        resolve({ id, name: '홍길동' });
      } else {
        reject(new Error('유효하지 않은 ID'));
      }
    }, 1000);
  });
}

// 성공
fetchUser(1)
  .then(user => console.log(user))   // { id: 1, name: '홍길동' }
  .catch(err => console.error(err));

// 실패
fetchUser(-1)
  .then(user => console.log(user))
  .catch(err => console.error(err));  // Error: 유효하지 않은 ID
```

> `resolve()`에 넘긴 값은 `.then()`으로, `reject()`에 넘긴 값은 `.catch()`로 전달된다.

---

## Promise — Callback 지옥

### 콜백 지옥이란?

비동기 작업을 콜백으로만 처리하면 중첩이 깊어져 코드가 읽기 어려워진다.

```javascript
// 콜백 지옥 예시
getUser(1, (user) => {
  getOrders(user.id, (orders) => {
    getOrderDetail(orders[0].id, (detail) => {
      getProduct(detail.productId, (product) => {
        console.log(product);  // 😵 4단 중첩
      });
    });
  });
});
```

Promise를 사용하면 `.then()` 체이닝으로 해결할 수 있다.

```javascript
getUser(1)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetail(orders[0].id))
  .then(detail => getProduct(detail.productId))
  .then(product => console.log(product))
  .catch(err => console.error(err));
```

### then 지옥

하지만 `.then()` 안에서 다시 `.then()`을 중첩하면 **then 지옥**에 빠질 수 있다.

```javascript
// then 지옥 — 이전 결과를 여러 개 참조해야 할 때 발생하기 쉽다
getUser(1).then(user => {
  return getOrders(user.id).then(orders => {
    return getOrderDetail(orders[0].id).then(detail => {
      console.log(user.name, detail);  // user를 참조하려고 중첩
    });
  });
});
```

> then 지옥을 근본적으로 해결하는 것이 바로 `async/await`이다.

---

## Promise.all

여러 비동기 작업을 **동시에** 실행하고, **모두 완료**되면 결과를 한꺼번에 받는다.
하나라도 실패하면 즉시 `reject`된다.

```javascript
const p1 = fetch('/api/users');
const p2 = fetch('/api/orders');
const p3 = fetch('/api/products');

Promise.all([p1, p2, p3])
  .then(([users, orders, products]) => {
    console.log('모두 완료:', users, orders, products);
  })
  .catch(err => {
    console.error('하나라도 실패:', err);
  });
```

### 관련 메서드 비교

| 메서드 | 동작 |
|--------|------|
| `Promise.all` | 모두 성공해야 fulfilled. 하나라도 실패하면 즉시 rejected |
| `Promise.allSettled` | 성공/실패 상관없이 모든 결과를 반환 |
| `Promise.race` | 가장 먼저 완료(성공 또는 실패)된 결과를 반환 |
| `Promise.any` | 가장 먼저 **성공**한 결과를 반환. 모두 실패하면 rejected |

```javascript
// allSettled 예시 — 실패해도 나머지 결과를 확인할 수 있다
Promise.allSettled([
  Promise.resolve('성공'),
  Promise.reject('실패'),
  Promise.resolve('성공2'),
]).then(results => {
  console.log(results);
  // [
  //   { status: 'fulfilled', value: '성공' },
  //   { status: 'rejected',  reason: '실패' },
  //   { status: 'fulfilled', value: '성공2' }
  // ]
});
```
