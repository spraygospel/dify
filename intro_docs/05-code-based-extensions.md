# Code-based Extensions

Code-based extensions memungkinkan Anda menambahkan functionality ke Dify tanpa kompleksitas plugin system. Tutorial ini fokus pada moderation extensions dan external data tools.

## Apa itu Code-based Extensions?

Code-based extensions adalah cara extend Dify functionality dengan:
- 🛡️ **Moderation Extensions** - Custom content filtering
- 📊 **External Data Tools** - Custom data source integrations
- 🔧 **Simpler Architecture** - Lebih mudah dari plugin system
- 🚀 **Quick Development** - Rapid prototyping dan deployment

## Moderation Extensions

### Konsep Dasar
Moderation extensions memfilter input dan output dari AI untuk:
- Content safety (NSFW, violence, hate speech)
- Business rules (compliance, brand guidelines)
- Custom validation (domain-specific rules)

### Tutorial: Spam Filter Extension

#### Step 1: Create Directory Structure
```bash
cd api/core/moderation
mkdir spam_filter
cd spam_filter
touch __init__.py
touch spam_filter.py
touch __builtin__
```

#### Step 2: Position File
**File: `__builtin__`**
```
5
```

#### Step 3: Implementation
**File: `spam_filter.py`**
```python
import re
from typing import Optional
from core.moderation.base import Moderation, ModerationAction, ModerationInputsResult, ModerationOutputsResult

class SpamFilterModeration(Moderation):
    name: str = "spam_filter"
    
    # Spam keywords database
    SPAM_KEYWORDS = [
        'buy now', 'limited time', 'click here', 'free money',
        'urgent', 'congratulations', 'winner', 'casino',
        'viagra', 'pharmacy', 'weight loss', 'debt free'
    ]
    
    SUSPICIOUS_PATTERNS = [
        r'\b\d{4}[\s-]\d{4}[\s-]\d{4}[\s-]\d{4}\b',  # Credit card
        r'\b\d{3}-\d{2}-\d{4}\b',  # SSN pattern
        r'[A-Z]{2,}\s+[A-Z]{2,}\s+[A-Z]{2,}',  # ALL CAPS SPAM
        r'([!]{2,}|[?]{2,}|[.]{3,})',  # Excessive punctuation
    ]
    
    @classmethod
    def validate_config(cls, tenant_id: str, config: dict) -> None:
        """
        Validate moderation configuration
        """
        if not config:
            raise ValueError("Spam filter config is required")
        
        # Validate sensitivity level
        sensitivity = config.get('sensitivity', 'medium')
        if sensitivity not in ['low', 'medium', 'high']:
            raise ValueError("Sensitivity must be 'low', 'medium', or 'high'")
        
        # Validate custom keywords
        custom_keywords = config.get('custom_keywords', [])
        if not isinstance(custom_keywords, list):
            raise ValueError("Custom keywords must be a list")
    
    def moderation_for_inputs(self, inputs: dict, query: str = "") -> ModerationInputsResult:
        """
        Moderate input messages for spam content
        """
        flagged = False
        flagged_categories = []
        
        # Combine all text to analyze
        text_to_check = query
        for value in inputs.values():
            if isinstance(value, str):
                text_to_check += " " + value
        
        text_to_check = text_to_check.lower().strip()
        
        if not text_to_check:
            return ModerationInputsResult(
                flagged=False,
                action=ModerationAction.DIRECT_OUTPUT,
                inputs=inputs,
                query=query
            )
        
        # Get configuration
        config = self.config or {}
        sensitivity = config.get('sensitivity', 'medium')
        custom_keywords = config.get('custom_keywords', [])
        
        # Check for spam keywords
        all_keywords = self.SPAM_KEYWORDS + custom_keywords
        spam_score = 0
        
        for keyword in all_keywords:
            if keyword.lower() in text_to_check:
                spam_score += 1
                if keyword not in flagged_categories:
                    flagged_categories.append(f"spam_keyword_{keyword}")
        
        # Check suspicious patterns
        for pattern in self.SUSPICIOUS_PATTERNS:
            if re.search(pattern, text_to_check, re.IGNORECASE):
                spam_score += 2
                flagged_categories.append("suspicious_pattern")
        
        # Determine if flagged based on sensitivity
        thresholds = {'low': 3, 'medium': 2, 'high': 1}
        threshold = thresholds.get(sensitivity, 2)
        
        if spam_score >= threshold:
            flagged = True
        
        return ModerationInputsResult(
            flagged=flagged,
            action=ModerationAction.OVERRIDED if flagged else ModerationAction.DIRECT_OUTPUT,
            preset_response="I cannot process requests that appear to be spam or promotional content. Please rephrase your request." if flagged else "",
            inputs=inputs,
            query=query
        )
    
    def moderation_for_outputs(self, text: str) -> ModerationOutputsResult:
        """
        Moderate output text for spam content
        """
        if not text:
            return ModerationOutputsResult(
                flagged=False,
                action=ModerationAction.DIRECT_OUTPUT,
                text=text
            )
        
        text_lower = text.lower()
        
        # Check for spam in outputs (less strict)
        spam_score = 0
        for keyword in self.SPAM_KEYWORDS[:5]:  # Only check top spam keywords
            if keyword in text_lower:
                spam_score += 1
        
        # Check for suspicious patterns in output
        for pattern in self.SUSPICIOUS_PATTERNS:
            if re.search(pattern, text, re.IGNORECASE):
                spam_score += 2
        
        flagged = spam_score >= 2
        
        return ModerationOutputsResult(
            flagged=flagged,
            action=ModerationAction.OVERRIDED if flagged else ModerationAction.DIRECT_OUTPUT,
            text="I apologize, but I cannot provide that response as it may contain inappropriate promotional content." if flagged else text
        )
```

