# 실시간 뉴스 감정 분석 대시보드 - 아키텍처 설계

> **Summary**: AWS 서버리스 파이프라인으로 뉴스 RSS/API를 실시간 수집하고, AI 감정 분석 + 키워드 추출 후 대시보드에 시각화하는 시스템
>
> **Project**: 캡스톤 (AWS 실전프로젝트 I)
> **Author**: Team B
> **Date**: 2026-03-13
> **Status**: Draft

---

## Executive Summary

| 관점 | 내용 |
|------|------|
| **Problem** | 뉴스의 양이 폭발적으로 증가하지만, 특정 주제에 대한 전반적 여론/감정 흐름을 파악하기 어려움 |
| **Solution** | AWS 서버리스 파이프라인으로 뉴스 자동 수집 -> Comprehend AI 감정 분석 -> 실시간 대시보드 시각화 |
| **Function/UX Effect** | 키워드 기반 뉴스 감정 트렌드를 시간축 그래프와 워드클라우드로 직관적 확인 가능 |
| **Core Value** | 완전 서버리스 아키텍처로 비용 최소화하면서 AI 기반 뉴스 인사이트를 실시간 제공 |

---

## 1. 시스템 아키텍처 다이어그램

### 1.1 전체 시스템 흐름

```
                          [데이터 수집 계층]
                               │
    ┌──────────────────────────┼──────────────────────────┐
    │                          │                          │
    ▼                          ▼                          ▼
 ┌────────┐             ┌────────────┐            ┌────────────┐
 │NewsAPI │             │Google News │            │ RSS Feeds  │
 │  .org  │             │   RSS      │            │ (네이버 등)│
 └───┬────┘             └─────┬──────┘            └─────┬──────┘
     │                        │                         │
     └────────────┬───────────┴─────────────────────────┘
                  │
                  ▼
         ┌────────────────┐
         │  EventBridge   │  (5분/15분 주기 스케줄)
         │  Scheduler     │
         └───────┬────────┘
                 │ 트리거
                 ▼
         ┌────────────────┐
         │  Lambda #1     │  뉴스 수집기 (Collector)
         │  (Python 3.12) │  - RSS 파싱 / API 호출
         │  128MB, 60s    │  - 중복 제거 (URL 해시)
         └───────┬────────┘
                 │ S3 Put (원본 저장)
                 ├──────────────────────┐
                 ▼                      ▼
         ┌────────────────┐    ┌────────────────┐
         │  S3 Bucket     │    │  SQS Queue     │
         │  (Raw Articles)│    │  (분석 대기열)  │
         └────────────────┘    └───────┬────────┘
                                       │ 트리거
                                       ▼
                               ┌────────────────┐
                               │  Lambda #2     │  NLP 분석기 (Analyzer)
                               │  (Python 3.12) │  - Comprehend 호출
                               │  256MB, 120s   │  - 감정 분석 + 키워드
                               └───────┬────────┘
                                       │
                                       ▼
                               ┌────────────────┐
                               │  DynamoDB      │  분석 결과 저장
                               │  (On-Demand)   │  - 감정 점수, 키워드
                               └───────┬────────┘
                                       │
                          ┌────────────┼────────────┐
                          ▼            ▼            ▼
                   ┌──────────┐ ┌──────────┐ ┌──────────────┐
                   │API GW    │ │API GW    │ │API GW        │
                   │GET/trend │ │GET/latest│ │GET/keywords  │
                   └────┬─────┘ └────┬─────┘ └──────┬───────┘
                        │            │               │
                        └────────┬───┴───────────────┘
                                 ▼
                         ┌────────────────┐
                         │  Lambda #3     │  API 핸들러
                         │  (Node.js 20)  │  - DynamoDB 쿼리
                         │  128MB, 10s    │  - 응답 포맷팅
                         └───────┬────────┘
                                 │
                                 ▼
                         ┌────────────────┐
                         │  CloudFront    │  CDN + HTTPS
                         └───────┬────────┘
                                 │
                                 ▼
                         ┌────────────────┐
                         │  S3 Bucket     │  정적 대시보드
                         │  (Frontend)    │  React SPA
                         └────────────────┘
                                 │
                                 ▼
                            [사용자 브라우저]
```

### 1.2 데이터 파이프라인 상세 흐름

```
[뉴스 소스]                  [수집 계층]                [분석 계층]           [저장 계층]         [표현 계층]
    │                            │                        │                    │                  │
    │  RSS/API 응답              │                        │                    │                  │
    ├──────────────────────────▶ │                        │                    │                  │
    │                            │  원본 기사 JSON         │                    │                  │
    │                            ├───────────────────────▶│                    │                  │
    │                            │  (SQS 메시지)          │                    │                  │
    │                            │                        │  Comprehend API    │                  │
    │                            │                        │  - detectSentiment │                  │
    │                            │                        │  - detectKeyPhrases│                  │
    │                            │                        │                    │                  │
    │                            │                        │  분석 결과 JSON     │                  │
    │                            │                        ├──────────────────▶ │                  │
    │                            │                        │  (DynamoDB Put)    │                  │
    │                            │                        │                    │                  │
    │                            │                        │                    │  API Gateway      │
    │                            │                        │                    ├────────────────▶ │
    │                            │                        │                    │  JSON 응답        │
    │                            │                        │                    │                  │
    │                            │                        │                    │          차트 렌더링
    │                            │                        │                    │                  │
```

