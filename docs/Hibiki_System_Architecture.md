# Hibiki - Multi-Tenant RAG System

**High-Level Solution Architecture**

*Version 1.0 — December 2025*

---

## 1. What is Hibiki?

Hibiki is a **multi-tenant Retrieval-Augmented Generation (RAG) system** that allows multiple client applications to leverage Large Language Models (LLMs) with their own private knowledge bases.

### Key Capabilities

- **Knowledge Retrieval**: Client applications query their private document collections via API tokens
- **Multi-Tenancy**: Complete data isolation between different client applications
- **Document Management**: Web-based portal for uploading and managing documents
- **Future Extensions**: Report generation using SQL, advanced analytics, and more

---

## 2. System Components

Hibiki consists of **three core services** plus supporting infrastructure:

### Core Services

```
┌─────────────────────────────────────────────────────────────────┐
│                         HIBIKI SYSTEM                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐   ┌──────────────────┐   ┌────────────┐   │
│  │ hibiki-console   │   │ hibiki-ingest    │   │ hibiki-    │   │
│  │ (Management      │   │ (ETL Pipeline)   │   │ engine     │   │
│  │  Portal)         │   │                  │   │ (RAG API)  │   │
│  └──────────────────┘   └──────────────────┘   └────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1 hibiki-console (Document Management Portal)

**Technology**: Next.js 15 (App Router), TypeScript, PostgreSQL, MinIO

**Purpose**: Web application where administrators and maintainers manage the RAG system

**Responsibilities**:
- **User Management**: Admin creates maintainer accounts
- **Application Management**: Create and configure client applications
- **Document Upload**: Maintainers upload PDFs, DOCX, TXT, MD files
- **Token Generation**: Create API tokens for client applications to access the RAG engine
- **Configuration**: Set system prompts and RAG parameters per client application

**Key Users**:
- **Admin**: Creates maintainer accounts and client applications
- **Maintainer**: Manages documents and tokens for assigned applications

**Data Storage**:
- PostgreSQL: User accounts, application metadata, document metadata, tokens
- MinIO (S3-compatible): Actual document files

---

### 2.2 hibiki-ingest (Ingestion Pipeline)

**Technology**: Python, OCR libraries, Vector database clients

**Purpose**: Background ETL service that transforms uploaded documents into searchable embeddings

**Responsibilities**:
- Monitor job queue for new document uploads
- Extract text from documents (including OCR for image-based PDFs)
- Split documents into chunks
- Generate embeddings using embedding models
- Store embeddings in client-specific vector stores
- Update job status and document processing status

**Processing Flow**:
```
Document Upload → Job Queue → hibiki-ingest → Vector Store
                                      ↓
                            Update Status in DB
```

---

### 2.3 hibiki-engine (RAG Engine)

**Technology**: Python FastAPI, Ollama (self-hosted LLM), Vector database

**Purpose**: API server that handles LLM queries with retrieval-augmented generation

**Responsibilities**:
- **Token Authentication**: Validate API tokens from client applications
- **Vector Search**: Retrieve relevant document chunks from client's vector store
- **LLM Inference**: Query Ollama with context from retrieved documents
- **Response Generation**: Return AI-generated answers based on client's knowledge base

**Request Flow**:
```
Client App → [Token] → hibiki-engine → Validate Token
                              ↓
                       Identify Client Application
                              ↓
                       Retrieve from Vector Store
                              ↓
                       Query Ollama LLM
                              ↓
                       Return Response
```

**Future Extensions**:
- Report generation using SQL queries
- Advanced analytics and insights
- Multi-modal retrieval (images, tables)

---

## 3. Supporting Infrastructure

### 3.1 PostgreSQL Database

**Used By**: hibiki-console, hibiki-ingest

**Stores**:
- User accounts and roles
- Client applications
- Document metadata
- Access tokens (hashed)
- Job queue entries
- Processing status

---

### 3.2 MinIO (S3-Compatible Storage)

**Used By**: hibiki-console, hibiki-ingest

**Stores**:
- Uploaded document files
- Configuration files per application

**Structure**:
```
bucket: rag-documents
├── {applicationId}/
│   ├── documents/
│   │   ├── {documentId}_{filename}.pdf
│   │   └── ...
│   └── _config.json
```

---

### 3.3 Vector Stores

**Used By**: hibiki-ingest (write), hibiki-engine (read)

**Purpose**: Store document embeddings for semantic search

**Multi-Tenancy**: One dedicated vector store per client application for complete data isolation

**Possible Technologies**:
- Chroma
- Qdrant
- Weaviate
- Pinecone
- Milvus

---

### 3.4 Job Queue

**Used By**: hibiki-console (producer), hibiki-ingest (consumer)

**Purpose**: Asynchronous communication between console and ingestion pipeline

**Technology Options**:
- Redis with Bull/BullMQ
- PostgreSQL with pg-boss

**Job Types**:
- `DOCUMENT_INGEST`: New document uploaded, needs processing
- `DOCUMENT_DELETE`: Document deleted, remove from vector store (future)
- `CONFIG_UPDATE`: Configuration changed, refresh settings (future)

---

## 4. Multi-Tenancy Architecture

### Isolation Model

Each client application operates in **complete isolation**:

```
Client App A          Client App B          Client App C
     │                     │                     │
     ├─ Token A            ├─ Token B            ├─ Token C
     ├─ Workspace A        ├─ Workspace B        ├─ Workspace C
     ├─ Vector Store A     ├─ Vector Store B     ├─ Vector Store C
     └─ Config A           └─ Config B           └─ Config C
