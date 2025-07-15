# Guidelines Customization Dify

Panduan ini menjelaskan area mana yang aman untuk dimodifikasi dan mana yang harus dihindari ketika melakukan customization Dify.

## Filosofi Customization

### Prinsip Utama: **Extend, Don't Modify**

Dify dirancang dengan **extension points** yang memungkinkan customization tanpa memodifikasi core code. Filosofi ini memastikan:

- ✅ **Compatibility** - Update Dify tidak break customization Anda
- ✅ **Security** - Core security mechanisms tetap intact
- ✅ **Stability** - Core functionality tidak terganggu
- ✅ **Maintainability** - Mudah debug dan maintain

## Zona Keamanan Customization

### 🟢 ZONA HIJAU - AMAN untuk Customization

Area ini **dirancang khusus** untuk extension dan customization:

#### 1. Tool System (`/api/core/tools/`)

**Custom Tool Providers:**
```
/api/core/tools/builtin_tool/providers/your_provider/
├── your_provider.py          # Provider implementation
├── your_provider.yaml        # Provider configuration
├── tools/
│   ├── tool_name.py         # Tool implementation
│   └── tool_name.yaml       # Tool configuration
└── _assets/
    └── icon.svg             # Provider icon
```

**Mengapa Aman:**
- Terisolasi dari core engine
- SSRF protection otomatis
- Schema validation built-in
- Error handling terstandardisasi

#### 2. Model Runtime Providers (`/api/core/model_runtime/model_providers/`)

**Custom Model Providers:**
```
/api/core/model_runtime/model_providers/your_provider/
├── provider.yaml            # Provider configuration
├── provider.py              # Provider implementation
├── llm/
│   └── llm_model.py        # LLM implementation
└── text_embedding/
    └── embedding_model.py   # Embedding implementation
```

**Mengapa Aman:**
- Standard interface untuk semua providers
- Credential encryption otomatis
- Rate limiting dan monitoring built-in
- Fallback mechanism tersedia

#### 3. Code-based Extensions (`/api/core/extension/`)

**Moderation Extensions:**
```
/api/core/moderation/your_moderation/
├── __init__.py
├── your_moderation.py       # Moderation implementation
└── __builtin__             # Position file
```

**External Data Tools:**
```
/api/core/external_data_tool/your_tool/
├── __init__.py
└── your_tool.py            # Data tool implementation
```

**Mengapa Aman:**
- Base class dengan validation
- Sandbox execution untuk security
- Input/output standardization

#### 4. Storage Backends (`/api/extensions/storage/`)

**Custom Storage:**
```
/api/extensions/storage/your_storage_storage.py
```

**Mengapa Aman:**
- Storage abstraction layer
- Access control built-in
- Encryption support

### 🟡 ZONA KUNING - HATI-HATI

Area ini bisa dimodifikasi tapi perlu pemahaman lebih dalam:

#### 1. Frontend Components (`/web/app/components/`)

**Boleh:**
- Membuat component baru untuk custom features
- Extend existing components dengan props tambahan
- Customize styling dengan Tailwind classes

**Hati-hati:**
- Jangan ubah core component behavior
- Maintain TypeScript types
- Follow existing component patterns

#### 2. API Controllers (`/api/controllers/`)

**Boleh:**
- Tambah endpoint baru untuk custom features
- Extend existing controllers dengan methods baru

**Hati-hati:**
- Jangan ubah existing endpoint behavior
- Maintain authentication dan authorization
- Follow error handling patterns

#### 3. Database Migrations (`/api/migrations/`)

**Boleh:**
- Tambah table baru untuk custom features
- Tambah column ke existing table (dengan migration)

**Hati-hati:**
- Jangan drop atau alter existing columns
- Test migration rollback
- Backup database sebelum migration

### 🔴 ZONA MERAH - HINDARI

Area ini **JANGAN DIMODIFIKASI** karena akan break system:

#### 1. Workflow Engine Core (`/api/core/workflow/graph_engine/`)

**Mengapa Bahaya:**
- Complex state management
- Performance-critical code
- Banyak interdependencies
- Breaking changes sulit di-debug

**Alternative:**
- Gunakan custom tools dalam workflow
- Extend workflow nodes jika perlu (advanced)

