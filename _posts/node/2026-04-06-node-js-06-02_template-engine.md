---
title: 06-02. template engine
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, express, template_engine]
tags: [Tech, Node.JS, express, template_engine]
pin: true
---

> **템플릿 엔진**은 HTML에 동적 데이터를 삽입하여 서버 사이드 렌더링(SSR)을 가능하게 하는 도구다. Express와 결합하면 데이터를 뷰(View)에 바인딩하여 완성된 HTML을 클라이언트에게 전달할 수 있다.

## 템플릿 엔진이란?

템플릿 엔진은 **정적 HTML**과 **동적 데이터**를 결합하여 최종 HTML을 생성하는 소프트웨어다.

```
서버 사이드 렌더링 흐름
──────────────────────────────────────────────
클라이언트 요청 → 라우트 핸들러 → 데이터 준비
                                      │
                               템플릿 + 데이터
                                      │
                               HTML 생성 (렌더링)
                                      │
                               완성된 HTML 응답
──────────────────────────────────────────────
```

### 왜 템플릿 엔진을 쓰는가?

```
순수 문자열 연결 방식:
  res.send('<h1>' + user.name + '</h1><p>' + user.email + '</p>');
  → 복잡하고, XSS 취약점 발생 가능, 유지보수 어려움

템플릿 엔진 방식:
  res.render('profile', { user });
  → HTML과 로직 분리, 자동 이스케이프, 레이아웃·반복·조건 지원
```

### 주요 템플릿 엔진 비교

| 엔진 | 문법 스타일 | 특징 |
|------|-------------|------|
| **Pug** | 들여쓰기 기반 (태그 축약) | 간결한 문법, 러닝 커브 있음 |
| **EJS** | `<%= %>` HTML 삽입 | HTML 그대로 사용, 진입 장벽 낮음 |
| **Nunjucks** | `{% raw %}{{ }}{% endraw %}` Jinja2 스타일 | 강력한 상속·매크로, Mozilla 개발 |
| **Handlebars** | `{% raw %}{{ }}{% endraw %}` 로직 최소 | 로직 없는 템플릿 지향 |

---

## Express에서 템플릿 엔진 설정

Express는 `app.set()`을 통해 템플릿 엔진을 설정한다.

```javascript
const express = require('express');
const path = require('path');
const app = express();

// 뷰 엔진 설정
app.set('view engine', 'pug');  // 사용할 템플릿 엔진

// 뷰 파일이 위치한 디렉터리 (기본값: ./views)
app.set('views', path.join(__dirname, 'views'));

app.get('/', (req, res) => {
  res.render('index', { title: '홈페이지', message: '환영합니다!' });
});

app.listen(3000);
```

```
res.render() 동작 과정:
  1. views 폴더에서 템플릿 파일 탐색
  2. 두 번째 인자(객체)를 템플릿에 전달
  3. 템플릿 엔진이 HTML로 변환
  4. 완성된 HTML을 클라이언트에게 응답
```

```
프로젝트 구조:
my-express-app/
├── app.js
├── views/
│   ├── layout.pug
│   ├── index.pug
│   └── users/
│       ├── list.pug
│       └── detail.pug
├── public/
│   ├── css/
│   └── js/
└── package.json
```

---

## Pug (구 Jade)

Pug는 들여쓰기 기반의 간결한 문법을 사용하는 템플릿 엔진이다. 닫는 태그가 없고, 들여쓰기로 계층 구조를 표현한다.

### 설치 및 설정

```bash
$ npm i pug
```

```javascript
app.set('view engine', 'pug');
```

### 기본 문법

```pug
//- views/index.pug
doctype html
html(lang="ko")
  head
    meta(charset="UTF-8")
    title= title
    link(rel="stylesheet" href="/css/style.css")
  body
    h1= message
    p 이것은 Pug 템플릿입니다.
```

위 코드가 렌더링하면 아래 HTML이 생성된다.

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8">
    <title>홈페이지</title>
    <link rel="stylesheet" href="/css/style.css">
  </head>
  <body>
    <h1>환영합니다!</h1>
    <p>이것은 Pug 템플릿입니다.</p>
  </body>
