# Best Practices

Kumpulan best practices untuk development Dify yang professional, maintainable, dan scalable.

## Code Quality Standards

### 1. Python Backend Standards

#### Type Hints & Documentation
```python
from typing import Any, Optional, Generator, Dict, List
from core.tools.entities.tool_entities import ToolInvokeMessage

class WeatherTool(BuiltinTool):
    """
    Weather information tool using OpenWeatherMap API.
    
    This tool provides current weather data for any city worldwide.
    Requires OpenWeatherMap API key for authentication.
    """
    
    def _invoke(
        self, 
        user_id: str, 
        tool_parameters: Dict[str, Any], 
        conversation_id: Optional[str] = None,
        app_id: Optional[str] = None,
        message_id: Optional[str] = None
    ) -> Generator[ToolInvokeMessage, None, None]:
        """
        Invoke weather tool to get current weather information.
        
        Args:
            user_id: ID of the user invoking the tool
            tool_parameters: Dict containing 'city' and optional 'units'
            conversation_id: Optional conversation context
            app_id: Optional application context
            message_id: Optional message context
            
        Yields:
            ToolInvokeMessage: Weather information or error message
            
        Raises:
            ValueError: If required parameters are missing
            RuntimeError: If API request fails
        """
        # Implementation here
        pass
```

#### Error Handling Patterns
```python
class ToolExecutionError(Exception):
    """Custom exception for tool execution errors"""
    pass

class APIConnectionError(ToolExecutionError):
    """Exception for API connection failures"""
    pass

def _invoke(self, user_id: str, tool_parameters: Dict[str, Any], ...) -> Generator[ToolInvokeMessage, None, None]:
    try:
        # Validate required parameters
        self._validate_parameters(tool_parameters)
        
        # Execute main logic
        result = self._execute_tool_logic(tool_parameters)
        
        yield self.create_text_message(str(result))
        
    except ValueError as e:
        # User input errors
        logger.warning(f"Invalid parameters for {self.__class__.__name__}: {str(e)}")
        yield self.create_text_message(f"Error: {str(e)}")
        
    except APIConnectionError as e:
        # API connectivity issues
        logger.error(f"API connection failed for {self.__class__.__name__}: {str(e)}")
        yield self.create_text_message("Error: Unable to connect to external service. Please try again later.")
        
    except requests.Timeout:
        # Timeout handling
        logger.error(f"Timeout in {self.__class__.__name__}")
        yield self.create_text_message("Error: Request timed out. Please try again.")
        
    except Exception as e:
        # Unexpected errors
        logger.error(f"Unexpected error in {self.__class__.__name__}: {str(e)}", exc_info=True)
        yield self.create_text_message("Error: An unexpected error occurred. Please contact support.")

def _validate_parameters(self, parameters: Dict[str, Any]) -> None:
    """Validate tool parameters with detailed error messages"""
    required_fields = ['city']
    for field in required_fields:
        if not parameters.get(field):
            raise ValueError(f"Parameter '{field}' is required")
    
    # Type validation
    if 'limit' in parameters:
        try:
            limit = int(parameters['limit'])
            if limit <= 0 or limit > 100:
                raise ValueError("Parameter 'limit' must be between 1 and 100")
        except (ValueError, TypeError):
            raise ValueError("Parameter 'limit' must be a valid integer")
```

#### Configuration Management
```python
from pydantic import BaseModel, Field, validator
from typing import Optional

class ToolConfig(BaseModel):
    """Configuration model for tools"""
    api_key: str = Field(..., description="API key for external service")
    base_url: str = Field(default="https://api.example.com", description="Base URL for API")
    timeout: int = Field(default=30, ge=1, le=300, description="Request timeout in seconds")
    retry_count: int = Field(default=3, ge=0, le=10, description="Number of retries")
    cache_ttl: Optional[int] = Field(default=3600, description="Cache TTL in seconds")
    
    @validator('api_key')
    def validate_api_key(cls, v):
        if not v or len(v.strip()) == 0:
            raise ValueError('API key cannot be empty')
        return v.strip()
    
    @validator('base_url')
    def validate_base_url(cls, v):
        if not v.startswith(('http://', 'https://')):
            raise ValueError('Base URL must start with http:// or https://')
        return v.rstrip('/')

# Usage in tool
class MyTool(BuiltinTool):
    def __init__(self):
        super().__init__()
        self._config: Optional[ToolConfig] = None
    
    @property
    def config(self) -> ToolConfig:
        if self._config is None:
            self._config = ToolConfig(**self.credentials)
        return self._config
```

