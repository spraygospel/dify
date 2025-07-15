# Membuat Custom Tools

Tutorial step-by-step untuk membuat custom tools di Dify. Tools adalah cara paling aman dan umum untuk extend functionality AI agent Anda.

## Apa itu Custom Tools?

Custom tools memungkinkan AI agent Anda:
- 🌐 **Mengakses API external** (weather, news, database)
- 🔧 **Menjalankan operasi khusus** (calculations, data processing)
- 📊 **Mengintegrasikan dengan sistem internal** (CRM, ERP, monitoring)
- 🤖 **Extend capabilities** tanpa modify core Dify

## Arsitektur Tool System

```
Tool Provider (Container)
├── Provider Configuration (YAML)
├── Provider Logic (Python)
└── Tools/
    ├── Tool 1 (YAML + Python)
    ├── Tool 2 (YAML + Python)
    └── Tool N (YAML + Python)
```

**Analogi:** Tool Provider seperti "aplikasi", dan Tools seperti "fitur" dalam aplikasi tersebut.

## Tutorial 1: Tool Sederhana - Random Quote Generator

Mari kita buat tool pertama yang menghasilkan quote random.

### Step 1: Buat Structure Directory

```bash
# Masuk ke directory tools
cd api/core/tools/builtin_tool/providers

# Buat directory untuk provider baru
mkdir random_quote_provider
cd random_quote_provider

# Buat structure file
mkdir tools
mkdir _assets
touch random_quote_provider.py
touch random_quote_provider.yaml
touch tools/get_random_quote.py
touch tools/get_random_quote.yaml
```

**Struktur yang terbentuk:**
```
random_quote_provider/
├── _assets/
├── tools/
│   ├── get_random_quote.py
│   └── get_random_quote.yaml
├── random_quote_provider.py
└── random_quote_provider.yaml
```

### Step 2: Provider Configuration (YAML)

**File: `random_quote_provider.yaml`**
```yaml
identity:
  author: "Your Name"
  name: "random_quote_provider"
  label:
    en_US: "Random Quote Provider"
    id_ID: "Penyedia Quote Random"
  description:
    en_US: "Generate random inspirational quotes"
    id_ID: "Menghasilkan quote inspiratif secara random"
  icon: "quote.svg"
  tags:
    - utilities
    - motivation
    - content
```

### Step 3: Provider Implementation (Python)

**File: `random_quote_provider.py`**
```python
from typing import Any
from core.tools.builtin_tool.provider import BuiltinToolProviderController


class RandomQuoteProviderToolProvider(BuiltinToolProviderController):
    """
    Provider untuk Random Quote tools
    """
    
    def _validate_credentials(self, user_id: str, credentials: dict[str, Any]) -> None:
        """
        Validasi credentials untuk provider ini.
        Untuk tool sederhana, tidak perlu credentials khusus.
        """
        # Untuk tool yang tidak perlu API key, validation bisa kosong
        pass
    
    def _get_provider_description(self) -> dict[str, str]:
        """
        Mengembalikan deskripsi provider
        """
        return {
            "en_US": "Random Quote Provider - Generate inspirational quotes",
            "id_ID": "Penyedia Quote Random - Menghasilkan quote inspiratif"
        }
```

### Step 4: Tool Configuration (YAML)