---

## 2. AWS 서비스 구성

### 2.1 서비스별 역할 및 연결 방식

| 서비스 | 역할 | 연결 방식 | 설정 핵심 |
|--------|------|-----------|-----------|
| **EventBridge Scheduler** | 주기적 뉴스 수집 트리거 | -> Lambda #1 직접 호출 | rate(5 minutes) 또는 cron |
| **Lambda #1 (Collector)** | 뉴스 RSS/API 수집, 전처리 | -> S3 (원본 저장), -> SQS (분석 요청) | Python 3.12, 128MB, 60s timeout |
| **S3 (Raw)** | 원본 뉴스 기사 아카이빙 | Lambda #1에서 PutObject | 수명주기 정책: 30일 후 삭제 |
| **SQS** | 수집-분석 디커플링 버퍼 | Lambda #1 -> SQS -> Lambda #2 | 배치 크기 10, 가시성 타임아웃 180s |
| **Lambda #2 (Analyzer)** | NLP 분석 (감정 + 키워드) | SQS 트리거, -> Comprehend API, -> DynamoDB | Python 3.12, 256MB, 120s timeout |
| **Comprehend** | 감정 분석, 키워드 추출 | Lambda #2에서 boto3 호출 | 한국어 지원: 감정(O), 키워드(O) |
| **DynamoDB** | 분석 결과 저장, 쿼리 | Lambda #2 PutItem, Lambda #3 Query | On-Demand 모드, GSI 2개 |
| **API Gateway** | REST API 엔드포인트 | -> Lambda #3 (프록시 통합) | HTTP API (비용 절감) |
| **Lambda #3 (API)** | 대시보드 데이터 API | API GW 트리거, -> DynamoDB Query | Node.js 20, 128MB, 10s timeout |
| **S3 (Frontend)** | 정적 대시보드 호스팅 | CloudFront Origin | 정적 웹 호스팅 활성화 |
| **CloudFront** | CDN, HTTPS, 캐싱 | S3 Origin + API GW Origin | OAC 설정, 커스텀 도메인(옵션) |
| **CloudWatch** | 모니터링, 로깅, 알람 | 전 서비스 통합 | Lambda 에러율 알람, 비용 알람 |
| **IAM** | 서비스 간 권한 관리 | 모든 서비스에 적용 | 최소 권한 원칙 |

### 2.2 Lambda 함수 상세 설계

#### Lambda #1: 뉴스 수집기 (news-collector)

```python
# 의사코드
def handler(event, context):
    # 1. 뉴스 소스에서 기사 수집
    articles = []
    articles += fetch_google_news_rss(keywords)   # 무료, 제한 없음
    articles += fetch_newsapi(keywords)            # 무료 티어: 100req/일
    articles += fetch_naver_news_rss(keywords)     # 무료, 한국어 뉴스

    # 2. 중복 제거 (URL 해시 기반)
    unique_articles = deduplicate(articles, dynamodb_table)

    # 3. 원본 저장 (S3)
    save_to_s3(unique_articles, bucket="raw-articles")

    # 4. 분석 요청 (SQS)
    send_to_sqs(unique_articles, queue="analysis-queue")

    return {"collected": len(unique_articles)}
```

#### Lambda #2: NLP 분석기 (news-analyzer)

```python
# 의사코드
def handler(event, context):
    comprehend = boto3.client('comprehend')

    for record in event['Records']:
        article = json.loads(record['body'])

        # 1. 감정 분석
        sentiment = comprehend.detect_sentiment(
            Text=article['title'] + ' ' + article['description'],
            LanguageCode='ko'  # 한국어
        )

        # 2. 키워드 추출
        key_phrases = comprehend.detect_key_phrases(
            Text=article['title'] + ' ' + article['description'],
            LanguageCode='ko'
        )

        # 3. DynamoDB 저장
        save_analysis_result({
            'articleId': article['url_hash'],
            'title': article['title'],
            'source': article['source'],
            'publishedAt': article['published_at'],
            'sentiment': sentiment['Sentiment'],          # POSITIVE/NEGATIVE/NEUTRAL/MIXED
            'sentimentScores': sentiment['SentimentScore'],# 각 감정별 확률
            'keywords': extract_keywords(key_phrases),
            'analyzedAt': datetime.now().isoformat()
        })
```

#### Lambda #3: API 핸들러 (dashboard-api)