### 2. TypeScript Frontend Standards

#### Component Structure
```typescript
// components/CustomToolConfig.tsx
import React, { useState, useCallback } from 'react'
import { useTranslation } from 'react-i18next'
import type { FC } from 'react'

interface CustomToolConfigProps {
  toolId: string
  initialConfig?: ToolConfiguration
  onConfigChange: (config: ToolConfiguration) => void
  disabled?: boolean
}

interface ToolConfiguration {
  apiKey: string
  baseUrl: string
  timeout: number
}

const CustomToolConfig: FC<CustomToolConfigProps> = ({
  toolId,
  initialConfig,
  onConfigChange,
  disabled = false
}) => {
  const { t } = useTranslation()
  const [config, setConfig] = useState<ToolConfiguration>(
    initialConfig ?? {
      apiKey: '',
      baseUrl: 'https://api.example.com',
      timeout: 30
    }
  )
  
  const handleConfigUpdate = useCallback((updates: Partial<ToolConfiguration>) => {
    const newConfig = { ...config, ...updates }
    setConfig(newConfig)
    onConfigChange(newConfig)
  }, [config, onConfigChange])
  
  return (
    <div className="space-y-4">
      <div>
        <label className="block text-sm font-medium text-gray-700">
          {t('tools.apiKey')}
        </label>
        <input
          type="password"
          value={config.apiKey}
          onChange={(e) => handleConfigUpdate({ apiKey: e.target.value })}
          disabled={disabled}
          className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
          placeholder={t('tools.apiKeyPlaceholder')}
        />
      </div>
      
      <div>
        <label className="block text-sm font-medium text-gray-700">
          {t('tools.baseUrl')}
        </label>
        <input
          type="url"
          value={config.baseUrl}
          onChange={(e) => handleConfigUpdate({ baseUrl: e.target.value })}
          disabled={disabled}
          className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
        />
      </div>
    </div>
  )
}

export default CustomToolConfig
```

#### API Client Patterns
```typescript
// services/customToolApi.ts
import ky from 'ky'

export interface ToolResponse<T = any> {
  success: boolean
  data?: T
  error?: string
  code?: string
}

export interface ToolExecutionRequest {
  toolId: string
  parameters: Record<string, any>
  userId: string
}

class CustomToolApiClient {
  private client = ky.create({
    prefixUrl: '/api/v1/tools',
    timeout: 30000,
    retry: {
      limit: 3,
      methods: ['get', 'post'],
      statusCodes: [408, 413, 429, 500, 502, 503, 504]
    },
    hooks: {
      beforeRequest: [
        request => {
          const token = localStorage.getItem('auth_token')
          if (token) {
            request.headers.set('Authorization', `Bearer ${token}`)
          }
        }
      ],
      afterResponse: [
        async (request, options, response) => {
          if (response.status === 401) {
            // Handle unauthorized - redirect to login
            window.location.href = '/login'
          }
        }
      ]
    }
  })
  
  async executeTool(request: ToolExecutionRequest): Promise<ToolResponse> {
    try {
      const response = await this.client.post('execute', {
        json: request
      }).json<ToolResponse>()
      
      return response
    } catch (error) {
      console.error('Tool execution failed:', error)
      return {
        success: false,
        error: 'Tool execution failed'
      }
    }
  }
  
  async getToolConfig(toolId: string): Promise<ToolResponse<ToolConfiguration>> {
    try {
      return await this.client.get(`config/${toolId}`).json<ToolResponse<ToolConfiguration>>()
    } catch (error) {
      console.error('Failed to get tool config:', error)
      return {
        success: false,
        error: 'Failed to get tool configuration'
      }
    }
  }
}

export const customToolApi = new CustomToolApiClient()
```

## Performance Optimization

### 1. Database Query Optimization

