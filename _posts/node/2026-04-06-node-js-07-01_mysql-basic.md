---
title: 07. MySQL 이란?
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, MySQL]
tags: [Tech, Node.JS, MySQL]
pin: true
---

> **MySQL**은 가장 널리 사용되는 오픈소스 관계형 데이터베이스 관리 시스템(RDBMS)이다. Node.JS와 결합하면 서버에서 데이터를 영구적으로 저장·조회·수정·삭제할 수 있다.

## 데이터베이스란?

데이터베이스(Database)는 **데이터를 체계적으로 저장·관리하는 시스템**이다. 서버의 메모리(변수, 배열)에 데이터를 저장하면 서버가 재시작될 때 모두 사라진다. 데이터베이스는 이 문제를 해결한다.

```
서버 메모리 vs 데이터베이스
──────────────────────────────────────────────
서버 메모리 (변수):
  - 서버 종료 시 데이터 소멸
  - 검색, 정렬 등 직접 구현 필요
  - 동시 접근 제어 없음

데이터베이스:
  - 서버와 독립적으로 데이터 영구 보존
  - SQL로 강력한 검색·정렬·집계 지원
  - 동시 접근 제어(트랜잭션) 내장
──────────────────────────────────────────────
```

### 데이터베이스 종류

| 분류 | 종류 | 특징 |
|------|------|------|
| **RDBMS** | MySQL, PostgreSQL, Oracle, SQLite | 테이블(행·열) 구조, SQL 사용, 관계(JOIN) 지원 |
| **NoSQL** | MongoDB, Redis, DynamoDB | 유연한 스키마, JSON/키-값 구조, 수평 확장 용이 |

```
RDBMS 데이터 구조 (테이블):
┌────┬────────┬─────────────────┬─────────┐
│ id │ name   │ email           │ age     │
├────┼────────┼─────────────────┼─────────┤
│  1 │ 홍길동 │ hong@example.com│      25 │
│  2 │ 김철수 │ kim@example.com │      30 │
│  3 │ 이영희 │ lee@example.com │      28 │
└────┴────────┴─────────────────┴─────────┘
  행(Row) = 레코드 = 하나의 데이터
  열(Column) = 필드 = 데이터 속성
```

---

## MySQL 설치

### Windows

```
1. https://dev.mysql.com/downloads/installer/ 접속
2. MySQL Installer 다운로드 및 실행
3. Developer Default 선택 후 설치
4. root 비밀번호 설정
5. MySQL Workbench (GUI 도구) 함께 설치됨
```

### 설치 확인

```bash
# MySQL 버전 확인
$ mysql --version

# MySQL 접속
$ mysql -u root -p
Enter password: ****

mysql>
```

---

## SQL 기초

SQL(Structured Query Language)은 관계형 데이터베이스를 다루는 표준 언어다.

### SQL 분류

| 분류 | 명령어 | 설명 |
|------|--------|------|
| **DDL** (Data Definition) | CREATE, ALTER, DROP | 테이블 구조 정의 |
| **DML** (Data Manipulation) | SELECT, INSERT, UPDATE, DELETE | 데이터 조작 |
| **DCL** (Data Control) | GRANT, REVOKE | 권한 관리 |

---

## 데이터베이스·테이블 생성

### 데이터베이스 관리

```sql
-- 데이터베이스 생성
CREATE DATABASE nodejs_db;

-- 데이터베이스 목록 조회
SHOW DATABASES;

-- 사용할 데이터베이스 선택
USE nodejs_db;

-- 데이터베이스 삭제
DROP DATABASE nodejs_db;
```

### MySQL 데이터 타입

```
숫자 타입:
  INT              정수 (-21억 ~ 21억)
  BIGINT           큰 정수
  FLOAT            단정밀도 부동소수점
  DOUBLE           배정밀도 부동소수점
  DECIMAL(M, D)    고정 소수점 (금액 등에 적합)

문자열 타입:
  CHAR(N)          고정 길이 문자열 (최대 255)
  VARCHAR(N)       가변 길이 문자열 (최대 65535)
  TEXT             긴 텍스트 (최대 65535 바이트)
  LONGTEXT         매우 긴 텍스트 (최대 4GB)

날짜/시간 타입:
  DATE             날짜 (YYYY-MM-DD)
  TIME             시간 (HH:MM:SS)
  DATETIME         날짜+시간 (YYYY-MM-DD HH:MM:SS)
  TIMESTAMP        타임스탬프 (시간대 자동 변환)

기타:
  BOOLEAN          논리값 (내부적으로 TINYINT(1))
  ENUM('A','B')    열거형 (지정된 값 중 하나)
  JSON             JSON 데이터
```

