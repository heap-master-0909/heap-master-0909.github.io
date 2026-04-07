---
title: 07-03. 시퀄라이즈 (Sequelize)
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, MySQL]
tags: [Tech, Node.JS, MySQL, Sequelize, ORM]
pin: true
---

> **Sequelize**는 Node.JS에서 가장 널리 사용되는 **ORM(Object-Relational Mapping)** 라이브러리다. SQL을 직접 작성하지 않고 자바스크립트 객체·메서드로 데이터베이스를 다룰 수 있다.

## ORM이란?

ORM(Object-Relational Mapping)은 **자바스크립트 객체와 데이터베이스 테이블을 자동으로 매핑**해주는 기술이다. SQL 대신 자바스크립트 메서드로 데이터를 조작할 수 있다.

```
ORM 개념:
──────────────────────────────────────────────
자바스크립트 세계           데이터베이스 세계
  클래스(Model)     ←→     테이블(Table)
  인스턴스(Object)  ←→     행(Row)
  속성(Property)    ←→     열(Column)

ORM이 자동으로 변환:
  User.findAll()    →  SELECT * FROM users;
  User.create({})   →  INSERT INTO users ...;
  user.update({})   →  UPDATE users SET ...;
  user.destroy()    →  DELETE FROM users ...;
──────────────────────────────────────────────
```

### ORM의 장단점

```
장점:
  - SQL을 몰라도 자바스크립트만으로 DB 조작 가능
  - 코드가 직관적이고 가독성이 좋다
  - MySQL, PostgreSQL, SQLite 등 DB 교체가 쉽다 (방언 자동 처리)
  - SQL 인젝션을 자동으로 방지한다
  - 마이그레이션으로 테이블 변경 이력을 관리할 수 있다

단점:
  - 복잡한 쿼리는 ORM으로 표현하기 어렵다
  - 자동 생성된 SQL이 비효율적일 수 있다
  - ORM 문법을 별도로 학습해야 한다
```

---

## Sequelize 설치 및 초기 설정

### 패키지 설치

```bash
# sequelize + MySQL 드라이버
$ npm i sequelize mysql2

# CLI 도구 (프로젝트 초기 구조 자동 생성)
$ npm i -g sequelize-cli
# 또는 npx로 실행
$ npx sequelize --version
```

### 프로젝트 초기화

```bash
$ npx sequelize init
```

실행하면 다음 폴더·파일이 자동 생성된다.

```
project/
├── config/
│   └── config.json      ← DB 접속 정보
├── models/
│   └── index.js         ← 모델 로딩 및 관계 설정
├── migrations/          ← 테이블 변경 이력
└── seeders/             ← 초기 데이터 삽입
```

### DB 접속 설정

```json
// config/config.json
{
  "development": {
    "username": "root",
    "password": "your_password",
    "database": "nodejs_db",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "test": {
    "username": "root",
    "password": null,
    "database": "nodejs_test",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "production": {
    "username": "root",
    "password": null,
    "database": "nodejs_prod",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
}
```

### models/index.js 구조

`sequelize init`으로 생성된 `models/index.js`는 모든 모델 파일을 자동으로 읽어들이고 Sequelize 인스턴스와 연결한다.

```javascript
// models/index.js (자동 생성, 핵심만 정리)
const Sequelize = require('sequelize');
const env = process.env.NODE_ENV || 'development';
const config = require('../config/config.json')[env];

const db = {};
const sequelize = new Sequelize(
  config.database, config.username, config.password, config
);

db.sequelize = sequelize;
db.Sequelize = Sequelize;

module.exports = db;
```

### Express와 연결

```javascript
// app.js
const express = require('express');
const { sequelize } = require('./models');

const app = express();

sequelize.sync({ force: false })
  .then(() => console.log('DB 연결 성공'))
  .catch((err) => console.error('DB 연결 실패:', err));

app.listen(3000, () => {
  console.log('http://localhost:3000');
});
```

