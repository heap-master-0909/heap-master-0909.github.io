---
title: 12-04. aws - s3, lambda
date: 2026-04-09 00:00:00 +0900
categories: [Node.JS, aws]
tags: [Tech, Node.JS, aws, s3, lambda]
pin: true
---

## AWS S3란?

S3(Simple Storage Service)는 AWS의 **파일 저장 서비스**이다. 이미지, 동영상, 문서 등 어떤 파일이든 저장할 수 있으며, URL을 통해 바로 접근할 수 있다.

Node.js 서버에 이미지를 직접 저장하면 서버 디스크가 부족해지고, 서버를 재배포하면 파일이 사라질 수 있다. S3를 사용하면 파일을 서버와 분리하여 안전하게 관리할 수 있다.

### 서버 저장 vs S3 저장

| 항목 | 서버 디스크 저장 | AWS S3 |
|:---|:---|:---|
| 용량 | EC2 디스크 용량 제한 | 무제한 |
| 서버 재배포 시 | 파일 유실 위험 | 영향 없음 |
| 다중 서버 | 파일 공유 불가 | 모든 서버에서 접근 가능 |
| CDN 연동 | 직접 구성 필요 | CloudFront와 쉽게 연동 |
| 비용 | EC2 요금에 포함 | 저장 용량 + 요청 수 기반 과금 |

### S3 핵심 개념

| 용어 | 설명 |
|:---|:---|
| Bucket | 파일을 담는 최상위 컨테이너 (폴더 같은 개념) |
| Object | 버킷에 저장되는 개별 파일 |
| Key | 객체의 고유 식별자 (파일 경로 + 이름) |
| Region | 버킷이 위치하는 AWS 리전 |

### 설치

```sh
npm i @aws-sdk/client-s3 multer multer-s3
```

| 패키지 | 역할 |
|:---|:---|
| `@aws-sdk/client-s3` | AWS S3 클라이언트 (v3) |
| `multer` | Express 파일 업로드 미들웨어 |
| `multer-s3` | multer로 받은 파일을 S3에 바로 저장 |

### S3 버킷 생성 (AWS 콘솔)

1. AWS 콘솔 → S3 → 버킷 만들기
2. 버킷 이름 입력 (전 세계에서 고유해야 함)
3. 리전 선택 (예: ap-northeast-2, 서울)
4. 퍼블릭 액세스 차단 설정 해제 (이미지 공개 필요 시)
5. 버킷 생성

### IAM 사용자 설정

S3에 접근하려면 **AccessKey**가 필요하다:

1. AWS 콘솔 → IAM → 사용자 → 사용자 추가
2. 프로그래밍 방식 액세스 선택
3. 권한: `AmazonS3FullAccess` 정책 연결
4. 생성 후 **Access Key ID**와 **Secret Access Key** 저장

```
# .env
S3_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
S3_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
S3_BUCKET=my-app-bucket
S3_REGION=ap-northeast-2
```

### Express + S3 이미지 업로드

```js
const { S3Client } = require('@aws-sdk/client-s3');
const multer = require('multer');
const multerS3 = require('multer-s3');
const path = require('path');

const s3 = new S3Client({
  region: process.env.S3_REGION,
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY_ID,
    secretAccessKey: process.env.S3_SECRET_ACCESS_KEY,
  },
});

const upload = multer({
  storage: multerS3({
    s3: s3,
    bucket: process.env.S3_BUCKET,
    key(req, file, cb) {
      cb(null, `original/${Date.now()}_${path.basename(file.originalname)}`);
    },
  }),
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB 제한
});
```

| 옵션 | 설명 |
|:---|:---|
| `s3` | S3 클라이언트 인스턴스 |
| `bucket` | 저장할 버킷 이름 |
| `key` | S3에 저장될 파일 경로 + 이름 |
| `limits.fileSize` | 업로드 파일 크기 제한 |

### 라우터에서 업로드 사용

```js
const express = require('express');
const router = express.Router();

// 이미지 업로드 라우트
router.post('/img', upload.single('img'), (req, res) => {
  res.json({ url: req.file.location });
});
```

`req.file.location`에 S3 URL이 담긴다:

```
https://my-app-bucket.s3.ap-northeast-2.amazonaws.com/original/1712649600000_photo.jpg
```

### 업로드 흐름

```
1. 클라이언트 → POST /img (이미지 파일)
2. multer → 파일 수신
3. multer-s3 → S3 버킷에 업로드
4. 서버 → S3 URL을 응답으로 반환
5. 클라이언트 → URL을 DB에 저장하여 사용
```

---

## AWS Lambda란?

Lambda는 AWS의 **서버리스 컴퓨팅 서비스**이다. 서버를 직접 관리하지 않고, 코드(함수)만 등록하면 이벤트가 발생할 때 자동으로 실행된다. 실행된 시간만큼만 과금되므로 비용 효율적이다.

