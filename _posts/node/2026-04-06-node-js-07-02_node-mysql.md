---
title: 07-02. Node에서의 MySQL
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, MySQL]
tags: [Tech, Node.JS, MySQL]
pin: true
---

## Node.JS에서 MySQL 사용

### mysql2 패키지

Node.JS에서 MySQL에 연결하려면 `mysql2` 패키지를 사용한다.

```bash
$ npm i mysql2
```

### 커넥션 생성 및 쿼리

```javascript
const mysql = require('mysql2');

const connection = mysql.createConnection({
  host: 'localhost',
  user: 'root',
  password: 'your_password',
  database: 'nodejs_db',
});

connection.connect((err) => {
  if (err) {
    console.error('MySQL 연결 실패:', err);
    return;
  }
  console.log('MySQL 연결 성공');
});

connection.query('SELECT * FROM users', (err, results) => {
  if (err) throw err;
  console.log(results);
});

connection.end();
```

### 커넥션 풀 (Connection Pool)

매 요청마다 커넥션을 생성·해제하면 성능이 저하된다. **커넥션 풀**은 미리 커넥션을 만들어두고 재사용한다.

```javascript
const mysql = require('mysql2');

const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'your_password',
  database: 'nodejs_db',
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
});

pool.query('SELECT * FROM users', (err, results) => {
  if (err) throw err;
  console.log(results);
});
```

```
커넥션 vs 커넥션 풀:
──────────────────────────────────────────────
커넥션:
  요청 → 커넥션 생성 → 쿼리 실행 → 커넥션 해제
  요청 → 커넥션 생성 → 쿼리 실행 → 커넥션 해제
  → 매번 생성/해제 비용 발생

커넥션 풀:
  서버 시작 → 커넥션 10개 미리 생성
  요청 → 풀에서 커넥션 대여 → 쿼리 실행 → 풀에 반납
  요청 → 풀에서 커넥션 대여 → 쿼리 실행 → 풀에 반납
  → 생성/해제 비용 없이 재사용
──────────────────────────────────────────────
```

### Promise 기반 사용 (async/await)

`mysql2`는 Promise 래퍼를 제공하여 `async/await`과 함께 사용할 수 있다.

```javascript
const mysql = require('mysql2/promise');

async function main() {
  const pool = mysql.createPool({
    host: 'localhost',
    user: 'root',
    password: 'your_password',
    database: 'nodejs_db',
  });

  const [rows] = await pool.query('SELECT * FROM users');
  console.log(rows);

  const [result] = await pool.query(
    'INSERT INTO users (name, email, age, married) VALUES (?, ?, ?, ?)',
    ['홍길동', 'hong@example.com', 25, false]
  );
  console.log('삽입된 ID:', result.insertId);
}

main().catch(console.error);
```

> `pool.query()`의 반환값은 `[rows, fields]` 배열이다. 구조 분해로 `rows`만 꺼내 사용한다.

### Prepared Statement (SQL 인젝션 방지)

사용자 입력을 SQL에 직접 삽입하면 **SQL 인젝션** 공격에 취약하다. `?` 플레이스홀더를 사용한다.

```javascript
// 위험: SQL 인젝션 가능
const name = "'; DROP TABLE users; --";
pool.query(`SELECT * FROM users WHERE name = '${name}'`);

// 안전: Prepared Statement
pool.query('SELECT * FROM users WHERE name = ?', [name]);
pool.query(
  'INSERT INTO users (name, email, age) VALUES (?, ?, ?)',
  ['홍길동', 'hong@example.com', 25]
);
pool.query('UPDATE users SET age = ? WHERE id = ?', [26, 1]);
pool.query('DELETE FROM users WHERE id = ?', [3]);
```

```
SQL 인젝션 공격 예시:
──────────────────────────────────────────────
사용자 입력:  ' OR 1=1 --
위험한 쿼리:  SELECT * FROM users WHERE name = '' OR 1=1 --'
결과:         모든 사용자 데이터 유출!

Prepared Statement 사용:
안전한 쿼리:  SELECT * FROM users WHERE name = ?
값 바인딩:    [" ' OR 1=1 --"]
결과:         해당 이름의 사용자만 조회 (특수문자가 이스케이프됨)
──────────────────────────────────────────────
```

