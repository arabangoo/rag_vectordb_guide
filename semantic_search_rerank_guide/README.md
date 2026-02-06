# 시맨틱 검색과 리랭킹 완벽 가이드

> 한국어 개발자를 위한 시맨틱 검색(Semantic Search)과 리랭킹(Reranking) 기술 완벽 구축 가이드

## 📋 목차

- [프로젝트 소개](#-프로젝트-소개)
- [핵심 개념 이해](#-핵심-개념-이해)
- [시맨틱 검색 심화](#-시맨틱-검색-심화)
- [리랭킹 심화](#-리랭킹-심화)
- [하이브리드 검색](#-하이브리드-검색)
- [주요 모델 비교](#-주요-모델-비교)
- [환경 구축](#-환경-구축)
- [실전 구현 가이드](#-실전-구현-가이드)
- [프레임워크 통합](#-프레임워크-통합)
- [성능 최적화](#-성능-최적화)
- [프로덕션 배포](#-프로덕션-배포)
- [트러블슈팅](#-트러블슈팅)
- [FAQ](#-faq)
- [참고 자료](#-참고-자료)

---

## 🎯 프로젝트 소개

이 가이드는 **시맨틱 검색(Semantic Search)**과 **리랭킹(Reranking)** 기술을 활용하여 RAG(Retrieval-Augmented Generation) 시스템의 검색 품질을 극대화하는 방법을 다룹니다.

### 왜 시맨틱 검색과 리랭킹인가?

기존 키워드 기반 검색의 한계를 넘어, **의미 기반 검색**과 **정밀 재정렬**을 통해 검색 정확도를 획기적으로 향상시킬 수 있습니다.

| 기술 | 역할 | 특징 |
|------|------|------|
| **시맨틱 검색** | 1단계: 후보 검색 (Retrieval) | 의미 유사도 기반, 빠른 속도, 대규모 처리 |
| **리랭킹** | 2단계: 정밀 재정렬 (Reranking) | 깊은 관련성 분석, 높은 정확도, 소규모 처리 |

### 학습 목표

이 가이드를 완료하면 다음을 할 수 있습니다:

1. 시맨틱 검색과 리랭킹의 핵심 원리 이해
2. Bi-Encoder와 Cross-Encoder의 차이점 파악
3. 다양한 임베딩 모델과 리랭커 모델 비교 및 선택
4. 하이브리드 검색 (BM25 + 시맨틱) 구현
5. LangChain/LlamaIndex에서 리랭커 통합
6. 프로덕션 환경에서의 성능 최적화
7. 2단계 검색 파이프라인 설계 및 구축

### 실제 활용 사례

- **📚 기업 지식 검색**: 사내 문서에서 정확한 정보 검색
- **🛒 이커머스 검색**: 상품 검색 정확도 향상
- **🏥 의료 정보 검색**: 논문/가이드라인 정밀 검색
- **⚖️ 법률 리서치**: 판례 및 법령 정확 검색
- **💬 고객 지원**: FAQ 및 티켓 히스토리 검색
- **🔬 학술 검색**: 연구 논문 관련성 기반 검색

---

## 🧠 핵심 개념 이해

### 시맨틱 검색 vs 키워드 검색

**키워드 검색 (Lexical Search)**

쿼리: "자동차 수리"
- ✗ "vehicle maintenance" → 매칭 안됨 (다른 단어)
- ✗ "차량 정비" → 매칭 안됨 (동의어 인식 불가)
- ✓ "자동차 수리" → 정확히 일치해야 매칭

**시맨틱 검색 (Semantic Search)**

쿼리: "자동차 수리"
- ✓ "vehicle maintenance" → 의미적 유사 (매칭)
- ✓ "차량 정비" → 동의어 인식 (매칭)
- ✓ "엔진 고장 해결" → 관련 개념 (매칭)

### Bi-Encoder vs Cross-Encoder

두 가지 인코더 아키텍처는 시맨틱 검색과 리랭킹의 핵심입니다.

#### Bi-Encoder (이중 인코더)

```
Query                           Document
  ↓                               ↓
Encoder (같은 모델)            Encoder (같은 모델)
  ↓                               ↓
[벡터 A]                       [벡터 B]
  ↓                               ↓
  └─────→ 코사인 유사도 계산 ←─────┘
                ↓
           유사도 점수
```

**특징:**
- 쿼리와 문서를 독립적으로 인코딩
- 문서 벡터를 미리 계산하여 저장 가능
- 빠른 검색 속도 (벡터 유사도만 계산)
- 대규모 문서 검색에 적합
- 정확도: 중간

#### Cross-Encoder (교차 인코더)

```
Query + Document
      ↓
[Query] [SEP] [Document]
      ↓
Transformer Encoder (전체를 함께 처리)
      ↓
관련성 점수 (0~1)
```

**특징:**
- 쿼리와 문서를 함께 인코딩 (상호작용 분석)
- 매 쿼리마다 계산 필요 (미리 계산 불가)
- 느린 속도 (전체 문서에 적용 불가)
- 소규모 후보군 재정렬에 적합
- 정확도: 높음

#### 성능 비교

| 특성 | Bi-Encoder | Cross-Encoder |
|------|------------|---------------|
| **속도** | 빠름 (1000ms에 1M 문서) | 느림 (100ms에 100 문서) |
| **정확도** | 65-80% | 85-95% |
| **사전 계산** | 가능 (벡터 저장) | 불가능 |
| **확장성** | 수백만 문서 | 수십~수백 문서 |
| **용도** | 1단계 검색 | 2단계 리랭킹 |
| **메모리** | 벡터 저장 필요 | 모델만 필요 |

### 2단계 검색 파이프라인

```
사용자 쿼리
    ↓
[1단계: 시맨틱 검색 (Bi-Encoder)]
  - 수백만 문서에서 Top-K 후보 검색 (예: 100개)
  - 벡터 유사도 기반 빠른 검색
  - 소요 시간: ~50ms
    ↓
(Top 100 문서)
    ↓
[2단계: 리랭킹 (Cross-Encoder)]
  - 100개 후보를 정밀 분석하여 재정렬
  - 쿼리-문서 상호작용 분석
  - 소요 시간: ~200-400ms
    ↓
(Top 10 문서)
    ↓
최종 결과 (높은 정확도)
```

---

## 🔍 시맨틱 검색 심화

### 임베딩 모델의 원리

시맨틱 검색의 핵심은 텍스트를 **고차원 벡터(임베딩)**로 변환하는 것입니다.

```python
from sentence_transformers import SentenceTransformer

# 임베딩 모델 로드
model = SentenceTransformer('BAAI/bge-large-en-v1.5')

# 텍스트를 벡터로 변환
texts = [
    "인공지능은 미래 기술입니다",
    "AI는 혁신적인 기술입니다",
    "오늘 날씨가 좋습니다"
]

embeddings = model.encode(texts)

# 결과: 각 문장이 1024차원 벡터로 변환됨
# embeddings.shape = (3, 1024)

# 유사도 계산
from sklearn.metrics.pairwise import cosine_similarity
similarities = cosine_similarity(embeddings)

# 결과:
# 문장 1-2 유사도: 0.89 (의미적으로 유사)
# 문장 1-3 유사도: 0.12 (의미적으로 다름)
```

### 주요 임베딩 모델 비교 (2025-2026)

| 모델 | 제공사 | 차원 | MTEB 점수 | 가격 (1M 토큰) | 특징 |
|------|--------|------|-----------|----------------|------|
| **text-embedding-3-large** | OpenAI | 3072 | 64.6 | $0.13 | 고성능, API 기반 |
| **text-embedding-3-small** | OpenAI | 1536 | 55.8 | $0.02 | 가성비 우수 |
| **voyage-3-large** | Voyage AI | 1536 | 63.8 | $0.12 | OpenAI 대비 10% 높은 성능 |
| **voyage-3.5** | Voyage AI | 1024 | 62.5 | $0.06 | 최신 모델, 비용 효율적 |
| **embed-multilingual-v3** | Cohere | 1024 | 64.0 | $0.10 | 100+ 언어 지원 |
| **BGE-M3** | BAAI | 1024 | 63.0 | 무료 | 오픈소스, 다국어 |
| **BGE-large-en-v1.5** | BAAI | 1024 | 64.2 | 무료 | 영어 최적화 |
| **jina-embeddings-v3** | Jina AI | 1024 | 65.6 | $0.02 | 최신, 고성능 |
| **all-MiniLM-L6-v2** | SBERT | 384 | 56.3 | 무료 | 가볍고 빠름 |
| **nomic-embed-text-v1.5** | Nomic | 768 | 62.3 | 무료 | 오픈소스, 8K 컨텍스트 |

### 임베딩 모델 선택 가이드

**다국어가 필요한 경우:**
- 예산 있음 → Cohere v3, Voyage-3.5
- 비용 절감 → BGE-M3

**영어만 필요한 경우:**
- 최고 품질 → voyage-3-large, text-embedding-3-large
- 비용 절감 → BGE-large, MiniLM-L6

### 벡터 인덱싱과 검색

대규모 벡터 검색을 위해 **ANN (Approximate Nearest Neighbor)** 알고리즘을 사용합니다.

```python
import faiss
import numpy as np

# 차원 설정
dimension = 1024
num_documents = 1000000

# 문서 임베딩 (예시)
document_embeddings = np.random.random((num_documents, dimension)).astype('float32')

# FAISS 인덱스 생성 (IVF + PQ 압축)
nlist = 1000  # 클러스터 수
m = 64        # 서브벡터 수
quantizer = faiss.IndexFlatIP(dimension)
index = faiss.IndexIVFPQ(quantizer, dimension, nlist, m, 8)

# 인덱스 학습 및 추가
index.train(document_embeddings)
index.add(document_embeddings)

# 검색 파라미터 설정
index.nprobe = 10  # 검색할 클러스터 수

# 쿼리 검색
query_embedding = np.random.random((1, dimension)).astype('float32')
k = 100  # 상위 100개 검색

distances, indices = index.search(query_embedding, k)
# indices: 가장 유사한 문서의 인덱스
# distances: 유사도 점수
```

---

## 🎯 리랭킹 심화

### 리랭킹의 필요성

Bi-Encoder(시맨틱 검색)만으로는 충분하지 않은 이유:

1. **정보 손실**: 전체 문서를 단일 벡터로 압축 → 의미 손실
2. **쿼리 무관 인코딩**: 문서 인코딩 시 쿼리 컨텍스트 부재
3. **제한된 상호작용**: 쿼리-문서 간 세밀한 상호작용 분석 불가

**문제 1: 시맨틱 검색의 노이즈**

쿼리: "Python에서 리스트 정렬하는 방법"

시맨틱 검색 결과 (유사도 순):
1. Python 리스트 기초 튜토리얼 (0.85)
2. 정렬 알고리즘 이론 (0.83)
3. Python sort() 메서드 사용법 (0.81) ← 실제 정답!

리랭킹 후:
1. Python sort() 메서드 사용법 (0.95) ← 정답이 1위로!
2. Python 리스트 기초 튜토리얼 (0.72)
3. 정렬 알고리즘 이론 (0.45)

**문제 2: 미묘한 의미 차이 구분**

쿼리: "Apple 주식 전망"
- 시맨틱 검색: Apple 회사, 사과 과일 문서 모두 높은 점수
- 리랭킹: 쿼리 맥락(주식, 전망)을 분석하여 회사 관련 문서 우선

### 리랭킹 성능 향상 수치

연구 및 벤치마크 결과:

| 지표 | 시맨틱 검색만 | + 리랭킹 | 향상 |
|------|--------------|----------|------|
| **NDCG@10** | 0.45 | 0.67 | +48% |
| **관련성 정확도** | 65-80% | 85-95% | +15-30% |
| **Top-1 정확도** | 52% | 78% | +50% |
| **MRR (Mean Reciprocal Rank)** | 0.58 | 0.82 | +41% |

### 주요 리랭커 모델 비교 (2025-2026)

#### 오픈소스 리랭커

| 모델 | 파라미터 | BEIR NDCG@10 | 속도 | 특징 |
|------|----------|--------------|------|------|
| **cross-encoder/ms-marco-MiniLM-L-6-v2** | 22M | 52.3 | 매우 빠름 | 가볍고 빠름, 입문용 |
| **cross-encoder/ms-marco-MiniLM-L-12-v2** | 33M | 54.1 | 빠름 | 균형잡힌 성능 |
| **BAAI/bge-reranker-base** | 278M | 57.8 | 중간 | 다국어 지원 |
| **BAAI/bge-reranker-large** | 560M | 59.2 | 느림 | 고성능 |
| **BAAI/bge-reranker-v2-m3** | 567M | 56.5 | 중간 | 최신 다국어 |
| **ColBERTv2** | 110M | 56.8 | 빠름 | Late Interaction |

#### 상용 리랭커 API

| 서비스 | 모델 | BEIR NDCG@10 | 가격 (1K 쿼리) | 특징 |
|--------|------|--------------|----------------|------|
| **Cohere** | rerank-v3.5 | 61.2 | $0.10 | 100+ 언어, 4K 컨텍스트 |
| **Cohere** | rerank-v4.0-pro | 63.5 | $0.20 | 32K 컨텍스트 |
| **Jina AI** | jina-reranker-v2 | 60.8 | $0.05 | 15x 빠름, 함수 호출 지원 |
| **Jina AI** | jina-reranker-v3 | 61.9 | $0.08 | SOTA, 64문서 동시 처리 |
| **Voyage AI** | rerank-2 | 60.5 | $0.05 | 코드 검색 특화 |
| **Contextual AI** | ctxl-rerank-en-v1 | 61.2 | 문의 | 명령어 기반 리랭킹 |

### 리랭커 유형별 비교

#### 1. Cross-Encoder (교차 인코더)

```python
from sentence_transformers import CrossEncoder

# Cross-Encoder 로드
model = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

# 쿼리-문서 쌍 점수 계산
query = "Python 리스트 정렬 방법"
documents = [
    "Python에서 sort() 메서드를 사용하면 리스트를 정렬할 수 있습니다.",
    "JavaScript에서 배열을 정렬하는 방법을 알아봅시다.",
    "Python의 sorted() 함수는 새로운 정렬된 리스트를 반환합니다."
]

# 점수 계산
pairs = [[query, doc] for doc in documents]
scores = model.predict(pairs)

# 결과 정렬
ranked_results = sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)

for doc, score in ranked_results:
    print(f"점수: {score:.4f} | {doc[:50]}...")
```

#### 2. ColBERT (Late Interaction)

ColBERT는 토큰 레벨에서 상호작용을 계산하는 혁신적인 방식입니다.

**ColBERT Late Interaction 방식:**

```
Query: "Python 정렬"
  → Tokens: [Python] [정렬]
  → 쿼리 토큰 벡터: [v1] [v2]

Document: "Python sort 메서드 사용법"
  → Tokens: [Python] [sort] [메서드] [사용법]
  → 문서 토큰 벡터: [d1] [d2] [d3] [d4]
```

**MaxSim 계산:**
- 쿼리 토큰 v1(Python): max(sim(v1,d1), sim(v1,d2), sim(v1,d3), sim(v1,d4)) = max(0.95, 0.30, 0.10, 0.08) = **0.95**
- 쿼리 토큰 v2(정렬): max(sim(v2,d1), sim(v2,d2), sim(v2,d3), sim(v2,d4)) = max(0.20, 0.88, 0.15, 0.12) = **0.88**
- 최종 점수 = 0.95 + 0.88 = **1.83**

```python
from fastembed import LateInteractionTextEmbedding

# ColBERT 모델 로드
model = LateInteractionTextEmbedding("colbert-ir/colbertv2.0")

# 문서 인코딩 (토큰별 벡터)
doc_embeddings = list(model.embed(documents))

# 쿼리 인코딩
query_embedding = list(model.query_embed([query]))[0]

# MaxSim 계산으로 점수 산출
# (각 쿼리 토큰과 문서 토큰 간 최대 유사도의 합)
```

#### 3. LLM 기반 리랭커 (RankGPT, RankLLM)

```python
from llama_index.postprocessor.rankgpt_rerank import RankGPTRerank
from llama_index.llms.openai import OpenAI

# LLM 기반 리랭커
reranker = RankGPTRerank(
    top_n=5,
    llm=OpenAI(model="gpt-4o-mini")
)

# 결과 재정렬
reranked_nodes = reranker.postprocess_nodes(nodes, query_str=query)
```

**LLM 리랭커의 장단점:**
- ✅ 복잡한 추론 가능
- ✅ 명령어 기반 커스터마이징
- ❌ 느린 속도
- ❌ 높은 비용
- ❌ 일관성 문제

---

## 🔄 하이브리드 검색

### 하이브리드 검색이란?

**키워드 검색(BM25)**과 **시맨틱 검색**을 결합하여 두 방식의 장점을 모두 활용합니다.

```
                    사용자 쿼리
                        ↓
          ┌─────────────┴─────────────┐
          ↓                           ↓
   BM25 검색 (키워드)          시맨틱 검색 (벡터)
          ↓                           ↓
     Top-K 결과                  Top-K 결과
          ↓                           ↓
          └─────────────┬─────────────┘
                        ↓
              결과 융합 (RRF 또는 가중합)
                        ↓
                  융합된 후보군
                        ↓
              Cross-Encoder 리랭킹
                        ↓
                    최종 결과
```

### 왜 하이브리드가 필요한가?

| 검색 방식 | 강점 | 약점 |
|----------|------|------|
| **BM25** | 정확한 키워드 매칭, 고유명사, 코드, ID | 동의어, 의미 파악 불가 |
| **시맨틱** | 의미 이해, 동의어 매칭 | 희귀 키워드, 정확한 매칭 약함 |
| **하이브리드** | 두 장점 결합 | 복잡성 증가 |

### 결과 융합 방법

#### 1. Reciprocal Rank Fusion (RRF)

가장 널리 사용되는 융합 방법입니다.

```python
def reciprocal_rank_fusion(rankings: list, k: int = 60):
    """
    RRF 점수 계산

    Args:
        rankings: 각 검색 시스템의 결과 순위 리스트
        k: 스무딩 파라미터 (기본값 60)

    Returns:
        융합된 점수 딕셔너리
    """
    fused_scores = {}

    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            if doc_id not in fused_scores:
                fused_scores[doc_id] = 0
            # RRF 공식: 1 / (k + rank)
            fused_scores[doc_id] += 1 / (k + rank + 1)

    # 점수 기준 정렬
    sorted_docs = sorted(fused_scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_docs

# 사용 예시
bm25_results = ["doc_1", "doc_3", "doc_5", "doc_2", "doc_4"]
semantic_results = ["doc_2", "doc_1", "doc_4", "doc_3", "doc_6"]

fused = reciprocal_rank_fusion([bm25_results, semantic_results])
# 결과: doc_1과 doc_2가 두 검색 모두에서 상위권 → 최종 상위
```

#### 2. 가중 선형 결합

```python
def weighted_combination(bm25_scores: dict, semantic_scores: dict, alpha: float = 0.5):
    """
    정규화 후 가중 결합

    Args:
        bm25_scores: BM25 점수 딕셔너리
        semantic_scores: 시맨틱 점수 딕셔너리
        alpha: BM25 가중치 (1-alpha는 시맨틱 가중치)
    """
    # Min-Max 정규화
    def normalize(scores):
        min_s = min(scores.values())
        max_s = max(scores.values())
        return {k: (v - min_s) / (max_s - min_s + 1e-10) for k, v in scores.items()}

    norm_bm25 = normalize(bm25_scores)
    norm_semantic = normalize(semantic_scores)

    # 가중 결합
    combined = {}
    all_docs = set(norm_bm25.keys()) | set(norm_semantic.keys())

    for doc_id in all_docs:
        bm25_score = norm_bm25.get(doc_id, 0)
        semantic_score = norm_semantic.get(doc_id, 0)
        combined[doc_id] = alpha * bm25_score + (1 - alpha) * semantic_score

    return dict(sorted(combined.items(), key=lambda x: x[1], reverse=True))
```

### 하이브리드 검색 구현 (LangChain)

```python
from langchain.retrievers import BM25Retriever, EnsembleRetriever
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

# 문서 준비
documents = [...]  # Document 객체 리스트

# BM25 리트리버
bm25_retriever = BM25Retriever.from_documents(documents)
bm25_retriever.k = 50

# 시맨틱 리트리버
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(documents, embeddings)
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 50})

# 앙상블 리트리버 (RRF 사용)
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.5, 0.5]  # BM25와 시맨틱 동일 가중치
)

# 검색 실행
results = ensemble_retriever.invoke("검색 쿼리")
```

---

## 📊 주요 모델 비교

### 임베딩 모델 상세 비교

**성능 vs 비용 포지셔닝:**

| 성능 (MTEB) | 무료 (오픈소스) | 저비용 ($0.02) | 중간 ($0.06-0.10) | 고비용 ($0.12+) |
|-------------|----------------|----------------|-------------------|-----------------|
| 66+ | - | jina-v3 ★ | - | - |
| 64 | BGE-large | - | Cohere-v3 | voyage-3-large, text-embed-3-lg |
| 62 | BGE-M3, nomic-1.5 | - | voyage-3.5 | - |
| 56-58 | MiniLM-L6 | text-embed-3-sm | - | - |

★ = 추천 모델

### 리랭커 모델 상세 비교

**정확도 vs 속도 포지셔닝:**

| 정확도 (BEIR) | 매우 빠름 (10-50ms) | 빠름 (50-100ms) | 중간 (100-200ms) | 느림 (200ms+) |
|---------------|---------------------|-----------------|------------------|---------------|
| 64 | - | - | - | Cohere-v4-pro ★ |
| 62 | - | jina-v3 ★ | Cohere-v3.5 | Contextual-AI |
| 60 | - | voyage-rerank-2 | - | - |
| 58 | jina-v2 | - | BGE-large | - |
| 56 | ColBERTv2 | BGE-v2-m3 | - | - |
| 52-54 | ms-marco-MiniLM-L6, L12 | - | - | - |

★ = 추천 모델

### 용도별 추천 조합

| 용도 | 임베딩 모델 | 리랭커 | 예상 비용/월 |
|------|------------|--------|-------------|
| **프로토타입** | all-MiniLM-L6-v2 | ms-marco-MiniLM-L6 | $0 |
| **스타트업** | text-embedding-3-small | jina-reranker-v2 | $20-50 |
| **프로덕션 (균형)** | BGE-M3 + text-embed-3-sm | Cohere rerank-v3.5 | $100-200 |
| **프로덕션 (품질 우선)** | voyage-3-large | Cohere rerank-v4-pro | $300-500 |
| **엔터프라이즈** | jina-embeddings-v3 | jina-reranker-v3 | $500+ |
| **온프레미스** | BGE-large-en | BGE-reranker-large | 서버 비용만 |

---

## 🛠 환경 구축

### 필수 패키지 설치

```bash
# 기본 패키지
pip install sentence-transformers
pip install faiss-cpu  # 또는 faiss-gpu
pip install rank_bm25

# LangChain 통합
pip install langchain langchain-community langchain-openai
pip install langchain-cohere  # Cohere 리랭커

# LlamaIndex 통합
pip install llama-index
pip install llama-index-postprocessor-cohere-rerank
pip install llama-index-postprocessor-colbert-rerank

# 추가 리랭커
pip install flashrank  # 경량 리랭커
pip install rerankers  # 다양한 리랭커 통합

# ColBERT
pip install colbert-ai
pip install fastembed
```

### API 키 설정

```bash
# .env 파일
OPENAI_API_KEY=sk-...
COHERE_API_KEY=...
JINA_API_KEY=...
VOYAGE_API_KEY=...
```

```python
import os
from dotenv import load_dotenv

load_dotenv()

# API 키 로드
openai_key = os.getenv("OPENAI_API_KEY")
cohere_key = os.getenv("COHERE_API_KEY")
```

---

## 💻 실전 구현 가이드

### 1. 기본 시맨틱 검색 + 리랭킹 파이프라인

```python
from sentence_transformers import SentenceTransformer, CrossEncoder
import faiss
import numpy as np

class SemanticSearchRerank:
    def __init__(
        self,
        embedding_model: str = "BAAI/bge-large-en-v1.5",
        reranker_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"
    ):
        # 모델 로드
        self.embedder = SentenceTransformer(embedding_model)
        self.reranker = CrossEncoder(reranker_model)

        # 인덱스 및 문서 저장소
        self.index = None
        self.documents = []

    def index_documents(self, documents: list):
        """문서 인덱싱"""
        self.documents = documents

        # 임베딩 생성
        embeddings = self.embedder.encode(
            documents,
            show_progress_bar=True,
            normalize_embeddings=True
        )

        # FAISS 인덱스 생성
        dimension = embeddings.shape[1]
        self.index = faiss.IndexFlatIP(dimension)  # Inner Product (코사인 유사도)
        self.index.add(embeddings.astype('float32'))

        print(f"✅ {len(documents)}개 문서 인덱싱 완료")

    def search(
        self,
        query: str,
        top_k_retrieval: int = 100,
        top_k_rerank: int = 10
    ) -> list:
        """2단계 검색 수행"""

        # 1단계: 시맨틱 검색
        query_embedding = self.embedder.encode(
            [query],
            normalize_embeddings=True
        ).astype('float32')

        scores, indices = self.index.search(query_embedding, top_k_retrieval)

        # 후보 문서 추출
        candidates = [(self.documents[idx], scores[0][i])
                      for i, idx in enumerate(indices[0])]

        # 2단계: 리랭킹
        pairs = [[query, doc] for doc, _ in candidates]
        rerank_scores = self.reranker.predict(pairs)

        # 최종 정렬
        reranked = sorted(
            zip(candidates, rerank_scores),
            key=lambda x: x[1],
            reverse=True
        )[:top_k_rerank]

        results = []
        for (doc, original_score), rerank_score in reranked:
            results.append({
                "document": doc,
                "original_score": float(original_score),
                "rerank_score": float(rerank_score)
            })

        return results


# 사용 예시
if __name__ == "__main__":
    # 샘플 문서
    documents = [
        "Python에서 리스트를 정렬하려면 sort() 메서드나 sorted() 함수를 사용합니다.",
        "JavaScript에서 배열 정렬은 sort() 메서드를 사용합니다.",
        "Python의 sort()는 리스트를 제자리에서 정렬하고, sorted()는 새 리스트를 반환합니다.",
        "데이터베이스에서 정렬은 ORDER BY 절을 사용합니다.",
        "Python 리스트는 동적 배열로 구현되어 있습니다.",
        "정렬 알고리즘에는 퀵정렬, 병합정렬, 힙정렬 등이 있습니다.",
        "Python에서 reverse=True 옵션으로 내림차순 정렬이 가능합니다.",
        "pandas DataFrame의 sort_values() 메서드로 데이터프레임을 정렬합니다.",
    ]

    # 파이프라인 생성
    pipeline = SemanticSearchRerank()
    pipeline.index_documents(documents)

    # 검색
    query = "파이썬에서 리스트 내림차순 정렬하는 방법"
    results = pipeline.search(query, top_k_retrieval=5, top_k_rerank=3)

    print(f"\n🔍 쿼리: {query}\n")
    for i, result in enumerate(results, 1):
        print(f"{i}. [리랭크: {result['rerank_score']:.4f}] {result['document'][:60]}...")
```

### 2. Cohere Rerank API 사용

```python
import cohere

class CohereReranker:
    def __init__(self, api_key: str):
        self.client = cohere.Client(api_key)

    def rerank(
        self,
        query: str,
        documents: list,
        top_n: int = 10,
        model: str = "rerank-v3.5"
    ) -> list:
        """
        Cohere Rerank API를 사용한 리랭킹

        Args:
            query: 검색 쿼리
            documents: 문서 리스트 (문자열 또는 딕셔너리)
            top_n: 반환할 상위 문서 수
            model: 사용할 모델 (rerank-v3.5, rerank-v3.0, rerank-english-v2.0 등)
        """
        response = self.client.rerank(
            query=query,
            documents=documents,
            top_n=top_n,
            model=model,
            return_documents=True
        )

        results = []
        for item in response.results:
            results.append({
                "index": item.index,
                "document": item.document.text if hasattr(item.document, 'text') else documents[item.index],
                "relevance_score": item.relevance_score
            })

        return results


# 사용 예시
reranker = CohereReranker(api_key="your-api-key")

query = "클라우드 컴퓨팅의 장점"
documents = [
    "클라우드 컴퓨팅은 인터넷을 통해 컴퓨팅 리소스를 제공합니다.",
    "AWS, Azure, GCP는 주요 클라우드 서비스 제공업체입니다.",
    "클라우드의 장점은 확장성, 비용 절감, 유연성입니다.",
    "온프레미스 서버는 직접 관리해야 합니다.",
    "클라우드 보안은 공동 책임 모델을 따릅니다."
]

results = reranker.rerank(query, documents, top_n=3)

for result in results:
    print(f"점수: {result['relevance_score']:.4f} | {result['document']}")
```

### 3. 하이브리드 검색 + 리랭킹 전체 파이프라인

```python
from rank_bm25 import BM25Okapi
from sentence_transformers import SentenceTransformer, CrossEncoder
import faiss
import numpy as np
from typing import List, Dict
import re

class HybridSearchRerank:
    def __init__(
        self,
        embedding_model: str = "BAAI/bge-large-en-v1.5",
        reranker_model: str = "BAAI/bge-reranker-base"
    ):
        self.embedder = SentenceTransformer(embedding_model)
        self.reranker = CrossEncoder(reranker_model)

        self.documents = []
        self.bm25 = None
        self.faiss_index = None

    def _tokenize(self, text: str) -> List[str]:
        """간단한 토크나이저"""
        text = text.lower()
        tokens = re.findall(r'\w+', text)
        return tokens

    def index_documents(self, documents: List[str]):
        """문서 인덱싱 (BM25 + 벡터)"""
        self.documents = documents

        # BM25 인덱싱
        tokenized_docs = [self._tokenize(doc) for doc in documents]
        self.bm25 = BM25Okapi(tokenized_docs)

        # 벡터 인덱싱
        embeddings = self.embedder.encode(
            documents,
            normalize_embeddings=True,
            show_progress_bar=True
        )

        dimension = embeddings.shape[1]
        self.faiss_index = faiss.IndexFlatIP(dimension)
        self.faiss_index.add(embeddings.astype('float32'))

        print(f"✅ {len(documents)}개 문서 하이브리드 인덱싱 완료")

    def _reciprocal_rank_fusion(
        self,
        rankings: List[List[int]],
        k: int = 60
    ) -> List[tuple]:
        """RRF 기반 결과 융합"""
        scores = {}

        for ranking in rankings:
            for rank, doc_idx in enumerate(ranking):
                if doc_idx not in scores:
                    scores[doc_idx] = 0
                scores[doc_idx] += 1 / (k + rank + 1)

        return sorted(scores.items(), key=lambda x: x[1], reverse=True)

    def search(
        self,
        query: str,
        top_k_retrieval: int = 50,
        top_k_fusion: int = 100,
        top_k_rerank: int = 10,
        bm25_weight: float = 0.5
    ) -> List[Dict]:
        """하이브리드 검색 + 리랭킹"""

        # 1단계: BM25 검색
        tokenized_query = self._tokenize(query)
        bm25_scores = self.bm25.get_scores(tokenized_query)
        bm25_ranking = np.argsort(bm25_scores)[::-1][:top_k_retrieval].tolist()

        # 2단계: 시맨틱 검색
        query_embedding = self.embedder.encode(
            [query],
            normalize_embeddings=True
        ).astype('float32')

        _, semantic_indices = self.faiss_index.search(query_embedding, top_k_retrieval)
        semantic_ranking = semantic_indices[0].tolist()

        # 3단계: RRF 융합
        fused_results = self._reciprocal_rank_fusion(
            [bm25_ranking, semantic_ranking]
        )[:top_k_fusion]

        # 4단계: 리랭킹
        candidate_docs = [(idx, self.documents[idx]) for idx, _ in fused_results]
        pairs = [[query, doc] for _, doc in candidate_docs]
        rerank_scores = self.reranker.predict(pairs)

        # 최종 결과 정렬
        final_results = sorted(
            zip(candidate_docs, rerank_scores),
            key=lambda x: x[1],
            reverse=True
        )[:top_k_rerank]

        return [
            {
                "index": idx,
                "document": doc,
                "rerank_score": float(score)
            }
            for (idx, doc), score in final_results
        ]


# 사용 예시
if __name__ == "__main__":
    documents = [
        "Python 3.12에서 새로 추가된 기능들을 살펴봅니다.",
        "파이썬 리스트 컴프리헨션의 고급 사용법",
        "Python의 asyncio를 활용한 비동기 프로그래밍",
        "Django REST Framework로 API 구축하기",
        "FastAPI vs Flask: Python 웹 프레임워크 비교",
        "Python 3.12의 성능 개선 사항",
        "파이썬에서 멀티스레딩과 멀티프로세싱의 차이",
        "Python 타입 힌트 완벽 가이드",
    ]

    pipeline = HybridSearchRerank()
    pipeline.index_documents(documents)

    query = "Python 3.12 새로운 기능"
    results = pipeline.search(query, top_k_rerank=3)

    print(f"\n🔍 쿼리: {query}\n")
    for i, result in enumerate(results, 1):
        print(f"{i}. [점수: {result['rerank_score']:.4f}]")
        print(f"   {result['document']}\n")
```

---

## 🔗 프레임워크 통합

### LangChain 통합

#### ContextualCompressionRetriever 사용

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from langchain.retrievers import ContextualCompressionRetriever
from langchain_cohere import CohereRerank
from langchain.schema import Document

# 문서 준비
docs = [
    Document(page_content="Python의 sort() 메서드는 리스트를 정렬합니다."),
    Document(page_content="JavaScript의 sort() 함수는 배열을 정렬합니다."),
    Document(page_content="Python에서 sorted() 함수는 새 리스트를 반환합니다."),
    # ... 더 많은 문서
]

# 벡터 스토어 생성
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(docs, embeddings)

# 기본 리트리버
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# Cohere 리랭커로 컨텍스트 압축
reranker = CohereRerank(
    model="rerank-v3.5",
    top_n=5
)

# 압축 리트리버 생성
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

# 검색 실행
query = "Python 리스트 정렬 방법"
results = compression_retriever.invoke(query)

for doc in results:
    print(f"📄 {doc.page_content}")
```

#### Cross-Encoder 리랭커 사용

```python
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

# Cross-Encoder 모델 로드
cross_encoder = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")

# 리랭커 생성
reranker = CrossEncoderReranker(
    model=cross_encoder,
    top_n=5
)

# 압축 리트리버 생성
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)
```

#### 하이브리드 검색 + 리랭킹 (LangChain)

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# BM25 리트리버
bm25_retriever = BM25Retriever.from_documents(docs)
bm25_retriever.k = 20

# 시맨틱 리트리버
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# 앙상블 (하이브리드)
ensemble = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.5, 0.5]
)

# 리랭킹 추가
final_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=ensemble
)

# RAG 체인에 통합
from langchain.chains import RetrievalQA
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=final_retriever
)

answer = qa_chain.invoke({"query": "Python 리스트를 정렬하는 방법은?"})
```

### LlamaIndex 통합

#### Node Postprocessor 사용

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core.postprocessor import SentenceTransformerRerank
from llama_index.postprocessor.cohere_rerank import CohereRerank

# 문서 로드 및 인덱싱
documents = SimpleDirectoryReader("./data").load_data()
index = VectorStoreIndex.from_documents(documents)

# 방법 1: Sentence Transformer 리랭커
st_reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n=5
)

# 방법 2: Cohere 리랭커
cohere_reranker = CohereRerank(
    api_key="your-api-key",
    model="rerank-v3.5",
    top_n=5
)

# 쿼리 엔진에 적용
query_engine = index.as_query_engine(
    similarity_top_k=20,  # 1단계에서 20개 검색
    node_postprocessors=[cohere_reranker]  # 2단계 리랭킹
)

response = query_engine.query("Python 리스트 정렬 방법은?")
print(response)
```

#### 다중 리랭커 적용

```python
from llama_index.core.postprocessor import SimilarityPostprocessor

# 여러 후처리기 체인
query_engine = index.as_query_engine(
    similarity_top_k=50,
    node_postprocessors=[
        # 1. 유사도 임계값 필터
        SimilarityPostprocessor(similarity_cutoff=0.5),
        # 2. 리랭킹
        cohere_reranker
    ]
)
```

#### ColBERT 리랭커 사용

```python
from llama_index.postprocessor.colbert_rerank import ColbertRerank

colbert_reranker = ColbertRerank(
    model="colbert-ir/colbertv2.0",
    top_n=5
)

query_engine = index.as_query_engine(
    similarity_top_k=20,
    node_postprocessors=[colbert_reranker]
)
```

---

## ⚡ 성능 최적화

### 리랭킹 최적화 전략

#### 1. 후보 수 최적화

```python
# 용도별 권장 후보 수
CANDIDATE_CONFIGS = {
    "chat_application": {
        "retrieval_k": 50,   # 1단계 검색
        "rerank_k": 5        # 최종 결과
    },
    "web_search": {
        "retrieval_k": 100,
        "rerank_k": 10
    },
    "document_analysis": {
        "retrieval_k": 200,
        "rerank_k": 20
    }
}
```

#### 2. 배치 처리

```python
from sentence_transformers import CrossEncoder
import numpy as np

class BatchReranker:
    def __init__(self, model_name: str, batch_size: int = 32):
        self.model = CrossEncoder(model_name)
        self.batch_size = batch_size

    def rerank_batch(self, query: str, documents: list) -> list:
        """배치 처리로 리랭킹 속도 향상"""
        pairs = [[query, doc] for doc in documents]

        # 배치 단위로 처리
        all_scores = []
        for i in range(0, len(pairs), self.batch_size):
            batch = pairs[i:i + self.batch_size]
            scores = self.model.predict(batch, show_progress_bar=False)
            all_scores.extend(scores)

        return all_scores
```

#### 3. 캐싱 전략

```python
from functools import lru_cache
import hashlib

class CachedReranker:
    def __init__(self, reranker, cache_size: int = 1000):
        self.reranker = reranker
        self.cache = {}
        self.cache_size = cache_size

    def _get_cache_key(self, query: str, doc: str) -> str:
        return hashlib.md5(f"{query}||{doc}".encode()).hexdigest()

    def rerank(self, query: str, documents: list) -> list:
        scores = []
        uncached_pairs = []
        uncached_indices = []

        # 캐시 확인
        for i, doc in enumerate(documents):
            key = self._get_cache_key(query, doc)
            if key in self.cache:
                scores.append((i, self.cache[key]))
            else:
                uncached_pairs.append([query, doc])
                uncached_indices.append(i)

        # 캐시되지 않은 것만 계산
        if uncached_pairs:
            new_scores = self.reranker.predict(uncached_pairs)
            for idx, score in zip(uncached_indices, new_scores):
                key = self._get_cache_key(query, documents[idx])
                self.cache[key] = score
                scores.append((idx, score))

        # 정렬하여 반환
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores
```

#### 4. 모델 양자화

```python
from sentence_transformers import CrossEncoder
import torch

# FP16 사용
model = CrossEncoder(
    "cross-encoder/ms-marco-MiniLM-L-6-v2",
    device="cuda"
)
model.model.half()  # FP16 변환

# 또는 ONNX 변환으로 추론 가속
# pip install optimum onnxruntime-gpu
from optimum.onnxruntime import ORTModelForSequenceClassification

ort_model = ORTModelForSequenceClassification.from_pretrained(
    "cross-encoder/ms-marco-MiniLM-L-6-v2",
    export=True
)
```

### 지연 시간 최적화

**현재 상태 (최적화 전):**
- 시맨틱 검색 (100개): 50ms
- 리랭킹 (100 → 10): 400ms
- 총 지연 시간: **450ms**

**최적화 단계:**

| 단계 | 최적화 방법 | 리랭킹 시간 |
|------|------------|------------|
| 1 | 후보 수 축소 (100 → 50) | 400ms → 200ms |
| 2 | 경량 모델 사용 (MiniLM-L6) | 200ms → 80ms |
| 3 | GPU + FP16 | 80ms → 40ms |
| 4 | ONNX 최적화 | 40ms → 25ms |

**최종 결과:** 50ms + 25ms = **75ms** (83% 개선)

---

## 🚀 프로덕션 배포

### 아키텍처 설계

**프로덕션 배포 아키텍처:**

```
클라이언트
    ↓
API Gateway (Rate Limiting, Auth)
    ↓
검색 서비스 (Kubernetes)
    ↓
로드 밸런서
    ↓
┌──────────────────────────────────┐
│  Pod 1 (Search)   Pod 2 (Search) │
│  Pod 3 (Rerank)   Pod 4 (Rerank) │
└──────────────────────────────────┘
    ↓                 ↓
벡터 DB          Redis Cache
(Pinecone/Qdrant)   (리랭크 캐시)
```

### FastAPI 서비스 예시

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List
import asyncio
from functools import lru_cache

app = FastAPI(title="Semantic Search & Rerank API")

class SearchRequest(BaseModel):
    query: str
    top_k: int = 10
    use_rerank: bool = True

class SearchResult(BaseModel):
    document: str
    score: float
    index: int

class SearchResponse(BaseModel):
    results: List[SearchResult]
    query: str
    latency_ms: float

# 모델 싱글톤
@lru_cache()
def get_search_pipeline():
    from pipeline import HybridSearchRerank
    pipeline = HybridSearchRerank()
    pipeline.load_index("./index")
    return pipeline

@app.post("/search", response_model=SearchResponse)
async def search(request: SearchRequest):
    import time
    start = time.time()

    pipeline = get_search_pipeline()

    # 비동기 검색 (블로킹 작업을 스레드풀에서 실행)
    results = await asyncio.get_event_loop().run_in_executor(
        None,
        lambda: pipeline.search(
            query=request.query,
            top_k_rerank=request.top_k if request.use_rerank else 0
        )
    )

    latency = (time.time() - start) * 1000

    return SearchResponse(
        results=[
            SearchResult(
                document=r["document"],
                score=r["rerank_score"],
                index=r["index"]
            )
            for r in results
        ],
        query=request.query,
        latency_ms=latency
    )

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

### Docker 배포

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 시스템 의존성
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Python 패키지
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 모델 미리 다운로드
RUN python -c "from sentence_transformers import SentenceTransformer, CrossEncoder; \
    SentenceTransformer('BAAI/bge-large-en-v1.5'); \
    CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')"

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  search-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - COHERE_API_KEY=${COHERE_API_KEY}
    volumes:
      - ./index:/app/index
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  redis_data:
```

### 모니터링 및 메트릭

```python
from prometheus_client import Counter, Histogram, generate_latest
import time

# 메트릭 정의
SEARCH_REQUESTS = Counter('search_requests_total', 'Total search requests')
SEARCH_LATENCY = Histogram('search_latency_seconds', 'Search latency',
                           buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.5])
RERANK_LATENCY = Histogram('rerank_latency_seconds', 'Rerank latency',
                           buckets=[0.05, 0.1, 0.25, 0.5, 1.0])

def search_with_metrics(query: str, pipeline):
    SEARCH_REQUESTS.inc()

    start = time.time()

    # 시맨틱 검색
    candidates = pipeline.semantic_search(query)
    search_time = time.time() - start
    SEARCH_LATENCY.observe(search_time)

    # 리랭킹
    rerank_start = time.time()
    results = pipeline.rerank(query, candidates)
    rerank_time = time.time() - rerank_start
    RERANK_LATENCY.observe(rerank_time)

    return results

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

---

## 🔧 트러블슈팅

### 일반적인 문제 및 해결

#### 1. 리랭킹 결과가 기대와 다른 경우

```python
# 문제: 리랭킹 후에도 관련 없는 문서가 상위에 있음

# 해결 1: 더 강력한 리랭커 모델 사용
# ms-marco-MiniLM-L6 → BGE-reranker-large 또는 Cohere

# 해결 2: 1단계 검색 후보 수 증가
retrieval_k = 100  # 50 → 100

# 해결 3: 도메인 특화 파인튜닝
# 해당 도메인 데이터로 리랭커 추가 학습
```

#### 2. 메모리 부족 (OOM)

```python
# 문제: GPU 메모리 부족

# 해결 1: 배치 크기 축소
batch_size = 16  # 32 → 16

# 해결 2: 경량 모델 사용
model = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-2-v2")  # L-6 → L-2

# 해결 3: FP16 사용
model.model.half()

# 해결 4: 그래디언트 체크포인팅 (학습 시)
model.model.gradient_checkpointing_enable()
```

#### 3. 지연 시간이 너무 높음

```python
# 문제: 응답 시간 > 1초

# 해결 1: 후보 수 축소
top_k_retrieval = 30  # 100 → 30

# 해결 2: ONNX 변환
from optimum.onnxruntime import ORTModelForSequenceClassification
ort_model = ORTModelForSequenceClassification.from_pretrained(model_name, export=True)

# 해결 3: 비동기 처리
import asyncio
results = await asyncio.gather(
    search_service_1(query),
    search_service_2(query)
)

# 해결 4: 결과 캐싱
@lru_cache(maxsize=1000)
def cached_rerank(query_hash, doc_hash):
    ...
```

#### 4. 다국어 처리 문제

```python
# 문제: 한국어/영어 혼합 쿼리에서 성능 저하

# 해결: 다국어 지원 모델 사용
embedding_model = "BAAI/bge-m3"  # 다국어 임베딩
reranker_model = "BAAI/bge-reranker-v2-m3"  # 다국어 리랭커

# 또는 Cohere 다국어 모델
# model="rerank-multilingual-v3.0"
```

### 성능 디버깅 체크리스트

```
□ 1단계 검색 결과 품질 확인
  - 관련 문서가 Top-100에 포함되는지?
  - 시맨틱 검색 점수 분포 확인

□ 리랭커 점수 분포 확인
  - 관련/비관련 문서 간 점수 차이가 있는지?
  - 점수가 한쪽으로 치우쳐 있지 않은지?

□ 지연 시간 프로파일링
  - 임베딩 생성: ?ms
  - 벡터 검색: ?ms
  - 리랭킹: ?ms
  - 어느 단계가 병목인지?

□ 메모리 사용량 확인
  - GPU 메모리 사용률
  - 배치 크기가 적절한지?

□ 캐시 적중률 확인
  - 반복 쿼리에 대한 캐시 활용
```

---

## ❓ FAQ

### Q1: 리랭킹을 항상 사용해야 하나요?

**A:** 아닙니다. 다음 상황에서는 리랭킹이 불필요할 수 있습니다:
- 문서 수가 적은 경우 (< 1000개)
- 실시간 응답이 중요한 경우
- 시맨틱 검색만으로도 충분한 품질인 경우

리랭킹은 **품질 vs 지연 시간** 트레이드오프입니다.

### Q2: 어떤 리랭커를 선택해야 하나요?

**A:**
- **빠른 프로토타입**: `ms-marco-MiniLM-L-6-v2` (무료, 빠름)
- **프로덕션 (균형)**: `Cohere rerank-v3.5` (좋은 성능, 합리적 비용)
- **최고 품질**: `Cohere rerank-v4-pro` 또는 `jina-reranker-v3`
- **온프레미스**: `BGE-reranker-large`

### Q3: 하이브리드 검색이 항상 더 좋은가요?

**A:** 대부분의 경우 예, 하지만:
- 도메인에 따라 BM25 가중치 조정 필요
- 코드/ID 검색: BM25 가중치 ↑
- 자연어 검색: 시맨틱 가중치 ↑

### Q4: 리랭킹 후보 수는 얼마가 적절한가요?

**A:**
- 채팅 애플리케이션: 50개
- 웹 검색: 100개
- 종합 분석: 150-200개

일반적으로 **50-100개**가 최적의 균형점입니다.

### Q5: 임베딩 모델과 리랭커 모델은 같은 것을 써야 하나요?

**A:** 아닙니다. 서로 다른 모델을 사용해도 됩니다:
- 임베딩: BGE-large (빠른 벡터 생성)
- 리랭커: Cohere rerank (높은 정확도)

다만, 같은 제공사의 모델을 사용하면 일관성이 높을 수 있습니다.

### Q6: 커스텀 데이터로 리랭커를 파인튜닝할 수 있나요?

**A:** 예, Sentence Transformers v4부터 리랭커 학습이 가능합니다:

```python
from sentence_transformers import CrossEncoder
from sentence_transformers.cross_encoder.trainer import CrossEncoderTrainer

# 학습 데이터 준비
train_samples = [
    {"query": "...", "positive": "...", "negative": "..."},
    ...
]

# 파인튜닝
model = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
trainer = CrossEncoderTrainer(model, train_samples)
trainer.train()
```

---

## 📚 참고 자료

### 공식 문서

- [Sentence Transformers - Cross-Encoders](https://www.sbert.net/examples/cross_encoder/applications/README.html)
- [Cohere Rerank API](https://docs.cohere.com/docs/rerank)
- [Jina AI Reranker](https://jina.ai/reranker/)
- [LangChain - Contextual Compression](https://python.langchain.com/docs/how_to/contextual_compression/)
- [LlamaIndex - Node Postprocessors](https://docs.llamaindex.ai/en/stable/module_guides/querying/node_postprocessors/)
- [Pinecone - Rerankers and Two-Stage Retrieval](https://www.pinecone.io/learn/series/rag/rerankers/)

### 모델 허브

- [Hugging Face - Cross-Encoders](https://huggingface.co/cross-encoder)
- [ColBERTv2](https://huggingface.co/colbert-ir/colbertv2.0)

### 튜토리얼 및 블로그

- [Hugging Face - Training Rerankers with Sentence Transformers v4](https://huggingface.co/blog/train-reranker)
- [Weaviate - Late Interaction Overview](https://weaviate.io/blog/late-interaction-overview)
- [NVIDIA - Enhancing RAG Pipelines with Re-Ranking](https://developer.nvidia.com/blog/enhancing-rag-pipelines-with-re-ranking/)
- [Superlinked - Optimizing RAG with Hybrid Search & Reranking](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking)

### 연구 논문

- [ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT (SIGIR 2020)](https://arxiv.org/abs/2004.12832)
- [Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks](https://arxiv.org/abs/1908.10084)
- [Text and Code Embeddings by Contrastive Pre-Training](https://arxiv.org/abs/2201.10005)

### 벤치마크

- [MTEB Leaderboard (임베딩 모델)](https://huggingface.co/spaces/mteb/leaderboard)
- [BEIR Benchmark (검색 모델)](https://github.com/beir-cellar/beir)