#### 2. Application Orchestration (`/api/core/app/`)

**File yang Jangan Diubah:**
- `/api/core/app/apps/base_app_runner.py`
- `/api/core/app/apps/base_app_generator.py`
- `/api/core/app/task_pipeline/`

**Mengapa Bahaya:**
- Core business logic
- App lifecycle management
- Queue dan task coordination

**Alternative:**
- Gunakan API endpoints untuk integrasi
- Custom tools untuk extend functionality

#### 3. Database Models Core (`/api/models/`)

**File yang Jangan Diubah:**
- Core entity relationships
- Primary key definitions
- Foreign key constraints

**Mengapa Bahaya:**
- Schema dependencies
- Data integrity
- Migration complexity

**Alternative:**
- Tambah custom fields via migration
- Extend models dengan composition pattern

#### 4. Authentication & Security (`/api/libs/`)

**File yang Jangan Diubah:**
- `/api/libs/passport.py`
- `/api/libs/login.py`
- `/api/libs/rsa.py`

**Mengapa Bahaya:**
- Security vulnerabilities
- Session management
- Credential handling

**Alternative:**
- Gunakan existing auth patterns
- Extend dengan OAuth providers

#### 5. Flask Extensions (`/api/extensions/ext_*.py`)

**File yang Jangan Diubah:**
- `/api/extensions/ext_database.py`
- `/api/extensions/ext_redis.py`
- `/api/extensions/ext_celery.py`

**Mengapa Bahaya:**
- Application startup dependencies
- Service initialization
- Global state management

## Best Practices untuk Safe Customization

### 1. Follow Existing Patterns

**DO:**
```python
# Follow existing tool pattern
class MyCustomTool(BuiltinTool):
    def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
        # Your implementation
        try:
            result = self.do_something(tool_parameters)
            yield self.create_text_message(str(result))
        except Exception as e:
            yield self.create_text_message(f"Error: {str(e)}")
```

**DON'T:**
```python
# Don't create tools from scratch
class MyBrokenTool:  # Missing base class
    def invoke(self):  # Wrong method signature
        return "result"  # Wrong return type
```

### 2. Use Configuration Over Hard-coding

**DO:**
```yaml
# tool.yaml
identity:
  name: my_tool
parameters:
  - name: api_endpoint
    type: string
    required: true
    label:
      en_US: "API Endpoint"
      id_ID: "Endpoint API"
```

**DON'T:**
```python
# Don't hard-code values
API_ENDPOINT = "https://hardcoded-api.com"  # BAD
```

### 3. Handle Errors Gracefully

**DO:**
```python
def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
    try:
        result = self.call_external_api(tool_parameters)
        if not result:
            yield self.create_text_message("No data found")
            return
        yield self.create_text_message(str(result))
    except requests.RequestException as e:
        yield self.create_text_message(f"API Error: {str(e)}")
    except Exception as e:
        yield self.create_text_message(f"Unexpected error: {str(e)}")
```

### 4. Respect Security Boundaries

**DO:**
```python
# Use proper authentication
def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
    # user_id is provided by framework - respect it
    if not user_id:
        yield self.create_text_message("Authentication required")
        return
    
    # Use credentials from tool configuration
    credentials = self.credentials
    api_key = credentials.get('api_key')
```

**DON'T:**
```python
# Don't bypass authentication
def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
    # DON'T ignore user_id
    # DON'T hard-code credentials
    api_key = "hardcoded-secret-key"  # SECURITY RISK
```

### 5. Use Proper Logging

**DO:**
```python
import logging

logger = logging.getLogger(__name__)

def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
    logger.info(f"Tool invoked by user {user_id} with params {tool_parameters}")
    try:
        # Implementation
        logger.debug("Tool execution successful")
    except Exception as e:
        logger.error(f"Tool execution failed: {str(e)}")
```

## Testing Your Customizations

### 1. Unit Testing

```python
# tests/unit_tests/tools/test_my_tool.py
import pytest
from core.tools.builtin_tool.providers.my_provider.tools.my_tool import MyTool

class TestMyTool:
    def test_invoke_success(self):
        tool = MyTool()
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"param": "value"}
        ))
        assert len(result) == 1
        assert "expected result" in result[0].message
    
    def test_invoke_error_handling(self):
        tool = MyTool()
        result = list(tool._invoke(
            user_id="test-user", 
            tool_parameters={"invalid": "param"}
        ))
        assert "Error:" in result[0].message
```

