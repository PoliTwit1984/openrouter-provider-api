# OpenRouter Tools

A collection of Python tools for working with OpenRouter's models and providers:
1. Provider Scraper - Extracts provider information from OpenRouter model pages
2. Models API - Flask API for searching OpenRouter models and retrieving their provider information

## Models API

A Flask-based REST API that provides endpoints for searching and retrieving OpenRouter model information.

### API Endpoints

1. `GET /api/models`
   - Returns a list of all model IDs
   - Response format:
   ```json
   {
     "success": true,
     "count": 123,
     "model_ids": ["model1", "model2", ...]
   }
   ```

2. `GET /api/models/search?q=<query>`
   - Searches for models containing the query string (case-insensitive)
   - Example: `/api/models/search?q=sonnet`
   - Response format:
   ```json
   {
     "success": true,
     "count": 6,
     "matches": ["anthropic/claude-3-sonnet", ...]
   }
   ```
   - Returns `"matches": null` if no models found

3. `GET /api/get_providers?q=<model_id>`
   - Gets provider information for a specific model
   - Example: `/api/get_providers?q=deepseek/deepseek-chat`
   - Response format:
   ```json
   {
     "success": true,
     "model_id": "deepseek/deepseek-chat",
     "providers": [
       {
         "name": "deepseek",
         "metrics": {
           "context_length": 64000,
           "max_output_tokens": 8000,
           "input_price_per_million": 0.14,
           "output_price_per_million": 0.28,
           "latency_seconds": 1.23,
           "throughput_tokens_per_second": 67.37
         }
       }
     ]
   }
   ```
   - Returns `"providers": null` if model not found

### Running the API

1. Start the Flask server:
```bash
python app.py
```

2. The server runs on `http://localhost:8000` by default

### API Implementation Details

- Uses Flask for routing and request handling
- Reads model data from `models.json` (generated by scrape_models.py)
- Returns JSON responses with consistent structure
- Includes error handling for missing data and invalid requests

## Model Data Scraper

The `scrape_models.py` script fetches the latest model information from OpenRouter's API and saves it to a JSON file.

### Usage

```bash
python scrape_models.py
```

The script will:
1. Load existing models.json if it exists
2. For each model:
   - Fetch latest model data from OpenRouter's API
   - Compare new data with existing data
   - If changes are detected:
     * Update the model's provider information
     * Save changes to models.json immediately
   - Continue to next model
3. This per-model saving ensures no changes are lost if the script is interrupted

### Requirements

- OpenRouter API key in `.env` file
- Python packages: requests, python-dotenv

## Provider Scraper

## Overview

This script uses Playwright to navigate OpenRouter's web interface and extract provider information for any given model. It handles various cases including:
- Models with multiple providers (e.g., DeepSeek)
- Models with single providers (e.g., Qwen)
- Models with free providers (e.g., Gemini)
- Models without any providers (e.g., Shap-E)

## Requirements

- Python 3.7+
- Playwright
- python-dotenv

## Installation

1. Clone the repository
2. Create and activate a virtual environment:
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

3. Install dependencies:
```bash
pip install playwright python-dotenv
playwright install
```

4. Create a `.env` file in the root directory and add your OpenRouter API key:
```
OPENROUTER_API_KEY=your_api_key_here
```

## Usage

Run the script by providing a model URL as an argument:

```bash
python scrape_providers.py <model_url>
```

Example:
```bash
python scrape_providers.py https://openrouter.ai/google/gemini-2.0-flash-thinking-exp:free
```

The script will:
1. Load existing models.json
2. For each model:
   - Navigate to its OpenRouter page
   - Extract provider information
   - Compare with existing provider data
   - If changes are detected:
     * Update the model's provider information
     * Save changes to models.json immediately
   - Continue to next model
3. This per-model saving ensures no changes are lost if the script is interrupted
4. Report which models were updated and why

## Output Format

The script outputs a JSON file with a consistent structure regardless of whether providers are found:

