# Troubleshooting

Panduan untuk mendiagnosis dan menyelesaikan masalah umum dalam development dan deployment Dify customization.

## Common Issues

### 1. Custom Tool Tidak Muncul di Dashboard

**Symptoms:**
- Tool tidak terlihat di tool selector
- Provider tidak terdaftar di dashboard
- Error "Tool not found" saat execution

**Diagnosis Steps:**

```bash
# 1. Check file structure
ls -la api/core/tools/builtin_tool/providers/your_provider/
# Expected:
# your_provider.py
# your_provider.yaml
# tools/
# _assets/ (optional)

# 2. Validate YAML syntax
cd api
uv run python -c "
import yaml
with open('core/tools/builtin_tool/providers/your_provider/your_provider.yaml') as f:
    config = yaml.safe_load(f)
    print('YAML valid:', config)
"

# 3. Check tool registration
uv run python -c "
from core.tools.tool_manager import ToolManager
manager = ToolManager()
tools = manager.get_available_tools()
print([t.name for t in tools if 'your_provider' in t.provider])
"

# 4. Check logs for errors
docker compose logs api | grep -i "your_provider"
docker compose logs api | grep -i "error"
```

**Common Solutions:**

1. **Missing required files:**
   ```bash
   # Create missing files
   touch api/core/tools/builtin_tool/providers/your_provider/__init__.py
   ```

2. **YAML syntax errors:**
   ```yaml
   # Fix indentation and structure
   identity:
     author: "Your Name"  # Use quotes for strings
     name: "provider_name"  # Snake_case, no spaces
   ```

3. **Python import errors:**
   ```python
   # Check class naming convention
   class YourProviderToolProvider(BuiltinToolProviderController):  # Must end with 'ToolProvider'
       pass
   ```

4. **Restart services:**
   ```bash
   docker compose restart api worker
   # Wait 30 seconds for full restart
   ```

### 2. Tool Execution Errors

**Symptoms:**
- "Tool execution failed" messages
- Timeout errors
- Connection errors to external APIs

**Diagnosis Steps:**

```bash
# 1. Test tool directly
cd api
uv run python -c "
from core.tools.builtin_tool.providers.your_provider.tools.your_tool import YourTool
tool = YourTool()
tool.credentials = {'api_key': 'test-key'}
result = list(tool._invoke('test-user', {'param': 'value'}))
print('Result:', result[0].message)
"

# 2. Check network connectivity
curl -v https://api.external-service.com/health

# 3. Validate credentials
# Use tool's validate_credentials method
uv run python -c "
from core.tools.builtin_tool.providers.your_provider.your_provider import YourProviderToolProvider
provider = YourProviderToolProvider()
provider._validate_credentials('test-user', {'api_key': 'your-key'})
print('Credentials valid')
"

# 4. Check SSRF proxy (if using external APIs)
docker compose logs ssrf_proxy
```

**Common Solutions:**

1. **API Key issues:**
   ```python
   # Debug credential access
   def _invoke(self, user_id, tool_parameters, ...):
       api_key = self.credentials.get('api_key')
       if not api_key:
           yield self.create_text_message("Error: API key not configured")
           return
       
       # Test API key format
       if not api_key.startswith('expected_prefix'):
           yield self.create_text_message("Error: Invalid API key format")
           return
   ```

2. **Timeout issues:**
   ```python
   # Increase timeout and add retry logic
   import requests
   from requests.adapters import HTTPAdapter
   from urllib3.util.retry import Retry
   
   def _make_request(self, url, data):
       session = requests.Session()
       retry_strategy = Retry(
           total=3,
           backoff_factor=1,
           status_forcelist=[429, 500, 502, 503, 504]
       )
       adapter = HTTPAdapter(max_retries=retry_strategy)
       session.mount("http://", adapter)
       session.mount("https://", adapter)
       
       return session.post(url, json=data, timeout=30)
   ```

