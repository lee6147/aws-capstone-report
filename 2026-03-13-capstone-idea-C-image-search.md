# 이미지 기반 자동 분류/검색 시스템 - 아키텍처 설계 문서

> **프로젝트**: AWS실전프로젝트I 캡스톤 (국민대학교 AWS/양자통신융합전공)
> **작성일**: 2026-03-13
> **팀 규모**: 2~3인
> **기간**: 15주 (2026-03-06 ~ 2026-06-12)
> **상태**: Draft

---

## Executive Summary

| 관점 | 내용 |
|------|------|
| **Problem** | 대량의 이미지를 수동으로 분류/검색하는 것은 비효율적이며, 텍스트 기반 검색은 시각적 유사성을 포착하지 못한다 |
| **Solution** | AWS Rekognition으로 자동 태깅하고, Bedrock Titan Embeddings로 벡터화하여 OpenSearch Serverless에서 유사 이미지 검색을 수행하는 서버리스 파이프라인 |
| **Function/UX Effect** | 이미지 업로드만으로 자동 라벨링 + 유사 이미지 검색이 가능한 원스톱 경험 |
| **Core Value** | 서버리스 아키텍처로 운영 부담 최소화, AWS AI 서비스 통합으로 ML 모델 직접 학습 없이 고품질 이미지 분석 제공 |

---

## 1. 시스템 아키텍처 다이어그램

### 1.1 전체 서비스 흐름

```
                          [사용자]
                             |
                             v
                   +-----------------+
                   |  CloudFront     |  (CDN, 정적 웹 호스팅)
                   |  + S3 (Static)  |
                   +-----------------+
                             |
                             v
                   +-----------------+
                   |  API Gateway    |  (REST API)
                   |  (HTTP API)     |
                   +-----------------+
                        |        |
                        v        v
              +----------+  +----------+
              | Lambda   |  | Lambda   |
              | (Upload) |  | (Search) |
              +----------+  +----------+
                   |              |
                   v              v
            +------------+  +-----------------------+
            | S3 Bucket  |  | OpenSearch Serverless |
            | (Images)   |  | (Vector Search)       |
            +------------+  +-----------------------+
                   |              ^
                   | (S3 Event)   |
                   v              |
            +------------+        |
            | Lambda     |        |
            | (Pipeline) |--------+
            +------------+
               |       |
               v       v
      +-----------+  +------------------+
      |Rekognition|  | Bedrock          |
      |(Labels)   |  | (Titan Multimodal|
      +-----------+  |  Embeddings)     |
                     +------------------+
```

### 1.2 상세 데이터 흐름

```
[이미지 업로드 흐름]
User -> CloudFront -> API Gateway -> Lambda(Upload)
  -> S3 Presigned URL 생성
  -> User가 S3에 직접 업로드 (대용량 파일 효율)
  -> S3 Event Notification 발생
  -> Lambda(Pipeline) 트리거

[자동 처리 파이프라인]
Lambda(Pipeline):
  1. S3에서 이미지 읽기
  2. Rekognition.detectLabels() -> 태그 목록 추출
  3. Bedrock Titan Multimodal Embeddings -> 1024차원 벡터 생성
  4. OpenSearch Serverless에 문서 인덱싱:
     {
       imageKey: "s3://bucket/image.jpg",
       labels: ["cat", "animal", "pet"],
       embedding: [0.12, -0.34, ...],  // 1024-dim
       metadata: { uploadedBy, uploadedAt, fileSize }
     }
  5. (Optional) DynamoDB에 메타데이터 저장

[유사 이미지 검색 흐름]
User -> 검색 이미지 업로드 또는 텍스트 입력
  -> API Gateway -> Lambda(Search)
  -> Bedrock Titan Embeddings (쿼리 벡터 생성)
  -> OpenSearch kNN 검색 (코사인 유사도)
  -> 상위 K개 결과 반환 (이미지 URL + 유사도 점수 + 라벨)

[라벨 기반 필터 검색]
User -> 태그 선택/입력
  -> API Gateway -> Lambda(Search)
  -> OpenSearch term 쿼리 (라벨 필터링)
  -> 결과 반환
```

---

## 2. AWS 서비스 구성

### 2.1 서비스별 역할과 설정