**File: `tools/get_random_quote.yaml`**
```yaml
identity:
  name: get_random_quote
  author: "Your Name"
  label:
    en_US: "Get Random Quote"
    id_ID: "Dapatkan Quote Random"
description:
  human:
    en_US: "Generate a random inspirational quote with optional category filter"
    id_ID: "Menghasilkan quote inspiratif random dengan filter kategori opsional"
  llm: "Use this tool to get inspirational quotes. You can specify a category like 'motivation', 'success', or 'wisdom', or leave it empty for any category."

parameters:
  - name: category
    type: select
    required: false
    label:
      en_US: "Quote Category"
      id_ID: "Kategori Quote"
    human_description:
      en_US: "Category of quote to generate"
      id_ID: "Kategori quote yang akan dihasilkan"
    form: form
    default: "any"
    options:
      - value: "any"
        label:
          en_US: "Any Category"
          id_ID: "Kategori Apapun"
      - value: "motivation"
        label:
          en_US: "Motivation"
          id_ID: "Motivasi"
      - value: "success"
        label:
          en_US: "Success"
          id_ID: "Sukses"
      - value: "wisdom"
        label:
          en_US: "Wisdom"
          id_ID: "Kebijaksanaan"
      - value: "life"
        label:
          en_US: "Life"
          id_ID: "Kehidupan"

  - name: include_author
    type: boolean
    required: false
    label:
      en_US: "Include Author"
      id_ID: "Sertakan Penulis"
    human_description:
      en_US: "Whether to include the author name with the quote"
      id_ID: "Apakah menyertakan nama penulis dengan quote"
    form: form
    default: true
```

### Step 5: Tool Implementation (Python)

**File: `tools/get_random_quote.py`**
```python
import random
from typing import Any, Optional, Generator
from core.tools.builtin_tool.tool import BuiltinTool
from core.tools.entities.tool_entities import ToolInvokeMessage


class GetRandomQuoteTool(BuiltinTool):
    """
    Tool untuk menghasilkan quote random
    """
    
    # Database quote sederhana
    QUOTES_DATABASE = {
        "motivation": [
            {
                "text": "The only way to do great work is to love what you do.",
                "author": "Steve Jobs"
            },
            {
                "text": "Innovation distinguishes between a leader and a follower.",
                "author": "Steve Jobs"
            },
            {
                "text": "Your limitation—it's only your imagination.",
                "author": "Unknown"
            },
            {
                "text": "Success is not final, failure is not fatal.",
                "author": "Winston Churchill"
            }
        ],
        "success": [
            {
                "text": "Success is walking from failure to failure with no loss of enthusiasm.",
                "author": "Winston Churchill"
            },
            {
                "text": "The road to success and the road to failure are almost exactly the same.",
                "author": "Colin R. Davis"
            },
            {
                "text": "Success is not how high you have climbed, but how you make a positive difference to the world.",
                "author": "Roy T. Bennett"
            }
        ],
        "wisdom": [
            {
                "text": "The only true wisdom is in knowing you know nothing.",
                "author": "Socrates"
            },
            {
                "text": "In the middle of difficulty lies opportunity.",
                "author": "Albert Einstein"
            },
            {
                "text": "It is during our darkest moments that we must focus to see the light.",
                "author": "Aristotle"
            }
        ],
        "life": [
            {
                "text": "Life is what happens to you while you're busy making other plans.",
                "author": "John Lennon"
            },
            {
                "text": "The purpose of our lives is to be happy.",
                "author": "Dalai Lama"
            },
            {
                "text": "Life is really simple, but we insist on making it complicated.",
                "author": "Confucius"
            }
        ]
    }
    
    def _invoke(
        self, 
        user_id: str, 
        tool_parameters: dict[str, Any], 
        conversation_id: Optional[str] = None, 
        app_id: Optional[str] = None,
        message_id: Optional[str] = None
    ) -> Generator[ToolInvokeMessage, None, None]:
        """
        Invoke the random quote tool
        
        Args:
            user_id: ID of the user invoking the tool
            tool_parameters: Parameters passed to the tool
            conversation_id: ID of the conversation (optional)
            app_id: ID of the app (optional)
            message_id: ID of the message (optional)
            
        Returns:
            Generator yielding ToolInvokeMessage objects
        """
        try:
            # Extract parameters dengan default values
            category = tool_parameters.get('category', 'any')
            include_author = tool_parameters.get('include_author', True)
            
            # Validate category
            if category not in ['any'] + list(self.QUOTES_DATABASE.keys()):
                yield self.create_text_message(
                    f"Error: Invalid category '{category}'. Available categories: any, motivation, success, wisdom, life"
                )
                return
            
            # Select quotes based on category
            if category == 'any':
                # Combine all quotes from all categories
                all_quotes = []
                for quotes_list in self.QUOTES_DATABASE.values():
                    all_quotes.extend(quotes_list)
                selected_quotes = all_quotes
            else:
                selected_quotes = self.QUOTES_DATABASE[category]
            
            # Check if quotes available
            if not selected_quotes:
                yield self.create_text_message("Error: No quotes available for the selected category")
                return
            
            # Select random quote
            quote = random.choice(selected_quotes)
            
            # Format output
            if include_author and quote.get('author'):
                formatted_quote = f'"{quote["text"]}" - {quote["author"]}'
            else:
                formatted_quote = f'"{quote["text"]}"'
            
            # Add category info if specific category was requested
            if category != 'any':
                formatted_quote += f"\n\nCategory: {category.title()}"
            
            yield self.create_text_message(formatted_quote)
            
        except Exception as e:
            # Error handling
            yield self.create_text_message(f"Error generating quote: {str(e)}")
```

