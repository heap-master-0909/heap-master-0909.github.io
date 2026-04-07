---
title: 04-05. Cluster
date: 2026-04-07 00:00:00 +0900
categories: [Node.JS, HTTP-SERVER, Cluster]
tags: [Tech, Node.JS, HTTP-SERVER, Cluster]
pin: true
---

> Node.JS는 기본적으로 **싱글 스레드**로 동작한다. CPU 코어가 8개인 서버에서도 1개의 코어만 사용한다면 자원 낭비다. `cluster` 모듈을 사용하면 **여러 프로세스**를 생성하여 모든 CPU 코어를 활용할 수 있다.

## 왜 Cluster가 필요한가?

Node.JS의 싱글 스레드 모델은 I/O 처리에는 효율적이지만, 두 가지 한계가 있다.

```
┌── 싱글 스레드의 한계 ──────────────────────────┐
│                                                │
│  1. CPU 활용 부족                               │
│     8코어 서버에서 1코어만 사용 → 87.5% 유휴     │
│                                                │
│  2. 장애 대응 불가                               │
│     프로세스 1개가 죽으면 서비스 전체가 중단       │
│                                                │
└────────────────────────────────────────────────┘
```

```
[싱글 프로세스]
                     CPU 코어
요청 ──→ Node.JS ──→ [■ 사용] [□ 유휴] [□ 유휴] [□ 유휴]
                     [□ 유휴] [□ 유휴] [□ 유휴] [□ 유휴]

[Cluster 사용]
                     CPU 코어
요청 ──→ Master ──→  [■ Worker 1] [■ Worker 2] [■ Worker 3] [■ Worker 4]
              │      [■ Worker 5] [■ Worker 6] [■ Worker 7] [■ Worker 8]
              │
              └─→ 요청을 Worker들에게 분배
```

---

## Cluster 모듈 기본 개념

### Master와 Worker

`cluster` 모듈은 **Master-Worker 패턴**으로 동작한다.

```
┌───── Master 프로세스 ─────────────────────────┐
│                                               │
│  - 워커 프로세스를 생성(fork)하고 관리          │
│  - 클라이언트 요청을 받아 워커에게 분배          │
│  - 직접 요청을 처리하지 않음                    │
│                                               │
│      fork()    fork()    fork()    fork()      │
│        │         │         │         │         │
│        ▼         ▼         ▼         ▼         │
│    Worker 1  Worker 2  Worker 3  Worker 4      │
│    (PID: 101)(PID: 102)(PID: 103)(PID: 104)    │
│                                               │
│  각 Worker는 독립된 프로세스로,                  │
│  자체 이벤트 루프와 메모리 공간을 가진다          │
└───────────────────────────────────────────────┘
```

| 구분 | Master | Worker |
|------|--------|--------|
| **역할** | 워커 생성·관리, 요청 분배 | 실제 요청 처리 |
| **개수** | 1개 | 보통 CPU 코어 수만큼 |
| **요청 처리** | 직접 처리하지 않음 | 서버 로직 실행 |
| **`cluster.isMaster`** | `true` | `false` |
| **`cluster.isWorker`** | `false` | `true` |

> Node.JS v16.0.0부터 `cluster.isMaster`는 `cluster.isPrimary`로 이름이 변경되었다. 기능은 동일하며, 이전 이름도 하위 호환을 위해 유지된다.

---

## 기본 사용법

### 최소 Cluster 코드

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  console.log(`Master 프로세스 ${process.pid} 실행`);
  console.log(`CPU 코어 수: ${numCPUs}`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} 종료 (code: ${code}, signal: ${signal})`);
  });
} else {
  http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`<h1>Worker ${process.pid}가 응답합니다</h1>`);
  }).listen(3000);

  console.log(`Worker 프로세스 ${process.pid} 실행`);
}
```

```bash
$ node server.js
Master 프로세스 12345 실행
CPU 코어 수: 8
Worker 프로세스 12346 실행
Worker 프로세스 12347 실행
Worker 프로세스 12348 실행
Worker 프로세스 12349 실행
Worker 프로세스 12350 실행
Worker 프로세스 12351 실행
Worker 프로세스 12352 실행
Worker 프로세스 12353 실행
```

### 코드 흐름 분석