| 서비스 | 역할 | 설정 상세 |
|--------|------|----------|
| **S3 (Images)** | 원본 이미지 저장소 | 버킷 정책, CORS 설정, Event Notification, Lifecycle 정책 |
| **S3 (Static)** | 프론트엔드 정적 호스팅 | CloudFront Origin, 빌드 산출물 저장 |
| **CloudFront** | CDN + HTTPS 종단점 | S3 Origin(웹), API Gateway Origin(API), OAC 설정 |
| **API Gateway** | REST API 엔드포인트 | HTTP API(비용 절감), Lambda 프록시 통합, CORS |
| **Lambda (Upload)** | Presigned URL 생성 | Node.js 20.x, 128MB, 10초 타임아웃 |
| **Lambda (Pipeline)** | 이미지 분석 파이프라인 | Python 3.12, 512MB~1GB, 60초 타임아웃, S3 트리거 |
| **Lambda (Search)** | 검색 쿼리 처리 | Python 3.12, 256MB, 15초 타임아웃 |
| **Rekognition** | 이미지 라벨링 | detectLabels API, MaxLabels=20, MinConfidence=70 |
| **Bedrock** | 이미지 임베딩 생성 | Titan Multimodal Embeddings G1, 1024-dim 벡터 |
| **OpenSearch Serverless** | 벡터 검색 엔진 | Vector Search Collection, kNN 인덱스, AOSS |
| **IAM** | 권한 관리 | Lambda 실행 역할, S3/Rekognition/Bedrock/AOSS 접근 정책 |
| **CloudWatch** | 모니터링/로깅 | Lambda 로그, 커스텀 메트릭, 알람 |

### 2.2 서비스 연결 다이어그램

```
[프론트엔드 계층]
  CloudFront ──OAC──> S3 (Static Website)
  CloudFront ──Origin──> API Gateway

[API 계층]
  API Gateway ──Proxy──> Lambda (Upload)   [POST /api/upload]
  API Gateway ──Proxy──> Lambda (Search)   [POST /api/search]
  API Gateway ──Proxy──> Lambda (Search)   [GET  /api/images]
  API Gateway ──Proxy──> Lambda (Search)   [GET  /api/labels]

[이벤트 처리 계층]
  S3 (Images) ──Event──> Lambda (Pipeline)

[AI/ML 계층]
  Lambda (Pipeline) ──API──> Rekognition (detectLabels)
  Lambda (Pipeline) ──API──> Bedrock (Titan Embeddings)
  Lambda (Search)   ──API──> Bedrock (Titan Embeddings)  // 쿼리 이미지 벡터화

[데이터 계층]
  Lambda (Pipeline) ──Index──> OpenSearch Serverless
  Lambda (Search)   ──Query──> OpenSearch Serverless
  Lambda (Upload)   ──PutObject──> S3 (Images) [Presigned URL]
```

### 2.3 IAM 정책 설계

```
Lambda-Upload-Role:
  - s3:PutObject (images bucket)
  - s3:GetObject (images bucket)

Lambda-Pipeline-Role:
  - s3:GetObject (images bucket)
  - rekognition:DetectLabels
  - bedrock:InvokeModel (titan-embed-image-v1)
  - aoss:APIAccessAll (OpenSearch collection)
  - logs:CreateLogGroup, logs:PutLogEvents

Lambda-Search-Role:
  - bedrock:InvokeModel (titan-embed-image-v1)
  - aoss:APIAccessAll (OpenSearch collection)
  - s3:GetObject (images bucket, Presigned URL 생성용)
  - logs:CreateLogGroup, logs:PutLogEvents
```

---

## 3. 데이터 파이프라인 상세

### 3.1 이미지 업로드 파이프라인

```
Step 1: Presigned URL 요청
  Client -> POST /api/upload { fileName, contentType }
  Lambda -> S3.createPresignedPost()
  Lambda -> Return { uploadUrl, fields, imageKey }

Step 2: 이미지 직접 업로드
  Client -> PUT uploadUrl (이미지 바이너리)
  S3 -> 이미지 저장 완료

Step 3: S3 Event 트리거
  S3 -> Event Notification (s3:ObjectCreated:*)
  -> Lambda(Pipeline) 비동기 호출

Step 4: AI 분석 파이프라인
  Lambda(Pipeline):
    4-1. S3에서 이미지 바이트 다운로드
    4-2. Rekognition.detect_labels(Image={S3Object})
         -> labels: [{Name: "Cat", Confidence: 99.2}, ...]
    4-3. Bedrock.invoke_model(
           modelId="amazon.titan-embed-image-v1",
           body={inputImage: base64_encoded_image}
         )
         -> embedding: [float x 1024]
    4-4. OpenSearch에 문서 인덱싱:
         POST /{index}/_doc
         {
           "imageKey": "uploads/2026/03/uuid.jpg",
           "imageUrl": "https://cdn.example.com/uploads/...",
           "labels": ["Cat", "Animal", "Pet", "Mammal"],
           "labelConfidences": {"Cat": 99.2, "Animal": 98.5, ...},
           "embedding": [0.12, -0.34, 0.56, ...],
           "fileSize": 1024000,
           "contentType": "image/jpeg",
           "uploadedAt": "2026-03-13T10:00:00Z",
           "uploadedBy": "user-123"
         }
```

