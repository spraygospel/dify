# Production Deployment

Panduan lengkap untuk deploy customization Dify ke production environment dengan aman dan reliable.

## Production vs Development

| Aspek | Development | Production |
|-------|-------------|------------|
| **Database** | SQLite/Docker Postgres | Managed PostgreSQL |
| **Cache** | Local Redis | Redis Cluster |
| **Storage** | Local files | S3/Azure Blob |
| **Monitoring** | Basic logs | Full observability |
| **Security** | Relaxed | Hardened |
| **Scaling** | Single instance | Load balanced |

## Pre-deployment Checklist

### ✅ Code Quality
```bash
# Backend checks
cd api
uv run ruff check ./                    # Linting
uv run mypy .                           # Type checking
uv run pytest --cov=./ --cov-report=xml # Test coverage

# Frontend checks
cd web
pnpm lint                               # ESLint
pnpm build                              # Build verification
pnpm test                               # Unit tests
```

### ✅ Security Review
```bash
# Check for secrets in code
grep -r "api[_-]key" . --exclude-dir=node_modules
grep -r "password" . --exclude-dir=.git
grep -r "secret" . --exclude-dir=logs

# Verify environment variables
echo "Checking .env files for production readiness..."
grep -E "(SECRET_KEY|DB_PASSWORD|API_KEY)" .env
```

### ✅ Performance Testing
```bash
# Load test API endpoints
ab -n 1000 -c 10 http://localhost:5001/health

# Memory usage check
docker stats dify-api dify-web --no-stream

# Database query performance
PGPASSWORD=password psql -h localhost -U postgres -d dify \
  -c "EXPLAIN ANALYZE SELECT * FROM messages WHERE conversation_id = 'test';"
```

## Infrastructure Setup

### 1. Database (PostgreSQL)

**Managed Database (Recommended):**
```yaml
# docker-compose.prod.yaml
services:
  api:
    environment:
      - DB_HOST=your-postgres-host.amazonaws.com
      - DB_USERNAME=${DB_USERNAME}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_DATABASE=dify_prod
      - DB_PORT=5432
      - SSL_MODE=require
```

**Self-hosted with Backup:**
```bash
# Automated backup script
#!/bin/bash
BACKUP_DIR="/backups/dify"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup
docker exec dify-db pg_dump -U postgres dify > "$BACKUP_DIR/dify_backup_$DATE.sql"

# Compress backup
gzip "$BACKUP_DIR/dify_backup_$DATE.sql"

# Upload to S3 (optional)
aws s3 cp "$BACKUP_DIR/dify_backup_$DATE.sql.gz" s3://your-backup-bucket/

# Clean old backups (keep 30 days)
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete
```

### 2. Redis Clustering

```yaml
# redis-cluster.yaml
services:
  redis-master:
    image: redis:7-alpine
    command: redis-server --appendonly yes --replica-read-only no
    volumes:
      - redis-master-data:/data
    networks:
      - dify-network
  
  redis-replica:
    image: redis:7-alpine
    command: redis-server --appendonly yes --replicaof redis-master 6379
    depends_on:
      - redis-master
    networks:
      - dify-network

volumes:
  redis-master-data:
```

### 3. Storage Configuration

**AWS S3:**
```bash
# Environment variables
STORAGE_TYPE=s3
S3_BUCKET_NAME=your-dify-bucket
S3_ACCESS_KEY=your-access-key
S3_SECRET_KEY=your-secret-key
S3_REGION=us-east-1
S3_ENDPOINT=https://s3.amazonaws.com
```

**Azure Blob:**
```bash
STORAGE_TYPE=azure-blob
AZURE_BLOB_ACCOUNT_NAME=yourstorageaccount
AZURE_BLOB_ACCOUNT_KEY=your-account-key
AZURE_BLOB_CONTAINER_NAME=dify-files
```

## Docker Production Configuration

### Production docker-compose.yaml

