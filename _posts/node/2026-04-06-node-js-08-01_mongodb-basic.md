---
title: 08-01. MongoDB란?
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, MongoDB]
tags: [Tech, Node.JS, MongoDB, NoSQL, Mongoose]
pin: true
---

> **MongoDB**는 가장 널리 사용되는 **NoSQL 문서 기반 데이터베이스**다. JSON과 유사한 **BSON(Binary JSON)** 형태로 데이터를 저장하며, 유연한 스키마와 수평 확장이 장점이다.

## NoSQL이란?

NoSQL(Not Only SQL)은 관계형 데이터베이스(RDBMS)의 한계를 보완하기 위해 등장한 데이터베이스 유형이다. 고정된 테이블 구조 대신 **유연한 데이터 모델**을 사용한다.

### NoSQL 분류

| 분류 | 예시 | 특징 |
|------|------|------|
| **문서(Document)** | MongoDB, CouchDB | JSON/BSON 형태의 문서 저장, 유연한 스키마 |
| **키-값(Key-Value)** | Redis, DynamoDB | 단순 키-값 쌍, 캐싱에 적합, 초고속 |
| **열(Column)** | Cassandra, HBase | 열 단위 저장, 대용량 분석에 적합 |
| **그래프(Graph)** | Neo4j, ArangoDB | 노드-엣지 관계, SNS·추천 시스템에 적합 |

---

## MongoDB vs RDBMS 비교

```
RDBMS (MySQL)                     MongoDB
──────────────────────────────────────────────────
Database(데이터베이스)      ←→    Database(데이터베이스)
Table(테이블)               ←→    Collection(컬렉션)
Row(행)                     ←→    Document(문서)
Column(열)                  ←→    Field(필드)
Primary Key                 ←→    _id (자동 생성)
JOIN                        ←→    $lookup (또는 임베딩)
SQL                         ←→    MQL (MongoDB Query Language)
──────────────────────────────────────────────────
```

```
RDBMS 데이터 구조 (테이블):
┌────┬────────┬─────────────────┬─────┐
│ id │ name   │ email           │ age │
├────┼────────┼─────────────────┼─────┤
│  1 │ 홍길동 │ hong@example.com│  25 │
│  2 │ 김철수 │ kim@example.com │  30 │
└────┴────────┴─────────────────┴─────┘

MongoDB 데이터 구조 (문서):
{
  "_id": ObjectId("663a1b2c..."),
  "name": "홍길동",
  "email": "hong@example.com",
  "age": 25,
  "hobbies": ["독서", "운동"],       ← 배열 저장 가능
  "address": {                       ← 중첩 객체 저장 가능
    "city": "서울",
    "zip": "12345"
  }
}
```

### MongoDB 장단점

```
장점:
  - 스키마가 자유롭다 (같은 컬렉션에 서로 다른 구조의 문서 저장 가능)
  - JSON 기반이라 자바스크립트/Node.JS와 궁합이 좋다
  - 수평 확장(샤딩)이 용이하다
  - 배열, 중첩 객체 등 복잡한 데이터를 하나의 문서에 저장 가능
  - 읽기/쓰기 성능이 빠르다

단점:
  - JOIN이 없다 ($lookup으로 대체하지만 RDBMS보다 비효율적)
  - 트랜잭션 지원이 제한적이다 (4.0부터 멀티 문서 트랜잭션 지원)
  - 데이터 중복이 발생할 수 있다
  - 복잡한 관계형 데이터에는 부적합하다
```

---

## MongoDB 설치

### Windows

```
1. https://www.mongodb.com/try/download/community 접속
2. MongoDB Community Server 다운로드 (MSI 설치 파일)
3. Complete 설치 선택
4. "Install MongoDB as a Service" 체크 (자동 실행)
5. MongoDB Compass (GUI 도구) 함께 설치 체크
```

### 설치 확인

```bash
# MongoDB 버전 확인
$ mongod --version

# MongoDB 셸 접속 (mongosh)
$ mongosh
Current Mongosh Log ID: ...
Connecting to: mongodb://127.0.0.1:27017/

test>
```

### MongoDB Compass

MongoDB Compass는 **GUI 기반 관리 도구**로, 데이터를 시각적으로 조회·수정할 수 있다.

```
MongoDB Compass 주요 기능:
  - 데이터베이스·컬렉션 생성/삭제
  - 문서 CRUD (시각적 에디터)
  - 쿼리 작성 및 결과 확인
  - 인덱스 관리
  - 성능 모니터링
```

---

## MongoDB 기본 개념