```javascript
// 의사코드
export const handler = async (event) => {
  const { path, queryStringParameters } = event;

  switch (path) {
    case '/api/trend':
      // 시간대별 감정 분포 조회
      return await getSentimentTrend(queryStringParameters);
    case '/api/latest':
      // 최신 분석 결과 조회
      return await getLatestArticles(queryStringParameters);
    case '/api/keywords':
      // 인기 키워드 조회
      return await getTopKeywords(queryStringParameters);
    case '/api/stats':
      // 전체 통계 조회
      return await getDashboardStats();
  }
};
```

### 2.3 DynamoDB 테이블 설계

#### 메인 테이블: `NewsArticles`

| 속성 | 타입 | 설명 |
|------|------|------|
| `PK` (Partition Key) | String | `ARTICLE#{url_hash}` |
| `SK` (Sort Key) | String | `ANALYZED#{ISO-8601 timestamp}` |
| `title` | String | 기사 제목 |
| `source` | String | 뉴스 출처 (google, naver 등) |
| `url` | String | 원본 기사 URL |
| `publishedAt` | String | 기사 발행 시각 (ISO-8601) |
| `sentiment` | String | POSITIVE / NEGATIVE / NEUTRAL / MIXED |
| `positiveScore` | Number | 긍정 확률 (0~1) |
| `negativeScore` | Number | 부정 확률 (0~1) |
| `neutralScore` | Number | 중립 확률 (0~1) |
| `mixedScore` | Number | 혼합 확률 (0~1) |
| `keywords` | List<String> | 추출된 핵심 키워드 |
| `searchKeyword` | String | 수집 시 사용한 검색 키워드 |
| `analyzedAt` | String | 분석 완료 시각 (ISO-8601) |
| `TTL` | Number | 자동 삭제 Unix timestamp (90일) |

#### GSI (Global Secondary Index)

| GSI 이름 | Partition Key | Sort Key | 용도 |
|----------|---------------|----------|------|
| `GSI-Sentiment` | `sentiment` | `publishedAt` | 감정별 시계열 조회 |
| `GSI-Keyword` | `searchKeyword` | `publishedAt` | 키워드별 시계열 조회 |

---

## 3. 데이터 파이프라인 상세

### 3.1 수집 단계

```
[뉴스 소스]                    [수집 전략]
─────────────────────────────────────────────
Google News RSS               무료, 무제한
 - 한국어 뉴스 RSS            - feedparser 라이브러리
 - 키워드별 RSS 피드           - XML 파싱

NewsAPI.org                    무료 티어: 100 요청/일
 - Developer Plan              - requests 라이브러리
 - 한국어 뉴스 지원            - JSON 응답

네이버 뉴스 RSS                무료, 무제한
 - 주제별 RSS 제공             - feedparser 라이브러리
 - 한국어 뉴스 특화            - XML 파싱

NewsData.io (백업)             무료 티어: 200 요청/일
 - 30개 언어 지원              - REST API
 - 한국 뉴스 커버리지          - JSON 응답
```

### 3.2 전처리 단계

```
원본 기사
    │
    ▼
[텍스트 정규화]
    - HTML 태그 제거
    - 특수문자 정리
    - 공백 정규화
    │
    ▼
[중복 제거]
    - URL 해시 생성 (SHA-256)
    - DynamoDB 조건부 쓰기 (ConditionExpression)
    │
    ▼
[언어 감지]
    - Comprehend.detect_dominant_language()
    - 한국어/영어 기사만 필터링
    │
    ▼
[텍스트 결합]
    - 제목 + 본문 요약 (최대 5,000바이트)
    - Comprehend 입력 크기 제한 준수
```

### 3.3 NLP 분석 단계

```
전처리된 텍스트
    │
    ├──▶ [Comprehend: detect_sentiment]
    │       입력: 텍스트 (최대 5,000 bytes)
    │       출력: {
    │           Sentiment: "POSITIVE" | "NEGATIVE" | "NEUTRAL" | "MIXED",
    │           SentimentScore: {
    │               Positive: 0.92,
    │               Negative: 0.02,
    │               Neutral: 0.05,
    │               Mixed: 0.01
    │           }
    │       }
    │
    └──▶ [Comprehend: detect_key_phrases]
            입력: 텍스트 (최대 5,000 bytes)
            출력: {
                KeyPhrases: [
                    { Text: "인공지능", Score: 0.99 },
                    { Text: "반도체 수출", Score: 0.95 },
                    ...
                ]
            }
```

### 3.4 저장 및 시각화 단계

```
분석 결과 (DynamoDB)
    │
    ├──▶ [/api/trend]     시간대별 감정 분포
    │       - 최근 24시간, 7일, 30일
    │       - 긍정/부정/중립 비율 추이
    │
    ├──▶ [/api/latest]    최신 분석 기사
    │       - 페이지네이션 (20건/페이지)
    │       - 감정 라벨 + 점수 포함
    │
    ├──▶ [/api/keywords]  인기 키워드
    │       - 기간별 키워드 빈도
    │       - 워드클라우드 데이터
    │
    └──▶ [/api/stats]     전체 통계
            - 총 수집/분석 건수
            - 감정 분포 요약
            - 소스별 통계
```

