---
title: 09-01. (example) sns 만들기기
date: 2026-04-08 00:00:00 +0900
categories: [Node.JS, example]
tags: [Tech, Node.JS]
pin: true
---

## app.js

### morgan

morgan은 Express용 HTTP 요청 로깅 미들웨어입니다.
브라우저나 클라이언트가 서버로 보낸 요청이 들어올 때마다, 메서드·URL·상태 코드·응답 시간 같은 정보를 터미널(콘솔)에 한 줄씩 찍어 줍니다.

```js
const morgan = require('morgan');
const app = express();
app.use(morgan('dev'));
```

```
GET / 200 60.867 ms - 3240
GET /main.css 200 7.625 ms - 2717
GET /favicon.ico 404 6.483 ms - 2759
GET /auth/kakao 404 3.166 ms - 2757
GET /main.css 304 2.975 ms - -
GET /join 200 5.919 ms - 2463
GET /main.css 304 1.159 ms - -
POST /auth/join 404 50.798 ms - 2757
GET /main.css 304 1.704 ms - -
```

### nunjucks

Nunjucks는 JavaScript(주로 Node·브라우저)에서 쓰는 HTML 템플릿 엔진입니다. Python의 Jinja2와 문법이 비슷해서, 변수 넣기·반복·조건·레이아웃 상속 같은 걸 HTML 안에서 처리합니다.

```js
app.set('view engine', 'html');
nunjucks.configure('views', {
  express: app,
  watch: true,
});
```

* Express의 뷰 엔진을 'html'로 두고, 실제 렌더링은 Nunjucks가 맡습니다.
* views 폴더 안의 .html 파일이 템플릿입니다.
* 컨트롤러/라우터에서 res.render('파일이름') 하면 그 템플릿이 HTML로 만들어져 응답됩니다.
* watch: true: 개발 중 템플릿 파일이 바뀌면 다시 읽어 오도록(핫 리로드에 가깝게) 동작합니다.

### 본론으로

```js
// app.js

const express = require('express');
const cookieParser = require('cookie-parser');
const morgan = require('morgan');               // Express용 HTTP 요청 로깅 미들웨어
const path = require('path');
const session = require('express-session');
const nunjucks = require('nunjucks');           // JavaScript(주로 Node·브라우저)에서 쓰는 HTML 템플릿 엔진
const dotenv = require('dotenv');

dotenv.config();
const pageRouter = require('./routes/page');

// ...
```

```js
// page.js

const express = require('express');
// controllers/page.js 에서 함수를 임포트트
const { renderProfile, renderJoin, renderMain } = require('../controllers/page');

const router = express.Router();

// res.locals는 뷰(템플릿)에 넘길 변수를 담는 객체
// 몇 가지를 초기화 한다다
router.use((req, res, next) => {
  res.locals.user = null;
  res.locals.followerCount = 0;
  res.locals.followingCount = 0;
  res.locals.followingIdList = [];
  next();
});

// 라우팅 설정정
router.get('/profile', renderProfile);

router.get('/join', renderJoin);

router.get('/', renderMain);

module.exports = router;
```

#### `module.exports = router;` 해석

`module.exports = router;`는 **“이 파일을 require로 불러오면, 그 결과로 router를 돌려준다”**는 뜻입니다.

```js
// 실제로 app.js에서 
const pageRouter = require('./routes/page'); // 이렇게 쓴다다
```

Node.js(CommonJS)에서 파일은 “모듈”이다
한 .js 파일 안에서 만든 변수·함수는 기본적으로 밖에서 못 씁니다.
밖으로내려면 module.exports에 넣어서 보내는 것이 관례입니다.
module.exports = router가 하는 일
이 줄은 router 객체 전체를 이 파일의 “대표 export”로 정한다는 의미입니다.
그래서 다른 파일에서는 보통 이렇게 받습니다:
const pageRouter = require('./routes/page');
app.use('/', pageRouter);
여기서 require('./routes/page')의 값이 곧 그 파일의 module.exports이고, 위 예에서는 router와 같은 객체입니다.

