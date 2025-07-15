# Model Providers Integration

Panduan untuk mengintegrasikan custom model providers ke Dify. Topic ini lebih advanced dan cocok untuk programmer yang sudah menguasai custom tools.

## Apa itu Model Providers?

Model providers adalah abstraction layer yang memungkinkan Dify mengakses berbagai AI models:
- 🤖 **LLM Providers** - GPT, Claude, Gemini, local models
- 📝 **Embedding Providers** - Text embeddings untuk RAG
- 🔍 **Rerank Providers** - Document reranking
- 🎤 **Speech Providers** - TTS dan STT

## Arsitektur Model Runtime

```
Model Runtime System
├── Provider Manager
│   ├── Provider Discovery
│   ├── Credential Management
│   └── Load Balancing
├── Model Providers/
│   ├── openai/
│   ├── anthropic/
│   ├── local_model/
│   └── your_provider/  ← Custom provider
└── Base Classes
    ├── LargeLanguageModel
    ├── TextEmbeddingModel
    └── RerankModel
```

## Tutorial: Local Ollama Provider

Mari buat provider untuk local Ollama models.

### Step 1: Provider Structure

```bash
cd api/core/model_runtime/model_providers
mkdir ollama_local
cd ollama_local

# Create directory structure
mkdir llm
mkdir text_embedding
touch provider.py
touch provider.yaml
touch llm/llm.py
touch text_embedding/text_embedding.py
```

### Step 2: Provider Configuration

**File: `provider.yaml`**
```yaml
provider: ollama_local
label:
  en_US: Ollama Local
  id_ID: Ollama Lokal
description:
  en_US: Local Ollama model provider for self-hosted AI models
  id_ID: Provider model Ollama lokal untuk model AI self-hosted
icon_small:
  en_US: icon_s_en.svg
icon_large:
  en_US: icon_l_en.svg
supported_model_types:
  - llm
  - text-embedding
configurator:
  - name: base_url
    type: text-input
    label:
      en_US: Base URL
      id_ID: URL Dasar
    placeholder:
      en_US: http://localhost:11434
      id_ID: http://localhost:11434
    required: true
    help:
      en_US: Ollama server URL
      id_ID: URL server Ollama
provider_credential_schema:
  credential_form_schemas:
    - variable: base_url
      label:
        en_US: Base URL
        id_ID: URL Dasar
      type: text-input
      required: true
      placeholder:
        en_US: http://localhost:11434
        id_ID: http://localhost:11434
```

### Step 3: Provider Implementation

**File: `provider.py`**
```python
import logging
import requests
from typing import Optional
from core.model_runtime.entities.model_entities import ModelType
from core.model_runtime.entities.provider_entities import (
    ProviderEntity,
    SimpleProviderEntity,
    ProviderQuotaType,
    ProviderQuotaUnit,
)
from core.model_runtime.model_providers.__base.model_provider import ModelProvider

logger = logging.getLogger(__name__)

class OllamaLocalProvider(ModelProvider):
    """
    Ollama local model provider
    """
    
    def validate_provider_credentials(self, credentials: dict) -> None:
        """
        Validate provider credentials by testing connection
        """
        base_url = credentials.get('base_url')
        if not base_url:
            raise ValueError('Base URL is required')
        
        # Test connection to Ollama
        try:
            response = requests.get(
                f"{base_url.rstrip('/')}/api/tags",
                timeout=10
            )
            if response.status_code != 200:
                raise ValueError(f'Failed to connect to Ollama: {response.status_code}')
                
            # Check if any models are available
            data = response.json()
            if not data.get('models'):
                logger.warning('No models found on Ollama server')
                
        except requests.RequestException as e:
            raise ValueError(f'Connection failed: {str(e)}')
    
    def get_provider_schema(self) -> ProviderEntity:
        """
        Returns provider schema
        """
        return ProviderEntity(
            provider='ollama_local',
            label={
                'en_US': 'Ollama Local',
                'id_ID': 'Ollama Lokal'
            },
            description={
                'en_US': 'Local Ollama model provider',
                'id_ID': 'Provider model Ollama lokal'
            },
            icon_small={
                'en_US': 'icon_s_en.svg'
            },
            icon_large={
                'en_US': 'icon_l_en.svg'
            },
            supported_model_types=[
                ModelType.LLM,
                ModelType.TEXT_EMBEDDING
            ],
            configurator=[
                {
                    'name': 'base_url',
                    'type': 'text-input',
                    'label': {
                        'en_US': 'Base URL',
                        'id_ID': 'URL Dasar'
                    },
                    'placeholder': {
                        'en_US': 'http://localhost:11434',
                        'id_ID': 'http://localhost:11434'
                    },
                    'required': True
                }
            ]
        )
    
    def get_simple_provider_schema(self) -> SimpleProviderEntity:
        """
        Returns simple provider schema
        """
        return SimpleProviderEntity(
            provider='ollama_local',
            label={
                'en_US': 'Ollama Local',
                'id_ID': 'Ollama Lokal'
            },
            icon_small={
                'en_US': 'icon_s_en.svg'
            },
            supported_model_types=[
                ModelType.LLM,
                ModelType.TEXT_EMBEDDING
            ]
        )
```

