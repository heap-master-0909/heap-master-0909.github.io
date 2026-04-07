---
title: 04-03. Session
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, HTTP-SERVER, Session]
tags: [Tech, Node.JS, HTTP-SERVER, Session]
pin: true
---

> 쿠키만으로 사용자 상태를 관리하면, 민감한 정보가 클라이언트에 노출되고 변조 위험이 있다. **세션(Session)**은 상태 정보를 **서버 측**에 저장하고, 클라이언트에는 **세션 ID**만 전달하여 이 문제를 해결한다.

## 세션이란?

세션은 **서버가 클라이언트별로 유지하는 상태 저장 공간**이다. 클라이언트가 최초 접속하면 서버는 고유한 세션 ID를 생성하고, 이 ID를 쿠키에 담아 클라이언트에 전달한다. 이후 클라이언트가 요청할 때마다 세션 ID 쿠키가 함께 전송되어, 서버는 해당 세션을 조회하여 사용자를 식별한다.

```
┌─ 브라우저 ─┐                          ┌─ 서버 ─────────────────┐
│            │ ── 1. 최초 요청 ────────→ │                        │
│            │                          │  세션 생성              │
│            │                          │  sessions[abc123] = {} │
│            │ ←── 2. 응답 ──────────── │                        │
│            │   Set-Cookie: sid=abc123 │                        │
│  쿠키 저장  │                          │                        │
│  sid=abc123│                          │                        │
│            │ ── 3. 재요청 ──────────→ │                        │
│            │   Cookie: sid=abc123     │  세션 조회              │
│            │                          │  sessions[abc123]      │
│            │ ←── 4. 응답 ──────────── │  → 사용자 식별!         │
└────────────┘                          └────────────────────────┘
```

### 핵심 원리

| 구분 | 내용 |
|------|------|
| **저장 위치** | 상태 데이터는 서버에, 세션 ID만 클라이언트에 저장 |
| **식별 방식** | 쿠키에 담긴 세션 ID로 서버의 세션 데이터를 조회 |
| **수명** | 서버에서 만료 시간을 제어 (일정 시간 미사용 시 자동 삭제) |

---

## 쿠키 방식 vs 세션 방식

이전 포스트의 로그인 예제에서는 쿠키에 사용자 이름을 직접 저장했다. 이 방식의 문제점을 살펴보자.

### 쿠키에 직접 저장하는 방식의 문제

```
[쿠키 방식 — 위험!]

브라우저                          서버
  │  POST /login (name=홍길동) →  │
  │  ←── Set-Cookie: user=홍길동  │
  │                               │
  │  (브라우저 개발자 도구에서       │
  │   user=관리자 로 변조!)         │
  │                               │
  │  GET / (Cookie: user=관리자) → │  "환영합니다, 관리자님!"
```

- 클라이언트가 쿠키 값을 **자유롭게 조회·변조**할 수 있다
- 민감한 정보(이름, 권한 등)가 **그대로 노출**된다

### 세션 방식으로 개선

```
[세션 방식 — 안전!]

브라우저                          서버
  │  POST /login (name=홍길동) →  │  세션 저장: { sid_x: { user: '홍길동' } }
  │  ←── Set-Cookie: sid=sid_x   │
  │                               │
  │  (브라우저에서 보이는 건         │
  │   sid=sid_x 뿐, 의미 없는 값)  │
  │                               │
  │  GET / (Cookie: sid=sid_x) →  │  sessions[sid_x].user → '홍길동'
```

- 클라이언트는 **세션 ID만** 가지고 있으며, 실제 데이터에 접근할 수 없다
- 세션 ID를 변조해도 서버에 해당 세션이 없으면 **무효 처리**된다

---

## Node.JS에서 세션 구현하기

`http` 모듈만으로 세션을 구현해 보자. 핵심은 세 가지다.

1. **고유한 세션 ID 생성**
2. **서버 메모리에 세션 데이터 저장**
3. **쿠키로 세션 ID 주고받기**

### 세션 ID 생성

```javascript
const crypto = require('crypto');

function generateSessionId() {
  return crypto.randomBytes(32).toString('hex');
}
```

> `crypto.randomBytes()`는 암호학적으로 안전한 난수를 생성한다. 예측 불가능한 세션 ID를 만들기 위해 `Math.random()` 대신 사용해야 한다.

### 쿠키 파싱 유틸리티

이전 포스트에서 만든 쿠키 파싱 함수를 재사용한다.

```javascript
function parseCookies(cookieHeader) {
  const cookies = {};
  if (!cookieHeader) return cookies;
  cookieHeader.split(';').forEach((pair) => {
    const [key, ...rest] = pair.trim().split('=');
    cookies[key] = decodeURIComponent(rest.join('='));
  });
  return cookies;
}
```

---

## 실습: 세션 기반 로그인 서버