### 3.2 벡터 검색 파이프라인

```
[이미지로 검색 (Image-to-Image)]
Step 1: 검색 이미지 업로드
  Client -> POST /api/search/image { image: base64 }

Step 2: 쿼리 벡터 생성
  Lambda -> Bedrock Titan Embeddings (검색 이미지)
  -> query_vector: [float x 1024]

Step 3: kNN 검색
  Lambda -> OpenSearch kNN query:
  {
    "size": 20,
    "query": {
      "knn": {
        "embedding": {
          "vector": query_vector,
          "k": 20
        }
      }
    }
  }

Step 4: 결과 반환
  -> [{imageUrl, labels, score, metadata}, ...]

[텍스트로 검색 (Text-to-Image)]
Step 1: 검색어 입력
  Client -> POST /api/search/text { query: "고양이" }

Step 2: 텍스트 벡터 생성
  Lambda -> Bedrock Titan Embeddings (inputText: "고양이")
  -> query_vector: [float x 1024]

Step 3-4: 위와 동일 (kNN 검색 + 결과 반환)

[하이브리드 검색]
  벡터 검색 + 라벨 필터링 결합:
  {
    "query": {
      "bool": {
        "must": [
          {"knn": {"embedding": {"vector": v, "k": 20}}},
          {"terms": {"labels": ["Cat", "Animal"]}}
        ]
      }
    }
  }
```

### 3.3 OpenSearch 인덱스 매핑

```json
{
  "settings": {
    "index": {
      "knn": true,
      "knn.algo_param.ef_search": 512
    }
  },
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 1024,
        "method": {
          "name": "hnsw",
          "space_type": "cosinesimil",
          "engine": "nmslib",
          "parameters": {
            "ef_construction": 512,
            "m": 16
          }
        }
      },
      "imageKey": { "type": "keyword" },
      "imageUrl": { "type": "keyword" },
      "labels": { "type": "keyword" },
      "labelConfidences": { "type": "object" },
      "uploadedAt": { "type": "date" },
      "uploadedBy": { "type": "keyword" },
      "fileSize": { "type": "long" },
      "contentType": { "type": "keyword" }
    }
  }
}
```

---

## 4. 구현 난이도 분석

### 4.1 컴포넌트별 난이도

| 컴포넌트 | 난이도 | 설명 | 예상 소요 |
|----------|--------|------|----------|
| S3 버킷 설정 + CORS | ★☆☆☆☆ | AWS 콘솔에서 기본 설정 | 0.5일 |
| CloudFront + S3 정적 호스팅 | ★★☆☆☆ | OAC 설정, 캐시 정책 | 1일 |
| API Gateway (HTTP API) | ★★☆☆☆ | Lambda 프록시 통합, CORS | 0.5일 |
| Lambda (Upload - Presigned URL) | ★★☆☆☆ | S3 SDK 사용, 비교적 단순 | 1일 |
| Lambda (Pipeline - Rekognition) | ★★★☆☆ | Rekognition API 호출 자체는 단순, 에러 핸들링이 핵심 | 1.5일 |
| Lambda (Pipeline - Bedrock) | ★★★☆☆ | 모델 접근 권한 설정 + API 호출 | 1.5일 |
| **OpenSearch Serverless 설정** | **★★★★☆** | **컬렉션 생성, 접근 정책, 데이터 접근 정책, 네트워크 정책 설정이 복잡** | **3일** |
| **OpenSearch kNN 인덱스 + 검색** | **★★★★☆** | **인덱스 매핑, kNN 쿼리 최적화, 하이브리드 검색** | **3일** |
| Lambda (Search) | ★★★☆☆ | 벡터 생성 + OpenSearch 쿼리 조합 | 2일 |
| 프론트엔드 (이미지 업로드 UI) | ★★☆☆☆ | 파일 선택, 드래그앤드롭, 프리뷰 | 2일 |
| 프론트엔드 (검색 결과 갤러리) | ★★★☆☆ | 그리드 레이아웃, 유사도 표시, 필터링 | 2일 |
| 프론트엔드 (라벨 필터/탐색) | ★★☆☆☆ | 태그 클라우드, 필터 조합 | 1일 |
| IAM 역할/정책 통합 | ★★★☆☆ | 서비스간 최소 권한 원칙 적용 | 1일 |
| 에러 핸들링 + 모니터링 | ★★☆☆☆ | CloudWatch 로그, 재시도 로직 | 1일 |