```
node server.js 실행
  │
  ├─ cluster.isPrimary === true (Master)
  │    │
  │    ├─ os.cpus().length 만큼 cluster.fork() 호출
  │    │    → 현재 파일(server.js)을 새 프로세스로 실행
  │    │
  │    └─ cluster.on('exit') 이벤트 등록
  │         → 워커가 종료되면 알림
  │
  └─ cluster.isPrimary === false (Worker)
       │
       └─ http.createServer() 로 서버 생성
            → 같은 포트(3000)를 공유
```

> 모든 Worker가 **같은 포트**를 `listen`할 수 있는 이유는 Master가 내부적으로 포트를 점유하고, 들어오는 연결을 Worker들에게 분배하기 때문이다.

---

## 로드 밸런싱

### Round-Robin 방식

Node.JS Cluster는 기본적으로 **Round-Robin** 방식으로 요청을 분배한다 (Windows 제외).

```
요청 순서:  1번  2번  3번  4번  5번  6번  7번  8번  9번

분배:      W1   W2   W3   W4   W1   W2   W3   W4   W1
           ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
         순서대로 돌아가며 배분 ─────────────────────→
```

| 스케줄링 정책 | 설명 | 환경 변수 값 |
|--------------|------|-------------|
| **Round-Robin** | 요청을 순서대로 워커에 배분 (기본값) | `cluster.SCHED_RR` |
| **OS 위임** | 운영체제가 직접 분배 (Windows 기본) | `cluster.SCHED_NONE` |

```javascript
// 스케줄링 정책을 명시적으로 설정
cluster.schedulingPolicy = cluster.SCHED_RR;
```

또는 환경 변수로 설정할 수 있다.

```bash
# Round-Robin 강제
$ NODE_CLUSTER_SCHED_POLICY=rr node server.js

# OS에 위임
$ NODE_CLUSTER_SCHED_POLICY=none node server.js
```

---

## 워커 자동 재시작

Cluster의 핵심 장점 중 하나는 **워커가 죽어도 서비스가 중단되지 않는다**는 것이다. Master에서 종료된 워커를 감지하고 새 워커를 생성하면 된다.

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  console.log(`Master ${process.pid} 실행`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} 종료됨. 새 워커 생성...`);
    cluster.fork();
  });
} else {
  http.createServer((req, res) => {
    if (req.url === '/crash') {
      process.exit(1);
    }

    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`<h1>Worker ${process.pid} 정상 응답</h1>`);
  }).listen(3000);

  console.log(`Worker ${process.pid} 실행`);
}
```

```
[워커 자동 재시작 흐름]

시간 →
───────────────────────────────────────────→
  Worker 1  ████████████ 죽음 ─→ 새 Worker 5 ████████
  Worker 2  ████████████████████████████████████████
  Worker 3  ████████████████████████████████████████
  Worker 4  ████████████████████████████████████████
                          ↑
                    Master가 감지하고
                    즉시 새 워커 생성
                    → 서비스 무중단!
```

```bash
# 정상 요청
$ curl http://localhost:3000
Worker 12346 정상 응답

# 워커 강제 종료
$ curl http://localhost:3000/crash
# → 서버 로그: Worker 12346 종료됨. 새 워커 생성...

# 다른 워커가 즉시 응답
$ curl http://localhost:3000
Worker 12354 정상 응답
```

---

## Master-Worker 간 통신 (IPC)

Master와 Worker는 **IPC(Inter-Process Communication)** 채널을 통해 메시지를 주고받을 수 있다.

```
┌─── Master ──────────────────────────────┐
│                                         │
│  worker.send({ type: 'task', data: … }) │
│            │                            │
│            ▼      IPC 채널              │
│  worker.on('message', (msg) => { … })   │
│            ▲                            │
│            │                            │
└────────────┼────────────────────────────┘
             │
┌─── Worker ─┼────────────────────────────┐
│            │                            │
│  process.on('message', (msg) => { … })  │
│            │                            │
│  process.send({ type: 'result', … })    │
│                                         │
└─────────────────────────────────────────┘
```

```javascript
const cluster = require('cluster');

if (cluster.isPrimary) {
  const worker = cluster.fork();

  worker.send({ type: 'greeting', message: '안녕, Worker!' });

  worker.on('message', (msg) => {
    console.log(`Master가 Worker로부터 받은 메시지:`, msg);
  });
} else {
  process.on('message', (msg) => {
    console.log(`Worker ${process.pid}가 받은 메시지:`, msg);

    process.send({ type: 'reply', message: '안녕, Master!' });
  });
}
```