왜 router를 export하나
express.Router()로 만든 건 경로·미들웨어 묶음입니다.
app.js 같은 곳에서 app.use(경로, router)로 붙이려면, 그 router를 다른 파일로 넘겨야 하니까 module.exports에 담는 겁니다.
정리하면, **module.exports = X는 “이 파일을 import/require 할 때 받게 될 값이 X다”**이고, 여기서는 그 X가 **router**인 것입니다.

### 다시 app.js

```js
// ...

const app = express();
app.set('port', process.env.PORT || 8001);
app.set('view engine', 'html');
nunjucks.configure('views', {
  express: app,         // nunjuck의 express등록
  watch: true,          // watch: true 자동업데이트 확인인
});

// 여기는 설명이 좀 필요해 아래에 설명 추가.
app.use(morgan('dev'));
app.use(express.static(path.join(__dirname, 'public')));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser(process.env.COOKIE_SECRET));
app.use(session({
  resave: false,
  saveUninitialized: false,
  secret: process.env.COOKIE_SECRET,
  cookie: {
    httpOnly: true,
    secure: false,
  },
}));

// ...

```

#### `app.use(express.static(path.join(__dirname, 'public')));`

express.static(폴더경로): 그 폴더 안의 파일을 정적 자원으로 제공하는 미들웨어를 만든다. 브라우저가 GET으로 요청하면, 해당 경로에 맞는 파일을 읽어 응답한다 (HTML/CSS/JS, 이미지, 폰트 등).
app.use(...): 그 미들웨어를 앱에 등록한다. 보통 루트 경로 / 에 붙이므로, 요청 URL이 곧 public 아래의 상대 경로로 매핑된다.
path.join(__dirname, 'public'): 실행 중인 app.js가 있는 디렉터리(__dirname) 기준으로 public 폴더의 절대 경로를 만든다. Windows·macOS 모두에서 경로 구분자가 올바르게 이어지도록 한다.

동작 예시
프로젝트가 이렇다고 하면:

```
project/
  app.js
  public/
    css/
      style.css
    js/
      main.js
    images/
      logo.png
```

app.js에 위 한 줄이 있으면 대략 다음과 같이 대응된다.

| 브라우저 요청 URL | 실제 파일 |
|:---|:---|
| http://localhost:8001/css/style.css | public/css/style.css |
| http://localhost:8001/js/main.js | public/js/main.js |
| http://localhost:8001/images/logo.png | public/images/logo.png |

템플릿(예: Nunjucks)에서는 보통 이렇게만 쓴다.

```html
<link rel="stylesheet" href="/css/style.css">
<script src="/js/main.js"></script>
<img src="/images/logo.png" alt="logo">
```

앞의 /는 사이트 루트이고, Express는 public을 루트에 매핑했기 때문에 위 경로가 그대로 public 아래 파일을 가리킨다.

왜 __dirname을 쓰나
cd로 어디서 서버를 실행하든, app.js 파일 위치 기준으로 public을 찾기 위해서다. ./public만 쓰면 “현재 작업 디렉터리” 기준이 되어, 실행 위치에 따라 경로가 어긋날 수 있다.

글 안에서의 위치
같은 app.js 흐름에서 morgan 다음에 두어 로그·정적 파일 제공을 먼저 처리하고, 그다음 express.json() 등 바디 파서·세션·라우터를 붙이는 전형적인 순서와 맞는다. 정적 파일은 자주 요청되므로 가능한 한 앞쪽에 두는 경우가 많다.

---

### `app.use(express.json());`

express.json() 은 요청 본문(body)이 JSON일 때 그 내용을 파싱해서 req.body에 넣어 주는 미들웨어를 만든다.

