---
title: 11-02. jest example
date: 2026-04-09 00:00:00 +0900
categories: [Node.JS, unit-test]
tags: [Tech, Node.JS, unit-test, jest]
pin: true
---

## Example Code

```js
const User = require('../models/user');
exports.follow = async (req, res, next) => {
  try {
    const user = await User.findOne({ where: { id: req.user.id } });
    if (user) {
      await user.addFollowing(parseInt(req.params.id, 10));
      res.send('success');
    } else {
      res.status(404).send('no user');
    }
  } catch (error) {
    console.error(error);
    next(error);
  }
};
```

* 이 함수는 3가지 경우가 존재한다:
    * 유저를 찾으면 → 팔로잉 추가 후 'success' 응답
    * 유저를 못 찾으면 → 404 상태코드와 'no user' 응답
    * DB 에러 발생 시 → next(error) 호출

```js
// 테스트 코드 전체
jest.mock('../models/user');
const User = require('../models/user');
const { follow } = require('./user');
describe('follow', () => {
  const req = {
    user: { id: 1 },
    params: { id: 2 },
  };
  const res = {
    status: jest.fn(() => res),
    send: jest.fn(),
  };
  const next = jest.fn();
  test('사용자를 찾아 팔로잉을 추가하고 success를 응답해야 함', async () => {
    User.findOne.mockReturnValue({
      addFollowing(id) {
        return Promise.resolve(true);
      }
    });
    await follow(req, res, next);
    expect(res.send).toBeCalledWith('success');
  });
  test('사용자를 못 찾으면 res.status(404).send(no user)를 호출함', async () => {
    User.findOne.mockReturnValue(null);
    await follow(req, res, next);
    expect(res.status).toBeCalledWith(404);
    expect(res.send).toBeCalledWith('no user');
  });
  test('DB에서 에러가 발생하면 next(error) 호출함', async () => {
    const message = 'DB에러';
    User.findOne.mockReturnValue(Promise.reject(message));
    await follow(req, res, next);
    expect(next).toBeCalledWith(message);
  });
});
```

### 핵심 개념 하나씩 뜯어보기

1. `jest.mock()` — 모듈 모킹

`jest.mock('../models/user');`

실제 User 모델은 데이터베이스에 연결되어 있다. 유닛 테스트에서는 DB 연결 없이 컨트롤러 로직만 테스트하고 싶기 때문에, jest.mock()으로 모듈 전체를 **가짜(mock)**로 교체한다.

이렇게 하면 User.findOne 같은 메서드가 실제 DB 쿼리를 하지 않고, 원하는 값을 반환하도록 조작할 수 있다.

2. `jest.fn()` — 가짜 함수 만들기

```js
const myFn = jest.fn();
myFn('hello');
myFn(123);
expect(myFn).toBeCalledWith('hello');  // ✅ 통과
expect(myFn).toBeCalledWith(123);     // ✅ 통과
expect(myFn).toBeCalledWith('world'); // ❌ 실패
```

jest.fn()은 가짜 함수를 만든다. 실제로 아무 동작도 하지 않지만, 누가 호출했는지, 어떤 인자를 넘겼는지 자동으로 기록한다. 나중에 expect().toBeCalledWith()로 검증할 수 있다.

"이 함수가 정말 호출되었는가?"를 확인하기 위한 감시 카메라 같은 것이다.

3. Express의 req, res, next 이해하기
Express 미들웨어는 항상 세 개의 인자를 받는다.

| 인자 | 역할 |
|:---|:---|
| req | 브라우저가 서버에게 보낸 요청 정보 (URL 파라미터, 로그인 유저 등) |
| res | 서버가 브라우저에게 보내는 응답 객체 |
| next | 에러 발생 시 다음 에러 처리 미들웨어로 넘기는 함수 |
| res.send() | 브라우저에게 데이터를 보내는 메서드. 넘긴 인자는 HTTP 응답의 body로 들어간다. |

| 호출 | Content-Type (자동 설정) | Body |
|:---|:---|:---|
| res.send('hello') | text/html | hello |
| res.send({ name: 'kim' }) | application/json | {"name":"kim"} |
| res.send([1, 2, 3]) | application/json | [1,2,3] |
| res.status() | HTTP 상태 코드를 지정하는 메서드. | status()만으로는 응답이 전송되지 않고, send()를 호출해야 실제 전송된다. |

```js
res.status(404).send('no user');
// status(404) → 상태 코드 설정, res 자신을 반환
// .send('no user') → 체이닝으로 이어서 데이터 전송
```

4. 가짜 req, res, next 만들기

테스트에서는 실제 HTTP 요청이 없으므로, 필요한 속성만 가진 가짜 객체를 만든다.

```js
const req = {
  user: { id: 1 },       // 로그인한 사용자 ID
  params: { id: 2 },     // URL 파라미터 (팔로우 대상 ID)
};
const res = {
  status: jest.fn(() => res),  // res를 반환해야 .send() 체이닝 가능
  send: jest.fn(),             // 호출 기록만 남기면 됨
};
const next = jest.fn();        // 에러 전달 함수의 가짜 버전
여기서 status: jest.fn(() => res)가 핵심이다. status()가 res 자신을 반환해야 .send()를 이어서 호출할 수 있기 때문이다.

// jest.fn(() => res)로 만든 경우:
res.status(404).send('no user');  // ✅ 정상 동작
// jest.fn()으로만 만든 경우:
res.status(404).send('no user');  // 💥 TypeError!
// status()가 undefined를 반환하므로 undefined.send()가 되어 에러
5. mockReturnValue — mock 함수의 반환값 지정
User.findOne.mockReturnValue({
  addFollowing(id) {
    return Promise.resolve(true);
  }
});
```

mockReturnValue는 mock 함수가 호출될 때 어떤 값을 반환할지 지정한다. 위 코드는 User.findOne()이 호출되면 addFollowing 메서드를 가진 가짜 유저 객체를 반환하라는 뜻이다.

| 테스트 케이스 | mockReturnValue 설정 | 시뮬레이션 상황 |
|:---|:---|:---|
| 성공 | { addFollowing() {...} } | DB에서 유저를 찾음 |
| 유저 없음 | null | DB에서 유저를 못 찾음 |
| DB 에러 | Promise.reject('DB에러') | DB 쿼리 중 에러 발생 |