3. **Parameter validation:**
   ```python
   def _invoke(self, user_id, tool_parameters, ...):
       # Add detailed parameter validation
       required_params = ['city', 'country']
       for param in required_params:
           if not tool_parameters.get(param):
               yield self.create_text_message(f"Error: Missing required parameter '{param}'")
               return
       
       # Validate parameter types and values
       try:
           limit = int(tool_parameters.get('limit', 10))
           if limit <= 0 or limit > 100:
               yield self.create_text_message("Error: limit must be between 1 and 100")
               return
       except ValueError:
           yield self.create_text_message("Error: limit must be a valid number")
           return
   ```

### 3. Database Migration Issues

**Symptoms:**
- "Table doesn't exist" errors
- Migration failures
- Foreign key constraint errors

**Diagnosis Steps:**

```bash
# 1. Check current migration status
cd api
uv run flask db current
uv run flask db history

# 2. Check database connection
uv run python -c "
from extensions.ext_database import db
from flask import Flask
app = Flask(__name__)
with app.app_context():
    db.session.execute('SELECT version();')
    print('Database connected')
"

# 3. Validate migration files
ls -la migrations/versions/
# Check for syntax errors in migration files

# 4. Check database schema
docker compose exec db psql -U postgres -d dify -c "\dt"
```

**Common Solutions:**

1. **Reset migrations (development only):**
   ```bash
   # DANGER: This will delete all data
   docker compose down
   docker volume rm dify_db_data
   docker compose up -d db
   # Wait for db to start
   cd api
   uv run flask db upgrade
   ```

2. **Manual migration fix:**
   ```bash
   # Connect to database
   docker compose exec db psql -U postgres -d dify
   
   # Check migration state
   SELECT * FROM alembic_version;
   
   # Fix migration version if needed
   UPDATE alembic_version SET version_num = 'target_version';
   ```

3. **Create new migration:**
   ```bash
   cd api
   uv run flask db migrate -m "Fix migration issue"
   uv run flask db upgrade
   ```

### 4. Performance Issues

**Symptoms:**
- Slow tool execution
- High memory usage
- Database query timeouts

**Diagnosis Steps:**

```bash
# 1. Monitor resource usage
docker stats dify-api dify-worker dify-db --no-stream

# 2. Check database performance
docker compose exec db psql -U postgres -d dify -c "
SELECT query, calls, total_time, mean_time 
FROM pg_stat_statements 
ORDER BY total_time DESC LIMIT 10;
"

# 3. Profile Python code
cd api
uv run python -m cProfile -o profile.stats app.py
# Analyze with:
uv run python -c "
import pstats
p = pstats.Stats('profile.stats')
p.sort_stats('cumulative').print_stats(20)
"

# 4. Check Redis performance
docker compose exec redis redis-cli INFO memory
docker compose exec redis redis-cli INFO stats
```

**Common Solutions:**

1. **Database optimization:**
   ```sql
   -- Add missing indexes
   CREATE INDEX CONCURRENTLY idx_messages_conversation_created 
     ON messages (conversation_id, created_at DESC);
   
   -- Analyze table statistics
   ANALYZE messages;
   ANALYZE conversations;
   ```

2. **Caching implementation:**
   ```python
   from functools import lru_cache
   import redis
   
   class OptimizedTool(BuiltinTool):
       def __init__(self):
           super().__init__()
           self.redis = redis.Redis(host='redis', password='password')
       
       @lru_cache(maxsize=1000)
       def _expensive_computation(self, input_data: str) -> str:
           # Cache expensive operations in memory
           return self._compute(input_data)
       
       def _invoke_with_cache(self, cache_key: str, compute_func):
           # Try Redis cache first
           cached = self.redis.get(cache_key)
           if cached:
               return json.loads(cached)
           
           # Compute and cache
           result = compute_func()
           self.redis.setex(cache_key, 3600, json.dumps(result))
           return result
   ```

