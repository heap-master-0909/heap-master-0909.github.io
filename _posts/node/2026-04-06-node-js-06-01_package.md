---
title: 06-01. express server
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, express]
tags: [Tech, Node.JS, express]
pin: true
---

> **Express**는 Node.JS에서 가장 널리 쓰이는 웹 프레임워크로, `http` 모듈의 복잡함을 추상화하여 간결하고 유연한 서버를 구축할 수 있게 해준다.

## Express란?

Express는 **미니멀하고 유연한** Node.JS 웹 애플리케이션 프레임워크다. 라우팅, 미들웨어, 요청/응답 처리 등 웹 서버에 필요한 핵심 기능을 제공한다.

```
http 모듈 vs Express
──────────────────────────────────────────────
http 모듈:  저수준, 모든 것을 직접 구현
Express:   고수준, 라우팅·미들웨어·파싱 내장
──────────────────────────────────────────────

http 모듈                      Express
├─ if/else로 라우팅            ├─ app.get(), app.post() 등
├─ 수동 body 파싱              ├─ express.json() 자동 파싱
├─ 수동 Content-Type 설정      ├─ res.json(), res.send() 자동 처리
└─ 공통 처리 직접 구현          └─ app.use() 미들웨어 체계
```

---

## 설치 및 프로젝트 셋업

```bash
# 프로젝트 생성
$ mkdir my-express-app && cd my-express-app
$ npm init -y

# Express 설치
$ npm i express

# 개발 편의 패키지 (선택)
$ npm i -D nodemon
```

### package.json 스크립트 설정

```json
{
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  }
}
```

```
프로젝트 구조:
my-express-app/
├── node_modules/
├── app.js           ← 서버 진입점
├── package.json
└── package-lock.json
```

---

## 기본 서버 만들기

### 최소 코드

```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello, Express!');
});

app.listen(3000, () => {
  console.log('서버 실행 중: http://localhost:3000');
});
```

```bash
$ node app.js
서버 실행 중: http://localhost:3000
```

### http 모듈과 비교

```javascript
// http 모듈 — 라우팅을 if/else로 처리
const http = require('http');
const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/') {
    res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
    res.end('Hello');
  } else {
    res.writeHead(404);
    res.end('Not Found');
  }
});
server.listen(3000);
```

```javascript
// Express — 메서드별 라우팅이 명확하게 분리됨
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello');
});

app.listen(3000);
```

> Express에서 `res.send()`는 Content-Type을 자동으로 설정하고, 문자열이면 `text/html`, 객체면 `application/json`으로 응답한다.

---

## 라우팅

라우팅은 특정 **HTTP 메서드**와 **URL 경로**에 대해 어떤 처리를 할지 정의하는 것이다.

### 기본 라우팅

```javascript
// HTTP 메서드별 라우팅
app.get('/users', (req, res) => {
  res.send('사용자 목록 조회');
});

app.post('/users', (req, res) => {
  res.send('사용자 생성');
});

app.put('/users/:id', (req, res) => {
  res.send(`사용자 ${req.params.id} 수정`);
});

app.delete('/users/:id', (req, res) => {
  res.send(`사용자 ${req.params.id} 삭제`);
});
```

### 라우트 파라미터

URL에 `:파라미터명`을 사용하면 동적 경로를 처리할 수 있다.

```javascript
app.get('/users/:id', (req, res) => {
  const { id } = req.params;
  res.send(`사용자 ID: ${id}`);
});

// 여러 파라미터
app.get('/users/:userId/posts/:postId', (req, res) => {
  const { userId, postId } = req.params;
  res.json({ userId, postId });
});
```

```
요청: GET /users/42/posts/7
req.params → { userId: '42', postId: '7' }
```

### 쿼리 스트링

`req.query`로 쿼리 파라미터에 접근한다.

```javascript
app.get('/search', (req, res) => {
  const { keyword, page } = req.query;
  res.json({ keyword, page });
});
```

```
요청: GET /search?keyword=node&page=2
req.query → { keyword: 'node', page: '2' }
```

### app.route() — 같은 경로의 메서드 묶기

```javascript
app.route('/articles')
  .get((req, res) => {
    res.send('글 목록 조회');
  })
  .post((req, res) => {
    res.send('글 작성');
  });

app.route('/articles/:id')
  .get((req, res) => {
    res.send(`글 ${req.params.id} 조회`);
  })
  .put((req, res) => {
    res.send(`글 ${req.params.id} 수정`);
  })
  .delete((req, res) => {
    res.send(`글 ${req.params.id} 삭제`);
  });
```

