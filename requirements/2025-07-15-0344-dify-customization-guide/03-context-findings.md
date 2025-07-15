# Context Findings

## Summary
Based on discovery answers and targeted analysis, I've identified key areas for the Dify customization guide.

## Specific Files That Need Creation (Not Modification)

### Documentation Structure
```
intro_docs/
├── README.md                          # Overview & navigation
├── 01-arsitektur-dify.md             # Architecture explanation
├── 02-setup-development.md           # Development environment
├── 03-customization-guidelines.md    # Safe vs unsafe areas  
├── 04-membuat-custom-tools.md        # Custom tools tutorial
├── 05-code-based-extensions.md       # Extensions tutorial
├── 06-model-providers.md             # Custom model providers
├── 07-production-deployment.md       # Deployment guide
├── 08-best-practices.md              # Best practices & patterns
└── 09-troubleshooting.md             # Common issues & solutions
```

## Key Implementation Patterns Found

### 1. Custom Tools (Primary Focus)
**Safe Extension Points:**
- `/api/core/tools/builtin_tool/providers/` - Add new provider directories
- `/api/core/tools/custom_tool/` - API-based tools (existing, extend patterns)

**Working Example Pattern:** `/api/core/tools/builtin_tool/providers/time/`
```python
# Provider class pattern
class TimeToolProvider(BuiltinToolProviderController):
    def _validate_credentials(self, user_id: str, credentials: dict[str, Any]) -> None:
        pass

# Tool implementation pattern  
class CurrentTimeTool(BuiltinTool):
    def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
        # Implementation with proper error handling
        yield self.create_text_message(result)
```

**Configuration Pattern:**
```yaml
# provider.yaml
identity:
  author: "Developer Name"
  name: "provider_name"
  label:
    en_US: "English Label"
    id_ID: "Indonesian Label"  # Add Indonesian support

# tool.yaml
identity:
  name: tool_name
parameters:
  - name: param_name
    type: string
    required: true
    label:
      en_US: "Parameter"
      id_ID: "Parameter"
```

### 2. Code-based Extensions (Secondary Focus)
**Safe Extension Points:**
- `/api/core/moderation/` - Add new moderation modules
- `/api/core/external_data_tool/` - Add data source integrations

**Pattern:** Inherit from base classes, follow directory structure
```python
# Extension pattern
from core.moderation.base import Moderation

class CustomModeration(Moderation):
    name: str = "custom_name"
    
    @classmethod  
    def validate_config(cls, tenant_id: str, config: dict) -> None:
        # Validation logic
```

### 3. Model Providers (Advanced Topic)
**Finding:** Provider implementations not fully visible in this codebase, but structure indicates pattern:
```
model_providers/custom_provider/
├── provider.yaml     # Configuration
├── provider.py       # Implementation
└── models/           # Model types
```

### 4. Production Deployment
**Configuration Files to Customize:**
- `docker/.env` - Environment variables
- `docker/docker-compose.yaml` - Service configuration
- Custom volumes for persistent data

**Key Patterns:**
- Tenant isolation in database
- Encrypted credential storage  
- SSRF protection for external APIs
- Sandbox execution for code

## Technical Architecture Insights

### Core vs Extension Areas
**✅ SAFE (Create new files/directories):**
- Tool providers in `/api/core/tools/builtin_tool/providers/`
- Moderation extensions in `/api/core/moderation/`
- External data tools in `/api/core/external_data_tool/`
- Model providers in `/api/core/model_runtime/model_providers/`
- Storage backends in `/api/extensions/storage/`

**🚫 AVOID (Core system files):**
- `/api/core/app/` - Application orchestration
- `/api/core/workflow/graph_engine/` - Workflow engine
- `/api/models/` - Database models (except for migrations)
- `/api/extensions/ext_*.py` - Flask extension initialization
- `/api/migrations/` - Database migrations

### Development Workflow
1. **Local Development:** Docker Compose setup
2. **Code Extensions:** Follow base class patterns
3. **Configuration:** YAML-based with i18n support
4. **Testing:** Pytest with coverage
5. **Production:** Environment variable configuration

## Documentation Standards Found

### Language Support
- Indonesian (`id-ID`) defined but not fully supported
- Opportunity to add Indonesian documentation
- Pattern: Bilingual EN/ID throughout

### Technical Writing Pattern
- Clear section hierarchy with ## headers
- Code examples with language specification
- > [!IMPORTANT] and > [!NOTE] for callouts
- Step-by-step numbered instructions
- Absolute file paths for clarity
- Working, copy-paste ready examples

## Integration Points for Junior Developers

### Recommended Learning Path
1. **Start with Custom Tools** (safest, most practical)
2. **Add Code-based Extensions** (moderation, data sources)
3. **Explore Model Providers** (advanced users only)
4. **Production Deployment** (after development mastery)

### Avoid These Areas
- Plugin system (too complex for juniors)
- Custom workflow nodes (affects core engine)
- Direct database model modifications
- Core application logic changes

## Related Features Analyzed
- Time tool provider (simple example)
- OpenAI moderation (extension example)  
- API tool system (external integration pattern)
- Workflow node architecture (for understanding, not modification)
- Docker deployment configuration

## Files to Reference in Documentation
- `/api/core/tools/builtin_tool/providers/time/` - Tool example
- `/api/core/moderation/openai_moderation/` - Extension example
- `/docker/docker-compose.yaml` - Deployment configuration
- `/api/.env.example` - Environment variables
- `/CONTRIBUTING.md` - Development guidelines

**Next Phase:** Create expert detail questions based on these findings.