클라이언트가 Content-Type: application/json 으로 보낸 데이터를 읽는다.
파싱에 성공하면 req.body 에 객체 형태로 담긴다.
이 한 줄이 없으면 POST/PUT 등으로 JSON을 보내도 req.body가 비어 있거나 undefined 에 가깝게 동작한다 (버전·설정에 따라 다르지만, 일반적으로 JSON 파서를 붙여야 한다).
app.use와 함께 쓰는 이유
app.use(express.json()); 는 그 미들웨어를 모든 라우트보다 앞에서 실행되게 등록한다는 뜻이다. 그래서 아래에 정의한 app.post, router.post 등에서 req.body를 바로 쓸 수 있다.

예시
클라이언트 (fetch)

```js
fetch('/api/posts', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title: '제목', content: '내용' }),
});
```

서버

```js
app.use(express.json());
app.post('/api/posts', (req, res) => {
  const { title, content } = req.body; // { title: '제목', content: '내용' }
  // ...
});
```

---

### `app.use(express.urlencoded({ extended: false }));`

express.urlencoded() 는 요청 본문이 application/x-www-form-urlencoded 형식일 때(HTML `<form method="POST">` 가 기본으로 보내는 형식) 그 내용을 읽어 req.body 에 객체로 넣어 주는 미들웨어를 만든다.

* `express.json()` → JSON 본문
* `express.urlencoded()` → 폼에서 온 “이름=값&이름=값…” 형태 본문
둘 다 최종적으로 req.body 를 채우지만, Content-Type 이 다르기 때문에 미들웨어도 따로 둔다.

* extended: false 의 의미
    * 쿼리스트링 파서(querystring)를 쓴다.
    * 단순한 키–값만 다룬다.
    * 배열·중첩 객체 같은 복잡한 구조는 기대하지 않는 설정에 적합하다.
* extended: true (생략 시 예전 기본값이었음)
    * qs 라이브러리를 쓴다.
    * a[b]=1 같은 중첩 표현을 파싱할 수 있다.
    * 그만큼 파싱 규칙이 무겁고, 과거에는 DoS 이슈 논의도 있었다(버전·설정에 따라 다름).
일반적인 로그인·간단 폼만 받는다면 extended: false 로 충분한 경우가 많다.

예시

```html
<form method="POST" action="/login">
  <input name="email" />
  <input name="password" type="password" />
  <button type="submit">로그인</button>
</form>
```

브라우저는 보통 Content-Type: application/x-www-form-urlencoded 로
email=a@b.com&password=secret 같은 본문을 보낸다.

서버

```js
app.use(express.urlencoded({ extended: false }));
app.post('/login', (req, res) => {
  const { email, password } = req.body;
  // ...
});
```

---

### `app.use(cookieParser(process.env.COOKIE_SECRET));`

서명 쿠키(Signed Cookie)란?

일반 쿠키는 이름과 값만 브라우저에 저장된다. 누군가 개발자 도구로 값을 바꿀 수 있고, 서버는 “이게 원래 내가 준 값인지”를 그 값만 보고는 알 수 없다.

서명 쿠키는 값에 비밀키로 만든 서명을 붙여 보낸다. 브라우저가 값을 바꾸면 서명이 맞지 않아서, 서버는 req.signedCookies에 넣지 않거나 검증에 실패한 것으로 처리할 수 있다.
즉, 무결성(변조 여부) 을 확인하는 용도에 가깝고, 암호화(내용 숨기기) 는 아니다. 값 자체가 Base64 등으로 보일 수 있어도 “비밀”이라고 보지 않는 것이 안전하다.

* Express에서의 흐름
    * cookieParser(secret) 에 secret 을 넘긴다 (예: process.env.COOKIE_SECRET).
    * 쿠키를 줄 때 { signed: true } 를 쓴다.
    * 읽을 때는 req.signedCookies 로 읽는다 (검증된 것만).