---

## 미들웨어

미들웨어는 요청(req)과 응답(res) 사이에서 실행되는 함수다. Express 앱의 핵심 구조이다.

### 미들웨어 동작 원리

```
클라이언트 요청
  │
  ▼
미들웨어 1 (로깅)
  │ next()
  ▼
미들웨어 2 (인증)
  │ next()
  ▼
미들웨어 3 (body 파싱)
  │ next()
  ▼
라우트 핸들러
  │
  ▼
클라이언트 응답
```

### 미들웨어 기본 형태

```javascript
function myMiddleware(req, res, next) {
  // 처리 로직
  console.log(`${req.method} ${req.url}`);
  next(); // 다음 미들웨어로 전달
}

app.use(myMiddleware);
```

> `next()`를 호출하지 않으면 요청이 해당 미들웨어에서 멈추고, 클라이언트는 응답을 받지 못한다.

### 애플리케이션 레벨 미들웨어

```javascript
const express = require('express');
const app = express();

// 모든 요청에 적용
app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next();
});

// 특정 경로에만 적용
app.use('/api', (req, res, next) => {
  console.log('API 요청 감지');
  next();
});

app.get('/', (req, res) => {
  res.send('홈');
});

app.get('/api/data', (req, res) => {
  res.json({ message: 'API 데이터' });
});

app.listen(3000);
```

### 내장 미들웨어

Express는 자주 쓰는 기능을 내장 미들웨어로 제공한다.

```javascript
// JSON 요청 본문 파싱
app.use(express.json());

// URL-encoded 폼 데이터 파싱
app.use(express.urlencoded({ extended: true }));

// 정적 파일 제공
app.use(express.static('public'));
```

| 미들웨어 | 설명 |
|----------|------|
| `express.json()` | `Content-Type: application/json` 본문 파싱 |
| `express.urlencoded()` | 폼 데이터 (`application/x-www-form-urlencoded`) 파싱 |
| `express.static()` | 정적 파일(HTML, CSS, 이미지 등) 제공 |

### 서드파티 미들웨어

```bash
$ npm i morgan cors helmet
```

```javascript
const morgan = require('morgan');
const cors = require('cors');
const helmet = require('helmet');

app.use(morgan('dev'));   // HTTP 요청 로깅
app.use(cors());          // CORS 허용
app.use(helmet());        // 보안 헤더 설정
```

```
morgan 출력 예시:
GET /api/users 200 3.241 ms - 156
POST /api/users 201 1.523 ms - 42
```

| 패키지 | 설명 |
|--------|------|
| **morgan** | HTTP 요청 로그를 자동으로 출력 |
| **cors** | Cross-Origin 요청 허용 설정 |
| **helmet** | 보안 관련 HTTP 헤더 자동 설정 |
| **cookie-parser** | 쿠키 파싱 |
| **express-session** | 세션 관리 |

---

## 요청(Request) 객체

Express의 `req` 객체는 `http.IncomingMessage`를 확장한 것으로, 다양한 편의 속성을 제공한다.

| 속성/메서드 | 설명 | 예시 |
|-------------|------|------|
| `req.params` | 라우트 파라미터 | `{ id: '42' }` |
| `req.query` | 쿼리 스트링 | `{ page: '2' }` |
| `req.body` | 요청 본문 (파싱 필요) | `{ name: '홍길동' }` |
| `req.headers` | 요청 헤더 | `{ 'content-type': '...' }` |
| `req.method` | HTTP 메서드 | `'GET'` |
| `req.url` | 요청 URL | `'/users?page=2'` |
| `req.path` | 경로 부분만 | `'/users'` |
| `req.ip` | 클라이언트 IP | `'127.0.0.1'` |
| `req.get(header)` | 특정 헤더 값 | `req.get('Content-Type')` |

---

## 응답(Response) 객체

`res` 객체는 `http.ServerResponse`를 확장하며, 편리한 응답 메서드를 제공한다.

```javascript
// 문자열/HTML 응답
res.send('Hello');
res.send('<h1>Hello</h1>');

// JSON 응답
res.json({ name: '홍길동', age: 25 });

// 상태 코드 설정 후 응답
res.status(201).json({ id: 1, name: '홍길동' });
res.status(404).send('Not Found');

// 리다이렉트
res.redirect('/login');
res.redirect(301, '/new-url');

// 파일 다운로드
res.download('./files/report.pdf');

// 파일 전송
res.sendFile(path.join(__dirname, 'public', 'index.html'));
```