### Database · Collection · Document

```
MongoDB 구조:
──────────────────────────────────────────────
MongoDB 서버
  └─ Database (nodejs_db)           ← 데이터베이스
       ├─ Collection (users)        ← 컬렉션 (= RDBMS의 테이블)
       │    ├─ Document { ... }     ← 문서 (= RDBMS의 행)
       │    ├─ Document { ... }
       │    └─ Document { ... }
       └─ Collection (comments)
            ├─ Document { ... }
            └─ Document { ... }
──────────────────────────────────────────────
```

### ObjectId

MongoDB는 각 문서에 `_id` 필드를 **자동으로 생성**한다. 이 값은 **ObjectId**라는 12바이트 고유 식별자다.

```
ObjectId 구조 (12바이트 = 24자리 16진수):
──────────────────────────────────────────────
  663a1b2c   3f1a   2b   001
  ────────   ────   ──   ───
  타임스탬프  랜덤   랜덤  카운터
  (4바이트)  (5바이트)     (3바이트)

예: ObjectId("663a1b2c3f1a2b001c0d1234")
  - 생성 시각이 포함되어 있어 정렬·시간 추출 가능
  - 전 세계적으로 고유한 값
──────────────────────────────────────────────
```

### BSON 데이터 타입

```
MongoDB 주요 데이터 타입:
  String         문자열 ("hello")
  Number         숫자 (정수, 실수)
  Boolean        논리값 (true / false)
  Date           날짜 (ISODate("2026-04-07T00:00:00Z"))
  ObjectId       고유 식별자
  Array          배열 (["a", "b", "c"])
  Object         중첩 객체 ({ key: value })
  Null           널 값
  Int32 / Int64  32비트 / 64비트 정수
  Double         64비트 부동소수점
  Decimal128     128비트 고정밀 소수 (금액에 적합)
```

---

## mongosh — MongoDB 셸 명령어

### 데이터베이스 관리

```javascript
// 현재 데이터베이스 확인
> db

// 데이터베이스 목록 조회
> show dbs

// 데이터베이스 선택 (없으면 자동 생성)
> use nodejs_db

// 현재 데이터베이스 삭제
> db.dropDatabase()

// 컬렉션 목록 조회
> show collections
```

```
데이터베이스 자동 생성:
──────────────────────────────────────────────
  use nodejs_db  →  데이터베이스가 없어도 에러 없음
                    첫 번째 문서를 삽입할 때 자동으로 생성됨
  
  컬렉션도 마찬가지:
  db.users.insertOne({})  →  users 컬렉션이 없으면 자동 생성
──────────────────────────────────────────────
```

### 컬렉션 관리

```javascript
// 컬렉션 생성
> db.createCollection('users')

// 컬렉션 삭제
> db.users.drop()

// 컬렉션 목록 조회
> show collections
```

---

## CRUD — 문서 조작

### Create (삽입)

```javascript
// 단건 삽입
> db.users.insertOne({
    name: '홍길동',
    email: 'hong@example.com',
    age: 25,
    married: false,
    hobbies: ['독서', '운동'],
    createdAt: new Date()
  })
// 결과: { acknowledged: true, insertedId: ObjectId("...") }

// 다건 삽입
> db.users.insertMany([
    {
      name: '김철수',
      email: 'kim@example.com',
      age: 30,
      married: true,
      hobbies: ['게임'],
      createdAt: new Date()
    },
    {
      name: '이영희',
      email: 'lee@example.com',
      age: 28,
      married: false,
      hobbies: ['음악', '여행'],
      createdAt: new Date()
    }
  ])
```

### Read (조회)