### 4.2 핵심 난관 (Critical Path)

1. **OpenSearch Serverless 설정 (최고 난이도)**
   - Collection 생성 시 3가지 정책(접근, 데이터, 네트워크) 모두 설정 필요
   - Vector Search Collection은 일반 Search Collection과 별도 OCU 할당
   - IAM 기반 인증으로 Lambda에서 요청 시 AWS Signature V4 서명 필요
   - 대응: AWS 문서 + 멘토 지원 적극 활용, 초기에 집중 투자

2. **Bedrock 모델 접근 권한**
   - Bedrock 모델 사용을 위해 Model Access 요청 필요 (승인까지 시간 소요 가능)
   - 리전별 가용 모델 상이
   - 대응: 프로젝트 시작 즉시 Model Access 요청

3. **Presigned URL 기반 업로드 흐름**
   - CORS, Content-Type, 만료 시간 등 세부 설정
   - 대응: 공식 문서 예제 기반 구현

---

## 5. 15주 타임라인 (주차별 개발 계획)

### 5.1 전체 로드맵

```
Week  1-2  [기획/설계]     ████░░░░░░░░░░░░░░░░░░░░░░░░░░  13%
Week  3-4  [인프라 구축]   ░░░░████░░░░░░░░░░░░░░░░░░░░░░  27%
Week  5-6  [백엔드 핵심]   ░░░░░░░░████░░░░░░░░░░░░░░░░░░  40%
Week  7    [MVP 통합]      ░░░░░░░░░░░░██░░░░░░░░░░░░░░░░  47%
Week  8    [중간 발표]     ░░░░░░░░░░░░░░██░░░░░░░░░░░░░░  53%  <-- 중간고사
Week  9-10 [프론트엔드]    ░░░░░░░░░░░░░░░░████░░░░░░░░░░  67%
Week 11-12 [고도화]        ░░░░░░░░░░░░░░░░░░░░████░░░░░░  80%
Week 13-14 [테스트/최적화] ░░░░░░░░░░░░░░░░░░░░░░░░████░░  93%
Week 15    [최종 발표]     ░░░░░░░░░░░░░░░░░░░░░░░░░░░░██ 100%
```

### 5.2 주차별 상세 계획

| 주차 | 기간 | 목표 | 산출물 | 마일스톤 |
|------|------|------|--------|----------|
| **1주** | 03/06-03/12 | 프로젝트 기획, 요구사항 정의, AWS 계정 설정 | 기획서, AWS 계정, Bedrock Model Access 요청 | 팀 빌딩 완료 |
| **2주** | 03/13-03/19 | 아키텍처 설계 확정, S3 버킷 생성, IAM 기본 설정 | 아키텍처 문서(본 문서), S3 버킷 | 설계 확정 |
| **3주** | 03/20-03/26 | Lambda 함수 스캐폴딩, API Gateway 설정, S3 이벤트 트리거 | Lambda 3개 기본 코드, API Gateway 배포 | 인프라 기반 완성 |
| **4주** | 03/27-04/02 | OpenSearch Serverless 컬렉션 생성, 벡터 인덱스 설정 | AOSS 컬렉션, kNN 인덱스 매핑 | 데이터 계층 완성 |
| **5주** | 04/03-04/09 | Rekognition 연동 (라벨링), Bedrock 연동 (임베딩) | Lambda Pipeline 함수 완성 | AI 파이프라인 완성 |
| **6주** | 04/10-04/16 | 검색 Lambda 구현, kNN 검색 + 라벨 검색 | Lambda Search 함수 완성 | 검색 기능 완성 |
| **7주** | 04/17-04/23 | **MVP 통합 테스트**, 업로드 -> 분석 -> 검색 E2E 검증 | 동작하는 MVP (API 수준) | **MVP 완성** |
| **8주** | 04/24-04/30 | **중간고사/중간 발표**, MVP 데모 준비 | 발표 자료, 데모 영상 | **중간 발표** |
| **9주** | 05/01-05/07 | 프론트엔드 개발 (업로드 UI, 검색 UI) | React 앱 기본 UI | 프론트엔드 시작 |
| **10주** | 05/08-05/14 | 프론트엔드 완성 (갤러리, 라벨 필터, 유사도 표시) | 완성된 웹 UI | 프론트엔드 완성 |
| **11주** | 05/15-05/21 | CloudFront 배포, 사용자 인증(Cognito), 에러 핸들링 | 프로덕션 배포, 인증 기능 | 배포 완료 |
| **12주** | 05/22-05/28 | 고도화: 하이브리드 검색, 이미지 유사도 시각화, 성능 최적화 | 고도화 기능 | 차별화 기능 |
| **13주** | 05/29-06/04 | 부하 테스트, 보안 점검, 비용 최적화 | 테스트 리포트 | QA 완료 |
| **14주** | 06/05-06/11 | 최종 문서화, 발표 준비, 데모 시나리오 확정 | 최종 보고서, 발표자료 | 문서화 완료 |
| **15주** | 06/12 | **최종 발표** | 최종 데모 | **프로젝트 완료** |