</html>
```

### 변수 출력

```pug
//- 이스케이프 출력 (XSS 방지, 기본)
h1= title
p #{user.name}님, 환영합니다.

//- 이스케이프하지 않는 출력 (HTML 그대로 삽입)
div!= htmlContent

//- 인라인 보간
p 오늘 날씨는 #{weather}입니다.
```

```
= 변수         → 태그의 내용으로 변수 출력 (이스케이프)
#{변수}        → 텍스트 내 보간 (이스케이프)
!= 변수        → 이스케이프 없이 HTML 그대로 출력
!{변수}        → 텍스트 내 보간 (이스케이프 없음)
```

### 조건문

```pug
//- if / else if / else
if user.isLoggedIn
  p #{user.name}님, 환영합니다.
  a(href="/logout") 로그아웃
else
  p 로그인이 필요합니다.
  a(href="/login") 로그인

//- unless (if의 반대)
unless user.isAdmin
  p 관리자 전용 페이지입니다.
```

### 반복문

```pug
//- each 반복
ul
  each item in items
    li= item.name

//- 인덱스 사용
ul
  each user, index in users
    li #{index + 1}. #{user.name} (#{user.email})

//- 빈 배열 처리
ul
  each todo in todos
    li= todo.text
  else
    li 할 일이 없습니다.
```

### 레이아웃 상속

Pug의 가장 강력한 기능 중 하나로, 공통 레이아웃을 정의하고 자식 템플릿에서 확장(extends)할 수 있다.

```pug
//- views/layout.pug — 기본 레이아웃
doctype html
html(lang="ko")
  head
    meta(charset="UTF-8")
    meta(name="viewport" content="width=device-width, initial-scale=1.0")
    title #{title} | My App
    link(rel="stylesheet" href="/css/style.css")
    block styles
  body
    header
      nav
        a(href="/") 홈
        a(href="/users") 사용자
        if user
          a(href="/logout") 로그아웃
        else
          a(href="/login") 로그인
    main
      block content
    footer
      p &copy; 2026 My App
    block scripts
```

```pug
//- views/index.pug — 레이아웃 확장
extends layout

block content
  h1= message
  p 메인 페이지에 오신 것을 환영합니다.

block scripts
  script(src="/js/main.js")
```

```pug
//- views/users/list.pug — 사용자 목록 페이지
extends ../layout

block styles
  link(rel="stylesheet" href="/css/users.css")

block content
  h1 사용자 목록
  table
    thead
      tr
        th 이름
        th 이메일
        th 가입일
    tbody
      each user in users
        tr
          td
            a(href=`/users/${user.id}`)= user.name
          td= user.email
          td= user.createdAt
```

```
레이아웃 상속 구조:
  layout.pug (기본 틀)
    ├── block styles    ← 추가 CSS
    ├── block content   ← 본문 내용
    └── block scripts   ← 추가 JS

  index.pug (extends layout)
    └── block content   ← 본문만 작성

  users/list.pug (extends ../layout)
    ├── block styles    ← 사용자 전용 CSS
    └── block content   ← 사용자 목록 테이블
```

### 믹스인 (재사용 컴포넌트)

```pug
//- 믹스인 정의
mixin card(title, content)
  .card
    .card-header
      h3= title
    .card-body
      p= content

//- 믹스인 사용
+card('공지사항', '서버 점검이 예정되어 있습니다.')
+card('이벤트', '봄맞이 할인 이벤트를 진행합니다.')
```

### include (파일 삽입)

```pug
//- views/index.pug
extends layout

block content
  include partials/header
  h1 메인 페이지
  include partials/sidebar
```

---

## EJS (Embedded JavaScript)

EJS는 HTML 문법을 그대로 사용하면서 JavaScript 코드를 삽입할 수 있는 템플릿 엔진이다. HTML에 익숙한 개발자에게 진입 장벽이 낮다.

### 설치 및 설정

```bash
$ npm i ejs
```

```javascript
app.set('view engine', 'ejs');
```

### 기본 문법

```html
<!-- views/index.ejs -->
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title><%= title %></title>
</head>
<body>
  <h1><%= message %></h1>
  <p>이것은 EJS 템플릿입니다.</p>