```
sequelize.sync() 옵션:
  { force: false }  — 테이블이 없으면 생성, 있으면 유지 (운영용)
  { force: true }   — 테이블을 삭제 후 재생성 (데이터 소멸! 개발용)
  { alter: true }   — 모델과 테이블 차이를 비교해 컬럼 추가/수정 (주의 필요)
```

---

## 모델 정의

Sequelize에서 **모델(Model)**은 데이터베이스 테이블에 대응하는 자바스크립트 클래스다.

### User 모델

```javascript
// models/user.js
const Sequelize = require('sequelize');

class User extends Sequelize.Model {
  static initiate(sequelize) {
    User.init({
      name: {
        type: Sequelize.STRING(20),
        allowNull: false,
      },
      email: {
        type: Sequelize.STRING(100),
        allowNull: false,
        unique: true,
      },
      age: {
        type: Sequelize.INTEGER.UNSIGNED,
        allowNull: true,
      },
      married: {
        type: Sequelize.BOOLEAN,
        allowNull: false,
        defaultValue: false,
      },
      comment: {
        type: Sequelize.TEXT,
        allowNull: true,
      },
    }, {
      sequelize,
      timestamps: true,
      underscored: false,
      modelName: 'User',
      tableName: 'users',
      paranoid: false,
      charset: 'utf8mb4',
      collate: 'utf8mb4_general_ci',
    });
  }

  static associate(db) {
    db.User.hasMany(db.Comment, { foreignKey: 'commenter', sourceKey: 'id' });
  }
}

module.exports = User;
```

### Comment 모델

```javascript
// models/comment.js
const Sequelize = require('sequelize');

class Comment extends Sequelize.Model {
  static initiate(sequelize) {
    Comment.init({
      comment: {
        type: Sequelize.STRING(500),
        allowNull: false,
      },
    }, {
      sequelize,
      timestamps: true,
      underscored: false,
      modelName: 'Comment',
      tableName: 'comments',
      paranoid: false,
      charset: 'utf8mb4',
      collate: 'utf8mb4_general_ci',
    });
  }

  static associate(db) {
    db.Comment.belongsTo(db.User, { foreignKey: 'commenter', targetKey: 'id' });
  }
}

module.exports = Comment;
```

### models/index.js에 모델 등록

```javascript
// models/index.js
const Sequelize = require('sequelize');
const User = require('./user');
const Comment = require('./comment');

const env = process.env.NODE_ENV || 'development';
const config = require('../config/config.json')[env];
const db = {};

const sequelize = new Sequelize(
  config.database, config.username, config.password, config
);

db.sequelize = sequelize;
db.User = User;
db.Comment = Comment;

User.initiate(sequelize);
Comment.initiate(sequelize);

User.associate(db);
Comment.associate(db);

module.exports = db;
```

### Sequelize 자료형 ↔ MySQL 자료형 매핑

```
Sequelize 자료형         →  MySQL 자료형
────────────────────────────────────────
STRING(100)              →  VARCHAR(100)
TEXT                     →  TEXT
INTEGER                  →  INT
INTEGER.UNSIGNED         →  INT UNSIGNED
FLOAT                    →  FLOAT
DOUBLE                   →  DOUBLE
BOOLEAN                  →  TINYINT(1)
DATE                     →  DATETIME
DATEONLY                 →  DATE
UUID                     →  CHAR(36)
JSON                     →  JSON
ENUM('A', 'B')           →  ENUM('A', 'B')
```

### 모델 옵션 정리

```
timestamps: true     — createdAt, updatedAt 컬럼 자동 생성
underscored: false   — createdAt (false) vs created_at (true)
modelName: 'User'    — 자바스크립트에서 사용할 모델 이름
tableName: 'users'   — 실제 DB 테이블 이름
paranoid: false      — true면 deletedAt 컬럼 추가 (소프트 삭제)
charset / collate    — 문자 인코딩 설정 (한글: utf8mb4)
```