```bash
$ node ipc.js
Worker 12346가 받은 메시지: { type: 'greeting', message: '안녕, Worker!' }
Master가 Worker로부터 받은 메시지: { type: 'reply', message: '안녕, Master!' }
```

### 활용 예시: 전체 워커에 설정 브로드캐스트

```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) {
    cluster.fork();
  }

  setTimeout(() => {
    const config = { maxRequestsPerMinute: 100 };
    for (const id in cluster.workers) {
      cluster.workers[id].send({ type: 'config', data: config });
    }
    console.log('모든 워커에 설정 전파 완료');
  }, 1000);
} else {
  let config = {};

  process.on('message', (msg) => {
    if (msg.type === 'config') {
      config = msg.data;
      console.log(`Worker ${process.pid} 설정 업데이트:`, config);
    }
  });
}
```

> Worker 간에는 **직접 통신이 불가능**하다. Worker끼리 데이터를 주고받으려면 Master를 경유해야 한다.

---

## 무중단 재시작 (Zero-Downtime Restart)

코드를 배포할 때 서비스를 중단하지 않고 워커를 순차적으로 재시작할 수 있다.

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  console.log(`Master ${process.pid} 실행`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  process.on('SIGUSR2', () => {
    const workers = Object.values(cluster.workers);
    console.log('무중단 재시작 시작...');

    function restartWorker(index) {
      if (index >= workers.length) {
        console.log('모든 워커 재시작 완료');
        return;
      }

      const worker = workers[index];
      console.log(`Worker ${worker.process.pid} 재시작 중...`);

      const newWorker = cluster.fork();

      newWorker.on('listening', () => {
        worker.disconnect();
        worker.on('disconnect', () => {
          console.log(`이전 Worker ${worker.process.pid} 종료`);
          restartWorker(index + 1);
        });
      });
    }

    restartWorker(0);
  });

  cluster.on('exit', (worker, code) => {
    if (code !== 0 && !worker.exitedAfterDisconnect) {
      console.log(`Worker ${worker.process.pid} 비정상 종료. 재생성...`);
      cluster.fork();
    }
  });
} else {
  http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`<h1>Worker ${process.pid} (v2) 응답</h1>`);
  }).listen(3000);

  console.log(`Worker ${process.pid} 실행`);
}
```

```
[무중단 재시작 과정]

1단계: 새 Worker 5 생성 → listening 상태 확인
       Worker 1 종료
       ┌─────────────────────────────────────┐
       │  W1(구) → 종료   W5(신) → 실행       │
       │  W2 ████████████████████████████     │
       │  W3 ████████████████████████████     │
       │  W4 ████████████████████████████     │
       └─────────────────────────────────────┘

2단계: 새 Worker 6 생성 → W2 종료
3단계: 새 Worker 7 생성 → W3 종료
4단계: 새 Worker 8 생성 → W4 종료

→ 항상 일부 워커가 요청을 처리 중이므로 서비스 중단 없음
```

```bash
# 서버 실행
$ node server.js

# 다른 터미널에서 재시작 신호 전송
$ kill -SIGUSR2 <Master PID>
```

---

## Cluster의 메모리 구조

각 Worker는 **독립된 프로세스**이므로 메모리를 공유하지 않는다.

```
┌─── Master 프로세스 ───┐
│  메모리 공간 A         │
│  - cluster.workers    │
│  - 관리 로직           │
└───────────────────────┘

┌─── Worker 1 ─────────┐  ┌─── Worker 2 ─────────┐
│  메모리 공간 B         │  │  메모리 공간 C         │
│  - 독자적인 V8 힙      │  │  - 독자적인 V8 힙      │
│  - sessions = {}      │  │  - sessions = {}      │
│  - 자체 이벤트 루프     │  │  - 자체 이벤트 루프     │
└───────────────────────┘  └───────────────────────┘
         ↑                          ↑
    메모리 공유 안 됨!         별도의 메모리 공간