```

### Access Control

1. **Console Access**: Maintainers can only access their assigned applications
2. **API Access**: Tokens are scoped to a single client application
3. **Data Access**: Vector stores are isolated; no cross-application queries

---

## 5. Complete Data Flow

### Document Upload Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Maintainer uploads PDF via hibiki-console                       │
│    ↓                                                                │
│ 2. hibiki-console saves file to MinIO                              │
│    ↓                                                                │
│ 3. hibiki-console creates document record in PostgreSQL            │
│    ↓                                                                │
│ 4. hibiki-console enqueues DOCUMENT_INGEST job                     │
│    ↓                                                                │
│ 5. hibiki-ingest picks up job from queue                           │
│    ↓                                                                │
│ 6. hibiki-ingest downloads file from MinIO                         │
│    ↓                                                                │
│ 7. hibiki-ingest extracts text (with OCR if needed)                │
│    ↓                                                                │
│ 8. hibiki-ingest generates embeddings                              │
│    ↓                                                                │
│ 9. hibiki-ingest stores embeddings in vector store                 │
│    ↓                                                                │
│ 10. hibiki-ingest updates document status to INDEXED               │
└─────────────────────────────────────────────────────────────────────┘
```

### Query Flow (RAG)

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Client App sends query with token to hibiki-engine              │
│    ↓                                                                │
│ 2. hibiki-engine validates token against PostgreSQL                │
│    ↓                                                                │
│ 3. hibiki-engine identifies client application from token          │
│    ↓                                                                │
│ 4. hibiki-engine retrieves system prompt from config               │
│    ↓                                                                │
│ 5. hibiki-engine searches client's vector store                    │
│    ↓                                                                │
│ 6. hibiki-engine assembles prompt with retrieved context           │
│    ↓                                                                │
│ 7. hibiki-engine queries Ollama LLM                                │
│    ↓                                                                │
│ 8. hibiki-engine returns AI-generated response to client           │
│    ↓                                                                │
│ 9. hibiki-engine updates token last_used_at timestamp              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Deployment Architecture

### Development Setup

```
Developer Machine
├── hibiki-console (Next.js on :3000)
├── PostgreSQL (:5432)
├── MinIO (:9000)
├── Redis (:6379)
├── hibiki-ingest (Python service)
├── hibiki-engine (FastAPI on :8000)
├── Ollama (:11434)
└── Vector Store (varies)
```

### Production Setup (Suggested)

```
┌─────────────────────────────────────────────────────────────────┐
│                         Load Balancer                           │
└─────────────────────────────────────────────────────────────────┘
                 │                           │
                 ▼                           ▼
    ┌────────────────────┐      ┌────────────────────┐
    │ hibiki-console     │      │ hibiki-engine      │
    │ (Next.js)          │      │ (FastAPI)          │
    │ Multiple instances │      │ Multiple instances │
    └────────────────────┘      └────────────────────┘
                 │                           │
                 └───────────┬───────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
    ┌──────────────┐  ┌──────────┐  ┌────────────────┐
    │ PostgreSQL   │  │ MinIO    │  │ Vector Stores  │
    │ (HA cluster) │  │ (cluster)│  │ (per client)   │
    └──────────────┘  └──────────┘  └────────────────┘
            │
            ▼
    ┌──────────────┐
    │ Redis/Queue  │
    └──────────────┘
            │
            ▼
    ┌────────────────────┐
    │ hibiki-ingest      │
    │ (Worker pool)      │
    └────────────────────┘
            │
            ▼
    ┌────────────────────┐
    │ Ollama             │
    │ (GPU instances)    │
    └────────────────────┘
```

---

## 7. Technology Stack Summary

| Component | Technology Stack |
|-----------|-----------------|
| **hibiki-console** | Next.js 15, TypeScript, TypeORM, Iron Session, Zod, Mantine UI |
| **hibiki-ingest** | Python, OCR libraries, LangChain/LlamaIndex (optional), Vector DB clients |
| **hibiki-engine** | Python, FastAPI, Ollama, Vector DB clients |
| **Database** | PostgreSQL 15+ |
| **Storage** | MinIO (S3-compatible) |
| **Queue** | Redis or pg-boss |
| **Vector Store** | TBD (Chroma, Qdrant, Weaviate, etc.) |
| **LLM Runtime** | Ollama (self-hosted) |