```python
# Efficient querying patterns
from sqlalchemy import and_, or_, func
from sqlalchemy.orm import selectinload, joinedload

class MessageService:
    @staticmethod
    def get_conversation_messages(
        conversation_id: str, 
        limit: int = 50, 
        offset: int = 0
    ) -> List[Message]:
        """
        Get conversation messages with optimized loading
        """
        return db.session.query(Message) \
            .options(
                joinedload(Message.files),  # Load related files in one query
                selectinload(Message.agent_thoughts)  # Efficient loading for collections
            ) \
            .filter(Message.conversation_id == conversation_id) \
            .order_by(Message.created_at.desc()) \
            .limit(limit) \
            .offset(offset) \
            .all()
    
    @staticmethod
    def get_tool_usage_stats(start_date: datetime, end_date: datetime) -> Dict[str, int]:
        """
        Get tool usage statistics with aggregation
        """
        results = db.session.query(
            ToolUsage.tool_name,
            func.count(ToolUsage.id).label('usage_count')
        ) \
        .filter(and_(
            ToolUsage.created_at >= start_date,
            ToolUsage.created_at <= end_date
        )) \
        .group_by(ToolUsage.tool_name) \
        .all()
        
        return {result.tool_name: result.usage_count for result in results}
```

### 2. Caching Strategies

```python
from functools import lru_cache, wraps
from typing import Callable, Any
import redis
import json
import hashlib

# Memory caching for expensive computations
class MemoizedTool:
    @lru_cache(maxsize=1000)
    def _compute_heavy_operation(self, input_data: str) -> str:
        """Cache expensive computations in memory"""
        # Heavy computation here
        return result

# Redis caching for API responses
class CachedAPITool:
    def __init__(self):
        self.redis_client = redis.Redis(
            host=REDIS_HOST,
            password=REDIS_PASSWORD,
            decode_responses=True
        )
    
    def _get_cache_key(self, tool_name: str, parameters: Dict[str, Any]) -> str:
        """Generate consistent cache key"""
        param_str = json.dumps(parameters, sort_keys=True)
        param_hash = hashlib.md5(param_str.encode()).hexdigest()
        return f"tool:{tool_name}:{param_hash}"
    
    def _invoke_with_cache(
        self, 
        tool_name: str, 
        parameters: Dict[str, Any],
        cache_ttl: int = 3600
    ) -> Any:
        cache_key = self._get_cache_key(tool_name, parameters)
        
        # Try to get from cache
        cached_result = self.redis_client.get(cache_key)
        if cached_result:
            return json.loads(cached_result)
        
        # Execute tool and cache result
        result = self._execute_tool(parameters)
        self.redis_client.setex(
            cache_key, 
            cache_ttl, 
            json.dumps(result, default=str)
        )
        
        return result

# Decorator for caching
def cached_tool_result(ttl: int = 3600):
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(self, *args, **kwargs):
            # Generate cache key from function name and arguments
            cache_key = f"{func.__name__}:{hash(str(args) + str(kwargs))}"
            
            # Check cache
            if hasattr(self, 'redis_client'):
                cached = self.redis_client.get(cache_key)
                if cached:
                    return json.loads(cached)
            
            # Execute and cache
            result = func(self, *args, **kwargs)
            if hasattr(self, 'redis_client'):
                self.redis_client.setex(
                    cache_key, 
                    ttl, 
                    json.dumps(result, default=str)
                )
            
            return result
        return wrapper
    return decorator
```

### 3. Async Processing

```python
from celery import Celery
from typing import Dict, Any

# Background task for heavy operations
@celery.task(bind=True, max_retries=3)
def process_document_async(self, document_id: str, processing_config: Dict[str, Any]):
    """
    Process document in background with retry logic
    """
    try:
        # Heavy document processing
        result = DocumentProcessor.process(document_id, processing_config)
        return result
    except Exception as exc:
        # Retry with exponential backoff
        countdown = 2 ** self.request.retries
        raise self.retry(exc=exc, countdown=countdown)

# Tool with async capabilities
class AsyncDocumentTool(BuiltinTool):
    def _invoke(self, user_id: str, tool_parameters: Dict[str, Any], ...) -> Generator[ToolInvokeMessage, None, None]:
        document_id = tool_parameters.get('document_id')
        
        if tool_parameters.get('async', False):
            # Start background task
            task = process_document_async.delay(document_id, tool_parameters)
            
            yield self.create_text_message(
                f"Document processing started. Task ID: {task.id}"
            )
        else:
            # Synchronous processing
            result = DocumentProcessor.process(document_id, tool_parameters)
            yield self.create_text_message(str(result))
```