```

| 특성 | 스레드 (worker_threads) | 프로세스 (cluster) |
|------|----------------------|-------------------|
| **메모리** | 공유 가능 (SharedArrayBuffer) | 독립 (공유 불가) |
| **안정성** | 한 스레드 오류 → 전체 영향 | 한 프로세스 죽어도 다른 프로세스 무관 |
| **비용** | 가벼움 | 무거움 (프로세스별 V8 인스턴스) |
| **통신** | 공유 메모리 | IPC 메시지 |
| **용도** | CPU 집약 작업 분담 | 서버 인스턴스 복제 |

> Cluster를 사용할 때 **세션 저장소**를 메모리에 두면, 각 워커가 서로 다른 세션 데이터를 가지게 된다. 세션을 공유하려면 Redis 같은 **외부 저장소**를 사용해야 한다.

---

## 실습: 완성형 Cluster 서버

CPU 코어를 모두 활용하고, 워커 자동 재시작과 상태 모니터링을 포함한 실전 예제다.

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  const workerRestartLimit = 5;
  const restartWindow = 60000;
  let restartCount = 0;
  let restartTimer = null;

  console.log(`=== Cluster Master ${process.pid} ===`);
  console.log(`CPU 코어: ${numCPUs}개`);
  console.log(`워커 ${numCPUs}개 생성 중...\n`);

  for (let i = 0; i < numCPUs; i++) {
    const worker = cluster.fork();
    console.log(`  Worker ${worker.process.pid} 생성`);
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`\n[${new Date().toISOString()}] Worker ${worker.process.pid} 종료 (code: ${code})`);

    if (worker.exitedAfterDisconnect) {
      console.log('  → 정상 종료 (disconnect 호출됨)');
      return;
    }

    restartCount++;
    if (!restartTimer) {
      restartTimer = setTimeout(() => {
        restartCount = 0;
        restartTimer = null;
      }, restartWindow);
    }

    if (restartCount > workerRestartLimit) {
      console.error('  → 재시작 한도 초과! 연쇄 장애 가능성. 재시작 중단.');
      return;
    }

    console.log(`  → 새 워커 생성 (재시작 ${restartCount}/${workerRestartLimit})`);
    cluster.fork();
  });

  cluster.on('message', (worker, msg) => {
    if (msg.type === 'status') {
      console.log(`  Worker ${worker.process.pid}: ${msg.requestCount}건 처리`);
    }
  });

  setInterval(() => {
    const workerIds = Object.keys(cluster.workers);
    console.log(`\n[상태] 활성 워커: ${workerIds.length}개`);
    for (const id of workerIds) {
      cluster.workers[id].send({ type: 'report' });
    }
  }, 30000);

} else {
  let requestCount = 0;

  process.on('message', (msg) => {
    if (msg.type === 'report') {
      process.send({ type: 'status', requestCount });
    }
  });

  http.createServer((req, res) => {
    requestCount++;

    if (req.url === '/health') {
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        worker: process.pid,
        uptime: process.uptime(),
        memory: process.memoryUsage(),
        requests: requestCount,
      }));
      return;
    }

    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`
      <h1>Cluster 서버</h1>
      <p>Worker PID: ${process.pid}</p>
      <p>처리한 요청: ${requestCount}건</p>
      <p>가동 시간: ${Math.floor(process.uptime())}초</p>
    `);
  }).listen(3000);

  console.log(`Worker ${process.pid} 준비 완료 (port: 3000)`);
}
```

```bash
$ node cluster-server.js
=== Cluster Master 12345 ===
CPU 코어: 4개
워커 4개 생성 중...

  Worker 12346 생성
  Worker 12347 생성
  Worker 12348 생성
  Worker 12349 생성
Worker 12346 준비 완료 (port: 3000)
Worker 12347 준비 완료 (port: 3000)
Worker 12348 준비 완료 (port: 3000)
Worker 12349 준비 완료 (port: 3000)

# 헬스 체크
$ curl http://localhost:3000/health
{"worker":12346,"uptime":15.23,"memory":{"rss":35651584,...},"requests":1}
```

### 연쇄 장애 방지

위 코드에서 `workerRestartLimit`을 설정한 이유는 **코드 버그로 워커가 반복적으로 죽는 상황**을 방지하기 위함이다.

```
[연쇄 장애 시나리오]

코드 버그 → Worker 시작 직후 에러로 종료
          → Master가 새 Worker 생성
          → 또 즉시 종료
          → 또 생성… (무한 반복)
          → CPU 100%, 시스템 불안정

해결: 일정 시간 내 재시작 횟수를 제한한다
```

---

## Cluster vs PM2

실무에서는 `cluster` 모듈을 직접 사용하기보다 **PM2** 같은 프로세스 매니저를 사용하는 경우가 많다.