3. **Memory optimization:**
   ```python
   # Use generators instead of lists
   def _process_large_dataset(self, data):
       for item in data:  # Instead of loading all into memory
           yield self._process_item(item)
   
   # Clean up resources
   def _invoke(self, ...):
       try:
           # Tool logic
           pass
       finally:
           # Cleanup connections, file handles, etc.
           if hasattr(self, '_connection'):
               self._connection.close()
   ```

### 5. Authentication & Authorization Issues

**Symptoms:**
- "Access denied" errors
- User context not available
- Tenant isolation failures

**Diagnosis Steps:**

```bash
# 1. Check user context
cd api
uv run python -c "
from flask import Flask, request, g
from extensions.ext_database import db
from models.account import Account

app = Flask(__name__)
with app.app_context():
    # Test user lookup
    user = Account.query.filter_by(email='test@example.com').first()
    print('User found:', user.id if user else None)
"

# 2. Check API authentication
curl -H "Authorization: Bearer your-api-key" \
     http://localhost:5001/v1/tools/execute

# 3. Verify tenant isolation
# Check that user_id is properly passed to tools
```

**Common Solutions:**

1. **Proper user context handling:**
   ```python
   def _invoke(self, user_id: str, tool_parameters: dict, ...):
       # Always validate user_id
       if not user_id:
           yield self.create_text_message("Error: User authentication required")
           return
       
       # Use user_id for tenant-specific operations
       user_data = self._get_user_data(user_id)
       if not user_data:
           yield self.create_text_message("Error: Invalid user")
           return
   ```

2. **Credential scope validation:**
   ```python
   def _validate_credentials(self, user_id: str, credentials: dict) -> None:
       # Ensure credentials belong to the user
       api_key = credentials.get('api_key')
       if not self._verify_api_key_for_user(api_key, user_id):
           raise ValueError("API key not authorized for this user")
   ```

## Debugging Techniques

### 1. Enable Debug Mode

```bash
# Development environment
export FLASK_DEBUG=true
export LOG_LEVEL=DEBUG

# In .env file
DEBUG=true
LOG_LEVEL=DEBUG
FLASK_DEBUG=true
```

### 2. Add Debug Logging

```python
import logging

# Setup debug logging in your tool
class DebugTool(BuiltinTool):
    def __init__(self):
        super().__init__()
        self.logger = logging.getLogger(self.__class__.__name__)
        self.logger.setLevel(logging.DEBUG)
    
    def _invoke(self, user_id: str, tool_parameters: dict, ...):
        self.logger.debug(f"Tool invoked with params: {tool_parameters}")
        
        try:
            # Add debug points throughout execution
            self.logger.debug("Starting API request")
            response = self._make_api_call()
            self.logger.debug(f"API response: {response.status_code}")
            
            result = self._process_response(response)
            self.logger.debug(f"Processed result: {result}")
            
            yield self.create_text_message(str(result))
            
        except Exception as e:
            self.logger.error(f"Tool execution failed: {str(e)}", exc_info=True)
            yield self.create_text_message(f"Error: {str(e)}")
```

### 3. Interactive Debugging

```python
# Add breakpoints in your code
import pdb

def _invoke(self, user_id: str, tool_parameters: dict, ...):
    pdb.set_trace()  # Debugger will stop here
    
    # Or use better debugger
    import ipdb
    ipdb.set_trace()
    
    # Tool logic
    pass
```

### 4. Unit Test Debugging

```python
# Run specific test with verbose output
cd api
uv run pytest tests/unit_tests/tools/test_your_tool.py::TestYourTool::test_specific_case -v -s

# Run with debugger
uv run pytest tests/unit_tests/tools/test_your_tool.py --pdb

# Run with coverage
uv run pytest tests/unit_tests/tools/ --cov=core/tools --cov-report=html
```

## Log Analysis

### 1. Finding Relevant Logs

