# LangChain을 활용한 RAG 시스템 구축 가이드

> 한국어 개발자를 위한 LangChain 기반 RAG(Retrieval-Augmented Generation) 시스템 완벽 구축 가이드

## 📋 목차

- [프로젝트 소개](#-프로젝트-소개)
- [왜 LangChain인가?](#-왜-langchain인가)
- [주요 사양](#-주요-사양-specs)
- [시스템 아키텍처](#-시스템-아키텍처)
- [환경 구축](#-환경-구축)
- [빠른 시작](#-빠른-시작)
- [상세 가이드](#-상세-가이드)
- [벡터 스토어 통합](#-벡터-스토어-통합)
- [고급 RAG 패턴](#-고급-rag-패턴)
- [성능 최적화](#-성능-최적화)
- [프로덕션 배포](#-프로덕션-배포)
- [실무 가이드](#-실무-가이드)
- [트러블슈팅](#-트러블슈팅)
- [FAQ](#-faq)
- [참고 자료](#-참고-자료)

---

## 🎯 프로젝트 소개

이 프로젝트는 **LangChain 프레임워크**를 활용하여 프로덕션급 RAG(Retrieval-Augmented Generation) 시스템을 구축하는 실전 가이드입니다.

### LangChain이란?

**LangChain**은 LLM(Large Language Model) 기반 애플리케이션을 쉽게 구축할 수 있도록 돕는 오픈소스 프레임워크입니다.

- **개발사**: LangChain Inc. (Harrison Chase, CEO)
- **출시**: 2022년 10월
- **GitHub Stars**: 100,000+ (2024년 12월 기준)
- **지원 언어**: Python, JavaScript/TypeScript
- **라이선스**: MIT License

### 주요 특징

- ✅ **통합 프레임워크**: 100+ LLM, 벡터DB, 문서로더를 하나의 인터페이스로 사용
- ✅ **체인 구성**: 여러 컴포넌트를 체인으로 연결하여 복잡한 워크플로우 구현
- ✅ **모듈화**: Document Loaders, Text Splitters, Embeddings, Vector Stores, Retrievers, Chains 등
- ✅ **LCEL (LangChain Expression Language)**: 선언적 체인 구성 및 스트리밍 지원
- ✅ **LangSmith**: 디버깅, 평가, 모니터링 도구 내장
- ✅ **LangGraph**: 복잡한 멀티 에이전트 워크플로우 구현
- ✅ **생태계**: 활발한 커뮤니티와 풍부한 문서화

### 학습 목표

이 가이드를 완료하면 다음을 할 수 있습니다:

1. LangChain 핵심 컴포넌트 이해 및 활용
2. 다양한 벡터 스토어(Pinecone, Chroma, FAISS, MongoDB, Supabase)와 RAG 통합
3. Document Loaders로 다양한 형식(PDF, DOCX, CSV, HTML 등) 문서 처리
4. Text Splitters로 최적화된 청킹 전략 구현
5. Retrieval Chains, Conversational Chains, RetrievalQA 구현
6. Self-Query, Parent-Document, Multi-Query 등 고급 RAG 패턴 구현
7. LangSmith로 RAG 파이프라인 디버깅 및 최적화
8. 프로덕션 환경 배포 및 모니터링

### 실제 활용 사례

LangChain 기반 RAG는 다음과 같은 시나리오에서 효과적입니다:

- **📚 기업 지식 관리**: 사내 문서, 정책, FAQ를 통합한 AI 챗봇
- **🏢 고객 지원**: 제품 매뉴얼, 티켓 히스토리 기반 자동 응답
- **📊 금융 분석**: 재무 보고서, 뉴스, 규정 통합 분석
- **🔍 법률 리서치**: 판례, 법령, 계약서 검색 및 요약
- **🎓 교육 플랫폼**: 강의 자료, 교재 기반 개인화 학습 도우미
- **💻 코드 검색**: 사내 코드베이스 검색 및 문서 자동 생성
- **🌐 멀티모달 검색**: 텍스트, 이미지, 표, 차트 통합 검색

---

## 🤔 왜 LangChain인가?

### 기존 방식 vs LangChain

| 기능 | 직접 구현 (Raw API) | LangChain |
|------|---------------------|-----------|
| **개발 속도** | 느림 (모든 것 직접 구현) | **빠름 (즉시 사용 가능)** |
| **LLM 전환** | 코드 전면 수정 | **설정만 변경** |
| **벡터DB 변경** | 인터페이스 재구현 | **클래스만 교체** |
| **문서 로딩** | 각 형식마다 파서 구현 | **100+ 로더 기본 제공** |
| **체인 구성** | 복잡한 로직 필요 | **선언적 LCEL 사용** |
| **디버깅** | print 문 또는 로깅 | **LangSmith 시각화** |
| **프롬프트 관리** | 코드에 하드코딩 | **템플릿 및 Hub 활용** |
| **스트리밍** | 직접 구현 필요 | **기본 지원** |
| **메모리 관리** | 수동 구현 | **ConversationBufferMemory 등** |

### RAG에 최적인 이유

1. **추상화와 유연성**: 벡터DB, LLM 제공사를 쉽게 교체 가능
2. **풍부한 통합**: 100+ 문서 로더, 50+ 벡터 스토어 기본 지원
3. **고급 RAG 패턴**: Self-Query, Parent-Document, Multi-Query 등 내장
4. **프로덕션 지원**: 스트리밍, 에러 핸들링, 재시도 로직 기본 제공
5. **모니터링**: LangSmith로 체인 실행 추적 및 성능 분석
6. **커뮤니티**: 활발한 생태계와 지속적인 업데이트
7. **표준화**: RAG 구현의 사실상 표준 프레임워크

---

## 📋 주요 사양 (Specs)

### 지원하는 LLM

| 제공사 | 모델 예시 | LangChain 클래스 |
|--------|----------|------------------|
| **OpenAI** | GPT-4o, GPT-4o-mini, GPT-3.5-turbo | `ChatOpenAI` |
| **Anthropic** | Claude 3.5 Sonnet, Claude 3 Opus/Haiku | `ChatAnthropic` |
| **Google** | Gemini 2.5 Flash, Gemini 2.5 Pro | `ChatGoogleGenerativeAI` |
| **AWS Bedrock** | Claude, Llama 3.1, Titan | `BedrockChat` |
| **Azure OpenAI** | GPT-4o (Azure) | `AzureChatOpenAI` |
| **Ollama** | Llama 3.1, Mistral, Qwen | `ChatOllama` |
| **Cohere** | Command R+, Command Light | `ChatCohere` |
| **Hugging Face** | 모든 text-generation 모델 | `HuggingFaceHub` |

### 지원하는 임베딩 모델

| 제공사 | 모델 | 차원 | LangChain 클래스 |
|--------|------|------|------------------|
| **OpenAI** | text-embedding-3-small | 1536 | `OpenAIEmbeddings` |
| **OpenAI** | text-embedding-3-large | 3072 | `OpenAIEmbeddings` |
| **Cohere** | embed-multilingual-v3 | 1024 | `CohereEmbeddings` |
| **Google** | text-embedding-004 | 768 | `GoogleGenerativeAIEmbeddings` |
| **AWS Bedrock** | amazon.titan-embed-text-v1 | 1024 | `BedrockEmbeddings` |
| **Hugging Face** | BAAI/bge-large-en-v1.5 | 1024 | `HuggingFaceEmbeddings` |
| **Ollama** | nomic-embed-text | 768 | `OllamaEmbeddings` |
| **Sentence Transformers** | all-MiniLM-L6-v2 | 384 | `HuggingFaceEmbeddings` |

### 지원하는 벡터 스토어

| 벡터 스토어 | 타입 | 특징 | LangChain 클래스 |
|------------|------|------|------------------|
| **FAISS** | 로컬 | 빠른 검색, 메모리 기반 | `FAISS` |
| **Chroma** | 로컬/클라우드 | 경량, 임베딩 관리 | `Chroma` |
| **Pinecone** | 클라우드 | 완전 관리형, 고성능 | `Pinecone` |
| **Weaviate** | 클라우드/자체호스팅 | GraphQL, 하이브리드 검색 | `Weaviate` |
| **MongoDB Atlas** | 클라우드 | 통합 DB, 메타데이터 필터 | `MongoDBAtlasVectorSearch` |
| **Supabase** | 클라우드 | PostgreSQL + pgvector | `SupabaseVectorStore` |
| **Qdrant** | 클라우드/자체호스팅 | Rust 기반, 고성능 | `Qdrant` |
| **Milvus** | 클라우드/자체호스팅 | 엔터프라이즈급, 확장성 | `Milvus` |
| **Elasticsearch** | 클라우드/자체호스팅 | 하이브리드 검색, 분석 | `ElasticsearchStore` |
| **Redis** | 클라우드/자체호스팅 | 인메모리, 초고속 | `Redis` |

### 지원하는 문서 로더

| 문서 타입 | LangChain Loader | 용도 |
|----------|------------------|------|
| **PDF** | `PyPDFLoader`, `PDFPlumberLoader`, `UnstructuredPDFLoader` | PDF 문서 로딩 |
| **Word** | `Docx2txtLoader`, `UnstructuredWordDocumentLoader` | DOCX, DOC 파일 |
| **CSV** | `CSVLoader` | 스프레드시트 데이터 |
| **HTML** | `UnstructuredHTMLLoader`, `BSHTMLLoader` | 웹 페이지 |
| **Markdown** | `UnstructuredMarkdownLoader` | 마크다운 문서 |
| **JSON** | `JSONLoader` | JSON 데이터 |
| **텍스트** | `TextLoader` | 일반 텍스트 파일 |
| **Excel** | `UnstructuredExcelLoader` | XLSX, XLS 파일 |
| **PowerPoint** | `UnstructuredPowerPointLoader` | PPT, PPTX 파일 |
| **이메일** | `UnstructuredEmailLoader` | EML, MSG 파일 |
| **코드** | `GenericLoader` + `LanguageParser` | Python, JS, Java 등 |
| **웹** | `WebBaseLoader`, `SeleniumURLLoader` | URL 크롤링 |
| **YouTube** | `YoutubeLoader` | YouTube 자막 |
| **Notion** | `NotionDirectoryLoader` | Notion 페이지 |
| **Google Drive** | `GoogleDriveLoader` | Google Docs, Sheets |
| **S3** | `S3FileLoader`, `S3DirectoryLoader` | AWS S3 파일 |

### 지원하는 Text Splitters

| Splitter | 설명 | 사용 시나리오 |
|----------|------|--------------|
| **RecursiveCharacterTextSplitter** | 계층적 구분자로 분할 (\n\n → \n → 공백) | **일반적인 텍스트 (기본 권장)** |
| **CharacterTextSplitter** | 단일 구분자로 분할 | 단순 텍스트 |
| **TokenTextSplitter** | 토큰 수 기준 분할 | LLM 토큰 제한 준수 |
| **MarkdownHeaderTextSplitter** | 마크다운 헤더 기준 분할 | 마크다운 문서 |
| **HTMLHeaderTextSplitter** | HTML 태그 기준 분할 | HTML 문서 |
| **LatexTextSplitter** | LaTeX 구조 기준 분할 | 학술 논문 |
| **PythonCodeTextSplitter** | Python 구문 기준 분할 | Python 코드 |
| **RecursiveJsonSplitter** | JSON 구조 기준 분할 | JSON 데이터 |
| **SentenceTransformersTokenTextSplitter** | 임베딩 모델 토큰 기준 | 임베딩 길이 제한 |

### 예상 비용 (2025년 기준)

#### 소규모 프로젝트 (개발/테스트)
```
✅ LangChain 프레임워크
- LangChain Core: 무료 (오픈소스)
- LangSmith: 무료 티어 (5,000 traces/월)

✅ 벡터 스토어
- Chroma (로컬): $0/월
- FAISS (로컬): $0/월
- Pinecone Starter: $0/월 (100,000 벡터)

✅ LLM API
- OpenAI GPT-4o-mini: ~$5-10/월 (월 1,000 쿼리)
- Ollama (로컬): $0/월

✅ 임베딩 API
- OpenAI text-embedding-3-small: ~$0.20/월
- Ollama (로컬): $0/월

📊 총 예상 비용: $0-10/월
```

#### 중규모 프로젝트 (프로덕션)
```
✅ LangChain 프레임워크
- LangSmith Pro: $39/월 (100,000 traces)

✅ 벡터 스토어
- Pinecone Standard: $70/월 (1M 벡터, 1 pod)
- MongoDB Atlas M10: $57/월
- Supabase Pro: $25/월

✅ LLM API
- OpenAI GPT-4o: ~$100-200/월 (월 10,000 쿼리)
- Anthropic Claude 3.5 Sonnet: ~$150/월

✅ 임베딩 API
- OpenAI embeddings: ~$2-5/월

📊 총 예상 비용: $200-400/월
```

#### 대규모 엔터프라이즈
```
✅ LangChain 프레임워크
- LangSmith Enterprise: Custom pricing (1M+ traces)

✅ 벡터 스토어
- Pinecone Enterprise: $500-2,000/월
- MongoDB Atlas M30+: $440+/월
- Weaviate Enterprise: Custom

✅ LLM API
- OpenAI GPT-4o: $1,000-5,000/월 (대량 트래픽)
- Azure OpenAI: Custom pricing

✅ 인프라
- AWS/GCP/Azure: $500-2,000/월

📊 총 예상 비용: $2,000-10,000+/월
```

**비용 최적화 팁:**
- **로컬 모델 활용**: Ollama로 임베딩/LLM 비용 절감
- **캐싱**: 동일 쿼리 재사용으로 API 호출 감소
- **배치 처리**: 벡터 임베딩 일괄 생성
- **모델 선택**: GPT-4o-mini, Claude 3 Haiku 등 저비용 모델 활용
- **LangSmith 무료 티어**: 개발 환경에서 활용

---

## 🏗 시스템 아키텍처

### LangChain RAG 워크플로우

```mermaid
graph TD
    User[사용자 질문] --> App[애플리케이션]

    App -->|LCEL Chain| RAGChain{RAG Chain}

    RAGChain -->|1. 임베딩| Embeddings[Embeddings<br/>OpenAI/Cohere/Ollama]
    Embeddings -->|쿼리 벡터| RAGChain

    RAGChain -->|2. 검색| VectorStore[(Vector Store<br/>Pinecone/Chroma/FAISS)]
    VectorStore -->|관련 문서| RAGChain

    RAGChain -->|3. 프롬프트 구성| PromptTemplate[Prompt Template<br/>Context + Question]
    PromptTemplate -->|최종 프롬프트| RAGChain

    RAGChain -->|4. LLM 호출| LLM[LLM<br/>GPT-4o/Claude/Gemini]
    LLM -->|답변| RAGChain

    RAGChain -->|결과| App
    App --> User

    style RAGChain fill:#4A90E2,stroke:#357ABD,stroke-width:3px,color:#fff
    style VectorStore fill:#50C878,stroke:#3A9B5C,stroke-width:2px
```

### 문서 인덱싱 파이프라인

```mermaid
graph LR
    Docs[원본 문서<br/>PDF/DOCX/HTML] -->|1. Load| Loader[Document Loaders<br/>PyPDFLoader, etc.]

    Loader -->|Document 객체| Splitter[Text Splitters<br/>RecursiveCharacterTextSplitter]

    Splitter -->|청크 분할| Chunks[Document Chunks<br/>1000자씩]

    Chunks -->|2. Embed| Embeddings[Embeddings Model<br/>OpenAI/Cohere]

    Embeddings -->|벡터 생성| VectorStore[(3. Store<br/>Vector Store<br/>Pinecone/Chroma)]

    VectorStore -->|인덱싱 완료| Ready[RAG 준비 완료]
```

### LangChain 핵심 컴포넌트 아키텍처

```
                    LangChain RAG 시스템
                            |
        +-------------------+-------------------+
        |                   |                   |
    Models            Retrievers            Chains
        |                   |                   |
   +----+----+         +----+----+         +----+----+
   |         |         |         |         |         |
  LLMs  Embeddings  Vector    Index     LCEL   Runnable
                     Store
        |                   |                   |
        +-------------------+-------------------+
                            |
              LangChain Core Framework
                            |
        +-------------------+-------------------+
        |                   |                   |
   Document              Memory            Callbacks
   Loaders               Buffer            LangSmith
```

### 상세 RAG 체인 구조

```python
# LCEL (LangChain Expression Language) 체인 구조

RAG_CHAIN = (
    # 1. 입력: 사용자 질문
    {"context": retriever, "question": RunnablePassthrough()}

    # 2. 프롬프트 생성
    | prompt_template

    # 3. LLM 호출
    | llm

    # 4. 출력 파싱
    | StrOutputParser()
)
```

### 프로덕션 아키텍처

```
[클라이언트 계층]
웹 애플리케이션 (React/Vue) | 모바일 앱 | Slack Bot
                    |
              HTTPS/WebSocket
                    |
                    v
[API 계층]
FastAPI/Flask/Express Server
├─ 인증/인가 (JWT)
├─ Rate Limiting (Redis)
├─ Request Validation
└─ LangChain 체인 실행
                    |
                    v
[LangChain 계층]
├─ RAG Chain (LCEL)
├─ Conversational Chain (Memory)
├─ Agent Executor (Tools)
└─ LangSmith Tracing
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
   [Vector DB]   [LLM API]  [Embeddings]
   - Pinecone    - OpenAI    - OpenAI
   - Chroma      - Anthropic - Cohere
   - MongoDB     - Bedrock   - Ollama
   - Supabase
                    |
                    v
[모니터링 계층]
├─ LangSmith (체인 추적, 성능 분석)
├─ Prometheus/Grafana (메트릭)
├─ ELK Stack (로그)
└─ Sentry (에러 추적)
```

---

## 🚀 환경 구축

### 시스템 요구사항

- **Python**: 3.8 이상 (3.10+ 권장)
- **메모리**: 최소 4GB RAM (8GB+ 권장)
- **운영체제**: Windows 10+, macOS 11+, Ubuntu 20.04+
- **패키지 관리자**: pip 또는 poetry

### 1. LangChain 설치

#### 핵심 패키지 설치

```bash
# 기본 LangChain 설치
pip install langchain langchain-community

# 주요 통합 패키지
pip install langchain-openai      # OpenAI 통합
pip install langchain-anthropic   # Anthropic (Claude) 통합
pip install langchain-google-genai # Google (Gemini) 통합
pip install langchain-cohere      # Cohere 통합
```

#### 벡터 스토어별 설치

```bash
# FAISS (로컬, 빠른 검색)
pip install faiss-cpu  # CPU 버전
# pip install faiss-gpu  # GPU 버전 (CUDA 필요)

# Chroma (로컬, 경량)
pip install chromadb

# Pinecone (클라우드)
pip install pinecone-client

# MongoDB Atlas
pip install pymongo

# Supabase
pip install supabase

# Weaviate
pip install weaviate-client

# Qdrant
pip install qdrant-client
```

#### 문서 로더 및 유틸리티

```bash
# 문서 처리
pip install pypdf          # PDF 로딩
pip install python-docx    # Word 문서
pip install beautifulsoup4 # HTML 파싱
pip install lxml           # HTML/XML 파싱

# Unstructured (다양한 포맷 지원)
pip install unstructured
pip install "unstructured[pdf]"

# 텍스트 분할
pip install tiktoken       # OpenAI 토큰 카운터

# 환경 변수 관리
pip install python-dotenv
```

#### 전체 requirements.txt

```txt
# LangChain 핵심
langchain==0.1.0
langchain-community==0.0.13
langchain-core==0.1.10

# LLM 통합
langchain-openai==0.0.2
langchain-anthropic==0.0.1
langchain-google-genai==0.0.5
langchain-cohere==0.0.3

# 벡터 스토어
faiss-cpu==1.7.4
chromadb==0.4.22
pinecone-client==3.0.0
pymongo==4.6.1
supabase==2.3.0

# 문서 처리
pypdf==3.17.4
python-docx==1.1.0
beautifulsoup4==4.12.2
lxml==5.1.0
unstructured==0.11.8
tiktoken==0.5.2

# 유틸리티
python-dotenv==1.0.0
requests==2.31.0

# 모니터링 (선택)
langsmith==0.0.77
```

### 2. 환경 변수 설정

`.env` 파일 생성:

```env
# LLM API 키
OPENAI_API_KEY=sk-your-openai-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
GOOGLE_API_KEY=your-google-api-key
COHERE_API_KEY=your-cohere-key

# 벡터 스토어
PINECONE_API_KEY=your-pinecone-key
PINECONE_ENVIRONMENT=us-east-1-aws  # 또는 다른 환경

# MongoDB Atlas
MONGODB_URI=mongodb+srv://user:password@cluster.mongodb.net/

# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-supabase-anon-key

# LangSmith (선택, 디버깅용)
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your-langsmith-key
LANGCHAIN_PROJECT=my-rag-project

# Ollama (로컬 LLM, 선택)
OLLAMA_BASE_URL=http://localhost:11434
```

### 3. 설치 확인

```python
# test_setup.py
import os
from dotenv import load_dotenv

load_dotenv()

def test_imports():
    """패키지 import 테스트"""
    print("1️⃣ 패키지 import 테스트...\n")

    try:
        import langchain
        print(f"✅ LangChain 버전: {langchain.__version__}")

        from langchain_openai import ChatOpenAI, OpenAIEmbeddings
        print("✅ LangChain OpenAI 통합")

        from langchain_community.vectorstores import FAISS, Chroma
        print("✅ 벡터 스토어 (FAISS, Chroma)")

        from langchain.text_splitter import RecursiveCharacterTextSplitter
        print("✅ Text Splitter")

        from langchain_community.document_loaders import PyPDFLoader
        print("✅ Document Loaders")

        from langchain.chains import RetrievalQA
        print("✅ Chains")

        return True
    except ImportError as e:
        print(f"❌ Import 실패: {e}")
        return False

def test_openai_connection():
    """OpenAI API 연결 테스트"""
    print("\n2️⃣ OpenAI API 연결 테스트...\n")

    try:
        from langchain_openai import ChatOpenAI, OpenAIEmbeddings

        # LLM 테스트
        llm = ChatOpenAI(
            model="gpt-4o-mini",
            temperature=0
        )
        response = llm.invoke("안녕하세요!")
        print(f"✅ LLM 응답: {response.content[:50]}...")

        # 임베딩 테스트
        embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
        vector = embeddings.embed_query("테스트")
        print(f"✅ 임베딩 차원: {len(vector)}")

        return True
    except Exception as e:
        print(f"❌ OpenAI 연결 실패: {e}")
        return False

def test_vector_store():
    """벡터 스토어 테스트"""
    print("\n3️⃣ 벡터 스토어 테스트...\n")

    try:
        from langchain_openai import OpenAIEmbeddings
        from langchain_community.vectorstores import FAISS
        from langchain.schema import Document

        # 샘플 문서
        docs = [
            Document(page_content="LangChain은 LLM 애플리케이션 프레임워크입니다."),
            Document(page_content="RAG는 검색 증강 생성을 의미합니다."),
        ]

        # FAISS 벡터 스토어 생성
        embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
        vectorstore = FAISS.from_documents(docs, embeddings)

        # 검색 테스트
        results = vectorstore.similarity_search("LangChain이란?", k=1)
        print(f"✅ 검색 결과: {results[0].page_content}")

        return True
    except Exception as e:
        print(f"❌ 벡터 스토어 테스트 실패: {e}")
        return False

def main():
    print("=" * 60)
    print("  LangChain RAG 환경 설정 확인")
    print("=" * 60 + "\n")

    imports_ok = test_imports()
    openai_ok = test_openai_connection()
    vectorstore_ok = test_vector_store()

    print("\n" + "=" * 60)
    if imports_ok and openai_ok and vectorstore_ok:
        print("🎉 모든 설정이 완료되었습니다!")
        print("   이제 LangChain RAG 시스템을 구축할 준비가 되었습니다.")
    else:
        print("⚠️  일부 설정이 완료되지 않았습니다.")
        print("   위의 오류 메시지를 확인하고 문제를 해결하세요.")
    print("=" * 60)

if __name__ == "__main__":
    main()
```

**실행:**
```bash
python test_setup.py
```

**예상 출력:**
```
============================================================
  LangChain RAG 환경 설정 확인
============================================================

1️⃣ 패키지 import 테스트...

✅ LangChain 버전: 0.1.0
✅ LangChain OpenAI 통합
✅ 벡터 스토어 (FAISS, Chroma)
✅ Text Splitter
✅ Document Loaders
✅ Chains

2️⃣ OpenAI API 연결 테스트...

✅ LLM 응답: 안녕하세요! 무엇을 도와드릴까요?...
✅ 임베딩 차원: 1536

3️⃣ 벡터 스토어 테스트...

✅ 검색 결과: LangChain은 LLM 애플리케이션 프레임워크입니다.

============================================================
🎉 모든 설정이 완료되었습니다!
   이제 LangChain RAG 시스템을 구축할 준비가 되었습니다.
============================================================
```

---

## ⚡ 빠른 시작

### 기본 RAG 시스템 (5분 만에 구현)

```python
# quick_start.py
import os
from dotenv import load_dotenv

from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import RetrievalQA
from langchain.schema import Document

# 환경 변수 로드
load_dotenv()

# 1. 샘플 문서 준비
documents = [
    Document(
        page_content="LangChain은 LLM 기반 애플리케이션을 구축하기 위한 오픈소스 프레임워크입니다. "
                     "2022년 10월 Harrison Chase가 개발했으며, Python과 JavaScript를 지원합니다.",
        metadata={"source": "langchain_intro.txt"}
    ),
    Document(
        page_content="RAG(Retrieval-Augmented Generation)는 외부 지식을 검색하여 LLM의 답변을 강화하는 기술입니다. "
                     "벡터 데이터베이스에서 관련 문서를 찾아 컨텍스트로 제공합니다.",
        metadata={"source": "rag_intro.txt"}
    ),
    Document(
        page_content="LangChain의 주요 컴포넌트는 Models, Prompts, Chains, Memory, Agents입니다. "
                     "LCEL(LangChain Expression Language)을 사용하여 체인을 구성할 수 있습니다.",
        metadata={"source": "langchain_components.txt"}
    ),
]

# 2. 텍스트 분할 (청킹)
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50
)
splits = text_splitter.split_documents(documents)
print(f"📄 문서 청크 수: {len(splits)}\n")

# 3. 임베딩 생성 및 벡터 스토어 구축
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(splits, embeddings)
print("✅ 벡터 스토어 생성 완료\n")

# 4. Retriever 생성
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 2}  # 상위 2개 문서 검색
)

# 5. LLM 설정
llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0
)

# 6. RAG Chain 구성
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",  # 모든 문서를 한 번에 전달
    retriever=retriever,
    return_source_documents=True
)

# 7. 질문하기
def ask_question(question: str):
    """RAG 시스템에 질문"""
    print(f"❓ 질문: {question}\n")

    result = qa_chain.invoke({"query": question})

    print(f"💡 답변: {result['result']}\n")
    print(f"📚 참고 문서 ({len(result['source_documents'])}개):")
    for i, doc in enumerate(result['source_documents'], 1):
        print(f"  {i}. {doc.metadata.get('source', 'Unknown')}")
        print(f"     {doc.page_content[:100]}...\n")

# 테스트
if __name__ == "__main__":
    print("=" * 70)
    print("  LangChain RAG 빠른 시작")
    print("=" * 70 + "\n")

    # 질문 1
    ask_question("LangChain이란 무엇인가요?")

    print("-" * 70 + "\n")

    # 질문 2
    ask_question("RAG의 주요 컴포넌트는 무엇인가요?")
```

**실행:**
```bash
python quick_start.py
```

**예상 출력:**
```
======================================================================
  LangChain RAG 빠른 시작
======================================================================

📄 문서 청크 수: 3

✅ 벡터 스토어 생성 완료

❓ 질문: LangChain이란 무엇인가요?

💡 답변: LangChain은 LLM 기반 애플리케이션을 구축하기 위한 오픈소스
프레임워크입니다. 2022년 10월 Harrison Chase가 개발했으며, Python과
JavaScript를 지원합니다. 주요 컴포넌트로는 Models, Prompts, Chains,
Memory, Agents가 있으며, LCEL(LangChain Expression Language)을 사용하여
체인을 구성할 수 있습니다.

📚 참고 문서 (2개):
  1. langchain_intro.txt
     LangChain은 LLM 기반 애플리케이션을 구축하기 위한 오픈소스 프레임워크입니다. ...

  2. langchain_components.txt
     LangChain의 주요 컴포넌트는 Models, Prompts, Chains, Memory, Agents입니다. ...

----------------------------------------------------------------------

❓ 질문: RAG의 주요 컴포넌트는 무엇인가요?

💡 답변: RAG(Retrieval-Augmented Generation)의 주요 컴포넌트는 외부 지식을
검색하는 검색 시스템과 벡터 데이터베이스입니다. 벡터 데이터베이스에서 관련
문서를 찾아 컨텍스트로 제공하여 LLM의 답변을 강화합니다.

📚 참고 문서 (2개):
  1. rag_intro.txt
     RAG(Retrieval-Augmented Generation)는 외부 지식을 검색하여 LLM의 답변을 강화하는 기술입니다. ...

  2. langchain_components.txt
     LangChain의 주요 컴포넌트는 Models, Prompts, Chains, Memory, Agents입니다. ...
```

### LCEL 방식으로 간단하게 구현

```python
# quick_start_lcel.py
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.schema import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

load_dotenv()

# 문서 준비
documents = [
    Document(page_content="LangChain은 LLM 애플리케이션 프레임워크입니다."),
    Document(page_content="RAG는 검색 증강 생성 기술입니다."),
]

# 벡터 스토어 구축
text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
splits = text_splitter.split_documents(documents)
vectorstore = FAISS.from_documents(splits, OpenAIEmbeddings())
retriever = vectorstore.as_retriever()

# 프롬프트 템플릿
template = """다음 컨텍스트를 바탕으로 질문에 답변하세요:

컨텍스트: {context}

질문: {question}

답변:"""

prompt = ChatPromptTemplate.from_template(template)

# LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# LCEL Chain 구성
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 실행
if __name__ == "__main__":
    response = rag_chain.invoke("LangChain이란?")
    print(f"답변: {response}")
```

---

## 📚 상세 가이드

### 1. Document Loaders: 다양한 문서 로딩

#### 1.1 PDF 문서 로딩

```python
from langchain_community.document_loaders import PyPDFLoader, PDFPlumberLoader

# 방법 1: PyPDFLoader (기본, 빠름)
loader = PyPDFLoader("documents/company_handbook.pdf")
pages = loader.load()
print(f"총 페이지: {len(pages)}")
print(f"첫 페이지 내용: {pages[0].page_content[:200]}...")

# 방법 2: PDFPlumberLoader (표 추출 우수)
loader = PDFPlumberLoader("documents/financial_report.pdf")
pages = loader.load()

# 각 페이지는 Document 객체
for i, page in enumerate(pages[:3]):
    print(f"\n페이지 {i+1} 메타데이터: {page.metadata}")
    print(f"내용: {page.page_content[:100]}...")
```

**메타데이터:**
```python
{
    'source': 'documents/company_handbook.pdf',
    'page': 0  # 페이지 번호
}
```

#### 1.2 Word 문서 로딩

```python
from langchain_community.document_loaders import Docx2txtLoader

loader = Docx2txtLoader("documents/policy.docx")
docs = loader.load()

print(f"문서 수: {len(docs)}")
print(f"내용: {docs[0].page_content}")
```

#### 1.3 CSV 데이터 로딩

```python
from langchain_community.document_loaders import CSVLoader

# 기본 로딩
loader = CSVLoader(
    file_path="data/products.csv",
    encoding="utf-8"
)
docs = loader.load()

# 특정 컬럼만 사용
loader = CSVLoader(
    file_path="data/products.csv",
    csv_args={
        'delimiter': ',',
        'quotechar': '"',
    },
    source_column="product_name"  # 메타데이터에 포함될 컬럼
)
docs = loader.load()

# 각 행이 하나의 Document
for doc in docs[:3]:
    print(f"내용: {doc.page_content}")
    print(f"메타데이터: {doc.metadata}\n")
```

#### 1.4 웹 페이지 로딩

```python
from langchain_community.document_loaders import WebBaseLoader

# 단일 URL
loader = WebBaseLoader("https://python.langchain.com/docs/introduction/")
docs = loader.load()

# 여러 URL
urls = [
    "https://python.langchain.com/docs/introduction/",
    "https://python.langchain.com/docs/get_started/quickstart/",
]
loader = WebBaseLoader(urls)
docs = loader.load()

print(f"로드된 문서: {len(docs)}개")
```

#### 1.5 디렉토리 일괄 로딩

```python
from langchain_community.document_loaders import DirectoryLoader, TextLoader

# 특정 확장자 파일만 로딩
loader = DirectoryLoader(
    "documents/",
    glob="**/*.txt",  # 모든 하위 디렉토리의 .txt 파일
    loader_cls=TextLoader,
    loader_kwargs={'encoding': 'utf-8'}
)
docs = loader.load()

print(f"로드된 문서: {len(docs)}개")

# PDF 파일 일괄 로딩
from langchain_community.document_loaders import PyPDFLoader

loader = DirectoryLoader(
    "documents/",
    glob="**/*.pdf",
    loader_cls=PyPDFLoader
)
docs = loader.load()
```

#### 1.6 코드 파일 로딩

```python
from langchain_community.document_loaders.generic import GenericLoader
from langchain_community.document_loaders.parsers import LanguageParser
from langchain_text_splitters import Language

# Python 코드 로딩
loader = GenericLoader.from_filesystem(
    "src/",
    glob="**/*.py",
    suffixes=[".py"],
    parser=LanguageParser(language=Language.PYTHON, parser_threshold=500)
)
docs = loader.load()

# JavaScript 코드 로딩
loader = GenericLoader.from_filesystem(
    "frontend/",
    glob="**/*.js",
    suffixes=[".js"],
    parser=LanguageParser(language=Language.JS)
)
docs = loader.load()
```

### 2. Text Splitters: 효과적인 청킹 전략

#### 2.1 RecursiveCharacterTextSplitter (가장 일반적)

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 기본 설정
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,        # 청크 크기
    chunk_overlap=200,      # 청크 간 중복
    length_function=len,    # 길이 측정 함수
    is_separator_regex=False,
)

# 텍스트 분할
text = """
LangChain은 LLM 애플리케이션을 구축하기 위한 프레임워크입니다.

주요 기능:
1. 모델 통합
2. 프롬프트 관리
3. 체인 구성
"""

chunks = text_splitter.split_text(text)
print(f"청크 수: {len(chunks)}")
for i, chunk in enumerate(chunks):
    print(f"\n청크 {i+1}:\n{chunk}")

# Document 객체 분할
from langchain.schema import Document

docs = [Document(page_content=text, metadata={"source": "intro.txt"})]
split_docs = text_splitter.split_documents(docs)

for doc in split_docs:
    print(f"내용: {doc.page_content[:100]}...")
    print(f"메타데이터: {doc.metadata}\n")
```

**분할 전략:**
```python
# 계층적 구분자 순서:
# 1. "\n\n" (문단)
# 2. "\n"   (줄바꿈)
# 3. " "    (공백)
# 4. ""     (문자 단위)
```

#### 2.2 TokenTextSplitter (토큰 기반)

```python
from langchain.text_splitter import TokenTextSplitter

# OpenAI 토큰 기준
text_splitter = TokenTextSplitter(
    encoding_name="cl100k_base",  # GPT-4 토큰 인코딩
    chunk_size=500,                # 500 토큰
    chunk_overlap=50
)

chunks = text_splitter.split_text(text)

# tiktoken으로 토큰 수 확인
import tiktoken
encoding = tiktoken.get_encoding("cl100k_base")
for i, chunk in enumerate(chunks):
    token_count = len(encoding.encode(chunk))
    print(f"청크 {i+1}: {token_count} 토큰")
```

#### 2.3 MarkdownHeaderTextSplitter

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

markdown_text = """
# LangChain 가이드

## 소개

LangChain은 LLM 프레임워크입니다.

## 주요 기능

### 모델 통합

다양한 LLM을 지원합니다.

### 체인 구성

LCEL을 사용합니다.
"""

# 헤더 기준 분할
headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]

markdown_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on
)
md_header_splits = markdown_splitter.split_text(markdown_text)

for doc in md_header_splits:
    print(f"내용: {doc.page_content}")
    print(f"메타데이터: {doc.metadata}\n")
```

**출력:**
```python
{
    'Header 1': 'LangChain 가이드',
    'Header 2': '소개'
}
{
    'Header 1': 'LangChain 가이드',
    'Header 2': '주요 기능',
    'Header 3': '모델 통합'
}
```

#### 2.4 PythonCodeTextSplitter

```python
from langchain.text_splitter import PythonCodeTextSplitter

python_code = """
def hello_world():
    print("Hello, World!")

class MyClass:
    def __init__(self):
        self.value = 0

    def increment(self):
        self.value += 1
"""

python_splitter = PythonCodeTextSplitter(
    chunk_size=100,
    chunk_overlap=0
)

chunks = python_splitter.split_text(python_code)
for i, chunk in enumerate(chunks):
    print(f"\n청크 {i+1}:\n{chunk}")
```

#### 2.5 최적 청킹 전략

```python
# 프로덕션 권장 설정
from langchain.text_splitter import RecursiveCharacterTextSplitter

def create_optimized_splitter(chunk_size=1000, chunk_overlap=200):
    """최적화된 텍스트 분할기 생성"""
    return RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        length_function=len,
        separators=[
            "\n\n",  # 문단
            "\n",    # 줄바꿈
            ". ",    # 문장
            "! ",
            "? ",
            ", ",    # 구문
            " ",     # 단어
            "",      # 문자
        ]
    )

# 문서 타입별 권장 크기
CHUNK_SIZES = {
    "general": {"size": 1000, "overlap": 200},      # 일반 문서
    "code": {"size": 500, "overlap": 50},           # 코드
    "chat": {"size": 500, "overlap": 100},          # 대화형
    "academic": {"size": 1500, "overlap": 300},     # 학술 논문
}

# 사용 예
splitter = create_optimized_splitter(**CHUNK_SIZES["general"])
chunks = splitter.split_documents(documents)
```

### 3. Embeddings: 벡터 임베딩 생성

#### 3.1 OpenAI Embeddings

```python
from langchain_openai import OpenAIEmbeddings

# 기본 설정
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",  # 또는 text-embedding-3-large
    # api_key="sk-..."  # 환경변수 OPENAI_API_KEY 사용
)

# 단일 텍스트 임베딩
text = "LangChain은 LLM 프레임워크입니다."
vector = embeddings.embed_query(text)
print(f"벡터 차원: {len(vector)}")
print(f"첫 5개 값: {vector[:5]}")

# 여러 텍스트 일괄 임베딩
texts = [
    "LangChain은 LLM 프레임워크입니다.",
    "RAG는 검색 증강 생성입니다.",
    "벡터 데이터베이스는 임베딩을 저장합니다."
]
vectors = embeddings.embed_documents(texts)
print(f"임베딩 수: {len(vectors)}")
```

**모델 비교:**
```python
# text-embedding-3-small
# - 차원: 1536
# - 비용: $0.00002/1K 토큰
# - 속도: 빠름
# - 용도: 일반적인 RAG

# text-embedding-3-large
# - 차원: 3072
# - 비용: $0.00013/1K 토큰
# - 정확도: 높음
# - 용도: 고정밀 검색
```

#### 3.2 Cohere Embeddings

```python
from langchain_cohere import CohereEmbeddings

embeddings = CohereEmbeddings(
    model="embed-multilingual-v3.0",
    # api_key="your-cohere-key"
)

# 한국어 지원 우수
vector = embeddings.embed_query("한국어 텍스트도 잘 처리합니다.")
print(f"벡터 차원: {len(vector)}")
```

#### 3.3 HuggingFace Embeddings (로컬)

```python
from langchain_community.embeddings import HuggingFaceEmbeddings

# 로컬 모델 사용 (API 비용 없음)
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    model_kwargs={'device': 'cpu'},  # 'cuda' for GPU
    encode_kwargs={'normalize_embeddings': True}
)

vector = embeddings.embed_query("로컬에서 실행됩니다.")
print(f"벡터 차원: {len(vector)}")  # 384
```

**추천 모델:**
```python
# 영어
"sentence-transformers/all-MiniLM-L6-v2"  # 384차원, 빠름
"sentence-transformers/all-mpnet-base-v2"  # 768차원, 정확

# 다국어 (한국어 포함)
"sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
"BAAI/bge-m3"  # 중국어 연구진, 한국어 우수
```

#### 3.4 Ollama Embeddings (로컬 LLM)

```python
from langchain_community.embeddings import OllamaEmbeddings

# Ollama 서버 필요: ollama pull nomic-embed-text
embeddings = OllamaEmbeddings(
    model="nomic-embed-text",
    base_url="http://localhost:11434"
)

vector = embeddings.embed_query("로컬 Ollama로 임베딩")
print(f"벡터 차원: {len(vector)}")  # 768
```

#### 3.5 임베딩 비용 최적화

```python
from functools import lru_cache
from langchain_openai import OpenAIEmbeddings

class CachedEmbeddings:
    """임베딩 결과 캐싱"""

    def __init__(self, embeddings_model):
        self.embeddings = embeddings_model
        self._cache = {}

    def embed_query(self, text: str):
        """쿼리 임베딩 (캐싱)"""
        if text not in self._cache:
            self._cache[text] = self.embeddings.embed_query(text)
        return self._cache[text]

    def embed_documents(self, texts: list[str]):
        """문서 임베딩 (배치 처리)"""
        return self.embeddings.embed_documents(texts)

# 사용
base_embeddings = OpenAIEmbeddings()
cached_embeddings = CachedEmbeddings(base_embeddings)

# 동일 텍스트 재사용 시 API 호출 없음
v1 = cached_embeddings.embed_query("같은 질문")
v2 = cached_embeddings.embed_query("같은 질문")  # 캐시에서 반환
```

### 4. Vector Stores: 벡터 데이터베이스 통합

#### 4.1 FAISS (로컬, 빠름)

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from langchain.schema import Document

# 문서 준비
docs = [
    Document(page_content="LangChain은 LLM 프레임워크입니다.", metadata={"id": 1}),
    Document(page_content="RAG는 검색 증강 생성입니다.", metadata={"id": 2}),
]

# 벡터 스토어 생성
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_documents(docs, embeddings)

# 검색
results = vectorstore.similarity_search("LLM 프레임워크", k=1)
print(f"결과: {results[0].page_content}")

# 점수와 함께 검색
results_with_scores = vectorstore.similarity_search_with_score("LLM", k=2)
for doc, score in results_with_scores:
    print(f"점수: {score:.4f} | 내용: {doc.page_content}")

# 저장 및 로드
vectorstore.save_local("faiss_index")
loaded_vectorstore = FAISS.load_local("faiss_index", embeddings)
```

#### 4.2 Chroma (로컬, 경량)

```python
from langchain_community.vectorstores import Chroma

# 벡터 스토어 생성
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    persist_directory="./chroma_db"  # 영구 저장
)

# 검색
results = vectorstore.similarity_search("RAG", k=2)

# 메타데이터 필터링
results = vectorstore.similarity_search(
    "프레임워크",
    k=2,
    filter={"id": 1}  # id=1인 문서만
)

# 컬렉션 삭제
vectorstore.delete_collection()
```

#### 4.3 Pinecone (클라우드, 프로덕션)

```python
from langchain_community.vectorstores import Pinecone
from pinecone import Pinecone as PineconeClient, ServerlessSpec

# Pinecone 초기화
pc = PineconeClient(api_key="your-api-key")

# 인덱스 생성 (처음 한 번만)
index_name = "langchain-rag"
if index_name not in pc.list_indexes().names():
    pc.create_index(
        name=index_name,
        dimension=1536,  # OpenAI embedding 차원
        metric="cosine",
        spec=ServerlessSpec(
            cloud="aws",
            region="us-east-1"
        )
    )

# 벡터 스토어 생성
vectorstore = Pinecone.from_documents(
    documents=docs,
    embedding=embeddings,
    index_name=index_name
)

# 검색
results = vectorstore.similarity_search("LangChain", k=3)
```

#### 4.4 MongoDB Atlas Vector Search

```python
from langchain_community.vectorstores import MongoDBAtlasVectorSearch
from pymongo import MongoClient

# MongoDB 연결
client = MongoClient("mongodb+srv://user:password@cluster.mongodb.net/")
collection = client["rag_database"]["documents"]

# 벡터 스토어 생성
vectorstore = MongoDBAtlasVectorSearch.from_documents(
    documents=docs,
    embedding=embeddings,
    collection=collection,
    index_name="vector_index"
)

# 메타데이터 필터와 함께 검색
results = vectorstore.similarity_search(
    "LLM",
    k=3,
    pre_filter={"metadata.category": "AI"}
)
```

#### 4.5 Supabase Vector Store

```python
from langchain_community.vectorstores import SupabaseVectorStore
from supabase import create_client

# Supabase 클라이언트
supabase_url = "https://your-project.supabase.co"
supabase_key = "your-anon-key"
supabase_client = create_client(supabase_url, supabase_key)

# 벡터 스토어
vectorstore = SupabaseVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    client=supabase_client,
    table_name="documents",
    query_name="match_documents"  # Supabase에서 생성한 함수 이름
)

# 검색
results = vectorstore.similarity_search("RAG", k=2)
```

### 5. Retrievers: 고급 검색 전략

#### 5.1 기본 Retriever

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

vectorstore = FAISS.from_documents(docs, OpenAIEmbeddings())

# Similarity Search (기본)
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}
)

results = retriever.invoke("LangChain")
print(f"검색 결과: {len(results)}개")

# MMR (Maximum Marginal Relevance) - 다양성 증가
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 5,
        "fetch_k": 20,    # 초기 후보 수
        "lambda_mult": 0.5  # 0=다양성, 1=유사성
    }
)

# 임계값 기반 검색
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "score_threshold": 0.7,  # 0.7 이상만 반환
        "k": 5
    }
)
```

#### 5.2 MultiQueryRetriever (질문 변형)

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
base_retriever = vectorstore.as_retriever()

# 질문을 여러 형태로 변형하여 검색
multi_retriever = MultiQueryRetriever.from_llm(
    retriever=base_retriever,
    llm=llm
)

# 질문 하나로 여러 변형된 쿼리 자동 생성
results = multi_retriever.invoke("LangChain의 장점은?")

# 내부적으로 생성된 질문들:
# - "LangChain의 장점은 무엇인가요?"
# - "LangChain을 사용하면 얻을 수 있는 이점은?"
# - "LangChain이 제공하는 혜택은?"
```

#### 5.3 ContextualCompressionRetriever (압축)

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
base_retriever = vectorstore.as_retriever()

# 검색 결과에서 관련 부분만 추출
compressor = LLMChainExtractor.from_llm(llm)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever
)

# 질문에 직접 관련된 부분만 반환
results = compression_retriever.invoke("LangChain의 주요 기능은?")
for doc in results:
    print(f"압축된 내용: {doc.page_content}")
```

#### 5.4 EnsembleRetriever (하이브리드 검색)

```python
from langchain.retrievers import EnsembleRetriever, BM25Retriever

# BM25 (키워드 검색)
bm25_retriever = BM25Retriever.from_documents(docs)
bm25_retriever.k = 3

# 벡터 검색
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 앙상블 (하이브리드)
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]  # BM25: 40%, Vector: 60%
)

results = ensemble_retriever.invoke("LangChain RAG")
```

#### 5.5 SelfQueryRetriever (메타데이터 자동 필터)

```python
from langchain.retrievers.self_query.base import SelfQueryRetriever
from langchain.chains.query_constructor.base import AttributeInfo

# 메타데이터 정의
metadata_field_info = [
    AttributeInfo(
        name="source",
        description="문서 출처 (파일명)",
        type="string",
    ),
    AttributeInfo(
        name="page",
        description="페이지 번호",
        type="integer",
    ),
    AttributeInfo(
        name="category",
        description="문서 카테고리 (AI, Tech, Business 등)",
        type="string",
    ),
]

document_content_description = "LangChain 기술 문서"

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Self-Query Retriever
retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=metadata_field_info,
    verbose=True
)

# 자연어로 쿼리 (자동으로 메타데이터 필터 생성)
results = retriever.invoke("AI 카테고리의 LangChain 문서를 찾아줘")
# 내부적으로 filter={"category": "AI"} 자동 생성
```

---

## 🔗 벡터 스토어 통합

### FAISS와 RAG 구축

```python
# faiss_rag.py
from langchain_community.vectorstores import FAISS
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import RetrievalQA

# 1. 문서 로딩
loader = PyPDFLoader("documents/guide.pdf")
pages = loader.load()

# 2. 청킹
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)
splits = text_splitter.split_documents(pages)

# 3. FAISS 벡터 스토어 생성
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(splits, embeddings)

# 4. 저장 (재사용 가능)
vectorstore.save_local("faiss_index")

# 5. 로드
loaded_vectorstore = FAISS.load_local(
    "faiss_index",
    embeddings,
    allow_dangerous_deserialization=True  # 로컬 파일 신뢰
)

# 6. RAG Chain
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=loaded_vectorstore.as_retriever(search_kwargs={"k": 3}),
    return_source_documents=True
)

# 7. 질문
result = qa_chain.invoke({"query": "가이드의 주요 내용은?"})
print(result['result'])
```

### Chroma와 RAG 구축

```python
# chroma_rag.py
from langchain_community.vectorstores import Chroma
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.schema import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# 문서 준비
docs = [
    Document(
        page_content="LangChain은 2022년 10월에 출시된 LLM 프레임워크입니다.",
        metadata={"source": "intro.txt", "category": "Framework"}
    ),
    Document(
        page_content="RAG는 검색을 통해 LLM의 답변을 향상시키는 기술입니다.",
        metadata={"source": "rag.txt", "category": "Technique"}
    ),
]

# Chroma 벡터 스토어 (영구 저장)
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    persist_directory="./chroma_db"
)

# 기존 DB 로드
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=embeddings
)

# Retriever
retriever = vectorstore.as_retriever(
    search_type="mmr",  # MMR로 다양성 증가
    search_kwargs={"k": 3, "fetch_k": 10}
)

# LCEL Chain
template = """다음 컨텍스트를 사용하여 질문에 답변하세요:

{context}

질문: {question}

답변:"""

prompt = ChatPromptTemplate.from_template(template)
llm = ChatOpenAI(model="gpt-4o-mini")

rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 스트리밍 실행
for chunk in rag_chain.stream("LangChain이란?"):
    print(chunk, end="", flush=True)
```

### Pinecone과 RAG 구축 (프로덕션)

```python
# pinecone_rag.py
import os
from pinecone import Pinecone as PineconeClient, ServerlessSpec
from langchain_community.vectorstores import Pinecone
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.document_loaders import DirectoryLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferMemory

# Pinecone 초기화
pc = PineconeClient(api_key=os.getenv("PINECONE_API_KEY"))
index_name = "production-rag"

# 인덱스 생성 (처음 한 번)
if index_name not in pc.list_indexes().names():
    pc.create_index(
        name=index_name,
        dimension=1536,
        metric="cosine",
        spec=ServerlessSpec(cloud="aws", region="us-east-1")
    )

# 문서 로딩
loader = DirectoryLoader(
    "knowledge_base/",
    glob="**/*.txt",
    loader_cls=TextLoader
)
docs = loader.load()

# 청킹
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)
splits = text_splitter.split_documents(docs)

# Pinecone에 업로드
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Pinecone.from_documents(
    documents=splits,
    embedding=embeddings,
    index_name=index_name
)

# 대화형 RAG Chain
llm = ChatOpenAI(model="gpt-4o", temperature=0)
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True,
    output_key="answer"
)

qa_chain = ConversationalRetrievalChain.from_llm(
    llm=llm,
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    memory=memory,
    return_source_documents=True
)

# 대화
while True:
    question = input("\n질문: ")
    if question.lower() in ["quit", "exit", "종료"]:
        break

    result = qa_chain.invoke({"question": question})
    print(f"\n답변: {result['answer']}")
    print(f"출처: {len(result['source_documents'])}개 문서")
```

---

## 🚀 고급 RAG 패턴

### 1. Conversational RAG (대화 기록 유지)

```python
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferMemory
from langchain_openai import ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

# 벡터 스토어 준비
vectorstore = FAISS.from_documents(documents, OpenAIEmbeddings())

# 메모리 설정
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True,
    output_key="answer"
)

# 대화형 체인
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)
qa_chain = ConversationalRetrievalChain.from_llm(
    llm=llm,
    retriever=vectorstore.as_retriever(),
    memory=memory,
    return_source_documents=True
)

# 대화
print(qa_chain.invoke({"question": "LangChain이란?"})['answer'])
print(qa_chain.invoke({"question": "그것의 주요 기능은?"})['answer'])  # "그것" = LangChain (컨텍스트 유지)
```

### 2. Parent Document Retriever (계층적 검색)

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma

# 부모 문서 저장소
parent_store = InMemoryStore()

# 작은 청크로 검색, 큰 청크 반환
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1000)

vectorstore = Chroma(
    collection_name="split_parents",
    embedding_function=OpenAIEmbeddings()
)

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=parent_store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# 문서 추가
retriever.add_documents(documents)

# 작은 청크로 검색하지만 전체 컨텍스트 반환
results = retriever.invoke("LangChain")
```

### 3. Reranking (재순위화)

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain_cohere import CohereRerank
from langchain_community.vectorstores import FAISS

# 기본 retriever
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# Cohere Reranker
compressor = CohereRerank(
    model="rerank-english-v2.0",
    top_n=5  # 상위 5개만 반환
)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever
)

# 검색 후 재순위화
results = compression_retriever.invoke("가장 관련있는 문서")
```

### 4. HyDE (Hypothetical Document Embeddings)

```python
from langchain.chains import HypotheticalDocumentEmbedder
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

# LLM으로 가상 문서 생성 후 검색
llm = ChatOpenAI(model="gpt-4o-mini")
base_embeddings = OpenAIEmbeddings()

hyde_embeddings = HypotheticalDocumentEmbedder.from_llm(
    llm=llm,
    base_embeddings=base_embeddings,
    prompt_key="web_search"  # 템플릿 선택
)

# 벡터 스토어에 적용
vectorstore = FAISS.from_documents(documents, hyde_embeddings)

# "LangChain의 장점"으로 검색 시:
# 1. LLM이 가상의 답변 생성
# 2. 가상 답변을 임베딩
# 3. 유사 문서 검색
results = vectorstore.similarity_search("LangChain의 장점")
```

### 5. Multi-Vector Retriever (다중 벡터)

```python
from langchain.retrievers.multi_vector import MultiVectorRetriever
from langchain.storage import InMemoryByteStore
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 문서 요약 생성 후 검색
llm = ChatOpenAI(model="gpt-4o-mini")
vectorstore = Chroma(collection_name="summaries", embedding_function=OpenAIEmbeddings())
byte_store = InMemoryByteStore()

retriever = MultiVectorRetriever(
    vectorstore=vectorstore,
    byte_store=byte_store,
    id_key="doc_id"
)

# 요약 생성 체인
summary_prompt = ChatPromptTemplate.from_template(
    "다음 문서를 한 문장으로 요약하세요:\n\n{doc}"
)
summary_chain = summary_prompt | llm | StrOutputParser()

# 요약 생성 및 저장
import uuid
doc_ids = [str(uuid.uuid4()) for _ in documents]

summaries = []
for doc in documents:
    summary = summary_chain.invoke({"doc": doc.page_content})
    summaries.append(Document(page_content=summary, metadata={"doc_id": doc_ids[len(summaries)]}))

# 요약으로 검색, 원본 문서 반환
retriever.vectorstore.add_documents(summaries)
retriever.docstore.mset(list(zip(doc_ids, documents)))

results = retriever.invoke("LangChain")  # 요약으로 검색, 원본 반환
```

---

## ⚡ 성능 최적화

### 1. 배치 임베딩

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# ❌ 나쁜 예: 루프로 하나씩
vectors = []
for text in texts:
    vector = embeddings.embed_query(text)  # 100번 API 호출
    vectors.append(vector)

# ✅ 좋은 예: 배치 처리
vectors = embeddings.embed_documents(texts)  # 1번 API 호출
```

### 2. 임베딩 캐싱

```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore
from langchain_openai import OpenAIEmbeddings

# 로컬 캐시 저장소
store = LocalFileStore("./embedding_cache")

# 캐시 래퍼
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=OpenAIEmbeddings(),
    document_embedding_cache=store,
    namespace="openai_embeddings"
)

# 동일 텍스트는 캐시에서 반환 (API 호출 없음)
vector1 = cached_embeddings.embed_query("같은 텍스트")
vector2 = cached_embeddings.embed_query("같은 텍스트")  # 캐시 hit
```

### 3. 청크 크기 최적화

```python
# 프로덕션 권장 설정
from langchain.text_splitter import RecursiveCharacterTextSplitter

# GPT-4o-mini (128K context)
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,      # 적당한 크기
    chunk_overlap=200,    # 20% 오버랩
    length_function=len,
)

# 더 정밀하게: 토큰 수 기준
from langchain.text_splitter import TokenTextSplitter

token_splitter = TokenTextSplitter(
    encoding_name="cl100k_base",  # GPT-4 인코딩
    chunk_size=500,               # 500 토큰
    chunk_overlap=50
)
```

### 4. 인덱스 최적화

```python
# FAISS 인덱스 타입 선택
from langchain_community.vectorstores import FAISS
import faiss

# ✅ 소규모 (<100K 벡터): Flat (정확도 100%)
index = faiss.IndexFlatL2(1536)

# ✅ 중규모 (100K-1M): IVFFlat (빠름, 정확도 95%+)
quantizer = faiss.IndexFlatL2(1536)
index = faiss.IndexIVFFlat(quantizer, 1536, 100)  # 100개 클러스터

# ✅ 대규모 (1M+): HNSW (매우 빠름, 정확도 99%+)
index = faiss.IndexHNSWFlat(1536, 32)  # M=32

# LangChain에서 사용
vectorstore = FAISS.from_documents(
    documents,
    embeddings,
    index=index
)
```

### 5. 스트리밍 응답

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough

llm = ChatOpenAI(model="gpt-4o-mini", streaming=True)

# LCEL 체인
chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
)

# 스트리밍 출력
for chunk in chain.stream("LangChain이란?"):
    print(chunk.content, end="", flush=True)
```

### 6. LLM 모델 선택

```python
# 비용 vs 성능 트레이드오프

# ✅ 빠르고 저렴 (일반 RAG)
llm = ChatOpenAI(model="gpt-4o-mini")

# ✅ 균형 (복잡한 RAG)
llm = ChatOpenAI(model="gpt-4o")

# ✅ 무료 (로컬)
from langchain_community.llms import Ollama
llm = Ollama(model="llama3.1:8b")

# ✅ 한국어 최적화
from langchain_anthropic import ChatAnthropic
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
```

---

## 🌐 프로덕션 배포

### 1. FastAPI 서버 구축

```python
# app.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain.chains import RetrievalQA
import uvicorn

app = FastAPI(title="LangChain RAG API")

# 전역 변수 (서버 시작 시 로드)
vectorstore = None
qa_chain = None

class QueryRequest(BaseModel):
    question: str
    k: int = 3

class QueryResponse(BaseModel):
    answer: str
    sources: list[dict]

@app.on_event("startup")
async def startup_event():
    """서버 시작 시 벡터 스토어 로드"""
    global vectorstore, qa_chain

    print("Loading vector store...")
    embeddings = OpenAIEmbeddings()
    vectorstore = FAISS.load_local(
        "faiss_index",
        embeddings,
        allow_dangerous_deserialization=True
    )

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    qa_chain = RetrievalQA.from_chain_type(
        llm=llm,
        retriever=vectorstore.as_retriever(),
        return_source_documents=True
    )
    print("Server ready!")

@app.post("/query", response_model=QueryResponse)
async def query(request: QueryRequest):
    """RAG 질의 엔드포인트"""
    try:
        result = qa_chain.invoke({"query": request.question})

        return QueryResponse(
            answer=result['result'],
            sources=[
                {
                    "content": doc.page_content[:200],
                    "metadata": doc.metadata
                }
                for doc in result['source_documents']
            ]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    """헬스 체크"""
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**실행:**
```bash
pip install fastapi uvicorn
python app.py
```

**테스트:**
```bash
curl -X POST "http://localhost:8000/query" \
  -H "Content-Type: application/json" \
  -d '{"question": "LangChain이란?"}'
```

### 2. Docker 컨테이너화

```dockerfile
# Dockerfile
FROM python:3.10-slim

WORKDIR /app

# 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 앱 복사
COPY . .

# 포트 노출
EXPOSE 8000

# 실행
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

**빌드 및 실행:**
```bash
docker build -t langchain-rag .
docker run -p 8000:8000 -e OPENAI_API_KEY=sk-... langchain-rag
```

### 3. LangSmith 모니터링

```python
# .env에 추가
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your-langsmith-key
LANGCHAIN_PROJECT=production-rag

# 코드에서 자동 추적
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
# 모든 호출이 LangSmith에 자동 기록됨

# https://smith.langchain.com 에서 확인:
# - 체인 실행 추적
# - 토큰 사용량
# - 지연 시간
# - 오류 로그
```

### 4. 에러 핸들링

```python
from langchain.callbacks import get_openai_callback
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def query_with_retry(question: str):
    """재시도 로직이 있는 쿼리"""
    try:
        with get_openai_callback() as cb:
            result = qa_chain.invoke({"query": question})

            print(f"토큰 사용: {cb.total_tokens}")
            print(f"비용: ${cb.total_cost:.4f}")

            return result
    except Exception as e:
        print(f"오류 발생: {e}")
        raise

# 사용
result = query_with_retry("LangChain이란?")
```

---

## 💼 실무 가이드

### 1. PDF 문서 기반 RAG 시스템 (완전 예제)

```python
# production_rag.py
import os
from pathlib import Path
from dotenv import load_dotenv

from langchain_community.document_loaders import PyPDFLoader, DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferWindowMemory
from langchain.callbacks import get_openai_callback

load_dotenv()

class ProductionRAG:
    """프로덕션급 RAG 시스템"""

    def __init__(self, docs_path: str, index_path: str = "faiss_index"):
        self.docs_path = docs_path
        self.index_path = index_path
        self.vectorstore = None
        self.qa_chain = None

        # LLM 설정
        self.llm = ChatOpenAI(
            model="gpt-4o-mini",
            temperature=0,
            max_tokens=1000
        )

        # 임베딩 설정
        self.embeddings = OpenAIEmbeddings(
            model="text-embedding-3-small"
        )

    def load_documents(self):
        """PDF 문서 로딩"""
        print(f"📂 Loading documents from {self.docs_path}...")

        loader = DirectoryLoader(
            self.docs_path,
            glob="**/*.pdf",
            loader_cls=PyPDFLoader,
            show_progress=True
        )
        documents = loader.load()

        print(f"✅ Loaded {len(documents)} pages")
        return documents

    def split_documents(self, documents):
        """문서 청킹"""
        print("✂️  Splitting documents...")

        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200,
            length_function=len,
        )
        splits = text_splitter.split_documents(documents)

        print(f"✅ Created {len(splits)} chunks")
        return splits

    def create_vectorstore(self, splits):
        """벡터 스토어 생성"""
        print("🔄 Creating vector store...")

        self.vectorstore = FAISS.from_documents(
            splits,
            self.embeddings
        )

        # 저장
        self.vectorstore.save_local(self.index_path)
        print(f"✅ Vector store saved to {self.index_path}")

    def load_vectorstore(self):
        """기존 벡터 스토어 로드"""
        print(f"📥 Loading vector store from {self.index_path}...")

        self.vectorstore = FAISS.load_local(
            self.index_path,
            self.embeddings,
            allow_dangerous_deserialization=True
        )
        print("✅ Vector store loaded")

    def setup_chain(self):
        """RAG 체인 설정"""
        print("⛓️  Setting up QA chain...")

        # 메모리 (최근 5개 대화 기억)
        memory = ConversationBufferWindowMemory(
            k=5,
            memory_key="chat_history",
            return_messages=True,
            output_key="answer"
        )

        # 체인 생성
        self.qa_chain = ConversationalRetrievalChain.from_llm(
            llm=self.llm,
            retriever=self.vectorstore.as_retriever(
                search_type="mmr",
                search_kwargs={"k": 5, "fetch_k": 20}
            ),
            memory=memory,
            return_source_documents=True,
            verbose=False
        )
        print("✅ QA chain ready")

    def initialize(self, force_rebuild=False):
        """시스템 초기화"""
        if force_rebuild or not Path(self.index_path).exists():
            # 새로 빌드
            documents = self.load_documents()
            splits = self.split_documents(documents)
            self.create_vectorstore(splits)
        else:
            # 기존 인덱스 로드
            self.load_vectorstore()

        self.setup_chain()
        print("\n🎉 RAG system initialized!\n")

    def query(self, question: str) -> dict:
        """질문 처리"""
        with get_openai_callback() as cb:
            result = self.qa_chain.invoke({"question": question})

            return {
                "answer": result["answer"],
                "sources": result["source_documents"],
                "tokens": cb.total_tokens,
                "cost": cb.total_cost
            }

    def chat(self):
        """대화형 인터페이스"""
        print("=" * 70)
        print("  LangChain RAG 시스템")
        print("=" * 70)
        print("종료: 'quit', 'exit', '종료'")
        print("-" * 70 + "\n")

        while True:
            question = input("❓ 질문: ").strip()

            if question.lower() in ["quit", "exit", "종료", ""]:
                print("\n👋 종료합니다.")
                break

            try:
                result = self.query(question)

                print(f"\n💡 답변:\n{result['answer']}\n")
                print(f"📚 참고 문서: {len(result['sources'])}개")
                print(f"🪙 토큰: {result['tokens']} | 비용: ${result['cost']:.4f}\n")
                print("-" * 70 + "\n")

            except Exception as e:
                print(f"\n❌ 오류: {e}\n")

# 실행
if __name__ == "__main__":
    # 1. RAG 시스템 생성
    rag = ProductionRAG(
        docs_path="./documents",
        index_path="./faiss_index"
    )

    # 2. 초기화 (force_rebuild=True로 인덱스 재생성)
    rag.initialize(force_rebuild=False)

    # 3. 대화 시작
    rag.chat()
```

**디렉토리 구조:**
```
project/
├── production_rag.py
├── documents/           # PDF 문서 저장
│   ├── guide1.pdf
│   ├── guide2.pdf
│   └── manual.pdf
├── faiss_index/         # 생성될 벡터 인덱스
├── .env
└── requirements.txt
```

### 2. 웹 크롤링 + RAG

```python
# web_rag.py
from langchain_community.document_loaders import WebBaseLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# 웹사이트 크롤링
urls = [
    "https://python.langchain.com/docs/introduction/",
    "https://python.langchain.com/docs/get_started/quickstart/",
    "https://python.langchain.com/docs/concepts/",
]

loader = WebBaseLoader(urls)
docs = loader.load()

# 청킹
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)
splits = text_splitter.split_documents(docs)

# 벡터 스토어
vectorstore = Chroma.from_documents(
    splits,
    OpenAIEmbeddings(),
    persist_directory="./chroma_web"
)

# RAG Chain
template = """다음 LangChain 공식 문서를 참고하여 답변하세요:

{context}

질문: {question}

답변:"""

prompt = ChatPromptTemplate.from_template(template)
llm = ChatOpenAI(model="gpt-4o-mini")

rag_chain = (
    {"context": vectorstore.as_retriever(), "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 질문
response = rag_chain.invoke("LangChain의 주요 컨셉은?")
print(response)
```

### 3. 다국어 RAG (한국어 최적화)

```python
# multilingual_rag.py
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_openai import ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain.chains import RetrievalQA

# 한국어 임베딩 모델
embeddings = HuggingFaceEmbeddings(
    model_name="jhgan/ko-sroberta-multitask",  # 한국어 최적화
    model_kwargs={'device': 'cpu'},
    encode_kwargs={'normalize_embeddings': True}
)

# 한국어 문서
korean_docs = [
    Document(page_content="랭체인은 LLM 애플리케이션 프레임워크입니다."),
    Document(page_content="RAG는 검색 증강 생성 기술입니다."),
]

# 벡터 스토어
vectorstore = FAISS.from_documents(korean_docs, embeddings)

# LLM (Claude는 한국어 우수)
from langchain_anthropic import ChatAnthropic
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")

# RAG Chain
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(),
    return_source_documents=True
)

# 한국어 질문
result = qa_chain.invoke({"query": "랭체인이 뭐야?"})
print(result['result'])
```

### 4. SQL 데이터베이스 + RAG

```python
# sql_rag.py
from langchain_community.utilities import SQLDatabase
from langchain.chains import create_sql_query_chain
from langchain_openai import ChatOpenAI
from langchain_community.tools.sql_database.tool import QuerySQLDataBaseTool
from operator import itemgetter
from langchain_core.runnables import RunnablePassthrough

# DB 연결
db = SQLDatabase.from_uri("sqlite:///company.db")

llm = ChatOpenAI(model="gpt-4o-mini")

# SQL 쿼리 생성 체인
query_chain = create_sql_query_chain(llm, db)

# SQL 실행 도구
execute_query = QuerySQLDataBaseTool(db=db)

# Chain 구성
chain = (
    RunnablePassthrough.assign(query=query_chain)
    | {
        "query": itemgetter("query"),
        "result": itemgetter("query") | execute_query
    }
)

# 자연어 질문
response = chain.invoke({"question": "2024년 매출 상위 5개 제품은?"})
print(f"SQL: {response['query']}")
print(f"결과: {response['result']}")
```

---

## 🐛 트러블슈팅

### 1. "Module not found" 에러

```bash
# 패키지 재설치
pip uninstall langchain langchain-community langchain-openai
pip install langchain langchain-community langchain-openai

# 가상 환경 확인
which python  # 가상 환경 경로 확인
```

### 2. OpenAI API 키 오류

```python
# .env 파일 확인
OPENAI_API_KEY=sk-your-actual-key  # 따옴표 없음

# 환경 변수 로드 확인
from dotenv import load_dotenv
import os

load_dotenv()
print(os.getenv("OPENAI_API_KEY"))  # None이면 .env 경로 문제
```

### 3. FAISS "allow_dangerous_deserialization" 에러

```python
# FAISS 로드 시 필수 파라미터
vectorstore = FAISS.load_local(
    "faiss_index",
    embeddings,
    allow_dangerous_deserialization=True  # 로컬 파일 신뢰
)
```

### 4. 메모리 부족 (대용량 문서)

```python
# 배치 처리
from langchain_community.vectorstores import FAISS

batch_size = 100
for i in range(0, len(splits), batch_size):
    batch = splits[i:i+batch_size]

    if i == 0:
        vectorstore = FAISS.from_documents(batch, embeddings)
    else:
        batch_vs = FAISS.from_documents(batch, embeddings)
        vectorstore.merge_from(batch_vs)

vectorstore.save_local("faiss_index")
```

### 5. 토큰 제한 초과

```python
# 청크 크기 줄이기
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,  # 기본 1000에서 줄임
    chunk_overlap=50
)

# 검색 결과 수 줄이기
retriever = vectorstore.as_retriever(
    search_kwargs={"k": 2}  # 기본 4에서 줄임
)
```

### 6. 느린 검색 속도

```python
# FAISS 인덱스 최적화
import faiss

# HNSW 인덱스 사용
index = faiss.IndexHNSWFlat(1536, 32)
vectorstore = FAISS.from_documents(
    documents,
    embeddings,
    index=index
)

# GPU 사용 (CUDA 설치 필요)
# pip install faiss-gpu
```

---

## ❓ FAQ

### Q1: LangChain vs LlamaIndex, 어떤 것을 선택해야 하나요?

**A:**
- **LangChain**: 범용 LLM 애플리케이션 (챗봇, 에이전트, 워크플로우)
- **LlamaIndex**: RAG 특화 (문서 인덱싱, 검색 최적화)

RAG만 필요하면 LlamaIndex, 다양한 기능이 필요하면 LangChain 추천.

### Q2: 어떤 벡터 스토어를 선택해야 하나요?

**A:**
- **개발/테스트**: FAISS (로컬, 무료)
- **소규모 프로덕션**: Chroma (관리 편함)
- **대규모 프로덕션**: Pinecone, Weaviate (완전 관리형)
- **기존 DB 활용**: MongoDB Atlas, Supabase

### Q3: 임베딩 모델 선택 기준은?

**A:**
- **영어 중심**: OpenAI text-embedding-3-small
- **다국어 (한국어)**: Cohere embed-multilingual-v3.0
- **비용 절감**: HuggingFace (로컬), Ollama
- **고정밀**: OpenAI text-embedding-3-large

### Q4: 청크 크기는 얼마가 적당한가요?

**A:**
- **일반 문서**: 1000-1500자, 오버랩 200-300자
- **코드**: 500-800자, 오버랩 50-100자
- **대화형**: 500자, 오버랩 100자

실험을 통해 도메인별로 최적화하세요.

### Q5: LangChain 학습 리소스는?

**A:**
- 공식 문서: https://python.langchain.com
- LangChain 블로그: https://blog.langchain.dev
- YouTube: LangChain 공식 채널
- 한국 커뮤니티: LangChain Korea (디스코드)

---

## 📚 참고 자료

### 공식 문서
- [LangChain 공식 문서](https://python.langchain.com/)
- [LangChain API Reference](https://api.python.langchain.com/)
- [LangSmith](https://smith.langchain.com/)
- [LangChain GitHub](https://github.com/langchain-ai/langchain)

### 튜토리얼
- [LangChain Quickstart](https://python.langchain.com/docs/get_started/quickstart/)
- [RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/)
- [LangChain Academy (무료 코스)](https://academy.langchain.com/)

### 커뮤니티
- [LangChain Discord](https://discord.gg/langchain)
- [LangChain Twitter](https://twitter.com/langchainai)

### 관련 프로젝트
- [LangGraph](https://github.com/langchain-ai/langgraph): 멀티 에이전트 워크플로우
- [LangServe](https://github.com/langchain-ai/langserve): LangChain 체인 배포
- [LangChain Templates](https://github.com/langchain-ai/langchain/tree/master/templates): 즉시 사용 가능한 템플릿

### 한국어 자료
- [LangChain 한국 사용자 모임](https://www.facebook.com/groups/langchainkr)
- [랭체인 공식 문서 한글 번역](https://wikidocs.net/book/14314)

---

## 📝 라이선스

LangChain 프레임워크는 MIT 라이선스를 따릅니다.

---

**🎉 이제 LangChain으로 강력한 RAG 시스템을 구축할 준비가 되었습니다!**
