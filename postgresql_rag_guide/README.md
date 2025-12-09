# PostgreSQL과 pgvector를 활용한 RAG 시스템 구축 완전 가이드

> 한국어 개발자를 위한 실전 RAG(Retrieval-Augmented Generation) 시스템 구축 튜토리얼

[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-blue.svg)](https://www.postgresql.org/)
[![pgvector](https://img.shields.io/badge/pgvector-0.5.1-green.svg)](https://github.com/pgvector/pgvector)
[![Python](https://img.shields.io/badge/Python-3.8+-yellow.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-red.svg)](LICENSE)

## 📋 목차

- [프로젝트 소개](#-프로젝트-소개)
- [왜 PostgreSQL인가?](#-왜-postgresql인가)
- [시스템 아키텍처](#-시스템-아키텍처)
- [환경 구축](#-환경-구축)
- [빠른 시작](#-빠른-시작)
- [상세 가이드](#-상세-가이드)
- [성능 최적화](#-성능-최적화)
- [트러블슈팅](#-트러블슈팅)
- [실전 예제](#-실전-예제)
- [FAQ](#-faq)
- [참고 자료](#-참고-자료)

---

## 🎯 프로젝트 소개

이 프로젝트는 **PostgreSQL + pgvector + Ollama(Llama3)**를 사용하여 완전한 RAG(Retrieval-Augmented Generation) 시스템을 구축하는 실전 가이드입니다.

### 주요 특징

- ✅ **완전 오픈소스**: 모든 구성 요소가 무료
- ✅ **실전 중심**: 바로 실행 가능한 코드 제공
- ✅ **한국어 최적화**: 한국어 문서 처리 및 검색
- ✅ **프로덕션 준비**: 성능 최적화 및 배포 가이드
- ✅ **확장 가능**: 대규모 데이터셋 지원

### 학습 목표

이 가이드를 완료하면 다음을 할 수 있습니다:

1. PostgreSQL과 pgvector 설치 및 설정
2. 문서를 벡터로 변환하여 데이터베이스에 저장
3. 의미 기반 유사도 검색 구현
4. LLM과 연동하여 RAG 시스템 구축
5. 하이브리드 검색(벡터 + 전문검색) 구현
6. 성능 최적화 및 프로덕션 배포

---

## 🤔 왜 PostgreSQL인가?

### PostgreSQL의 강점

PostgreSQL은 **전 세계적으로 가장 성공한 Enterprise Level Open Source RDBMS**입니다.

| 기능 | PostgreSQL | 전용 벡터 DB |
|------|------------|--------------|
| **벡터 검색** | ✅ pgvector 확장 | ✅ 네이티브 지원 |
| **SQL 지원** | ✅ ANSI SQL 90%+ | ❌ 제한적 |
| **트랜잭션** | ✅ ACID 보장 | ⚠️ 제한적 |
| **확장성** | ✅ 다양한 확장 | ❌ 제한적 |
| **비용** | ✅ 완전 무료 | 💰 유료 또는 제한 |
| **생태계** | ✅ 성숙한 도구 | ⚠️ 발전 중 |

### RAG에 최적인 이유

1. **하이브리드 검색**: 벡터 검색 + 전문 검색(Full-text Search) 결합
2. **데이터 무결성**: ACID 특성으로 안정적 데이터 관리
3. **확장성**: 수백만 개 문서 처리 가능
4. **운영 편의성**: 기존 PostgreSQL 운영 경험 활용
5. **비용 효율성**: 별도 벡터 DB 불필요

---

## 🏗 시스템 아키텍처

### RAG 워크플로우

```
사용자 질문
    ↓
임베딩 생성 (Ollama Llama3, 4096차원)
    ↓
PostgreSQL + pgvector 벡터 검색
    ↓
Top-K 문서 검색 (기본 3개)
    ↓
컨텍스트 구성 (프롬프트 생성)
    ↓
LLM 응답 생성 (Ollama Llama3)
    ↓
최종 답변
```

### 시스템 구성도

```
Application Layer
    FastAPI (REST) | Streamlit (Web UI) | CLI Interface | Custom App
                            ↓
RAG Engine Layer
    RAG Orchestrator
        - Query Understanding & Processing
        - Context Assembly & Management
        - Response Generation & Streaming
        - Chat History Management
            ↓                           ↓
    Document Retriever          LLM Generator
        - Vector Search             - Prompt Engineering
        - Hybrid Search             - Streaming Response
        - Metadata Filter           - Context Injection
        - Re-ranking                - Token Management
                            ↓
Data Layer (PostgreSQL)
    documents → document_chunks → metadata (JSONB)
        - id, title, content       - category, tags[], author
        - source, embedding        - version, language
        - created_at               - custom_fields
                ↓
    pgvector Extension
        - Vector Operations (<->, <#>, <=>)
        - IVFFlat Index (Fast Approximate Search)
        - HNSW Index (High Accuracy Search)
        - Distance Metrics (Cosine, L2, Inner Product)
                ↓
External Services
    Ollama Server              Document Store
        - Llama3                   - File System
        - Embeddings               - S3/MinIO
        - Generation               - Google Drive
```

### 핵심 컴포넌트

1. **EmbeddingManager**: 텍스트 → 벡터 변환
2. **DocumentRetriever**: 유사도 기반 문서 검색
3. **RAGEngine**: 검색 + 생성 통합
4. **DocumentLoader**: 다양한 형식 문서 로드

---

## 🚀 환경 구축

### 시스템 요구사항

- **OS**: Ubuntu 20.04+, macOS 11+, Windows 10+ (WSL2)
- **Python**: 3.8 이상
- **PostgreSQL**: 12 이상 (15 권장)
- **RAM**: 최소 8GB (16GB 권장)
- **디스크**: 최소 10GB 여유 공간

### 1. PostgreSQL 설치

#### Ubuntu/Debian

```bash
# PostgreSQL 공식 저장소 추가
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# GPG 키 추가
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# 설치
sudo apt-get update
sudo apt-get install -y postgresql-15 postgresql-contrib-15

# 서비스 시작
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

#### macOS

```bash
# Homebrew 사용
brew install postgresql@15

# 서비스 시작
brew services start postgresql@15
```

#### Windows

1. https://www.postgresql.org/download/windows/ 에서 설치 프로그램 다운로드
2. 설치 마법사 실행
3. 포트: 5432 (기본값)
4. 비밀번호 설정 (기억할 것!)

#### Docker (모든 OS)

```bash
docker run -d \
  --name postgres-rag \
  -e POSTGRES_PASSWORD=your_password \
  -e POSTGRES_DB=rag_db \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15
```

### 2. pgvector 설치

```bash
# 의존성 설치 (Ubuntu/Debian)
sudo apt-get install -y postgresql-server-dev-15 build-essential git

# pgvector 다운로드 및 설치
cd /tmp
git clone --branch v0.5.1 https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install

# 설치 확인
sudo -u postgres psql -c "SELECT * FROM pg_available_extensions WHERE name = 'vector';"
```

**macOS:**
```bash
brew install pgvector
```

**Docker:** pgvector가 포함된 이미지 사용
```bash
docker run -d \
  --name postgres-rag \
  -e POSTGRES_PASSWORD=your_password \
  -p 5432:5432 \
  ankane/pgvector
```

### 3. Ollama 설치

#### Linux

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Llama3 모델 다운로드
ollama pull llama3
```

#### macOS

```bash
# Homebrew 사용
brew install ollama

# 또는 공식 사이트에서 다운로드
# https://ollama.com/download
```

#### Windows

1. https://ollama.com/download 에서 설치 프로그램 다운로드
2. 설치 후 명령 프롬프트에서:
```cmd
ollama pull llama3
```

### 4. Python 환경 설정

```bash
# 가상 환경 생성
python -m venv rag-env

# 활성화
source rag-env/bin/activate  # Linux/macOS
# rag-env\Scripts\activate   # Windows

# 패키지 설치
pip install --upgrade pip
pip install psycopg2-binary pgvector requests numpy python-dotenv
```

---

## ⚡ 빠른 시작

### 1. 프로젝트 클론

```bash
cd C:\git_clone
git clone <repository-url> postgresql_rag_guide
cd postgresql_rag_guide
```

### 2. 데이터베이스 초기화

```bash
# PostgreSQL 접속
psql -U postgres

# 데이터베이스 생성
CREATE DATABASE rag_db ENCODING 'UTF8';
\c rag_db

# pgvector 확장 설치
CREATE EXTENSION vector;

# 스키마 생성
\i schema.sql

# 종료
\q
```

### 3. 환경 변수 설정

`.env` 파일 생성:

```env
# PostgreSQL 설정
DB_HOST=localhost
DB_PORT=5432
DB_NAME=rag_db
DB_USER=postgres
DB_PASSWORD=your_password

# Ollama 설정
OLLAMA_URL=http://localhost:11434
EMBEDDING_MODEL=llama3
GENERATION_MODEL=llama3

# RAG 설정
EMBEDDING_DIMENSION=4096
TOP_K_RESULTS=3
MAX_CONTEXT_LENGTH=4000
```

### 4. 샘플 실행

```python
from embedding_manager import EmbeddingManager
from rag_engine import RAGEngine

# 문서 추가
emb_mgr = EmbeddingManager()
doc_id = emb_mgr.insert_document(
    title="PostgreSQL 소개",
    content="PostgreSQL은 강력한 오픈소스 관계형 데이터베이스입니다.",
    source="sample.txt"
)
print(f"✓ 문서 추가 완료 (ID: {doc_id})")

# RAG 질의
rag = RAGEngine()
result = rag.generate_response("PostgreSQL의 특징은?")
print(f"\n질문: {result['query']}")
print(f"답변: {result['answer']}")

# 정리
emb_mgr.close()
rag.close()
```

---

## 📚 상세 가이드

### 데이터베이스 스키마
#### 기본 스키마

```sql
-- 문서 테이블
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(500),
    content TEXT NOT NULL,
    source VARCHAR(255),
    metadata JSONB,
    embedding vector(4096),  -- Llama3 임베딩 차원
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 업데이트 트리거
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_documents_updated_at 
    BEFORE UPDATE ON documents
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- 인덱스 생성
CREATE INDEX idx_documents_source ON documents(source);
CREATE INDEX idx_documents_created_at ON documents(created_at);
CREATE INDEX idx_documents_metadata ON documents USING GIN(metadata);

-- 벡터 인덱스 (IVFFlat - 빠른 근사 검색)
CREATE INDEX documents_embedding_idx ON documents 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 전문 검색 인덱스
CREATE INDEX idx_documents_content_fts ON documents 
USING GIN(to_tsvector('english', content));
```

#### 문서 분할 스키마 (긴 문서용)

```sql
-- 청크 테이블
CREATE TABLE document_chunks (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    embedding vector(4096),
    token_count INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(document_id, chunk_index)
);

-- 인덱스
CREATE INDEX idx_chunks_document_id ON document_chunks(document_id);
CREATE INDEX chunks_embedding_idx ON document_chunks 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### Python 구현

#### 1. 설정 파일 (config.py)

```python
import os
from dotenv import load_dotenv

load_dotenv()

# PostgreSQL 설정
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': os.getenv('DB_PORT', '5432'),
    'database': os.getenv('DB_NAME', 'rag_db'),
    'user': os.getenv('DB_USER', 'postgres'),
    'password': os.getenv('DB_PASSWORD')
}

# Ollama 설정
OLLAMA_BASE_URL = os.getenv('OLLAMA_URL', 'http://localhost:11434')
EMBEDDING_MODEL = os.getenv('EMBEDDING_MODEL', 'llama3')
GENERATION_MODEL = os.getenv('GENERATION_MODEL', 'llama3')

# RAG 설정
EMBEDDING_DIMENSION = int(os.getenv('EMBEDDING_DIMENSION', '4096'))
TOP_K_RESULTS = int(os.getenv('TOP_K_RESULTS', '3'))
MAX_CONTEXT_LENGTH = int(os.getenv('MAX_CONTEXT_LENGTH', '4000'))
```

#### 2. 임베딩 매니저 (embedding_manager.py)

```python
import requests
import psycopg2
from pgvector.psycopg2 import register_vector
from typing import List, Dict, Optional
import config

class EmbeddingManager:
    """문서 임베딩 생성 및 저장"""
    
    def __init__(self):
        self.conn = psycopg2.connect(**config.DB_CONFIG)
        register_vector(self.conn)
        self.cursor = self.conn.cursor()
    
    def generate_embedding(self, text: str) -> List[float]:
        """Ollama를 통해 텍스트 임베딩 생성"""
        try:
            response = requests.post(
                f"{config.OLLAMA_BASE_URL}/api/embeddings",
                json={
                    "model": config.EMBEDDING_MODEL,
                    "prompt": text
                },
                timeout=30
            )
            response.raise_for_status()
            return response.json()['embedding']
        except Exception as e:
            print(f"❌ 임베딩 생성 실패: {e}")
            raise
    
    def insert_document(
        self, 
        title: str, 
        content: str, 
        source: str,
        metadata: Optional[Dict] = None
    ) -> int:
        """문서와 임베딩을 데이터베이스에 저장"""
        try:
            # 임베딩 생성
            embedding = self.generate_embedding(content)
            
            # 데이터베이스에 저장
            query = """
                INSERT INTO documents (title, content, source, metadata, embedding)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING id
            """
            
            self.cursor.execute(
                query,
                (title, content, source, 
                 psycopg2.extras.Json(metadata or {}),
                 embedding)
            )
            
            doc_id = self.cursor.fetchone()[0]
            self.conn.commit()
            return doc_id
            
        except Exception as e:
            self.conn.rollback()
            print(f"❌ 문서 삽입 실패: {e}")
            raise
    
    def bulk_insert_documents(self, documents: List[Dict]) -> List[int]:
        """여러 문서 일괄 삽입"""
        doc_ids = []
        
        for i, doc in enumerate(documents, 1):
            try:
                doc_id = self.insert_document(
                    title=doc['title'],
                    content=doc['content'],
                    source=doc['source'],
                    metadata=doc.get('metadata')
                )
                doc_ids.append(doc_id)
                print(f"✓ [{i}/{len(documents)}] {doc['title']} (ID: {doc_id})")
            except Exception as e:
                print(f"✗ [{i}/{len(documents)}] {doc['title']} 실패: {e}")
        
        return doc_ids
    
    def close(self):
        """연결 종료"""
        self.cursor.close()
        self.conn.close()
```

#### 3. 문서 검색기 (retriever.py)

```python
import psycopg2
from pgvector.psycopg2 import register_vector
from typing import List, Dict
import config

class DocumentRetriever:
    """벡터 유사도 기반 문서 검색"""
    
    def __init__(self):
        self.conn = psycopg2.connect(**config.DB_CONFIG)
        register_vector(self.conn)
        self.cursor = self.conn.cursor()
    
    def search_similar_documents(
        self,
        query_embedding: List[float],
        top_k: int = config.TOP_K_RESULTS,
        metadata_filter: Dict = None,
        distance_metric: str = 'cosine'
    ) -> List[Dict]:
        """벡터 유사도 검색"""
        
        # 거리 측정 연산자
        operators = {
            'cosine': '<->',      # 코사인 거리
            'l2': '<->',          # L2 거리
            'inner_product': '<#>' # 내적
        }
        operator = operators.get(distance_metric, '<->')
        
        # 쿼리 구성
        query = f"""
            SELECT 
                id,
                title,
                content,
                source,
                metadata,
                embedding {operator} %s AS distance
            FROM documents
        """
        
        params = [query_embedding]
        
        # 메타데이터 필터
        if metadata_filter:
            conditions = []
            for key, value in metadata_filter.items():
                conditions.append(f"metadata->>%s = %s")
                params.extend([key, value])
            query += " WHERE " + " AND ".join(conditions)
        
        query += f" ORDER BY distance LIMIT %s"
        params.append(top_k)
        
        self.cursor.execute(query, params)
        
        results = []
        for row in self.cursor.fetchall():
            results.append({
                'id': row[0],
                'title': row[1],
                'content': row[2],
                'source': row[3],
                'metadata': row[4],
                'distance': float(row[5]),
                'similarity': 1 - float(row[5])  # 유사도 (0~1)
            })
        
        return results
    
    def hybrid_search(
        self,
        query_text: str,
        query_embedding: List[float],
        top_k: int = config.TOP_K_RESULTS,
        vector_weight: float = 0.7,
        text_weight: float = 0.3
    ) -> List[Dict]:
        """하이브리드 검색 (벡터 + 전문 검색)"""
        
        query = """
            WITH vector_search AS (
                SELECT 
                    id,
                    title,
                    content,
                    source,
                    metadata,
                    embedding <-> %s AS vector_distance
                FROM documents
                ORDER BY vector_distance
                LIMIT 20
            ),
            text_search AS (
                SELECT 
                    id,
                    ts_rank(to_tsvector('english', content), 
                            plainto_tsquery('english', %s)) AS text_score
                FROM documents
                WHERE to_tsvector('english', content) @@ 
                      plainto_tsquery('english', %s)
            )
            SELECT 
                v.id,
                v.title,
                v.content,
                v.source,
                v.metadata,
                v.vector_distance,
                ((%s * (1 - v.vector_distance)) + 
                 (%s * COALESCE(t.text_score, 0))) AS combined_score
            FROM vector_search v
            LEFT JOIN text_search t ON v.id = t.id
            ORDER BY combined_score DESC
            LIMIT %s
        """
        
        self.cursor.execute(
            query,
            (query_embedding, query_text, query_text, 
             vector_weight, text_weight, top_k)
        )
        
        results = []
        for row in self.cursor.fetchall():
            results.append({
                'id': row[0],
                'title': row[1],
                'content': row[2],
                'source': row[3],
                'metadata': row[4],
                'distance': float(row[5]),
                'score': float(row[6])
            })
        
        return results
    
    def close(self):
        """연결 종료"""
        self.cursor.close()
        self.conn.close()
```

#### 4. RAG 엔진 (rag_engine.py)

```python
import requests
from typing import List, Dict
from embedding_manager import EmbeddingManager
from retriever import DocumentRetriever
import config

class RAGEngine:
    """RAG 파이프라인 통합"""
    
    def __init__(self):
        self.embedding_mgr = EmbeddingManager()
        self.retriever = DocumentRetriever()
    
    def generate_response(
        self,
        query: str,
        top_k: int = config.TOP_K_RESULTS,
        use_hybrid: bool = False,
        metadata_filter: Dict = None,
        stream: bool = False
    ) -> Dict:
        """RAG 전체 파이프라인 실행"""
        
        print(f"🔍 질문 분석 중: {query}")
        
        # 1. 쿼리 임베딩 생성
        query_embedding = self.embedding_mgr.generate_embedding(query)
        print("✓ 임베딩 생성 완료")
        
        # 2. 관련 문서 검색
        if use_hybrid:
            documents = self.retriever.hybrid_search(
                query_text=query,
                query_embedding=query_embedding,
                top_k=top_k
            )
            print(f"✓ 하이브리드 검색 완료 ({len(documents)}개 문서)")
        else:
            documents = self.retriever.search_similar_documents(
                query_embedding=query_embedding,
                top_k=top_k,
                metadata_filter=metadata_filter
            )
            print(f"✓ 벡터 검색 완료 ({len(documents)}개 문서)")
        
        # 3. 컨텍스트 구성
        context = self._build_context(documents)
        
        # 4. 프롬프트 생성
        prompt = self._build_prompt(query, context)
        
        # 5. LLM 응답 생성
        print("💬 응답 생성 중...")
        llm_response = self._generate_llm_response(prompt, stream=stream)
        print("✓ 응답 생성 완료")
        
        return {
            'query': query,
            'answer': llm_response,
            'sources': documents,
            'context': context
        }
    
    def _build_context(self, documents: List[Dict]) -> str:
        """검색된 문서로 컨텍스트 구성"""
        if not documents:
            return "관련 문서를 찾을 수 없습니다."
        
        context_parts = []
        for i, doc in enumerate(documents, 1):
            similarity = doc.get('similarity', 1 - doc.get('distance', 0))
            context_parts.append(
                f"[문서 {i}] {doc['title']} (유사도: {similarity:.2%})\n"
                f"출처: {doc['source']}\n"
                f"{doc['content'][:500]}{'...' if len(doc['content']) > 500 else ''}\n"
            )
        
        return "\n" + "="*80 + "\n".join(context_parts)
    
    def _build_prompt(self, query: str, context: str) -> str:
        """RAG 프롬프트 생성"""
        return f"""아래 참고 문서들을 바탕으로 질문에 답변해주세요.

참고 문서:
{context}

질문: {query}

답변 지침:
1. 참고 문서의 정보를 기반으로 정확하게 답변하세요
2. 문서에 없는 내용은 추측하지 마세요
3. 가능하면 어떤 문서를 참고했는지 언급하세요
4. 간결하고 명확하게 답변하세요
5. 한국어로 답변하세요

답변:"""
    
    def _generate_llm_response(self, prompt: str, stream: bool = False) -> str:
        """Llama3로 응답 생성"""
        try:
            response = requests.post(
                f"{config.OLLAMA_BASE_URL}/api/generate",
                json={
                    "model": config.GENERATION_MODEL,
                    "prompt": prompt,
                    "stream": stream
                },
                timeout=60
            )
            response.raise_for_status()
            
            if stream:
                # 스트리밍 처리 (향후 구현)
                pass
            else:
                return response.json()['response']
                
        except Exception as e:
            return f"응답 생성 중 오류 발생: {e}"
    
    def conversational_query(
        self,
        query: str,
        chat_history: List[Dict],
        top_k: int = config.TOP_K_RESULTS
    ) -> Dict:
        """대화형 쿼리 처리"""
        
        # 대화 기록 포함
        history_text = "\n".join([
            f"사용자: {h['query']}\n조수: {h['answer']}"
            for h in chat_history[-5:]  # 최근 5개만
        ])
        
        # 쿼리 임베딩
        query_embedding = self.embedding_mgr.generate_embedding(query)
        
        # 문서 검색
        documents = self.retriever.search_similar_documents(
            query_embedding=query_embedding,
            top_k=top_k
        )
        
        # 컨텍스트 구성
        context = self._build_context(documents)
        
        # 대화형 프롬프트
        prompt = f"""이전 대화:
{history_text}

참고 문서:
{context}

현재 질문: {query}

위 대화 기록과 참고 문서를 바탕으로 한국어로 답변해주세요.

답변:"""
        
        llm_response = self._generate_llm_response(prompt)
        
        return {
            'query': query,
            'answer': llm_response,
            'sources': documents
        }
    
    def close(self):
        """연결 종료"""
        self.embedding_mgr.close()
        self.retriever.close()
```

#### 5. 문서 로더 (document_loader.py)

```python
import os
import json
from typing import List, Dict
from pathlib import Path

class DocumentLoader:
    """다양한 형식의 문서 로드"""
    
    @staticmethod
    def load_text_file(filepath: str) -> Dict:
        """텍스트 파일 로드"""
        with open(filepath, 'r', encoding='utf-8') as f:
            content = f.read()
        
        return {
            'title': Path(filepath).stem,
            'content': content,
            'source': filepath,
            'metadata': {
                'file_type': 'txt',
                'file_size': os.path.getsize(filepath),
                'file_name': os.path.basename(filepath)
            }
        }
    
    @staticmethod
    def load_markdown_file(filepath: str) -> Dict:
        """마크다운 파일 로드"""
        with open(filepath, 'r', encoding='utf-8') as f:
            content = f.read()
        
        # 첫 줄에서 제목 추출
        lines = content.split('\n')
        title = lines[0].replace('#', '').strip() if lines else Path(filepath).stem
        
        return {
            'title': title,
            'content': content,
            'source': filepath,
            'metadata': {
                'file_type': 'markdown',
                'file_size': os.path.getsize(filepath),
                'file_name': os.path.basename(filepath)
            }
        }
    
    @staticmethod
    def load_json_file(filepath: str) -> List[Dict]:
        """JSON 파일 로드"""
        with open(filepath, 'r', encoding='utf-8') as f:
            data = json.load(f)
        
        if isinstance(data, list):
            return data
        else:
            return [data]
    
    @staticmethod
    def load_directory(
        directory: str,
        file_extensions: List[str] = ['.txt', '.md', '.json'],
        recursive: bool = True
    ) -> List[Dict]:
        """디렉토리의 모든 문서 로드"""
        documents = []
        
        if recursive:
            iterator = os.walk(directory)
        else:
            iterator = [(directory, [], os.listdir(directory))]
        
        for root, _, files in iterator:
            for file in files:
                if any(file.endswith(ext) for ext in file_extensions):
                    filepath = os.path.join(root, file)
                    
                    try:
                        if file.endswith('.txt'):
                            doc = DocumentLoader.load_text_file(filepath)
                            documents.append(doc)
                        elif file.endswith('.md'):
                            doc = DocumentLoader.load_markdown_file(filepath)
                            documents.append(doc)
                        elif file.endswith('.json'):
                            docs = DocumentLoader.load_json_file(filepath)
                            documents.extend(docs)
                        
                        print(f"✓ {file} 로드 완료")
                    
                    except Exception as e:
                        print(f"✗ {file} 로드 실패: {e}")
        
        return documents
    
    @staticmethod
    def chunk_text(
        text: str, 
        chunk_size: int = 1000, 
        overlap: int = 200
    ) -> List[str]:
        """긴 텍스트를 청크로 분할"""
        chunks = []
        start = 0
        
        while start < len(text):
            end = start + chunk_size
            chunks.append(text[start:end])
            start = end - overlap
        
        return chunks
```

#### 6. CLI 애플리케이션 (app.py)

```python
from embedding_manager import EmbeddingManager
from rag_engine import RAGEngine
from document_loader import DocumentLoader
import sys

def print_header():
    """헤더 출력"""
    print("=" * 80)
    print("PostgreSQL RAG 시스템".center(80))
    print("=" * 80)
    print()

def print_menu():
    """메뉴 출력"""
    print("\n" + "-" * 80)
    print("1. 문서 추가 (단일 파일)")
    print("2. 문서 추가 (디렉토리)")
    print("3. 질문하기 (단일)")
    print("4. 질문하기 (대화형)")
    print("5. 종료")
    print("-" * 80)

def add_single_file(rag):
    """단일 파일 추가"""
    print("\n=== 문서 추가 (단일 파일) ===")
    filepath = input("파일 경로: ").strip()
    
    if not filepath:
        print("❌ 파일 경로를 입력하세요")
        return
    
    try:
        if filepath.endswith('.txt'):
            doc = DocumentLoader.load_text_file(filepath)
        elif filepath.endswith('.md'):
            doc = DocumentLoader.load_markdown_file(filepath)
        else:
            print("❌ 지원하지 않는 파일 형식 (.txt, .md만 지원)")
            return
        
        doc_id = rag.embedding_mgr.insert_document(
            title=doc['title'],
            content=doc['content'],
            source=doc['source'],
            metadata=doc['metadata']
        )
        print(f"✅ 문서 추가 완료 (ID: {doc_id})")
        
    except Exception as e:
        print(f"❌ 문서 추가 실패: {e}")

def add_directory(rag):
    """디렉토리 문서 추가"""
    print("\n=== 문서 추가 (디렉토리) ===")
    directory = input("디렉토리 경로: ").strip()
    
    if not directory:
        print("❌ 디렉토리 경로를 입력하세요")
        return
    
    try:
        documents = DocumentLoader.load_directory(directory)
        
        if not documents:
            print("❌ 로드할 문서가 없습니다")
            return
        
        print(f"\n📚 총 {len(documents)}개 문서 발견")
        confirm = input("추가하시겠습니까? (y/n): ").strip().lower()
        
        if confirm == 'y':
            doc_ids = rag.embedding_mgr.bulk_insert_documents(documents)
            print(f"\n✅ {len(doc_ids)}개 문서 추가 완료")
        else:
            print("❌ 취소되었습니다")
    
    except Exception as e:
        print(f"❌ 문서 추가 실패: {e}")

def ask_question(rag):
    """단일 질문"""
    print("\n=== 질문하기 ===")
    query = input("질문: ").strip()
    
    if not query:
        return
    
    use_hybrid = input("하이브리드 검색 사용? (y/n, 기본:n): ").strip().lower() == 'y'
    
    print("\n" + "=" * 80)
    result = rag.generate_response(query, use_hybrid=use_hybrid)
    print("=" * 80)
    
    print(f"\n📝 답변:\n{result['answer']}\n")
    print(f"📚 참고 문서 ({len(result['sources'])}개):")
    for i, source in enumerate(result['sources'], 1):
        sim = source.get('similarity', 1 - source.get('distance', 0))
        print(f"  {i}. {source['title']} (유사도: {sim:.2%})")
        print(f"     출처: {source['source']}")

def conversational_mode(rag):
    """대화형 모드"""
    print("\n=== 대화형 모드 ===")
    print("💡 종료하려면 'quit', 'exit', '종료' 입력")
    
    chat_history = []
    
    while True:
        print("\n" + "-" * 80)
        query = input("질문: ").strip()
        
        if query.lower() in ['quit', 'exit', '종료', 'q']:
            print("👋 대화를 종료합니다")
            break
        
        if not query:
            continue
        
        result = rag.conversational_query(query, chat_history)
        
        print(f"\n💬 답변:\n{result['answer']}\n")
        
        # 기록 저장
        chat_history.append({
            'query': query,
            'answer': result['answer']
        })
        
        # 최근 10개만 유지
        if len(chat_history) > 10:
            chat_history = chat_history[-10:]

def main():
    """메인 함수"""
    print_header()
    
    try:
        # RAG 엔진 초기화
        print("🔧 RAG 시스템 초기화 중...")
        rag = RAGEngine()
        print("✅ 초기화 완료\n")
        
        while True:
            print_menu()
            choice = input("\n선택 (1-5): ").strip()
            
            if choice == '1':
                add_single_file(rag)
            elif choice == '2':
                add_directory(rag)
            elif choice == '3':
                ask_question(rag)
            elif choice == '4':
                conversational_mode(rag)
            elif choice == '5':
                print("\n👋 시스템을 종료합니다")
                rag.close()
                break
            else:
                print("❌ 잘못된 선택입니다 (1-5 입력)")
    
    except KeyboardInterrupt:
        print("\n\n⚠️  사용자에 의해 중단되었습니다")
        sys.exit(0)
    except Exception as e:
        print(f"\n❌ 오류 발생: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## 🎯 성능 최적화

### 인덱스 전략

#### IVFFlat 인덱스 (빠른 근사 검색)

```sql
-- 기본 IVFFlat 인덱스
CREATE INDEX documents_embedding_idx ON documents 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- lists 파라미터 선택 가이드:
-- 작은 데이터셋 (< 100K): lists = 100
-- 중간 데이터셋 (100K-1M): lists = sqrt(행수)
-- 큰 데이터셋 (> 1M): lists = sqrt(행수) * 2

-- 예: 1M 행인 경우
-- lists = sqrt(1000000) * 2 = 1000 * 2 = 2000
```

#### HNSW 인덱스 (높은 정확도)
```sql
-- PostgreSQL 14+ 필요
CREATE INDEX documents_embedding_hnsw_idx ON documents 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 파라미터:
-- m: 연결 수 (높을수록 정확, 메모리 증가, 기본 16)
-- ef_construction: 구축 시 탐색 깊이 (높을수록 정확하지만 느림)
```

### 연결 풀링

```python
from psycopg2 import pool
from contextlib import contextmanager
import config

class DatabasePool:
    """데이터베이스 연결 풀"""
    
    def __init__(self, minconn=1, maxconn=10):
        self.pool = pool.ThreadedConnectionPool(
            minconn,
            maxconn,
            **config.DB_CONFIG
        )
    
    @contextmanager
    def get_connection(self):
        """컨텍스트 매니저로 안전한 연결 관리"""
        conn = self.pool.getconn()
        try:
            yield conn
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            self.pool.putconn(conn)
    
    def close_all(self):
        """모든 연결 종료"""
        self.pool.closeall()

# 사용 예시
db_pool = DatabasePool(minconn=2, maxconn=10)

with db_pool.get_connection() as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM documents LIMIT 10")
    results = cursor.fetchall()
```

### 캐싱

```python
import hashlib
from functools import lru_cache
from typing import List

class CachedEmbeddingManager(EmbeddingManager):
    """임베딩 캐싱을 지원하는 매니저"""
    
    def __init__(self, cache_size=1000):
        super().__init__()
        self._cache = {}
        self._max_cache_size = cache_size
    
    def generate_embedding(self, text: str) -> List[float]:
        # MD5 해시로 캐시 키 생성
        cache_key = hashlib.md5(text.encode()).hexdigest()
        
        # 캐시 확인
        if cache_key in self._cache:
            print("✓ 캐시에서 임베딩 로드")
            return self._cache[cache_key]
        
        # 캐시 미스 - 새로 생성
        embedding = super().generate_embedding(text)
        
        # 캐시 저장 (크기 제한)
        if len(self._cache) >= self._max_cache_size:
            # FIFO: 가장 오래된 항목 제거
            self._cache.pop(next(iter(self._cache)))
        
        self._cache[cache_key] = embedding
        return embedding
```

### 쿼리 최적화

```sql
-- 1. EXPLAIN ANALYZE로 쿼리 분석
EXPLAIN ANALYZE
SELECT id, title, content, 
       embedding <-> '[0.1, 0.2, ...]'::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;

-- 2. VACUUM으로 성능 향상
VACUUM ANALYZE documents;

-- 3. 통계 정보 갱신
ANALYZE documents;

-- 4. 인덱스 재구축
REINDEX INDEX documents_embedding_idx;

-- 5. 테이블 통계 확인
SELECT 
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE tablename = 'documents';
```

---

## 🔧 트러블슈팅

### 1. pgvector 설치 실패

**문제:**
```
ERROR: extension "vector" does not exist
```

**해결 방법:**

```bash
# 1. 개발 도구 확인
sudo apt-get install postgresql-server-dev-15 build-essential git

# 2. pgvector 재설치
cd /tmp
rm -rf pgvector
git clone --branch v0.5.1 https://github.com/pgvector/pgvector.git
cd pgvector
make clean
make
sudo make install

# 3. PostgreSQL 재시작
sudo systemctl restart postgresql

# 4. 확장 생성
psql -U postgres -d rag_db
CREATE EXTENSION IF NOT EXISTS vector;

# 5. 확인
SELECT * FROM pg_extension WHERE extname = 'vector';
```

### 2. 임베딩 차원 불일치

**문제:**
```
ERROR: dimension of vector must match declared dimension
```

**해결 방법:**

```python
# 1. 실제 임베딩 차원 확인
from embedding_manager import EmbeddingManager

emb_mgr = EmbeddingManager()
test_embedding = emb_mgr.generate_embedding("test")
print(f"실제 차원: {len(test_embedding)}")

# 2. 테이블 재생성 (데이터 백업 필수!)
# DROP TABLE documents;
# CREATE TABLE documents (..., embedding vector(실제_차원));
```

### 3. Ollama 연결 실패

**문제:**
```
requests.exceptions.ConnectionError: Connection refused
```

**해결 방법:**

```bash
# 1. Ollama 서비스 확인
systemctl status ollama  # Linux
ps aux | grep ollama     # macOS/Linux

# 2. Ollama 시작
ollama serve  # 수동 시작

# 3. 모델 다운로드 확인
ollama list
ollama pull llama3

# 4. 포트 확인
curl http://localhost:11434/api/tags

# 5. 방화벽 확인 (필요시)
sudo ufw allow 11434/tcp
```

### 4. 검색 속도 느림

**문제:** 검색 시간이 수 초 이상 소요

**해결 방법:**

```sql
-- 1. 인덱스 확인
SELECT 
    indexname, 
    indexdef,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes 
WHERE tablename = 'documents';

-- 2. 인덱스가 없다면 생성
CREATE INDEX IF NOT EXISTS documents_embedding_idx ON documents 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 3. VACUUM 실행
VACUUM ANALYZE documents;

-- 4. 쿼리 플랜 확인
EXPLAIN ANALYZE
SELECT * FROM documents
ORDER BY embedding <-> '[...]'::vector
LIMIT 10;
```

### 5. 메모리 부족 (대용량 문서)

**문제:**
```
ERROR: out of memory
```

**해결 방법:**

```python
# 문서를 청크로 분할 처리
from document_loader import DocumentLoader

def process_large_document(filepath, chunk_size=1000):
    """대용량 문서를 청크로 처리"""
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()
    
    # 청크로 분할
    chunks = DocumentLoader.chunk_text(content, chunk_size=chunk_size)
    
    # 각 청크를 별도 문서로 저장
    for i, chunk in enumerate(chunks):
        doc_id = emb_mgr.insert_document(
            title=f"{Path(filepath).stem} - Part {i+1}",
            content=chunk,
            source=filepath,
            metadata={'chunk_index': i, 'total_chunks': len(chunks)}
        )
```

### 6. PostgreSQL 비밀번호 분실

**해결 방법:**

```bash
# 1. pg_hba.conf 수정 (trust 모드로 변경)
sudo nano /var/lib/pgsql/15/data/pg_hba.conf

# 다음 줄을 찾아서 md5를 trust로 변경
# local all all md5
local all all trust

# 2. PostgreSQL 재시작
sudo systemctl restart postgresql

# 3. 비밀번호 변경
psql -U postgres
\password postgres
# 새 비밀번호 입력

# 4. pg_hba.conf 원래대로 복원 (md5)
sudo nano /var/lib/pgsql/15/data/pg_hba.conf
# trust를 다시 md5로 변경

# 5. 재시작
sudo systemctl restart postgresql
```

---

## 💡 실전 예제

### 예제 1: 기술 문서 RAG

```python
# 샘플 기술 문서
tech_docs = [
    {
        'title': 'PostgreSQL 인덱스 최적화',
        'content': '''
        PostgreSQL에서 인덱스는 쿼리 성능을 크게 향상시킵니다.
        B-tree 인덱스는 가장 일반적이며, 대부분의 경우에 사용됩니다.
        GIN 인덱스는 JSONB나 배열 타입에 적합합니다.
        ''',
        'source': '/docs/postgresql/indexing.md',
        'metadata': {'category': 'database', 'level': 'advanced'}
    },
    {
        'title': 'Python 비동기 프로그래밍',
        'content': '''
        asyncio는 Python의 비동기 프로그래밍 라이브러리입니다.
        async/await 키워드를 사용하여 코루틴을 정의합니다.
        I/O 바운드 작업에서 성능 향상을 기대할 수 있습니다.
        ''',
        'source': '/docs/python/async.md',
        'metadata': {'category': 'programming', 'language': 'python'}
    }
]

# 문서 추가
from embedding_manager import EmbeddingManager

emb_mgr = EmbeddingManager()
doc_ids = emb_mgr.bulk_insert_documents(tech_docs)
print(f"✅ {len(doc_ids)}개 문서 추가 완료")

# RAG 질의
from rag_engine import RAGEngine

rag = RAGEngine()

# 일반 검색
result = rag.generate_response("PostgreSQL 인덱스에 대해 설명해주세요")
print(f"답변: {result['answer']}")

# 메타데이터 필터링
result = rag.generate_response(
    "프로그래밍 관련 문서를 찾아주세요",
    metadata_filter={'category': 'programming'}
)

# 정리
emb_mgr.close()
rag.close()
```

### 예제 2: 회사 정책 문서 RAG

```python
# 회사 정책 문서
policy_docs = [
    {
        'title': '재택근무 정책',
        'content': '''
        재택근무는 주 2회까지 가능합니다.
        사전에 팀장 승인이 필요하며, 근무 일지를 작성해야 합니다.
        재택근무 시에도 정상 근무 시간(9-6시)을 준수해야 합니다.
        ''',
        'source': '/policies/remote_work.md',
        'metadata': {
            'category': 'hr',
            'effective_date': '2024-01-01',
            'department': 'all'
        }
    },
    {
        'title': '연차 사용 규정',
        'content': '''
        연차는 입사일 기준으로 부여됩니다.
        1년 미만: 월 1개, 1년 이상: 15개 (최대 25개)
        연차는 시스템을 통해 신청하며, 3일 전 승인이 필요합니다.
        ''',
        'source': '/policies/annual_leave.md',
        'metadata': {
            'category': 'hr',
            'effective_date': '2024-01-01',
            'department': 'all'
        }
    }
]

# 추가 및 검색
emb_mgr = EmbeddingManager()
emb_mgr.bulk_insert_documents(policy_docs)

rag = RAGEngine()
result = rag.generate_response("재택근무를 하려면 어떻게 해야 하나요?")
print(result['answer'])

# 정리
emb_mgr.close()
rag.close()
```

### 예제 3: 대화형 고객 지원

```python
from rag_engine import RAGEngine

rag = RAGEngine()
chat_history = []

# 시뮬레이션
questions = [
    "제품 보증 기간은 얼마나 되나요?",
    "보증 기간이 지나면 수리 비용은 어떻게 되나요?",
    "환불은 가능한가요?"
]

for question in questions:
    print(f"\n사용자: {question}")
    
    result = rag.conversational_query(question, chat_history)
    print(f"봇: {result['answer']}")
    
    # 기록 저장
    chat_history.append({
        'query': question,
        'answer': result['answer']
    })

rag.close()
```

---

## ❓ FAQ

### Q1: pgvector와 전용 벡터 DB(Pinecone, Weaviate 등)의 차이는?

**A:** 

| 항목 | pgvector | 전용 벡터 DB |
|------|----------|--------------|
| **설치** | PostgreSQL 확장 | 별도 서비스 필요 |
| **SQL 지원** | 완전 지원 | 제한적 |
| **비용** | 무료 | 유료 또는 제한 |
| **학습 곡선** | 낮음 (SQL 사용) | 높음 (새 API) |
| **확장성** | 수백만 벡터 | 수억 벡터 |
| **성능** | 좋음 | 매우 좋음 |

**추천:** 대부분의 경우 pgvector로 충분합니다. 수억 개 이상의 벡터를 다루거나 초고속 검색이 필요한 경우에만 전용 벡터 DB를 고려하세요.

### Q2: OpenAI API 대신 Ollama를 사용하는 이유는?

**A:**

**Ollama 장점:**
- ✅ 완전 무료
- ✅ 로컬 실행 (개인정보 보호)
- ✅ 인터넷 불필요
- ✅ 속도 제한 없음

**OpenAI API 장점:**
- ✅ 더 높은 품질 (GPT-4)
- ✅ 설정 간단
- ✅ 서버 불필요

**추천:** 개발/테스트는 Ollama, 프로덕션은 상황에 따라 선택

### Q3: 한국어 문서 처리가 잘 되나요?

**A:** 네, Llama3는 한국어를 잘 지원합니다. 추가 최적화:

```python
# 한국어 전문 검색 설정
CREATE INDEX idx_documents_content_fts_korean ON documents 
USING GIN(to_tsvector('simple', content));  # 한국어는 'simple' 사용

# 한국어 토크나이저 (선택사항)
# pip install konlpy
from konlpy.tag import Okt
okt = Okt()
tokens = okt.morphs(text)
```

### Q4: 얼마나 많은 문서를 저장할 수 있나요?

**A:** PostgreSQL + pgvector는 수백만 개의 문서를 처리할 수 있습니다.

**용량 가이드:**
- 1M 문서 (4096 차원): ~16GB (벡터만)
- 10M 문서: ~160GB
- 최적화 시 100M+ 가능

**권장 사항:**
- 10K 미만: 기본 설정
- 100K - 1M: IVFFlat 인덱스 + 파티셔닝
- 1M 이상: HNSW 인덱스 + 샤딩 고려

### Q5: 실시간으로 문서를 업데이트할 수 있나요?

**A:** 네, 가능합니다.

```python
# 문서 업데이트
def update_document(doc_id, new_content):
    new_embedding = emb_mgr.generate_embedding(new_content)
    
    query = """
        UPDATE documents 
        SET content = %s, embedding = %s, updated_at = CURRENT_TIMESTAMP
        WHERE id = %s
    """
    cursor.execute(query, (new_content, new_embedding, doc_id))
    conn.commit()

# 문서 삭제
def delete_document(doc_id):
    cursor.execute("DELETE FROM documents WHERE id = %s", (doc_id,))
    conn.commit()
```

### Q6: 이미지나 PDF도 처리할 수 있나요?

**A:** 네, 추가 라이브러리가 필요합니다.

```bash
# PDF 처리
pip install PyPDF2 pdfplumber

# 이미지 OCR
pip install pytesseract Pillow

# Office 문서
pip install python-docx openpyxl
```

```python
# PDF 로더 예시
import PyPDF2

def load_pdf(filepath):
    with open(filepath, 'rb') as f:
        reader = PyPDF2.PdfReader(f)
        text = ""
        for page in reader.pages:
            text += page.extract_text()
    return text
```

---

## 📖 참고 자료

### 공식 문서

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Ollama 문서](https://ollama.ai/docs)
- [psycopg2 문서](https://www.psycopg.org/docs/)

### 학습 자료

- [RAG 논문 (원문)](https://arxiv.org/abs/2005.11401)
- [PostgreSQL을 여행하는 입문자를 위한 안내서](https://wikidocs.net/book/8814)
- [pgvector Tutorial](https://github.com/pgvector/pgvector#getting-started)

### 관련 프로젝트

- [LangChain](https://python.langchain.com/) - RAG 프레임워크
- [LlamaIndex](https://www.llamaindex.ai/) - 데이터 프레임워크
- [Haystack](https://haystack.deepset.ai/) - NLP 프레임워크