```
paranoid 옵션 (소프트 삭제):
──────────────────────────────────────────────
paranoid: false (기본값)
  destroy() → DELETE FROM users WHERE id = 1
  → 데이터가 실제로 삭제됨

paranoid: true
  destroy() → UPDATE users SET deletedAt = NOW() WHERE id = 1
  → 데이터는 남아있지만 일반 조회에서 제외됨
  → 복구 가능 (deletedAt을 NULL로 변경)
──────────────────────────────────────────────
```

---

## 관계 정의

Sequelize에서 테이블 간 관계를 정의하면 **JOIN 쿼리를 자동 생성**하고 **외래키(Foreign Key)**를 관리해준다.

### 1:N 관계 (hasMany / belongsTo)

한 명의 사용자가 여러 개의 댓글을 작성할 수 있다.

```javascript
// User 모델 (1 쪽)
db.User.hasMany(db.Comment, { foreignKey: 'commenter', sourceKey: 'id' });

// Comment 모델 (N 쪽)
db.Comment.belongsTo(db.User, { foreignKey: 'commenter', targetKey: 'id' });
```

```
1:N 관계 (hasMany / belongsTo):
──────────────────────────────────────────────
  User (1)               Comment (N)
┌────┬────────┐       ┌────┬───────────┬────────┐
│ id │ name   │       │ id │ commenter │ comment│
├────┼────────┤       ├────┼───────────┼────────┤
│  1 │ 홍길동 │──┐    │  1 │     1     │ 안녕   │
│  2 │ 김철수 │  ├──→ │  2 │     1     │ 반갑   │
│  3 │ 이영희 │  │    │  3 │     2     │ Hi     │
└────┴────────┘  │    └────┴───────────┴────────┘
                 │
  sourceKey: 'id'     foreignKey: 'commenter'

hasMany:   "User는 Comment를 여러 개 가진다"
belongsTo: "Comment는 User에게 속한다"
──────────────────────────────────────────────
```

### 1:1 관계 (hasOne / belongsTo)

```javascript
db.User.hasOne(db.Profile, { foreignKey: 'userId', sourceKey: 'id' });
db.Profile.belongsTo(db.User, { foreignKey: 'userId', targetKey: 'id' });
```

### N:M 관계 (belongsToMany)

하나의 게시글에 여러 해시태그가 달릴 수 있고, 하나의 해시태그가 여러 게시글에 사용될 수 있다.

```javascript
db.Post.belongsToMany(db.Hashtag, { through: 'PostHashtag' });
db.Hashtag.belongsToMany(db.Post, { through: 'PostHashtag' });
```

```
N:M 관계 (belongsToMany):
──────────────────────────────────────────────
  Post                PostHashtag (중간 테이블)       Hashtag
┌────┬────────┐     ┌────────┬───────────┐      ┌────┬──────────┐
│ id │ title  │     │ PostId │ HashtagId │      │ id │ title    │
├────┼────────┤     ├────────┼───────────┤      ├────┼──────────┤
│  1 │ 글A    │──→  │   1    │     1     │  ←── │  1 │ #Node    │
│  2 │ 글B    │──→  │   1    │     2     │  ←── │  2 │ #Express │
└────┴────────┘     │   2    │     1     │      └────┴──────────┘
                    └────────┴───────────┘

through: 'PostHashtag'  — 자동 생성되는 중간 테이블 이름
──────────────────────────────────────────────
```

---

## Sequelize 쿼리

### CRUD 기본 쿼리

```javascript
const { User, Comment } = require('./models');
```

#### Create (생성)

```javascript
// SQL: INSERT INTO users (name, email, age, married) VALUES ('홍길동', 'hong@example.com', 25, false);
const user = await User.create({
  name: '홍길동',
  email: 'hong@example.com',
  age: 25,
  married: false,
});
console.log(user.id); // 자동 생성된 id
```

#### Read (조회)

