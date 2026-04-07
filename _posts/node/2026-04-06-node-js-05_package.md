---
title: 05. Package(npm)
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, npm]
tags: [Tech, Node.JS]
pin: true
---

> Node.JS 생태계에서 **패키지(Package)**는 재사용 가능한 코드 묶음이며, **npm(Node Package Manager)**을 통해 설치·관리한다.

## npm이란?

npm은 Node.JS의 기본 패키지 관리자로, 전 세계 개발자들이 만든 200만 개 이상의 패키지를 제공한다.

```
npm 역할
├─ 패키지 설치 / 삭제 / 업데이트
├─ 의존성(dependency) 관리
├─ 스크립트 실행 (build, test, start 등)
└─ 패키지 배포 (npm publish)
```

Node.JS를 설치하면 npm이 함께 설치된다.

```bash
$ node -v
v20.x.x

$ npm -v
10.x.x
```

---

## package.json

`package.json`은 프로젝트의 **메타 정보**와 **의존성 목록**을 관리하는 핵심 파일이다.

### 프로젝트 초기화

```bash
$ mkdir my-project && cd my-project

# 대화형으로 생성
$ npm init

# 기본값으로 즉시 생성
$ npm init -y
```

### package.json 구조

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "description": "프로젝트 설명",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "node --watch index.js",
    "test": "jest"
  },
  "keywords": ["node", "example"],
  "author": "홍길동",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.7.0"
  }
}
```

### 주요 필드 설명

| 필드 | 설명 |
|------|------|
| **name** | 패키지 이름 (소문자, 하이픈 사용 가능) |
| **version** | 버전 (`Major.Minor.Patch` — SemVer) |
| **main** | 진입점 파일 경로 |
| **scripts** | `npm run <이름>`으로 실행할 명령어 모음 |
| **dependencies** | 프로덕션에 필요한 패키지 |
| **devDependencies** | 개발 시에만 필요한 패키지 |

---

## 패키지 설치

### 프로덕션 의존성 설치

```bash
# dependencies에 추가
$ npm install express
$ npm i express          # 축약형
```

### 개발 의존성 설치

```bash
# devDependencies에 추가
$ npm install --save-dev jest
$ npm i -D jest          # 축약형
```

### 전역 설치

```bash
# 시스템 어디서든 CLI로 사용 가능
$ npm install -g nodemon
$ nodemon --version
```

### 설치 옵션 비교

| 명령어 | 저장 위치 | 용도 |
|--------|-----------|------|
| `npm i 패키지` | dependencies | 프로덕션 코드에 필요 |
| `npm i -D 패키지` | devDependencies | 개발·테스트용 |
| `npm i -g 패키지` | 전역 | CLI 도구 |

---

## node_modules와 package-lock.json

### node_modules

패키지가 설치되면 `node_modules` 폴더에 실제 코드가 저장된다.

```
my-project/
├── node_modules/      ← 설치된 패키지 (Git에 포함하지 않음)
│   ├── express/
│   └── ...
├── package.json
└── package-lock.json
```

> `node_modules`는 `.gitignore`에 반드시 추가해야 한다. 다른 환경에서는 `npm install`로 재설치한다.

```gitignore
# .gitignore
node_modules/
```

### package-lock.json

설치된 패키지의 **정확한 버전**을 기록하는 파일이다.

| 파일 | 역할 |
|------|------|
| package.json | 허용 버전 **범위** 지정 (`^4.18.2`) |
| package-lock.json | 실제 설치된 **정확한 버전** 고정 |

> `package-lock.json`은 Git에 커밋해야 한다. 이를 통해 팀원 모두 동일한 버전의 패키지를 사용할 수 있다.

---

## SemVer (유의적 버전)

npm은 **Semantic Versioning(SemVer)** 규칙을 따른다.

```
버전: Major.Minor.Patch
예시: 4.18.2

Major (4) — 하위 호환이 깨지는 변경
Minor (18) — 하위 호환되는 새 기능 추가
Patch (2) — 하위 호환되는 버그 수정
```

### 버전 범위 기호

| 기호 | 의미 | 예시 | 허용 범위 |
|------|------|------|-----------|
| `^` | Minor 업데이트 허용 | `^4.18.2` | `4.18.2` ~ `4.x.x` |
| `~` | Patch 업데이트만 허용 | `~4.18.2` | `4.18.2` ~ `4.18.x` |
| 없음 | 정확히 해당 버전 | `4.18.2` | `4.18.2`만 |
| `*` | 모든 버전 | `*` | 전부 |
| `>=` | 이상 | `>=4.0.0` | `4.0.0` 이상 전부 |

```bash
# 현재 설치된 패키지 버전 확인
$ npm list

