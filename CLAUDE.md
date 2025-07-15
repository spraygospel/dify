# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Dify is an open-source platform for developing LLM applications with two main components:
- **Backend API** (`/api`): Python-based Flask application with business logic, AI model integrations, and data processing
- **Frontend Web** (`/web`): Next.js React application providing the user interface

## Development Environment Setup

### Requirements
- **Backend**: Python >=3.11,<3.13, UV package manager
- **Frontend**: Node.js >=v22.11.0, pnpm package manager
- **Database**: PostgreSQL (with pgvector extension for vector operations)
- **Cache**: Redis
- **Container**: Docker and Docker Compose for local development

### Quick Start
```bash
# Clone and setup with Docker Compose (recommended)
cd dify/docker
cp .env.example .env
docker compose up -d

# Access at http://localhost/install for initial setup
```

## Common Development Commands

### Backend (API)
```bash
cd api

# Install dependencies
uv sync

# Run development server
uv run python app.py

# Run tests
uv run pytest

# Code formatting and linting
../dev/reformat  # runs ruff format + check + dotenv-linter + mypy
uv run --dev ruff check --fix ./
uv run --dev ruff format ./
uv run --dev mypy .

# Database migrations
uv run flask db upgrade

# Run background worker
uv run celery -A app.celery worker --loglevel=info
```

### Frontend (Web)
```bash
cd web

# Install dependencies
pnpm install

# Run development server
pnpm dev

# Build for production
pnpm build

# Linting and formatting
pnpm lint           # ESLint check
pnpm fix           # ESLint fix
pnpm test          # Jest tests
```

### Docker Development
```bash
# Build Docker images
make build-all      # builds both web and api images
make build-api      # api only
make build-web      # web only

# Start services
docker compose up -d

# View logs
docker compose logs -f api
docker compose logs -f web
```

## Code Architecture

### Backend Structure (`/api`)
- **`core/`**: Core business logic modules
  - `app/`: Application orchestration and request handling
  - `model_runtime/`: LLM provider integrations and model management
  - `workflow/`: Visual workflow engine with node-based execution
  - `rag/`: Retrieval-Augmented Generation components (embedding, vector stores, retrieval)
  - `tools/`: Built-in tools, custom tools, and tool management
  - `agent/`: Agent execution logic (Chain-of-Thought, Function Calling)
  - `prompt/`: Prompt templates and transformation utilities
- **`controllers/`**: Flask route controllers (console, web, service API endpoints)
- **`models/`**: SQLAlchemy database models
- **`services/`**: Business logic services (apps, datasets, auth, billing)
- **`tasks/`**: Celery background tasks (document processing, indexing)
- **`migrations/`**: Alembic database migration scripts

### Frontend Structure (`/web`)
- **`app/`**: Next.js app router pages and layouts
- **`components/`**: Reusable React components
- **`context/`**: React context providers for global state
- **`service/`**: API client functions and hooks
- **`i18n/`**: Internationalization translations
- **`utils/`**: Utility functions and helpers

### Key Concepts
- **Apps**: Core entities representing LLM applications (Chat, Completion, Agent, Workflow types)
- **Datasets**: Knowledge bases with document processing and vector storage
- **Workflows**: Visual node-based workflow builder with 20+ node types
- **Tools**: Extensible tool system supporting built-in, custom, and plugin tools
- **Providers**: Model provider integrations (OpenAI, Anthropic, local models, etc.)

## Testing

### Backend Tests
```bash
cd api

# Run all tests
uv run pytest

# Run specific test suites
../dev/pytest/pytest_unit_tests.sh      # unit tests
../dev/pytest/pytest_model_runtime.sh   # model runtime tests
../dev/pytest/pytest_workflow.sh        # workflow tests
../dev/pytest/pytest_tools.sh           # tools tests
../dev/pytest/pytest_vdb.sh             # vector database tests

# Run with coverage
uv run pytest --cov=./api --cov-report=html
```

### Frontend Tests
```bash
cd web
pnpm test          # Jest unit tests
pnpm test:watch    # Watch mode
```

## Configuration

### Environment Variables
- **Backend**: Copy `api/.env.example` to `api/.env` and configure
- **Frontend**: Copy `web/.env.example` to `web/.env.local` and configure
- **Docker**: Use `docker/.env.example` as template for `docker/.env`

### Key Configuration Areas
- **Database**: PostgreSQL connection settings
- **Redis**: Cache and session storage
- **Model Providers**: API keys and endpoints for LLM providers
- **Vector Stores**: Configuration for different vector database backends
- **Storage**: File storage backend (local, S3, Azure, etc.)
- **Authentication**: OAuth and SSO settings

## Database

### Migrations
```bash
cd api
# Create new migration
uv run flask db migrate -m "description"

# Apply migrations
uv run flask db upgrade

# Downgrade
uv run flask db downgrade
```

### Key Models
- **Account/User**: User management and authentication
- **App**: Application definitions and configurations
- **Dataset**: Knowledge base and document storage
- **Conversation/Message**: Chat interactions and history
- **Workflow**: Visual workflow definitions
- **Provider**: Model provider configurations

## Development Tips

### Code Style
- **Backend**: Follow PEP 8, use Ruff for formatting/linting, MyPy for type checking
- **Frontend**: Use ESLint + TypeScript, follow React best practices
- **Commits**: Link issues in PR descriptions, use conventional commit messages

### Debugging
- **Backend**: Use Flask debug mode, check Docker logs for errors
- **Frontend**: Use React DevTools, browser console for debugging
- **Database**: Monitor logs and query performance
- **Vector Operations**: Test embedding and retrieval separately

### Performance
- **Backend**: Use async where appropriate, optimize database queries
- **Frontend**: Lazy load components, optimize bundle size
- **Caching**: Leverage Redis for expensive operations
- **Vector Search**: Monitor embedding cache hit rates

## Plugin Development

Dify supports plugins for extending functionality:
- **Model Providers**: Add new LLM/embedding providers
- **Tools**: Create custom tools for agents and workflows
- **Extensions**: Build API-based extensions

Refer to the [plugin repository](https://github.com/langgenius/dify-plugins) for development guidelines.