```yaml
version: '3.8'

services:
  # Nginx Load Balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx-logs:/var/log/nginx
    depends_on:
      - api
      - web
    networks:
      - dify-network
    restart: unless-stopped

  # API Service (Multiple replicas)
  api:
    image: langgenius/dify-api:latest
    environment:
      - SECRET_KEY=${SECRET_KEY}
      - DB_HOST=${DB_HOST}
      - DB_USERNAME=${DB_USERNAME}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=${REDIS_HOST}
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - STORAGE_TYPE=${STORAGE_TYPE}
      - S3_BUCKET_NAME=${S3_BUCKET_NAME}
      - CELERY_WORKER_AMOUNT=2
      - GUNICORN_WORKER_AMOUNT=4
      - LOG_LEVEL=INFO
    volumes:
      - ./custom-tools:/app/api/core/tools/builtin_tool/providers/custom:ro
    networks:
      - dify-network
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'

  # Web Service
  web:
    image: langgenius/dify-web:latest
    environment:
      - NEXT_PUBLIC_API_PREFIX=${API_URL}
      - NEXT_PUBLIC_SENTRY_DSN=${SENTRY_DSN}
    networks:
      - dify-network
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 1G
          cpus: '0.5'

  # Worker Service
  worker:
    image: langgenius/dify-api:latest
    command: celery -A app.celery worker --loglevel=info --concurrency=4
    environment:
      - SECRET_KEY=${SECRET_KEY}
      - DB_HOST=${DB_HOST}
      - DB_USERNAME=${DB_USERNAME} 
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=${REDIS_HOST}
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    volumes:
      - ./custom-tools:/app/api/core/tools/builtin_tool/providers/custom:ro
    networks:
      - dify-network
    restart: unless-stopped
    deploy:
      replicas: 3

volumes:
  nginx-logs:

networks:
  dify-network:
    driver: bridge
```

### Nginx Configuration

**File: `nginx/nginx.conf`**
```nginx
upstream api_backend {
    least_conn;
    server api:5001 max_fails=3 fail_timeout=30s;
}

upstream web_backend {
    least_conn;
    server web:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    
    # Security Headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
    
    # API Routes
    location /api {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Web Routes
    location / {
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # File Upload
    client_max_body_size 100M;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;
}
```

## Environment Variables

### Production .env Template

```bash
# =================================
# CORE CONFIGURATION
# =================================
SECRET_KEY=your-super-secret-production-key
EDITION=SELF_HOSTED
DEPLOY_ENV=PRODUCTION
LOG_LEVEL=INFO

# =================================
# DATABASE CONFIGURATION
# =================================
DB_HOST=your-postgres-host.com
DB_USERNAME=dify_user
DB_PASSWORD=your-secure-db-password
DB_DATABASE=dify_production
DB_PORT=5432
DB_CHARSET=utf8mb4

# =================================
# REDIS CONFIGURATION
# =================================
REDIS_HOST=your-redis-host.com
REDIS_PORT=6379
REDIS_PASSWORD=your-secure-redis-password
REDIS_DB=0
REDIS_USE_SSL=true

# =================================
# STORAGE CONFIGURATION
# =================================
STORAGE_TYPE=s3
S3_BUCKET_NAME=your-production-bucket
S3_ACCESS_KEY=your-s3-access-key
S3_SECRET_KEY=your-s3-secret-key
S3_REGION=us-east-1

# =================================
# MODEL PROVIDERS
# =================================
OPENAI_API_KEY=your-openai-key
ANTHROPIC_API_KEY=your-anthropic-key
GOOGLE_API_KEY=your-google-key

# =================================
# SECURITY CONFIGURATION
# =================================
CORS_ALLOW_ORIGINS=https://your-domain.com
WEB_API_CORS_ALLOW_ORIGINS=https://your-domain.com
CONSOLE_CORS_ALLOW_ORIGINS=https://your-domain.com

# =================================
# MONITORING
# =================================
SENTRY_DSN=your-sentry-dsn
SENTRY_ENVIRONMENT=production

# =================================
# EMAIL CONFIGURATION
# =================================
MAIL_TYPE=smtp
SMTP_SERVER=your-smtp-server.com
SMTP_PORT=587
SMTP_USERNAME=your-email@domain.com
SMTP_PASSWORD=your-email-password
SMTP_USE_TLS=true

# =================================
# WORKER CONFIGURATION
# =================================
CELERY_WORKER_AMOUNT=4
GUNICORN_WORKER_AMOUNT=8
GUNICORN_TIMEOUT=200
```

## Deployment Strategies

### 1. Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

SET_COLOR=$1  # blue or green
CURRENT_COLOR=$(docker service ls --filter "name=dify" --format "{{.Name}}" | head -1 | cut -d'-' -f2)

