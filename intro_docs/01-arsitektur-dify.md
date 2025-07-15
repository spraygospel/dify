# Arsitektur Dify

Memahami arsitektur Dify adalah langkah pertama yang penting sebelum melakukan customization. Dokumen ini menjelaskan struktur keseluruhan platform dengan cara yang mudah dipahami.

## Gambaran Umum

Dify menggunakan **arsitektur microservices** dengan pemisahan yang jelas antara komponen frontend dan backend:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Frontend  │◄──►│   API Backend   │◄──►│    Database     │
│   (Next.js)     │    │   (Flask)       │    │  (PostgreSQL)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│      Redis      │    │  Celery Worker  │    │  Vector Store   │
│    (Cache)      │    │ (Background)    │    │   (pgvector)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Komponen Utama

### 1. Web Frontend (`/web/`)

**Teknologi:** Next.js 15.3 + React 19 + TypeScript

**Fungsi Utama:**
- Dashboard untuk management aplikasi AI
- Visual workflow designer
- Tool dan model provider configuration
- User interface untuk end-users

**Struktur Penting:**
```
web/
├── app/                    # Next.js app router
│   ├── (commonLayout)/     # Dashboard layout
│   └── components/         # Reusable components
├── components/             # UI component library
│   ├── workflow/           # Visual workflow designer
│   ├── app/               # App configuration UI
│   └── tools/             # Tool management UI
└── service/               # API client functions
```

### 2. API Backend (`/api/`)

**Teknologi:** Flask + SQLAlchemy + PostgreSQL

**Fungsi Utama:**
- REST API untuk semua operasi
- AI model orchestration
- Workflow engine execution
- Background task processing

**Struktur Core (`/api/core/`):**
```
core/
├── workflow/              # 🎯 Workflow engine
├── tools/                 # 🛠️ Tool system (SAFE untuk extend)
├── model_runtime/         # 🤖 Model provider abstraction
├── rag/                   # 📚 RAG pipeline
├── agent/                 # 🤖 Agent execution logic
├── app/                   # 📱 Application orchestration (CORE)
└── extension/             # 🔌 Extension system (SAFE untuk extend)
```

### 3. Database Layer

**Primary Database:** PostgreSQL dengan pgvector extension
- User accounts dan applications
- Conversation history
- Document metadata
- Configuration settings

**Vector Database:** Opsional (pgvector, Qdrant, Chroma, dll)
- Document embeddings
- Semantic search
- RAG operations

**Cache Layer:** Redis
- Session storage
- API response caching
- Background job queue

### 4. Background Services

**Celery Worker:**
- Document processing
- Embedding generation
- Email notifications
- Scheduled tasks

**Plugin Daemon:**
- Plugin lifecycle management
- Plugin sandbox execution
- Plugin marketplace integration

## Alur Data Utama

### 1. Chat Flow
```
User Input → Web UI → API → Model Provider → Response → Web UI
                ↓
            Workflow Engine → Tools → External APIs
                ↓
            Vector Store → Document Retrieval → RAG
```

### 2. Document Processing Flow
```
File Upload → API → Celery Worker → Text Extraction → Chunking → Embedding → Vector Store
```

### 3. Tool Execution Flow
```
Workflow Node → Tool Manager → Tool Provider → External API → Response → Workflow
```

## Extension Points (AMAN untuk Customization)

### ✅ Tool System
- **Location:** `/api/core/tools/`
- **Pattern:** Provider-based dengan YAML configuration
- **Keamanan:** Terisolasi dengan SSRF protection
- **Examples:** Time tools, API tools, custom tools

### ✅ Model Runtime
- **Location:** `/api/core/model_runtime/model_providers/`
- **Pattern:** Provider abstraction dengan standard interface
- **Keamanan:** Credential encryption, rate limiting
- **Examples:** OpenAI, local models, custom providers

### ✅ Code-based Extensions
- **Location:** `/api/core/extension/`, `/api/core/moderation/`
- **Pattern:** Base class inheritance
- **Keamanan:** Input validation, sandbox execution
- **Examples:** Content moderation, external data sources

### ✅ Storage Backends
- **Location:** `/api/extensions/storage/`
- **Pattern:** Storage abstraction interface
- **Keamanan:** Access control, encryption
- **Examples:** S3, Azure Blob, local storage