| 메서드 | 설명 |
|--------|------|
| `res.send()` | 문자열, Buffer, 객체 등 다양한 타입 응답 |
| `res.json()` | JSON 응답 (Content-Type 자동 설정) |
| `res.status()` | HTTP 상태 코드 설정 (체이닝 가능) |
| `res.redirect()` | 리다이렉트 응답 |
| `res.render()` | 템플릿 엔진으로 렌더링 |
| `res.download()` | 파일 다운로드 응답 |
| `res.sendFile()` | 파일을 응답으로 전송 |

---

## 정적 파일 제공

`express.static()` 미들웨어를 사용하면 별도 라우팅 없이 정적 파일을 제공할 수 있다.

```javascript
const express = require('express');
const path = require('path');
const app = express();

// public 폴더의 파일을 정적으로 제공
app.use(express.static(path.join(__dirname, 'public')));

// 가상 경로 접두사 지정
app.use('/static', express.static(path.join(__dirname, 'public')));

app.listen(3000);
```

```
프로젝트 구조:
my-express-app/
├── app.js
└── public/
    ├── index.html
    ├── css/
    │   └── style.css
    ├── js/
    │   └── main.js
    └── images/
        └── logo.png
```

```
접근 URL (가상 경로 없이):
  http://localhost:3000/index.html
  http://localhost:3000/css/style.css
  http://localhost:3000/images/logo.png

접근 URL (가상 경로 /static):
  http://localhost:3000/static/index.html
  http://localhost:3000/static/css/style.css
```

> `http` 모듈에서는 MIME 타입을 직접 관리하고 `fs.readFile`로 파일을 읽어야 했지만, Express에서는 한 줄로 해결된다.

---

## 에러 처리

### 기본 에러 처리

```javascript
app.get('/error', (req, res) => {
  throw new Error('의도적인 에러 발생!');
});
```

Express는 동기 코드에서 발생한 에러를 자동으로 잡아 500 응답을 보낸다.

### 비동기 에러 처리

```javascript
// async/await에서의 에러 처리
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await findUserById(req.params.id);
    if (!user) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
    res.json(user);
  } catch (err) {
    next(err); // 에러 미들웨어로 전달
  }
});
```

### 에러 처리 미들웨어

에러 미들웨어는 **4개의 매개변수** `(err, req, res, next)`를 가진다.

```javascript
// 404 처리 — 모든 라우트 뒤에 배치
app.use((req, res, next) => {
  res.status(404).json({ error: '요청한 리소스를 찾을 수 없습니다.' });
});

// 에러 처리 미들웨어 — 가장 마지막에 배치
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({
    error: err.message || '서버 내부 오류가 발생했습니다.',
  });
});
```

```
미들웨어 배치 순서:
  1. 내장/서드파티 미들웨어 (json, cors, morgan 등)
  2. 라우트 핸들러 (app.get, app.post 등)
  3. 404 처리 미들웨어
  4. 에러 처리 미들웨어
```

---

## Router — 라우트 모듈 분리

프로젝트가 커지면 라우트를 별도 파일로 분리해야 한다. `express.Router()`를 사용한다.

### 프로젝트 구조

```
my-express-app/
├── app.js
├── routes/
│   ├── users.js
│   └── posts.js
└── package.json
```

### 라우터 파일 작성

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();

let users = [
  { id: 1, name: '홍길동', email: 'hong@example.com' },
  { id: 2, name: '김철수', email: 'kim@example.com' },
];
let nextId = 3;

router.get('/', (req, res) => {
  res.json(users);
});

router.get('/:id', (req, res) => {
  const user = users.find((u) => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'Not Found' });
  res.json(user);
});

router.post('/', (req, res) => {
  const { name, email } = req.body;
  const user = { id: nextId++, name, email };
  users.push(user);
  res.status(201).json(user);
});

router.put('/:id', (req, res) => {
  const user = users.find((u) => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'Not Found' });
  Object.assign(user, req.body);
  res.json(user);
});

router.delete('/:id', (req, res) => {
  const index = users.findIndex((u) => u.id === parseInt(req.params.id));
  if (index === -1) return res.status(404).json({ error: 'Not Found' });
  const deleted = users.splice(index, 1)[0];
  res.json(deleted);
});

module.exports = router;
```

### 메인 앱에 연결

```javascript
// app.js
const express = require('express');
const app = express();

const usersRouter = require('./routes/users');
const postsRouter = require('./routes/posts');