</body>
</html>
```

### EJS 태그 종류

```
<%= 변수 %>     이스케이프 출력 (XSS 방지, 가장 많이 사용)
<%- 변수 %>     이스케이프 없는 출력 (HTML 그대로 삽입)
<% 코드 %>      실행만 하고 출력하지 않음 (제어문 등)
<%# 주석 %>     주석 (렌더링되지 않음)
<%- include('파일') %>   다른 EJS 파일 삽입
```

### 조건문

```html
<!-- views/index.ejs -->
<% if (user) { %>
  <p><%= user.name %>님, 환영합니다.</p>
  <a href="/logout">로그아웃</a>
<% } else { %>
  <p>로그인이 필요합니다.</p>
  <a href="/login">로그인</a>
<% } %>
```

### 반복문

```html
<ul>
  <% todos.forEach((todo, index) => { %>
    <li>
      <span><%= index + 1 %>.</span>
      <span><%= todo.text %></span>
      <% if (todo.done) { %>
        <span>✓ 완료</span>
      <% } %>
    </li>
  <% }) %>
</ul>

<!-- 빈 배열 처리 -->
<% if (todos.length === 0) { %>
  <p>할 일이 없습니다.</p>
<% } %>
```

### 레이아웃 (include 방식)

EJS는 Pug처럼 상속(extends) 기능이 없으므로, `include`로 공통 부분을 분리한다.

```html
<!-- views/partials/header.ejs -->
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= typeof title !== 'undefined' ? title : 'My App' %></title>
  <link rel="stylesheet" href="/css/style.css">
</head>
<body>
  <header>
    <nav>
      <a href="/">홈</a>
      <a href="/users">사용자</a>
    </nav>
  </header>
  <main>
```

```html
<!-- views/partials/footer.ejs -->
  </main>
  <footer>
    <p>&copy; 2026 My App</p>
  </footer>
</body>
</html>
```

```html
<!-- views/index.ejs -->
<%- include('partials/header') %>

<h1><%= message %></h1>
<p>메인 페이지에 오신 것을 환영합니다.</p>

<%- include('partials/footer') %>
```

```html
<!-- views/users/list.ejs -->
<%- include('../partials/header') %>

<h1>사용자 목록</h1>
<table>
  <thead>
    <tr>
      <th>이름</th>
      <th>이메일</th>
    </tr>
  </thead>
  <tbody>
    <% users.forEach(user => { %>
      <tr>
        <td><a href="/users/<%= user.id %>"><%= user.name %></a></td>
        <td><%= user.email %></td>
      </tr>
    <% }) %>
  </tbody>
</table>

<%- include('../partials/footer') %>
```

### include에 데이터 전달

```html
<!-- 데이터를 전달하며 include -->
<%- include('partials/card', { title: '공지사항', content: '서버 점검 예정' }) %>
```

```html
<!-- views/partials/card.ejs -->
<div class="card">
  <div class="card-header">
    <h3><%= title %></h3>
  </div>
  <div class="card-body">
    <p><%= content %></p>
  </div>
</div>
```

---

## Nunjucks

Nunjucks는 Mozilla에서 개발한 템플릿 엔진으로, Python의 Jinja2에서 영감을 받았다. 강력한 템플릿 상속과 매크로 기능을 제공한다.

### 설치 및 설정

```bash
$ npm i nunjucks
```

```javascript
const nunjucks = require('nunjucks');

app.set('view engine', 'njk');
nunjucks.configure('views', {
  express: app,
  autoescape: true,  // XSS 자동 방지
  watch: true,       // 파일 변경 시 자동 갱신 (개발용)
});
```

### 기본 문법

```html
{% raw %}<!-- views/index.njk -->
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>{{ title }}</title>
</head>
<body>
  <h1>{{ message }}</h1>
  <p>이것은 Nunjucks 템플릿입니다.</p>