---

## Express + MySQL 연동

### 프로젝트 구조

```
express-mysql-app/
├── app.js
├── database.js
├── routes/
│   ├── users.js
│   └── comments.js
├── package.json
└── .env
```

### 데이터베이스 설정 모듈

```javascript
// database.js
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST || 'localhost',
  user: process.env.DB_USER || 'root',
  password: process.env.DB_PASSWORD || '',
  database: process.env.DB_NAME || 'nodejs_db',
  waitForConnections: true,
  connectionLimit: 10,
});

module.exports = pool;
```

### 메인 서버

```javascript
// app.js
const express = require('express');
const morgan = require('morgan');
const app = express();

app.use(morgan('dev'));
app.use(express.json());

const usersRouter = require('./routes/users');
const commentsRouter = require('./routes/comments');

app.use('/api/users', usersRouter);
app.use('/api/comments', commentsRouter);

app.use((req, res) => {
  res.status(404).json({ error: '요청한 리소스를 찾을 수 없습니다.' });
});

app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: '서버 내부 오류가 발생했습니다.' });
});

app.listen(3000, () => {
  console.log('http://localhost:3000');
});
```

### 사용자 라우터

```javascript
// routes/users.js
const express = require('express');
const pool = require('../database');
const router = express.Router();

// 전체 사용자 조회
router.get('/', async (req, res, next) => {
  try {
    const [users] = await pool.query('SELECT * FROM users ORDER BY id ASC');
    res.json(users);
  } catch (err) {
    next(err);
  }
});

// 단건 사용자 조회
router.get('/:id', async (req, res, next) => {
  try {
    const [users] = await pool.query('SELECT * FROM users WHERE id = ?', [req.params.id]);
    if (users.length === 0) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
    res.json(users[0]);
  } catch (err) {
    next(err);
  }
});

// 사용자 생성
router.post('/', async (req, res, next) => {
  try {
    const { name, email, age, married } = req.body;
    const [result] = await pool.query(
      'INSERT INTO users (name, email, age, married) VALUES (?, ?, ?, ?)',
      [name, email, age, married]
    );
    res.status(201).json({ id: result.insertId, name, email, age, married });
  } catch (err) {
    next(err);
  }
});

// 사용자 수정
router.put('/:id', async (req, res, next) => {
  try {
    const { name, email, age, married } = req.body;
    const [result] = await pool.query(
      'UPDATE users SET name = ?, email = ?, age = ?, married = ? WHERE id = ?',
      [name, email, age, married, req.params.id]
    );
    if (result.affectedRows === 0) {
      return res.status(404).json({ error: '사용자를 찾을 수 없습니다.' });
    }
    res.json({ id: parseInt(req.params.id), name, email, age, married });
  } catch (err) {
    next(err);
  }
});

// 사용자 삭제
router.delete('/:id', async (req, res, next) => {
  try {
    const [result] = await pool.query('DELETE FROM users WHERE id = ?', [req.params.id]);
    if (result.affectedRows === 0) {
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
const pool = require('../database');
const router = express.Router();

// 전체 댓글 조회 (작성자 이름 포함)
router.get('/', async (req, res, next) => {
  try {
    const [comments] = await pool.query(`
      SELECT c.id, c.comment, c.created_at, u.name AS commenter_name
      FROM comments AS c
      INNER JOIN users AS u ON c.commenter = u.id
      ORDER BY c.created_at DESC
    `);
    res.json(comments);
  } catch (err) {
    next(err);
  }
});

// 특정 사용자의 댓글 조회
router.get('/user/:userId', async (req, res, next) => {
  try {
    const [comments] = await pool.query(
      'SELECT * FROM comments WHERE commenter = ? ORDER BY created_at DESC',
      [req.params.userId]
    );
    res.json(comments);
  } catch (err) {
    next(err);
  }
});

// 댓글 생성
router.post('/', async (req, res, next) => {
  try {
    const { commenter, comment } = req.body;
    const [result] = await pool.query(
      'INSERT INTO comments (commenter, comment) VALUES (?, ?)',
      [commenter, comment]
    );
    res.status(201).json({ id: result.insertId, commenter, comment });
  } catch (err) {
    next(err);
  }
});

// 댓글 수정
router.put('/:id', async (req, res, next) => {
  try {
    const { comment } = req.body;
    const [result] = await pool.query(
      'UPDATE comments SET comment = ? WHERE id = ?',
      [comment, req.params.id]
    );
    if (result.affectedRows === 0) {
      return res.status(404).json({ error: '댓글을 찾을 수 없습니다.' });
    }
    res.json({ id: parseInt(req.params.id), comment });
  } catch (err) {
    next(err);
  }
});

// 댓글 삭제
router.delete('/:id', async (req, res, next) => {
  try {
    const [result] = await pool.query('DELETE FROM comments WHERE id = ?', [req.params.id]);
    if (result.affectedRows === 0) {
      return res.status(404).json({ error: '댓글을 찾을 수 없습니다.' });
    }
    res.json({ message: '삭제되었습니다.' });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

### API 테스트

```bash
# 사용자 생성
$ curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"홍길동","email":"hong@example.com","age":25,"married":false}'