### 테이블 생성

```sql
-- 사용자 테이블
CREATE TABLE users (
  id        INT           NOT NULL AUTO_INCREMENT,
  name      VARCHAR(20)   NOT NULL,
  email     VARCHAR(100)  NOT NULL UNIQUE,
  age       INT           UNSIGNED,
  married   BOOLEAN       NOT NULL DEFAULT false,
  comment   TEXT,
  created_at DATETIME     NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

```
컬럼 옵션 설명:
  NOT NULL        — NULL 값 불허 (필수 입력)
  AUTO_INCREMENT  — 자동 증가 (1, 2, 3, ...)
  UNIQUE          — 중복 값 불허
  DEFAULT 값      — 기본값 설정
  UNSIGNED        — 양수만 허용 (음수 범위를 양수로 확장)
  PRIMARY KEY     — 기본키 (행을 고유하게 식별)
```

```sql
-- 댓글 테이블 (외래키로 users 참조)
CREATE TABLE comments (
  id         INT          NOT NULL AUTO_INCREMENT,
  commenter  INT          NOT NULL,
  comment    VARCHAR(500) NOT NULL,
  created_at DATETIME     NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id),
  FOREIGN KEY (commenter) REFERENCES users (id)
    ON DELETE CASCADE
    ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

```
외래키(Foreign Key) 옵션:
  ON DELETE CASCADE   — 참조하는 사용자 삭제 시 댓글도 함께 삭제
  ON UPDATE CASCADE   — 참조하는 사용자 ID 변경 시 댓글의 commenter도 자동 변경
  ON DELETE SET NULL  — 참조하는 사용자 삭제 시 NULL로 설정
  ON DELETE RESTRICT  — 참조하는 데이터가 있으면 삭제 불가 (기본값)
```

```
테이블 관계도:
  users (1) ──── (N) comments
    │                  │
    ├─ id (PK)         ├─ id (PK)
    ├─ name            ├─ commenter (FK → users.id)
    ├─ email           ├─ comment
    ├─ age             └─ created_at
    ├─ married
    ├─ comment
    └─ created_at
```

### 테이블 확인·수정

```sql
-- 테이블 목록 조회
SHOW TABLES;

-- 테이블 구조 확인
DESC users;
-- 또는
DESCRIBE users;

-- 컬럼 추가
ALTER TABLE users ADD phone VARCHAR(15);

-- 컬럼 수정
ALTER TABLE users MODIFY phone VARCHAR(20) NOT NULL;

-- 컬럼 삭제
ALTER TABLE users DROP phone;

-- 테이블 삭제
DROP TABLE comments;
```

---

## CRUD — 데이터 조작

### CREATE (INSERT)

```sql
-- 단건 삽입
INSERT INTO users (name, email, age, married)
VALUES ('홍길동', 'hong@example.com', 25, false);

INSERT INTO users (name, email, age, married)
VALUES ('김철수', 'kim@example.com', 30, true);

-- 다건 삽입
INSERT INTO users (name, email, age, married) VALUES
  ('이영희', 'lee@example.com', 28, false),
  ('박민수', 'park@example.com', 35, true);

-- 댓글 삽입 (commenter는 users.id 참조)
INSERT INTO comments (commenter, comment)
VALUES (1, '안녕하세요, 첫 번째 댓글입니다.');
```

### READ (SELECT)

```sql
-- 전체 조회
SELECT * FROM users;

-- 특정 컬럼만 조회
SELECT name, email FROM users;

-- 조건 조회 (WHERE)
SELECT * FROM users WHERE age >= 28;
SELECT * FROM users WHERE married = true AND age > 25;
SELECT * FROM users WHERE age = 25 OR age = 30;

-- 정렬 (ORDER BY)
SELECT * FROM users ORDER BY age ASC;          -- 오름차순
SELECT * FROM users ORDER BY age DESC;         -- 내림차순
SELECT * FROM users ORDER BY age DESC, name ASC;

-- 제한 (LIMIT, OFFSET)
SELECT * FROM users LIMIT 5;                   -- 상위 5건
SELECT * FROM users LIMIT 5 OFFSET 10;         -- 11번째부터 5건 (페이지네이션)

-- 패턴 검색 (LIKE)
SELECT * FROM users WHERE name LIKE '김%';     -- '김'으로 시작
SELECT * FROM users WHERE email LIKE '%gmail%'; -- 'gmail' 포함

-- 범위 조회
SELECT * FROM users WHERE age BETWEEN 25 AND 30;
SELECT * FROM users WHERE age IN (25, 28, 30);
```

