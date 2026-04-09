---
title: 12-03. aws - redis
date: 2026-04-09 00:00:00 +0900
categories: [Node.JS, aws]
tags: [Tech, Node.JS, aws, redis]
pin: true
---

## Redis란?

Redis(Remote Dictionary Server)는 **인메모리 데이터 저장소**이다. 데이터를 디스크가 아닌 메모리(RAM)에 저장하기 때문에 읽기/쓰기가 매우 빠르다. Node.js 서버에서는 주로 **세션 저장소**, **캐싱**, **Rate Limiting** 등에 사용한다.

### 왜 Redis가 필요한가?

PM2 클러스터 모드로 서버를 띄우면 프로세스가 여러 개 생긴다. 이때 Express의 기본 세션 저장소(MemoryStore)를 사용하면 문제가 발생한다:

```
사용자 → 로그인 → 프로세스 A에 세션 저장
사용자 → 다음 요청 → 프로세스 B로 분배 → 세션 없음 → 로그인 풀림!
```

각 프로세스는 독립된 메모리를 가지므로, A에 저장된 세션을 B에서는 찾을 수 없다. Redis를 사용하면 모든 프로세스가 **하나의 외부 세션 저장소를 공유**하여 이 문제를 해결한다.

```
프로세스 A ──┐
프로세스 B ──┼── Redis (세션 공유) ── 세션 데이터
프로세스 C ──┘
```

### Redis vs MemoryStore 비교

| 항목 | MemoryStore (기본) | Redis |
|:---|:---|:---|
| 저장 위치 | 프로세스 메모리 | 외부 Redis 서버 |
| 클러스터 모드 | 세션 공유 불가 | 세션 공유 가능 |
| 서버 재시작 시 | 세션 전부 삭제 | 세션 유지 |
| 속도 | 빠름 | 빠름 (네트워크 비용 약간 있음) |
| 프로덕션 적합 | 부적합 | 적합 |

### 설치

```sh
# Redis 클라이언트 + 세션 연동 패키지
npm i redis connect-redis express-session
```

| 패키지 | 역할 |
|:---|:---|
| `redis` | Node.js에서 Redis 서버에 연결하는 클라이언트 |
| `connect-redis` | Express 세션을 Redis에 저장하는 어댑터 |
| `express-session` | Express 세션 미들웨어 |

### Redis 서버 설치 (로컬 개발용)

```sh
# Ubuntu / AWS EC2
sudo apt update
sudo apt install redis-server

# Redis 서버 시작
sudo systemctl start redis-server

# 부팅 시 자동 시작
sudo systemctl enable redis-server

# 동작 확인
redis-cli ping
# 응답: PONG
```

### Express + Redis 세션 연동

```js
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const app = express();

// Redis 클라이언트 생성 및 연결
const redisClient = createClient({
  url: 'redis://localhost:6379',
});
redisClient.connect().catch(console.error);

redisClient.on('error', (err) => console.error('Redis 연결 에러:', err));
redisClient.on('connect', () => console.log('Redis 연결 성공'));

// 세션 미들웨어 설정
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.COOKIE_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: false,
    maxAge: 1000 * 60 * 60 * 24, // 1일
  },
}));
```

| 옵션 | 설명 |
|:---|:---|
| `store` | 세션 저장소 (RedisStore 지정) |
| `secret` | 세션 암호화 키 |
| `resave` | 변경 없어도 매 요청마다 세션 다시 저장할지 여부 |
| `saveUninitialized` | 초기화되지 않은 세션을 저장할지 여부 |
| `cookie.httpOnly` | JavaScript로 쿠키 접근 차단 |
| `cookie.maxAge` | 세션 유효 기간 (밀리초) |

### Redis 기본 명령어 (redis-cli)

```sh
# Redis CLI 접속
redis-cli

# 키-값 저장
SET name "홍길동"

# 값 조회
GET name
# "홍길동"

# 만료 시간과 함께 저장 (60초 후 자동 삭제)
SETEX token 60 "abc123"

# 키 삭제
DEL name

# 모든 키 조회
KEYS *

# 키 존재 여부 확인
EXISTS name

# 남은 TTL(만료 시간) 확인
TTL token
```

### Redis를 캐싱에 활용하기

DB 쿼리 결과를 Redis에 캐싱하면 반복 요청 시 DB 부하를 줄일 수 있다:

```js
app.get('/posts', async (req, res) => {
  const cacheKey = 'posts:all';

  // 1. Redis에서 캐시 확인
  const cached = await redisClient.get(cacheKey);
  if (cached) {
    return res.json(JSON.parse(cached));
  }

  // 2. 캐시 없으면 DB에서 조회
  const posts = await Post.findAll();

  // 3. Redis에 캐싱 (60초 동안 유지)
  await redisClient.setEx(cacheKey, 60, JSON.stringify(posts));

  res.json(posts);
});
```

흐름:

```
요청 → Redis 캐시 확인
         ├── 캐시 있음 → 바로 응답 (빠름)
         └── 캐시 없음 → DB 조회 → Redis에 저장 → 응답
```

### AWS ElastiCache (프로덕션 Redis)

로컬에 Redis를 설치하면 서버가 꺼지면 Redis도 꺼진다. AWS에서는 **ElastiCache**라는 관리형 Redis 서비스를 사용한다.

| 항목 | 로컬 Redis | AWS ElastiCache |
|:---|:---|:---|
| 관리 | 직접 설치/운영 | AWS가 관리 |
| 고가용성 | 없음 | 자동 장애 복구 |
| 백업 | 수동 | 자동 스냅샷 |
| 확장 | 수동 | 클릭 한 번으로 확장 |
| 비용 | 무료 (EC2 리소스 사용) | 별도 과금 |

ElastiCache 사용 시 연결 URL만 변경하면 된다:

```js
const redisClient = createClient({
  url: 'redis://my-redis-cluster.abc123.ng.0001.apn2.cache.amazonaws.com:6379',
});
```

### Redis 데이터 타입

Redis는 단순한 키-값 저장소가 아니라 다양한 데이터 구조를 지원한다:

| 타입 | 설명 | 활용 예시 |
|:---|:---|:---|
| String | 단순 문자열/숫자 | 세션, 캐시, 토큰 |
| Hash | 필드-값 쌍의 집합 | 사용자 프로필 |
| List | 순서가 있는 문자열 목록 | 최근 활동 로그 |
| Set | 중복 없는 문자열 집합 | 좋아요한 유저 목록 |
| Sorted Set | 점수 기반 정렬 집합 | 랭킹, 리더보드 |

### .env에서 Redis 설정 관리

```
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
```

```js
const redisClient = createClient({
  url: `redis://${process.env.REDIS_HOST}:${process.env.REDIS_PORT}`,
  password: process.env.REDIS_PASSWORD || undefined,
});
```

환경 변수로 관리하면 로컬/개발/프로덕션 환경별로 다른 Redis 서버에 연결할 수 있다.