---

## 4. 구현 난이도 분석

| 컴포넌트 | 난이도 | 소요 시간 | 설명 |
|----------|:------:|:---------:|------|
| **EventBridge 스케줄러 설정** | ★☆☆☆☆ | 0.5일 | 콘솔에서 클릭 몇 번으로 설정 가능 |
| **Lambda #1 (뉴스 수집)** | ★★☆☆☆ | 2일 | RSS 파싱은 라이브러리가 잘 되어 있음, API 호출도 단순 |
| **S3 원본 저장** | ★☆☆☆☆ | 0.5일 | boto3로 PutObject 호출만 하면 됨 |
| **SQS 연동** | ★★☆☆☆ | 1일 | Lambda 트리거 설정, 배치 처리 이해 필요 |
| **Lambda #2 (NLP 분석)** | ★★★☆☆ | 3일 | Comprehend API 호출 자체는 쉬우나 에러 핸들링/배치 처리 필요 |
| **Comprehend 감정 분석** | ★★☆☆☆ | 1일 | 관리형 서비스라 API 호출만 하면 됨, 한국어 지원 확인 필요 |
| **DynamoDB 설계 및 구현** | ★★★☆☆ | 3일 | 테이블 설계(PK/SK), GSI 이해, 쿼리 패턴 설계 |
| **API Gateway + Lambda #3** | ★★★☆☆ | 3일 | REST API 설계, CORS 설정, 쿼리 최적화 |
| **S3 + CloudFront 호스팅** | ★★☆☆☆ | 1일 | 정적 호스팅 설정, OAC, HTTPS |
| **프론트엔드 대시보드** | ★★★★☆ | 5일 | React + 차트 라이브러리, 반응형, API 연동 |
| **IAM 권한 설정** | ★★☆☆☆ | 1일 | 최소 권한 원칙 적용, 역할별 정책 |
| **모니터링 (CloudWatch)** | ★★☆☆☆ | 1일 | 로그 그룹, 알람, 대시보드 |
| **IaC (CloudFormation/SAM)** | ★★★☆☆ | 3일 | SAM 템플릿으로 인프라 코드화 (차별화 포인트) |
| **CI/CD (GitHub Actions)** | ★★★☆☆ | 2일 | 자동 배포 파이프라인 구축 |

### 난이도 종합

| 카테고리 | 평균 난이도 | 비고 |
|----------|:----------:|------|
| 백엔드 (데이터 파이프라인) | ★★☆☆☆ | AWS 관리형 서비스 활용으로 부담 적음 |
| AI/NLP (Comprehend) | ★★☆☆☆ | API 호출만으로 고급 NLP 가능 |
| 프론트엔드 (대시보드) | ★★★★☆ | 시각화 품질이 평가에 큰 영향 |
| 인프라/DevOps | ★★★☆☆ | IaC + CI/CD는 중급 수준 |

---

## 5. 15주 타임라인

### 5.1 주차별 개발 계획

| 주차 | 기간 | 마일스톤 | 상세 작업 | 산출물 |
|:----:|------|---------|-----------|--------|
| **1주** | 03/06~03/12 | 킥오프 + 기획 | 팀 구성, 역할 분담, 요구사항 정의 | 프로젝트 계획서 |
| **2주** | 03/13~03/19 | 아키텍처 설계 | AWS 아키텍처 확정, 서비스 선정, DynamoDB 스키마 설계 | 아키텍처 설계서 (본 문서) |
| **3주** | 03/20~03/26 | 환경 구축 | AWS 계정 설정, IAM, S3 버킷, DynamoDB 테이블 생성 | 인프라 기본 셋업 완료 |
| **4주** | 03/27~04/02 | 수집기 개발 | Lambda #1 구현, RSS 파싱, API 연동, EventBridge 스케줄 | 뉴스 자동 수집 동작 |
| **5주** | 04/03~04/09 | 분석기 개발 | Lambda #2 구현, Comprehend 연동, SQS 트리거 설정 | NLP 분석 파이프라인 동작 |
| **6주** | 04/10~04/16 | API 개발 | API Gateway + Lambda #3, REST API 4개 엔드포인트 | API 응답 확인 |
| **7주** | 04/17~04/23 | MVP 통합 | 파이프라인 End-to-End 테스트, 버그 수정 | **MVP 동작 확인** |
| **8주** | 04/24~04/30 | **중간고사** | MVP 시연, 중간 발표 준비 | **중간 발표 자료** |
| **9주** | 05/01~05/07 | 프론트엔드 시작 | React 프로젝트 셋업, 대시보드 레이아웃, 차트 기본 연동 | 대시보드 프로토타입 |
| **10주** | 05/08~05/14 | 대시보드 개발 | 감정 트렌드 차트, 워드클라우드, 실시간 피드 | 대시보드 핵심 기능 완성 |
| **11주** | 05/15~05/21 | 고도화 | CloudFront 배포, 키워드 필터, 기간 필터, 반응형 | 프로덕션 수준 대시보드 |
| **12주** | 05/22~05/28 | IaC + CI/CD | SAM 템플릿 작성, GitHub Actions 배포 파이프라인 | 인프라 코드화 완료 |
| **13주** | 05/29~06/04 | 보안 + 최적화 | WAF 설정(옵션), Lambda 최적화, 비용 모니터링 | 보안 점검 완료 |
| **14주** | 06/05~06/11 | 문서화 + 발표 준비 | 기술 문서, 발표 자료, 시연 리허설 | 최종 발표 자료 |
| **15주** | 06/12 | **기말 발표** | 최종 시연 및 발표 | **최종 산출물 제출** |

