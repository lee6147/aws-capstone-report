# AI 학습 도우미 챗봇 (RAG 기반) - 아키텍처 설계 문서

> **Summary**: 학생이 교과서/강의노트 PDF를 업로드하면 해당 자료 기반으로 질문에 답변하는 RAG 챗봇
>
> **Project**: 국민대학교 AWS-양자통신융합전공 캡스톤 (AWS실전프로젝트I)
> **Author**: CTO Lead (bkit)
> **Date**: 2026-03-13
> **Status**: Draft (수업계획서 정합성 검토 후 수정)

---

## Executive Summary

| 관점 | 내용 |
|------|------|
| **Problem** | 청소년/학생이 교과서나 강의노트 내용을 복습할 때 즉각적인 질의응답 도구가 없어 학습 효율이 떨어진다 |
| **Solution** | PDF 문서를 업로드하면 RAG(Retrieval-Augmented Generation) 파이프라인으로 청킹-임베딩-벡터검색 후, Bedrock LLM이 근거 기반 답변을 생성하는 서버리스 챗봇 |
| **Function/UX Effect** | 문서 업로드 후 즉시 대화형 Q&A 가능, 답변마다 출처(페이지/문단) 표시로 신뢰성 확보 |
| **Core Value** | AWS 완전관리형 서버리스 아키텍처로 운영 부담 최소화, 청소년 교육 접근성 향상 (공적가치) |

---

## 1. 시스템 아키텍처 다이어그램

### 1.1 전체 서비스 흐름

```
                           [사용자 (브라우저)]
                                  |
                                  v
                    ┌─────────────────────────┐
                    │   CloudFront (CDN)       │
                    │   + S3 정적 호스팅       │
                    │   (React SPA)            │
                    └────────────┬────────────┘
                                 |
                                 v
                    ┌─────────────────────────┐
                    │   API Gateway (REST)     │
                    │   - /upload              │
                    │   - /chat                │
                    │   - /documents           │
                    └────────┬───────┬────────┘
                             |       |
                    ┌────────┘       └────────┐
                    v                         v
       ┌────────────────────┐   ┌────────────────────┐
       │  Lambda: upload     │   │  Lambda: chat       │
       │  (문서 처리)        │   │  (질문 응답)        │
       └────────┬───────────┘   └──┬──────────┬──────┘
                |                   |          |
                v                   v          v
       ┌────────────────┐  ┌──────────┐  ┌──────────────┐
       │  S3 Bucket      │  │ Bedrock  │  │ Bedrock      │
       │  (원본 PDF)     │  │ Titan    │  │ Claude 3     │
       └────────┬───────┘  │ Embedding│  │ Haiku/Sonnet │
                |           └────┬─────┘  └──────────────┘
                v                |
       ┌────────────────┐       v
       │ Bedrock         │  ┌──────────────────┐
       │ Knowledge Bases │──│ S3 Vectors       │
       │ (자동 청킹 +    │  │ (벡터 저장/검색) │
       │  임베딩 처리)   │  └──────────────────┘
       └────────────────┘
```

### 1.2 핵심 설계 결정

| 결정 사항 | 선택 | 대안 | 선택 근거 |
|-----------|------|------|-----------|
| 벡터 DB | **S3 Vectors** | OpenSearch Serverless | OpenSearch Serverless 최소 월 $350 vs S3 Vectors 사용량 기반 과금 (캡스톤 규모에서 월 $1 미만). [AWS 공식 문서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-bedrock-kb.html)에서 Bedrock KB 연동 지원 확인. **단, 3주차에 us-east-1에서 직접 PoC 필수** |
| RAG 파이프라인 | **Bedrock Knowledge Bases** | 직접 구현 (LangChain) | 청킹/임베딩/인덱싱 자동 관리, 코드량 80% 감소, 2~3인 팀에 적합 |
| LLM | **Claude 3 Haiku** (기본) | Claude Sonnet, Titan | Haiku가 토큰당 비용 최저이면서 학습 Q&A에 충분한 품질 |
| 임베딩 모델 | **Titan Text Embeddings V2** | Cohere Embed | AWS 네이티브 통합, $0.00011/1K 토큰 최저가 |
| 프론트엔드 | **React SPA (Vite)** | Next.js | 정적 호스팅만 필요, SSR 불필요, 빌드 단순 |

---

## 2. AWS 서비스 구성

### 2.1 서비스별 역할과 연결 방식