### 2. Integration Testing

```bash
# Test dalam development environment
cd api
uv run pytest tests/integration_tests/tools/test_my_tool.py -v

# Test dengan real API calls
uv run pytest tests/integration_tests/ -k "my_tool" --api-tests
```

### 3. Manual Testing

```bash
# Test via API endpoint
curl -X POST "http://localhost:5001/v1/workflows/run" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": {},
    "query": "Test my custom tool",
    "response_mode": "blocking",
    "user": "test-user"
  }'
```

## Version Compatibility

### Maintaining Compatibility

**Semantic Versioning:**
- **Major Version** (1.x.x → 2.x.x): Breaking changes, review customizations
- **Minor Version** (1.6.x → 1.7.x): New features, usually compatible
- **Patch Version** (1.6.0 → 1.6.1): Bug fixes, fully compatible

**Update Strategy:**
1. **Backup** customizations sebelum update
2. **Test** di staging environment
3. **Review** changelog untuk breaking changes
4. **Update** customizations jika perlu

### Extension Points Stability

| Area | Stability | Upgrade Risk |
|------|-----------|--------------|
| Tool System | 🟢 High | Low - Well-defined interface |
| Model Providers | 🟢 High | Low - Standard abstraction |
| Code Extensions | 🟡 Medium | Medium - API might evolve |
| Database Schema | 🟡 Medium | Medium - Migration required |
| Frontend Components | 🟠 Low | High - React/Next.js updates |

## Migration Guide untuk Updates

### Before Update

```bash
# 1. Backup customizations
cp -r api/core/tools/builtin_tool/providers/my_provider ./backup/
cp api/.env ./backup/

# 2. Export database
docker compose exec db pg_dump -U postgres dify > ./backup/dify_backup.sql

# 3. Document customizations
echo "Custom tools: my_provider, other_provider" > ./backup/customization_notes.txt
```

### After Update

```bash
# 1. Check breaking changes
git diff HEAD~1 api/core/tools/__base/tool.py

# 2. Update custom code if needed
# Review changes in base classes

# 3. Test customizations
uv run pytest tests/unit_tests/tools/test_my_provider.py

# 4. Deploy gradually
# Test di staging dulu
```

## Common Pitfalls & Solutions

### ❌ Pitfall 1: Modifying Core Files
```python
# WRONG: Editing core/app/apps/base_app_runner.py
class BaseAppRunner:
    def run(self):
        # Your modification here - WILL BREAK ON UPDATE
```

**✅ Solution: Use Extension Points**
```python
# CORRECT: Create custom tool
class MyCustomTool(BuiltinTool):
    def _invoke(self, ...):
        # Your logic here - UPDATE SAFE
```

### ❌ Pitfall 2: Hard-coding Configurations
```python
# WRONG: Hard-coded values
API_KEY = "sk-1234567890"
ENDPOINT = "https://api.example.com"
```

**✅ Solution: Use Configuration System**
```yaml
# tool.yaml
parameters:
  - name: api_key
    type: secret-input
  - name: endpoint
    type: string
```

### ❌ Pitfall 3: Ignoring Security
```python
# WRONG: Bypassing validation
def _invoke(self, user_id: str, ...):
    # Ignore user_id and security
    return self.call_admin_api()
```

**✅ Solution: Respect Security Boundaries**
```python
# CORRECT: Proper security handling
def _invoke(self, user_id: str, ...):
    if not self.validate_user_permission(user_id):
        yield self.create_text_message("Permission denied")
        return
```

## Next Steps

Setelah memahami guidelines:

1. **[Custom Tools Tutorial](./04-membuat-custom-tools.md)** - Praktik safe customization
2. **[Code-based Extensions](./05-code-based-extensions.md)** - Advanced extension patterns
3. **[Best Practices](./08-best-practices.md)** - Production-ready development

---

> 💡 **Key Takeaway**: Dify menyediakan extension points yang powerful dan aman. Gunakan zona hijau untuk 90% customization needs, dan hindari zona merah untuk menjaga stability sistem.