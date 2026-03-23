---
title: "S3 Presigned URL"
date: 2026-03-23 10:00:00 +0900
categories: [AWS, S3]
tags: [s3, presigned-url, sigv4, aws, upload]
---

# S3 Presigned URL
이해도 ⭐⭐⭐⭐ (4/5)


## 0. 개념 & Concept
- S3 객체에 대한 **임시 접근 권한**을 URL 형태로 부여하는 방식이다.
- 서버가 AWS 자격증명(IAM)의 (AccessKey + SecretKey)로 서명한 URL을 클라이언트에게 전달.
- 클라이언트는 AWS 자격증명(IAM) 없이도 미리 서버가 서명한 URL을 이용하여 직접 S3에 업로드/다운로드가 가능하다.
- 이는 유효기간(Expiration) 이후 만료된다.

## 1. 어떠한 문제를 해결하고자 하였는가?
#### 1. S3는 기본적으로 비공개이며, 공개 전환은 가능하지만 바람직하지 않다.
- Bucket Policy, ACL 등으로 공개 전환은 가능하나, 민감한 데이터에 대해서는 보안상 권장되지 않는다.
- 즉, "특정 사람에게, 특정 파일을, 잠깐만" 열어주고싶다. 를 어떻게 해결할 수 있을까?
#### 2. 파일을 서버를 통해 전송하면 비싼 서버의 리소스를 사용하게 된다.
- client에 있는 파일을, 서버가 다시 모두 받아서 다시 S3로 올리게 되는 불필요한 트래픽이 생긴다. (서버 리소스 절약 및 불필요한 트래픽 감소로 인한 부하 감소)
- ex) 100MB파일을 100명이 동시에 올리면? 서버 메모리 및 네트워크 대역폭 폭발 & 서버 비용 선형 증가.


## 2. 해결 원리
- 신뢰를 URL에 위임하자.
- AWS의 sigV4서명 매커니즘을 활용.
- 요청내용 + SecretKey -> 해시 -> Siganture
- AWS는 만들어진 Signature로 요청이 진짜인지 검증한다.
- 일반방식 : Client가 요청하면 서버에서 SecretKey를 이용해 서명이 생성된다. 
- Presigned방식: Server가 미리 서명을 만들어서 URL에 포함시켜서 Client에게 알려준다. 
- 시점은 Client가 S3에 업로드/다운로드 하는 시점을 기준으로 요청해서 만들어지는지 (일반방식), 요청 전에 만들어지는지(Presigned)이다.

## 3. 원리 구현 방식.
```
⚠ 아래는 이해를 위한 단순화된 의사코드이다.
서명 = HMAC-SHA256(
    Secret Key,
    HTTP Method (GET or PUT)  ← 읽기/쓰기 중 딱 하나만
  + 버킷 이름 + 파일 경로      ← 이 파일만
  + 만료 시간                  ← 이 시간까지만
  + Content-Type (PUT의 경우) ← 이 타입의 파일만
)
```
- 실제로는 Secret Key를 직접 사용하지 않고, Date + Region + Service를 조합한 **파생 키(Derived Signing Key)**로 서명한다.
- 다른 파일경로, 만료 시간 이후 접근, 다른 행위 등이 불가해짐.
- SecretKey없이 위 조건을 바꾼 유효한 서명을 만들 수 없음.

### 주요 파라미터
- `bucket` : 대상 버킷
- `key` : 파일 경로 (e.g. `uploads/user/123/profile.png`)
- `expiresIn` : 만료 시간 (초 단위). 자격증명 유형에 따라 최대값이 다르다.
  - IAM User (장기 자격증명): 최대 **7일** (604,800초)
  - IAM Role (EC2, ECS, Lambda 등): 역할 세션 만료에 따라 제한됨 (일반적으로 **최대 6~12시간**)
  - 프로덕션 환경에서는 대부분 IAM Role을 사용하므로, 7일은 현실적으로 불가능한 경우가 많다.
- `contentType` : PUT 시 업로드할 파일 타입 지정 (서명에 포함)


### 동작 흐름 (업로드 기준)

```
┌────────┐   ①파일 업로드 요청      ┌──────────────┐
│        │ ───────────────────────▶ │              │
│ Client │                          │    Server    │
│        │ ◀─────────────────────── │  (Kotlin +   │
└────────┘   ②Presigned URL 발급    │   Spring)    │
                                    └──────┬───────┘
     ③URL로 S3에 직접 PUT                  │ Secret Key로
                                           │ 서명 생성
     ┌────────┐                    ┌───────▼──────┐
     │ Client │ ─────────────────▶ │     S3       │
     └────────┘  (서버 안 거침!)    │  서명 검증 후 │
                                   │  저장/응답    │
                                   └──────────────┘
```

### 종류

| 종류 | 설명 |
|------|------|
| **GET** presigned URL | 비공개 객체 다운로드 허용 (가장 많이 사용) |
| **PUT** presigned URL | 특정 경로에 업로드 허용 (가장 많이 사용) |
| **HEAD** presigned URL | 객체 메타데이터 조회 허용 |
| **DELETE** presigned URL | 객체 삭제 허용 |

- 주로 GET/PUT이 사용되며, HEAD/DELETE도 필요에 따라 presigned URL로 발급 가능하다.

---

