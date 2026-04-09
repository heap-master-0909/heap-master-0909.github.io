---
title: 12-02. aws - winston
date: 2026-04-09 00:00:00 +0900
categories: [Node.JS, aws]
tags: [Tech, Node.JS, aws, winston]
pin: true
---

## Winston이란?

Winston은 Node.js에서 가장 널리 사용되는 **로깅 라이브러리**이다. `console.log()`로 남긴 로그는 서버를 재시작하면 사라지고, 로그 레벨 구분도 안 된다. 프로덕션 환경에서는 Winston으로 로그를 **파일에 저장**하고, **레벨별로 분류**하여 관리해야 한다.

### 왜 console.log()로는 부족한가?

| 항목 | `console.log()` | Winston |
|:---|:---|:---|
| 로그 저장 | 콘솔에만 출력 (휘발성) | 파일로 영구 저장 |
| 로그 레벨 | 구분 없음 | error, warn, info, debug 등 구분 |
| 타임스탬프 | 없음 | 자동 기록 |
| 로그 포맷 | 일반 텍스트 | JSON, 커스텀 포맷 가능 |
| 로그 회전 | 불가 | 날짜별/용량별 자동 분리 |
| 운영 환경 | 부적합 | 프로덕션 표준 |

### 설치

```sh
npm i winston
```

### 기본 사용법

```js
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

logger.error('에러 발생!');
logger.warn('경고 메시지');
logger.info('정보 로그');
```

- `level: 'info'` — info 이상 레벨만 기록 (info, warn, error)
- `transports` — 로그를 어디에 출력할지 지정 (파일, 콘솔 등)
- `error.log` — error 레벨 로그만 저장
- `combined.log` — 모든 레벨의 로그 저장

### 로그 레벨

Winston의 기본 로그 레벨은 숫자가 낮을수록 심각도가 높다:

| 레벨 | 숫자 | 용도 |
|:---|:---|:---|
| error | 0 | 에러 발생 시 |
| warn | 1 | 경고 (당장 문제는 아니지만 주의 필요) |
| info | 2 | 일반 정보 (요청 처리 완료 등) |
| http | 3 | HTTP 요청 로그 |
| verbose | 4 | 상세한 정보 |
| debug | 5 | 디버깅용 |
| silly | 6 | 모든 것을 다 기록 |

`level: 'info'`로 설정하면 info(2) 이상, 즉 **error, warn, info**만 기록된다. debug나 silly는 무시된다.

### 개발 환경에서 콘솔 출력 추가

프로덕션에서는 파일에만 저장하고, 개발 환경에서는 콘솔에도 출력하면 편리하다:

```js
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple(),
    ),
  }));
}
```

- `winston.format.colorize()` — 레벨별로 색상 표시 (error는 빨강, warn은 노랑 등)
- `winston.format.simple()` — `info: 서버 시작` 같은 간결한 포맷

### 커스텀 포맷 설정

타임스탬프를 포함한 커스텀 포맷:

```js
const { combine, timestamp, printf } = winston.format;

const logFormat = printf(({ level, message, timestamp }) => {
  return `${timestamp} [${level}]: ${message}`;
});

const logger = winston.createLogger({
  level: 'info',
  format: combine(
    timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    logFormat,
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});
```

출력 예시:

```
2026-04-09 14:30:00 [info]: 서버가 3000번 포트에서 시작되었습니다
2026-04-09 14:30:05 [error]: DB 연결 실패
```

### 로그 회전 (Log Rotation)

로그 파일이 계속 커지면 디스크가 가득 찬다. `winston-daily-rotate-file`로 날짜별로 로그 파일을 분리하고, 오래된 로그를 자동 삭제할 수 있다.

```sh
npm i winston-daily-rotate-file
```

```js
const winston = require('winston');
require('winston-daily-rotate-file');

const transport = new winston.transports.DailyRotateFile({
  filename: 'logs/%DATE%.log',
  datePattern: 'YYYY-MM-DD',
  maxSize: '20m',
  maxFiles: '14d',
});

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    winston.format.json(),
  ),
  transports: [transport],
});
```

| 옵션 | 설명 |
|:---|:---|
| `filename` | 로그 파일 경로 (`%DATE%`가 날짜로 치환) |
| `datePattern` | 날짜 포맷 |
| `maxSize` | 파일 최대 크기 (초과 시 새 파일 생성) |
| `maxFiles` | 보관 기간 (`'14d'`면 14일 후 자동 삭제) |

생성되는 파일 예시:

```
logs/
├── 2026-04-07.log
├── 2026-04-08.log
└── 2026-04-09.log
```

### Express에서 Winston 적용하기

실제 프로젝트에서 logger 모듈을 만들어 사용하는 패턴:

```js
// logger.js
const winston = require('winston');
require('winston-daily-rotate-file');

const logger = winston.createLogger({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  format: winston.format.combine(
    winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    winston.format.json(),
  ),
  transports: [
    new winston.transports.DailyRotateFile({
      filename: 'logs/%DATE%-error.log',
      datePattern: 'YYYY-MM-DD',
      level: 'error',
      maxFiles: '30d',
    }),
    new winston.transports.DailyRotateFile({
      filename: 'logs/%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      maxFiles: '30d',
    }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple(),
    ),
  }));
}

module.exports = logger;
```

```js
// app.js
const logger = require('./logger');
const express = require('express');
const app = express();

app.use((req, res, next) => {
  logger.info(`${req.method} ${req.url}`);
  next();
});

app.get('/', (req, res) => {
  res.send('Hello');
});

app.use((err, req, res, next) => {
  logger.error(err.message);
  res.status(500).send('서버 에러');
});

app.listen(3000, () => {
  logger.info('서버가 3000번 포트에서 시작되었습니다');
});
```

이렇게 하면 `console.log()` 대신 `logger.info()`, `logger.error()`를 사용하여 모든 로그가 파일에 저장되고 레벨별로 분류된다.

### console.log를 Winston으로 교체하기

| 기존 코드 | Winston 코드 |
|:---|:---|
| `console.log('메시지')` | `logger.info('메시지')` |
| `console.error('에러')` | `logger.error('에러')` |
| `console.warn('경고')` | `logger.warn('경고')` |
| `console.debug('디버그')` | `logger.debug('디버그')` |