## Security Best Practices

### 1. Input Validation & Sanitization

```python
import re
from typing import Any, Dict
from html import escape
from urllib.parse import urlparse

class InputValidator:
    @staticmethod
    def validate_email(email: str) -> bool:
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
    
    @staticmethod
    def validate_url(url: str) -> bool:
        try:
            result = urlparse(url)
            return all([result.scheme, result.netloc])
        except Exception:
            return False
    
    @staticmethod
    def sanitize_html(text: str) -> str:
        """Remove HTML tags and escape special characters"""
        # Remove HTML tags
        clean = re.sub(r'<[^>]+>', '', text)
        # Escape special characters
        return escape(clean)
    
    @staticmethod
    def validate_file_upload(file_path: str, allowed_extensions: set) -> bool:
        """Validate file upload security"""
        # Check file extension
        _, ext = os.path.splitext(file_path.lower())
        if ext not in allowed_extensions:
            return False
        
        # Check for path traversal
        if '../' in file_path or '..\\' in file_path:
            return False
        
        return True

class SecureTool(BuiltinTool):
    def _invoke(self, user_id: str, tool_parameters: Dict[str, Any], ...) -> Generator[ToolInvokeMessage, None, None]:
        # Validate and sanitize inputs
        try:
            sanitized_params = self._sanitize_parameters(tool_parameters)
            validated_params = self._validate_parameters(sanitized_params)
        except ValueError as e:
            yield self.create_text_message(f"Invalid input: {str(e)}")
            return
        
        # Process with sanitized inputs
        result = self._execute_safely(validated_params)
        yield self.create_text_message(result)
    
    def _sanitize_parameters(self, params: Dict[str, Any]) -> Dict[str, Any]:
        sanitized = {}
        for key, value in params.items():
            if isinstance(value, str):
                # Sanitize string inputs
                sanitized[key] = InputValidator.sanitize_html(value.strip())
            else:
                sanitized[key] = value
        return sanitized
```

### 2. Credential Management

```python
from cryptography.fernet import Fernet
import os
from typing import Dict, Any

class CredentialManager:
    def __init__(self):
        # Use environment variable for encryption key
        key = os.environ.get('CREDENTIAL_ENCRYPTION_KEY')
        if not key:
            raise ValueError('CREDENTIAL_ENCRYPTION_KEY environment variable required')
        self.cipher = Fernet(key.encode())
    
    def encrypt_credentials(self, credentials: Dict[str, Any]) -> str:
        """Encrypt credentials for secure storage"""
        credential_json = json.dumps(credentials)
        encrypted = self.cipher.encrypt(credential_json.encode())
        return encrypted.decode()
    
    def decrypt_credentials(self, encrypted_credentials: str) -> Dict[str, Any]:
        """Decrypt credentials for use"""
        try:
            decrypted = self.cipher.decrypt(encrypted_credentials.encode())
            return json.loads(decrypted.decode())
        except Exception as e:
            raise ValueError(f"Failed to decrypt credentials: {str(e)}")
    
    def mask_sensitive_data(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Mask sensitive data for logging"""
        sensitive_keys = {'api_key', 'password', 'secret', 'token'}
        masked = {}
        
        for key, value in data.items():
            if any(sensitive in key.lower() for sensitive in sensitive_keys):
                if isinstance(value, str) and len(value) > 4:
                    masked[key] = value[:4] + '*' * (len(value) - 4)
                else:
                    masked[key] = '****'
            else:
                masked[key] = value
                
        return masked
```

## Testing Strategies

### 1. Unit Testing

