---
title: 08-02. Mongoose
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, MongoDB]
tags: [Tech, Node.JS, MongoDB, NoSQL, Mongoose]
pin: true
---

## Mongoose — Node.JS에서 MongoDB 사용

**Mongoose**는 MongoDB를 Node.JS에서 사용하기 위한 **ODM(Object-Document Mapping)** 라이브러리다. RDBMS의 ORM(Sequelize)에 대응하는 개념이다.

```
ODM (Object-Document Mapping) 개념:
──────────────────────────────────────────────
자바스크립트 세계             MongoDB 세계
  스키마(Schema)       ←→     컬렉션 구조 정의
  모델(Model)          ←→     컬렉션(Collection)
  인스턴스(Object)     ←→     문서(Document)
  속성(Property)       ←→     필드(Field)

Mongoose가 자동으로 변환:
  User.find()          →  db.users.find()
  User.create({})      →  db.users.insertOne({})
  user.save()          →  db.users.updateOne({})
  user.deleteOne()     →  db.users.deleteOne({})
──────────────────────────────────────────────
```

### 설치 및 연결

```bash
$ npm i mongoose
```

```javascript
// schemas/index.js
const mongoose = require('mongoose');

const connect = () => {
  if (process.env.NODE_ENV !== 'production') {
    mongoose.set('debug', true);
  }

  mongoose.connect('mongodb://127.0.0.1:27017/nodejs_db', {})
    .then(() => console.log('MongoDB 연결 성공'))
    .catch((err) => console.error('MongoDB 연결 에러:', err));
};

mongoose.connection.on('error', (error) => {
  console.error('MongoDB 연결 에러:', error);
});

mongoose.connection.on('disconnected', () => {
  console.error('MongoDB 연결 끊김. 재연결 시도...');
  connect();
});

module.exports = connect;
```

```javascript
// app.js
const express = require('express');
const connect = require('./schemas');

const app = express();

connect();

app.listen(3000, () => {
  console.log('http://localhost:3000');
});
```

```
mongoose.connect() 주요 옵션:
  'mongodb://127.0.0.1:27017/nodejs_db'  — 접속 URI
  
  URI 구조:
    mongodb://[username:password@]host:port/database
    mongodb://root:1234@127.0.0.1:27017/nodejs_db   (인증 포함)

mongoose.set('debug', true):
  → 실행되는 쿼리를 콘솔에 출력 (개발 시 유용)
```

---

## 스키마 · 모델 정의

MongoDB는 스키마가 없지만, Mongoose는 **스키마를 정의하여 데이터 구조를 강제**할 수 있다.

### User 스키마 · 모델

```javascript
// schemas/user.js
const mongoose = require('mongoose');

const { Schema } = mongoose;

const userSchema = new Schema({
  name: {
    type: String,
    required: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
  },
  age: {
    type: Number,
    required: true,
    min: 0,
  },
  married: {
    type: Boolean,
    required: true,
    default: false,
  },
  comment: {
    type: String,
  },
  createdAt: {
    type: Date,
    default: Date.now,
  },
});

module.exports = mongoose.model('User', userSchema);
```

### Comment 스키마 · 모델

```javascript
// schemas/comment.js
const mongoose = require('mongoose');

const { Schema } = mongoose;

const commentSchema = new Schema({
  commenter: {
    type: Schema.Types.ObjectId,
    required: true,
    ref: 'User',
  },
  comment: {
    type: String,
    required: true,
  },
  createdAt: {
    type: Date,
    default: Date.now,
  },
});

module.exports = mongoose.model('Comment', commentSchema);
```

```
ref: 'User' — 다른 컬렉션과의 관계를 정의
──────────────────────────────────────────────
  commenter 필드에 User 모델(users 컬렉션)의 _id를 저장한다.
  populate() 메서드를 사용하면 자동으로 참조 문서를 가져온다.

  RDBMS의 외래키(Foreign Key)와 유사한 개념이지만
  MongoDB 레벨의 제약이 아닌 Mongoose 레벨의 참조다.
──────────────────────────────────────────────
```

### Mongoose 스키마 타입

```
Mongoose 타입          자바스크립트 타입         설명
────────────────────────────────────────────────────
String                 String                   문자열
Number                 Number                   숫자
Boolean                Boolean                  논리값
Date                   Date                     날짜
Buffer                 Buffer                   바이너리 데이터
Schema.Types.ObjectId  ObjectId                 MongoDB ObjectId
Array                  Array                    배열
Schema.Types.Mixed     any                      어떤 타입이든 가능
Map                    Map                      키-값 쌍
```

### 스키마 옵션 (Validator)

```
필드 옵션:
  type         타입 지정 (필수)
  required     필수 여부 (true/false 또는 [true, '에러 메시지'])
  default      기본값
  unique       유니크 제약
  min / max    숫자 최소/최대
  minlength / maxlength  문자열 최소/최대 길이
  enum         허용 값 목록 (['admin', 'user', 'guest'])
  match        정규식 검증 (/^[a-z]+$/)
  validate     커스텀 검증 함수
  trim         공백 제거 (String)
  lowercase    소문자 변환 (String)
  uppercase    대문자 변환 (String)
```