| 서비스 | 역할 | 연결 대상 | 비고 |
|--------|------|-----------|------|
| **S3 (정적 호스팅)** | React 앱 호스팅 | CloudFront | 빌드된 정적 파일 서빙 |
| **CloudFront** | CDN + HTTPS | S3, API Gateway | 커스텀 도메인, SSL 인증서 |
| **API Gateway** | REST API 엔드포인트 | Lambda 함수들 | CORS 설정, 요청 검증 |
| **Lambda: upload** | 문서 업로드 처리 | S3 (문서 버킷), Bedrock KB | PDF 저장 후 KB 동기화 트리거 |
| **Lambda: chat** | 질문-답변 처리 | Bedrock KB (RetrieveAndGenerate) | 벡터 검색 + LLM 답변 생성 통합 API |
| **Lambda: documents** | 문서 목록/삭제 관리 | S3 (문서 버킷) | 업로드된 문서 CRUD |
| **S3 (문서 버킷)** | 원본 PDF 저장 | Bedrock Knowledge Bases | 데이터 소스로 연결 |
| **Bedrock Knowledge Bases** | RAG 파이프라인 자동화 | S3 (문서), S3 Vectors, Titan Embedding | 청킹+임베딩+인덱싱 자동 처리 |
| **S3 Vectors** | 벡터 저장/검색 | Bedrock Knowledge Bases | 비용 효율적 벡터 스토어 |
| **Bedrock (Titan Embedding V2)** | 텍스트 임베딩 생성 | Bedrock Knowledge Bases | 1024차원 임베딩 |
| **Bedrock (Claude 3 Haiku)** | 답변 생성 LLM | Lambda: chat | RetrieveAndGenerate API 내부에서 호출 |
| **IAM** | 권한 관리 | 모든 서비스 | 최소 권한 원칙 |
| **CloudWatch** | 모니터링/로깅 | Lambda, API Gateway | 비용 알림 설정 필수 |

### 2.2 서비스 연결 아키텍처

```
[CloudFront] ─── Origin ──→ [S3: 정적 호스팅]
     |
     └─── Origin ──→ [API Gateway]
                          |
                          ├─── /api/upload  ──→ [Lambda: upload]  ──→ [S3: 문서] ──→ [Bedrock KB 동기화]
                          |                                                              |
                          ├─── /api/chat    ──→ [Lambda: chat]    ──→ [Bedrock KB: RetrieveAndGenerate]
                          |                                              |              |
                          |                                         [S3 Vectors]   [Claude Haiku]
                          |                                         (벡터 검색)    (답변 생성)
                          |
                          └─── /api/documents ──→ [Lambda: documents] ──→ [S3: 문서]
```

---

## 3. 데이터 파이프라인

### 3.1 문서 업로드 파이프라인

```
[사용자]
   │  PDF 파일 선택 (최대 10MB)
   v
[프론트엔드]
   │  1. Presigned URL 요청 (POST /upload/presign)
   │  2. S3 직접 업로드 (PUT presigned URL)
   │  3. 업로드 완료 알림 (POST /upload/complete)
   v
[Lambda: upload]
   │  4. 메타데이터 저장 (MVP: S3 Object Metadata, 고도화: DynamoDB)
   │  5. Bedrock KB 데이터 소스 동기화 시작
   v
[Bedrock Knowledge Bases]
   │  6. PDF 텍스트 추출 (내장 파서)
   │  7. 청킹 (Semantic Chunking, 300~500 토큰)
   │  8. Titan Embedding V2로 벡터 변환 (1024차원)
   │  9. S3 Vectors에 인덱싱
   v
[S3 Vectors]
   │  벡터 + 메타데이터 저장 완료
   v
[완료 상태 반환 → 프론트엔드]
```

### 3.2 질문-답변 파이프라인

```
[사용자]
   │  질문 입력: "광합성의 과정을 설명해줘"
   v
[프론트엔드]
   │  POST /chat { question, knowledgeBaseId }
   v
[Lambda: chat]
   │  Bedrock KB RetrieveAndGenerate API 호출
   v
[Bedrock Knowledge Bases]
   │  1. 질문 텍스트 → Titan Embedding V2 → 질문 벡터
   │  2. S3 Vectors에서 유사 벡터 Top-K 검색 (k=5)
   │  3. 검색된 청크들을 프롬프트에 컨텍스트로 삽입
   │  4. Claude 3 Haiku에게 답변 생성 요청
   v
[Claude 3 Haiku]
   │  컨텍스트 기반 답변 생성 + 출처 정보 포함
   v
[Lambda: chat]
   │  응답 포맷팅: { answer, sources: [{page, content}] }
   v
[프론트엔드]
   │  답변 표시 + 출처(페이지 번호) 하이라이트
   v
[사용자]
```

### 3.3 데이터 모델

