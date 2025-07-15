# Requirements Specification: Dify Customization Guide

## Problem Statement

Programmer junior membutuhkan panduan lengkap yang mudah dipahami untuk membuat AI agent menggunakan Dify platform. Karena Dify adalah aplikasi mature, mereka perlu tahu area mana yang aman untuk customization dan bagaimana melakukan customization dengan benar tanpa merusak core functionality.

## Solution Overview

Membuat dokumentasi komprehensif dalam bahasa Indonesia di folder `intro_docs/` yang mencakup:
1. Arsitektur Dify yang mudah dipahami
2. Guidelines customization yang aman
3. Tutorial step-by-step membuat AI agent
4. Best practices untuk development dan deployment

## Functional Requirements

### FR1: Dokumentasi Arsitektur
- **Requirement:** Penjelasan sederhana tentang struktur Dify (backend API, frontend Web, database, cache)
- **Detail:** Diagram arsitektur, penjelasan setiap komponen, dan hubungan antar komponen
- **Acceptance Criteria:** Junior programmer dapat memahami overall architecture tanpa membaca source code

### FR2: Custom Tools Tutorial
- **Requirement:** Panduan lengkap membuat custom tools untuk AI agent
- **Detail:** Menggunakan pattern dari `/api/core/tools/builtin_tool/providers/time/` sebagai reference
- **Implementation:** 
  - Struktur file provider dan tool
  - YAML configuration dengan dukungan bahasa Indonesia
  - Python implementation dengan error handling
  - Testing dengan pytest framework
- **Acceptance Criteria:** Programmer dapat membuat dan deploy custom tool yang working

### FR3: Code-based Extensions Guide
- **Requirement:** Tutorial untuk membuat moderation dan external data tool extensions
- **Detail:** Menggunakan pattern dari `/api/core/moderation/` dan `/api/core/external_data_tool/`
- **Implementation:**
  - Base class inheritance pattern
  - Directory structure requirements
  - Configuration validation
- **Acceptance Criteria:** Programmer dapat extend Dify dengan custom extensions

### FR4: Model Provider Integration
- **Requirement:** Panduan integrasi custom model providers (untuk advanced users)
- **Detail:** Berdasarkan struktur `/api/core/model_runtime/model_providers/`
- **Implementation:**
  - Provider configuration pattern
  - Authentication handling
  - Model type implementations
- **Acceptance Criteria:** Advanced users dapat menambahkan local models atau custom providers

### FR5: Production Deployment Guide
- **Requirement:** Cara deploy customization ke production environment
- **Detail:** Docker configuration, environment variables, dan best practices
- **Implementation:**
  - Environment variable configuration
  - Volume mounting untuk persistence
  - Security considerations
- **Acceptance Criteria:** Customization dapat di-deploy dengan aman ke production

### FR6: Authentication & Security Understanding
- **Requirement:** Pemahaman dasar sistem authentication dan tenant isolation
- **Detail:** Bagaimana custom code harus handle user_id dan tenant_id
- **Implementation:**
  - Parameter patterns dalam tool methods
  - Security best practices
  - Tenant isolation requirements
- **Acceptance Criteria:** Custom code respect security dan multi-tenancy

### FR7: Indonesian Language Support
- **Requirement:** Tutorial menambahkan dukungan bahasa Indonesia
- **Detail:** Menggunakan pattern i18n yang sudah ada di Dify
- **Implementation:**
  - YAML label configuration dengan id-ID locale
  - Frontend component localization
  - Error message translation
- **Acceptance Criteria:** Custom tools mendukung bahasa Indonesia

### FR8: Testing Framework Integration
- **Requirement:** Cara testing custom tools menggunakan pytest framework
- **Detail:** Menggunakan test infrastructure di `/api/pytest.ini`
- **Implementation:**
  - Test file structure
  - Mock patterns untuk external APIs
  - Coverage requirements
- **Acceptance Criteria:** Custom tools memiliki proper test coverage

## Technical Requirements

### TR1: File Locations
- **Target Directory:** `/intro_docs/`
- **File Structure:**
  ```
  intro_docs/
  ├── README.md                          # Navigation & overview
  ├── 01-arsitektur-dify.md             # Architecture explanation
  ├── 02-setup-development.md           # Development setup
  ├── 03-customization-guidelines.md    # Safe vs unsafe areas
  ├── 04-membuat-custom-tools.md        # Custom tools tutorial
  ├── 05-code-based-extensions.md       # Extensions tutorial
  ├── 06-model-providers.md             # Model provider integration
  ├── 07-production-deployment.md       # Deployment guide
  ├── 08-best-practices.md              # Best practices
  └── 09-troubleshooting.md             # Common issues
  ```