### Step 4: LLM Implementation

**File: `llm/llm.py`**
```python
import json
import logging
import requests
from typing import Optional, List, Dict, Any, Generator
from core.model_runtime.entities.llm_entities import LLMResult, LLMResultChunk, LLMResultChunkDelta
from core.model_runtime.entities.message_entities import (
    PromptMessage,
    PromptMessageTool,
    SystemPromptMessage,
    UserPromptMessage,
    AssistantPromptMessage
)
from core.model_runtime.entities.model_entities import (
    ModelPropertyKey,
    ModelType,
    I18nObject
)
from core.model_runtime.model_providers.__base.large_language_model import LargeLanguageModel

logger = logging.getLogger(__name__)

class OllamaLargeLanguageModel(LargeLanguageModel):
    """
    Model class for Ollama LLM
    """
    
    def _invoke(
        self,
        model: str,
        credentials: dict,
        prompt_messages: list[PromptMessage],
        model_parameters: dict,
        tools: Optional[list[PromptMessageTool]] = None,
        stop: Optional[list[str]] = None,
        stream: bool = True,
        user: Optional[str] = None,
    ) -> LLMResult | Generator[LLMResultChunk, None, None]:
        """
        Invoke large language model
        """
        # Prepare request
        base_url = credentials.get('base_url', 'http://localhost:11434')
        
        # Convert messages to Ollama format
        ollama_messages = self._convert_messages(prompt_messages)
        
        # Prepare request data
        data = {
            'model': model,
            'messages': ollama_messages,
            'stream': stream,
            'options': {
                'temperature': model_parameters.get('temperature', 0.7),
                'top_p': model_parameters.get('top_p', 1.0),
                'max_tokens': model_parameters.get('max_tokens', 1024),
            }
        }
        
        if stop:
            data['options']['stop'] = stop
        
        # Make request
        url = f"{base_url.rstrip('/')}/api/chat"
        
        try:
            if stream:
                return self._handle_stream_response(url, data)
            else:
                return self._handle_sync_response(url, data)
                
        except Exception as e:
            raise RuntimeError(f"Ollama request failed: {str(e)}")
    
    def _convert_messages(self, messages: list[PromptMessage]) -> list[dict]:
        """
        Convert Dify messages to Ollama format
        """
        ollama_messages = []
        
        for message in messages:
            if isinstance(message, SystemPromptMessage):
                ollama_messages.append({
                    'role': 'system',
                    'content': message.content
                })
            elif isinstance(message, UserPromptMessage):
                ollama_messages.append({
                    'role': 'user', 
                    'content': message.content
                })
            elif isinstance(message, AssistantPromptMessage):
                ollama_messages.append({
                    'role': 'assistant',
                    'content': message.content
                })
        
        return ollama_messages
    
    def _handle_stream_response(self, url: str, data: dict) -> Generator[LLMResultChunk, None, None]:
        """
        Handle streaming response
        """
        response = requests.post(
            url,
            json=data,
            stream=True,
            timeout=60
        )
        response.raise_for_status()
        
        for line in response.iter_lines():
            if line:
                try:
                    chunk_data = json.loads(line)
                    content = chunk_data.get('message', {}).get('content', '')
                    
                    if content:
                        yield LLMResultChunk(
                            model=data['model'],
                            prompt_messages=data.get('prompt_messages', []),
                            delta=LLMResultChunkDelta(
                                index=0,
                                message=AssistantPromptMessage(content=content)
                            )
                        )
                    
                    # Check if done
                    if chunk_data.get('done'):
                        break
                        
                except json.JSONDecodeError:
                    continue
    
    def _handle_sync_response(self, url: str, data: dict) -> LLMResult:
        """
        Handle synchronous response
        """
        response = requests.post(url, json=data, timeout=60)
        response.raise_for_status()
        
        result = response.json()
        content = result.get('message', {}).get('content', '')
        
        return LLMResult(
            model=data['model'],
            prompt_messages=data.get('prompt_messages', []),
            message=AssistantPromptMessage(content=content),
            usage={
                'prompt_tokens': result.get('prompt_eval_count', 0),
                'completion_tokens': result.get('eval_count', 0),
                'total_tokens': result.get('prompt_eval_count', 0) + result.get('eval_count', 0)
            }
        )
    
    def get_num_tokens(
        self,
        model: str,
        credentials: dict,
        prompt_messages: list[PromptMessage],
        tools: Optional[list[PromptMessageTool]] = None
    ) -> int:
        """
        Get number of tokens in prompt
        Simple estimation for local models
        """
        # Simple token estimation: ~4 chars per token
        total_chars = sum(len(msg.content) for msg in prompt_messages)
        return max(1, total_chars // 4)
    
    def validate_credentials(self, model: str, credentials: dict) -> None:
        """
        Validate model credentials
        """
        base_url = credentials.get('base_url')
        if not base_url:
            raise ValueError('Base URL is required')
        
        # Test if model exists
        try:
            response = requests.get(
                f"{base_url.rstrip('/')}/api/tags",
                timeout=10
            )
            response.raise_for_status()
            
            data = response.json()
            models = [m['name'] for m in data.get('models', [])]
            
            if model not in models:
                available = ', '.join(models) if models else 'None'
                raise ValueError(f'Model {model} not found. Available: {available}')
                
        except requests.RequestException as e:
            raise ValueError(f'Failed to validate model: {str(e)}')
```