### 5.2 마일스톤 다이어그램

```
주차:  1    2    3    4    5    6    7    8    9   10   11   12   13   14   15
      ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
기획   ████
설계        ████
인프라           ████                              ████████
백엔드                ████████████████
      [수집기]   [분석기]  [API]
MVP                                   ████████
                                           ▲ 중간고사
프론트                                          ████████████
고도화                                                    ████████
보안/최적화                                                    ████
문서화                                                              ████
                                                                      ▲ 기말
```

---

## 6. 예상 비용

### 6.1 AWS 프리티어 + 학생 크레딧 기준

전제 조건:
- 뉴스 수집: 5분 간격, 하루 288회
- 기사 수: 회당 평균 20건 = 하루 약 5,760건
- Comprehend: 기사당 약 500자 = 5유닛
- DynamoDB: On-Demand 모드

| 서비스 | 월 사용량 (예상) | 프리티어 한도 | 초과 비용 | 비고 |
|--------|:----------------:|:------------:|:---------:|------|
| **Lambda** | ~180,000 호출, ~2,000 GB-s | 1M 호출, 400K GB-s | $0 | 프리티어 내 |
| **EventBridge** | ~8,640 호출 | 14M 무료 | $0 | 프리티어 내 |
| **SQS** | ~180,000 메시지 | 1M 무료 | $0 | 프리티어 내 |
| **DynamoDB** | ~200K 쓰기, ~50K 읽기/일 | 25 WCU/RCU (프리티어) | $2~5/월 | On-Demand 권장 |
| **Comprehend** | ~180K 유닛 (기사) | 50K 유닛/API/월 (12개월) | $2~8/월 | 12개월 프리티어 |
| **S3** | ~2GB (원본 + 프론트) | 5GB (12개월) | $0 | 프리티어 내 |
| **CloudFront** | ~5GB 전송 | 1TB 무료 | $0 | Always Free |
| **API Gateway** | ~100K 호출 | 1M/월 (12개월) | $0 | HTTP API 사용 |
| **CloudWatch** | 기본 로깅 | 5GB 로그 | $0 | 기본 모니터링 |

### 6.2 월간 총 비용 추정

| 시나리오 | 월 비용 | 설명 |
|----------|:-------:|------|
| **최소 운영** (5분 간격, 3개 소스) | **$2~5** | 대부분 프리티어 범위 내 |
| **일반 운영** (5분 간격, 5개 소스) | **$5~13** | Comprehend + DynamoDB 소량 과금 |
| **집중 운영** (1분 간격, 10개 소스) | **$15~30** | Comprehend 사용량 증가 |

### 6.3 비용 최적화 전략

1. **Comprehend 호출 최소화**: 제목+요약만 분석 (본문 전체 분석 지양)
2. **DynamoDB TTL 활용**: 90일 후 자동 삭제로 스토리지 비용 관리
3. **Lambda 메모리 최적화**: 실제 사용량 기반 Right-sizing
4. **SQS 배치 처리**: 한 번의 Lambda 호출로 10건씩 처리
5. **CloudWatch 알람**: 비용 임계치 초과 시 알림 설정 ($10, $20)
6. **AWS Budget**: 월별 예산 한도 설정 ($30)

> AWS 학생 크레딧($100~$200)이 있다면 15주 프로젝트 기간 동안 비용 걱정 없이 운영 가능

---

## 7. 위험 요소 및 대응 방안

### 7.1 기술적 리스크