1. For models with providers:
```json
{
  "providers": [
    {
      "name": "provider_name",
      "metrics": {
        "context_length": 40000,
        "max_output_tokens": 8000,
        "input_price_per_million": 0.14,
        "output_price_per_million": 0.28,
        "latency_seconds": 1.23,
        "throughput_tokens_per_second": 67.37
      }
    }
  ]
}
```

2. For models without providers:
```json
{
  "providers": [
    {
      "name": null,
      "metrics": {
        "context_length": null,
        "max_output_tokens": null,
        "input_price_per_million": null,
        "output_price_per_million": null,
        "latency_seconds": null,
        "throughput_tokens_per_second": null
      }
    }
  ]
}
```

## Technical Implementation

### HTML Structure

The provider data on OpenRouter is structured as follows:

```html
<!-- Provider Tab -->
<button role="tab" name="Providers">Providers</button>

<!-- Provider Section -->
<div class="flex flex-col gap-3">
  <!-- Provider Row -->
  <tr class="flex flex-col py-4 border-t border-border/50">
    <!-- Provider Name -->
    <a class="text-muted-foreground/80">Provider Name</a>
    
    <!-- Metrics Container -->
    <div class="flex flex-wrap items-center justify-between gap-8">
      <!-- Six metrics in this order: -->
      <div class="text-lg">64K</div>        <!-- Context Length -->
      <div class="text-lg">8K</div>         <!-- Max Output -->
      <div class="text-lg">$0.14</div>      <!-- Input Cost -->
      <div class="text-lg">$0.28</div>      <!-- Output Cost -->
      <div class="text-lg">1.23s</div>      <!-- Latency -->
      <div class="text-lg">67.37t/s</div>   <!-- Throughput -->
    </div>
  </tr>
</div>
```

### Playwright Selectors

The script uses these specific selectors to extract data:

1. Finding the Providers tab:
```python
providers_tab = page.get_by_role("tab", name="Providers")
```

2. Waiting for provider section:
```python
page.wait_for_selector('div.flex.flex-col.gap-3', state='visible')
```

3. Getting provider rows:
```python
rows = page.locator('tr.flex.flex-col.py-4.border-t.border-border\\/50').all()
```

4. For each row:
   - Provider name:
   ```python
   name_el = row.locator('a.text-muted-foreground\\/80').first
   ```
   - Metrics container:
   ```python
   metrics_container = row.locator('div.flex.flex-wrap.items-center.justify-between.gap-8').first
   ```
   - Metric values:
   ```python
   metrics = metrics_container.locator('div.text-lg').all()
   ```

### Data Cleaning

The script handles these specific formats:

1. Context Length & Max Output:
   - Input: "64K", "128K"
   - Conversion: Multiplies by 1000 (64000, 128000)

2. Pricing:
   - Input: "$0.14", "$0.28"
   - Conversion: Removes "$" and converts to float

3. Latency:
   - Input: "1.23s"
   - Conversion: Removes "s" and converts to float

4. Throughput:
   - Input: "67.37t/s"
   - Conversion: Removes "t/s" and converts to float

## Error Handling

The script includes robust error handling:

1. **Navigation Errors**
   - Handles missing Providers tab
   - Handles failed page loads
   - Handles timeout issues

2. **Data Extraction Errors**
   - Handles missing elements
   - Handles malformed data
   - Handles conversion errors

3. **Consistent Output**
   - Always returns valid JSON
   - Uses null values for missing data
   - Maintains structure consistency

## Common Issues

1. Selector Timeouts:
   - The script waits 5000ms for selectors
   - Increase timeout if needed:
   ```python
   page.wait_for_selector('selector', timeout=10000)
   ```

2. Missing Elements:
   - Some models don't have providers
   - Script returns null values in standardized format

3. Data Format Changes:
   - If OpenRouter changes their UI:
   - Update selectors in script
   - Update value parsing logic if formats change

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

MIT