```javascript
// 옵션 사용 예시
const userSchema = new Schema({
  email: {
    type: String,
    required: [true, '이메일은 필수입니다.'],
    unique: true,
    match: [/^\S+@\S+\.\S+$/, '유효한 이메일 형식이 아닙니다.'],
    lowercase: true,
    trim: true,
  },
  role: {
    type: String,
    enum: ['admin', 'user', 'guest'],
    default: 'user',
  },
  age: {
    type: Number,
    min: [0, '나이는 0 이상이어야 합니다.'],
    max: [150, '나이는 150 이하여야 합니다.'],
  },
});
```

---

## Mongoose CRUD 쿼리

```javascript
const User = require('./schemas/user');
const Comment = require('./schemas/comment');
```

### Create (생성)

```javascript
const user = await User.create({
  name: '홍길동',
  email: 'hong@example.com',
  age: 25,
  married: false,
});
console.log(user._id); // ObjectId

// 또는 인스턴스 생성 후 저장
const user2 = new User({
  name: '김철수',
  email: 'kim@example.com',
  age: 30,
  married: true,
});
await user2.save();
```

### Read (조회)

```javascript
// 전체 조회
const users = await User.find();

// 특정 필드만 조회
const users = await User.find({}, 'name email');
// 또는
const users = await User.find().select('name email');

// 조건 조회
const user = await User.findOne({ name: '홍길동' });
const user = await User.findById('663a1b2c...');

// 비교 연산자
const users = await User.find({ age: { $gt: 25 } });
const users = await User.find({ age: { $gte: 25, $lte: 30 } });

// OR 조건
const users = await User.find({
  $or: [{ age: 25 }, { age: 30 }],
});

// 정렬
const users = await User.find().sort({ age: -1 });
const users = await User.find().sort('-age name');

// 제한 (페이지네이션)
const users = await User.find()
  .sort({ age: -1 })
  .skip(10)
  .limit(5);

// 개수
const count = await User.countDocuments({ married: true });
```

### Update (수정)

```javascript
// 단건 수정
const result = await User.updateOne(
  { name: '홍길동' },
  { $set: { age: 26 } }
);
console.log(result.modifiedCount); // 수정된 문서 수

// findOneAndUpdate — 수정 후 문서 반환
const user = await User.findOneAndUpdate(
  { name: '홍길동' },
  { $set: { age: 26 } },
  { new: true }          // true: 수정 후 문서 반환, false: 수정 전 문서 반환
);

// findByIdAndUpdate
const user = await User.findByIdAndUpdate(
  '663a1b2c...',
  { $set: { age: 26, married: true } },
  { new: true }
);
```

### Delete (삭제)

```javascript
// 단건 삭제
const result = await User.deleteOne({ name: '홍길동' });

// 다건 삭제
const result = await User.deleteMany({ married: false });

// findOneAndDelete — 삭제 후 문서 반환
const user = await User.findOneAndDelete({ name: '홍길동' });

// findByIdAndDelete
const user = await User.findByIdAndDelete('663a1b2c...');
```

---

## populate — 관계 조회

Mongoose의 `populate()`는 **참조(ref)된 다른 컬렉션의 문서를 자동으로 가져오는** 기능이다. RDBMS의 JOIN에 해당한다.

```javascript
// 댓글 조회 시 작성자 정보 함께 가져오기
const comments = await Comment.find()
  .populate('commenter');
// comments[0].commenter → { _id: ..., name: '홍길동', email: '...' }

// 특정 필드만 populate
const comments = await Comment.find()
  .populate('commenter', 'name email');
// comments[0].commenter → { _id: ..., name: '홍길동', email: '...' }

// 특정 사용자의 댓글 조회
const comments = await Comment.find({ commenter: userId })
  .populate('commenter', 'name')
  .sort({ createdAt: -1 });
```

```
populate 동작 원리:
──────────────────────────────────────────────
  1단계: Comment.find() → 댓글 문서들 조회
    { commenter: ObjectId("663a..."), comment: "안녕" }

  2단계: .populate('commenter') → ObjectId로 User 컬렉션에서 조회
    User.findById("663a...") 를 내부적으로 실행

  3단계: 결과 합성
    { commenter: { _id: "663a...", name: "홍길동", ... }, comment: "안녕" }

  주의: populate는 내부적으로 별도 쿼리를 실행한다 (실제 JOIN이 아님)
──────────────────────────────────────────────
```

---

## Express + Mongoose 전체 프로젝트 구조

### 프로젝트 구조

```
express-mongoose-app/
├── app.js
├── schemas/
│   ├── index.js         ← MongoDB 연결 설정
│   ├── user.js          ← User 스키마·모델
│   └── comment.js       ← Comment 스키마·모델
├── routes/
│   ├── users.js         ← 사용자 라우터
│   └── comments.js      ← 댓글 라우터
├── package.json
└── .env
```