## 4. 어떤 목적으로 어떨때 사용할까?
#### 예시1. 프로필 이미지/ 첨부파일 업로드
#### 예시2. 민감한 파일의 임시 공유
- 계약서, 청구서 등 "이 사람에게 1시간만" 열어주기.
#### 예시3. 대용량 파일 처리
- 동영상, 데이터 셋 등 수백 MB ~ GB 단위 파일
- 서버 경유 시, 서버 메모리 & 대역폭 낭비
- Presigned URL -> 클라이언트 <-> S3 파일 전송
#### 예시4. 프론트엔드에서 직접 S3접근
- Mobile App/ SPA(single page application) 에서 AWS 자격증명 노출없이 S3에 직접 접근해야 할 때.
- 서버가 키를 관리하고, 이를 통해 만든 URL을 활용

---

## 5. 구현과 Spring (Kotlin) 예시

```kotlin
val presignedUrl = s3Presigner.presignPutObject {
    it.putObjectRequest { req ->
        req.bucket("my-bucket")
        req.key("uploads/$userId/$fileName")
        req.contentType("image/png")
    }
    it.signatureDuration(Duration.ofMinutes(10))
}

return presignedUrl.url().toString()

ex) https://my-bucket.s3.amazonaws.com/uploads/photo.jpg
?X-Amz-Algorithm=AWS4-HMAC-SHA256
&X-Amz-Credential=AKIAIOSFODNN7/20260323/ap-northeast-2/s3/aws4_request
&X-Amz-Expires=3600 &X-Amz-Signature=abc123ef...
↑ 누가 서명했나 ↑ 유효시간(초) ↑ 위조 불가 서명값
```
- Put(업로드) 시에 미리 파일명을 만들어놔야하기 때문에 (경로지정), 이를 겹치지 않도록 잘 설정해서 미리 만들어두어야한다. 
- 만약 client로부터 어떠한 파일명을 받아서 저장하도록 하는 케이스에는, 중복되지 않는지 확인해야한다.
- 한글의 경우 인코딩 문제로 같은 파일명인데도 잘 보이지 않는 케이스도 있으니 주의하길..
- 원본파일명은 DB 또는 업로드 후처리에 넣자. metadata에 넣어도 되지만..

### 정상 업로드된걸 어떻게 서버가 알 수 있나요?
- 클라이언트가 업로드 이후 BE에게 알려줄 수 있습니다. 
- S3 Event를 이용해 BE 내부에서 처리할 수도 있습니다.


### 보안 고려사항
- URL이 유출되면 만료 전까지 누구나 접근 가능 → 만료 시간을 짧게 설정
- `contentType`을 서명에 포함시켜야 다른 타입의 파일 업로드 차단 가능
- 업로드 완료 후 서버에서 실제 파일 존재 여부 검증 권장
- IAM Role에 최소 권한 원칙 적용 (해당 버킷/키만 허용)
- 용량이 작고, 바이러스 검사, 이미지 리사이징 등 업로든 전처리가 필요한 케이스에 대해서는 꼭 presigned Url을 써야하는지 고민해볼 필요가 있다. (파일크기 제한 등)
- 허용가능한 MIME whitelist 관리


---

## 기타. 
### AWS SigV4?
- SigV4는 "매 요청마다 서명을 생성하는 방식"으로 AWS의 모든 HTTP 요청을 인증하는 **범용 인증 방식** 이다.
![SigV4 서명 흐름](/assets/posts/S3-presignedUrl/S3-presignedUrl_img_001.png)
```
HMAC-SHA256( Secret Key, HTTP Method + URL + Headers + Body + Timestamp )
→ Signature 값 생성 → Authorization 헤더에 포함 → AWS가 동일하게 재계산해서 검증
⚠ 매 요청마다 서버가 서명을 생성해야 함 / Secret Key가 서버에만 존재
```
```
SDK (SigV4)          서명 위치: Authorization 헤더
                     누가 서명: SDK가 매 요청마다 자동 생성
                     사용 주체: 서버 (Secret Key 보유)

Presigned URL        서명 위치: URL 쿼리 파라미터
                     누가 서명: 서버가 미리 생성해서 URL에 박음
                     사용 주체: 클라이언트 (Secret Key 없음)
```

#### 그럼 구현은 어디서? 
- 우리가 사용하는 AWS sdk, 또는 라이브러리에 내장 구현되어 있기 때문에 개발자가 구현할일은 없다. 
- 그럼 AWS에서 Credentials는 어디서 가져오나요? 는 `DefaultChainCredentialsProvider`를 참조 (Credential Provider Chain)
    - Credential chain에서 인증되어야 aws sdk사용이 가능하고, 결국 IAM의 AccessKey, SecretKey는 항상 채워짐.

#### 한 줄 정리
SigV4는 AWS 전체의 인증 인프라, Presigned URL은 그 위에서 S3 같은 스토리지 서비스에 특화된 접근 위임 패턴이에요.

## 더 알아보기
- [ ] Presigned URL vs CloudFront Signed URL 차이는?
- [ ] multipart upload와 presigned URL을 함께 쓰는 방법 (대용량 upload)
- [ ] 업로드 완료 이벤트 처리 (S3 Event → SQS/SNS)
- [ ] contentType 미지정 시 발생할 수 있는 보안 이슈
- [ ] SigV4a — ECDSA(Elliptic Curve Digital Signature Algorithm) 기반의 서명 방식. 멀티 리전 서명을 지원하며, S3 Multi-Region Access Point 등에서 사용된다.
- [ ] S3 Multi-Region Access Point란?

## 참조 문서
- [S3 Presigned URL 개요](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Presigned URL로 객체 업로드](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [AWS SigV4 서명 프로세스](https://docs.aws.amazon.com/general/latest/gr/sigv4_signing.html)
- [SigV4 Query String Auth (Presigned URL 서명 상세)](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html)
- [AWS Credential Provider Chain](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)
- [CloudFront Signed URL](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-urls.html)