## Model Configuration

### Predefined Models

**File: `llm/llama3.yaml`**
```yaml
model: llama3
label:
  en_US: Llama 3
  id_ID: Llama 3
model_type: llm
features:
  - agent-thought
  - vision
model_properties:
  mode: chat
  context_size: 8192
  max_chunks: 1
  file_upload_limit: 10
parameter_rules:
  - name: temperature
    use_template: temperature
  - name: top_p
    use_template: top_p
  - name: max_tokens
    use_template: max_tokens
pricing:
  input: 0.0
  output: 0.0
  unit: "1M"
  currency: USD
```

## Testing Provider

### Unit Tests

**File: `tests/unit_tests/model_providers/test_ollama.py`**
```python
import pytest
from unittest.mock import Mock, patch
from core.model_runtime.model_providers.ollama_local.llm.llm import OllamaLargeLanguageModel
from core.model_runtime.entities.message_entities import UserPromptMessage

class TestOllamaProvider:
    
    @patch('requests.get')
    def test_validate_credentials_success(self, mock_get):
        """Test successful credential validation"""
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            'models': [{'name': 'llama3'}]
        }
        mock_get.return_value = mock_response
        
        model = OllamaLargeLanguageModel()
        # Should not raise exception
        model.validate_credentials('llama3', {'base_url': 'http://localhost:11434'})
    
    @patch('requests.post')
    def test_sync_invoke(self, mock_post):
        """Test synchronous model invocation"""
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            'message': {'content': 'Hello, world!'},
            'prompt_eval_count': 10,
            'eval_count': 5
        }
        mock_post.return_value = mock_response
        
        model = OllamaLargeLanguageModel()
        result = model._invoke(
            model='llama3',
            credentials={'base_url': 'http://localhost:11434'},
            prompt_messages=[UserPromptMessage(content='Hello')],
            model_parameters={'temperature': 0.7},
            stream=False
        )
        
        assert result.message.content == 'Hello, world!'
        assert result.usage['total_tokens'] == 15
```

## Production Considerations

### 1. Error Handling
```python
def _invoke(self, model: str, credentials: dict, ...):
    try:
        # Model invocation logic
        return self._make_request()
    except requests.Timeout:
        raise RuntimeError("Model request timed out")
    except requests.ConnectionError:
        raise RuntimeError("Cannot connect to model server")
    except Exception as e:
        logger.error(f"Model invocation failed: {str(e)}")
        raise RuntimeError(f"Model error: {str(e)}")
```

### 2. Resource Management
```python
class ModelProvider:
    def __init__(self):
        self._session = requests.Session()
        # Configure connection pooling
        adapter = requests.adapters.HTTPAdapter(
            pool_connections=10,
            pool_maxsize=20,
            max_retries=3
        )
        self._session.mount('http://', adapter)
        self._session.mount('https://', adapter)
```

### 3. Monitoring & Logging
```python
import time
import logging

logger = logging.getLogger(__name__)

def _invoke(self, ...):
    start_time = time.time()
    try:
        result = self._make_request()
        
        # Log success metrics
        duration = time.time() - start_time
        logger.info(f"Model invocation successful: {duration:.2f}s")
        
        return result
    except Exception as e:
        logger.error(f"Model invocation failed after {time.time() - start_time:.2f}s: {str(e)}")
        raise
```

## Integration dengan Apps

### Configuration dalam App
```python
# App model configuration
app_model_config = {
    'provider': 'ollama_local',
    'model': 'llama3',
    'credentials': {
        'base_url': 'http://ollama-server:11434'
    },
    'parameters': {
        'temperature': 0.7,
        'max_tokens': 2048,
        'top_p': 0.9
    }
}
```

### Load Balancing
```python
# Multiple Ollama instances
ollama_configs = [
    {'base_url': 'http://ollama-1:11434'},
    {'base_url': 'http://ollama-2:11434'},
    {'base_url': 'http://ollama-3:11434'}
]
```

## Next Steps

1. **[Production Deployment](./07-production-deployment.md)** - Deploy custom providers
2. **[Best Practices](./08-best-practices.md)** - Advanced development patterns
3. **[Troubleshooting](./09-troubleshooting.md)** - Debug provider issues

---

> 💡 **Key Takeaway**: Model provider integration memerlukan pemahaman mendalam tentang AI model APIs dan Dify architecture. Mulai dengan simple local providers sebelum beralih ke complex cloud providers.