### 5.3 주차별 팀 분담 (2인 기준)

| 주차 | 팀원 A (백엔드/인프라) | 팀원 B (프론트엔드/통합) |
|------|----------------------|------------------------|
| 1-2주 | AWS 인프라 설계 + 계정 설정 | 요구사항 정리 + UI 와이어프레임 |
| 3-4주 | Lambda + API Gateway + AOSS | S3 설정 + Presigned URL 테스트 |
| 5-6주 | Rekognition + Bedrock 연동 | 검색 Lambda + OpenSearch 쿼리 |
| 7-8주 | MVP 통합 + 디버깅 | 발표 자료 + 간단 데모 UI |
| 9-10주 | API 개선 + 에러 핸들링 | 프론트엔드 개발 |
| 11-12주 | CloudFront + Cognito + 보안 | UI 고도화 + 시각화 |
| 13-15주 | 성능 테스트 + 최적화 | 문서화 + 발표 준비 |

---

## 6. 예상 비용

### 6.1 AWS 프리티어 + 학생 크레딧 기준

> **가정**: AWS Educate / Academy 크레딧 $100~$200 사용 가능, 프리티어 12개월 내 계정

| 서비스 | 프리티어 범위 | 초과 시 단가 | 월 예상 비용 (개발기간) |
|--------|-------------|-------------|----------------------|
| **S3** | 5GB 저장, 2만 GET, 2천 PUT | $0.023/GB | **$0** (프리티어 내) |
| **Lambda** | 100만 요청, 40만 GB-초/월 | $0.20/100만 요청 | **$0** (프리티어 내) |
| **API Gateway** | HTTP API 100만 호출/월 (12개월) | $1.00/100만 | **$0** (프리티어 내) |
| **CloudFront** | 1TB 전송/월 (12개월) | $0.085/GB | **$0** (프리티어 내) |
| **Rekognition** | 5,000 이미지/월 (12개월) | $0.001/이미지 | **$0** (프리티어 내) |
| **Bedrock (Titan Embeddings)** | 프리티어 없음 | $0.00006/이미지 | **$0.06** (1,000이미지) |
| **OpenSearch Serverless** | **프리티어 없음** | $0.24/OCU/시간 (최소 2 OCU) | **$173~$346/월** |
| **CloudWatch** | 5GB 로그, 10 메트릭 | 소량 | **$0** (프리티어 내) |
| **Cognito** | 50,000 MAU (프리티어) | - | **$0** (프리티어 내) |

### 6.2 비용 최적화 전략

**OpenSearch Serverless가 최대 비용 요인** (최소 2 OCU = ~$346/월)

#### Option A: OpenSearch Serverless 사용 (권장 - 학습 목적)
- 월 비용: ~$346 (최소 OCU 기준)
- 15주간 총 비용: ~$1,200~$1,400
- AWS 학생 크레딧으로 커버 가능 여부 확인 필요
- **Half OCU 적용 시**: ~$173/월, 총 ~$600~$700

#### Option B: OpenSearch Serverless 대안 - 자체 벡터 DB (비용 절감)
- **EC2 (t3.micro) + PostgreSQL + pgvector** 확장
  - 프리티어 내 (t3.micro 750시간/월)
  - 총 비용: ~$0 (프리티어)
  - 단점: 서버리스가 아님, 관리 부담 증가

#### Option C: 혼합 접근 (현실적 권장)
- **개발/테스트 기간** (1~12주): pgvector on EC2 (프리티어)
- **최종 데모** (13~15주): OpenSearch Serverless 활성화 (3주만 = ~$260)
- 총 비용: ~$260 + Bedrock 소량 = **~$270**

### 6.3 비용 요약