| 리스크 | 영향 | 가능성 | 대응 방안 |
|--------|:----:|:------:|-----------|
| **Comprehend 한국어 감정 분석 정확도 낮음** | High | Medium | 영문 번역 후 분석 백업 경로 준비, Amazon Translate 연동 (추가 비용 미미) |
| **뉴스 API 무료 티어 제한 초과** | Medium | Medium | Google News RSS(무제한)를 주력, NewsAPI는 보조 소스로만 활용 |
| **Lambda Cold Start 지연** | Low | High | Provisioned Concurrency 없이도 128MB Lambda는 1~2초 내 시작, 5분 주기라 영향 적음 |
| **DynamoDB 핫 파티션** | Medium | Low | 파티션 키를 URL 해시로 분산, On-Demand 모드로 자동 스케일링 |
| **SQS 메시지 유실** | High | Low | DLQ(Dead Letter Queue) 설정, 3회 재시도 후 DLQ로 이동 |
| **프론트엔드 차트 성능** | Medium | Medium | 대량 데이터는 서버 측 집계, 클라이언트는 집계 결과만 렌더링 |
| **CORS 설정 문제** | Low | High | CloudFront로 API + 프론트엔드 동일 도메인 제공, CORS 불필요 |

### 7.2 프로젝트 리스크

| 리스크 | 영향 | 가능성 | 대응 방안 |
|--------|:----:|:------:|-----------|
| **팀원 이탈/일정 불참** | High | Low | 모든 코드 GitHub 관리, 각 컴포넌트 독립 배포 가능하게 설계 |
| **AWS 학습 곡선** | Medium | High | 1~3주차 집중 학습, AWS 멘토 활용, 공식 워크숍 참고 |
| **중간고사까지 MVP 미완성** | High | Medium | 4~7주차 백엔드만 집중, 프론트엔드는 8주차 이후 |
| **AWS 비용 초과** | Medium | Low | 예산 알람 설정, 프리티어 모니터링, 학생 크레딧 활용 |
| **뉴스 소스 차단/변경** | Medium | Medium | 3개 이상 소스 확보, 소스별 어댑터 패턴으로 교체 용이하게 설계 |

### 7.3 리스크 완화를 위한 아키텍처 설계 원칙

1. **느슨한 결합 (Loose Coupling)**: SQS로 수집-분석 분리, 한쪽 장애가 전체에 영향 없음
2. **어댑터 패턴**: 뉴스 소스별 어댑터로 소스 추가/교체 용이
3. **점진적 개발**: 각 Lambda 독립 배포, 개별 테스트 가능
4. **자동 복구**: SQS DLQ, Lambda 재시도, CloudWatch 알람
5. **비용 안전망**: AWS Budget + 비용 알람 + DynamoDB TTL

---

## 8. 팀 역할 분담

### 8.1 2인 팀 구성

| 역할 | 담당 영역 | 주요 작업 |
|------|-----------|-----------|
| **팀원 A: 백엔드 + 인프라** | 데이터 파이프라인 | Lambda #1/#2 개발, DynamoDB 설계, SQS/EventBridge 설정, IAM, SAM 템플릿, CI/CD |
| **팀원 B: API + 프론트엔드** | API 및 대시보드 | Lambda #3 (API) 개발, API Gateway, React 대시보드, S3/CloudFront 배포, 차트 시각화 |

**공통 담당**: 아키텍처 설계, 테스트, 문서화, 발표 준비

### 8.2 3인 팀 구성

| 역할 | 담당 영역 | 주요 작업 |
|------|-----------|-----------|
| **팀원 A: 데이터 엔지니어** | 수집 + NLP | Lambda #1/#2 개발, 뉴스 소스 관리, Comprehend 연동, SQS/EventBridge |
| **팀원 B: 백엔드 개발자** | API + DB + 인프라 | Lambda #3, API Gateway, DynamoDB 설계, SAM, CI/CD, IAM |
| **팀원 C: 프론트엔드 개발자** | 대시보드 + 배포 | React 대시보드, 차트 시각화, S3/CloudFront, 반응형 UI |

**공통 담당**: 아키텍처 설계, 코드 리뷰, 문서화, 발표 준비

### 8.3 협업 도구

| 도구 | 용도 |
|------|------|
| **GitHub** | 코드 관리, PR 리뷰, Issue 트래킹 |
| **GitHub Actions** | CI/CD 자동 배포 |
| **AWS Console** | 서비스 모니터링, 디버깅 |
| **Notion / Google Docs** | 회의록, 문서 공유 |
| **Slack / Discord** | 실시간 커뮤니케이션 |

---

## 9. 차별화 포인트

### 9.1 기술적 차별화

| 차별화 요소 | 설명 | 어필 포인트 |
|-------------|------|-------------|
| **완전 서버리스** | EC2 없이 100% 서버리스 아키텍처 | 운영 비용 최소화, 자동 스케일링, 관리 부담 제로 |
| **IaC (SAM)** | 전체 인프라를 코드로 관리 | 재현 가능한 배포, DevOps 역량 시연 |
| **이벤트 기반 아키텍처** | SQS로 디커플링된 비동기 파이프라인 | 확장성, 장애 격리, 실무 수준 설계 |
| **CI/CD 자동화** | GitHub Actions -> SAM Deploy | 코드 변경 시 자동 배포, DevOps 파이프라인 완성 |
| **비용 최적화** | 프리티어 중심 설계 + 비용 모니터링 | AWS 비용 관리 역량, FinOps 인식 |

### 9.2 비즈니스 차별화