# 최상위 의존성만 확인
$ npm list --depth=0

# 업데이트 가능한 패키지 확인
$ npm outdated
```

---

## 자주 쓰는 npm 명령어

### 패키지 관리

```bash
# 패키지 설치 (package.json 기반 전체 설치)
$ npm install
$ npm i

# 특정 버전 설치
$ npm i express@4.18.2

# 패키지 삭제
$ npm uninstall express
$ npm rm express         # 축약형

# 패키지 업데이트
$ npm update

# 특정 패키지 업데이트
$ npm update express
```

### 스크립트 실행

```bash
# package.json의 scripts 실행
$ npm run start
$ npm start              # start는 run 생략 가능

$ npm run dev
$ npm test               # test도 run 생략 가능
```

### 정보 조회

```bash
# 패키지 정보 확인
$ npm info express

# 패키지 검색
$ npm search http-server

# 전역 설치 목록
$ npm list -g --depth=0
```

---

## npx — 패키지 실행 도구

`npx`는 패키지를 **설치 없이 바로 실행**하거나, 로컬에 설치된 패키지를 실행할 때 사용한다.

```bash
# 설치 없이 일회성 실행
$ npx create-react-app my-app

# 로컬에 설치된 패키지 실행
$ npx jest --coverage

# 특정 버전으로 실행
$ npx node@18 -e "console.log(process.version)"
```

```
npm vs npx
─────────────────────────────────────────
npm install -g create-react-app   → 전역 설치 후 사용
npx create-react-app              → 설치 없이 즉시 실행
```

> `npx`는 항상 최신 버전을 사용하므로, 전역 설치 시 발생하는 버전 불일치 문제를 방지할 수 있다.

---

## 나만의 패키지 만들기

### 기본 구조

```
my-package/
├── package.json
├── index.js         ← main 진입점
├── lib/
│   └── utils.js     ← 내부 모듈
└── README.md
```

### 모듈 작성

```javascript
// lib/utils.js
function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

module.exports = { add, multiply };
```

```javascript
// index.js
const { add, multiply } = require('./lib/utils');

module.exports = { add, multiply };
```

### 사용

```javascript
// 다른 프로젝트에서
const { add, multiply } = require('my-package');

console.log(add(2, 3));       // 5
console.log(multiply(4, 5));  // 20
```

---

## npm 배포하기

```bash
# 1. npm 계정 생성 (https://www.npmjs.com)

# 2. 로그인
$ npm login

# 3. 배포
$ npm publish

# 4. 버전 올리기 (patch: 0.0.x → 0.0.y)
$ npm version patch
$ npm publish

# 5. 배포 취소 (72시간 이내)
$ npm unpublish my-package --force
```

```
배포 흐름:
  npm login → npm publish → npm version patch → npm publish
      │                            │
      └─ npmjs.com 계정 인증        └─ version 자동 증가
```

> `name` 필드가 이미 npm에 존재하면 배포가 실패한다. **scoped package** (`@username/package-name`) 형태를 사용하면 이름 충돌을 피할 수 있다.

---

## 유용한 npm 패키지 모음

| 패키지 | 설명 | 설치 |
|--------|------|------|
| **express** | 가장 인기 있는 웹 프레임워크 | `npm i express` |
| **dotenv** | `.env` 파일로 환경변수 관리 | `npm i dotenv` |
| **nodemon** | 코드 변경 시 자동 재시작 | `npm i -D nodemon` |
| **morgan** | HTTP 요청 로깅 | `npm i morgan` |
| **cors** | CORS 설정 | `npm i cors` |
| **uuid** | 고유 ID 생성 | `npm i uuid` |
| **dayjs** | 날짜/시간 처리 (경량) | `npm i dayjs` |
| **chalk** | 터미널 색상 출력 | `npm i chalk` |
| **jest** | 테스트 프레임워크 | `npm i -D jest` |
| **eslint** | 코드 품질 검사 | `npm i -D eslint` |

---

## yarn과 pnpm

npm 외에도 대체 패키지 관리자가 있다.

| 관리자 | 특징 |
|--------|------|
| **npm** | Node.JS 기본 제공, 가장 넓은 호환성 |
| **yarn** | Facebook 개발, 빠른 설치 속도, `yarn.lock` |
| **pnpm** | 디스크 공간 절약 (심볼릭 링크 방식), 엄격한 의존성 관리 |

```bash
# yarn 설치 및 사용
$ npm i -g yarn
$ yarn add express
$ yarn add --dev jest
$ yarn install

# pnpm 설치 및 사용
$ npm i -g pnpm
$ pnpm add express
$ pnpm add -D jest
$ pnpm install
```