```javascript
const http = require('http');
const crypto = require('crypto');

function parseCookies(cookieHeader) {
  const cookies = {};
  if (!cookieHeader) return cookies;
  cookieHeader.split(';').forEach((pair) => {
    const [key, ...rest] = pair.trim().split('=');
    cookies[key] = decodeURIComponent(rest.join('='));
  });
  return cookies;
}

function generateSessionId() {
  return crypto.randomBytes(32).toString('hex');
}

// 세션 저장소 (메모리)
const sessions = {};

const server = http.createServer((req, res) => {
  const cookies = parseCookies(req.headers.cookie);
  const { url, method } = req;

  // 로그인 페이지
  if (url === '/login' && method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`
      <form method="POST" action="/login">
        <label>이름: <input name="name" /></label>
        <button type="submit">로그인</button>
      </form>
    `);
    return;
  }

  // 로그인 처리
  if (url === '/login' && method === 'POST') {
    let body = '';
    req.on('data', (chunk) => { body += chunk; });
    req.on('end', () => {
      const params = new URLSearchParams(body);
      const name = params.get('name');

      const sid = generateSessionId();
      sessions[sid] = {
        user: name,
        createdAt: Date.now(),
      };

      res.writeHead(302, {
        'Set-Cookie': `sid=${sid}; Path=/; HttpOnly`,
        Location: '/',
      });
      res.end();
    });
    return;
  }

  // 로그아웃
  if (url === '/logout') {
    const sid = cookies.sid;
    if (sid) {
      delete sessions[sid];
    }
    res.writeHead(302, {
      'Set-Cookie': 'sid=; Path=/; HttpOnly; Max-Age=0',
      Location: '/',
    });
    res.end();
    return;
  }

  // 홈 페이지 — 세션 확인
  const session = sessions[cookies.sid];

  if (session) {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`
      <h1>환영합니다, ${session.user}님!</h1>
      <p>세션 ID: ${cookies.sid.slice(0, 8)}...</p>
      <a href="/logout">로그아웃</a>
    `);
  } else {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`
      <h1>로그인이 필요합니다</h1>
      <a href="/login">로그인 하기</a>
    `);
  }
});

server.listen(3000, () => {
  console.log('세션 서버: http://localhost:3000');
});
```

### 실행 흐름

```
[세션 로그인 흐름]

GET /login               → 로그인 폼 표시
POST /login (name=홍길동) → 세션 생성: sessions[abc...] = { user: '홍길동' }
                         → Set-Cookie: sid=abc... → 302 → /
GET / (Cookie: sid=abc.) → sessions[abc...].user → "환영합니다, 홍길동님!"
GET /logout              → delete sessions[abc...] 
                         → Set-Cookie: sid=; Max-Age=0 → 302 → /
GET / (쿠키 없음)         → "로그인이 필요합니다"
```

### 쿠키 방식과 비교

| 비교 항목 | 이전 포스트 (쿠키 방식) | 현재 (세션 방식) |
|-----------|------------------------|-----------------|
| 쿠키 내용 | `user=홍길동` | `sid=a3f8c2...` |
| 데이터 저장 위치 | 클라이언트 | 서버 |
| 변조 위험 | 쿠키 값을 바꾸면 다른 사용자로 위장 가능 | 세션 ID를 추측하기 극히 어려움 |
| 로그아웃 | 쿠키 삭제만으로 처리 | 서버 세션 삭제 + 쿠키 삭제 |

---

## 세션 만료 처리

세션은 무한히 유지되면 메모리가 계속 늘어난다. 일정 시간 이후 자동으로 만료시켜야 한다.

```javascript
const SESSION_MAX_AGE = 30 * 60 * 1000; // 30분

function cleanExpiredSessions() {
  const now = Date.now();
  for (const sid in sessions) {
    if (now - sessions[sid].createdAt > SESSION_MAX_AGE) {
      delete sessions[sid];
    }
  }
}

// 주기적으로 만료 세션 정리
setInterval(cleanExpiredSessions, 60 * 1000);
```

또한 요청이 들어올 때마다 세션의 유효성을 검사할 수도 있다.

```javascript
function getSession(sid) {
  const session = sessions[sid];
  if (!session) return null;

  if (Date.now() - session.createdAt > SESSION_MAX_AGE) {
    delete sessions[sid];
    return null;
  }

  // 세션 갱신 (활동 시마다 수명 연장)
  session.createdAt = Date.now();
  return session;
}
```

> 사용자가 활동할 때마다 `createdAt`을 갱신하면, **슬라이딩 만료(sliding expiration)** 방식이 된다. 마지막 활동으로부터 30분이 지나야 만료되므로 사용성이 좋아진다.

---

## 세션 저장소

위 예제에서는 `sessions` 객체(메모리)에 세션을 저장했다. 이 방식의 한계를 알아보자.

### 메모리 저장의 한계

| 문제 | 설명 |
|------|------|
| **서버 재시작 시 소멸** | 프로세스가 종료되면 모든 세션이 사라진다 |
| **메모리 제한** | 동시 접속자가 많으면 메모리 부족 발생 |
| **다중 서버 불가** | 서버가 여러 대면 세션을 공유할 수 없다 |

### 외부 저장소 사용

실무에서는 세션을 외부 저장소에 보관한다.