```python
import pytest
from unittest.mock import Mock, patch, MagicMock
from core.tools.builtin_tool.providers.weather.tools.weather import WeatherTool

class TestWeatherTool:
    @pytest.fixture
    def weather_tool(self):
        tool = WeatherTool()
        tool.credentials = {'api_key': 'test_key'}
        return tool
    
    @pytest.fixture
    def mock_weather_response(self):
        return {
            'name': 'Jakarta',
            'sys': {'country': 'ID'},
            'weather': [{'main': 'Clouds', 'description': 'broken clouds'}],
            'main': {
                'temp': 28.5,
                'feels_like': 32.1,
                'humidity': 78,
                'pressure': 1012
            },
            'wind': {'speed': 3.2, 'deg': 180}
        }
    
    @patch('requests.get')
    def test_successful_weather_request(self, mock_get, weather_tool, mock_weather_response):
        # Setup mock
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = mock_weather_response
        mock_get.return_value = mock_response
        
        # Execute
        result = list(weather_tool._invoke(
            user_id='test_user',
            tool_parameters={'city': 'Jakarta', 'units': 'metric'}
        ))
        
        # Assertions
        assert len(result) == 1
        message = result[0].message
        assert 'Jakarta, ID' in message
        assert '28.5°C' in message
        assert 'broken clouds' in message.lower()
        
        # Verify API call
        mock_get.assert_called_once()
        call_args = mock_get.call_args
        assert 'Jakarta' in str(call_args)
    
    def test_missing_city_parameter(self, weather_tool):
        result = list(weather_tool._invoke(
            user_id='test_user',
            tool_parameters={'units': 'metric'}
        ))
        
        assert len(result) == 1
        assert 'City name is required' in result[0].message
    
    @patch('requests.get')
    def test_api_error_handling(self, mock_get, weather_tool):
        # Setup mock for API error
        mock_response = Mock()
        mock_response.status_code = 404
        mock_get.return_value = mock_response
        
        result = list(weather_tool._invoke(
            user_id='test_user',
            tool_parameters={'city': 'NonexistentCity'}
        ))
        
        assert len(result) == 1
        assert 'not found' in result[0].message.lower()
    
    @patch('requests.get', side_effect=requests.RequestException("Connection failed"))
    def test_network_error_handling(self, mock_get, weather_tool):
        result = list(weather_tool._invoke(
            user_id='test_user',
            tool_parameters={'city': 'Jakarta'}
        ))
        
        assert len(result) == 1
        assert 'Failed to fetch weather data' in result[0].message
```

### 2. Integration Testing

```python
import pytest
from flask import Flask
from core.tools.tool_manager import ToolManager

@pytest.fixture
def app():
    app = Flask(__name__)
    app.config['TESTING'] = True
    return app

@pytest.fixture
def client(app):
    return app.test_client()

class TestToolIntegration:
    def test_tool_registration(self):
        """Test that custom tools are properly registered"""
        manager = ToolManager()
        tools = manager.get_available_tools()
        
        # Check if our custom tool is registered
        custom_tools = [t for t in tools if t.provider == 'weather_provider']
        assert len(custom_tools) > 0
        
        weather_tool = custom_tools[0]
        assert weather_tool.name == 'get_weather'
    
    def test_tool_execution_via_api(self, client):
        """Test tool execution through API endpoint"""
        response = client.post('/api/v1/tools/execute', json={
            'tool_name': 'weather_provider.get_weather',
            'parameters': {
                'city': 'Jakarta',
                'units': 'metric'
            },
            'user_id': 'test_user'
        })
        
        assert response.status_code == 200
        data = response.get_json()
        assert data['success'] is True
        assert 'weather' in data['result'].lower()
```

### 3. Performance Testing

```python
import time
import pytest
from concurrent.futures import ThreadPoolExecutor, as_completed

class TestPerformance:
    def test_tool_execution_time(self, weather_tool):
        """Test that tool execution completes within acceptable time"""
        start_time = time.time()
        
        result = list(weather_tool._invoke(
            user_id='test_user',
            tool_parameters={'city': 'Jakarta'}
        ))
        
        execution_time = time.time() - start_time
        
        assert execution_time < 5.0  # Should complete within 5 seconds
        assert len(result) == 1
    
    def test_concurrent_tool_execution(self, weather_tool):
        """Test tool under concurrent load"""
        def execute_tool():
            return list(weather_tool._invoke(
                user_id='test_user',
                tool_parameters={'city': 'Jakarta'}
            ))
        
        # Execute 10 concurrent requests
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = [executor.submit(execute_tool) for _ in range(10)]
            
            results = []
            for future in as_completed(futures, timeout=30):
                result = future.result()
                results.append(result)
        
        # All requests should succeed
        assert len(results) == 10
        for result in results:
            assert len(result) == 1
            assert 'error' not in result[0].message.lower()
```

## Monitoring & Observability

### 1. Structured Logging

