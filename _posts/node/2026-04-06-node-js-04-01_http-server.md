---
title: 04-01. HTTP Server Example
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, HTTP-SERVER]
tags: [Tech, Node.JS, HTTP-SERVER]
pin: true
---

> Node.JS는 `http` 내장 모듈을 통해 별도 설치 없이 HTTP 서버를 만들 수 있다.

## 기본 HTTP 서버 만들기

### 최소 코드

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('안녕하세요, Node.JS HTTP 서버입니다!');
});

server.listen(3000, () => {
  console.log('서버가 http://localhost:3000 에서 실행 중입니다.');
});
```

```bash
$ node server.js
서버가 http://localhost:3000 에서 실행 중입니다.
```

브라우저에서 `http://localhost:3000`에 접속하면 **"안녕하세요, Node.JS HTTP 서버입니다!"**가 표시된다.

### 코드 분석

```
http.createServer(callback)
       │
       │  callback = (req, res) => { ... }
       │
       ├─ req (IncomingMessage) ─ 클라이언트의 요청 정보
       │     ├─ req.method   → GET, POST 등
       │     ├─ req.url      → 요청 경로
       │     └─ req.headers  → 요청 헤더
       │
       └─ res (ServerResponse) ─ 서버의 응답 객체
             ├─ res.writeHead(statusCode, headers)
             ├─ res.write(data)  → 응답 본문 작성
             └─ res.end(data)    → 응답 종료
```

> `res.end()`를 호출하지 않으면 클라이언트는 응답을 계속 기다리게 되므로, 반드시 호출해야 한다.

---

## HTML 응답 보내기

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
  res.end(`
    <!DOCTYPE html>
    <html>
    <head><title>Node.JS 서버</title></head>
    <body>
      <h1>Hello, Node.JS!</h1>
      <p>HTTP 서버에서 보내는 HTML 페이지입니다.</p>
    </body>
    </html>
  `);
});

server.listen(3000, () => {
  console.log('서버 실행 중: http://localhost:3000');
});
```

> `Content-Type`을 `text/html`로 설정하면 브라우저가 HTML로 렌더링한다. `text/plain`이면 태그가 그대로 문자열로 표시된다.

---

## 라우팅 (URL별 분기 처리)

`req.url`을 이용하면 요청 경로에 따라 다른 응답을 보낼 수 있다.

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });

  if (req.url === '/') {
    res.end('<h1>홈 페이지</h1><a href="/about">소개</a>');
  } else if (req.url === '/about') {
    res.end('<h1>소개 페이지</h1><a href="/">홈으로</a>');
  } else {
    res.writeHead(404);
    res.end('<h1>404 Not Found</h1>');
  }
});

server.listen(3000, () => {
  console.log('서버 실행 중: http://localhost:3000');
});
```

```
요청 URL        응답
─────────────────────────────
/            → 홈 페이지
/about       → 소개 페이지
/other       → 404 Not Found
```

---

## JSON 응답 (REST API 기초)

API 서버를 만들 때는 JSON 형식으로 응답한다.

```javascript
const http = require('http');

const users = [
  { id: 1, name: '홍길동', age: 25 },
  { id: 2, name: '김철수', age: 30 },
  { id: 3, name: '이영희', age: 28 },
];

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/api/users') {
    res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' });
    res.end(JSON.stringify(users));
  } else {
    res.writeHead(404, { 'Content-Type': 'application/json; charset=utf-8' });
    res.end(JSON.stringify({ error: 'Not Found' }));
  }
});

server.listen(3000, () => {
  console.log('API 서버 실행 중: http://localhost:3000');
});
```

```bash
$ curl http://localhost:3000/api/users
[{"id":1,"name":"홍길동","age":25},{"id":2,"name":"김철수","age":30},{"id":3,"name":"이영희","age":28}]
```

---

## POST 요청 처리

POST 요청의 본문(body) 데이터는 **스트림**으로 전달되므로, 이벤트를 통해 수집해야 한다.

```javascript
const http = require('http');

const users = [];

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/api/users') {
    res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' });
    res.end(JSON.stringify(users));

  } else if (req.method === 'POST' && req.url === '/api/users') {
    let body = '';

    req.on('data', (chunk) => {
      body += chunk;
    });

    req.on('end', () => {
      const newUser = JSON.parse(body);
      newUser.id = users.length + 1;
      users.push(newUser);

      res.writeHead(201, { 'Content-Type': 'application/json; charset=utf-8' });
      res.end(JSON.stringify(newUser));
    });

  } else {
    res.writeHead(404, { 'Content-Type': 'application/json; charset=utf-8' });
    res.end(JSON.stringify({ error: 'Not Found' }));
  }
});

server.listen(3000, () => {
  console.log('API 서버 실행 중: http://localhost:3000');
});
```

```bash
# 사용자 추가
$ curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"박지민","age":22}'

{"name":"박지민","age":22,"id":1}

# 목록 조회
$ curl http://localhost:3000/api/users
[{"name":"박지민","age":22,"id":1}]
```

### 요청 본문 처리 흐름

```
POST /api/users
Body: {"name":"박지민","age":22}

  req.on('data') ─→ chunk 수집 ─→ body 문자열 누적
  req.on('end')  ─→ JSON.parse(body) ─→ 처리 후 응답
```

> 대용량 데이터를 받을 때는 body 크기 제한을 두어야 한다. 그렇지 않으면 메모리 부족 공격(DoS)에 취약해진다.

---

## 정적 파일 제공하기

`fs` 모듈과 함께 사용하면 HTML, CSS, JS 파일 등을 제공하는 정적 파일 서버를 만들 수 있다.

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');