# 전체 사용자 조회
$ curl http://localhost:3000/api/users

# 댓글 생성
$ curl -X POST http://localhost:3000/api/comments \
  -H "Content-Type: application/json" \
  -d '{"commenter":1,"comment":"안녕하세요!"}'

# 댓글 조회 (JOIN 결과)
$ curl http://localhost:3000/api/comments
```

---

## 트랜잭션

트랜잭션은 **여러 쿼리를 하나의 작업 단위**로 묶는 것이다. 모두 성공하면 반영(COMMIT), 하나라도 실패하면 전부 취소(ROLLBACK)한다.

```javascript
async function transfer(fromId, toId, amount) {
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();

    await conn.query('UPDATE accounts SET balance = balance - ? WHERE id = ?', [amount, fromId]);
    await conn.query('UPDATE accounts SET balance = balance + ? WHERE id = ?', [amount, toId]);

    await conn.commit();
    console.log('송금 완료');
  } catch (err) {
    await conn.rollback();
    console.error('송금 실패, 롤백:', err.message);
    throw err;
  } finally {
    conn.release();
  }
}
```

```
트랜잭션의 ACID 속성:
  A (Atomicity, 원자성)    — 전부 성공하거나 전부 실패
  C (Consistency, 일관성)  — 트랜잭션 전후로 데이터 정합성 유지
  I (Isolation, 격리성)    — 동시 실행 트랜잭션 간 간섭 없음
  D (Durability, 지속성)   — 커밋된 데이터는 영구 보존

트랜잭션 흐름:
  BEGIN → 쿼리1 → 쿼리2 → 쿼리3
    │                         │
    │  에러 발생 시 ROLLBACK   │  모두 성공 시 COMMIT
    │  (전부 취소)             │  (전부 반영)
```

> 트랜잭션에서는 `pool.query()` 대신 `conn.query()`를 사용해야 한다. `pool.getConnection()`으로 커넥션을 직접 가져와서 같은 커넥션 내에서 모든 쿼리를 실행해야 트랜잭션이 보장된다.

---

## SQL 정리 치트시트

```
데이터베이스:
  CREATE DATABASE db_name;
  USE db_name;
  DROP DATABASE db_name;

테이블:
  CREATE TABLE table_name (컬럼 정의);
  DESC table_name;
  ALTER TABLE table_name ADD/MODIFY/DROP 컬럼;
  DROP TABLE table_name;

데이터 조작:
  INSERT INTO table (col1, col2) VALUES (val1, val2);
  SELECT col FROM table WHERE 조건 ORDER BY col LIMIT n;
  UPDATE table SET col = val WHERE 조건;
  DELETE FROM table WHERE 조건;

조인:
  SELECT * FROM A INNER JOIN B ON A.id = B.a_id;
  SELECT * FROM A LEFT JOIN B ON A.id = B.a_id;

집계:
  SELECT col, COUNT(*) FROM table GROUP BY col HAVING 조건;

Node.JS에서:
  mysql2/promise → pool.query('SELECT * FROM users WHERE id = ?', [id])
  트랜잭션 → conn.beginTransaction() → conn.commit() / conn.rollback()
```