| 시나리오 | 월간 비용 | 15주 총 비용 | 적합도 |
|----------|----------|-------------|--------|
| Option A (AOSS 풀타임) | ~$346 | ~$1,400 | 크레딧 충분 시 |
| Option A (Half OCU) | ~$173 | ~$700 | 크레딧 $200+ 시 |
| Option B (EC2+pgvector) | ~$0 | ~$10 | 비용 최소화 |
| **Option C (혼합)** | 가변 | **~$270** | **가장 현실적** |

> **멘토에게 확인할 사항**: AWS 크레딧 규모, OpenSearch Serverless 사용 가능 여부, 비용 지원 한도

---

## 7. 위험 요소와 대응 방안

### 7.1 기술적 리스크

| 리스크 | 영향도 | 발생 가능성 | 대응 방안 |
|--------|--------|-----------|----------|
| **OpenSearch Serverless 설정 복잡도** | High | High | 3~4주차에 집중 투자, 공식 Workshop 자료 활용, 멘토 지원 요청 |
| **Bedrock Model Access 승인 지연** | High | Medium | 1주차에 즉시 요청, 대안으로 Rekognition 라벨 기반 검색 먼저 구현 |
| **OpenSearch Serverless 비용 초과** | High | High | Option C(혼합) 전략 사용, 미사용 시 컬렉션 삭제 |
| **Lambda Cold Start 지연** | Medium | Medium | Provisioned Concurrency(추가 비용) 또는 Warm-up 전략 |
| **대용량 이미지 처리 타임아웃** | Medium | Medium | 이미지 리사이즈 전처리, Lambda 타임아웃 60초 설정, 이미지 크기 제한(10MB) |
| **리전별 서비스 가용성** | Medium | Low | us-east-1(버지니아) 사용 권장 (모든 서비스 가용) |
| **CORS/Presigned URL 이슈** | Low | High | 초기에 업로드 파이프라인 검증, 디버깅 시간 확보 |

### 7.2 프로젝트 리스크

| 리스크 | 영향도 | 발생 가능성 | 대응 방안 |
|--------|--------|-----------|----------|
| 팀원 이탈/불참 | High | Low | 모든 코드 GitHub 관리, 문서화 철저 |
| 일정 지연 | Medium | Medium | MVP 우선, 고도화는 버퍼 활용 |
| AWS 크레딧 소진 | High | Medium | 비용 모니터링 알람 설정, Option C 전략 |
| 학기 중 다른 과목 부담 | Medium | High | 주차별 마일스톤 명확화, 최소 주 2회 작업 |

### 7.3 대응 우선순위

```
[Critical - 즉시 대응]
1. Bedrock Model Access 요청 (1주차)
2. OpenSearch Serverless 비용 전략 확정 (2주차)
3. AWS 크레딧 규모 확인 (1주차)

[Important - 2주 내 대응]
4. 리전 결정 (us-east-1 권장)
5. IAM 정책 설계
6. S3 CORS + Presigned URL 테스트

[Nice to have - 진행하며 대응]
7. CloudWatch 알람 설정
8. Lambda 성능 최적화
```

---

## 8. 팀 역할 분담

### 8.1 2인 구성 (최소)

| 역할 | 담당 영역 | 필요 스킬 |
|------|----------|----------|
| **팀원 A: 백엔드/인프라 엔지니어** | Lambda 함수 개발(Python), IAM 정책, OpenSearch 설정, Rekognition/Bedrock 연동, CloudWatch 모니터링 | Python, AWS SDK(boto3), REST API, 인프라 설정 |
| **팀원 B: 프론트엔드/통합 엔지니어** | React 웹 앱 개발, S3 업로드 구현, API 연동, CloudFront 배포, UI/UX 디자인, 발표 자료 | JavaScript/React, HTML/CSS, S3 Presigned URL, 발표 |

**공통 담당**: 아키텍처 설계, 코드 리뷰, 테스트, 문서화

### 8.2 3인 구성 (권장)

| 역할 | 담당 영역 |
|------|----------|
| **팀원 A: 인프라/AI 엔지니어** | AWS 인프라 설정(S3, Lambda, API GW, IAM), Rekognition/Bedrock 연동, OpenSearch Serverless 구축 |
| **팀원 B: 백엔드 개발자** | Lambda 함수 구현(Pipeline, Search, Upload), 데이터 파이프라인, 에러 핸들링, 테스트 |
| **팀원 C: 프론트엔드 개발자** | React 앱 개발, UI/UX, CloudFront 배포, 발표 자료, 문서화 |

### 8.3 협업 도구