### Step 6: Icon (Opsional)

**File: `_assets/quote.svg`**
```svg
<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
  <path d="M3 21c3 0 7-1 7-8V5c0-1.25-.756-2.017-2-2H4c-1.25 0-2 .75-2 1.972V11c0 1.25.75 2 2 2 1 0 1 0 1 1v1c0 1-1 2-2 2s-1 .008-1 1.031V20c0 1 0 1 1 1z"/>
  <path d="M15 21c3 0 7-1 7-8V5c0-1.25-.757-2.017-2-2h-4c-1.25 0-2 .75-2 1.972V11c0 1.25.75 2 2 2h.75c0 2.25.25 4-2.75 4v3c0 1 0 1 1 1z"/>
</svg>
```

### Step 7: Testing Tool

**File: `tests/unit_tests/tools/test_random_quote.py`**
```python
import pytest
from unittest.mock import patch
from core.tools.builtin_tool.providers.random_quote_provider.tools.get_random_quote import GetRandomQuoteTool


class TestGetRandomQuoteTool:
    """Test cases untuk Random Quote Tool"""
    
    def test_get_random_quote_any_category(self):
        """Test generating quote from any category"""
        tool = GetRandomQuoteTool()
        
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"category": "any", "include_author": True}
        ))
        
        assert len(result) == 1
        message = result[0].message
        assert '"' in message  # Quote should be in quotes
        assert ' - ' in message  # Should include author
    
    def test_get_random_quote_specific_category(self):
        """Test generating quote from specific category"""
        tool = GetRandomQuoteTool()
        
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"category": "motivation", "include_author": True}
        ))
        
        assert len(result) == 1
        message = result[0].message
        assert "Category: Motivation" in message
    
    def test_get_random_quote_without_author(self):
        """Test generating quote without author"""
        tool = GetRandomQuoteTool()
        
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"category": "wisdom", "include_author": False}
        ))
        
        assert len(result) == 1
        message = result[0].message
        assert '"' in message
        assert ' - ' not in message  # Should not include author
    
    def test_invalid_category(self):
        """Test error handling for invalid category"""
        tool = GetRandomQuoteTool()
        
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"category": "invalid_category"}
        ))
        
        assert len(result) == 1
        message = result[0].message
        assert "Error: Invalid category" in message
    
    @patch('random.choice')
    def test_consistent_output_format(self, mock_choice):
        """Test output format consistency"""
        tool = GetRandomQuoteTool()
        
        # Mock random.choice to return predictable result
        mock_choice.return_value = {
            "text": "Test quote",
            "author": "Test Author"
        }
        
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"category": "motivation", "include_author": True}
        ))
        
        expected_message = '"Test quote" - Test Author\n\nCategory: Motivation'
        assert result[0].message == expected_message
```

### Step 8: Menjalankan Test