```
문서 메타데이터 (S3 Object Metadata 또는 간단 JSON):
{
  "documentId": "uuid",
  "fileName": "생물학_교과서.pdf",
  "uploadedAt": "2026-04-15T10:30:00Z",
  "fileSize": 5242880,
  "pageCount": 120,
  "status": "indexed" | "processing" | "failed",
  "knowledgeBaseId": "kb-xxx"
}

채팅 요청/응답:
Request:  { "question": "광합성의 과정을 설명해줘", "documentId": "uuid" }
Response: {
  "answer": "광합성은 크게 명반응과 암반응으로 나뉩니다...",
  "sources": [
    { "page": 45, "content": "명반응은 틸라코이드 막에서...", "score": 0.92 },
    { "page": 47, "content": "암반응(캘빈 회로)은...", "score": 0.87 }
  ]
}
```

---

## 4. 구현 난이도 분석

### 4.1 컴포넌트별 난이도

| 컴포넌트 | 난이도 | 설명 | 예상 소요 시간 |
|----------|:------:|------|:-------------:|
| S3 정적 호스팅 + CloudFront | ★☆☆☆☆ | AWS 콘솔에서 클릭 몇 번으로 설정 완료 | 2시간 |
| React 프론트엔드 (채팅 UI) | ★★☆☆☆ | 파일 업로드 + 채팅 인터페이스, 컴포넌트 5~6개 | 1주 |
| API Gateway + Lambda 기본 설정 | ★★☆☆☆ | REST API 3개 엔드포인트, SAM/CDK 템플릿 활용 | 3일 |
| S3 Presigned URL 업로드 | ★★☆☆☆ | Lambda에서 presigned URL 생성, 프론트에서 직접 업로드 | 1일 |
| **Bedrock Knowledge Bases 설정** | ★★★☆☆ | 콘솔에서 데이터소스+벡터스토어 연결, IAM 권한 설정이 핵심 | 3일 |
| **Bedrock KB 동기화 자동화** | ★★★☆☆ | 문서 업로드 시 StartIngestionJob API 호출 연동 | 2일 |
| **RetrieveAndGenerate API 연동** | ★★★☆☆ | Lambda에서 Bedrock KB API 호출, 응답 파싱 | 2일 |
| S3 Vectors 설정 | ★★☆☆☆ | Bedrock KB가 자동 생성/관리, 직접 설정 최소 | 1일 |
| 출처 표시 기능 | ★★★☆☆ | 검색된 청크의 메타데이터(페이지 번호) 추출 및 매핑 | 2일 |
| IAM 권한 설정 | ★★★★☆ | 서비스 간 권한 체인이 복잡, 최소 권한 원칙 적용 | 2일 |
| 에러 핸들링 + 모니터링 | ★★★☆☆ | Lambda 타임아웃, Bedrock 스로틀링 대응 | 2일 |
| CI/CD (GitHub Actions) | ★★☆☆☆ | S3 배포 자동화, Lambda 코드 업데이트 | 1일 |

### 4.2 난이도 요약

| 영역 | 평균 난이도 | 팀 역량 요구 |
|------|:----------:|-------------|
| 인프라 (S3, CloudFront, API GW) | ★★☆☆☆ | AWS 콘솔 기본 조작 |
| 백엔드 (Lambda + Bedrock) | ★★★☆☆ | Python/Node.js, AWS SDK, IAM 이해 |
| RAG 파이프라인 (Bedrock KB) | ★★★☆☆ | Bedrock Knowledge Bases API 학습 필요 |
| 프론트엔드 (React) | ★★☆☆☆ | React 기초, 파일 업로드, 채팅 UI |
| 통합 + 배포 | ★★★☆☆ | 서비스 간 연결, 디버깅 경험 |

> **핵심 포인트**: Bedrock Knowledge Bases를 활용하면 청킹/임베딩/벡터검색을 직접 구현할 필요가 없어 난이도가 크게 감소한다. 직접 LangChain으로 구현할 경우 난이도가 ★★★★☆~★★★★★로 상승한다.

---

## 5. 15주 타임라인

### 5.1 주차별 개발 계획