## Core Areas (HINDARI Modifikasi)

### 🚫 Workflow Engine Core
- **Location:** `/api/core/workflow/graph_engine/`
- **Alasan:** Execution engine yang complex, break bisa affect semua workflows
- **Alternative:** Gunakan custom workflow nodes atau tools

### 🚫 Application Orchestration
- **Location:** `/api/core/app/`
- **Alasan:** Core business logic, banyak dependencies
- **Alternative:** Gunakan API endpoints untuk integrasi

### 🚫 Database Models
- **Location:** `/api/models/`
- **Alasan:** Schema changes affect semua components
- **Alternative:** Gunakan migration system atau extend existing models

### 🚫 Authentication Core
- **Location:** `/api/libs/login.py`, `/api/libs/passport.py`
- **Alasan:** Security-critical code
- **Alternative:** Gunakan existing auth patterns dalam custom code

## Teknologi Stack Detail

### Backend Stack
```python
# Core Framework
Flask 3.1.0              # Web framework
SQLAlchemy 2.0.29        # ORM
Pydantic 2.11.4          # Data validation

# AI & ML
OpenAI 1.61.0            # LLM integration
Transformers 4.51.0      # Local model support
NumPy 1.26.4             # Mathematical operations

# Database & Cache
PostgreSQL               # Primary database
Redis 6.1.0              # Cache & job queue
pgvector                 # Vector operations

# Background Processing
Celery 5.5.2             # Distributed task queue
Gunicorn 23.0.0          # WSGI server

# Development
pytest 8.3.2             # Testing framework
mypy 1.16.0              # Type checking
ruff 0.12.3              # Linting & formatting
```

### Frontend Stack
```typescript
// Core Framework
Next.js 15.3             // React framework
React 19.1.0             // UI library
TypeScript 5.8.3         // Type safety

// State Management
Zustand 4.5.2            // State management
SWR 2.3.0                // Server state

// UI Components
Tailwind CSS 3.4.14      // Styling
Headless UI 2.2.0        // Accessible components
ReactFlow 11.11.3        // Workflow designer

// Development
ESLint 9.20.1            // Linting
Jest 29.7.0              // Testing
```

## Security Architecture

### 1. Multi-tenancy
- Setiap resource diisolasi berdasarkan `tenant_id`
- User hanya bisa akses data dalam tenant mereka
- Custom code HARUS respect tenant isolation

### 2. SSRF Protection
- Semua outbound requests melalui `ssrf_proxy`
- Blacklist internal IPs dan sensitive endpoints
- Custom tools otomatis protected

### 3. Sandbox Execution
- Code execution dalam container terisolasi
- Resource limits (CPU, memory, time)
- Network restrictions

### 4. Credential Management
- API keys encrypted dalam database
- Environment variable untuk sensitive config
- Credential rotation support

## Performance Considerations

### 1. Caching Strategy
- Redis untuk API response caching
- Model prediction caching
- Embedding cache untuk RAG

### 2. Background Processing
- Heavy tasks (document processing) di Celery
- Async processing untuk better UX
- Queue prioritization

### 3. Database Optimization
- Connection pooling
- Index optimization untuk queries
- Vector index untuk similarity search

## Deployment Architecture

### Development (Docker Compose)
```yaml
services:
  web:          # Frontend development server
  api:          # Backend Flask app
  worker:       # Celery background worker
  db:           # PostgreSQL database
  redis:        # Cache & message broker
  sandbox:      # Code execution environment
```

### Production (Kubernetes/Docker Swarm)
```yaml
# High availability setup
- Web: Multiple replicas behind load balancer
- API: Horizontal scaling dengan shared cache
- Worker: Auto-scaling berdasarkan queue length
- Database: Master-slave replication
- Storage: Distributed file system
```

## Next Steps

Setelah memahami arsitektur:

1. **[Setup Development Environment](./02-setup-development.md)** - Hands-on setup
2. **[Guidelines Customization](./03-customization-guidelines.md)** - Detail safe areas
3. **[Custom Tools Tutorial](./04-membuat-custom-tools.md)** - Praktik pertama

---

> 💡 **Key Takeaway**: Dify dirancang dengan extension points yang jelas. Fokus pada tool system, model providers, dan code-based extensions untuk customization yang aman dan sustainable.