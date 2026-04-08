---
title: 10-01. jwt and cors
date: 2026-04-08 00:00:00 +0900
categories: [Node.JS, jwt-cors]
tags: [Tech, Node.JS, jwt, cors]
pin: true
---

## JWT

JWT(JSON Web Token) 발급/검증 라이브러리입니다.

```
[User/Client]                                   [Server]
     |                                             |
     | 1) POST /login (id, password)              |
     |-------------------------------------------->|
     |                                             |  계정 검증
     |                                             |
     | 2) 200 OK + access_token                    |
     |<--------------------------------------------|
     |                                             |
     | 3) GET /profile                             |
     |    Authorization: Bearer <access_token>     |
     |-------------------------------------------->|
     |                                             |  토큰 서명/만료 검증
     |                                             |
     | 4) 200 OK + user data                       |
     |<--------------------------------------------|
     |                                             |
     | ...시간 경과... (토큰 만료)                 |
     |                                             |
     | 5) GET /profile + Bearer <old_token>        |
     |-------------------------------------------->|
     |                                             |  만료 확인
     | 6) 401 Unauthorized                         |
     |<--------------------------------------------|
     |                                             |
     | 7) POST /token/refresh (refresh_token)      |
     |-------------------------------------------->|
     |                                             |  refresh_token 검증
     | 8) 200 OK + new access_token                |
     |<--------------------------------------------|
     |                                             |
     | 9) GET /profile + Bearer <new_token>        |
     |-------------------------------------------->|
     |                                             |
     |10) 200 OK + user data                       |
     |<--------------------------------------------|
```

## CORS

CORS 정책은 브라우저 보안 규칙이에요.

* 브라우저는 기본적으로 다른 출처(origin) 로 가는 요청 응답을 JS에서 못 읽게 막습니다.
* 출처(origin)는 프로토콜 + 도메인 + 포트 조합이에요.
* 예: http://localhost:3000 과 http://localhost:4000은 포트가 달라서 다른 출처.
* 서버가 응답 헤더로 “이 출처는 허용”이라고 명시하면 브라우저가 풀어줍니다.
    * 핵심 헤더가 Access-Control-Allow-Origin.

간단 예시:

```
프론트: http://localhost:3000
API: http://localhost:8002
프론트가 API 호출 시, API가 아래처럼 보내면 허용:
Access-Control-Allow-Origin: http://localhost:3000
```

---

## 예시코드드

```js
// app.js
const express = require('express');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const cookieParser = require('cookie-parser');

const app = express();
app.use(express.json());
app.use(cookieParser());

// 보통 .env로 분리
const ACCESS_SECRET = 'access-secret';
const REFRESH_SECRET = 'refresh-secret';

// 실제로는 Redis/DB 사용 권장
const refreshStore = new Map(); // key: userId, value: refreshToken

// 1) CORS: 운영/개발 도메인만 허용 + credentials
const allowlist = [
  'http://localhost:3000',
  'https://app.myservice.com',
];

app.use(
  cors({
    origin(origin, cb) {
      // 서버 간 호출, health check 등 origin 없는 케이스 허용
      if (!origin) return cb(null, true);
      if (allowlist.includes(origin)) return cb(null, true);
      return cb(new Error('CORS blocked'));
    },
    credentials: true, // refresh token 쿠키 전송 허용
  })
);

// (가짜) 사용자 조회
function findUser(email, password) {
  if (email === 'admin@test.com' && password === '1234') {
    return { id: 1, role: 'admin', email };
  }
  return null;
}

// 2) 로그인: access는 응답 바디, refresh는 httpOnly 쿠키
app.post('/auth/login', (req, res) => {
  const { email, password } = req.body;
  const user = findUser(email, password);

  if (!user) {
    return res.status(401).json({ message: '이메일 또는 비밀번호 오류' });
  }

  const accessToken = jwt.sign(
    { sub: user.id, role: user.role, email: user.email },
    ACCESS_SECRET,
    { expiresIn: '15m', issuer: 'myservice', audience: 'web' }
  );

  const refreshToken = jwt.sign(
    { sub: user.id, type: 'refresh' },
    REFRESH_SECRET,
    { expiresIn: '14d', issuer: 'myservice', audience: 'web' }
  );

  refreshStore.set(String(user.id), refreshToken);

  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: false, // 운영에서는 true(HTTPS)
    sameSite: 'lax', // 크로스 사이트면 'none' + secure:true 필요
    path: '/auth/refresh',
    maxAge: 14 * 24 * 60 * 60 * 1000,
  });

  return res.json({
    accessToken,
    user: { id: user.id, email: user.email, role: user.role },
  });
});

// 3) 보호 라우트 미들웨어: Bearer 토큰 검증
function requireAuth(req, res, next) {
  const auth = req.headers.authorization || '';
  const [scheme, token] = auth.split(' ');

  if (scheme !== 'Bearer' || !token) {
    return res.status(401).json({ message: '인증 헤더 누락/형식 오류' });
  }

  try {
    const decoded = jwt.verify(token, ACCESS_SECRET, {
      issuer: 'myservice',
      audience: 'web',
    });
    req.user = decoded; // { sub, role, email, iat, exp ...}
    return next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ code: 'ACCESS_EXPIRED', message: '액세스 토큰 만료' });
    }
    return res.status(401).json({ code: 'INVALID_TOKEN', message: '유효하지 않은 토큰' });
  }
}

// 4) 보호 API
app.get('/me', requireAuth, (req, res) => {
  res.json({
    id: req.user.sub,
    email: req.user.email,
    role: req.user.role,
  });
});

// 5) 토큰 재발급: 쿠키의 refresh 검증 후 access 재발급
app.post('/auth/refresh', (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) {
    return res.status(401).json({ message: '리프레시 토큰 없음' });
  }

  try {
    const decoded = jwt.verify(refreshToken, REFRESH_SECRET, {
      issuer: 'myservice',
      audience: 'web',
    });

    const saved = refreshStore.get(String(decoded.sub));
    if (!saved || saved !== refreshToken) {
      return res.status(401).json({ message: '리프레시 토큰 불일치' });
    }

    const newAccess = jwt.sign(
      { sub: decoded.sub, role: 'admin', email: 'admin@test.com' },
      ACCESS_SECRET,
      { expiresIn: '15m', issuer: 'myservice', audience: 'web' }
    );

    return res.json({ accessToken: newAccess });
  } catch {
    return res.status(401).json({ message: '리프레시 토큰 만료/위조' });
  }
});

// 6) 로그아웃: 서버 저장소 제거 + 쿠키 삭제
app.post('/auth/logout', requireAuth, (req, res) => {
  refreshStore.delete(String(req.user.sub));
  res.clearCookie('refreshToken', { path: '/auth/refresh' });
  res.status(204).end();
});

app.listen(4000, () => console.log('API listening on :4000'));
```
