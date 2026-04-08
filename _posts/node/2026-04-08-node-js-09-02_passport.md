---
title: 09-02. (example) passport 추가해보기
date: 2026-04-08 00:00:00 +0900
categories: [Node.JS, example]
tags: [Tech, Node.JS, passport]
pin: true
---

## 우선, passport의 이해

Passport는 Node.js(Express)에서 “로그인/인증”을 쉽게 붙이기 위한 미들웨어 프레임워크예요.
핵심은 Passport 자체가 “로그인 로직을 한 번에 다 해주는 제품”이 아니라, 여러 인증 방식(전략, Strategy)을 끼워 넣을 수 있는 공통 뼈대라는 점입니다.

* Passport(본체): 인증 흐름을 Express 미들웨어로 연결해주는 프레임워크
* Strategy(전략): 실제 로그인 방식 구현체
    * 예: passport-local(아이디/비번), passport-kakao(카카오 OAuth), passport-google-oauth20 등

* Passport가 해결해주는 2가지 큰 문제
1. “이 요청은 로그인한 사용자냐?”를 판단하는 공통 방식 제공
로그인 성공 시 “사용자”를 Passport가 req.user로 달아주고, 이후 미들웨어/라우트에서 그걸 기준으로 접근제어를 합니다.
2. 로그인 상태 유지(세션 연동) 표준화
로그인 성공 후 **세션에 최소 정보(보통 user id)**만 저장하고, 다음 요청 때 그 id로 사용자를 다시 읽어와 req.user로 복구합니다.
(이게 serializeUser / deserializeUser 역할)

### 잘 이해가 안될테니 예시로

아래 코드는 **“아이디/비번 로그인 → 세션에 user.id 저장 → 다음 요청에서 req.user로 복원”**을 최소 구성으로 보여줘요. (DB 없이 메모리 사용자 1명)

```js
const express = require("express");
const session = require("express-session");
const passport = require("passport");
const LocalStrategy = require("passport-local").Strategy;

const app = express();

// 예제용 "DB" (메모리)
const USERS = [
  { id: 1, username: "neo", password: "1234", nickname: "네오" },
];

// (1) 전략(Strategy): 실제로 아이디/비번 검증하는 부분
passport.use(
  new LocalStrategy(
    { usernameField: "username", passwordField: "password" },
    async (username, password, done) => {
      try {
        const user = USERS.find((u) => u.username === username);
        if (!user) return done(null, false, { message: "없는 사용자" });
        if (user.password !== password)
          return done(null, false, { message: "비밀번호 틀림" });
        return done(null, user); // 성공: user 객체를 넘김
      } catch (err) {
        return done(err);
      }
    }
  )
);

// (2) serializeUser: 로그인 성공 직후 세션에 "무엇"을 저장할지
passport.serializeUser((user, done) => {
  done(null, user.id); // 세션에는 id만
});

// (3) deserializeUser: 매 요청마다 세션 id로 유저를 복원해서 req.user에 주입
passport.deserializeUser((id, done) => {
  const user = USERS.find((u) => u.id === id);
  done(null, user || false);
});

app.use(express.json());
app.use(express.urlencoded({ extended: false }));

// 세션이 passport.session()보다 먼저 와야 함
app.use(
  session({
    secret: "secret-key",
    resave: false,
    saveUninitialized: false,
  })
);

app.use(passport.initialize());
app.use(passport.session());

// 로그인 안 했으면 막는 미들웨어
function isLoggedIn(req, res, next) {
  if (req.isAuthenticated && req.isAuthenticated()) return next();
  return res.status(401).json({ message: "로그인 필요" });
}

// 로그인 요청
app.post("/login", (req, res, next) => {
  passport.authenticate("local", (authErr, user, info) => {
    if (authErr) return next(authErr);
    if (!user) return res.status(401).json({ message: info?.message || "실패" });

    // req.login()이 호출되면 serializeUser가 실행되며 세션에 저장됨
    req.login(user, (loginErr) => {
      if (loginErr) return next(loginErr);
      return res.json({ message: "로그인 성공", user: req.user });
    });
  })(req, res, next);
});

// 로그아웃
app.post("/logout", (req, res) => {
  req.logout(() => {
    req.session.destroy(() => {
      res.json({ message: "로그아웃 완료" });
    });
  });
});

// 로그인 상태 확인(세션 유지 확인)
app.get("/me", isLoggedIn, (req, res) => {
  res.json({ user: req.user }); // deserializeUser 덕분에 req.user가 있음
});

app.listen(3000, () => console.log("http://localhost:3000"));
```