```javascript
// SQL: SELECT * FROM users;
const users = await User.findAll();

// SQL: SELECT name, email FROM users;
const users = await User.findAll({
  attributes: ['name', 'email'],
});

// SQL: SELECT * FROM users WHERE id = 1;
const user = await User.findOne({
  where: { id: 1 },
});

// SQL: SELECT * FROM users WHERE married = true AND age > 25;
const { Op } = require('sequelize');
const users = await User.findAll({
  where: {
    married: true,
    age: { [Op.gt]: 25 },
  },
});

// SQL: SELECT * FROM users WHERE age = 25 OR age = 30;
const users = await User.findAll({
  where: {
    [Op.or]: [
      { age: 25 },
      { age: 30 },
    ],
  },
});

// SQL: SELECT * FROM users ORDER BY age DESC LIMIT 5 OFFSET 10;
const users = await User.findAll({
  order: [['age', 'DESC']],
  limit: 5,
  offset: 10,
});
```

#### Update (수정)

```javascript
// SQL: UPDATE users SET age = 26 WHERE id = 1;
const [affectedCount] = await User.update(
  { age: 26 },
  { where: { id: 1 } }
);
```

#### Delete (삭제)

```javascript
// SQL: DELETE FROM users WHERE id = 1;
const deletedCount = await User.destroy({
  where: { id: 1 },
});
```

### Op (연산자) 정리

```
Sequelize 연산자            →  SQL 변환
────────────────────────────────────────
Op.eq                       →  = (같다)
Op.ne                       →  != (같지 않다)
Op.gt                       →  > (초과)
Op.gte                      →  >= (이상)
Op.lt                       →  < (미만)
Op.lte                      →  <= (이하)
Op.between                  →  BETWEEN
Op.notBetween               →  NOT BETWEEN
Op.in                       →  IN
Op.notIn                    →  NOT IN
Op.like                     →  LIKE
Op.notLike                  →  NOT LIKE
Op.or                       →  OR
Op.and                      →  AND
```

```javascript
const { Op } = require('sequelize');

// BETWEEN
where: { age: { [Op.between]: [25, 30] } }
// → WHERE age BETWEEN 25 AND 30

// IN
where: { id: { [Op.in]: [1, 3, 5] } }
// → WHERE id IN (1, 3, 5)

// LIKE
where: { name: { [Op.like]: '김%' } }
// → WHERE name LIKE '김%'

// NOT LIKE
where: { email: { [Op.notLike]: '%gmail%' } }
// → WHERE email NOT LIKE '%gmail%'
```

---

## 관계 쿼리 (JOIN)

### include를 사용한 JOIN 조회

```javascript
// SQL: SELECT * FROM users u INNER JOIN comments c ON u.id = c.commenter;
const users = await User.findAll({
  include: [{
    model: Comment,
  }],
});
// users[0].Comments 로 접근
```

```javascript
// 특정 사용자와 그 댓글 조회
const user = await User.findOne({
  where: { id: 1 },
  include: [{
    model: Comment,
    attributes: ['id', 'comment'],
  }],
});
console.log(user.name);
console.log(user.Comments); // 해당 사용자의 모든 댓글
```

```javascript
// 댓글과 작성자 정보 함께 조회
const comments = await Comment.findAll({
  include: [{
    model: User,
    attributes: ['id', 'name'],
  }],
});
// comments[0].User.name 으로 접근
```

### 관계 메서드 (get / set / add / remove)

관계를 정의하면 Sequelize가 자동으로 편의 메서드를 생성한다.

```javascript
const user = await User.findOne({ where: { id: 1 } });

// getComments — 해당 사용자의 댓글 조회
const comments = await user.getComments();

// addComment — 해당 사용자에게 댓글 추가
await user.addComment(commentInstance);
// 또는 직접 생성
const newComment = await Comment.create({
  commenter: user.id,
  comment: '새 댓글입니다.',
});

// removeComment — 관계 제거
await user.removeComment(commentInstance);
```

```
관계 메서드 자동 생성 규칙:
──────────────────────────────────────────────
hasMany(Comment)  →  user.getComments()
                     user.addComment(comment)
                     user.addComments([c1, c2])
                     user.removeComment(comment)
                     user.setComments([c1, c2])
                     user.countComments()

belongsTo(User)   →  comment.getUser()
                     comment.setUser(user)

belongsToMany     →  post.getHashtags()
                     post.addHashtag(hashtag)
                     post.addHashtags([h1, h2])
                     post.removeHashtag(hashtag)
──────────────────────────────────────────────
```