const MIME_TYPES = {
  '.html': 'text/html',
  '.css': 'text/css',
  '.js': 'application/javascript',
  '.json': 'application/json',
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.gif': 'image/gif',
};

const server = http.createServer((req, res) => {
  let filePath = path.join(__dirname, 'public', req.url === '/' ? 'index.html' : req.url);
  const ext = path.extname(filePath);
  const contentType = MIME_TYPES[ext] || 'application/octet-stream';

  fs.readFile(filePath, (err, data) => {
    if (err) {
      res.writeHead(404, { 'Content-Type': 'text/html; charset=utf-8' });
      res.end('<h1>404 Not Found</h1>');
      return;
    }

    res.writeHead(200, { 'Content-Type': `${contentType}; charset=utf-8` });
    res.end(data);
  });
});

server.listen(3000, () => {
  console.log('정적 파일 서버 실행 중: http://localhost:3000');
});
```

```
프로젝트 구조:

project/
├── server.js
└── public/
    ├── index.html
    ├── style.css
    └── script.js
```

---

## 주요 HTTP 상태 코드

| 코드 | 의미 | 설명 |
|------|------|------|
| **200** | OK | 요청 성공 |
| **201** | Created | 리소스 생성 성공 |
| **301** | Moved Permanently | 영구 리다이렉트 |
| **304** | Not Modified | 캐시된 버전 사용 |
| **400** | Bad Request | 잘못된 요청 |
| **404** | Not Found | 리소스를 찾을 수 없음 |
| **500** | Internal Server Error | 서버 내부 오류 |

---

## 완성 예제: 간단한 CRUD API 서버

GET, POST, PUT, DELETE를 모두 지원하는 간단한 API 서버를 만들어 보자.

```javascript
const http = require('http');

let todos = [
  { id: 1, text: 'Node.JS 공부하기', done: false },
  { id: 2, text: 'HTTP 서버 만들기', done: true },
];
let nextId = 3;

function sendJSON(res, statusCode, data) {
  res.writeHead(statusCode, { 'Content-Type': 'application/json; charset=utf-8' });
  res.end(JSON.stringify(data));
}

function parseBody(req) {
  return new Promise((resolve, reject) => {
    let body = '';
    req.on('data', (chunk) => { body += chunk; });
    req.on('end', () => {
      try {
        resolve(JSON.parse(body));
      } catch (e) {
        reject(e);
      }
    });
  });
}

const server = http.createServer(async (req, res) => {
  const { method, url } = req;

  // GET /api/todos — 전체 목록 조회
  if (method === 'GET' && url === '/api/todos') {
    return sendJSON(res, 200, todos);
  }

  // POST /api/todos — 새 항목 추가
  if (method === 'POST' && url === '/api/todos') {
    const { text } = await parseBody(req);
    const todo = { id: nextId++, text, done: false };
    todos.push(todo);
    return sendJSON(res, 201, todo);
  }

  // PUT /api/todos/:id — 항목 수정
  const putMatch = method === 'PUT' && url.match(/^\/api\/todos\/(\d+)$/);
  if (putMatch) {
    const id = parseInt(putMatch[1]);
    const todo = todos.find((t) => t.id === id);
    if (!todo) return sendJSON(res, 404, { error: 'Not Found' });

    const updates = await parseBody(req);
    Object.assign(todo, updates);
    return sendJSON(res, 200, todo);
  }

  // DELETE /api/todos/:id — 항목 삭제
  const deleteMatch = method === 'DELETE' && url.match(/^\/api\/todos\/(\d+)$/);
  if (deleteMatch) {
    const id = parseInt(deleteMatch[1]);
    const index = todos.findIndex((t) => t.id === id);
    if (index === -1) return sendJSON(res, 404, { error: 'Not Found' });

    const deleted = todos.splice(index, 1)[0];
    return sendJSON(res, 200, deleted);
  }

  sendJSON(res, 404, { error: 'Not Found' });
});

server.listen(3000, () => {
  console.log('TODO API 서버 실행 중: http://localhost:3000');
});
```

```bash
# 전체 조회
$ curl http://localhost:3000/api/todos

# 추가
$ curl -X POST http://localhost:3000/api/todos \
  -H "Content-Type: application/json" \
  -d '{"text":"Express 배우기"}'

# 수정 (id=1의 done을 true로)
$ curl -X PUT http://localhost:3000/api/todos/1 \
  -H "Content-Type: application/json" \
  -d '{"done":true}'

# 삭제
$ curl -X DELETE http://localhost:3000/api/todos/2
```

---

## http 모듈의 한계와 Express

`http` 모듈만으로도 서버를 만들 수 있지만, 프로젝트가 커지면 다음과 같은 불편함이 생긴다.

| 한계 | 설명 |
|------|------|
| 라우팅 | `if/else`로 URL을 분기하면 코드가 복잡해짐 |
| 미들웨어 | 로깅, 인증 등 공통 처리를 직접 구현해야 함 |
| 요청 파싱 | body, query string, cookie 등을 수동으로 파싱 |
| 정적 파일 | 직접 MIME 타입을 관리해야 함 |

이러한 문제를 해결하기 위해 **Express**, **Koa**, **Fastify** 같은 웹 프레임워크를 사용한다.

```javascript
// Express를 사용하면 같은 기능이 훨씬 간결해진다
const express = require('express');
const app = express();

app.use(express.json());

app.get('/api/todos', (req, res) => {
  res.json(todos);
});

app.post('/api/todos', (req, res) => {
  const todo = { id: nextId++, text: req.body.text, done: false };
  todos.push(todo);
  res.status(201).json(todo);
});

app.listen(3000);
```

> 다음 포스트에서 Express를 활용한 서버 구축을 다룰 예정이다.
