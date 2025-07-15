# Setup Development Environment

Dokumen ini menjelaskan cara setup environment development Dify dengan benar dan aman untuk programmer junior.

## Prerequisites

Sebelum memulai, pastikan sistem Anda memenuhi requirements berikut:

### System Requirements
- **OS:** Linux, macOS, atau Windows dengan WSL2
- **CPU:** Minimum 2 cores (recommended 4+ cores)
- **RAM:** Minimum 4GB (recommended 8GB+)
- **Storage:** Minimum 20GB free space

### Software Requirements
```bash
# Cek versi yang terinstall
docker --version          # Docker 20.10+
docker compose version    # Docker Compose 2.0+
git --version             # Git 2.30+
```

## Metode Setup

Dify menyediakan beberapa cara setup development environment:

| Metode | Tingkat | Keterangan |
|--------|---------|------------|
| **Docker Compose** | 🟢 Pemula | **Recommended** - Simple, isolated, production-like |
| Manual Setup | 🟡 Menengah | Custom configuration, lebih flexible |
| Dev Container | 🟠 Lanjutan | VS Code integration, consistent environment |

## Setup dengan Docker Compose (Recommended)

### 1. Clone Repository

```bash
# Clone official Dify repository
git clone https://github.com/langgenius/dify.git
cd dify

# Periksa struktur project
ls -la
# Output:
# drwxr-xr-x  api/          # Backend Python code
# drwxr-xr-x  web/          # Frontend React code
# drwxr-xr-x  docker/       # Docker configuration
# -rw-r--r--  README.md
# -rw-r--r--  LICENSE
```

### 2. Environment Configuration

```bash
# Masuk ke directory docker
cd docker

# Copy environment template
cp .env.example .env

# Edit file .env sesuai kebutuhan
nano .env  # atau gunakan editor favorit Anda
```

**Konfigurasi Penting dalam `.env`:**

```bash
# ===== Core Configuration =====
# Secret key untuk encryption (WAJIB diganti!)
SECRET_KEY=sk-generate-your-own-secret-key-here

# Database configuration
DB_USERNAME=postgres
DB_PASSWORD=dify123456  # Ganti dengan password yang kuat
DB_HOST=db
DB_PORT=5432
DB_DATABASE=dify

# Redis configuration
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=dify123456  # Ganti dengan password yang kuat

# ===== Storage Configuration =====
STORAGE_TYPE=local
STORAGE_LOCAL_PATH=storage

# ===== LLM Provider Configuration =====
# Tambahkan API keys untuk model yang akan digunakan
OPENAI_API_KEY=your-openai-api-key-here
ANTHROPIC_API_KEY=your-anthropic-api-key-here

# ===== Development Configuration =====
# Set ke development mode
EDITION=SELF_HOSTED
DEPLOY_ENV=DEVELOPMENT
DEBUG=true

# API URL untuk frontend
NEXT_PUBLIC_API_PREFIX=http://localhost/api
APP_API_URL=http://api:5001
```

### 3. Start Services

```bash
# Start semua services dalam background
docker compose up -d

# Monitor logs untuk memastikan startup berhasil
docker compose logs -f api web worker

# Cek status services
docker compose ps
```

**Expected Output:**
```
NAME           SERVICE        STATUS         PORTS
dify-api       api           running        0.0.0.0:5001->5001/tcp
dify-web       web           running        0.0.0.0:3000->3000/tcp
dify-worker    worker        running        
dify-db        db            running        0.0.0.0:5432->5432/tcp
dify-redis     redis         running        0.0.0.0:6379->6379/tcp
```

### 4. Initial Setup

```bash
# Buka browser dan akses setup wizard
# URL: http://localhost/install

# Atau langsung ke dashboard
# URL: http://localhost
```

**Setup Wizard Steps:**
1. **Language Selection** - Pilih bahasa (English/中文)
2. **Administrator Account** - Buat admin user pertama
3. **Model Configuration** - Setup LLM provider (OpenAI/Anthropic/dll)
4. **Complete Setup** - Finish dan redirect ke dashboard

## Manual Setup (Opsional)

Jika Anda ingin setup manual untuk development yang lebih flexible:

### Backend Setup

```bash
# Masuk ke directory API
cd api

# Install UV package manager (if not installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync --dev

# Setup environment variables
cp .env.example .env
# Edit .env dengan database dan service configurations

# Database setup (perlu PostgreSQL dengan pgvector)
# Install PostgreSQL dan pgvector extension terlebih dahulu

# Run database migrations
uv run flask db upgrade

# Start development server
uv run python app.py
# API akan running di http://localhost:5001
```

### Frontend Setup

```bash
# Masuk ke directory Web (terminal baru)
cd web

# Install pnpm package manager (if not installed)
npm install -g pnpm

# Install dependencies
pnpm install

# Setup environment variables
cp .env.example .env.local
# Edit .env.local dengan API URL

# Start development server
pnpm dev
# Web akan running di http://localhost:3000
```

### Background Services

```bash
# Start Celery worker (terminal baru)
cd api
uv run celery -A app.celery worker --loglevel=info

# Start Redis (jika tidak menggunakan Docker)
redis-server

# Start PostgreSQL dengan pgvector
# (tergantung OS dan installation method)
```