```bash
# Masuk ke directory API
cd api

# Run test untuk tool kita
uv run pytest tests/unit_tests/tools/test_random_quote.py -v

# Expected output:
# tests/unit_tests/tools/test_random_quote.py::TestGetRandomQuoteTool::test_get_random_quote_any_category PASSED
# tests/unit_tests/tools/test_random_quote.py::TestGetRandomQuoteTool::test_get_random_quote_specific_category PASSED
# tests/unit_tests/tools/test_random_quote.py::TestGetRandomQuoteTool::test_get_random_quote_without_author PASSED
# tests/unit_tests/tools/test_random_quote.py::TestGetRandomQuoteTool::test_invalid_category PASSED
# tests/unit_tests/tools/test_random_quote.py::TestGetRandomQuoteTool::test_consistent_output_format PASSED
```

### Step 9: Restart Dify untuk Load Tool

```bash
# Restart API service untuk load tool baru
cd docker
docker compose restart api worker

# Check logs untuk memastikan tool loaded
docker compose logs api | grep "random_quote"
```

### Step 10: Test di Dashboard

1. **Buka Dify Dashboard** - `http://localhost`
2. **Masuk ke Workflow Designer** 
3. **Tambah Tool Node** - Pilih "Random Quote Provider"
4. **Configure Tool** - Set parameters (category, include_author)
5. **Test Run** - Execute workflow dan lihat hasil

## Tutorial 2: Tool dengan API External - Weather Tool

Sekarang mari buat tool yang lebih complex dengan integrasi API external.

### Step 1: Setup Provider Structure

```bash
cd api/core/tools/builtin_tool/providers
mkdir weather_provider
cd weather_provider

mkdir tools
mkdir _assets
touch weather_provider.py
touch weather_provider.yaml
touch tools/get_weather.py
touch tools/get_weather.yaml
```

### Step 2: Provider Configuration dengan Credentials

**File: `weather_provider.yaml`**
```yaml
identity:
  author: "Your Name"
  name: "weather_provider"
  label:
    en_US: "Weather Provider"
    id_ID: "Penyedia Cuaca"
  description:
    en_US: "Get current weather and forecast information"
    id_ID: "Dapatkan informasi cuaca terkini dan prediksi"
  icon: "weather.svg"
  tags:
    - weather
    - information
    - api

credentials_for_provider:
  api_key:
    type: secret-input
    required: true
    label:
      en_US: "OpenWeatherMap API Key"
      id_ID: "API Key OpenWeatherMap"
    placeholder:
      en_US: "Enter your OpenWeatherMap API key"
      id_ID: "Masukkan API key OpenWeatherMap Anda"
    help:
      en_US: "Get your free API key from https://openweathermap.org/api"
      id_ID: "Dapatkan API key gratis dari https://openweathermap.org/api"
```

### Step 3: Provider Implementation dengan Credential Validation

**File: `weather_provider.py`**
```python
import requests
from typing import Any
from core.tools.builtin_tool.provider import BuiltinToolProviderController


class WeatherProviderToolProvider(BuiltinToolProviderController):
    """
    Provider untuk Weather tools dengan OpenWeatherMap API
    """
    
    def _validate_credentials(self, user_id: str, credentials: dict[str, Any]) -> None:
        """
        Validasi API key dengan test request ke OpenWeatherMap
        """
        api_key = credentials.get('api_key')
        if not api_key:
            raise ValueError("API key is required")
        
        # Test API key dengan request simple
        test_url = "http://api.openweathermap.org/data/2.5/weather"
        test_params = {
            'q': 'London',  # Test city
            'appid': api_key,
            'units': 'metric'
        }
        
        try:
            response = requests.get(test_url, params=test_params, timeout=10)
            if response.status_code == 401:
                raise ValueError("Invalid API key")
            elif response.status_code != 200:
                raise ValueError(f"API validation failed: {response.status_code}")
        except requests.RequestException as e:
            raise ValueError(f"Failed to validate API key: {str(e)}")
```

### Step 4: Tool Configuration