| 주차 | 날짜 | 단계 | 주요 활동 | 산출물 |
|:----:|------|------|-----------|--------|
| **1** | 03/06 | 오리엔테이션 | 팀 빌딩, AWS 계정 설정, Bedrock Model Access 요청 | 팀 구성표, AWS 계정 |
| **2** | 03/13 | 주제 선정 + 팀 구성 | 아이디어 탐색 (3안 비교), 주제 선정 | 아이디어 비교 분석 |
| **3** | 03/20 | 문제 정의 + 요구사항 | 요구사항 정의, **S3 Vectors + Bedrock KB PoC 검증**, 출처(페이지) 추출 검증 | PoC 결과, 요구사항 명세 |
| **4** | 03/27 | AWS 아키텍처 설계 | 아키텍처 설계 확정, SAM 템플릿 기초, IAM 설계 | **아키텍처 설계서** |
| **5** | 04/03 | 데이터 전략 수립 | Bedrock KB 생성, S3 Vectors 연결, 인프라 구축 | KB 동기화 성공 |
| **6** | 04/10 | 시스템 구현 I | Lambda ×4 스캐폴딩, RetrieveAndGenerate 연동, 출처 표시 구현 | RAG Q&A 동작 |
| **7** | 04/17 | 시스템 구현 II | React 채팅 UI, 파일 업로드 UI, API 연동 | 프론트엔드 프로토타입 |
| **8** | 04/24 | **중간고사(MVP)** | 통합 테스트, 버그 수정, **중간고사 발표** | **MVP 데모: PDF 업로드 + 질문 응답** |
| **9** | 05/01 | 시스템 고도화 | 대화 이력, 다중 문서 지원, UI/UX 개선 | 고도화 기능 |
| **10** | 05/08 | **보안/운영 설계** | API Key/사용량 제한, IAM 최소 권한 검증, CloudWatch 비용 알림 | **보안 설계서** |
| **11** | 05/15 | **테스트/검증** | 사용자 테스트 (동료 학생), 성능 테스트, 에러 시나리오 검증 | **테스트 보고서** |
| **12** | 05/22 | **프로젝트 문서화** | 기술 문서 작성, 아키텍처 다이어그램 최종화, API 문서 | **기술 문서** |
| **13** | 05/29 | 발표 준비 | 최종 기능 정리, 발표 자료 작성 | 발표 자료 |
| **14** | 06/05 | 최종 멘토링 | 데모 영상 제작, Proserve 최종 점검 | 데모 영상 |
| **15** | 06/12 | **최종발표** | **기말 발표 + 데모** | 최종 보고서, 소스코드 |

### 5.2 마일스톤 요약

```
[Week 1-3]  기획 + PoC 검증
     |
[Week 4-5]  설계 + 인프라 구축  ──────── AWS 멘토 점검 (1차)
     |
[Week 6-7]  시스템 구현 (RAG + FE)
     |
[Week 8]    ★ MVP 중간발표 ★  ───────── AWS 멘토 점검 (2차)
     |
[Week 9]    고도화 (대화이력, 다중문서)
     |
[Week 10-12] 보안/테스트/문서화  ────── AWS 멘토 점검 (3차)
     |
[Week 13-15] ★ 발표 준비 + 최종발표 ★
```

---

## 6. 예상 비용

### 6.1 AWS 프리티어 활용 범위

| 서비스 | 프리티어 한도 | 프로젝트 예상 사용량 | 초과 여부 |
|--------|-------------|--------------------:|:---------:|
| **Lambda** | 100만 요청/월, 40만 GB-초 | ~5,000 요청/월 | 무료 |
| **API Gateway** | 100만 요청/월 (12개월) | ~5,000 요청/월 | 무료 |
| **S3 (스토리지)** | 5GB | ~2GB (PDF + 정적파일) | 무료 |
| **CloudFront** | 1TB 전송/월 | ~5GB/월 | 무료 |
| **CloudWatch** | 기본 모니터링 무료 | 기본 수준 | 무료 |

### 6.2 유료 서비스 예상 비용

| 서비스 | 단가 | 예상 사용량 | 월 비용 |
|--------|------|----------:|--------:|
| **Bedrock (Claude 3 Haiku)** | $0.25/1M 입력, $1.25/1M 출력 토큰 | ~50만 토큰/월 | ~$0.5 |
| **Bedrock (Titan Embedding V2)** | $0.00011/1K 토큰 | ~20만 토큰/월 | ~$0.02 |
| **S3 Vectors (스토리지)** | $0.06/GB/월 | ~0.5GB | ~$0.03 |
| **S3 Vectors (쿼리)** | $0.004/TB | ~5,000 쿼리/월 | ~$0.01 |
| **Bedrock KB (동기화)** | API 호출 비용에 포함 | ~50회/월 | 포함 |

### 6.3 월별 비용 요약

| 항목 | 월 비용 |
|------|--------:|
| 프리티어 서비스 (Lambda, API GW, S3, CloudFront) | $0 |
| Bedrock LLM (Claude 3 Haiku) | ~$0.50 |
| Bedrock Embedding (Titan V2) | ~$0.02 |
| S3 Vectors | ~$0.04 |
| 개발/테스트 (Bedrock 반복 호출, KB 동기화 등) | ~$0.50 |
| **월 합계** | **~$1.06** |
| **15주 (4개월) 총 비용** | **~$4.25** |