---

## SQL vs Sequelize 비교 치트시트

```
SQL                                    Sequelize
────────────────────────────────────────────────────────────────
INSERT INTO users (name, age)          User.create({ name: '홍길동', age: 25 })
VALUES ('홍길동', 25);

SELECT * FROM users;                   User.findAll()

SELECT * FROM users                    User.findAll({
WHERE age > 25;                          where: { age: { [Op.gt]: 25 } }
                                       })

SELECT * FROM users                    User.findOne({ where: { id: 1 } })
WHERE id = 1 LIMIT 1;

SELECT name, email FROM users;         User.findAll({
                                         attributes: ['name', 'email']
                                       })

UPDATE users SET age = 26              User.update({ age: 26 },
WHERE id = 1;                            { where: { id: 1 } })

DELETE FROM users                      User.destroy({ where: { id: 1 } })
WHERE id = 1;

SELECT * FROM users u                  User.findAll({
INNER JOIN comments c                    include: [{ model: Comment }]
ON u.id = c.commenter;                })

SELECT * FROM users                    User.findAll({
ORDER BY age DESC                        order: [['age', 'DESC']],
LIMIT 5 OFFSET 10;                      limit: 5, offset: 10
                                       })
```

---

## Express + Sequelize 전체 프로젝트 구조

### 프로젝트 구조

```
express-sequelize-app/
├── app.js
├── config/
│   └── config.json
├── models/
│   ├── index.js
│   ├── user.js
│   └── comment.js
├── routes/
│   ├── users.js
│   └── comments.js
├── package.json
└── .env
```

### 사용자 라우터

```javascript
// routes/users.js
const express = require('express');
const { User, Comment } = require('../models');
const router = express.Router();

// 전체 사용자 조회
router.get('/', async (req, res, next) => {
  try {
    const users = await User.findAll();
    res.json(users);
  } catch (err) {
    next(err);
  }
});

// 단건 사용자 + 댓글 조회
router.get('/:id', async (req, res, next) => {
  try {
    const user = await User.findOne({
      where: { id: req.params.id },
      include: [{ model: Comment }],
    });
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
    });
    res.status(201).json(user);
  } catch (err) {
    next(err);
  }
});

// 사용자 수정
router.put('/:id', async (req, res, next) => {
  try {
    const [affectedCount] = await User.update({
      name: req.body.name,
      email: req.body.email,
      age: req.body.age,
      married: req.body.married,
    }, {
      where: { id: req.params.id },
    });
    if (affectedCount === 0) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
    res.json({ message: '수정되었습니다.' });
  } catch (err) {
    next(err);
  }
});

// 사용자 삭제
router.delete('/:id', async (req, res, next) => {
  try {
    const deletedCount = await User.destroy({
      where: { id: req.params.id },
    });
    if (deletedCount === 0) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
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
const { User, Comment } = require('../models');
const router = express.Router();

// 전체 댓글 조회 (작성자 포함)
router.get('/', async (req, res, next) => {
  try {
    const comments = await Comment.findAll({
      include: [{
        model: User,
        attributes: ['id', 'name'],
      }],
      order: [['createdAt', 'DESC']],
    });
    res.json(comments);
  } catch (err) {
    next(err);
  }
});

// 특정 사용자의 댓글 조회
router.get('/user/:userId', async (req, res, next) => {
  try {
    const comments = await Comment.findAll({
      where: { commenter: req.params.userId },
      order: [['createdAt', 'DESC']],
    });
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
    res.status(201).json(comment);
  } catch (err) {
    next(err);
  }
});

// 댓글 수정
router.put('/:id', async (req, res, next) => {
  try {
    const [affectedCount] = await Comment.update(
      { comment: req.body.comment },
      { where: { id: req.params.id } }
    );
    if (affectedCount === 0) {
      return res.status(404).json({ error: '댓글을 찾을 수 없습니다.' });
    }
    res.json({ message: '수정되었습니다.' });
  } catch (err) {
    next(err);
  }
});

// 댓글 삭제
router.delete('/:id', async (req, res, next) => {
  try {
    const deletedCount = await Comment.destroy({
      where: { id: req.params.id },
    });
    if (deletedCount === 0) {
      return res.status(404).json({ error: '댓글을 찾을 수 없습니다.' });
    }
    res.json({ message: '삭제되었습니다.' });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

### mysql2 직접 쿼리 vs Sequelize 비교

```javascript
// mysql2 직접 쿼리
const [users] = await pool.query(
  'SELECT u.*, c.comment FROM users u INNER JOIN comments c ON u.id = c.commenter WHERE u.id = ?',
  [1]
);