**File: `tools/get_weather.yaml`**
```yaml
identity:
  name: get_weather
  author: "Your Name"  
  label:
    en_US: "Get Weather"
    id_ID: "Dapatkan Cuaca"

description:
  human:
    en_US: "Get current weather information for any city"
    id_ID: "Dapatkan informasi cuaca terkini untuk kota manapun"
  llm: "Use this tool to get current weather information including temperature, humidity, wind speed, and weather conditions for any city worldwide."

parameters:
  - name: city
    type: string
    required: true
    label:
      en_US: "City Name"
      id_ID: "Nama Kota"
    human_description:
      en_US: "Name of the city to get weather for"
      id_ID: "Nama kota untuk mendapatkan info cuaca"
    form: llm
    
  - name: units
    type: select
    required: false
    label:
      en_US: "Temperature Units"
      id_ID: "Satuan Suhu"
    human_description:
      en_US: "Temperature units to use"
      id_ID: "Satuan suhu yang digunakan"
    form: form
    default: "metric"
    options:
      - value: "metric"
        label:
          en_US: "Celsius"
          id_ID: "Celsius"
      - value: "imperial"
        label:
          en_US: "Fahrenheit"
          id_ID: "Fahrenheit"
      - value: "kelvin"
        label:
          en_US: "Kelvin"
          id_ID: "Kelvin"
```

### Step 5: Tool Implementation dengan Error Handling

**File: `tools/get_weather.py`**
```python
import requests
from typing import Any, Optional, Generator
from core.tools.builtin_tool.tool import BuiltinTool
from core.tools.entities.tool_entities import ToolInvokeMessage


class GetWeatherTool(BuiltinTool):
    """
    Tool untuk mendapatkan informasi cuaca dari OpenWeatherMap API
    """
    
    OPENWEATHER_BASE_URL = "http://api.openweathermap.org/data/2.5/weather"
    
    def _invoke(
        self, 
        user_id: str, 
        tool_parameters: dict[str, Any], 
        conversation_id: Optional[str] = None, 
        app_id: Optional[str] = None,
        message_id: Optional[str] = None
    ) -> Generator[ToolInvokeMessage, None, None]:
        """
        Get weather information for specified city
        """
        try:
            # Extract parameters
            city = tool_parameters.get('city', '').strip()
            units = tool_parameters.get('units', 'metric')
            
            # Validate required parameters
            if not city:
                yield self.create_text_message("Error: City name is required")
                return
            
            # Get API key from credentials
            api_key = self.credentials.get('api_key')
            if not api_key:
                yield self.create_text_message("Error: API key not configured")
                return
            
            # Prepare API request
            params = {
                'q': city,
                'appid': api_key,
                'units': units
            }
            
            # Make API request
            response = requests.get(
                self.OPENWEATHER_BASE_URL, 
                params=params, 
                timeout=30
            )
            
            # Handle API errors
            if response.status_code == 404:
                yield self.create_text_message(f"Error: City '{city}' not found")
                return
            elif response.status_code == 401:
                yield self.create_text_message("Error: Invalid API key")
                return
            elif response.status_code != 200:
                yield self.create_text_message(
                    f"Error: Weather API returned {response.status_code}: {response.text}"
                )
                return
            
            # Parse response
            data = response.json()
            
            # Extract weather information
            weather_info = self._format_weather_response(data, units)
            
            yield self.create_text_message(weather_info)
            
        except requests.RequestException as e:
            yield self.create_text_message(f"Error: Failed to fetch weather data: {str(e)}")
        except Exception as e:
            yield self.create_text_message(f"Error: Unexpected error occurred: {str(e)}")
    
    def _format_weather_response(self, data: dict, units: str) -> str:
        """
        Format weather data into readable text
        """
        try:
            # Extract basic info
            city_name = data['name']
            country = data['sys']['country']
            
            # Weather details
            weather = data['weather'][0]
            main_weather = weather['main']
            description = weather['description'].title()
            
            # Temperature and feels like
            main = data['main']
            temp = main['temp']
            feels_like = main['feels_like']
            humidity = main['humidity']
            pressure = main['pressure']
            
            # Wind
            wind = data.get('wind', {})
            wind_speed = wind.get('speed', 0)
            wind_deg = wind.get('deg', 0)
            
            # Determine temperature unit symbol
            if units == 'metric':
                temp_unit = '°C'
                speed_unit = 'm/s'
            elif units == 'imperial':
                temp_unit = '°F'
                speed_unit = 'mph'
            else:  # kelvin
                temp_unit = 'K'
                speed_unit = 'm/s'
            
            # Format response
            weather_report = f"""🌤️ **Weather in {city_name}, {country}**

**Current Conditions:** {description}
**Temperature:** {temp}{temp_unit} (feels like {feels_like}{temp_unit})
**Humidity:** {humidity}%
**Pressure:** {pressure} hPa
**Wind:** {wind_speed} {speed_unit}"""
            
            # Add wind direction if available
            if wind_deg:
                direction = self._get_wind_direction(wind_deg)
                weather_report += f" {direction}"
            
            return weather_report
            
        except KeyError as e:
            return f"Error: Missing data in weather response: {str(e)}"
    
    def _get_wind_direction(self, degrees: float) -> str:
        """
        Convert wind degree to compass direction
        """
        directions = [
            "N", "NNE", "NE", "ENE", "E", "ESE", "SE", "SSE",
            "S", "SSW", "SW", "WSW", "W", "WNW", "NW", "NNW"
        ]
        
        # Normalize degree and calculate index
        degree = (degrees + 11.25) % 360
        index = int(degree / 22.5)
        
        return f"({directions[index]})"
```