</body>
</html>{% endraw %}
```

### Nunjucks 태그 종류

```
{% raw %}{{ 변수 }}          변수 출력 (자동 이스케이프)
{{ 변수 | safe }}    이스케이프 없이 출력
{% 제어문 %}         조건문, 반복문 등 제어 구조
{# 주석 #}          주석 (렌더링되지 않음){% endraw %}
```

### 조건문과 반복문

```html
{% raw %}{# 조건문 #}
{% if user %}
  <p>{{ user.name }}님, 환영합니다.</p>
{% elif guest %}
  <p>게스트 모드입니다.</p>
{% else %}
  <p>로그인이 필요합니다.</p>
{% endif %}

{# 반복문 #}
<ul>
  {% for todo in todos %}
    <li>{{ loop.index }}. {{ todo.text }}
      {% if todo.done %}✓{% endif %}
    </li>
  {% else %}
    <li>할 일이 없습니다.</li>
  {% endfor %}
</ul>{% endraw %}
```

```
{% raw %}Nunjucks 반복문 내장 변수:
  loop.index     — 1부터 시작하는 인덱스
  loop.index0    — 0부터 시작하는 인덱스
  loop.first     — 첫 번째 항목이면 true
  loop.last      — 마지막 항목이면 true
  loop.length    — 전체 항목 수{% endraw %}
```

### 필터

```html
{% raw %}{# 내장 필터 #}
{{ name | capitalize }}
{{ description | truncate(100) }}
{{ price | float | round(2) }}
{{ items | join(', ') }}
{{ createdAt | date('YYYY-MM-DD') }}
{{ content | safe }}
{{ name | default('익명') }}{% endraw %}
```

### 레이아웃 상속

```html
{% raw %}<!-- views/layout.njk — 기본 레이아웃 -->
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ title }} | My App</title>
  <link rel="stylesheet" href="/css/style.css">
  {% block styles %}{% endblock %}
</head>
<body>
  <header>
    <nav>
      <a href="/">홈</a>
      <a href="/users">사용자</a>
    </nav>
  </header>
  <main>
    {% block content %}{% endblock %}
  </main>
  <footer>
    <p>&copy; 2026 My App</p>
  </footer>
  {% block scripts %}{% endblock %}
</body>
</html>{% endraw %}
```

```html
{% raw %}<!-- views/index.njk — 레이아웃 확장 -->
{% extends "layout.njk" %}

{% block content %}
  <h1>{{ message }}</h1>
  <p>메인 페이지에 오신 것을 환영합니다.</p>
{% endblock %}{% endraw %}
```

### 매크로 (재사용 컴포넌트)

```html
{% raw %}<!-- views/macros/form.njk -->
{% macro input(name, label, type="text", value="") %}
  <div class="form-group">
    <label for="{{ name }}">{{ label }}</label>
    <input type="{{ type }}" id="{{ name }}" name="{{ name }}" value="{{ value }}">
  </div>
{% endmacro %}

{% macro button(text, type="submit", className="btn") %}
  <button type="{{ type }}" class="{{ className }}">{{ text }}</button>
{% endmacro %}{% endraw %}
```

```html
{% raw %}<!-- views/users/create.njk -->
{% extends "../layout.njk" %}
{% from "macros/form.njk" import input, button %}

{% block content %}
  <h1>사용자 등록</h1>
  <form method="POST" action="/users">
    {{ input('name', '이름') }}
    {{ input('email', '이메일', 'email') }}
    {{ input('password', '비밀번호', 'password') }}
    {{ button('가입하기') }}
  </form>
{% endblock %}{% endraw %}
```

---

## Express와 결합한 완성 예제

Pug를 사용한 사용자 CRUD 앱 예제:

### 프로젝트 구조

```
my-pug-app/
├── app.js
├── routes/
│   └── users.js
├── views/
│   ├── layout.pug
│   ├── index.pug
│   ├── error.pug
│   └── users/
│       ├── list.pug
│       ├── detail.pug
│       └── form.pug
├── public/
│   └── css/
│       └── style.css
└── package.json
```

### 서버 코드

```javascript
// app.js
const express = require('express');
const path = require('path');
const morgan = require('morgan');
const app = express();

app.set('view engine', 'pug');
app.set('views', path.join(__dirname, 'views'));

app.use(morgan('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

const usersRouter = require('./routes/users');
app.use('/users', usersRouter);

app.get('/', (req, res) => {
  res.render('index', { title: '홈' });
});

app.use((req, res) => {
  res.status(404).render('error', {
    title: '404',
    message: '페이지를 찾을 수 없습니다.',
  });
});

app.listen(3000, () => {
  console.log('http://localhost:3000');
});
```

### 라우터

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
  res.render('users/list', { title: '사용자 목록', users });
});

router.get('/new', (req, res) => {
  res.render('users/form', { title: '사용자 등록', user: null });
});

router.post('/', (req, res) => {
  const { name, email } = req.body;
  users.push({ id: nextId++, name, email });
  res.redirect('/users');
});

router.get('/:id', (req, res) => {
  const user = users.find((u) => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).render('error', { title: '404', message: '사용자 없음' });
  res.render('users/detail', { title: user.name, user });
});

module.exports = router;
```

### 뷰 템플릿

```pug
//- views/layout.pug
doctype html
html(lang="ko")
  head
    meta(charset="UTF-8")
    meta(name="viewport" content="width=device-width, initial-scale=1.0")
    title #{title} | My App
    link(rel="stylesheet" href="/css/style.css")
  body
    nav
      a(href="/") 홈
      | &nbsp;|&nbsp;
      a(href="/users") 사용자
    hr
    block content
```

```pug
//- views/users/list.pug
extends ../layout

block content
  h1 사용자 목록
  a(href="/users/new") + 새 사용자
  ul
    each user in users
      li
        a(href=`/users/${user.id}`) #{user.name}
        span  (#{user.email})
    else
      li 등록된 사용자가 없습니다.
```

```pug
//- views/users/form.pug
extends ../layout

block content
  h1 사용자 등록
  form(method="POST" action="/users")
    div
      label(for="name") 이름
      input(type="text" id="name" name="name" required)
    div
      label(for="email") 이메일
      input(type="email" id="email" name="email" required)
    button(type="submit") 등록
```

```pug
//- views/users/detail.pug
extends ../layout

block content
  h1= user.name
  p 이메일: #{user.email}
  a(href="/users") ← 목록으로
```

---

## 템플릿 엔진 비교 정리

### 문법 비교 (같은 내용을 각 엔진으로 표현)

```
[Pug]
ul
  each user in users
    li= user.name

[EJS]
<ul>
  <% users.forEach(user => { %>
    <li><%= user.name %></li>
  <% }) %>
</ul>

[Nunjucks]
<ul>
  {% raw %}{% for user in users %}{% endraw %}
    {% raw %}<li>{{ user.name }}</li>{% endraw %}
  {% raw %}{% endfor %}{% endraw %}
</ul>
```

### 기능 비교

| 기능 | Pug | EJS | Nunjucks |
|------|-----|-----|----------|
| HTML 친화도 | 낮음 (독자 문법) | 높음 (HTML 그대로) | 높음 (HTML 그대로) |
| 레이아웃 상속 | `extends` / `block` | `include`로 대체 | `extends` / `block` |
| 재사용 컴포넌트 | `mixin` | `include` + 데이터 전달 | `macro` |
| 필터 | 내장 필터 지원 | JavaScript 직접 사용 | 풍부한 내장 필터 |
| 자동 이스케이프 | 기본 적용 | `<%= %>`로 적용 | `autoescape` 옵션 |
| 러닝 커브 | 높음 | 낮음 | 중간 |
| 코드 간결성 | 매우 간결 | 보통 | 보통 |

### 선택 가이드

```
HTML에 익숙하고 빠르게 시작하고 싶다
  → EJS

간결한 문법과 강력한 레이아웃 상속이 필요하다
  → Pug

Jinja2 경험이 있거나 매크로·필터를 적극 활용하고 싶다
  → Nunjucks

프론트엔드와 완전 분리된 REST API를 만든다
  → 템플릿 엔진 불필요 (res.json()으로 충분)
```

> 최근에는 React, Vue 등 **프론트엔드 프레임워크**가 클라이언트 사이드 렌더링(CSR)을 담당하고, 서버는 API만 제공하는 구조가 일반적이다. 하지만 SEO가 중요하거나, 간단한 관리자 페이지를 만들 때는 여전히 서버 사이드 템플릿 엔진이 유용하다.