#### Step 4: Testing
**File: `tests/unit_tests/moderation/test_spam_filter.py`**
```python
import pytest
from core.moderation.spam_filter.spam_filter import SpamFilterModeration
from core.moderation.base import ModerationAction

class TestSpamFilterModeration:
    def test_spam_keyword_detection(self):
        moderation = SpamFilterModeration()
        moderation.config = {'sensitivity': 'medium'}
        
        result = moderation.moderation_for_inputs(
            inputs={},
            query="Click here to buy now and get free money!"
        )
        
        assert result.flagged is True
        assert result.action == ModerationAction.OVERRIDED
    
    def test_clean_input(self):
        moderation = SpamFilterModeration()
        moderation.config = {'sensitivity': 'medium'}
        
        result = moderation.moderation_for_inputs(
            inputs={},
            query="What is the weather like today?"
        )
        
        assert result.flagged is False
        assert result.action == ModerationAction.DIRECT_OUTPUT
```

## External Data Tools

### Tutorial: Database Connector

#### Step 1: Structure
```bash
cd api/core/external_data_tool
mkdir database_connector
cd database_connector
touch __init__.py
touch database_connector.py
```

#### Step 2: Implementation
**File: `database_connector.py`**
```python
import sqlite3
from typing import Optional
from core.external_data_tool.base import ExternalDataTool

class DatabaseConnector(ExternalDataTool):
    """
    External data tool untuk mengakses SQLite database
    """
    
    def validate_config(self, config: dict) -> None:
        """
        Validate database configuration
        """
        required_fields = ['database_path']
        for field in required_fields:
            if field not in config:
                raise ValueError(f"Missing required field: {field}")
        
        # Test database connection
        try:
            conn = sqlite3.connect(config['database_path'])
            conn.close()
        except Exception as e:
            raise ValueError(f"Cannot connect to database: {str(e)}")
    
    def query(self, query: str, config: dict) -> dict:
        """
        Execute database query
        """
        try:
            database_path = config['database_path']
            
            # Security: Only allow SELECT statements
            if not query.strip().upper().startswith('SELECT'):
                raise ValueError("Only SELECT queries are allowed")
            
            conn = sqlite3.connect(database_path)
            conn.row_factory = sqlite3.Row  # Enable column access by name
            cursor = conn.cursor()
            
            cursor.execute(query)
            rows = cursor.fetchall()
            
            # Convert to list of dictionaries
            results = [dict(row) for row in rows]
            
            conn.close()
            
            return {
                'status': 'success',
                'data': results,
                'count': len(results)
            }
            
        except Exception as e:
            return {
                'status': 'error',
                'error': str(e),
                'data': []
            }
```

## Integration dengan Workflow

### Moderation Integration
```python
# Di app configuration
app_config = {
    'moderation_config': {
        'enabled': True,
        'type': 'spam_filter',
        'config': {
            'sensitivity': 'high',
            'custom_keywords': ['custom_spam_word']
        }
    }
}
```

### External Data Integration
```python
# Di workflow node
data_tool_config = {
    'type': 'database_connector',
    'config': {
        'database_path': '/path/to/database.db'
    },
    'query': 'SELECT * FROM users WHERE active = 1'
}
```

## Best Practices

### 1. Error Handling
```python
def moderation_for_inputs(self, inputs: dict, query: str = "") -> ModerationInputsResult:
    try:
        # Main moderation logic
        result = self.analyze_content(query)
        return ModerationInputsResult(flagged=result.is_flagged)
    except Exception as e:
        # Log error but don't break the flow
        logger.error(f"Moderation error: {str(e)}")
        # Return safe default (allow content to pass)
        return ModerationInputsResult(flagged=False)
```

### 2. Performance Optimization
```python
# Cache heavy computations
from functools import lru_cache

class MyModeration(Moderation):
    @lru_cache(maxsize=1000)
    def _analyze_text_cached(self, text: str) -> bool:
        # Expensive analysis here
        return self._heavy_analysis(text)
```

### 3. Configuration Validation
```python
@classmethod
def validate_config(cls, tenant_id: str, config: dict) -> None:
    # Validate all required fields
    required = ['api_key', 'endpoint']
    for field in required:
        if not config.get(field):
            raise ValueError(f"{field} is required")
    
    # Validate data types
    if not isinstance(config.get('timeout', 30), int):
        raise ValueError("timeout must be an integer")
```

## Next Steps

1. **[Model Providers](./06-model-providers.md)** - Advanced provider integration
2. **[Production Deployment](./07-production-deployment.md)** - Deploy extensions safely
3. **[Best Practices](./08-best-practices.md)** - Professional development patterns

---

> 💡 **Key Takeaway**: Code-based extensions lebih sederhana dari plugin system tapi tetap powerful. Fokus pada moderation untuk content safety dan external data tools untuk custom integrations.