> **참고**: 개발 기간 중 반복 테스트 비용을 포함한 금액이다. 학생 크레딧 $100의 약 5% 수준.

> **비용 핵심 포인트**:
> - S3 Vectors를 사용함으로써 OpenSearch Serverless ($350/월) 대비 **99.9% 비용 절감**
> - Bedrock Knowledge Bases가 청킹/임베딩을 자동 처리하므로 별도 EC2 인스턴스 불필요
> - 학생 크레딧 $100 기준, 15주 총 비용 $4~5 (개발/테스트 포함)로 크레딧의 5% 수준
> - 나머지 크레딧으로 Sonnet 모델 테스트, 추가 기능 실험 여유 충분
>
> **비용 폭주 방지**: CloudWatch 예산 알림을 $10, $30, $50 단계로 설정

---

## 7. 위험 요소

### 7.1 기술적 리스크와 대응 방안

| # | 위험 요소 | 영향도 | 발생 확률 | 대응 방안 |
|:-:|-----------|:------:|:---------:|-----------|
| 1 | **Bedrock 리전 가용성**: 사용 리전에서 Bedrock/KB 미지원 | 높음 | 중간 | us-east-1 (버지니아) 사용 권장. 모든 Bedrock 기능 최우선 지원 리전 |
| 2 | **IAM 권한 복잡도**: Lambda→Bedrock→S3 Vectors 간 권한 체인 | 중간 | 높음 | AWS 멘토에게 IAM 설정 리뷰 요청, 초기에 넓은 권한 → 점진적 축소 |
| 3 | **PDF 파싱 품질**: 수식, 표, 이미지가 포함된 교과서 PDF 파싱 실패 | 중간 | 중간 | Bedrock KB 내장 파서 먼저 테스트, 품질 낮으면 Textract 병행 검토 |
| 4 | **Lambda Cold Start**: 첫 요청 시 3~5초 지연 | 낮음 | 높음 | Provisioned Concurrency 불필요 (학생 프로젝트), UX에서 로딩 상태 표시 |
| 5 | **Bedrock 스로틀링**: 동시 요청 제한 초과 | 낮음 | 낮음 | 캡스톤 규모에서 거의 발생 안 함, 프론트에서 재시도 로직 추가 |
| 6 | **비용 폭주**: 예상 외 API 호출 급증 | 중간 | 중간 | CloudWatch 예산 알림, **API Key/사용량 제한** (10주차 보안 설계 시 적용) |
| 7 | **팀원 이탈/일정 지연**: 2~3인 소규모 팀 리스크 | 높음 | 중간 | MVP 먼저 확보 (8주차), 역할 중복 학습으로 대체 가능성 확보 |
| 8 | **출처(페이지 번호) 추출 불가**: RetrieveAndGenerate가 페이지 메타데이터 미반환 | 중간 | 중간 | **3주차 PoC에서 반드시 검증**, 불가 시 청크 텍스트 + S3 URI로 대체 |
| 9 | **인증 없는 API 악용**: 공개 API로 Bedrock 무단 호출 가능 | 중간 | 중간 | MVP는 허용, **10주차 보안 설계 시 API Key + throttling 적용** |

### 7.2 기술 의존성 리스크

| 의존 서비스 | 대체 가능 여부 | 대체 방안 |
|-------------|:-----------:|-----------|
| Bedrock Knowledge Bases | 가능 | LangChain + FAISS (Lambda 패키징), 난이도 상승 |
| S3 Vectors | 가능 | pgvector (RDS Free Tier), 또는 FAISS 인메모리 |
| Claude 3 Haiku | 가능 | Titan Text Express, 또는 다른 Bedrock 모델 |
| Titan Embedding V2 | 가능 | Cohere Embed (Bedrock), 비용 약간 상승 |

---

## 8. 팀 역할 분담

### 8.1 2인 팀 구성

| 역할 | 담당 영역 | 주요 작업 |
|------|-----------|-----------|
| **A: 백엔드 + 인프라** | AWS 서비스 설정, Lambda, Bedrock 연동 | S3/CloudFront 설정, Lambda 함수 구현, Bedrock KB 설정, IAM 관리, CI/CD |
| **B: 프론트엔드 + RAG** | React UI, Bedrock KB 파이프라인 튜닝 | 채팅 UI, 파일 업로드, 출처 표시, 프롬프트 엔지니어링, 사용자 테스트 |
| **공통** | 설계, 테스트, 문서화 | 아키텍처 설계, 통합 테스트, 발표 자료 |

### 8.2 3인 팀 구성