app.use(express.json());

app.use('/api/users', usersRouter);
app.use('/api/posts', postsRouter);

app.listen(3000, () => {
  console.log('서버 실행 중: http://localhost:3000');
});
```

```
라우터 매핑:
  app.use('/api/users', usersRouter)
  ─────────────────────────────────
  router.get('/')        → GET    /api/users
  router.get('/:id')     → GET    /api/users/:id
  router.post('/')       → POST   /api/users
  router.put('/:id')     → PUT    /api/users/:id
  router.delete('/:id')  → DELETE /api/users/:id
```

> Router는 **미니 앱(mini-app)**이라고도 불린다. 독립적인 미들웨어와 라우트를 가질 수 있으며, `app.use()`로 특정 경로에 마운트된다.

---

## 완성 예제: Express CRUD API 서버

`http` 모듈로 만든 TODO API를 Express로 다시 작성해 보자.

```javascript
const express = require('express');
const morgan = require('morgan');
const app = express();

// 미들웨어
app.use(morgan('dev'));
app.use(express.json());

// 데이터
let todos = [
  { id: 1, text: 'Express 배우기', done: false },
  { id: 2, text: '미들웨어 이해하기', done: false },
];
let nextId = 3;

// 전체 조회
app.get('/api/todos', (req, res) => {
  res.json(todos);
});

// 단건 조회
app.get('/api/todos/:id', (req, res) => {
  const todo = todos.find((t) => t.id === parseInt(req.params.id));
  if (!todo) return res.status(404).json({ error: 'Not Found' });
  res.json(todo);
});

// 생성
app.post('/api/todos', (req, res) => {
  const { text } = req.body;
  if (!text) return res.status(400).json({ error: 'text 필드는 필수입니다.' });

  const todo = { id: nextId++, text, done: false };
  todos.push(todo);
  res.status(201).json(todo);
});

// 수정
app.put('/api/todos/:id', (req, res) => {
  const todo = todos.find((t) => t.id === parseInt(req.params.id));
  if (!todo) return res.status(404).json({ error: 'Not Found' });

  const { text, done } = req.body;
  if (text !== undefined) todo.text = text;
  if (done !== undefined) todo.done = done;
  res.json(todo);
});

// 삭제
app.delete('/api/todos/:id', (req, res) => {
  const index = todos.findIndex((t) => t.id === parseInt(req.params.id));
  if (index === -1) return res.status(404).json({ error: 'Not Found' });

  const deleted = todos.splice(index, 1)[0];
  res.json(deleted);
});

// 404 처리
app.use((req, res) => {
  res.status(404).json({ error: '요청한 리소스를 찾을 수 없습니다.' });
});

// 에러 처리
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: '서버 내부 오류가 발생했습니다.' });
});

app.listen(3000, () => {
  console.log('Express TODO API 서버 실행 중: http://localhost:3000');
});
```

```bash
# 전체 조회
$ curl http://localhost:3000/api/todos

# 단건 조회
$ curl http://localhost:3000/api/todos/1

# 추가
$ curl -X POST http://localhost:3000/api/todos \
  -H "Content-Type: application/json" \
  -d '{"text":"라우터 분리하기"}'

# 수정
$ curl -X PUT http://localhost:3000/api/todos/1 \
  -H "Content-Type: application/json" \
  -d '{"done":true}'

# 삭제
$ curl -X DELETE http://localhost:3000/api/todos/2
```

---

## http 모듈 코드 vs Express 코드 비교

| 항목 | http 모듈 | Express |
|------|-----------|---------|
| 라우팅 | `if (req.method === 'GET' && req.url === '/') {}` | `app.get('/', handler)` |
| Body 파싱 | `req.on('data')` / `req.on('end')` 스트림 수집 | `express.json()` 한 줄 |
| JSON 응답 | `res.writeHead(200, {...}); res.end(JSON.stringify(data))` | `res.json(data)` |
| 상태 코드 | `res.writeHead(404)` | `res.status(404)` |
| 정적 파일 | `fs.readFile` + MIME 타입 관리 | `express.static('public')` |
| 에러 처리 | `try/catch` 직접 구현 | 에러 미들웨어 자동 처리 |
| 코드 분리 | 단일 파일에서 모든 분기 | `Router`로 모듈 분리 |

> Express를 사용하면 같은 기능을 **절반 이하의 코드**로 구현할 수 있으며, 미들웨어를 통해 기능을 쉽게 확장할 수 있다.