| 차별화 요소 | 설명 |
|-------------|------|
| **실시간 뉴스 인사이트** | 특정 키워드/주제에 대한 여론 트렌드를 즉시 파악 |
| **다중 뉴스 소스** | 단일 소스 의존 없이 여러 소스에서 균형 잡힌 데이터 수집 |
| **한국어 NLP** | 한국어 뉴스에 특화된 감정 분석 (Comprehend 한국어 지원) |
| **시각적 대시보드** | 감정 트렌드 그래프 + 워드클라우드로 직관적 인사이트 전달 |

### 9.3 발표/시연 시 강조할 포인트

1. **아키텍처 다이어그램**: 서비스 간 연결 흐름을 명확히 시각화
2. **실시간 시연**: 키워드 입력 -> 뉴스 수집 -> 감정 분석 -> 대시보드 반영 라이브 데모
3. **비용 분석**: 실제 AWS 비용 대시보드 공개 (월 $5 미만 운영 가능)
4. **IaC 시연**: `sam deploy` 한 번으로 전체 인프라 배포
5. **보안 설계**: IAM 최소 권한, CloudFront HTTPS, OAC 적용

### 9.4 추가 고도화 아이디어 (시간 여유 시)

| 아이디어 | 난이도 | 설명 |
|----------|:------:|------|
| **Amazon Bedrock 요약** | ★★★☆☆ | Claude/Titan으로 뉴스 기사 자동 요약 생성 |
| **SNS 알림** | ★★☆☆☆ | 급격한 감정 변화 감지 시 이메일/Slack 알림 |
| **Cognito 인증** | ★★★☆☆ | 사용자 로그인 + 개인화 키워드 관리 |
| **QuickSight 연동** | ★★★☆☆ | BI 수준의 고급 시각화, AWS 서비스 활용도 증가 |
| **다국어 지원** | ★★☆☆☆ | Amazon Translate로 영문 뉴스도 한국어로 분석 |

---

## 10. MVP 정의 (중간고사 8주차까지)

### 10.1 MVP 범위

```
[MVP - 8주차까지 완성 목표]
───────────────────────────────────────────────────────────
포함 (Must Have):
  [x] EventBridge -> Lambda #1: 뉴스 수집 (최소 2개 소스)
  [x] SQS -> Lambda #2: Comprehend 감정 분석
  [x] DynamoDB: 분석 결과 저장
  [x] API Gateway + Lambda #3: REST API (최소 2개 엔드포인트)
  [x] CLI 또는 간단한 HTML로 결과 확인 가능

제외 (중간고사 이후):
  [ ] React 대시보드 (9~11주차)
  [ ] CloudFront 배포 (11주차)
  [ ] SAM/IaC (12주차)
  [ ] CI/CD (12주차)
  [ ] 보안 강화 (13주차)
───────────────────────────────────────────────────────────
```

### 10.2 MVP 아키텍처 (간소화)

```
EventBridge (5분) -> Lambda #1 (수집) -> SQS -> Lambda #2 (분석)
                                                      │
                                                      ▼
                                                  DynamoDB
                                                      │
                                                      ▼
                                          API Gateway + Lambda #3
                                                      │
                                                      ▼
                                             간단한 HTML 페이지
                                            (S3 정적 호스팅)
```

### 10.3 MVP 검증 기준

| 항목 | 기준 | 검증 방법 |
|------|------|-----------|
| **데이터 수집** | 5분마다 자동으로 뉴스 수집 | CloudWatch 로그 확인 |
| **NLP 분석** | 수집된 기사의 감정 분석 완료 | DynamoDB 테이블 조회 |
| **API 응답** | /api/latest, /api/stats 정상 응답 | curl 또는 Postman 테스트 |
| **중복 제거** | 같은 기사 중복 저장 없음 | DynamoDB 레코드 수 확인 |
| **에러 처리** | API 장애 시 DLQ로 이동 | SQS DLQ 메시지 확인 |

### 10.4 MVP 시연 시나리오 (중간고사)

```
1. AWS 콘솔에서 아키텍처 설명 (2분)
2. EventBridge 스케줄 확인 (1분)
3. CloudWatch에서 Lambda 실행 로그 확인 (2분)
4. DynamoDB에서 분석 결과 조회 (2분)
5. API 호출하여 JSON 응답 확인 (2분)
6. 간단한 HTML 페이지에서 결과 표시 (1분)
───────────────────────────────
총 시연 시간: 약 10분
```

---

## 부록 A: 프론트엔드 대시보드 화면 설계

### A.1 대시보드 레이아웃