### 사용자 라우터

```javascript
// routes/users.js
const express = require('express');
const User = require('../schemas/user');
const Comment = require('../schemas/comment');
const router = express.Router();

// 전체 사용자 조회
router.get('/', async (req, res, next) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    next(err);
  }
});

// 단건 사용자 조회
router.get('/:id', async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);
    if (!user) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
    res.json(user);
  } catch (err) {
    next(err);
  }
});

// 사용자 생성
router.post('/', async (req, res, next) => {
  try {
    const user = await User.create({
      name: req.body.name,
      email: req.body.email,
      age: req.body.age,
      married: req.body.married,
      comment: req.body.comment,
    });
    res.status(201).json(user);
  } catch (err) {
    next(err);
  }
});

// 사용자 수정
router.put('/:id', async (req, res, next) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { $set: req.body },
      { new: true, runValidators: true }
    );
    if (!user) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
    res.json(user);
  } catch (err) {
    next(err);
  }
});

// 사용자 삭제
router.delete('/:id', async (req, res, next) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
    await Comment.deleteMany({ commenter: req.params.id });
    res.json({ message: '삭제되었습니다.' });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

### 댓글 라우터

```javascript
// routes/comments.js
const express = require('express');
const Comment = require('../schemas/comment');
const router = express.Router();

// 전체 댓글 조회 (작성자 포함)
router.get('/', async (req, res, next) => {
  try {
    const comments = await Comment.find()
      .populate('commenter', 'name email')
      .sort({ createdAt: -1 });
    res.json(comments);
  } catch (err) {
    next(err);
  }
});

// 특정 사용자의 댓글 조회
router.get('/user/:userId', async (req, res, next) => {
  try {
    const comments = await Comment.find({ commenter: req.params.userId })
      .sort({ createdAt: -1 });
    res.json(comments);
  } catch (err) {
    next(err);
  }
});

// 댓글 생성
router.post('/', async (req, res, next) => {
  try {
    const comment = await Comment.create({
      commenter: req.body.commenter,
      comment: req.body.comment,
    });
    const populatedComment = await Comment.findById(comment._id)
      .populate('commenter', 'name email');
    res.status(201).json(populatedComment);
  } catch (err) {
    next(err);
  }
});

// 댓글 수정
router.put('/:id', async (req, res, next) => {
  try {
    const comment = await Comment.findByIdAndUpdate(
      req.params.id,
      { $set: { comment: req.body.comment } },
      { new: true }
    );
    if (!comment) {
      return res.status(404).json({ error: '댓글을 찾을 수 없습니다.' });
    }
    res.json(comment);
  } catch (err) {
    next(err);
  }
});

// 댓글 삭제
router.delete('/:id', async (req, res, next) => {
  try {
    const comment = await Comment.findByIdAndDelete(req.params.id);
    if (!comment) {
      return res.status(404).json({ error: '댓글을 찾을 수 없습니다.' });
    }
    res.json({ message: '삭제되었습니다.' });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

---

## mongosh vs Mongoose 비교 치트시트

```
mongosh                                  Mongoose
────────────────────────────────────────────────────────────────
db.users.insertOne({ name: '홍길동' })  User.create({ name: '홍길동' })

db.users.find()                         User.find()

db.users.find({ age: { $gt: 25 } })    User.find({ age: { $gt: 25 } })

db.users.findOne({ _id: id })           User.findById(id)

db.users.find({}, { name: 1 })          User.find().select('name')

db.users.updateOne(                     User.updateOne(
  { _id: id },                            { _id: id },
  { $set: { age: 26 } }                   { $set: { age: 26 } }
)                                       )
                                        User.findByIdAndUpdate(id, { age: 26 })

db.users.deleteOne({ _id: id })         User.deleteOne({ _id: id })
                                        User.findByIdAndDelete(id)

db.users.find()                         User.find()
  .sort({ age: -1 })                     .sort({ age: -1 })
  .skip(10)                               .skip(10)
  .limit(5)                               .limit(5)

db.comments.aggregate([                 Comment.find()
  { $lookup: {                            .populate('commenter')
      from: 'users', ...
  }}
])
```

---

## MySQL vs MongoDB 선택 가이드

```
MySQL (RDBMS)을 선택해야 할 때:
  - 데이터 간 관계(JOIN)가 복잡한 경우
  - 트랜잭션이 중요한 경우 (결제, 은행 등)
  - 데이터 구조가 고정적이고 일관된 경우
  - 데이터 무결성이 최우선인 경우

MongoDB (NoSQL)를 선택해야 할 때:
  - 데이터 구조가 자주 변경되는 경우
  - 대량의 데이터를 빠르게 읽고 써야 하는 경우
  - JSON 형태의 비정형 데이터를 다루는 경우
  - 수평 확장(Scale-out)이 필요한 경우
  - 실시간 분석, 로그, IoT 데이터 등

실무에서는 두 가지를 함께 사용하기도 한다:
  예) 사용자·결제 → MySQL, 로그·채팅 → MongoDB
```

---
