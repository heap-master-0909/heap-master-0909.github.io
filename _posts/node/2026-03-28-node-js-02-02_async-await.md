---
title: 02-02. JS 문법 - async/await
date: 2026-03-28 00:00:00 +0900
categories: [Node.JS, 문법]
tags: [Tech, Node.JS, 문법, Async, Await]
pin: true
---

## async/await

`async/await`는 Promise를 **동기 코드처럼** 읽히게 작성할 수 있는 문법이다.
ES2017(ES8)에서 도입되었으며, 내부적으로는 Promise를 사용한다.

### 기본 문법

```javascript
// async 함수는 항상 Promise를 반환한다
async function getOrderSummary(userId) {
  const user   = await getUser(userId);       // Promise가 resolve될 때까지 대기
  const orders = await getOrders(user.id);
  const detail = await getOrderDetail(orders[0].id);
  return { user: user.name, detail };
}

getOrderSummary(1)
  .then(summary => console.log(summary))
  .catch(err => console.error(err));
```

> `await`는 반드시 `async` 함수 안에서만 사용할 수 있다.

### 실전 예시 — 파일 읽기

Node.JS의 `fs.promises`를 활용한 실전 예시다. 콜백 방식과 async/await 방식을 비교해 보자.

```javascript
const fs = require('fs').promises;

// 여러 설정 파일을 읽어서 하나로 합치는 함수
async function loadConfig() {
  // ⚠️ 순차 실행: db.json을 다 읽은 뒤에 app.json을 읽기 시작한다
  const dbConfig   = await fs.readFile('./config/db.json', 'utf8');
  const appConfig  = await fs.readFile('./config/app.json', 'utf8');

  return {
    db:  JSON.parse(dbConfig),
    app: JSON.parse(appConfig),
  };
}

// 사용
async function main() {
  try {
    const config = await loadConfig();
    console.log('DB Host:', config.db.host);
    console.log('App Port:', config.app.port);
  } catch (err) {
    console.error('설정 파일 로드 실패:', err.message);
  }
}

main();
```

위 코드에서 `await`를 연속으로 쓰면 **순차 실행**이 된다. `db.json` 읽기가 끝나야 `app.json` 읽기가 시작된다.
두 파일은 서로 의존하지 않으므로, `Promise.all`로 **동시에** 읽는 것이 더 효율적이다.

```javascript
// ✅ 병렬 실행: 두 파일을 동시에 읽기 시작, 둘 다 끝나면 결과 반환
async function loadConfigFast() {
  const [dbConfig, appConfig] = await Promise.all([
    fs.readFile('./config/db.json', 'utf8'),
    fs.readFile('./config/app.json', 'utf8'),
  ]);

  return {
    db:  JSON.parse(dbConfig),
    app: JSON.parse(appConfig),
  };
}
```

```
순차: ██████ db.json ██████ → ██████ app.json ██████ → 완료  (합산 시간)
병렬: ██████ db.json  ██████ ┐
      ██████ app.json ██████ ┘→ 완료  (둘 중 긴 쪽만큼만 소요)
```

> 파일 2개 정도는 차이가 미미하지만, 네트워크 요청이나 파일이 많아질수록 병렬 처리의 성능 이점이 커진다.

### 에러 처리 — try/catch

```javascript
async function safeFetch(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    const data = await response.json();
    return data;
  } catch (err) {
    console.error('요청 실패:', err.message);
    return null;
  }
}
```

### 병렬 실행 — `await` + `Promise.all` (정리)

앞의 `loadConfig` 예시와 같이, **서로 기다릴 필요 없는** 비동기 작업을 `await`로만 이어 쓰면 순차 실행이 된다. 같은 패턴을 작업이 늘어난 경우로만 다시 보면 다음과 같다. (`fs`는 위와 같이 `require('fs').promises`로 둔다.)

```javascript
// ❌ 순차: a.json 끝 → b.json 끝 → c.json 순으로 읽음 (대기 시간이 합쳐짐)
async function readThreeSequential() {
  const a = await fs.readFile('./config/a.json', 'utf8');
  const b = await fs.readFile('./config/b.json', 'utf8');
  const c = await fs.readFile('./config/c.json', 'utf8');
  return [a, b, c];
}

// ✅ 병렬: 세 읽기를 동시에 시작, 모두 끝나면 한꺼번에 결과 수신
async function readThreeParallel() {
  const [a, b, c] = await Promise.all([
    fs.readFile('./config/a.json', 'utf8'),
    fs.readFile('./config/b.json', 'utf8'),
    fs.readFile('./config/c.json', 'utf8'),
  ]);
  return [a, b, c];
}
```

> 네트워크 `fetch`처럼 I/O가 겹칠수록 병렬화 이점이 크다. `Promise.all` 중 하나라도 reject되면 전체가 reject되므로, 일부만 실패해도 나머지 결과를 쓰고 싶다면 `Promise.allSettled`를 검토하면 된다.