```python
import logging
import json
from typing import Dict, Any
from datetime import datetime

class StructuredLogger:
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
        
    def log_tool_execution(
        self, 
        tool_name: str, 
        user_id: str, 
        parameters: Dict[str, Any],
        execution_time: float,
        success: bool,
        error: str = None
    ):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': 'tool_execution',
            'tool_name': tool_name,
            'user_id': user_id,
            'parameters': self._mask_sensitive_data(parameters),
            'execution_time_ms': round(execution_time * 1000, 2),
            'success': success,
            'error': error
        }
        
        if success:
            self.logger.info(json.dumps(log_data))
        else:
            self.logger.error(json.dumps(log_data))
    
    def _mask_sensitive_data(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Mask sensitive data in logs"""
        sensitive_keys = {'api_key', 'password', 'token', 'secret'}
        masked = {}
        
        for key, value in data.items():
            if any(sensitive in key.lower() for sensitive in sensitive_keys):
                masked[key] = '***masked***'
            else:
                masked[key] = value
                
        return masked

# Usage in tool
class MonitoredTool(BuiltinTool):
    def __init__(self):
        super().__init__()
        self.logger = StructuredLogger(self.__class__.__name__)
    
    def _invoke(self, user_id: str, tool_parameters: Dict[str, Any], ...) -> Generator[ToolInvokeMessage, None, None]:
        start_time = time.time()
        success = False
        error = None
        
        try:
            # Tool execution logic
            result = self._execute_tool(tool_parameters)
            success = True
            yield self.create_text_message(str(result))
            
        except Exception as e:
            error = str(e)
            yield self.create_text_message(f"Error: {error}")
            
        finally:
            execution_time = time.time() - start_time
            self.logger.log_tool_execution(
                tool_name=self.__class__.__name__,
                user_id=user_id,
                parameters=tool_parameters,
                execution_time=execution_time,
                success=success,
                error=error
            )
```

### 2. Metrics Collection

```python
from prometheus_client import Counter, Histogram, Gauge
from typing import Dict, Any

# Define metrics
TOOL_EXECUTIONS = Counter(
    'dify_tool_executions_total',
    'Total tool executions',
    ['tool_name', 'status', 'user_id']
)

TOOL_EXECUTION_TIME = Histogram(
    'dify_tool_execution_duration_seconds',
    'Tool execution duration',
    ['tool_name']
)

ACTIVE_TOOL_EXECUTIONS = Gauge(
    'dify_active_tool_executions',
    'Currently active tool executions'
)

class MetricsCollector:
    @staticmethod
    def record_tool_execution(
        tool_name: str,
        user_id: str, 
        execution_time: float,
        success: bool
    ):
        # Record execution count
        status = 'success' if success else 'error'
        TOOL_EXECUTIONS.labels(
            tool_name=tool_name,
            status=status,
            user_id=user_id
        ).inc()
        
        # Record execution time
        TOOL_EXECUTION_TIME.labels(tool_name=tool_name).observe(execution_time)
    
    @staticmethod
    def track_active_execution(tool_name: str):
        """Context manager to track active executions"""
        class ExecutionTracker:
            def __enter__(self):
                ACTIVE_TOOL_EXECUTIONS.inc()
                return self
            
            def __exit__(self, exc_type, exc_val, exc_tb):
                ACTIVE_TOOL_EXECUTIONS.dec()
        
        return ExecutionTracker()
```

## Documentation Standards

### 1. Code Documentation

