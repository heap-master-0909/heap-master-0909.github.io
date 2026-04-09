---
title: 11-01. unit test (jest)
date: 2026-04-09 00:00:00 +0900
categories: [Node.JS, unit-test]
tags: [Tech, Node.JS, unit-test, jest]
pin: true
---

## Jest란?

Jest는 Facebook(Meta)이 만든 JavaScript 테스트 프레임워크입니다. Node.js와 React 프로젝트에서 가장 널리 사용됩니다.

### 설치 및 설정

```
npm i -D jest
```

package.json에 테스트 스크립트를 추가합니다:

```json
{
  "scripts": {
    "test": "jest"
  }
}
```

Jest는 기본적으로 다음 패턴의 파일을 자동으로 찾아 실행합니다:
* *.test.js
* *.spec.js
* __tests__ 폴더 안의 *.js 파일

### 기본 문법

test / it
테스트 케이스를 정의합니다. test와 it은 동일합니다.

```js
test('1 + 1은 2입니다.', () => {
  expect(1 + 1).toEqual(2);
});
it('빈 문자열은 falsy입니다.', () => {
  expect('').toBeFalsy();
});
```

describe
관련된 테스트를 그룹으로 묶습니다.

```js
describe('계산기', () => {
  test('덧셈', () => {
    expect(1 + 2).toBe(3);
  });
  test('곱셈', () => {
    expect(2 * 3).toBe(6);
  });
});

```

### 실전 예시 — 미들웨어 테스트

현재 프로젝트의 middlewares/index.js처럼 Express 미들웨어를 테스트할 때는 req, res 객체를 **모킹(가짜 객체)**해서 사용합니다:

```js
const { isLoggedIn, isNotLoggedIn } = require('./index');
describe('isLoggedIn', () => {
  const res = {
    status: jest.fn(() => res),
    send: jest.fn(),
  };
  const next = jest.fn();
  test('로그인되어 있으면 next를 호출해야 합니다.', () => {
    const req = { isAuthenticated: jest.fn(() => true) };
    isLoggedIn(req, res, next);
    expect(next).toHaveBeenCalledTimes(1);
  });
  test('로그인되어 있지 않으면 403 에러를 응답해야 합니다.', () => {
    const req = { isAuthenticated: jest.fn(() => false) };
    isLoggedIn(req, res, next);
    expect(res.status).toHaveBeenCalledWith(403);
  });
});
```

jest.fn() — 호출 여부, 호출 횟수, 전달된 인자를 추적할 수 있는 가짜 함수
toHaveBeenCalledTimes(n) — 함수가 n번 호출되었는지 확인
toHaveBeenCalledWith(값) — 함수가 특정 인자로 호출되었는지 확인

### 테스트 실행

```sh
# 전체 테스트 실행
npm test
# 특정 파일만 실행
npx jest middlewares/index.test.js
# 워치 모드 (파일 변경 시 자동 재실행)
npx jest --watchAll
# 코드 커버리지 확인
npx jest --coverage
```