| 도구 | 용도 |
|------|------|
| GitHub | 코드 관리, PR 리뷰, Issues |
| Discord/Slack | 실시간 커뮤니케이션 |
| Notion/Google Docs | 문서 공유, 주간 회의록 |
| AWS CloudWatch | 공동 모니터링 대시보드 |

---

## 9. 차별화 포인트

### 9.1 기술적 특장점

| 차별화 요소 | 설명 | 어필 포인트 |
|------------|------|------------|
| **멀티모달 벡터 검색** | 텍스트 -> 이미지, 이미지 -> 이미지 교차 검색 | 단순 키워드 매칭이 아닌 의미론적(semantic) 유사도 검색 |
| **완전 서버리스 아키텍처** | EC2 없이 Lambda + 관리형 서비스만으로 구성 | 운영 비용 최소화, 자동 스케일링, Zero Admin |
| **이벤트 드리븐 파이프라인** | S3 업로드 -> 자동 분석 -> 자동 인덱싱 | 실시간 처리, 느슨한 결합, 확장 용이 |
| **하이브리드 검색** | 벡터 유사도 + 라벨 필터링 결합 | 검색 정확도 향상, 사용자 맞춤 필터링 |
| **AWS AI 서비스 통합** | Rekognition + Bedrock 2개 AI 서비스 연계 | ML 모델 학습 없이 프로덕션 수준 AI 구현 |

### 9.2 발표 시 강조할 포인트

1. **"ML 모델 학습 없이 프로덕션급 AI"**: Rekognition과 Bedrock을 조합하여 자체 모델 훈련 없이도 고품질 이미지 분석/검색 구현
2. **"이미지의 의미를 이해하는 검색"**: 기존 텍스트 기반 태그 매칭과 달리, 시각적 유사성을 벡터 공간에서 측정하는 시맨틱 검색
3. **"확장 가능한 서버리스 설계"**: 0에서 수천 사용자까지 자동 확장, 사용한 만큼만 비용 발생
4. **"실시간 이벤트 드리븐 처리"**: 업로드 즉시 AI 분석이 시작되는 비동기 파이프라인

### 9.3 데모 시나리오 (인상적인 데모 흐름)

```
데모 1: "여행 사진 자동 분류"
  1. 여행 사진 10장 업로드 (해변, 음식, 도시, 자연 등)
  2. 자동 태깅 결과 확인 (실시간 라벨 표시)
  3. "해변" 라벨로 필터링 -> 해변 사진만 표시
  4. 특정 해변 사진으로 유사 이미지 검색 -> 비슷한 풍경 사진 검색

데모 2: "텍스트로 이미지 찾기"
  1. 사전에 100장 이미지 인덱싱
  2. "sunset over ocean" 텍스트 입력
  3. 일몰/바다 관련 이미지가 유사도 순으로 표시
  4. 한국어 "강아지 산책" 입력 -> 관련 이미지 검색

데모 3: "비슷한 상품 찾기" (e-commerce 활용 시나리오)
  1. 상품 이미지 업로드
  2. 유사 상품 이미지 자동 추천
  3. 카테고리 라벨로 추가 필터링
```

---

## 10. MVP 정의 (중간고사 8주차 기준)

### 10.1 MVP 범위 (Must Have)

| 기능 | 상세 | 완료 기준 |
|------|------|----------|
| **이미지 업로드** | S3 Presigned URL 기반 이미지 업로드 | API로 업로드 성공, S3에 저장 확인 |
| **자동 라벨링** | S3 이벤트 -> Lambda -> Rekognition | 업로드된 이미지에 자동 라벨 추출 |
| **벡터 임베딩 생성** | Lambda -> Bedrock Titan Embeddings | 1024차원 벡터 생성 및 저장 |
| **벡터 인덱싱** | OpenSearch Serverless에 문서 저장 | 이미지 메타데이터 + 벡터 인덱싱 |
| **유사 이미지 검색 (API)** | kNN 검색 API | 쿼리 이미지 기반 유사 이미지 Top-K 반환 |
| **간단한 웹 UI** | 업로드 폼 + 검색 결과 리스트 | 최소한의 동작 가능한 UI |

### 10.2 MVP 이후 고도화 (Nice to Have)

| 기능 | 우선순위 | 예상 주차 |
|------|---------|----------|
| 텍스트 -> 이미지 검색 | P1 | 9주차 |
| 하이브리드 검색 (벡터+라벨) | P1 | 10주차 |
| 프론트엔드 갤러리 UI | P1 | 9-10주차 |
| 라벨 태그 클라우드/필터 | P2 | 10주차 |
| 사용자 인증 (Cognito) | P2 | 11주차 |
| 유사도 시각화 (점수 표시) | P2 | 12주차 |
| 이미지 클러스터링 시각화 | P3 | 12주차 |
| 배치 업로드 (ZIP) | P3 | 12주차 |
| CloudFront CDN 배포 | P2 | 11주차 |
| 비용 대시보드 | P3 | 13주차 |