if [ "$CURRENT_COLOR" = "blue" ]; then
    NEW_COLOR="green"
else
    NEW_COLOR="blue"
fi

echo "Deploying to $NEW_COLOR environment..."

# Deploy new version
docker stack deploy -c docker-compose.$NEW_COLOR.yaml dify-$NEW_COLOR

# Health check
echo "Waiting for health check..."
sleep 30

HEALTH=$(curl -s http://localhost:5001/health | jq -r '.status')
if [ "$HEALTH" = "ok" ]; then
    echo "Switching traffic to $NEW_COLOR"
    # Update load balancer
    docker service update --image nginx:alpine dify-nginx
    
    # Remove old environment
    docker stack rm dify-$CURRENT_COLOR
else
    echo "Health check failed, rolling back"
    docker stack rm dify-$NEW_COLOR
    exit 1
fi
```

### 2. Rolling Update

```bash
# rolling-update.sh
#!/bin/bash

echo "Starting rolling update..."

# Update API services one by one
docker service update --image langgenius/dify-api:latest \
  --update-parallelism 1 \
  --update-delay 30s \
  dify-api

# Update web services
docker service update --image langgenius/dify-web:latest \
  --update-parallelism 1 \
  --update-delay 10s \
  dify-web

echo "Rolling update completed"
```

## Monitoring & Observability

### 1. Health Checks

```python
# Custom health check endpoint
@app.route('/health/detailed')
def detailed_health():
    health_status = {
        'status': 'ok',
        'timestamp': datetime.utcnow().isoformat(),
        'services': {}
    }
    
    # Database check
    try:
        db.session.execute('SELECT 1')
        health_status['services']['database'] = 'ok'
    except Exception as e:
        health_status['services']['database'] = f'error: {str(e)}'
        health_status['status'] = 'degraded'
    
    # Redis check
    try:
        redis_client.ping()
        health_status['services']['redis'] = 'ok'
    except Exception as e:
        health_status['services']['redis'] = f'error: {str(e)}'
        health_status['status'] = 'degraded'
    
    # Custom tools check
    try:
        # Test custom tool availability
        health_status['services']['custom_tools'] = 'ok'
    except Exception as e:
        health_status['services']['custom_tools'] = f'error: {str(e)}'
    
    return jsonify(health_status)
```

### 2. Logging Configuration

```python
# logging.conf
[loggers]
keys=root,dify

[handlers]
keys=console,file,syslog

[formatters]
keys=generic

[logger_root]
level=INFO
handlers=console

[logger_dify]
level=INFO
handlers=file,syslog
qualname=dify
propagate=0

[handler_console]
class=StreamHandler
args=(sys.stdout,)
level=INFO
formatter=generic

[handler_file]
class=logging.handlers.RotatingFileHandler
args=('/var/log/dify/dify.log', 'a', 10485760, 5)
level=INFO
formatter=generic

[handler_syslog]
class=logging.handlers.SysLogHandler
args=('/dev/log',)
level=INFO
formatter=generic

[formatter_generic]
format=%(asctime)s [%(levelname)s] %(name)s: %(message)s
```

### 3. Metrics Collection

```python
# Prometheus metrics
from prometheus_client import Counter, Histogram, Gauge, generate_latest

# Custom metrics
CUSTOM_TOOL_INVOCATIONS = Counter(
    'dify_custom_tool_invocations_total',
    'Total custom tool invocations',
    ['tool_name', 'status']
)

CUSTOM_TOOL_DURATION = Histogram(
    'dify_custom_tool_duration_seconds',
    'Custom tool execution duration',
    ['tool_name']
)

ACTIVE_CONVERSATIONS = Gauge(
    'dify_active_conversations',
    'Number of active conversations'
)

@app.route('/metrics')
def metrics():
    return generate_latest()
```

## Security Hardening

### 1. Container Security

```dockerfile
# Secure Dockerfile example
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r dify && useradd -r -g dify dify

# Install security updates
RUN apt-get update && apt-get upgrade -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy application
COPY --chown=dify:dify . /app
WORKDIR /app

# Switch to non-root user
USER dify

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5001/health || exit 1

CMD ["gunicorn", "app:app"]
```

### 2. Network Security

```yaml
# Docker network isolation
networks:
  frontend:
    driver: bridge
    internal: false
  backend:
    driver: bridge
    internal: true
  database:
    driver: bridge
    internal: true

services:
  nginx:
    networks:
      - frontend
  
  api:
    networks:
      - frontend
      - backend
  
  db:
    networks:
      - database
```

### 3. Secrets Management

```bash
# Using Docker Secrets
echo "super-secret-password" | docker secret create db_password -
echo "api-key-12345" | docker secret create openai_key -

# Reference in compose
services:
  api:
    secrets:
      - db_password
      - openai_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - OPENAI_API_KEY_FILE=/run/secrets/openai_key

secrets:
  db_password:
    external: true
  openai_key:
    external: true
```

## Backup & Disaster Recovery

### Automated Backup Script

```bash
#!/bin/bash
# backup-production.sh

BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/dify/$BACKUP_DATE"

mkdir -p $BACKUP_DIR

# Database backup
echo "Backing up database..."
pg_dump -h $DB_HOST -U $DB_USERNAME -d $DB_DATABASE > "$BACKUP_DIR/database.sql"

# Redis backup
echo "Backing up Redis..."
redis-cli -h $REDIS_HOST -p $REDIS_PORT --rdb "$BACKUP_DIR/redis.rdb"

# Custom tools backup
echo "Backing up custom tools..."
tar -czf "$BACKUP_DIR/custom_tools.tar.gz" /app/api/core/tools/builtin_tool/providers/custom/

# Configuration backup
echo "Backing up configuration..."
cp .env "$BACKUP_DIR/env_backup"
cp docker-compose.prod.yaml "$BACKUP_DIR/"

# Upload to cloud storage
echo "Uploading to S3..."
aws s3 sync $BACKUP_DIR s3://your-backup-bucket/dify-backups/$BACKUP_DATE/

# Cleanup local backups (keep 7 days)
find /backups/dify -type d -mtime +7 -exec rm -rf {} \;

echo "Backup completed: $BACKUP_DATE"
```

## Maintenance Procedures

### Database Maintenance

```sql
-- Weekly maintenance queries

-- Analyze table statistics
ANALYZE;

-- Reindex performance-critical tables
REINDEX TABLE messages;
REINDEX TABLE conversations;
REINDEX TABLE document_segments;

-- Clean up old logs (older than 30 days)
DELETE FROM app_logs WHERE created_at < NOW() - INTERVAL '30 days';

-- Vacuum to reclaim space
VACUUM ANALYZE;
```

### Log Rotation

```bash
# /etc/logrotate.d/dify
/var/log/dify/*.log {
    daily
    missingok
    rotate 30
    compress
    notifempty
    create 644 dify dify
    postrotate
        docker kill -s USR1 $(docker ps -q --filter ancestor=langgenius/dify-api)
    endscript
}
```

## Performance Optimization

### Database Optimization

```sql
-- Create indexes for custom tools
CREATE INDEX CONCURRENTLY idx_tool_provider_user_id 
  ON tool_provider_credentials (user_id);

CREATE INDEX CONCURRENTLY idx_messages_conversation_created 
  ON messages (conversation_id, created_at DESC);

-- Partition large tables
CREATE TABLE messages_2024 PARTITION OF messages 
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

### Caching Strategy

```python
# Redis caching for custom tools
from redis import Redis
import json

class CustomToolCache:
    def __init__(self):
        self.redis = Redis(host=REDIS_HOST, password=REDIS_PASSWORD)
    
    def get_tool_result(self, tool_name: str, params_hash: str):
        key = f"tool:{tool_name}:{params_hash}"
        result = self.redis.get(key)
        if result:
            return json.loads(result)
        return None
    
    def set_tool_result(self, tool_name: str, params_hash: str, result: dict, ttl: int = 3600):
        key = f"tool:{tool_name}:{params_hash}"
        self.redis.setex(key, ttl, json.dumps(result))
```

## Next Steps

1. **[Best Practices](./08-best-practices.md)** - Professional development patterns
2. **[Troubleshooting](./09-troubleshooting.md)** - Debug production issues
3. **[Monitoring Setup](./monitoring-guide.md)** - Advanced observability

---

> 💡 **Key Takeaway**: Production deployment memerlukan perhatian detail pada security, monitoring, dan disaster recovery. Gunakan automation untuk mengurangi human error dan maintain consistency.