### Step 6: Testing dengan Mocking

**File: `tests/unit_tests/tools/test_weather.py`**
```python
import pytest
from unittest.mock import patch, Mock
from core.tools.builtin_tool.providers.weather_provider.tools.get_weather import GetWeatherTool


class TestGetWeatherTool:
    """Test cases untuk Weather Tool"""
    
    @patch('requests.get')
    def test_get_weather_success(self, mock_get):
        """Test successful weather request"""
        # Mock API response
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
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
        mock_get.return_value = mock_response
        
        # Create tool with mock credentials
        tool = GetWeatherTool()
        tool.credentials = {'api_key': 'test_api_key'}
        
        # Test invocation
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"city": "Jakarta", "units": "metric"}
        ))
        
        assert len(result) == 1
        message = result[0].message
        assert "Jakarta, ID" in message
        assert "28.5°C" in message
        assert "broken clouds" in message.lower()
    
    def test_missing_city_parameter(self):
        """Test error handling for missing city"""
        tool = GetWeatherTool()
        tool.credentials = {'api_key': 'test_api_key'}
        
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"units": "metric"}
        ))
        
        assert len(result) == 1
        assert "Error: City name is required" in result[0].message
    
    @patch('requests.get')
    def test_city_not_found(self, mock_get):
        """Test handling of city not found"""
        mock_response = Mock()
        mock_response.status_code = 404
        mock_get.return_value = mock_response
        
        tool = GetWeatherTool()
        tool.credentials = {'api_key': 'test_api_key'}
        
        result = list(tool._invoke(
            user_id="test-user",
            tool_parameters={"city": "NonexistentCity"}
        ))
        
        assert len(result) == 1
        assert "City 'NonexistentCity' not found" in result[0].message
    
    def test_wind_direction_calculation(self):
        """Test wind direction calculation"""
        tool = GetWeatherTool()
        
        # Test different wind directions
        assert tool._get_wind_direction(0) == "(N)"
        assert tool._get_wind_direction(90) == "(E)"
        assert tool._get_wind_direction(180) == "(S)"
        assert tool._get_wind_direction(270) == "(W)"
        assert tool._get_wind_direction(45) == "(NE)"
```

## Menggunakan Tool di Workflow

### 1. Melalui Dashboard

1. **Buka Workflow Designer**
2. **Drag Tool Node** ke canvas
3. **Select Provider** - Pilih "Weather Provider" atau "Random Quote Provider"
4. **Configure Parameters** - Set input parameters
5. **Connect Nodes** - Hubungkan dengan input/output nodes
6. **Test Run** - Execute dan verify hasil