```bash
# API logs
docker compose logs api | grep -i "error\|exception\|traceback"
docker compose logs api | grep "your_tool_name"

# Worker logs (for background tasks)
docker compose logs worker | tail -100

# Database logs
docker compose logs db | grep -i "error\|slow"

# All service logs with timestamps
docker compose logs -t --since="1h" | grep -i "error"
```

### 2. Log Filtering Techniques

```bash
# Filter by time range
docker compose logs --since="2024-01-15T10:00:00" --until="2024-01-15T11:00:00"

# Filter by service and follow
docker compose logs -f api worker

# Search for specific patterns
docker compose logs api 2>&1 | grep -E "(ERROR|CRITICAL|Exception)"

# Export logs for analysis
docker compose logs api > api_logs.txt
```

### 3. Structured Log Analysis

```python
# If using structured logging (JSON)
import json

def analyze_logs(log_file):
    errors = []
    with open(log_file, 'r') as f:
        for line in f:
            try:
                log_entry = json.loads(line)
                if log_entry.get('level') == 'ERROR':
                    errors.append(log_entry)
            except json.JSONDecodeError:
                continue
    
    # Analyze error patterns
    error_types = {}
    for error in errors:
        error_type = error.get('error_type', 'unknown')
        error_types[error_type] = error_types.get(error_type, 0) + 1
    
    print("Error frequency:", error_types)
```

## Environment Issues

### 1. Docker Environment

```bash
# Check Docker resources
docker system df
docker system prune  # Clean up unused resources

# Check container health
docker compose ps
docker inspect dify-api | grep -i health

# Reset Docker environment
docker compose down --volumes
docker compose up -d
```

### 2. Network Issues

```bash
# Test internal network connectivity
docker compose exec api ping db
docker compose exec api ping redis

# Test external connectivity
docker compose exec api curl -v https://api.openai.com

# Check DNS resolution
docker compose exec api nslookup google.com
```

### 3. Resource Constraints

```bash
# Check available resources
free -h
df -h

# Docker resource usage
docker stats --no-stream

# Increase Docker resources if needed
# Docker Desktop: Settings > Resources
```

## Production Issues

### 1. Service Health Monitoring

```bash
# Health check script
#!/bin/bash
services=("api" "web" "worker" "db" "redis")

for service in "${services[@]}"; do
    status=$(docker compose ps $service --format "table {{.State}}" | tail -1)
    if [ "$status" != "running" ]; then
        echo "❌ $service is $status"
        docker compose logs $service | tail -20
    else
        echo "✅ $service is running"
    fi
done

# Test API endpoints
curl -f http://localhost:5001/health || echo "❌ API health check failed"
curl -f http://localhost:3000 || echo "❌ Web health check failed"
```

### 2. Performance Monitoring

```bash
# Monitor key metrics
while true; do
    echo "=== $(date) ==="
    
    # API response time
    curl -w "API Response Time: %{time_total}s\n" -o /dev/null -s http://localhost:5001/health
    
    # Memory usage
    docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
    
    # Database connections
    docker compose exec db psql -U postgres -d dify -c "
        SELECT count(*) as active_connections 
        FROM pg_stat_activity 
        WHERE state = 'active';
    "
    
    sleep 60
done
```

### 3. Backup & Recovery

```bash
# Emergency backup
#!/bin/bash
BACKUP_DIR="/tmp/emergency_backup_$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# Database backup
docker compose exec db pg_dump -U postgres dify > "$BACKUP_DIR/database.sql"

# Redis backup
docker compose exec redis redis-cli SAVE
docker cp $(docker compose ps -q redis):/data/dump.rdb "$BACKUP_DIR/redis.rdb"

# Configuration backup
cp .env "$BACKUP_DIR/"
cp docker-compose.yaml "$BACKUP_DIR/"

# Custom tools backup
tar -czf "$BACKUP_DIR/custom_tools.tar.gz" api/core/tools/builtin_tool/providers/custom/

echo "Emergency backup created: $BACKUP_DIR"
```

## Getting Help

### 1. Community Resources