### 10.3 MVP 아키텍처 (최소 구성)

```
[MVP - 8주차까지]

  사용자 (로컬 React 앱)
       |
       v
  API Gateway (HTTP API)
    |          |
    v          v
  Lambda     Lambda
  (Upload)   (Search)
    |          |
    v          |
  S3 ──Event──> Lambda(Pipeline) ──> Rekognition
  (Images)                       ──> Bedrock (Titan)
                                 ──> OpenSearch Serverless (또는 pgvector)

* CloudFront, Cognito, 고도화 UI는 MVP 이후
* React 앱은 로컬 개발 서버(localhost)에서 API Gateway 호출
```

### 10.4 MVP 완료 체크리스트

- [ ] S3 버킷 생성 및 이벤트 알림 설정
- [ ] Lambda 함수 3개 배포 (Upload, Pipeline, Search)
- [ ] API Gateway HTTP API 배포
- [ ] Rekognition detectLabels 연동
- [ ] Bedrock Titan Embeddings 연동
- [ ] OpenSearch Serverless (또는 pgvector) 벡터 인덱스 설정
- [ ] E2E 테스트: 이미지 업로드 -> 자동 분석 -> 벡터 검색 성공
- [ ] 간단한 웹 UI에서 업로드 + 검색 동작 확인
- [ ] 중간 발표 자료 준비

---

## 부록 A: 기술 스택 요약

### 백엔드

| 항목 | 선택 | 이유 |
|------|------|------|
| 런타임 | Python 3.12 (Lambda) | boto3 네이티브 지원, AI/ML 생태계 풍부 |
| Upload Lambda | Node.js 20.x | S3 Presigned URL 생성에 AWS SDK v3 활용 |
| API 형식 | REST (API Gateway HTTP API) | 비용 효율적, 충분한 기능 |
| 벡터 DB | OpenSearch Serverless (Option A) 또는 pgvector (Option B) | 프로젝트 예산에 따라 결정 |

### 프론트엔드

| 항목 | 선택 | 이유 |
|------|------|------|
| 프레임워크 | React 18 + Vite | 빠른 개발, SPA 적합 |
| 스타일링 | Tailwind CSS | 빠른 UI 개발, 반응형 |
| HTTP 클라이언트 | fetch API | 추가 의존성 불필요 |
| 상태 관리 | React useState/useContext | 앱 규모 대비 충분 |
| 이미지 업로드 | react-dropzone | 드래그앤드롭 UX |

### 인프라

| 항목 | 선택 | 이유 |
|------|------|------|
| IaC | AWS SAM 또는 CloudFormation | Lambda 배포에 최적화 |
| CI/CD | GitHub Actions | 무료, GitHub 통합 |
| 모니터링 | CloudWatch | AWS 네이티브, 추가 비용 없음 |
| 리전 | us-east-1 | 모든 서비스 가용, Bedrock 모델 최다 |

---

## 부록 B: 핵심 API 명세 (Draft)

### POST /api/upload
```
Request:  { fileName: string, contentType: string }
Response: { uploadUrl: string, fields: object, imageKey: string }
```

### POST /api/search/image
```
Request:  { image: string (base64), k: number (default 10) }
Response: { results: [{ imageUrl, labels, score, metadata }] }
```

### POST /api/search/text
```
Request:  { query: string, k: number (default 10) }
Response: { results: [{ imageUrl, labels, score, metadata }] }
```

### GET /api/images?page=1&limit=20
```
Response: { images: [{ imageUrl, labels, uploadedAt }], total: number }
```

### GET /api/labels
```
Response: { labels: [{ name: string, count: number }] }
```

### GET /api/images/:id
```
Response: { imageUrl, labels, labelConfidences, metadata, similarImages }
```

---

## 부록 C: 참고 자료

- [AWS Rekognition DetectLabels](https://docs.aws.amazon.com/rekognition/latest/dg/labels-detect-labels-image.html)
- [Amazon Titan Multimodal Embeddings](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-multiemb-models.html)
- [OpenSearch Serverless Vector Search](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-vector-search.html)
- [S3 Event Notifications to Lambda](https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html)
- [S3 Presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)

---

## 버전 히스토리

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1 | 2026-03-13 | 초안 작성 | CTO Team |