// Sequelize
const user = await User.findOne({
  where: { id: 1 },
  include: [{ model: Comment }],
});
// → user.Comments 로 관계 데이터에 접근
```

---

## 마이그레이션

마이그레이션은 **테이블 구조 변경을 코드로 관리**하는 기능이다. Git처럼 변경 이력을 추적하고 되돌릴 수 있다.

### 마이그레이션 생성

```bash
$ npx sequelize migration:generate --name add-phone-to-users
```

```javascript
// migrations/20260407-add-phone-to-users.js
module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.addColumn('users', 'phone', {
      type: Sequelize.STRING(20),
      allowNull: true,
    });
  },

  async down(queryInterface) {
    await queryInterface.removeColumn('users', 'phone');
  },
};
```

### 마이그레이션 실행·롤백

```bash
# 미실행 마이그레이션 실행
$ npx sequelize db:migrate

# 마지막 마이그레이션 롤백
$ npx sequelize db:migrate:undo

# 전체 롤백
$ npx sequelize db:migrate:undo:all
```

```
마이그레이션 흐름:
──────────────────────────────────────────────
  up()   — 마이그레이션 실행 (테이블/컬럼 추가·수정)
  down() — 마이그레이션 롤백 (up의 반대 작업)

  db:migrate        →  up() 실행  →  테이블 변경 적용
  db:migrate:undo   →  down() 실행 →  변경 되돌리기

  이력 관리: SequelizeMeta 테이블에 실행된 마이그레이션 기록
──────────────────────────────────────────────
```

---

## 시더 (Seeder) — 초기 데이터

시더는 테스트용 초기 데이터를 삽입하는 기능이다.

```bash
$ npx sequelize seed:generate --name demo-users
```

```javascript
// seeders/demo-users.js
module.exports = {
  async up(queryInterface) {
    await queryInterface.bulkInsert('users', [
      { name: '홍길동', email: 'hong@example.com', age: 25, married: false, createdAt: new Date(), updatedAt: new Date() },
      { name: '김철수', email: 'kim@example.com', age: 30, married: true, createdAt: new Date(), updatedAt: new Date() },
    ]);
  },

  async down(queryInterface) {
    await queryInterface.bulkDelete('users', null, {});
  },
};
```

```bash
# 시더 실행
$ npx sequelize db:seed:all

# 시더 롤백
$ npx sequelize db:seed:undo:all
```

---

## Sequelize CLI 명령어 정리

```
프로젝트 초기화:
  npx sequelize init                     폴더 구조 자동 생성

데이터베이스:
  npx sequelize db:create                DB 생성
  npx sequelize db:drop                  DB 삭제

모델 생성:
  npx sequelize model:generate           모델 + 마이그레이션 동시 생성
    --name User
    --attributes name:string,age:integer

마이그레이션:
  npx sequelize migration:generate       마이그레이션 파일 생성
    --name migration-name
  npx sequelize db:migrate               마이그레이션 실행
  npx sequelize db:migrate:undo          마지막 마이그레이션 롤백
  npx sequelize db:migrate:undo:all      전체 롤백

시더:
  npx sequelize seed:generate            시더 파일 생성
    --name seed-name
  npx sequelize db:seed:all              시더 실행
  npx sequelize db:seed:undo:all         시더 롤백
```

---