### 2. Melalui API

```bash
# Test weather tool via API
curl -X POST "http://localhost:5001/v1/workflows/run" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": {
      "city": "Jakarta"
    },
    "query": "What is the weather like?",
    "response_mode": "blocking",
    "user": "test-user"
  }'
```

## Best Practices untuk Tool Development

### 1. Error Handling yang Robust

```python
def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
    try:
        # Main logic
        result = self.do_something()
        yield self.create_text_message(str(result))
        
    except requests.Timeout:
        yield self.create_text_message("Error: Request timed out. Please try again.")
    except requests.ConnectionError:
        yield self.create_text_message("Error: Unable to connect to external service.")
    except ValueError as e:
        yield self.create_text_message(f"Error: Invalid parameter - {str(e)}")
    except Exception as e:
        yield self.create_text_message(f"Error: Unexpected error - {str(e)}")
```

### 2. Credential Security

```python
# GOOD: Use credentials from provider
api_key = self.credentials.get('api_key')

# BAD: Hard-code credentials
api_key = "hardcoded-secret-key"  # NEVER DO THIS
```

### 3. Input Validation

```python
def _invoke(self, user_id: str, tool_parameters: dict[str, Any], ...):
    # Validate required parameters
    required_params = ['city', 'country']
    for param in required_params:
        if not tool_parameters.get(param):
            yield self.create_text_message(f"Error: {param} is required")
            return
    
    # Validate parameter types
    try:
        limit = int(tool_parameters.get('limit', 10))
        if limit <= 0 or limit > 100:
            yield self.create_text_message("Error: limit must be between 1 and 100")
            return
    except ValueError:
        yield self.create_text_message("Error: limit must be a valid number")
        return
```

### 4. Response Formatting

```python
def _format_response(self, data: dict) -> str:
    """Format API response into readable text"""
    try:
        # Extract key information
        title = data.get('title', 'Unknown')
        description = data.get('description', 'No description')
        
        # Create formatted output
        response = f"**{title}**\n\n{description}"
        
        # Add optional fields
        if 'author' in data:
            response += f"\n\nAuthor: {data['author']}"
        
        return response
        
    except Exception as e:
        return f"Error formatting response: {str(e)}"
```

## Troubleshooting

### Tool Tidak Muncul di Dashboard

```bash
# 1. Check file structure
ls -la api/core/tools/builtin_tool/providers/your_provider/

# 2. Check YAML syntax
cd api
uv run python -c "import yaml; yaml.safe_load(open('core/tools/builtin_tool/providers/your_provider/your_provider.yaml'))"

# 3. Restart services
cd docker
docker compose restart api worker

# 4. Check logs
docker compose logs api | grep "your_provider"
```

### Credential Validation Error

```bash
# Check provider validation logic
cd api
uv run python -c "
from core.tools.builtin_tool.providers.your_provider.your_provider import YourProviderToolProvider
provider = YourProviderToolProvider()
provider._validate_credentials('test-user', {'api_key': 'test-key'})
"
```

### Tool Execution Error

```bash
# Test tool directly
cd api
uv run python -c "
from core.tools.builtin_tool.providers.your_provider.tools.your_tool import YourTool
tool = YourTool()
tool.credentials = {'api_key': 'your-test-key'}
result = list(tool._invoke('test-user', {'param': 'value'}))
print(result[0].message)
"
```

## Next Steps

Setelah menguasai custom tools:

1. **[Code-based Extensions](./05-code-based-extensions.md)** - Level up ke extensions
2. **[Production Deployment](./07-production-deployment.md)** - Deploy tools ke production
3. **[Best Practices](./08-best-practices.md)** - Professional development patterns

---

> 💡 **Key Takeaway**: Custom tools adalah entry point terbaik untuk extend Dify. Mulai sederhana dengan random quote, lalu progress ke API integrations yang lebih complex. Selalu prioritaskan error handling dan security.