### TR2: Documentation Standards
- **Language:** Bahasa Indonesia (primary) dengan code examples dalam English
- **Format:** Markdown dengan consistent formatting
- **Code Examples:** Working, copy-pasteable examples dengan proper error handling
- **Structure:** Numbered sections, clear headers, step-by-step instructions

### TR3: Reference Implementation Patterns
- **Custom Tools:** Follow `/api/core/tools/builtin_tool/providers/time/` pattern
- **Extensions:** Follow `/api/core/moderation/openai_moderation/` pattern  
- **Configuration:** YAML-based dengan i18n labels
- **Testing:** pytest dengan coverage menggunakan existing framework

### TR4: Safe Customization Areas (INCLUDE)
- `/api/core/tools/builtin_tool/providers/` - Custom tool providers
- `/api/core/moderation/` - Custom moderation extensions
- `/api/core/external_data_tool/` - Custom data source tools
- `/api/core/model_runtime/model_providers/` - Custom model providers
- `/api/extensions/storage/` - Custom storage backends

### TR5: Unsafe Areas (EXCLUDE)
- `/api/core/app/` - Application orchestration
- `/api/core/workflow/graph_engine/` - Workflow engine core
- `/api/models/` - Database models (except migrations)
- `/api/extensions/ext_*.py` - Flask extension initialization

## Implementation Hints

### Pattern 1: Tool Provider Implementation
```python
# File: /api/core/tools/builtin_tool/providers/my_provider/my_provider.py
from core.tools.builtin_tool.provider import BuiltinToolProviderController

class MyProviderToolProvider(BuiltinToolProviderController):
    def _validate_credentials(self, user_id: str, credentials: dict[str, Any]) -> None:
        # Credential validation logic
        pass
```

### Pattern 2: YAML Configuration
```yaml
# File: /api/core/tools/builtin_tool/providers/my_provider/my_provider.yaml
identity:
  author: "Developer Name"
  name: "my_provider"
  label:
    en_US: "My Provider"
    id_ID: "Penyedia Saya"
  description:
    en_US: "Custom tool provider"
    id_ID: "Penyedia tool kustom"
```

### Pattern 3: Testing Structure
```python
# File: /api/tests/unit_tests/tools/test_my_provider.py
import pytest
from core.tools.builtin_tool.providers.my_provider.tools.my_tool import MyTool

class TestMyTool:
    def test_invoke_success(self):
        # Test implementation
        pass
```

## Acceptance Criteria

### AC1: Documentation Completeness
- [ ] Semua 9 file dokumentasi dibuat dalam bahasa Indonesia
- [ ] Setiap file memiliki contoh code yang working
- [ ] Step-by-step instructions untuk setiap customization type
- [ ] Troubleshooting section untuk common issues

### AC2: Practical Implementation
- [ ] Programmer junior dapat follow tutorial untuk membuat custom tool
- [ ] Custom tool dapat di-test menggunakan pytest framework
- [ ] Deployment guide dapat digunakan untuk production setup
- [ ] Indonesian language support working untuk custom tools

### AC3: Safety Guidelines
- [ ] Clear distinction antara safe vs unsafe modification areas
- [ ] Warning untuk areas yang tidak boleh dimodifikasi
- [ ] Best practices untuk maintain compatibility dengan Dify updates
- [ ] Security considerations untuk authentication dan tenant isolation

### AC4: Code Quality
- [ ] Semua code examples memiliki proper error handling
- [ ] Type hints dan docstrings dalam code examples
- [ ] Consistent dengan Dify coding standards
- [ ] Working examples yang dapat copy-paste

## Assumptions

### A1: Development Environment
- Programmer menggunakan Docker Compose untuk development (tidak manual database setup)
- Python 3.11+ dan Node.js 22+ tersedia
- Basic knowledge tentang Python, REST API, dan React

### A2: Security Approach
- Custom code menggunakan existing security patterns (tidak modify security layer)
- SSRF protection dan sandbox execution handled otomatis oleh Dify
- Focus pada proper usage pattern, bukan security implementation details

### A3: Scope Limitations
- Tidak mencakup plugin system (terlalu complex untuk junior)
- Tidak mencakup custom workflow nodes (affects core engine)
- Focus pada code-based extensions, bukan full plugin development

### A4: Documentation Maintenance
- Documentation updated ketika ada breaking changes di Dify
- Examples tested dengan current Dify version (1.6.0)
- Indonesian language support maintained sesuai dengan community needs

---

**Requirements Phase Complete**
**Status:** Ready for implementation
**Next Step:** Execute implementation menggunakan /execute-requirements command