```javascript
// 전체 조회
> db.users.find()

// 보기 좋게 출력
> db.users.find().pretty()

// 조건 조회
> db.users.find({ age: 25 })
> db.users.find({ married: true })

// 특정 필드만 조회 (프로젝션)
> db.users.find({}, { name: 1, email: 1, _id: 0 })
// 1 = 포함, 0 = 제외

// 단건 조회
> db.users.findOne({ name: '홍길동' })

// 비교 연산자
> db.users.find({ age: { $gt: 25 } })         // age > 25
> db.users.find({ age: { $gte: 25 } })        // age >= 25
> db.users.find({ age: { $lt: 30 } })         // age < 30
> db.users.find({ age: { $lte: 30 } })        // age <= 30
> db.users.find({ age: { $ne: 25 } })         // age != 25

// 논리 연산자
> db.users.find({ $and: [{ age: { $gte: 25 } }, { married: false }] })
> db.users.find({ $or: [{ age: 25 }, { age: 30 }] })

// 정렬
> db.users.find().sort({ age: 1 })             // 오름차순
> db.users.find().sort({ age: -1 })            // 내림차순
> db.users.find().sort({ age: -1, name: 1 })   // 다중 정렬

// 제한 (페이지네이션)
> db.users.find().limit(5)                     // 상위 5건
> db.users.find().skip(10).limit(5)            // 11번째부터 5건

// 개수
> db.users.countDocuments()
> db.users.countDocuments({ married: true })

// 배열 필드 검색
> db.users.find({ hobbies: '독서' })            // 배열에 '독서' 포함
> db.users.find({ hobbies: { $in: ['독서', '게임'] } })   // 하나라도 포함
> db.users.find({ hobbies: { $all: ['독서', '운동'] } })   // 모두 포함

// 정규식 검색 (LIKE 대체)
> db.users.find({ name: /^김/ })               // '김'으로 시작
> db.users.find({ email: /gmail/ })            // 'gmail' 포함
```

### 쿼리 연산자 정리

```
비교 연산자:
  $eq      같다 (=)               { age: { $eq: 25 } }
  $ne      같지 않다 (!=)         { age: { $ne: 25 } }
  $gt      초과 (>)               { age: { $gt: 25 } }
  $gte     이상 (>=)              { age: { $gte: 25 } }
  $lt      미만 (<)               { age: { $lt: 30 } }
  $lte     이하 (<=)              { age: { $lte: 30 } }
  $in      목록 내 포함           { age: { $in: [25, 28, 30] } }
  $nin     목록 내 미포함         { age: { $nin: [25, 30] } }

논리 연산자:
  $and     모두 만족              { $and: [조건1, 조건2] }
  $or      하나라도 만족          { $or: [조건1, 조건2] }
  $not     조건 부정              { age: { $not: { $gt: 30 } } }
  $nor     모두 불만족            { $nor: [조건1, 조건2] }

요소 연산자:
  $exists  필드 존재 여부         { phone: { $exists: true } }
  $type    필드 타입 확인         { age: { $type: 'number' } }

배열 연산자:
  $in      하나라도 포함          { hobbies: { $in: ['독서'] } }
  $all     모두 포함              { hobbies: { $all: ['독서', '운동'] } }
  $size    배열 크기              { hobbies: { $size: 2 } }

평가 연산자:
  $regex   정규식 매칭            { name: { $regex: /^김/ } }
```

### Update (수정)

```javascript
// 단건 수정
> db.users.updateOne(
    { name: '홍길동' },                 // 조건
    { $set: { age: 26 } }              // 수정 내용
  )

// 다건 수정
> db.users.updateMany(
    { married: false },                 // 조건
    { $set: { status: 'single' } }     // 수정 내용
  )

// 여러 필드 수정
> db.users.updateOne(
    { name: '홍길동' },
    { $set: { age: 26, married: true } }
  )

// 숫자 증가/감소
> db.users.updateOne(
    { name: '홍길동' },
    { $inc: { age: 1 } }               // age를 1 증가
  )

// 필드 삭제
> db.users.updateOne(
    { name: '홍길동' },
    { $unset: { status: '' } }         // status 필드 제거
  )

// 배열에 요소 추가
> db.users.updateOne(
    { name: '홍길동' },
    { $push: { hobbies: '코딩' } }     // hobbies 배열에 '코딩' 추가
  )

// 배열에서 요소 제거
> db.users.updateOne(
    { name: '홍길동' },
    { $pull: { hobbies: '코딩' } }     // hobbies 배열에서 '코딩' 제거
  )

// 문서 교체 (주의: 전체 문서가 교체됨)
> db.users.replaceOne(
    { name: '홍길동' },
    { name: '홍길동', email: 'new@example.com', age: 27 }
  )
```

```
수정 연산자:
  $set     필드 값 설정/변경      { $set: { age: 26 } }
  $unset   필드 삭제             { $unset: { status: '' } }
  $inc     숫자 증가/감소        { $inc: { age: 1 } }  (음수면 감소)
  $min     현재 값보다 작으면 변경 { $min: { age: 20 } }
  $max     현재 값보다 크면 변경  { $max: { age: 40 } }
  $mul     곱하기                { $mul: { score: 1.5 } }
  $rename  필드 이름 변경        { $rename: { nm: 'name' } }
  $push    배열에 요소 추가      { $push: { hobbies: '코딩' } }
  $pull    배열에서 요소 제거    { $pull: { hobbies: '코딩' } }
  $addToSet 배열에 중복 없이 추가 { $addToSet: { hobbies: '코딩' } }
  $pop     배열 첫/끝 요소 제거  { $pop: { hobbies: 1 } } (1=끝, -1=첫)
```