```js
const express = require('express');
const cookieParser = require('cookie-parser');

const app = express();
const SECRET = process.env.COOKIE_SECRET || '개발용만-실서비스에서는-환경변수';

app.use(cookieParser(SECRET));

// 서명 쿠키 설정
app.get('/set', (req, res) => {
  res.cookie('userId', '42', {
    signed: true,
    httpOnly: true,
    maxAge: 60 * 60 * 1000, // 1시간
  });
  res.send('쿠키 설정됨');
});

// 읽기: 검증 성공 시 req.signedCookies.userId 에만 값이 들어감
app.get('/profile', (req, res) => {
  const userId = req.signedCookies.userId;
  if (!userId) {
    return res.status(401).send('쿠키 없음 또는 서명 불일치(변조됨)');
  }
  res.send(`로그인된 사용자 id: ${userId}`);
});
```

* req.cookies.userId 에는 서명이 포함된 원시 문자열 형태가 올 수 있고,
* req.signedCookies.userId 에는 서명 검증을 통과한 경우에만 '42' 같은 순수 값이 들어온다 (cookie-parser 동작 기준).
변조 예: 사용자가 userId 를 다른 숫자로 바꾸면 서명이 어긋나서 req.signedCookies.userId 가 비어 있게 되는 식으로 동작한다.

| 구분 | 설정 | 읽기 | 특징 |
|:---|:---|:---|:---|
| 일반 | res.cookie('a', 'b') | req.cookies.a | 값 바꾸면 그대로 서버에 전달됨 |
| 서명 | res.cookie('a', 'b', { signed: true }) | req.signedCookies.a | 변조 시 검증 실패 → 보통 undefined |

---

### `app.use(session`

```js
app.use(session({
  resave: false,
  saveUninitialized: false,
  secret: process.env.COOKIE_SECRET,
  cookie: {
    httpOnly: true,
    secure: false,
  },
}));
```

이 코드는 express-session 미들웨어를 등록하는 부분이다. 브라우저에는 보통 세션 ID만 쿠키로 보내고, 로그인 상태·사용자 정보 같은 실제 데이터는 서버 쪽 세션 저장소에 둔다.

| 옵션 | 값 | 의미 |
|:---|:---|:---|
| resave | false | 요청이 올 때마다 세션을 강제로 다시 저장하지 않는다. 세션이 수정되지 않았는데 매번 저장하면 불필요한 DB/스토어 쓰기가 생길 수 있어, 보통 false 를 권장한다. |
| saveUninitialized | false | 한 번도 내용이 채워지지 않은(빈) 세션은 저장소에 저장하지 않는다. 방문만 하고 아무 것도 안 한 사용자에게 세션 쿠키를 만들지 않게 하여 쿠키 남발·저장소 낭비를 줄인다. 로그인할 때만 req.session 에 값을 넣는 패턴과 잘 맞는다. |
| secret | process.env.COOKIE_SECRET | 세션 쿠키(서명)에 쓰는 비밀키. cookie-parser 에 넘긴 값과 같게 두는 예가 많다(한 env로 통일). 유출되면 세션 위조 위험이 커지므로 환경 변수로만 관리한다. |
| cookie.httpOnly | true | 해당 세션 쿠키를 JavaScript(document.cookie)에서 읽지 못하게 한다. XSS로 스크립트가 쿠키를 훔치는 것을 어렵게 한다. |
| cookie.secure | false | https 일 때만 브라우저가 쿠키를 보내도록 할지 여부. false 이면 HTTP에서도 쿠키가 전송된다. 로컬 개발(HTTP)에서는 이렇게 두는 경우가 많고, 실제 배포(HTTPS) 에서는 secure: true 로 바꾸는 것이 일반적이다. |

---

## 다시 본론으로

```js
// app.js

app.use('/', pageRouter);

app.use((req, res, next) => {
  const error =  new Error(`${req.method} ${req.url} 라우터가 없습니다.`);
  error.status = 404;
  next(error);
});

app.use((err, req, res, next) => {
  res.locals.message = err.message;
  res.locals.error = process.env.NODE_ENV !== 'production' ? err : {};
  res.status(err.status || 500);
  res.render('error');
});

app.listen(app.get('port'), () => {
  console.log(app.get('port'), '번 포트에서 대기중');
});
```