---

## 8. Security Model

### Authentication & Authorization

1. **Console Users**: Email/password with Iron Session (encrypted cookies)
2. **API Clients**: Bearer tokens (SHA-256 hashed in database)
3. **Role-Based Access**: Admin vs Maintainer permissions
4. **Application-Level Access**: Maintainers only see assigned applications

### Data Security

1. **Passwords**: bcrypt hashing (12+ salt rounds)
2. **Tokens**: SHA-256 hashed, plaintext shown only once
3. **Session**: Encrypted cookies, 30-minute expiry
4. **File Storage**: No direct public access, pre-signed URLs
5. **Multi-Tenancy**: Complete data isolation per client application

---

## 9. Future Extensions

### Phase 2 Features (Post-MVP)

1. **SQL Report Generation**
   - hibiki-engine extension for structured data queries
   - Connect to client databases for reporting
   - Template-based report generation

2. **Advanced Analytics**
   - Usage metrics per client application
   - Query performance tracking
   - Document relevance scoring

3. **Multi-Modal RAG**
   - Image-based retrieval
   - Table extraction and querying
   - Chart/diagram understanding

4. **Workflow Automation**
   - Scheduled document ingestion
   - Automated report delivery
   - Webhook integrations

---

## 10. Development Roadmap

### Phase 1: Foundation (Weeks 1-2)
- Set up hibiki-console with authentication
- Database schema and migrations
- Basic admin functions (user/application management)

### Phase 2: Document Pipeline (Weeks 3-4)
- Document upload in hibiki-console
- Job queue integration
- Basic hibiki-ingest implementation
- Vector store integration

### Phase 3: RAG Engine (Weeks 5-6)
- hibiki-engine with Ollama integration
- Token authentication
- Vector retrieval and LLM querying
- API endpoint implementation

### Phase 4: Integration & Testing (Week 7)
- End-to-end testing
- Performance optimization
- Documentation
- Deployment preparation

### Phase 5: Production Launch (Week 8)
- Production deployment
- Monitoring and logging
- Client onboarding

---

## 11. Getting Started (For Developers)

### Prerequisites

```bash
# Required installations
- Node.js 20+
- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- MinIO (or use cloud S3)
- Ollama
```

### Quick Start

```bash
todo
```

---

## 12. Key Design Decisions

### Why Multi-Tenant?

- **Scalability**: Single deployment serves multiple clients
- **Cost-Effective**: Shared infrastructure with data isolation
- **Maintainability**: Centralized updates and monitoring

### Why Self-Hosted Ollama?

- **Privacy**: Complete control over data and models
- **Cost**: No per-token API costs
- **Customization**: Fine-tune models for specific use cases

### Why Next.js for Console?

- **Server Actions**: Simplified backend logic without separate API
- **Type Safety**: End-to-end TypeScript
- **Modern DX**: Great developer experience with App Router

### Why Job Queue?

- **Asynchronous Processing**: Don't block uploads on ingestion
- **Reliability**: Retry failed processing jobs
- **Scalability**: Multiple workers for parallel processing

---

## 13. Monitoring & Observability

### Key Metrics to Track

**hibiki-console**:
- Active users and sessions
- Document upload success rate
- Token generation rate

**hibiki-ingest**:
- Job processing time
- OCR success/failure rate
- Queue depth

**hibiki-engine**:
- Query latency (p50, p95, p99)
- Token validation time
- Vector search performance
- LLM inference time

### Logging Strategy

- **Structured Logging**: JSON format for easy parsing
- **Correlation IDs**: Track requests across services
- **Error Tracking**: Integration with Sentry or similar

---

## 14. Questions & Answers

### Q: Can one maintainer manage multiple applications?
**A**: Yes, admins can assign a maintainer to multiple applications. The console has an application switcher.

### Q: How do we handle large documents?
**A**: Files are uploaded to MinIO, then processed asynchronously by hibiki-ingest. Max file size is 20MB per file in v1.

### Q: Can we use different LLMs?
**A**: Yes, Ollama supports multiple models. Each client application can configure their preferred model in future versions.

### Q: What happens if ingestion fails?
**A**: Job status is marked as FAILED with error message. Maintainer can re-upload or investigate the issue.

### Q: How do we scale?
**A**:
- hibiki-console: Horizontal scaling (multiple Next.js instances)
- hibiki-ingest: Multiple worker processes
- hibiki-engine: Horizontal scaling with load balancer
- Ollama: GPU instances with model sharding

---

*— End of Document —*

**Authors**: Hibiki Development Team
**Last Updated**: December 2024
**Version**: 1.0