```python
class WeatherTool(BuiltinTool):
    """
    Weather information tool using OpenWeatherMap API.
    
    This tool provides current weather data for any city worldwide.
    It supports multiple temperature units and includes detailed
    weather information like humidity, pressure, and wind data.
    
    Features:
        - Current weather for any city
        - Multiple temperature units (Celsius, Fahrenheit, Kelvin)
        - Detailed weather data (humidity, pressure, wind)
        - Error handling for invalid cities
        - Caching for improved performance
    
    Requirements:
        - OpenWeatherMap API key
        - Internet connectivity
    
    Rate Limits:
        - Free tier: 1000 calls/day
        - Paid tier: Based on subscription
    
    Example:
        >>> tool = WeatherTool()
        >>> result = tool._invoke(
        ...     user_id='user123',
        ...     tool_parameters={'city': 'Jakarta', 'units': 'metric'}
        ... )
        >>> print(list(result)[0].message)
        🌤️ Weather in Jakarta, ID
        Current Conditions: Partly Cloudy
        Temperature: 28.5°C (feels like 32.1°C)
        ...
    """
    
    def _invoke(
        self, 
        user_id: str, 
        tool_parameters: Dict[str, Any], 
        conversation_id: Optional[str] = None,
        app_id: Optional[str] = None,
        message_id: Optional[str] = None
    ) -> Generator[ToolInvokeMessage, None, None]:
        """
        Execute weather tool to get current weather information.
        
        Args:
            user_id: Unique identifier for the user
            tool_parameters: Dictionary containing:
                - city (str): Name of the city (required)
                - units (str): Temperature units - 'metric', 'imperial', or 'kelvin' (optional, default: 'metric')
            conversation_id: Optional conversation context
            app_id: Optional application context  
            message_id: Optional message context
            
        Yields:
            ToolInvokeMessage: Weather information formatted as text or error message
            
        Raises:
            ValueError: If required parameters are missing or invalid
            RuntimeError: If API request fails or times out
            
        Note:
            This method yields messages rather than returning them directly
            to support streaming responses in chat interfaces.
        """
        pass
```

### 2. API Documentation

```yaml
# openapi.yaml
components:
  schemas:
    ToolExecutionRequest:
      type: object
      required:
        - tool_name
        - parameters
        - user_id
      properties:
        tool_name:
          type: string
          description: Full name of the tool (provider.tool_name)
          example: "weather_provider.get_weather"
        parameters:
          type: object
          description: Tool-specific parameters
          example:
            city: "Jakarta"
            units: "metric"
        user_id:
          type: string
          description: User identifier
          example: "user_123"
    
    ToolExecutionResponse:
      type: object
      properties:
        success:
          type: boolean
          description: Whether execution was successful
        result:
          type: string
          description: Tool execution result or error message
        execution_time:
          type: number
          description: Execution time in milliseconds
          
paths:
  /api/v1/tools/execute:
    post:
      summary: Execute a custom tool
      description: |
        Execute a custom tool with provided parameters.
        
        This endpoint allows you to invoke any registered custom tool
        and receive the execution result. Tools are executed in a
        sandboxed environment with proper error handling.
        
        **Rate Limiting**: 100 requests per minute per user
        
        **Authentication**: Bearer token required
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ToolExecutionRequest'
      responses:
        '200':
          description: Tool executed successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ToolExecutionResponse'
        '400':
          description: Invalid parameters
        '401':
          description: Authentication required
        '429':
          description: Rate limit exceeded
        '500':
          description: Internal server error
```

## Deployment Automation

### 1. CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy Custom Tools

on:
  push:
    branches: [main]
    paths: ['api/core/tools/builtin_tool/providers/custom/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          
      - name: Install dependencies
        run: |
          cd api
          pip install uv
          uv sync --dev
          
      - name: Run tests
        run: |
          cd api
          uv run pytest tests/unit_tests/tools/ -v --cov=core/tools
          
      - name: Run linting
        run: |
          cd api
          uv run ruff check core/tools/
          uv run mypy core/tools/
          
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to staging
        run: |
          # Copy custom tools to staging
          rsync -av api/core/tools/builtin_tool/providers/custom/ \
            ${{ secrets.STAGING_HOST }}:/app/custom-tools/
          
          # Restart staging services
          ssh ${{ secrets.STAGING_HOST }} 'docker-compose restart api worker'
          
      - name: Run integration tests
        run: |
          # Wait for services to start
          sleep 30
          
          # Run integration tests against staging
          pytest tests/integration_tests/ --host=${{ secrets.STAGING_HOST }}
          
      - name: Deploy to production
        if: success()
        run: |
          # Blue-green deployment to production
          ./scripts/blue-green-deploy.sh custom-tools
```

### 2. Monitoring Setup

```yaml
# monitoring/docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
      
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
      
volumes:
  prometheus-data:
  grafana-data:
```

## Next Steps

1. **[Troubleshooting](./09-troubleshooting.md)** - Debug common issues
2. **Production Monitoring** - Setup comprehensive observability
3. **Advanced Patterns** - Learn complex integration patterns

---

> 💡 **Key Takeaway**: Best practices bukan hanya tentang code quality, tapi juga tentang maintainability, security, dan observability. Implement incrementally dan prioritaskan berdasarkan impact ke production system.