```
┌── 서버 A ──┐
│            │──┐
└────────────┘  │     ┌─── 세션 저장소 ───┐
                ├───→ │  Redis / DB       │
┌── 서버 B ──┐  │     │  sid_1: {...}     │
│            │──┘     │  sid_2: {...}     │
└────────────┘        └──────────────────┘
```

| 저장소 | 특징 |
|--------|------|
| **Redis** | 인메모리 DB, 빠른 읽기/쓰기, TTL 지원 (가장 많이 사용) |
| **데이터베이스** | MySQL, MongoDB 등에 세션 테이블 저장, 영구 보관 가능 |
| **파일 시스템** | 파일로 저장, 구현이 간단하지만 성능 낮음 |

---

## express-session

Express에서는 `express-session` 미들웨어로 세션을 간단하게 관리할 수 있다.

```bash
$ npm install express express-session
```

```javascript
const express = require('express');
const session = require('express-session');

const app = express();
app.use(express.urlencoded({ extended: false }));

app.use(session({
  secret: 'my-secret-key',    // 세션 ID 서명용 비밀 키
  resave: false,               // 변경 없어도 세션을 다시 저장할지
  saveUninitialized: false,    // 초기화되지 않은 세션을 저장할지
  cookie: {
    httpOnly: true,
    secure: false,             // HTTPS에서만 전송 (개발 시 false)
    maxAge: 30 * 60 * 1000,    // 30분
  },
}));

// 로그인 페이지
app.get('/login', (req, res) => {
  res.send(`
    <form method="POST" action="/login">
      <label>이름: <input name="name" /></label>
      <button type="submit">로그인</button>
    </form>
  `);
});

// 로그인 처리
app.post('/login', (req, res) => {
  req.session.user = req.body.name;
  res.redirect('/');
});

// 로그아웃
app.get('/logout', (req, res) => {
  req.session.destroy(() => {
    res.redirect('/');
  });
});

// 홈
app.get('/', (req, res) => {
  if (req.session.user) {
    res.send(`
      <h1>환영합니다, ${req.session.user}님!</h1>
      <a href="/logout">로그아웃</a>
    `);
  } else {
    res.send(`
      <h1>로그인이 필요합니다</h1>
      <a href="/login">로그인 하기</a>
    `);
  }
});

app.listen(3000, () => {
  console.log('Express 세션 서버: http://localhost:3000');
});
```

### express-session 주요 옵션

| 옵션 | 설명 |
|------|------|
| **secret** | 세션 ID 쿠키를 서명하는 비밀 키. 노출되면 안 된다 |
| **resave** | 요청 중 세션이 수정되지 않아도 저장소에 다시 저장할지 여부 |
| **saveUninitialized** | 아무 데이터도 없는 빈 세션을 저장소에 저장할지 여부 |
| **cookie** | 세션 ID 쿠키의 속성 (HttpOnly, Secure, MaxAge 등) |
| **store** | 세션 저장소 지정 (기본값: 메모리, Redis 등으로 교체 가능) |

> `resave: false`, `saveUninitialized: false`는 불필요한 세션 저장을 방지하고, 사용자 동의 없이 추적 쿠키를 설정하지 않도록 한다.

### Redis 저장소 연동

```bash
$ npm install connect-redis redis
```

```javascript
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const redisClient = createClient();
redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, maxAge: 30 * 60 * 1000 },
}));
```

---

## 세션 보안 주의사항

### 세션 하이재킹 방지

세션 하이재킹은 공격자가 다른 사용자의 세션 ID를 탈취하여 해당 사용자로 위장하는 공격이다.

| 대응 방법 | 설명 |
|-----------|------|
| **HttpOnly 쿠키** | JavaScript로 세션 ID 쿠키 접근 불가 → XSS 방어 |
| **Secure 쿠키** | HTTPS에서만 쿠키 전송 → 도청 방어 |
| **세션 재생성** | 로그인 성공 시 새 세션 ID 발급 → 세션 고정 공격 방어 |
| **짧은 만료 시간** | 세션 유효 기간을 최소화 |

### 세션 고정(Session Fixation) 공격

공격자가 미리 알고 있는 세션 ID를 피해자에게 사용하도록 유도하는 공격이다.

```
[세션 고정 공격]

공격자                              서버
  │  1. 세션 ID 획득 (sid=evil) →   │  sessions[evil] = {}
  │                                 │
  │  2. 피해자에게 sid=evil 쿠키를    │
  │     심은 링크 전달                │
  │                                 │
피해자                              서버
  │  3. 로그인 (Cookie: sid=evil) → │  sessions[evil] = { user: '피해자' }
  │                                 │
공격자                              서버
  │  4. sid=evil로 요청 ──────────→ │  sessions[evil].user → '피해자'
  │                                 │  → 피해자로 위장 성공!
```

**대응**: 로그인 성공 시 반드시 **새 세션 ID를 발급**한다.

```javascript
app.post('/login', (req, res) => {
  // 기존 세션 파기 후 새 세션 생성
  req.session.regenerate((err) => {
    req.session.user = req.body.name;
    res.redirect('/');
  });
});
```