| 역할 | 담당 영역 | 주요 작업 |
|------|-----------|-----------|
| **A: 인프라 리드** | AWS 인프라, 배포, 보안 | S3/CloudFront/API GW 설정, IAM, CI/CD, 비용 모니터링 |
| **B: 백엔드 리드** | Lambda, Bedrock, RAG 파이프라인 | Lambda 함수, Bedrock KB 설정, RetrieveAndGenerate 연동, 프롬프트 튜닝 |
| **C: 프론트엔드 리드** | React UI/UX, 사용자 경험 | 채팅 인터페이스, 파일 업로드, 출처 표시, 반응형, 사용자 테스트 |
| **공통** | 설계, 통합, 발표 | 아키텍처 설계, 통합 테스트, 발표 자료, 기술 문서 |

### 8.3 주차별 협업 포인트

| 주차 | 병렬 작업 | 통합 포인트 |
|:----:|-----------|------------|
| 3~4 | 인프라(A) / 백엔드 스캐폴딩(B) | API 인터페이스 합의 |
| 5~6 | Bedrock KB 설정(B) / Lambda 구현(A) | RAG 엔드투엔드 테스트 |
| 7 | 프론트엔드(B/C) / API 연동(A) | 풀 스택 통합 |
| 8 | **전원: MVP 통합 + 발표 준비** | 중간 발표 |
| 9~11 | 기능별 분담 (병렬) | 주 1회 통합 테스트 |

---

## 9. 차별화 포인트

### 9.1 기술적 차별화

| # | 차별화 요소 | 설명 | 어필 포인트 |
|:-:|------------|------|------------|
| 1 | **S3 Vectors (신기술)** | 2025년 12월 GA된 최신 서비스, 기존 OpenSearch Serverless 대비 99% 비용 절감 | "최신 AWS 서비스를 실전에 적용한 사례" |
| 2 | **Bedrock Knowledge Bases** | 완전관리형 RAG 파이프라인, 인프라 관리 없이 엔터프라이즈급 RAG 구축 | "서버리스 아키텍처로 운영 비용 제로에 가까운 교육 플랫폼" |
| 3 | **출처 기반 답변** | 모든 답변에 원문 페이지/문단 표시, LLM 환각(hallucination) 검증 가능 | "신뢰할 수 있는 AI 답변 - 근거 추적 기능" |
| 4 | **완전 서버리스** | EC2 인스턴스 0개, 모든 컴포넌트 서버리스 | "24/7 가용성, 사용한 만큼만 과금" |

### 9.2 비즈니스 차별화 (공적가치: 청소년 교육)

| # | 차별화 요소 | 설명 |
|:-:|------------|------|
| 1 | **교육 접근성** | 과외/학원 없이도 교과서 기반 즉각 Q&A 가능 |
| 2 | **자기주도 학습 지원** | 업로드한 자료 기반 퀴즈 자동 생성, 학습 요약 |
| 3 | **비용 효율성** | 월 $1 미만 운영 비용으로 지속 가능한 교육 서비스 |
| 4 | **개인정보 보호** | MVP는 단일 사용자 가정. 고도화 시 S3 prefix 기반 사용자 격리 + Bedrock KB 메타데이터 필터링(`vectorSearchConfiguration.filter`) 적용 |

### 9.3 심사 어필 포인트 정리

```
1. AWS 최신 서비스 활용도: S3 Vectors (2025.12 GA) + Bedrock Knowledge Bases
2. 서버리스 완전체: Lambda + API GW + S3 + CloudFront + Bedrock = EC2 제로
3. 비용 효율성 극대화: 월 $1 미만 운영 (OpenSearch 대비 99.9% 절감)
4. 공적가치 실현: 청소년 학습 격차 해소를 위한 AI 튜터
5. 확장 가능성: 다국어 지원, 음성 질문(Transcribe), 이미지 인식(Rekognition)
```

---

## 10. MVP 정의 (8주차 중간고사)

### 10.1 MVP 필수 기능

| # | 기능 | 상세 | 우선순위 |
|:-:|------|------|:--------:|
| 1 | PDF 업로드 | 단일 PDF 파일 업로드 (최대 10MB) | P0 |
| 2 | 문서 인덱싱 | 업로드된 PDF를 Bedrock KB가 자동 처리 | P0 |
| 3 | 질문-답변 | 업로드된 문서 기반 텍스트 질문에 AI 답변 | P0 |
| 4 | 출처 표시 | 답변에 참조한 원문 청크 표시 | P0 |
| 5 | 기본 채팅 UI | 메시지 입력/표시, 로딩 상태 | P0 |

### 10.2 MVP 제외 기능 (8주차 이후)