```
WHERE 조건 연산자:
  =, !=, <>, <, >, <=, >=    비교 연산자
  AND, OR, NOT               논리 연산자
  LIKE                       패턴 매칭 (% = 여러 문자, _ = 한 문자)
  BETWEEN A AND B            범위 조건
  IN (값1, 값2, ...)         목록 내 포함 여부
  IS NULL / IS NOT NULL      NULL 체크
```

### UPDATE

```sql
-- 특정 사용자 수정
UPDATE users SET age = 26 WHERE id = 1;

-- 여러 컬럼 수정
UPDATE users SET age = 31, married = true WHERE id = 2;
```

> **주의**: `WHERE` 절 없이 UPDATE를 실행하면 **모든 행**이 수정된다.

### DELETE

```sql
-- 특정 사용자 삭제
DELETE FROM users WHERE id = 4;

-- 조건에 맞는 데이터 삭제
DELETE FROM users WHERE age < 25;
```

> **주의**: `WHERE` 절 없이 DELETE를 실행하면 **모든 행**이 삭제된다.

---

## JOIN — 테이블 결합

JOIN은 여러 테이블의 데이터를 **관계(외래키)**를 기반으로 결합하여 조회하는 기능이다.

### JOIN 종류

```
INNER JOIN — 양쪽 테이블에 모두 있는 데이터만
LEFT JOIN  — 왼쪽 테이블 전체 + 오른쪽 매칭 데이터 (없으면 NULL)
RIGHT JOIN — 오른쪽 테이블 전체 + 왼쪽 매칭 데이터 (없으면 NULL)

  users                    comments
┌────┬────────┐        ┌────┬───────────┬────────┐
│ id │ name   │        │ id │ commenter │ comment│
├────┼────────┤        ├────┼───────────┼────────┤
│  1 │ 홍길동 │        │  1 │     1     │ 안녕   │
│  2 │ 김철수 │        │  2 │     1     │ 반갑   │
│  3 │ 이영희 │        │  3 │     2     │ Hi     │
└────┴────────┘        └────┴───────────┴────────┘

INNER JOIN 결과: 홍길동-안녕, 홍길동-반갑, 김철수-Hi (이영희 제외)
LEFT JOIN 결과:  홍길동-안녕, 홍길동-반갑, 김철수-Hi, 이영희-NULL
```

### JOIN 쿼리

```sql
-- INNER JOIN: 댓글과 작성자 정보 함께 조회
SELECT c.id, c.comment, c.created_at, u.name AS commenter_name, u.email
FROM comments AS c
INNER JOIN users AS u ON c.commenter = u.id;

-- LEFT JOIN: 모든 사용자 + 댓글 (댓글 없는 사용자도 포함)
SELECT u.name, c.comment
FROM users AS u
LEFT JOIN comments AS c ON u.id = c.commenter;

-- 특정 사용자의 댓글만 JOIN으로 조회
SELECT u.name, c.comment, c.created_at
FROM comments AS c
INNER JOIN users AS u ON c.commenter = u.id
WHERE u.id = 1
ORDER BY c.created_at DESC;
```

---

## 집계 함수와 GROUP BY

```sql
-- 집계 함수
SELECT COUNT(*) AS total_users FROM users;
SELECT AVG(age) AS avg_age FROM users;
SELECT MAX(age) AS max_age, MIN(age) AS min_age FROM users;
SELECT SUM(age) AS total_age FROM users;

-- GROUP BY: 그룹별 집계
SELECT married, COUNT(*) AS count, AVG(age) AS avg_age
FROM users
GROUP BY married;

-- HAVING: 그룹 조건 필터
SELECT commenter, COUNT(*) AS comment_count
FROM comments
GROUP BY commenter
HAVING comment_count >= 2;
```

```
집계 함수 정리:
  COUNT(*)    — 행 수
  SUM(컬럼)   — 합계
  AVG(컬럼)   — 평균
  MAX(컬럼)   — 최대값
  MIN(컬럼)   — 최소값

WHERE vs HAVING:
  WHERE   — 그룹화 전에 개별 행을 필터
  HAVING  — 그룹화 후에 그룹을 필터
```

---