## Development Tools Setup

### Code Editor Configuration

**VS Code (Recommended):**

```json
// .vscode/settings.json
{
  "python.defaultInterpreterPath": "./api/.venv/bin/python",
  "python.formatting.provider": "ruff",
  "python.linting.enabled": true,
  "python.linting.ruffEnabled": true,
  "typescript.preferences.importModuleSpecifier": "relative",
  "eslint.workingDirectories": ["web"]
}
```

**Recommended Extensions:**
- Python
- TypeScript and JavaScript Language Features
- ESLint
- Tailwind CSS IntelliSense
- Docker
- REST Client

### Git Hooks Setup

```bash
# Install pre-commit hooks (optional tapi recommended)
cd dify

# Setup pre-commit untuk backend
cd api
uv run pre-commit install

# Setup pre-commit untuk frontend
cd ../web
pnpm run prepare
```

## Verification & Testing

### 1. Basic Health Check

```bash
# Test API endpoint
curl http://localhost:5001/health

# Expected response:
# {"status": "ok", "version": "1.6.0"}

# Test Web frontend
curl http://localhost:3000
# Should return HTML content
```

### 2. Database Connection

```bash
# Connect to PostgreSQL
docker compose exec db psql -U postgres -d dify

# Check tables
\dt

# Check pgvector extension
SELECT * FROM pg_extension WHERE extname = 'vector';
```

### 3. Redis Connection

```bash
# Connect to Redis
docker compose exec redis redis-cli

# Test basic operations
ping
# Expected: PONG

set test "hello world"
get test
# Expected: "hello world"
```

### 4. Background Worker

```bash
# Check Celery worker logs
docker compose logs worker

# Should show:
# celery@worker ready
# Connected to redis://...
```

## Troubleshooting Common Issues

### Port Conflicts

```bash
# Jika port 80/3000/5001 sudah digunakan
# Edit docker-compose.yaml atau .env file:

# Untuk menggunakan port berbeda:
WEB_PORT=3001
API_PORT=5002
```

### Permission Issues

```bash
# Fix Docker permission issues di Linux
sudo chown -R $USER:$USER .
sudo chmod -R 755 ./volumes
```

### Database Connection Issues

```bash
# Reset database jika ada masalah
docker compose down
docker volume rm dify_db_data
docker compose up -d db
# Wait for db to be ready, then:
docker compose up -d
```

### Memory Issues

```bash
# Jika sistem kehabisan memory
# Kurangi worker processes di docker-compose.yaml:

services:
  api:
    environment:
      - CELERY_WORKER_AMOUNT=1  # Default: 2
      - GUNICORN_WORKER_AMOUNT=1  # Default: 4
```

### Slow Performance

```bash
# Enable development optimizations
# Edit .env:
DEBUG=true
FLASK_DEBUG=true

# Disable unnecessary services:
# Comment out services yang tidak digunakan di docker-compose.yaml
```

## Development Workflow

### Daily Workflow

```bash
# 1. Start development environment
cd dify/docker
docker compose up -d

# 2. Check logs untuk errors
docker compose logs -f api web

# 3. Make changes ke code
# Files di ./api dan ./web akan auto-reload

# 4. Test changes
# API: http://localhost:5001
# Web: http://localhost:3000

# 5. Stop environment
docker compose down
```

### Code Formatting & Linting

```bash
# Backend formatting
cd api
../dev/reformat              # Run ruff + mypy + dotenv-linter

# Frontend formatting
cd web
pnpm lint                     # ESLint check
pnpm fix                      # ESLint fix
```

### Running Tests

```bash
# Backend tests
cd api
uv run pytest                # All tests
uv run pytest tests/unit_tests/  # Unit tests only

# Frontend tests  
cd web
pnpm test                     # Jest tests
```

## Environment Variables Reference

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `SECRET_KEY` | - | **Required** - Encryption key |
| `DB_USERNAME` | postgres | Database username |
| `DB_PASSWORD` | - | **Required** - Database password |
| `REDIS_PASSWORD` | - | **Required** - Redis password |

### LLM Providers

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `ANTHROPIC_API_KEY` | Anthropic (Claude) API key |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI API key |
| `GOOGLE_API_KEY` | Google (Gemini) API key |

### Development

| Variable | Default | Description |
|----------|---------|-------------|
| `DEBUG` | false | Enable debug mode |
| `FLASK_DEBUG` | false | Flask debug mode |
| `LOG_LEVEL` | INFO | Logging level |

## Next Steps

Setelah setup berhasil:

1. **[Customization Guidelines](./03-customization-guidelines.md)** - Pelajari area yang aman untuk dimodifikasi
2. **[Custom Tools Tutorial](./04-membuat-custom-tools.md)** - Buat tool pertama Anda
3. **[Production Deployment](./07-production-deployment.md)** - Setup untuk production

---

> 💡 **Tips**: Gunakan Docker Compose untuk development karena lebih mudah dan sesuai dengan production environment. Manual setup hanya untuk kasus khusus yang memerlukan customization lebih dalam.