| # | 기능 | 구현 시기 |
|:-:|------|:---------:|
| 1 | 대화 이력 유지 | 9주차 |
| 2 | 다중 문서 지원 | 9주차 |
| 3 | 문서 목록/삭제 관리 | 9주차 |
| 4 | 반응형 모바일 UI | 9주차 |
| 5 | 퀴즈 자동 생성 | 여유 시 (P3) |
| 6 | 학습 요약 기능 | 여유 시 (P3) |
| 7 | CI/CD 파이프라인 | 4~5주차 (인프라 구축 시) |

### 10.3 MVP 아키텍처 (최소 구성)

```
[React SPA]           [API Gateway]         [AWS 백엔드]
    |                      |                     |
    ├── 파일 업로드 ──→ /upload ──→ Lambda ──→ S3 + Bedrock KB 동기화
    |                      |                     |
    └── 질문 전송 ───→ /chat ───→ Lambda ──→ Bedrock KB RetrieveAndGenerate
                                                  |
                                            [S3 Vectors + Claude Haiku]
                                                  |
                                            답변 + 출처 반환
```

### 10.4 MVP 데모 시나리오

```
1. 브라우저에서 앱 접속 (CloudFront URL)
2. "생물학 교과서.pdf" 업로드 (10초 내 완료)
3. 인덱싱 대기 (1~2분, 진행 상태 표시)
4. 질문 입력: "세포 분열의 단계를 설명해줘"
5. AI 답변 표시 + 출처 (교과서 p.34, p.37 참조)
6. 추가 질문: "감수분열과 체세포분열의 차이는?"
7. AI 답변 표시 + 출처
```

---

## 부록 AA: 에러 핸들링 전략

| 시나리오 | 대응 | HTTP 코드 |
|----------|------|:---------:|
| PDF가 아닌 파일 업로드 | 프론트에서 MIME 타입 검증 + Lambda에서 이중 검증 | 400 |
| 10MB 초과 파일 | Presigned URL 생성 시 `Content-Length` 조건 설정 | 400 |
| Bedrock KB 동기화 실패 | 프론트에서 재시도 버튼 제공, CloudWatch 알림 | 500 |
| Bedrock API 스로틀링 | Lambda에서 exponential backoff (최대 3회) | 429→503 |
| API Gateway 29초 타임아웃 초과 | Lambda 타임아웃 25초 설정, 프롬프트 최적화로 응답 시간 단축 | 504 |
| Lambda 콜드 스타트 | UX에서 로딩 상태 표시 (3~5초 허용) | - |
| Presigned URL 만료 (15분) | 프론트에서 만료 감지 후 새 URL 요청 | 403→재요청 |
| 미인증 API 남용 | API Gateway API Key + 사용량 플랜 (일 100회 제한) | 429 |
| KB 인덱싱 지연 (>5분) | FE 폴링 간격 10초, 최대 대기 15분, 타임아웃 안내 표시 | — |

## 부록 AB: CORS 및 배포 전략

**접근 방식**: CloudFront에서 S3(정적)와 API Gateway를 **단일 도메인**으로 서빙
- `/api/*` 경로 → API Gateway Origin
- 그 외 → S3 Origin
- 이 방식으로 CORS 이슈 완전 회피 (동일 출처)

**환경 분리** (네이밍 컨벤션):
- S3 버킷: `studybot-{env}-documents` (dev/prod)
- Lambda: SAM template `Stage` 파라미터로 분리
- Bedrock KB: 개발용/프로덕션용 각 1개

## 부록 A: API 명세서

### A.1 엔드포인트 목록

| Method | Path | 설명 | 인증 |
|--------|------|------|:----:|
| POST | `/api/upload/presign` | Presigned URL 생성 | 불필요 (MVP) → 10주차 API Key |
| POST | `/api/upload/complete` | 업로드 완료 알림 + KB 동기화 트리거 | 불필요 (MVP) → 10주차 API Key |
| GET | `/api/upload/status/{jobId}` | KB 동기화 상태 조회 (GetIngestionJob) | 불필요 (MVP) → 10주차 API Key |
| POST | `/api/chat` | 질문-답변 | 불필요 (MVP) → 10주차 API Key |
| GET | `/api/documents` | 업로드 문서 목록 | 불필요 (MVP) → 10주차 API Key |
| DELETE | `/api/documents/{id}` | 문서 삭제 | 불필요 (MVP) → 10주차 API Key |

### A.2 주요 API 상세

#### POST /chat