- **Discord Community**: https://discord.gg/FngNHpbcY7
- **GitHub Issues**: https://github.com/langgenius/dify/issues
- **Documentation**: https://docs.dify.ai
- **Plugin Repository**: https://github.com/langgenius/dify-plugins

### 2. Creating Effective Bug Reports

```markdown
# Bug Report Template

## Environment
- Dify Version: 1.6.0
- OS: Ubuntu 22.04
- Docker Version: 24.0.0
- Python Version: 3.11.0

## Expected Behavior
Describe what you expected to happen

## Actual Behavior
Describe what actually happened

## Steps to Reproduce
1. Step one
2. Step two
3. Step three

## Error Messages
```
Paste error messages or logs here
```

## Additional Context
- Screenshots if relevant
- Configuration files (remove sensitive data)
- Related custom tools or extensions
```

### 3. Debug Information Collection

```bash
# Debug info collection script
#!/bin/bash
DEBUG_DIR="debug_info_$(date +%Y%m%d_%H%M%S)"
mkdir -p $DEBUG_DIR

# System information
uname -a > "$DEBUG_DIR/system_info.txt"
docker version > "$DEBUG_DIR/docker_version.txt"
docker compose version > "$DEBUG_DIR/compose_version.txt"

# Service status
docker compose ps > "$DEBUG_DIR/service_status.txt"
docker compose logs api > "$DEBUG_DIR/api_logs.txt"
docker compose logs worker > "$DEBUG_DIR/worker_logs.txt"

# Configuration (remove sensitive data)
cp .env.example "$DEBUG_DIR/env_template.txt"
cp docker-compose.yaml "$DEBUG_DIR/"

# Resource usage
docker stats --no-stream > "$DEBUG_DIR/resource_usage.txt"

echo "Debug information collected in: $DEBUG_DIR"
echo "Please review and remove any sensitive information before sharing"
```

## Prevention Strategies

### 1. Code Review Checklist

```markdown
## Custom Tool Review Checklist

### Functionality
- [ ] Tool executes successfully with valid inputs
- [ ] Error handling covers all failure scenarios
- [ ] Parameter validation is comprehensive
- [ ] Documentation is complete and accurate

### Security
- [ ] No hardcoded credentials or secrets
- [ ] Input sanitization implemented
- [ ] External API calls use proper authentication
- [ ] User context is properly validated

### Performance
- [ ] No blocking operations in main thread
- [ ] Appropriate timeout values set
- [ ] Caching implemented where beneficial
- [ ] Resource cleanup in finally blocks

### Testing
- [ ] Unit tests cover happy path
- [ ] Unit tests cover error scenarios
- [ ] Integration tests pass
- [ ] Load testing completed
```

### 2. Monitoring Setup

```python
# Proactive monitoring
import logging
from datetime import datetime, timedelta

class ToolMonitor:
    def __init__(self):
        self.logger = logging.getLogger('tool_monitor')
        self.error_counts = {}
        self.last_reset = datetime.now()
    
    def log_tool_error(self, tool_name: str, error: str):
        # Track error frequency
        if tool_name not in self.error_counts:
            self.error_counts[tool_name] = 0
        self.error_counts[tool_name] += 1
        
        # Alert if error rate is high
        if self.error_counts[tool_name] > 10:  # 10 errors
            self.logger.critical(f"High error rate for {tool_name}: {self.error_counts[tool_name]} errors")
            # Send alert to monitoring system
            self._send_alert(tool_name, self.error_counts[tool_name])
    
    def _send_alert(self, tool_name: str, error_count: int):
        # Integration with monitoring systems
        # (Slack, email, PagerDuty, etc.)
        pass
```

---

> 💡 **Key Takeaway**: Troubleshooting yang efektif memerlukan systematic approach. Mulai dengan symptoms, gather data, isolate the problem, lalu apply targeted solutions. Proactive monitoring dan logging sangat membantu dalam mencegah dan mendiagnosis issues.