```bash
# PM2 설치
$ npm install -g pm2

# 클러스터 모드로 실행 (CPU 코어 수만큼 자동 생성)
$ pm2 start server.js -i max

# 상태 확인
$ pm2 status

┌────┬────────────┬─────────┬──────┬───────┬──────────┐
│ id │ name       │ mode    │ pid  │ status│ restart  │
├────┼────────────┼─────────┼──────┼───────┼──────────┤
│ 0  │ server     │ cluster │ 1234 │ online│ 0        │
│ 1  │ server     │ cluster │ 1235 │ online│ 0        │
│ 2  │ server     │ cluster │ 1236 │ online│ 0        │
│ 3  │ server     │ cluster │ 1237 │ online│ 0        │
└────┴────────────┴─────────┴──────┴───────┴──────────┘

# 무중단 재시작
$ pm2 reload server

# 로그 확인
$ pm2 logs
```

### cluster 모듈 vs PM2

| 비교 항목 | cluster 모듈 | PM2 |
|-----------|-------------|-----|
| **설정** | 직접 코드 작성 | CLI 또는 설정 파일 |
| **워커 관리** | 직접 구현 | 자동 (재시작, 모니터링) |
| **무중단 재시작** | 직접 구현 | `pm2 reload` 한 줄 |
| **로그 관리** | 직접 구현 | 내장 (`pm2 logs`) |
| **모니터링** | 직접 구현 | 내장 (`pm2 monit`) |
| **자유도** | 높음 | 제한적이지만 충분 |

```javascript
// PM2 ecosystem 설정 파일 (ecosystem.config.js)
module.exports = {
  apps: [{
    name: 'my-app',
    script: 'server.js',
    instances: 'max',
    exec_mode: 'cluster',
    max_memory_restart: '300M',
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
  }],
};
```

```bash
$ pm2 start ecosystem.config.js
```

---

## Cluster 사용 시 주의사항

| 주의사항 | 설명 |
|---------|------|
| **메모리 미공유** | 워커 간 데이터 공유 불가. 세션·캐시는 Redis 등 외부 저장소 사용 |
| **Sticky Session** | WebSocket 사용 시 같은 클라이언트를 같은 워커로 보내야 함 |
| **포트 충돌** | 모든 워커가 같은 포트를 공유하지만, Master가 아닌 곳에서 `server.listen()`을 호출해야 함 |
| **연쇄 장애** | 코드 버그로 워커가 반복 종료되면 재시작 제한 필요 |
| **로그 관리** | 여러 워커의 로그가 섞이므로 PID를 포함하거나 별도 로그 시스템 필요 |
| **Graceful Shutdown** | 워커 종료 시 진행 중인 요청을 마무리한 뒤 종료해야 함 |

### Sticky Session이 필요한 경우

```
[문제 상황 — WebSocket]

클라이언트 ── HTTP 업그레이드 ──→ Worker 1 (WebSocket 연결 수립)
클라이언트 ── 다음 요청 ──────→ Worker 3 (WebSocket 연결 없음!) ← 에러

[해결 — Sticky Session]

클라이언트 ── 모든 요청 ──────→ 항상 Worker 1로 라우팅
```

### Graceful Shutdown 구현

```javascript
if (!cluster.isPrimary) {
  const server = http.createServer((req, res) => {
    res.writeHead(200);
    res.end('OK');
  }).listen(3000);

  process.on('SIGTERM', () => {
    console.log(`Worker ${process.pid}: 종료 신호 수신. 새 연결 거부...`);

    server.close(() => {
      console.log(`Worker ${process.pid}: 진행 중인 요청 완료. 종료.`);
      process.exit(0);
    });

    setTimeout(() => {
      console.error(`Worker ${process.pid}: 강제 종료 (타임아웃)`);
      process.exit(1);
    }, 10000);
  });
}
```

---

## 정리

```
┌─── Node.JS 확장 전략 ────────────────────────────────────┐
│                                                          │
│  1. Cluster (이번 포스트)                                 │
│     → 같은 서버에서 여러 프로세스로 CPU 코어 활용          │
│                                                          │
│  2. PM2                                                  │
│     → Cluster를 쉽게 관리하는 프로세스 매니저              │
│                                                          │
│  3. Docker + Load Balancer                               │
│     → 여러 서버에 걸쳐 수평 확장                          │
│                                                          │
│  4. worker_threads                                       │
│     → CPU 집약 작업을 별도 스레드에서 처리                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```