```json
// Request
{
  "question": "광합성의 과정을 설명해줘",
  "documentId": "optional-uuid"
}

// Response (200 OK)
{
  "answer": "광합성은 빛에너지를 이용하여 이산화탄소와 물로부터 유기물을 합성하는 과정입니다...",
  "sources": [
    {
      "content": "광합성은 크게 명반응과 암반응(캘빈 회로)으로 나뉜다...",
      "location": { "type": "S3", "uri": "s3://bucket/생물학.pdf" },
      "score": 0.92,
      "metadata": { "page": 45 }
    }
  ],
  "sessionId": "optional-session-id"
}

// Error Response (500)
{
  "error": {
    "code": "BEDROCK_ERROR",
    "message": "답변을 생성하는 중 오류가 발생했습니다. 잠시 후 다시 시도해주세요."
  }
}
```

---

## 부록 B: 프론트엔드 컴포넌트 구조

```
src/
├── App.tsx                    # 메인 앱 (라우팅)
├── components/
│   ├── ChatWindow.tsx         # 채팅 메시지 목록 표시
│   ├── ChatInput.tsx          # 질문 입력 폼
│   ├── MessageBubble.tsx      # 개별 메시지 (사용자/AI)
│   ├── SourceCard.tsx         # 출처 표시 카드
│   ├── FileUpload.tsx         # PDF 업로드 드래그앤드롭
│   ├── DocumentList.tsx       # 업로드된 문서 목록
│   ├── LoadingSpinner.tsx     # 로딩 인디케이터
│   └── IngestionStatus.tsx    # KB 동기화 상태 표시 (폴링 5초)
├── hooks/
│   ├── useChat.ts             # 채팅 API 호출 훅
│   └── useUpload.ts           # 업로드 API 호출 훅
├── lib/
│   └── api.ts                 # API 클라이언트 (fetch wrapper)
├── types/
│   └── index.ts               # 타입 정의
└── styles/
    └── globals.css            # Tailwind 글로벌 스타일

### 프론트엔드 기술 스택 상세
| 항목 | 선택 | 버전 |
|------|------|------|
| UI 프레임워크 | React | 19 |
| 빌드 | Vite | 6.x |
| CSS | Tailwind CSS | 4.x |
| HTTP 클라이언트 | fetch (네이티브) | - |
| 상태관리 | React useState/useReducer | 내장 (별도 라이브러리 불필요) |
| 마크다운 렌더링 | react-markdown | AI 답변 포맷팅용 |
```

---

## 부록 C: Lambda 함수 구조

### C.1 런타임 선택

| 항목 | 선택 | 이유 |
|------|------|------|
| 런타임 | **Python 3.12** | Bedrock SDK(boto3) 기본 포함, 문서 처리 라이브러리 풍부 |
| 패키지 관리 | Lambda Layer | boto3 최신 버전 + 공통 유틸리티 |
| IaC | AWS SAM | Lambda + API Gateway 통합 배포, 학습 곡선 낮음 |

### C.1.1 Lambda 함수별 설정

| 함수 | 메모리 | 타임아웃 | 비고 |
|------|:------:|:--------:|------|
| upload/presign | 128MB | 10초 | S3 Presigned URL 생성 (경량) |
| upload/complete | 256MB | 30초 | KB StartIngestionJob 호출 |
| chat/handler | 256MB | **25초** | Bedrock RetrieveAndGenerate (API GW 29초 하드리밋 고려) |
| documents/handler | 128MB | 10초 | S3 ListObjects (경량) |

### C.2 Lambda 함수 목록

```
functions/
├── upload/
│   ├── presign.py          # Presigned URL 생성
│   └── complete.py         # 업로드 완료 + KB 동기화
├── chat/
│   └── handler.py          # RetrieveAndGenerate 호출
├── documents/
│   └── handler.py          # 문서 목록/삭제
└── shared/
    └── layer/              # Lambda Layer (공통 유틸리티)
        ├── bedrock_client.py
        └── response_helper.py
```

---

## 부록 D: 참고 자료

- [Amazon Bedrock Knowledge Bases 공식 문서](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Amazon S3 Vectors 공식 문서](https://aws.amazon.com/s3/features/vectors/)
- [AWS Bedrock RAG Workshop (GitHub)](https://github.com/aws-samples/bedrock-kb-rag-workshop)
- [Amazon Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)
- [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)
- [Amazon S3 Vectors Pricing](https://aws.amazon.com/s3/pricing/)
- [CloudFront Free Tier Plans](https://aws.amazon.com/cloudfront/pricing/)

---

## 버전 이력

| 버전 | 날짜 | 변경 사항 | 작성자 |
|------|------|-----------|--------|
| 0.1 | 2026-03-13 | 초안 작성 | CTO Lead (bkit) |
| 0.2 | 2026-03-13 | 스펙 리뷰 반영 (C1~C2, M1~M5, m1~m6) | Spec Review |
| 0.3 | 2026-03-15 | 수업계획서 정합성 검토 — 타임라인/보안/비용 수정 | Team |