### 서버 vs Lambda 비교

| 항목 | EC2 서버 | Lambda |
|:---|:---|:---|
| 서버 관리 | 직접 운영 | AWS가 관리 |
| 과금 | 24시간 가동 비용 | 실행 시간만큼만 과금 |
| 확장 | 수동 스케일링 | 자동 스케일링 |
| 실행 시간 | 제한 없음 | 최대 15분 |
| 용도 | 상시 운영 서버 | 이벤트 기반 작업 |

### S3 + Lambda 조합: 이미지 리사이징

가장 대표적인 활용 사례가 **이미지 리사이징**이다. 사용자가 고해상도 이미지를 업로드하면, Lambda가 자동으로 썸네일을 생성한다.

```
사용자 → 이미지 업로드 → S3 (original/)
                           ↓ (S3 이벤트 트리거)
                        Lambda 실행
                           ↓ (이미지 리사이징)
                        S3 (thumb/) ← 썸네일 저장
```

### Lambda 함수 작성 (이미지 리사이징)

```js
const { S3Client, GetObjectCommand, PutObjectCommand } = require('@aws-sdk/client-s3');
const sharp = require('sharp');

const s3 = new S3Client();

exports.handler = async (event) => {
  const bucket = event.Records[0].s3.bucket.name;
  const key = decodeURIComponent(event.Records[0].s3.object.key);
  const filename = key.split('/').pop();

  // S3에서 원본 이미지 가져오기
  const getResult = await s3.send(new GetObjectCommand({
    Bucket: bucket,
    Key: key,
  }));
  const imageBuffer = Buffer.concat(
    await getResult.Body.toArray()
  );

  // sharp로 이미지 리사이징
  const resizedImage = await sharp(imageBuffer)
    .resize(200, 200, { fit: 'inside' })
    .toBuffer();

  // 리사이징된 이미지를 thumb/ 경로에 저장
  await s3.send(new PutObjectCommand({
    Bucket: bucket,
    Key: `thumb/${filename}`,
    Body: resizedImage,
  }));

  return { statusCode: 200, body: 'OK' };
};
```

| 구성 요소 | 설명 |
|:---|:---|
| `event.Records` | S3에서 전달한 이벤트 정보 (버킷명, 파일 키 등) |
| `GetObjectCommand` | S3에서 파일 다운로드 |
| `sharp` | Node.js 이미지 처리 라이브러리 |
| `PutObjectCommand` | 리사이징된 이미지를 S3에 업로드 |

### Lambda 배포 순서

1. 프로젝트 폴더 생성 후 의존성 설치

```sh
mkdir image-resizer && cd image-resizer
npm init -y
npm i @aws-sdk/client-s3 sharp
```

2. `index.js`에 핸들러 함수 작성

3. 프로젝트를 zip으로 압축

```sh
zip -r function.zip .
```

4. AWS 콘솔 → Lambda → 함수 생성
   - 런타임: Node.js 20.x
   - 아키텍처: x86_64
   - zip 파일 업로드

5. S3 트리거 설정
   - 트리거 추가 → S3 선택
   - 버킷: 이미지 업로드 버킷
   - 이벤트 유형: PUT (파일 업로드 시)
   - 접두사: `original/` (original 폴더에 업로드될 때만 실행)

### Lambda 실행 권한 설정

Lambda가 S3에 접근하려면 IAM 역할에 S3 권한이 필요하다:

- `AmazonS3ReadOnlyAccess` — 원본 이미지 읽기
- `AmazonS3FullAccess` — 썸네일 저장까지 필요한 경우

### 프론트엔드에서 이미지 사용

```js
// 원본 이미지 URL
const originalUrl = 'https://my-bucket.s3.amazonaws.com/original/photo.jpg';

// 썸네일 URL (Lambda가 자동 생성)
const thumbUrl = originalUrl.replace('/original/', '/thumb/');
```

서버에서 원본 이미지 URL을 저장하고, 프론트엔드에서 목록 표시 시 `original`을 `thumb`으로 바꿔서 썸네일을 보여주면 된다. 상세 페이지에서는 원본 URL을 사용한다.

### S3 + Lambda 전체 흐름 정리

```
[클라이언트]
    │
    ├── POST /img (이미지 업로드)
    │
[Express 서버 + multer-s3]
    │
    ├── S3 original/ 에 저장
    ├── S3 URL을 DB에 저장
    ├── 응답: { url: "https://...original/photo.jpg" }
    │
[S3 이벤트 트리거]
    │
    ├── original/에 파일 생성 감지
    │
[Lambda 함수]
    │
    ├── 원본 다운로드 → 리사이징 → thumb/에 저장
    │
[프론트엔드]
    │
    ├── 목록: thumb/photo.jpg (썸네일)
    └── 상세: original/photo.jpg (원본)
```