### Delete (삭제)

```javascript
// 단건 삭제
> db.users.deleteOne({ name: '홍길동' })

// 다건 삭제
> db.users.deleteMany({ married: false })

// 전체 삭제
> db.users.deleteMany({})
```

---

## Aggregation — 집계 파이프라인

MongoDB의 Aggregation은 **데이터를 단계별(파이프라인)로 처리**하여 집계·변환하는 기능이다. SQL의 GROUP BY, JOIN 등을 대체한다.

```
Aggregation 파이프라인 흐름:
──────────────────────────────────────────────
  컬렉션 → [$match] → [$group] → [$sort] → [$project] → 결과
           (필터)     (그룹화)   (정렬)    (필드 선택)
──────────────────────────────────────────────
```

### 주요 스테이지

```javascript
// $match — 조건 필터 (WHERE)
> db.users.aggregate([
    { $match: { age: { $gte: 25 } } }
  ])

// $group — 그룹화 (GROUP BY)
> db.users.aggregate([
    { $group: {
        _id: '$married',              // 그룹 기준 (married 필드)
        count: { $sum: 1 },           // 개수
        avgAge: { $avg: '$age' }      // 평균 나이
    }}
  ])

// $sort — 정렬
> db.users.aggregate([
    { $group: { _id: '$married', count: { $sum: 1 } } },
    { $sort: { count: -1 } }          // 내림차순
  ])

// $project — 필드 선택/변환 (SELECT)
> db.users.aggregate([
    { $project: {
        name: 1,
        email: 1,
        _id: 0
    }}
  ])

// $limit / $skip — 페이지네이션
> db.users.aggregate([
    { $sort: { age: -1 } },
    { $skip: 10 },
    { $limit: 5 }
  ])

// $lookup — JOIN (다른 컬렉션 결합)
> db.comments.aggregate([
    { $lookup: {
        from: 'users',               // 결합할 컬렉션
        localField: 'commenter',     // comments의 필드
        foreignField: '_id',         // users의 필드
        as: 'userInfo'               // 결과를 담을 필드명
    }}
  ])
```

```
집계 연산자:
  $sum     합계       { $sum: '$price' }  또는  { $sum: 1 } (개수)
  $avg     평균       { $avg: '$age' }
  $min     최소       { $min: '$age' }
  $max     최대       { $max: '$age' }
  $first   첫 번째    { $first: '$name' }
  $last    마지막     { $last: '$name' }
  $count   개수       (스테이지로도 사용)

주요 스테이지:
  $match      조건 필터 (SQL WHERE)
  $group      그룹화 (SQL GROUP BY)
  $sort       정렬 (SQL ORDER BY)
  $project    필드 선택 (SQL SELECT)
  $limit      결과 제한 (SQL LIMIT)
  $skip       건너뛰기 (SQL OFFSET)
  $lookup     JOIN (SQL JOIN)
  $unwind     배열을 개별 문서로 분해
  $count      문서 수 카운트
  $addFields  새 필드 추가
```

---

## 인덱스

인덱스는 쿼리 성능을 높이기 위해 특정 필드에 대한 **검색 색인**을 생성하는 기능이다.

```javascript
// 인덱스 생성
> db.users.createIndex({ email: 1 })            // 오름차순 인덱스
> db.users.createIndex({ age: -1 })             // 내림차순 인덱스

// 유니크 인덱스
> db.users.createIndex({ email: 1 }, { unique: true })

// 복합 인덱스
> db.users.createIndex({ name: 1, age: -1 })

// 인덱스 조회
> db.users.getIndexes()

// 인덱스 삭제
> db.users.dropIndex({ email: 1 })

// 모든 인덱스 삭제 (_id 인덱스 제외)
> db.users.dropIndexes()
```

```
인덱스 특징:
──────────────────────────────────────────────
  장점:
    - 읽기(조회) 속도가 크게 향상된다
    - 정렬 성능도 함께 향상된다

  단점:
    - 쓰기(삽입/수정/삭제) 시 인덱스도 갱신 → 약간 느려짐
    - 저장 공간을 추가로 사용한다

  참고:
    - _id 필드에는 자동으로 유니크 인덱스가 생성된다
    - 자주 조회하는 필드에만 인덱스를 생성해야 한다
──────────────────────────────────────────────
```

---