```
┌──────────────────────────────────────────────────────────────┐
│  [로고]  실시간 뉴스 감정 분석 대시보드     [키워드 검색]    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌────────┐│
│  │ 총 수집 기사 │ │  긍정 비율   │ │  부정 비율   │ │ 중립   ││
│  │   12,345    │ │   45.2%     │ │   23.1%     │ │ 31.7%  ││
│  └─────────────┘ └─────────────┘ └─────────────┘ └────────┘│
│                                                              │
│  ┌──────────────────────────────┐ ┌─────────────────────────┐│
│  │                              │ │                         ││
│  │    감정 트렌드 차트           │ │    워드클라우드          ││
│  │    (Line Chart)              │ │    (Word Cloud)         ││
│  │    - 긍정/부정/중립 추이      │ │    - 인기 키워드         ││
│  │    - 24시간/7일/30일 전환     │ │    - 크기 = 빈도        ││
│  │                              │ │                         ││
│  └──────────────────────────────┘ └─────────────────────────┘│
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │  최신 뉴스 피드                                          ││
│  │  ┌────────────────────────────────────────────────────┐  ││
│  │  │ [긍정] 반도체 수출 호조... - Google News - 5분 전   │  ││
│  │  │ [부정] 환율 불안 지속... - 네이버 - 12분 전         │  ││
│  │  │ [중립] 정부 AI 정책 발표... - NewsAPI - 18분 전     │  ││
│  │  └────────────────────────────────────────────────────┘  ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌──────────────────────────────┐ ┌─────────────────────────┐│
│  │    소스별 분포               │ │    감정별 파이 차트      ││
│  │    (Bar Chart)              │ │    (Pie Chart)          ││
│  └──────────────────────────────┘ └─────────────────────────┘│
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### A.2 프론트엔드 기술 스택

| 카테고리 | 기술 | 이유 |
|----------|------|------|
| **프레임워크** | React 18+ (Vite) | 빠른 빌드, SPA, 정적 빌드 가능 |
| **차트** | Recharts | React 네이티브, 간단한 API, 반응형 |
| **워드클라우드** | react-wordcloud | 키워드 시각화 전용 |
| **HTTP** | fetch API | 외부 의존성 최소화 |
| **스타일링** | Tailwind CSS | 빠른 프로토타이핑, 반응형 |
| **빌드** | Vite | 빠른 번들링, S3 배포 최적화 |

---

## 부록 B: AWS SAM 템플릿 구조 (예시)

```yaml
# template.yaml (AWS SAM)
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    Timeout: 60
    MemorySize: 128
    Environment:
      Variables:
        TABLE_NAME: !Ref NewsTable
        QUEUE_URL: !Ref AnalysisQueue

Resources:
  # 뉴스 수집 Lambda
  CollectorFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: collector.handler
      CodeUri: functions/collector/
      Events:
        Schedule:
          Type: ScheduleV2
          Properties:
            ScheduleExpression: rate(5 minutes)

  # NLP 분석 Lambda
  AnalyzerFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: analyzer.handler
      CodeUri: functions/analyzer/
      MemorySize: 256
      Timeout: 120
      Events:
        SQSTrigger:
          Type: SQS
          Properties:
            Queue: !GetAtt AnalysisQueue.Arn
            BatchSize: 10

  # API 핸들러 Lambda
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: api.handler
      CodeUri: functions/api/
      Runtime: nodejs20.x
      Events:
        GetTrend:
          Type: HttpApi
          Properties:
            Path: /api/trend
            Method: GET
        GetLatest:
          Type: HttpApi
          Properties:
            Path: /api/latest
            Method: GET

  # DynamoDB 테이블
  NewsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE
      TimeToLiveSpecification:
        AttributeName: TTL
        Enabled: true

  # SQS 큐
  AnalysisQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 180
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt DLQ.Arn
        maxReceiveCount: 3

  # Dead Letter Queue
  DLQ:
    Type: AWS::SQS::Queue

  # S3 프론트엔드 버킷
  FrontendBucket:
    Type: AWS::S3::Bucket
    Properties:
      WebsiteConfiguration:
        IndexDocument: index.html
```

---

## 부록 C: 참고 자료

### AWS 공식 문서
- [Amazon Comprehend 개발자 가이드](https://docs.aws.amazon.com/comprehend/)
- [AWS SAM 개발자 가이드](https://docs.aws.amazon.com/serverless-application-model/)
- [DynamoDB 모범 사례](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)

### AWS 샘플 아키텍처
- [aws-samples/serverless-sentiment-analysis](https://github.com/aws-samples/serverless-sentiment-analysis)
- [aws-samples/devlab-serverless-sentiment-analysis](https://github.com/aws-samples/devlab-serverless-sentiment-analysis)
- [AWS 블로그: Detect sentiment from customer reviews](https://aws.amazon.com/blogs/machine-learning/detect-sentiment-from-customer-reviews-using-amazon-comprehend/)

### 뉴스 API
- [NewsAPI.org](https://newsapi.org/) - Developer 무료 티어
- [NewsData.io](https://newsdata.io/) - 200 요청/일 무료
- [Google News RSS](https://news.google.com/rss) - 무제한 무료

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 0.1 | 2026-03-13 | 초기 아키텍처 설계 문서 작성 | Team B |
