# Knowledge Graph & Graph RAG 완전 가이드

> 개념부터 PostgreSQL 실전 구현까지 — 한국어 개발자를 위한 상세 가이드

[![Python](https://img.shields.io/badge/Python-3.10+-yellow.svg)](https://www.python.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-blue.svg)](https://www.postgresql.org/)
[![pgvector](https://img.shields.io/badge/pgvector-0.7+-green.svg)](https://github.com/pgvector/pgvector)
[![License](https://img.shields.io/badge/License-MIT-red.svg)](LICENSE)

## 📋 목차

- [RAG란 무엇인가](#-rag란-무엇인가)
- [Standard RAG의 한계](#-standard-rag의-한계)
- [Knowledge Graph란](#-knowledge-graph란)
- [Graph RAG란](#-graph-rag란)
- [Knowledge Graph vs Graph RAG](#-knowledge-graph-vs-graph-rag)
- [Knowledge Graph 핵심 구성 요소](#-knowledge-graph-핵심-구성-요소)
- [Graph RAG 구현 방식 비교](#-graph-rag-구현-방식-비교)
- [Microsoft GraphRAG 상세](#-microsoft-graphrag-상세)
- [LightRAG 상세](#-lightrag-상세)
- [PostgreSQL로 Knowledge Graph 구현하기](#-postgresql로-knowledge-graph-구현하기)
- [하이브리드 검색 — 벡터 + 키워드 통합](#-하이브리드-검색--벡터--키워드-통합)
- [Graph RAG + Vector 검색 통합 방안](#-graph-rag--vector-검색-통합-방안)
- [Python 통합 구현](#-python-통합-구현)
- [한국어 임베딩 모델 선택 가이드](#-한국어-임베딩-모델-선택-가이드)
- [언제 무엇을 선택할까](#-언제-무엇을-선택할까)
- [언어 선택 — Python vs JVM](#-언어-선택--python-vs-jvm-javakotlin)
- [청킹 전략 — Graph RAG의 필수 요건](#️-청킹-전략--graph-rag의-필수-요건)
- [PDF 파서 선택 가이드](#-pdf-파서-선택-가이드)
- [트러블슈팅](#-트러블슈팅)
- [참고 자료](#-참고-자료)

---

## 🎯 RAG란 무엇인가

**RAG(Retrieval-Augmented Generation)** 는 LLM(대형 언어 모델)이 답변을 생성할 때, 외부 지식 베이스에서 관련 정보를 **검색(Retrieve)** 하여 함께 활용하는 기법입니다.

```
[사용자 질문]
      ↓
[검색 엔진] → 지식 베이스에서 관련 문서 검색
      ↓
[LLM] → 검색된 문서 + 질문을 함께 받아 답변 생성
      ↓
[답변]
```

### RAG가 필요한 이유

LLM은 학습 데이터에 없는 최신 정보나 특정 도메인 지식(사내 문서, 전문 규정, 제품 매뉴얼 등)에 대해 정확한 답변을 하기 어렵습니다. RAG는 이 문제를 해결합니다.

| 방식 | 장점 | 단점 |
|------|------|------|
| **LLM 단독** | 빠르고 간단 | 학습 데이터 외 정보 모름, 할루시네이션 발생 |
| **Fine-tuning** | 도메인 특화 | 비용 높음, 재학습 필요, 지식 업데이트 어려움 |
| **RAG** | 실시간 지식 반영, 출처 추적 가능 | 검색 품질에 의존 |

### Standard RAG 동작 방식

```
[인덱싱 단계]
문서 → 청킹(Chunking) → 임베딩(Embedding) → 벡터 DB 저장

[검색 단계]
질문 → 임베딩 → 벡터 유사도 검색 → top-k 청크 반환 → LLM 입력
```

---

## ⚠️ Standard RAG의 한계

Standard RAG는 강력하지만, 문서 내 **관계(Relationship)** 를 파악하지 못하는 근본적인 한계가 있습니다.

### 한계 1: 독립적 청크 검색

```
문서 구조:
  [용어 정의 섹션] ──정의──► [핵심 개념 A]
  [핵심 개념 A]   ──조건──► [예외 규정 섹션]
  [예외 규정 섹션] ──참조──► [별첨 표 1]

질문: "핵심 개념 A의 예외는 무엇인가요?"

Standard RAG:
  → "핵심 개념 A" 관련 청크 검색
  → ❌ "예외 규정 섹션"이 별도 청크로 분리되어 있어 누락 가능
  → ❌ "별첨 표 1"은 아예 포함 안 됨
```

### 한계 2: 다단계 추론 불가

```
질문: "A 기능을 사용하려면 어떤 사전 조건이 필요한가요?"

실제 문서 관계:
  A 기능 → 의존 → B 모듈 → 의존 → C 라이브러리 → 요구 → 특정 OS 버전

Standard RAG:
  → "A 기능" 청크만 검색
  → ❌ B → C → OS 버전으로 이어지는 체인 탐색 불가
```

### 한계 3: 전체 맥락 파악 어려움

```
질문: "이 시스템의 전반적인 아키텍처를 설명해줘"

Standard RAG:
  → 특정 청크 몇 개만 반환
  → ❌ 시스템 전체 구조(컴포넌트 간 관계)를 조합해서 설명하지 못함
```

### 언제 Standard RAG로 충분한가

- FAQ, 블로그, 뉴스 등 독립적인 콘텐츠
- 단순 키워드 검색이 목적인 경우
- 문서 간 상호 참조가 거의 없는 경우
- 빠른 응답 속도가 최우선인 경우

---

## 🧠 Knowledge Graph란

**Knowledge Graph(지식 그래프)** 는 현실 세계의 개념, 개체(Entity), 사건을 **노드(Node)** 로, 그것들 사이의 관계를 **엣지(Edge)** 로 표현한 구조화된 지식 표현 방식입니다.

### 직관적 이해

일반적인 텍스트와 Knowledge Graph의 차이:

```
[텍스트]
"Python은 Guido van Rossum이 만든 프로그래밍 언어이며,
 Django는 Python 기반의 웹 프레임워크다."

[Knowledge Graph]
Python ──[만든사람]──► Guido van Rossum
Python ──[분류]──────► 프로그래밍 언어
Django ──[기반언어]──► Python
Django ──[분류]──────► 웹 프레임워크
```

텍스트는 순차적으로 읽어야 의미를 파악하지만, Knowledge Graph는 어떤 노드에서 시작하든 관계를 따라 탐색할 수 있습니다.

### 실제 활용 예시

**기술 문서 Knowledge Graph**:
```
[API 엔드포인트: /users/{id}]
      │
      ├─[요청방법]──► GET, PUT, DELETE
      ├─[인증필요]──► OAuth 2.0 토큰
      ├─[응답형식]──► UserResponse 스키마
      │                    │
      │                    └─[포함필드]──► id, name, email, createdAt
      └─[에러코드]──► 404 (Not Found), 403 (Forbidden)
```

**전자상거래 Knowledge Graph**:
```
[스마트폰 A]
      │
      ├─[브랜드]────► 제조사 X
      ├─[호환악세]──► 케이스 B, 충전기 C, 이어폰 D
      ├─[경쟁상품]──► 스마트폰 E, 스마트폰 F
      └─[카테고리]──► 모바일 > 스마트폰 > 플래그십
```

### Google Knowledge Graph

Google이 검색에서 사용하는 Knowledge Graph가 대표적인 예입니다.
"아인슈타인"을 검색하면 단순 텍스트 결과가 아니라 출생일, 국적, 업적, 관련 인물 등 구조화된 정보가 함께 표시되는 것이 Knowledge Graph의 활용입니다.

---

## 🔗 Graph RAG란

**Graph RAG** 는 RAG 시스템에 Knowledge Graph를 결합하여 검색 능력을 확장한 기법입니다.

Standard RAG가 "벡터 유사도로 관련 청크 검색"에 그친다면, Graph RAG는 Knowledge Graph의 **관계 정보를 활용해 연결된 컨텍스트를 함께 가져옵니다.**

### Graph RAG 동작 방식

```
[인덱싱 단계]
문서
  ↓ 파싱/청킹
텍스트 청크
  ↓ LLM 또는 규칙 기반
엔티티(개념) 추출 + 관계 추출
  ↓
Knowledge Graph 구축 (노드 + 엣지)
  ↓
벡터 임베딩 + 그래프 저장

[검색 단계]
질문
  ↓
① 벡터 검색 → 관련 청크 top-k
  ↓
② 해당 청크의 엔티티 확인
  ↓
③ Knowledge Graph 탐색 → 연관 엔티티 및 청크 확장
  ↓
④ 원래 top-k + 그래프 확장 컨텍스트 → LLM
  ↓
답변 (관계 정보까지 반영된 풍부한 답변)
```

### Standard RAG vs Graph RAG 비교

| 구분 | Standard RAG | Graph RAG |
|------|-------------|-----------|
| **검색 방식** | 벡터 유사도 | 벡터 유사도 + 그래프 탐색 |
| **컨텍스트 범위** | 유사한 청크들 | 유사한 청크 + 관계로 연결된 청크들 |
| **다단계 추론** | ❌ 불가 | ✅ 그래프 체인 탐색 |
| **전체 구조 파악** | ❌ 어려움 | ✅ 커뮤니티 요약 활용 |
| **구현 복잡도** | 낮음 | 중간-높음 |
| **인제스트 비용** | 낮음 | 높음 (관계 추출 필요) |
| **검색 속도** | 빠름 | 상대적으로 느림 |
| **적합한 문서** | 독립적 콘텐츠 | 상호 참조가 많은 문서 |

---

## 🔍 Knowledge Graph vs Graph RAG

> 두 개념을 혼동하기 쉽습니다. 명확히 구분하세요.

**Knowledge Graph**는 지식을 표현하는 **데이터 구조/모델**입니다.
**Graph RAG**는 Knowledge Graph를 검색에 활용하는 **기법/시스템**입니다.

```
Knowledge Graph          Graph RAG
─────────────────        ──────────────────────────────
데이터 구조               검색 + 생성 시스템

노드: 엔티티, 개념        Knowledge Graph를 구축하고
엣지: 관계, 속성          벡터 검색과 결합하여
                          LLM이 더 좋은 답변을 생성하도록
                          하는 전체 파이프라인
```

**비유**:
- Knowledge Graph = 도서관의 **색인 카드 시스템** (모든 책의 관계 정보)
- Graph RAG = 색인 카드를 활용해서 **필요한 정보를 찾아주는 사서**

---

## 🏗 Knowledge Graph 핵심 구성 요소

### 1. 엔티티(Entity) — 노드

Knowledge Graph의 노드. 실세계의 개념, 개체, 사건을 나타냅니다.

```
엔티티 유형 예시:
  Person    : 인물 (개발자, 저자, 관련 인물)
  Product   : 제품, 서비스
  Concept   : 추상적 개념 (알고리즘, 패턴, 원칙)
  Event     : 사건, 버전 릴리즈
  Location  : 장소, 조직
  Document  : 문서, 섹션, 조항
```

### 2. 관계(Relation) — 엣지

노드 간의 연결. 방향성이 있으며 의미를 가집니다.

```
관계 유형 예시:
  IS_A          : 상위 개념 관계    (Python IS_A 프로그래밍언어)
  PART_OF       : 구성 요소 관계    (엔진 PART_OF 자동차)
  DEPENDS_ON    : 의존 관계         (Django DEPENDS_ON Python)
  CREATED_BY    : 생성자 관계       (Python CREATED_BY Guido)
  REFERS_TO     : 참조 관계         (섹션3 REFERS_TO 섹션7)
  CONTRADICTS   : 상충 관계         (정책A CONTRADICTS 정책B)
  REQUIRES      : 선행 조건 관계    (배포 REQUIRES 테스트통과)
  RELATED_TO    : 일반 연관 관계    (기능A RELATED_TO 기능B)
```

### 3. 트리플(Triple)

Knowledge Graph의 기본 단위. `(주어, 관계, 목적어)` 형태:

```
(Python, CREATED_BY, Guido van Rossum)
(Django, DEPENDS_ON, Python)
(Django, IS_A, 웹프레임워크)
(웹프레임워크, IS_A, 소프트웨어)
```

### 4. 커뮤니티(Community)

밀접하게 연결된 노드들의 그룹. Microsoft GraphRAG에서 핵심 개념입니다.

```
[커뮤니티 예시: Python 생태계]
  Python, Django, Flask, FastAPI, Pip, PyPI, Guido van Rossum
  → 이 그룹 전체를 요약한 "커뮤니티 요약" 생성
  → 전체 데이터셋에 대한 광범위한 질문 답변에 활용
```

### 5. 그래프 탐색 깊이 (Hop)

```
1-hop: 직접 연결된 노드만 탐색
  Django → (DEPENDS_ON) → Python

2-hop: 2단계 연결까지 탐색
  Django → Python → (CREATED_BY) → Guido van Rossum

n-hop: n단계 연결까지 탐색 (비용 증가, 노이즈 증가 주의)
```

---

## 🛠 Graph RAG 구현 방식 비교

### 주요 라이브러리/도구

| 구현 방식 | 도구 | 그래프 저장소 | 특징 | 난이도 |
|-----------|------|--------------|------|--------|
| **Microsoft GraphRAG** | Python 패키지 | Azure AI Search / 파일 | LLM 기반 자동 추출, 커뮤니티 탐색 | ⭐⭐⭐ |
| **LightRAG** | Python 패키지 | Neo4j / PostgreSQL / 파일 | 경량, 듀얼 레벨 검색 | ⭐⭐⭐ |
| **Neo4j + LangChain** | 전용 그래프 DB | Neo4j | Cypher 쿼리, 강력한 시각화 | ⭐⭐⭐⭐ |
| **Apache AGE** | PostgreSQL 확장 | PostgreSQL | Cypher를 PostgreSQL에서 사용 | ⭐⭐⭐⭐ |
| **PostgreSQL 직접 구현** | pgvector + CTE | PostgreSQL | 추가 인프라 불필요, SQL로 제어 | ⭐⭐ |

### 선택 기준

```
추가 인프라를 설치하기 어렵다
  → PostgreSQL 직접 구현

빠른 프로토타이핑이 필요하다
  → LightRAG (pip install lightrag-hku)

대규모 문서, 전체 요약 질의가 많다
  → Microsoft GraphRAG

강력한 그래프 쿼리/시각화가 필요하다
  → Neo4j + LangChain

PostgreSQL에서 Cypher 문법을 쓰고 싶다
  → Apache AGE
```

---

## 🔬 Microsoft GraphRAG 상세

Microsoft에서 오픈소스로 공개한 Graph RAG 프레임워크입니다.
[GitHub: microsoft/graphrag](https://github.com/microsoft/graphrag)

### 핵심 아이디어

일반 RAG가 개별 청크를 검색하는 것과 달리, GraphRAG는:
1. 문서에서 **엔티티와 관계를 자동 추출** (LLM 활용)
2. 추출된 그래프를 **커뮤니티로 클러스터링**
3. 각 커뮤니티의 **요약(Summary)을 사전 생성**
4. 질의 유형에 따라 **Global / Local / Hybrid 검색** 수행

### 인덱싱 파이프라인

```
[1단계] 문서 분할
  문서 → TextUnit (기본 2400 토큰 단위로 분할)

[2단계] 엔티티·관계 추출 (LLM 호출)
  각 TextUnit → LLM → 엔티티 목록 + 관계 목록 추출
  예: "Python은 Guido가 만들었다"
       → 엔티티: [Python, Guido van Rossum]
       → 관계: (Python) -[CREATED_BY]→ (Guido van Rossum)

[3단계] 그래프 구축
  모든 TextUnit에서 추출된 엔티티·관계를 통합
  동일 엔티티는 병합 (예: "Python"과 "파이썬"이 같은 노드)

[4단계] 커뮤니티 탐지
  Leiden 알고리즘으로 밀접한 노드 그룹 식별

[5단계] 커뮤니티 요약 생성 (LLM 호출)
  각 커뮤니티 → LLM → 커뮤니티 요약 텍스트 생성
  요약도 임베딩하여 저장
```

### 검색 모드

```
Global Search (전체 탐색)
  ├─ 언제: "이 문서 전체에서 가장 중요한 주제는?"
  ├─ 방법: 커뮤니티 요약들을 순위화 → 상위 요약 LLM에 전달
  └─ 특징: 전체적인 질문에 강함, 비용 높음

Local Search (로컬 탐색)
  ├─ 언제: "특정 기능 X는 어떻게 동작하나?"
  ├─ 방법: 관련 엔티티 찾기 → 이웃 엔티티 + 관계 + 청크 포함
  └─ 특징: 특정 개념에 대한 질문에 강함

DRIFT Search (Dynamic Reasoning and Inference with Flexible Traversal)
  ├─ 언제: 복합적인 추론이 필요한 질문
  ├─ 방법: Local + 커뮤니티 컨텍스트 결합
  └─ 특징: 정확도 높음, 가장 느림
```

### 설치 및 기본 사용

```bash
pip install graphrag
```

```python
# 기본 사용 예시
import asyncio
from graphrag.query.context_builder.entity_extraction import EntityVectorStoreKey
from graphrag.query.indexer_adapters import read_indexer_entities, read_indexer_reports
from graphrag.query.llm.oai.chat_openai import ChatOpenAI
from graphrag.query.structured_search.global_search.community_context import GlobalCommunityContext
from graphrag.query.structured_search.global_search.search import GlobalSearch

# 설정
llm = ChatOpenAI(api_key="YOUR_API_KEY", model="gpt-4o")

# Global Search 실행
search_engine = GlobalSearch(
    llm=llm,
    context_builder=GlobalCommunityContext(...),
    token_encoder=tiktoken.get_encoding("cl100k_base"),
    max_data_tokens=12000,
)

result = await search_engine.asearch("시스템의 주요 구성 요소는 무엇인가요?")
print(result.response)
```

### 주의사항

- **비용**: 인덱싱 시 LLM을 대량 호출 → 대용량 문서에서 비용 높음
- **시간**: 커뮤니티 요약 생성까지 오래 걸림
- **언어**: 영어 최적화. 한국어는 프롬프트 커스터마이징 필요

---

## ⚡ LightRAG 상세

홍콩대학교에서 개발한 경량 Graph RAG 프레임워크입니다.
[GitHub: HKUDS/LightRAG](https://github.com/HKUDS/LightRAG)

### 핵심 아이디어

GraphRAG의 무거운 커뮤니티 클러스터링 대신, **듀얼 레벨 검색**으로 빠르고 효과적인 Graph RAG를 구현합니다.

```
Dual-Level Retrieval:
  Low-level  (Local)  : 특정 엔티티와 직접 연결된 관계 탐색
  High-level (Global) : 전체 그래프에서 광범위한 패턴 탐색
```

### 5가지 검색 모드

```python
from lightrag import LightRAG, QueryParam

rag = LightRAG(working_dir="./my_rag")

# naive: 기본 벡터 검색 (Knowledge Graph 미사용)
result = rag.query("질문", param=QueryParam(mode="naive"))

# local: 관련 엔티티 주변 로컬 탐색
result = rag.query("질문", param=QueryParam(mode="local"))

# global: 전체 그래프의 고수준 패턴 탐색
result = rag.query("질문", param=QueryParam(mode="global"))

# hybrid: local + global 결합
result = rag.query("질문", param=QueryParam(mode="hybrid"))

# mix: Knowledge Graph + 벡터 검색 통합 (가장 강력)
result = rag.query("질문", param=QueryParam(mode="mix"))
```

### 설치 및 기본 사용

```bash
pip install lightrag-hku
```

```python
import asyncio
from lightrag import LightRAG, QueryParam
from lightrag.llm.openai import gpt_4o_mini_complete, openai_embed

async def main():
    rag = LightRAG(
        working_dir="./lightrag_data",
        llm_model_func=gpt_4o_mini_complete,
        embedding_func=openai_embed,
    )

    # 문서 인덱싱
    with open("my_document.txt", "r") as f:
        await rag.ainsert(f.read())

    # 검색
    result = await rag.aquery(
        "이 시스템의 핵심 컴포넌트는 무엇인가요?",
        param=QueryParam(mode="hybrid")
    )
    print(result)

asyncio.run(main())
```

### LightRAG 스토리지 백엔드

```python
# PostgreSQL 백엔드 사용
from lightrag.kg.postgres_impl import PostgreSQLStorage

rag = LightRAG(
    working_dir="./lightrag_data",
    graph_storage="PostgreSQLStorage",
    vector_storage="PostgreSQLStorage",
    kv_storage="PostgreSQLStorage",
    # PostgreSQL 연결 설정
    addon_params={
        "host": "localhost",
        "port": 5432,
        "user": "postgres",
        "password": "password",
        "database": "lightrag_db",
    }
)
```

---

## 🐘 PostgreSQL로 Knowledge Graph 구현하기

전용 그래프 DB 없이 **PostgreSQL + pgvector만으로** Knowledge Graph 기반 RAG를 구현하는 방법입니다.

### 전체 스키마 설계

```sql
-- pgvector 확장 활성화
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- 전문 검색용

-- ① 문서 테이블
CREATE TABLE documents (
    id          BIGSERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    source      TEXT,               -- 파일 경로, URL 등
    doc_type    VARCHAR(50),        -- 문서 유형
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- ② 청크 테이블 (벡터 포함)
CREATE TABLE chunks (
    id           BIGSERIAL PRIMARY KEY,
    doc_id       BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index  INT NOT NULL,
    section_id   VARCHAR(100),       -- 섹션 식별자 (예: "3.2.1")
    title        TEXT,               -- 섹션 제목
    content      TEXT NOT NULL,
    content_hash VARCHAR(64),        -- SHA-256: 동일 내용 중복 인덱싱 방지
    embedding    vector(1024),       -- KURE-v1 / bge-m3 계열 기준 1024차원
    chunk_type   VARCHAR(30),        -- 'text', 'table', 'code', 'list'
    token_count  INT,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- content_hash 인덱스 (중복 체크용)
CREATE UNIQUE INDEX ON chunks (content_hash) WHERE content_hash IS NOT NULL;

-- ③ 엔티티 테이블 (Knowledge Graph 노드)
CREATE TABLE kg_entities (
    id           BIGSERIAL PRIMARY KEY,
    name         TEXT NOT NULL,
    entity_type  VARCHAR(50) NOT NULL,
    -- 유형 예: 'concept', 'person', 'product', 'technology',
    --          'process', 'standard', 'term', 'component'
    description  TEXT,
    embedding    vector(1024),       -- 엔티티 자체 임베딩
    metadata     JSONB DEFAULT '{}',
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (name, entity_type)
);

-- ④ 관계 테이블 (Knowledge Graph 엣지)
CREATE TABLE kg_relations (
    id           BIGSERIAL PRIMARY KEY,
    from_entity  BIGINT REFERENCES kg_entities(id) ON DELETE CASCADE,
    to_entity    BIGINT REFERENCES kg_entities(id) ON DELETE CASCADE,
    relation     VARCHAR(100) NOT NULL,
    -- 관계 유형: 'IS_A', 'PART_OF', 'DEPENDS_ON', 'REFERS_TO',
    --            'CREATED_BY', 'REQUIRES', 'CONTRADICTS', 'RELATED_TO'
    description  TEXT,               -- 관계 설명 텍스트
    weight       FLOAT DEFAULT 1.0,  -- 관계 강도/중요도
    confidence   FLOAT DEFAULT 1.0,  -- 추출 신뢰도 (LLM 추출 시)
    source_chunk BIGINT REFERENCES chunks(id),  -- 이 관계가 나온 청크
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- ⑤ 청크-엔티티 매핑 (청크에서 어떤 엔티티가 언급됐는지)
CREATE TABLE kg_chunk_entity (
    chunk_id    BIGINT REFERENCES chunks(id) ON DELETE CASCADE,
    entity_id   BIGINT REFERENCES kg_entities(id) ON DELETE CASCADE,
    mention     TEXT,    -- 실제 텍스트에서 언급된 표현 (동의어 등)
    PRIMARY KEY (chunk_id, entity_id)
);

-- ⑥ 직접 참조 관계 (청크 간 단순 참조 — 빠른 구현용)
CREATE TABLE chunk_relations (
    id            BIGSERIAL PRIMARY KEY,
    from_chunk    BIGINT REFERENCES chunks(id) ON DELETE CASCADE,
    to_chunk      BIGINT REFERENCES chunks(id) ON DELETE CASCADE,
    relation_type VARCHAR(50) NOT NULL,
    -- 'references': "3.2절 참조", "위 정의에 따라"
    -- 'defines'   : 용어 정의
    -- 'extends'   : 내용 확장/부연
    -- 'contradicts': 상충/예외
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- 인덱스
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON kg_entities USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON kg_entities (name);
CREATE INDEX ON kg_entities (entity_type);
CREATE INDEX ON kg_relations (from_entity);
CREATE INDEX ON kg_relations (to_entity);
CREATE INDEX ON kg_relations (relation);
CREATE INDEX ON kg_chunk_entity (chunk_id);
CREATE INDEX ON kg_chunk_entity (entity_id);
CREATE INDEX ON chunk_relations (from_chunk);
CREATE INDEX ON chunk_relations (to_chunk);
-- 전문 검색 인덱스
-- 주의: 한국어 문서라면 'simple' 딕셔너리는 형태소 분석 없이 공백만 분리함
-- 권장: pg_bigm 확장(바이그램 인덱스)을 함께 사용하면 한국어 부분 매칭 정확도 향상
CREATE INDEX ON chunks USING gin(to_tsvector('simple', content));
CREATE EXTENSION IF NOT EXISTS pg_bigm;
CREATE INDEX idx_chunks_bigm ON chunks USING gin (content gin_bigm_ops);
```

### 엔티티·관계 추출 (인제스트 시)

#### 방법 1: LLM 기반 추출 (정확도 높음)

```python
import json
import anthropic

client = anthropic.Anthropic()

KG_EXTRACT_PROMPT = """
다음 텍스트에서 핵심 엔티티(개념, 기술, 컴포넌트 등)와 엔티티 간의 관계를 추출하세요.

텍스트:
{text}

다음 JSON 형식으로 반환하세요. 반드시 JSON만 반환하세요:
{{
  "entities": [
    {{
      "name": "엔티티 이름",
      "type": "concept|technology|process|component|person|standard|term",
      "description": "간단한 설명 (1-2문장)"
    }}
  ],
  "relations": [
    {{
      "from": "출발 엔티티 이름",
      "relation": "IS_A|PART_OF|DEPENDS_ON|REFERS_TO|CREATED_BY|REQUIRES|RELATED_TO",
      "to": "도착 엔티티 이름",
      "description": "관계 설명"
    }}
  ]
}}

주의사항:
- 텍스트에 명시적으로 나타난 관계만 추출하세요
- 엔티티 이름은 정확하고 일관성 있게 작성하세요
- 불명확한 관계는 포함하지 마세요
"""

def extract_kg_from_chunk(chunk_text: str) -> dict:
    """청크 텍스트에서 KG 엔티티와 관계 추출"""
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": KG_EXTRACT_PROMPT.format(text=chunk_text)
        }]
    )

    try:
        return json.loads(response.content[0].text)
    except json.JSONDecodeError:
        # JSON 파싱 실패 시 빈 결과 반환
        return {"entities": [], "relations": []}
```

#### 방법 2: 정규식 기반 추출 (빠름, 패턴이 명확한 경우)

```python
import re
from typing import list

# 문서 내 참조 패턴 정의
REFERENCE_PATTERNS = {
    "section": r'(\d+(?:\.\d+)*)절',              # "3.2절", "5.1.3절"
    "chapter": r'제(\d+)장',                      # "제3장"
    "appendix": r'부록\s*([A-Z\d]+)',             # "부록A", "부록1"
    "figure": r'그림\s*(\d+(?:\.\d+)*)',          # "그림 3.2"
    "table": r'표\s*(\d+(?:\.\d+)*)',             # "표 2.1"
    "definition": r'"([^"]{2,30})"(?:이란|이라 함)', # 정의 패턴
}

def extract_references_by_pattern(content: str) -> list[dict]:
    """정규식으로 참조 패턴 추출"""
    refs = []
    for ref_type, pattern in REFERENCE_PATTERNS.items():
        for match in re.finditer(pattern, content):
            refs.append({
                "type": ref_type,
                "value": match.group(1),
                "full_match": match.group(0),
                "position": match.start()
            })
    return refs

def build_chunk_relations_from_patterns(
    chunks: list[dict],
    section_map: dict[str, int]  # section_id → chunk_id
) -> list[dict]:
    """패턴 기반으로 청크 간 참조 관계 생성"""
    relations = []
    for chunk in chunks:
        refs = extract_references_by_pattern(chunk["content"])
        for ref in refs:
            target_id = section_map.get(ref["value"])
            if target_id and target_id != chunk["id"]:
                relations.append({
                    "from_chunk": chunk["id"],
                    "to_chunk": target_id,
                    "relation_type": "references"
                })
    return relations
```

### Recursive CTE로 그래프 탐색

PostgreSQL의 `WITH RECURSIVE`를 사용해 Knowledge Graph를 n-hop 탐색합니다.

#### 엔티티 중심 탐색

```sql
-- 특정 엔티티에서 시작해 연결된 엔티티를 최대 2-hop 탐색
WITH RECURSIVE entity_traversal AS (
    -- 시작 엔티티 (벡터 검색으로 찾은 관련 청크의 엔티티들)
    SELECT
        kce.entity_id,
        0 AS depth,
        ARRAY[kce.entity_id] AS path  -- 순환 방지용 경로 추적
    FROM kg_chunk_entity kce
    WHERE kce.chunk_id = ANY(:initial_chunk_ids)

    UNION

    -- 관계를 따라 이웃 엔티티 탐색
    SELECT
        CASE
            WHEN r.from_entity = t.entity_id THEN r.to_entity
            ELSE r.from_entity
        END AS entity_id,
        t.depth + 1,
        t.path || CASE
            WHEN r.from_entity = t.entity_id THEN r.to_entity
            ELSE r.from_entity
        END
    FROM entity_traversal t
    JOIN kg_relations r
        ON r.from_entity = t.entity_id
        OR r.to_entity = t.entity_id
    WHERE
        t.depth < :max_hops                         -- 최대 탐색 깊이
        AND NOT (CASE                               -- 순환 방지
            WHEN r.from_entity = t.entity_id THEN r.to_entity
            ELSE r.from_entity
        END = ANY(t.path))
)
-- 탐색된 엔티티가 언급된 청크 반환
SELECT DISTINCT
    c.id,
    c.content,
    c.section_id,
    c.title,
    MIN(t.depth) AS graph_distance  -- 최단 거리
FROM entity_traversal t
JOIN kg_chunk_entity kce ON kce.entity_id = t.entity_id
JOIN chunks c ON c.id = kce.chunk_id
WHERE c.id != ALL(:initial_chunk_ids)  -- 이미 포함된 청크 제외
GROUP BY c.id, c.content, c.section_id, c.title
ORDER BY graph_distance;
```

#### 벡터 검색 + 그래프 탐색 통합 쿼리

```sql
WITH
-- 1단계: 벡터 검색으로 초기 청크 탐색
vector_results AS (
    SELECT
        id,
        content,
        section_id,
        title,
        1 - (embedding <=> :query_embedding::vector) AS similarity,
        0 AS graph_distance,
        'vector' AS source
    FROM chunks
    ORDER BY embedding <=> :query_embedding::vector
    LIMIT :top_k
),
-- 2단계: 초기 청크의 엔티티 확인
initial_entities AS (
    SELECT DISTINCT kce.entity_id
    FROM kg_chunk_entity kce
    JOIN vector_results vr ON vr.id = kce.chunk_id
),
-- 3단계: Knowledge Graph 탐색 (2-hop)
graph_traversal AS (
    SELECT entity_id, 1 AS depth
    FROM initial_entities

    UNION

    SELECT
        CASE WHEN r.from_entity = t.entity_id THEN r.to_entity
             ELSE r.from_entity END,
        t.depth + 1
    FROM graph_traversal t
    JOIN kg_relations r
        ON r.from_entity = t.entity_id OR r.to_entity = t.entity_id
    WHERE t.depth < 2
),
-- 4단계: 그래프로 확장된 청크
graph_results AS (
    SELECT DISTINCT
        c.id,
        c.content,
        c.section_id,
        c.title,
        0.0 AS similarity,
        MIN(gt.depth) AS graph_distance,
        'graph' AS source
    FROM graph_traversal gt
    JOIN kg_chunk_entity kce ON kce.entity_id = gt.entity_id
    JOIN chunks c ON c.id = kce.chunk_id
    WHERE c.id NOT IN (SELECT id FROM vector_results)
    GROUP BY c.id, c.content, c.section_id, c.title
)
-- 최종: 벡터 결과 + 그래프 확장 결과 통합
SELECT * FROM vector_results
UNION ALL
SELECT * FROM graph_results
ORDER BY
    CASE source
        WHEN 'vector' THEN (1 - similarity)     -- 유사도 낮을수록 후순위
        WHEN 'graph'  THEN graph_distance + 1   -- 거리 멀수록 후순위
    END;
```

---

## 🔀 하이브리드 검색 — 벡터 + 키워드 통합

순수 벡터(의미) 검색만으로는 **조항 번호나 전문 용어를 정확히 지정해서 검색하는 경우** 원하는 결과를 찾지 못할 수 있습니다. 하이브리드 검색은 벡터 검색과 키워드 검색을 병렬로 실행한 뒤 점수를 결합합니다.

### 왜 필요한가

| 질의 예시 | 벡터 검색 결과 | 키워드 검색 결과 |
|----------|--------------|----------------|
| "3.2절 인증 프로세스" | 인증 관련 청크들이 유사도 순으로 반환 | "3.2절"이 정확히 포함된 청크 반환 |
| "납입면제 조건" | 의미상 유사한 다른 조항이 먼저 나올 수 있음 | 정확히 "납입면제" 단어가 포함된 청크 반환 |
| 전문 용어 직접 검색 | 학습 데이터 품질에 의존 | 형태소 분석 후 정확 매칭 |

두 채널의 결과를 합치면 **의미 검색의 유연함**과 **키워드 검색의 정확성**을 모두 얻을 수 있습니다.

### 사전 조건: 한국어 tsvector 품질 개선

`to_tsvector('simple', content)`는 공백 기준으로만 토큰을 분리합니다. 한국어는 조사·어미 변화가 심해서 "납입면제가", "납입면제를", "납입면제의"가 모두 다른 토큰으로 저장되어, "납입면제"로 검색하면 어느 것도 매칭되지 않습니다.

**pg_bigm** 확장은 모든 텍스트를 2글자(바이그램) 단위로 분해해서 저장하기 때문에 형태소 분석 없이도 한국어 부분 문자열 매칭이 정확하게 작동합니다.

```sql
-- pg_bigm 설치 (PostgreSQL 슈퍼유저 권한 필요)
CREATE EXTENSION IF NOT EXISTS pg_bigm;

-- 바이그램 인덱스 생성
CREATE INDEX idx_chunks_bigm ON chunks USING gin (content gin_bigm_ops);

-- 검색 시 사용
SELECT id, content FROM chunks
WHERE content LIKE '%납입면제%'   -- pg_bigm이 자동으로 인덱스 활용
ORDER BY similarity(content, '납입면제') DESC
LIMIT 10;
```

### 점수 결합 방식: 정규화 가중 합산

> **중요:** 벡터 검색과 키워드 검색처럼 **성격이 다른 이종 채널**을 결합할 때는 RRF(Reciprocal Rank Fusion)보다 **정규화된 점수의 가중 합산**이 더 적합합니다.
>
> RRF는 순위 정보만 사용해서 점수 크기를 버립니다. 벡터 검색 1–5위가 모두 유사도 0.97–0.98처럼 밀집해 있다면 순위 차이가 실제로는 무의미한데, RRF는 이를 구분할 수 없습니다. RRF는 **동일 모달리티** 내 여러 검색 변형(예: 여러 임베딩 모델 앙상블, 여러 키워드 설정 결합)에 적합합니다.

```
final_score = α × norm(vector_score) + (1-α) × norm(keyword_score)

norm(x) : 해당 검색 결과 내 min-max 정규화 → [0, 1] 범위로 통일
α       : 벡터 검색 가중치 (기본값 0.7, 질의 유형에 따라 동적 조정)
```

#### α 동적 조정

| 질의 유형 | α 권장값 | 이유 |
|-----------|:-------:|------|
| 일반 의미 질의 ("인증 프로세스 설명") | 0.7 | 의미 검색이 더 적합 |
| 섹션 번호 직접 지정 ("3.2절") | 0.3 | 키워드 검색이 더 정확 |
| 짧은 전문 용어 ("납입면제") | 0.4 | 키워드 비중 상향 |

### 구현 코드

```python
import re
import asyncio
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

def _get_alpha(query: str) -> float:
    """질의 유형 감지 → 벡터/키워드 가중치(α) 결정"""
    # 섹션/조항 번호 패턴: "3.2절", "제5조" 등
    if re.search(r'\d+\.\d+절|제\s*\d+\s*[조항]', query):
        return 0.3
    # 단어 수 3개 이하 — 전문 용어 직접 질의로 판단
    if len(query.split()) <= 3:
        return 0.4
    return 0.7  # 기본: 의미 검색 우선

async def keyword_search(
    db: AsyncSession,
    query: str,
    top_k: int = 10,
) -> list[dict]:
    """tsvector 키워드 검색 — 조항 번호·전문 용어 정확 매칭에 강점"""
    sql = """
        SELECT
            id,
            content,
            section_id,
            title,
            ts_rank(to_tsvector('simple', content),
                    plainto_tsquery('simple', :query)) AS keyword_score
        FROM chunks
        WHERE to_tsvector('simple', content) @@ plainto_tsquery('simple', :query)
        ORDER BY keyword_score DESC
        LIMIT :top_k
    """
    result = await db.execute(text(sql), {"query": query, "top_k": top_k})
    return [dict(row._mapping) for row in result.fetchall()]

def score_merge(
    vector_chunks: list[dict],
    keyword_chunks: list[dict],
    alpha: float = 0.7,
    top_n: int = 5,
) -> list[dict]:
    """정규화 점수 가중 합산으로 두 검색 결과 통합"""

    def normalize(chunks: list[dict], key: str) -> dict[str, float]:
        scores = [c[key] for c in chunks if c.get(key) is not None]
        if not scores:
            return {}
        min_s, max_s = min(scores), max(scores)
        denom = max_s - min_s or 1.0
        return {c["id"]: (c[key] - min_s) / denom for c in chunks if c.get(key) is not None}

    v_norm = normalize(vector_chunks, "similarity")
    k_norm = normalize(keyword_chunks, "keyword_score")

    # 두 결과의 합집합
    all_ids = set(v_norm) | set(k_norm)
    chunk_map = {c["id"]: c for c in vector_chunks + keyword_chunks}

    merged = []
    for cid in all_ids:
        score = alpha * v_norm.get(cid, 0.0) + (1 - alpha) * k_norm.get(cid, 0.0)
        chunk = dict(chunk_map[cid])
        chunk["hybrid_score"] = score
        merged.append((score, chunk))

    merged.sort(key=lambda x: x[0], reverse=True)
    return [c for _, c in merged[:top_n]]
```

### 하이브리드 검색 흐름

```
[사용자 질의]
     │
     │  _get_alpha(query) → α 결정
     │
     ├──────────────────────────────────┐
     ▼                                  ▼
[벡터 검색]                         [키워드 검색]
pgvector 코사인 유사도 top-K        tsvector / pg_bigm top-K
코사인 유사도 점수 (0–1)            ts_rank 점수 (0–1, 정규화)
     │                                  │
     └──────────┬───────────────────────┘
                ▼
   [정규화 점수 가중 합산]
   α × norm(vector) + (1-α) × norm(keyword)
                │
                ▼
        [상위 N개 청크 선택]
                │
                ▼
        [Graph RAG 확장]
                │
                ▼
          [LLM 답변 생성]
```

---

## 🔗 Graph RAG + Vector 검색 통합 방안

현재 대부분의 Graph RAG 구현은 벡터 검색과 그래프 확장을 **독립 파이프라인**으로 처리합니다. 두 채널을 더 긴밀하게 연동하면 검색 품질을 한 단계 더 높일 수 있습니다.

### 현재 구조의 문제

```
벡터 검색 → [seed 청크 5개, 신뢰도 0.6–0.99 혼재]
                   │
                   ▼  ← seed 신뢰도와 무관하게 동일하게 처리
Graph 확장 → [참조 청크 3개, 점수 없이 단순 추가]
                   │
                   ▼
         단순 합산 8개 → LLM
```

**문제점:**
- 벡터 유사도 0.62처럼 신뢰도 낮은 seed의 참조 청크도 0.98 seed와 동일한 가중치로 포함됨
- 그래프에서 수십 개 청크가 참조하는 허브 노드(정의 조항 등)가 벡터 검색에서 낮은 순위이면 무시됨
- Graph 확장 결과가 실제 질문과 관련 있는지 검증하지 않음

### 방향 A: 점수 기반 선택적 Graph 확장 (즉시 적용 가능)

벡터 점수가 높은 seed 청크에서만 graph expand를 수행합니다. 신뢰도 낮은 청크의 참조 조항은 노이즈가 될 수 있으므로 제외합니다.

```python
GRAPH_SCORE_THRESHOLD = 0.75  # 코사인 유사도 0.75 이상만 seed로 사용

async def expand_with_graph_selective(
    initial_chunks: list[dict],
    db: AsyncSession,
    max_hops: int = 2,
    expand_limit: int = 5,
    score_threshold: float = GRAPH_SCORE_THRESHOLD,
) -> list[dict]:
    """점수 기반 선택적 Graph 확장"""
    # 신뢰도 높은 seed만 추출
    reliable_seeds = [
        c["id"] for c in initial_chunks
        if c.get("similarity", 0) >= score_threshold
    ]

    # 신뢰도 기준을 만족하는 seed가 없으면 상위 3개로 fallback
    if not reliable_seeds:
        reliable_seeds = [c["id"] for c in initial_chunks[:3]]

    return await expand_with_knowledge_graph(reliable_seeds, db, max_hops, expand_limit)
```

### 방향 B: 그래프 중심성을 검색 점수에 반영 (중기 적용)

허브 노드(많은 청크가 참조하는 정의 조항 등)는 **in-degree(참조 받는 횟수)**가 높습니다. 이런 청크는 벡터 검색 점수가 낮더라도 답변의 맥락 이해에 필수적입니다. in-degree를 벡터 점수에 가산해 허브 노드가 검색 결과 상위에 오도록 합니다.

```sql
-- 사전에 각 청크의 in-degree 계산
SELECT to_chunk AS chunk_id, COUNT(*) AS in_degree
FROM chunk_relations
GROUP BY to_chunk;
```

```python
GRAPH_BOOST = 0.05  # 중심성 보정 계수

def apply_centrality_boost(
    chunks: list[dict],
    in_degree_map: dict[int, int],  # {chunk_id: in_degree}
) -> list[dict]:
    """그래프 중심성(in-degree)을 벡터 점수에 보정 적용"""
    for chunk in chunks:
        in_degree = in_degree_map.get(chunk["id"], 0)
        # in_degree가 10 이상이면 최대 보정치(0.05) 적용
        boost = GRAPH_BOOST * min(in_degree / 10, 1.0)
        chunk["score_boosted"] = min(chunk.get("similarity", 0) + boost, 1.0)
    return sorted(chunks, key=lambda x: x["score_boosted"], reverse=True)
```

### 방향 C: Graph 확장 결과 유사도 재검증 (중기 적용)

Graph expand로 가져온 청크가 실제로 질문과 관련 있는지 질문 벡터와의 유사도를 재계산해서 관련성 낮은 청크를 제외합니다.

```python
GRAPH_RELEVANCE_THRESHOLD = 0.60

async def filter_graph_chunks_by_relevance(
    graph_chunks: list[dict],
    query_embedding: list[float],
    db: AsyncSession,
    threshold: float = GRAPH_RELEVANCE_THRESHOLD,
) -> list[dict]:
    """Graph 확장 결과를 질문 벡터와의 유사도로 재검증"""
    if not graph_chunks:
        return []

    chunk_ids = [c["id"] for c in graph_chunks]

    # DB에서 임베딩 가져와 유사도 재계산
    sql = """
        SELECT id,
               1 - (embedding <=> :query_emb::vector) AS relevance
        FROM chunks
        WHERE id = ANY(:ids)
          AND 1 - (embedding <=> :query_emb::vector) >= :threshold
    """
    result = await db.execute(text(sql), {
        "query_emb": query_embedding,
        "ids": chunk_ids,
        "threshold": threshold,
    })
    relevant_ids = {row.id for row in result.fetchall()}

    return [c for c in graph_chunks if c["id"] in relevant_ids]
```

### 단계별 적용 순서

| 단계 | 방향 | 난이도 | 언제 |
|------|------|--------|------|
| 1단계 | A: 점수 기반 선택적 확장 | 낮음 — 코드 몇 줄 | 즉시 |
| 2단계 | B: 그래프 중심성 보정 | 중간 — in-degree 사전 계산 필요 | A 효과 확인 후 |
| 3단계 | C: 확장 결과 재검증 | 중간 — 추가 DB 조회 발생 | B 효과 확인 후 |

---

## 🐍 Python 통합 구현

### 프로젝트 구조

```
my_kg_rag/
├── app/
│   ├── db/
│   │   ├── connection.py       # DB 연결
│   │   └── models.py           # ORM 모델
│   ├── services/
│   │   ├── parser.py           # 문서 파싱
│   │   ├── chunker.py          # 청킹
│   │   ├── embedder.py         # 임베딩
│   │   ├── kg_extractor.py     # KG 엔티티/관계 추출
│   │   ├── kg_search.py        # KG 기반 검색 확장
│   │   └── rag_service.py      # RAG 오케스트레이터
│   └── api/
│       └── routes.py           # FastAPI 엔드포인트
├── scripts/
│   ├── init_db.py              # DB 초기화
│   └── ingest.py               # 문서 인덱싱 CLI
└── requirements.txt
```

### 인덱싱 파이프라인

```python
# scripts/ingest.py
import asyncio
from pathlib import Path
from app.services.parser import parse_document
from app.services.chunker import chunk_text
from app.services.embedder import embed_chunks
from app.services.kg_extractor import extract_and_save_kg
from app.db.connection import get_db_session

async def ingest_document(file_path: str):
    """문서 인덱싱 전체 파이프라인"""
    async with get_db_session() as db:
        # 1. 문서 파싱
        print(f"[1/5] 파싱: {file_path}")
        text_content = parse_document(file_path)

        # 2. 문서 메타데이터 저장
        doc = await db.execute(
            "INSERT INTO documents (title, source) VALUES (:title, :source) RETURNING id",
            {"title": Path(file_path).stem, "source": file_path}
        )
        doc_id = doc.scalar()

        # 3. 청킹
        print("[2/5] 청킹...")
        chunks = chunk_text(text_content, doc_id=doc_id)

        # 4. 임베딩 + 저장 (content_hash 기반 중복 제거)
        print("[3/5] 임베딩...")
        chunks_with_embeddings = await embed_chunks(chunks)
        chunk_ids = await save_chunks_with_dedup(db, chunks_with_embeddings)

        # 5. Knowledge Graph 추출
        print("[4/5] Knowledge Graph 추출...")
        for chunk, chunk_id in zip(chunks, chunk_ids):
            await extract_and_save_kg(db, chunk["content"], chunk_id)

        # 6. 청크 간 직접 참조 관계 추출
        print("[5/5] 참조 관계 구축...")
        await build_chunk_relations(db, chunks, chunk_ids)

        print(f"완료: {len(chunks)}개 청크, doc_id={doc_id}")
```

### Knowledge Graph 추출 서비스

```python
# app/services/kg_extractor.py
import json
import anthropic
from sqlalchemy.ext.asyncio import AsyncSession

client = anthropic.Anthropic()

KG_EXTRACT_PROMPT = """
다음 텍스트에서 핵심 엔티티와 관계를 추출하세요.

텍스트:
{text}

JSON 형식으로만 반환하세요:
{{
  "entities": [
    {{"name": "이름", "type": "concept|technology|process|component|term", "description": "설명"}}
  ],
  "relations": [
    {{"from": "엔티티명", "relation": "IS_A|PART_OF|DEPENDS_ON|REFERS_TO|REQUIRES|RELATED_TO", "to": "엔티티명", "description": "관계 설명"}}
  ]
}}
"""

async def extract_and_save_kg(
    db: AsyncSession,
    chunk_text: str,
    chunk_id: int
):
    """청크에서 KG 추출 후 DB 저장"""
    # LLM으로 추출
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": KG_EXTRACT_PROMPT.format(text=chunk_text)}]
    )

    try:
        kg_data = json.loads(response.content[0].text)
    except json.JSONDecodeError:
        return  # 파싱 실패 시 건너뜀

    entity_id_map = {}

    # 엔티티 저장 (중복 시 기존 ID 사용)
    for ent in kg_data.get("entities", []):
        result = await db.execute(
            """
            INSERT INTO kg_entities (name, entity_type, description)
            VALUES (:name, :type, :desc)
            ON CONFLICT (name, entity_type) DO UPDATE SET description = EXCLUDED.description
            RETURNING id
            """,
            {"name": ent["name"], "type": ent["type"], "desc": ent.get("description", "")}
        )
        entity_id = result.scalar()
        entity_id_map[ent["name"]] = entity_id

        # 청크-엔티티 매핑
        await db.execute(
            """
            INSERT INTO kg_chunk_entity (chunk_id, entity_id)
            VALUES (:chunk_id, :entity_id)
            ON CONFLICT DO NOTHING
            """,
            {"chunk_id": chunk_id, "entity_id": entity_id}
        )

    # 관계 저장
    for rel in kg_data.get("relations", []):
        from_id = entity_id_map.get(rel["from"])
        to_id = entity_id_map.get(rel["to"])
        if from_id and to_id:
            await db.execute(
                """
                INSERT INTO kg_relations (from_entity, to_entity, relation, description, source_chunk)
                VALUES (:from, :to, :rel, :desc, :chunk)
                ON CONFLICT DO NOTHING
                """,
                {
                    "from": from_id, "to": to_id,
                    "rel": rel["relation"], "desc": rel.get("description", ""),
                    "chunk": chunk_id
                }
            )

    await db.commit()
```

### Knowledge Graph 검색 확장 서비스

```python
# app/services/kg_search.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

KG_EXPAND_SQL = """
WITH RECURSIVE entity_traversal AS (
    SELECT kce.entity_id, 0 AS depth, ARRAY[kce.entity_id] AS path
    FROM kg_chunk_entity kce
    WHERE kce.chunk_id = ANY(:chunk_ids)

    UNION

    SELECT
        CASE WHEN r.from_entity = t.entity_id THEN r.to_entity ELSE r.from_entity END,
        t.depth + 1,
        t.path || CASE WHEN r.from_entity = t.entity_id THEN r.to_entity ELSE r.from_entity END
    FROM entity_traversal t
    JOIN kg_relations r ON r.from_entity = t.entity_id OR r.to_entity = t.entity_id
    WHERE t.depth < :max_hops
      AND NOT (CASE WHEN r.from_entity = t.entity_id THEN r.to_entity ELSE r.from_entity END = ANY(t.path))
)
SELECT DISTINCT c.id, c.content, c.section_id, c.title, MIN(t.depth) AS depth
FROM entity_traversal t
JOIN kg_chunk_entity kce ON kce.entity_id = t.entity_id
JOIN chunks c ON c.id = kce.chunk_id
WHERE c.id != ALL(:chunk_ids)
GROUP BY c.id, c.content, c.section_id, c.title
ORDER BY depth
LIMIT :expand_limit;
"""

async def expand_with_knowledge_graph(
    initial_chunk_ids: list[int],
    db: AsyncSession,
    max_hops: int = 2,
    expand_limit: int = 5
) -> list[dict]:
    """Knowledge Graph로 검색 컨텍스트 확장"""
    if not initial_chunk_ids:
        return []

    result = await db.execute(
        text(KG_EXPAND_SQL),
        {
            "chunk_ids": initial_chunk_ids,
            "max_hops": max_hops,
            "expand_limit": expand_limit
        }
    )
    rows = result.fetchall()
    return [dict(row._mapping) for row in rows]
```

### RAG 오케스트레이터

```python
# app/services/rag_service.py
from app.services.embedder import embed_text
from app.services.kg_search import expand_with_knowledge_graph

async def answer_with_kg_rag(
    question: str,
    db: AsyncSession,
    top_k: int = 5,
    use_kg: bool = True,
    max_hops: int = 2
) -> str:
    """Knowledge Graph RAG로 질문에 답변"""

    # 1. 질문 임베딩
    query_embedding = await embed_text(question)

    # 2. 벡터 검색
    vector_results = await db.execute(
        text("""
            SELECT id, content, section_id, title,
                   1 - (embedding <=> :emb::vector) AS similarity
            FROM chunks
            ORDER BY embedding <=> :emb::vector
            LIMIT :k
        """),
        {"emb": query_embedding, "k": top_k}
    )
    initial_chunks = [dict(r._mapping) for r in vector_results.fetchall()]
    initial_ids = [c["id"] for c in initial_chunks]

    # 3. Knowledge Graph로 컨텍스트 확장
    expanded_chunks = []
    if use_kg:
        expanded_chunks = await expand_with_knowledge_graph(
            initial_ids, db, max_hops=max_hops
        )

    # 4. 전체 컨텍스트 조합
    all_chunks = initial_chunks + expanded_chunks
    context = "\n\n---\n\n".join([
        f"[{c.get('title', '섹션')}]\n{c['content']}"
        for c in all_chunks
    ])

    # 5. LLM으로 답변 생성
    import anthropic
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"""다음 컨텍스트를 참고하여 질문에 답하세요.

컨텍스트:
{context}

질문: {question}

답변:"""
        }]
    )
    return response.content[0].text
```

---

## 🇰🇷 한국어 임베딩 모델 선택 가이드

한국어 문서에 RAG를 구축할 경우 임베딩 모델 선택이 검색 품질에 큰 영향을 미칩니다.

### 벤치마크 비교 (MTEB-ko-retrieval NDCG@10)

| 모델 | NDCG | 차원 | 토큰 제한 | 비용 | 특징 |
|------|------|------|---------|------|------|
| **nlpai-lab/KURE-v1** | **0.655** | 1024 | 8192 | 무료 | bge-m3 기반 한국어 파인튜닝, 한국어 검색 1위 |
| dragonkue/BGE-m3-ko | 0.649 | 1024 | 8192 | 무료 | 한국어 금융/뉴스 코퍼스 파인튜닝 |
| BAAI/bge-m3 | 0.646 | 1024 | 8192 | 무료 | 100개+ 언어 지원, 장문 처리 강점 |
| intfloat/multilingual-e5-large | 0.628 | 1024 | **512** | 무료 | 512 토큰 제한으로 장문 문서 불리 |
| OpenAI text-embedding-3-large | 0.573 | 3072 | 8191 | $0.13/1M | API 호출, GPU 불필요 |

### 선택 기준

```
한국어 정밀도가 최우선이고 장문 문서를 다룬다
  → nlpai-lab/KURE-v1 (추천)
  → 이유: MTEB-ko 1위, 8192 토큰, bge-m3와 동급 속도

한국어 금융/법률 도메인 특화가 필요하다
  → dragonkue/BGE-m3-ko
  → 이유: 금융 코퍼스 파인튜닝, 장문 처리 가능

다국어(영어 포함) 문서를 함께 다룬다
  → BAAI/bge-m3
  → 이유: 100개+ 언어 지원, Dense+Sparse+ColBERT 하이브리드

로컬 모델 서빙 없이 API로 처리하고 싶다
  → OpenAI text-embedding-3-large
  → 이유: GPU 불필요, 전체 코퍼스 임베딩 비용 약 $0.20 수준

※ KoSimCSE, ko-sroberta-multitask, KoE5는 512 토큰 제한으로
  장문 문서 RAG에는 부적합
```

### CPU 환경 임베딩 속도 예측

KURE-v1 / BGE-m3-ko / bge-m3는 모두 동일한 XLM-RoBERTa 24 레이어 백본 구조로
CPU 추론 속도가 사실상 동일합니다.

- chunk당 400-800 토큰 기준: 수천 개 chunk 처리에 20-60분 (1회성 배치)
- 실시간 질의 시 query 임베딩만 수행 → 수십 ms 수준

### 모델 전환 방법

벡터 차원이 동일(1024)한 모델 간 전환은 EmbeddingService 클래스만 교체하면 됩니다.
OpenAI text-embedding-3-large(3072차원)로 전환 시에는 DB `vector(1024)` → `vector(3072)` 수정 필요.

```python
# app/services/embedder.py

from sentence_transformers import SentenceTransformer

# 1순위: 한국어 특화
MODEL_NAME = "nlpai-lab/KURE-v1"

# 차선책: OpenAI API
# from openai import OpenAI
# client = OpenAI()

model = SentenceTransformer(MODEL_NAME)

def embed_text(text: str) -> list[float]:
    return model.encode(text, normalize_embeddings=True).tolist()

def embed_batch(texts: list[str], batch_size: int = 100) -> list[list[float]]:
    results = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        embeddings = model.encode(batch, normalize_embeddings=True)
        results.extend(embeddings.tolist())
    return results
```

---

## 🎯 언제 무엇을 선택할까

### 의사결정 트리

```
문서 간 참조/관계가 복잡한가?
  ├─ No  → Standard RAG로 충분 (simpler is better)
  └─ Yes → Graph RAG 고려
              │
              ├─ 인프라 제약이 있는가?
              │    └─ Yes → PostgreSQL 직접 구현
              │
              ├─ 전체 문서셋 요약/전반적 질의가 많은가?
              │    └─ Yes → Microsoft GraphRAG (Global Search)
              │
              ├─ 빠른 프로토타이핑이 필요한가?
              │    └─ Yes → LightRAG
              │
              └─ 강력한 그래프 쿼리/시각화가 필요한가?
                   └─ Yes → Neo4j + LangChain
```

### 문서 유형별 추천

| 문서 유형 | 추천 방식 | 이유 |
|-----------|-----------|------|
| 기술 문서 / API 레퍼런스 | Knowledge Graph RAG | 컴포넌트 간 의존성, 용어 정의 연결 |
| 법률 / 규정 문서 | Knowledge Graph RAG | 조항 간 상호 참조, 예외 조항 연결 |
| 기업 지식베이스 | LightRAG or GraphRAG | 부서 간 업무 연결, 프로세스 체인 |
| 연구 논문 | GraphRAG | 개념 간 관계, 인용 네트워크 |
| 제품 매뉴얼 | PostgreSQL KG (간단) | 기능 간 참조, 트러블슈팅 연결 |
| FAQ / 블로그 | Standard RAG | 독립적 콘텐츠, 관계 적음 |
| 뉴스 / SNS | Standard RAG | 실시간성 중요, 관계 단순 |

### 단계별 도입 전략

```
Phase 1 (빠른 시작)
  → Standard RAG 구현
  → 답변 품질 베이스라인 측정

Phase 2 (Graph RAG 간단 도입)
  → chunk_relations 테이블 추가
  → 정규식으로 명시적 참조 관계 추출
  → 검색 시 참조 청크 포함
  → 품질 개선 측정

Phase 3 (Knowledge Graph 본격 도입)
  → kg_entities + kg_relations 테이블 추가
  → LLM으로 엔티티·관계 추출
  → Recursive CTE로 다단계 탐색
  → 복합 질의 성능 검증

Phase 4 (고도화)
  → 커뮤니티 클러스터링
  → 커뮤니티 요약 생성
  → 전체 데이터셋 요약 질의 지원
```

---

## 🏗 언어 선택 — Python vs JVM (Java/Kotlin)

> Graph RAG 파이프라인을 구축할 때 자주 받는 질문: "나중에 Java로 바꿔도 되나요?"

### 결론

**ML 파이프라인(임베딩·파싱·RAG 쿼리)은 Python에 두어야 한다. JVM으로 이관하면 실질적 손실이 발생한다.**

### 역할 분담 원칙

| 구성 요소 | 권장 언어 | 이유 |
|-----------|----------|------|
| 문서 파싱 (Docling, pdfplumber 등) | **Python** | Java 생태계에 동급 파서 없음. 표/이미지 구조 인식 불가 |
| 임베딩 생성 (sentence-transformers, KURE-v1 등) | **Python** | PyTorch 기반 — Java에서 직접 실행 불가 |
| KG 엣지 탐색 / 벡터 유사도 검색 | **PostgreSQL** (SQL) | DB가 처리 — 어느 언어에서 호출해도 성능 동일 |
| LLM 호출 (Anthropic, OpenAI API) | 어디서든 가능 | HTTP 호출이라 언어 무관 |
| 인증 / 비즈니스 로직 / 오케스트레이션 | **Java/Spring Boot** | Spring Security, 기업 표준 프레임워크 강점 |

### Java로 이관 시 발생하는 구체적 문제

```
문제 1: 임베딩 모델
  한국어 특화 KURE-v1 (sentence-transformers + PyTorch)
  → Java에서 직접 실행 불가
  → 대안: OpenAI API로 대체 (비용 발생 + 한국어 검색 품질 하락)
  → 대안: JNI/ONNX 변환 (복잡도 급증, 유지보수 어려움)

문제 2: 문서 파싱
  Docling — 표 구조, 레이아웃 인식, OCR 통합 파서
  → Java에 동급 라이브러리 없음
  → 대안: Apache PDFBox, iText (표 구조 인식 품질 현저히 낮음)

문제 3: 불필요한 우회
  Java → Python 서비스 HTTP 호출로 임베딩 처리
  → 결국 Python 서비스가 다시 생기는 구조 (분리 의미 없음)
```

### 핵심: KG 탐색의 실질 연산은 DB가 담당

```
오해: "Graph RAG의 그래프 탐색 로직이 복잡하니 Python이 필요하다"

실제: KG 엣지 탐색(Recursive CTE), pgvector 유사도 검색은
      PostgreSQL이 처리 → Java에서 동일한 SQL을 호출해도 성능 동일

Python이 필요한 진짜 이유:
  임베딩 생성 (query embedding) — PyTorch 기반 모델 로드
  문서 파싱 — Docling, pdfplumber
  이 두 가지가 Python 전용이기 때문
```

### 권장 아키텍처 (마이크로서비스 분리)

```
[클라이언트]
     ↓
[Java/Spring Boot]  — 인증, 비즈니스 로직, 라우팅
     ↓ HTTP
[Python FastAPI]    — 파싱, 임베딩, RAG 파이프라인
     ↓ SQL
[PostgreSQL + pgvector]  — KG 저장, 벡터 검색, 그래프 탐색
```

이 구조에서 각 서비스는 자신이 가장 잘하는 역할에 집중한다.
Java를 도입하더라도 Python FastAPI는 ML 전담 내부 서비스로 유지하는 것이 최선이다.

---

## ✂️ 청킹 전략 — Graph RAG의 필수 요건

Graph RAG에서 청킹 전략은 **벡터 검색 품질**과 **KG 노드 매핑 정확도** 모두에 직접 영향을 미치는 핵심 설계 요소다.

### 청킹이 Graph RAG에서 특히 중요한 이유

Standard RAG에서 청킹은 "검색 단위를 어떻게 나눌 것인가"의 문제다.
Graph RAG에서는 거기서 한 걸음 더 나아가 **청크 = KG 노드** 여야 한다.

```
[질의 벡터] → 유사 청크(KG 노드) 검색
           → 해당 노드의 KG 엣지 탐색 (Recursive CTE)
           → 연관 노드 컨텍스트 수집 → 답변 생성
```

이 흐름이 작동하려면 **각 청크가 완결된 하나의 의미 단위**여야 한다.
청크 중간에 조항이 잘리면, 검색된 청크의 KG 노드를 찾을 수 없거나 엣지 탐색이 끊긴다.

### 청킹 방식별 Graph RAG 적합성 비교

| 청킹 방식 | 방법 | KG 노드 매핑 | Graph RAG 적합성 |
|----------|------|-------------|----------------|
| **헤더/구조 경계 기반** | 문서의 섹션·조항 경계에서 분할 | 1:1 매핑 가능 ✅ | **최적** |
| 고정 크기(Fixed-size) | N자/토큰 단위로 기계적 분할 | 경계 불일치, 매핑 불가 ❌ | 부적합 |
| 문장 단위 | 마침표 기준 분할 | 지나치게 세분화 ❌ | 부적합 |
| 키워드 기반 | 특정 키워드 중심 재구성 | 원본 구조 파괴 ❌ | 부적합 |
| 의미 단위(Semantic) | 임베딩 유사도 기반 병합 | 구조 경계와 불일치 가능 ⚠️ | 문서 유형에 따라 |

### 문서 유형별 권장 청킹 전략

| 문서 유형 | 구조 특성 | 권장 청킹 | 분할 기준 |
|----------|----------|----------|----------|
| 법령·약관·계약서 | 장/조/항/호 명확 | 헤더/조항 경계 | `제N조`, `#` 헤더 |
| 기술 문서·API 레퍼런스 | 섹션 구조 명확 | 헤더 경계 | `##`, `###` 헤더 |
| 논문·보고서 | Abstract/Introduction/방법 구조 | 헤더 + 단락 | `#` 헤더 + 빈 줄 |
| 뉴스·블로그 | 비구조적 자유 서술 | 고정 크기 + 오버랩 | 500–1000자, 오버랩 10–20% |
| 대화·Q&A | 질문-답변 쌍 | 대화 쌍 단위 | Q/A 경계 |

### 구조화 문서를 위한 2단계 청킹 패턴

```python
import re

CHUNK_SIZE = 800      # 목표 청크 크기 (자)
CHUNK_OVERLAP = 100   # 오버랩 크기 (자)

def chunk_structured_document(markdown: str) -> list[str]:
    """
    헤더/구조 경계 기반 2단계 청킹.
    Graph RAG에서 청크 ↔ KG 노드 1:1 매핑을 보장하기 위해 설계.

    1단계: 헤더(#) 경계에서 분할 → 의미 단위 보존
    2단계: 800자 초과 섹션은 슬라이딩 윈도우 추가 분할
    """
    # 1단계: 헤더 경계 분할
    sections = re.split(r"(?=^#{1,4} )", markdown, flags=re.MULTILINE)
    sections = [s.strip() for s in sections if s.strip()]

    chunks = []
    for section in sections:
        if len(section) <= CHUNK_SIZE:
            chunks.append(section)           # 크기 적합 → 그대로 사용
        else:
            # 2단계: 슬라이딩 윈도우 분할
            for i in range(0, len(section), CHUNK_SIZE - CHUNK_OVERLAP):
                piece = section[i:i + CHUNK_SIZE]
                if piece.strip():
                    chunks.append(piece.strip())
    return chunks
```

### 청킹과 KG 노드의 관계

```
Markdown 청크                    KG 노드
─────────────────               ──────────────────
## 제3조(보험금의 지급)    →    node_clause (article_no="제3조")
① 회사는 피보험자가...              ↕ has_clause 엣지
② 다음 각 호의 경우...              ↕
                                node_section (주계약/특약)
                                    ↕ belongs_to 엣지
                                node_product (상품)
```

헤더 기반 청킹은 이 매핑을 자동으로 성립시킨다.
고정 크기 청킹은 `제3조`가 두 청크에 걸쳐 잘리므로 어느 청크도 `node_clause`와 매핑할 수 없다.

### 오버랩(Overlap) 설계 원칙

| 청킹 유형 | 오버랩 필요성 | 권장 설정 |
|----------|-------------|----------|
| 헤더/구조 경계 기반 1차 분할 | 불필요 (경계가 명확) | 오버랩 0 |
| 슬라이딩 윈도우 2차 분할 | 필요 (절단 보완) | 전체 크기의 10–15% |
| 고정 크기 분할 | 필요 | 전체 크기의 15–20% |

> **핵심 원칙**: 헤더/경계 기반으로 1차 분할한 청크에는 오버랩이 필요 없다.
> 오버랩은 "의미 단위를 모르고 기계적으로 자를 때" 경계 손실을 보완하는 장치다.
> Graph RAG에서는 오버랩으로 만든 중복 청크가 KG 노드를 오히려 중복 생성할 수 있어 주의해야 한다.

---

## 📄 PDF 파서 선택 가이드

Knowledge Graph RAG 시스템에서 문서 인제스트의 첫 단계는 PDF 파싱입니다. 파서 품질이 KG 추출 품질에 직접 영향을 줍니다.

### 파서 비교표

| 파서 | 텍스트 추출 | 표 구조 | 수식/이미지 | OCR | 속도(CPU) | 설치 |
|------|------------|---------|------------|-----|-----------|------|
| **Docling** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ | 중간 | 간단 |
| **pdfplumber** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ❌ | ❌ | 빠름 | 간단 |
| **PyMuPDF (fitz)** | ⭐⭐⭐⭐ | ⭐⭐ | ❌ | ❌ | 매우 빠름 | 간단 |
| **Marker** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ | 매우 느림 | 복잡 |

### 추천 선택

```
텍스트 레이어가 있는 PDF (일반 문서)
  → Docling (OCR 비활성화) 또는 pdfplumber
  → 이유: 빠르고 구조 보존 우수

스캔된 PDF (이미지 기반)
  → Docling (OCR 활성화)
  → 이유: OCR + 구조 인식 결합

표/그래프가 많은 기술 문서
  → Docling
  → 이유: 표 구조를 Markdown으로 변환

빠른 처리가 최우선
  → PyMuPDF (fitz)
  → 이유: 가장 빠른 속도
```

### Docling 기본 사용

```python
from docling.document_converter import DocumentConverter
from docling.datamodel.pipeline_options import PdfPipelineOptions
import fitz  # PyMuPDF

def has_text_layer(pdf_path: str, sample_pages: int = 3) -> bool:
    """텍스트 레이어 유무 자동 판별 — OCR 필요 여부 결정"""
    doc = fitz.open(pdf_path)
    for page in doc[:sample_pages]:
        if len(page.get_text().strip()) > 50:
            return True
    return False

def parse_pdf_to_markdown(pdf_path: str) -> str:
    """
    텍스트 레이어 유무를 자동 감지하여 OCR 활성/비활성화 결정.
    텍스트 레이어가 있는 PDF는 OCR 비활성화로 속도 향상.
    스캔 PDF는 자동으로 OCR 활성화 전환.
    """
    use_ocr = not has_text_layer(pdf_path)
    pipeline_options = PdfPipelineOptions()
    pipeline_options.do_ocr = use_ocr

    converter = DocumentConverter(pipeline_options=pipeline_options)
    result = converter.convert(pdf_path)
    return result.document.export_to_markdown()
```

> **실측 성능 (CPU 환경 기준)**
> - 텍스트 레이어 있는 일반 문서 (100-200페이지): 2-8분/파일
> - 병목 구간: DocLayNet 레이아웃 분석 (OCR 활성/비활성 관계없이 동일)
> - 이미지 기반 표·도해는 `<!-- image -->` 로 처리되어 내용 손실 발생

### Docker 환경에서 Docling 모델 캐싱

Docling은 처음 실행 시 HuggingFace에서 모델 파일(약 300MB)을 다운로드합니다.
컨테이너가 재시작될 때마다 재다운로드되는 것을 방지하려면 Docker named volume을 사용하세요.

```bash
# named volume 생성 (최초 1회)
docker volume create hf_model_cache

# 컨테이너 실행 시 모델 캐시 볼륨 마운트
docker run --rm \
  -v hf_model_cache:/root/.cache/huggingface \
  -v $(pwd)/pdfs:/pdfs \
  my-docling-image \
  python ingest.py
```

> **주의**: HuggingFace는 XetHub 방식으로 대용량 파일을 배포하며, pre-signed S3 URL이 수 시간 후 만료됩니다.
> 다운로드 도중 컨테이너가 중단되면 `.incomplete` 파일이 남고 재시작 시 이어받기를 시도합니다.
> named volume을 사용하면 한 번 다운로드 후 영구 보존됩니다.

---

## 🔧 트러블슈팅

### pgvector 관련

```sql
-- pgvector 확장이 없을 때
CREATE EXTENSION IF NOT EXISTS vector;

-- 차원 불일치 오류
-- 임베딩 모델 변경 시 기존 데이터와 차원이 다를 수 있음
-- 해결: 새 컬럼 추가 후 마이그레이션
ALTER TABLE chunks ADD COLUMN embedding_new vector(768);
-- 기존 데이터 변환 후 컬럼 교체
```

### Recursive CTE 무한 루프 방지

```sql
-- 반드시 순환 방지 조건 추가
WHERE t.depth < :max_hops                    -- 최대 깊이 제한
  AND NOT (next_entity = ANY(t.path))        -- 방문한 노드 재방문 금지
```

### HuggingFace 모델 다운로드 중단 (Docker 환경)

```
증상: Docling/임베딩 모델 다운로드 중 컨테이너가 수 시간째 멈춤
원인: HuggingFace XetHub pre-signed S3 URL 만료 (수 시간 후 403 Forbidden)
```

```bash
# 해결: Docker named volume으로 모델 영구 캐싱
docker volume create hf_model_cache

docker run --rm \
  -v hf_model_cache:/root/.cache/huggingface \
  my-image python my_script.py

# 이미 다운로드된 파일을 volume으로 복사 (기존 컨테이너에서 복구 시)
docker cp <container_id>:/root/.cache/huggingface/hub ./hf_backup

docker run --rm \
  -v hf_model_cache:/root/.cache/huggingface \
  -v ./hf_backup:/hf_backup \
  alpine cp -r /hf_backup/hub /root/.cache/huggingface/
```

> `.incomplete` 파일이 남아 있으면 삭제 후 재시작 시 처음부터 재다운로드합니다.

### Docling libxcb 오류 (Linux/Docker)

```bash
# 오류: libxcb.so.1: cannot open shared object file
apt-get install -y libxcb1 libgl1 libglib2.0-0
```

### Docling OCR 비활성화로 속도 개선

```python
# 텍스트 레이어가 있는 PDF에서 OCR 비활성화
# → 처리 속도 대폭 향상 (OCR 활성화 대비 10-20배 빠름)
pipeline_options.do_ocr = False
```

### LLM 기반 KG 추출 비용 절감

```python
# 모든 청크에 LLM 추출 적용하면 비용 큼
# → 중요 섹션만 선택적으로 적용

def should_extract_kg(chunk: dict) -> bool:
    """KG 추출 대상 청크 필터링"""
    # 너무 짧은 청크 제외
    if len(chunk["content"]) < 200:
        return False
    # 목차, 페이지 번호 등 제외
    if chunk.get("chunk_type") in ["header", "footer", "toc"]:
        return False
    return True
```

### PostgreSQL Recursive CTE 성능 최적화

```sql
-- 탐색 전 인덱스 확인
EXPLAIN ANALYZE WITH RECURSIVE ...

-- 필요한 인덱스
CREATE INDEX ON kg_relations (from_entity, to_entity);
CREATE INDEX ON kg_chunk_entity (entity_id, chunk_id);

-- max_hops를 너무 크게 설정하지 말 것 (3 이상은 성능 급락)
-- 대부분의 경우 max_hops=2면 충분
```

---

## 📚 참고 자료

### 논문

- [RAG 원본 논문 (Lewis et al., 2020)](https://arxiv.org/abs/2005.11401)
- [From Local to Global: A Graph RAG Approach (Microsoft, 2024)](https://arxiv.org/abs/2404.16130)
- [LightRAG: Simple and Fast Retrieval-Augmented Generation (HKUDS, 2024)](https://arxiv.org/abs/2410.05779)

### 공식 문서

- [Microsoft GraphRAG GitHub](https://github.com/microsoft/graphrag)
- [LightRAG GitHub](https://github.com/HKUDS/LightRAG)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Docling GitHub](https://github.com/DS4SD/docling)
- [PostgreSQL Recursive CTE 공식 문서](https://www.postgresql.org/docs/current/queries-with.html)

### 관련 가이드

- [PostgreSQL 공식 웹사이트](https://www.postgresql.org/)
- [Supabase AI & Vector 가이드](https://supabase.com/docs/guides/ai)
- [MongoDB Atlas Vector Search](https://www.mongodb.com/docs/atlas/atlas-vector-search)
