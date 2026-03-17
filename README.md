# Azure AI Foundry Agents via APIM + Application Insights Telemetry

This notebook demonstrates how to call **Microsoft AI Foundry hosted agents** through **Azure API Management (APIM)** and send token-usage telemetry to **Application Insights** using OpenTelemetry.

> **Important:** This notebook does **not** deploy or provision any agents. Your Microsoft AI Foundry agents (e.g. WeatherAgent, HotelAgent, FinanceAgent) must already be created and configured behind Azure API Management **before** running this notebook. Any agent hosted in Microsoft Foundry and exposed through APIM can be integrated — simply add a new `call_agent()` invocation with the agent's name and prompt.

## Architecture

```
┌──────────────┐       ┌─────────────────────┐       ┌──────────────────┐
│   Notebook    │──────▶│  Azure API Mgmt     │──────▶│  Azure AI        │
│  (this repo)  │       │  (AI Gateway)       │       │  Foundry Agents  │
└──────┬───────┘       └─────────────────────┘       └──────────────────┘
       │
       │  OpenTelemetry
       ▼
┌──────────────────┐
│  Application     │
│  Insights        │
└──────────────────┘
```

| Component | Role |
|-----------|------|
| **Azure AI Foundry** | Hosts the AI agents (e.g. WeatherAgent, HotelAgent, FinanceAgent) |
| **Azure API Management** | AI Gateway — routes, rate-limits, and traces agent requests |
| **Application Insights** | Collects token-usage metrics and cost estimates via OpenTelemetry |

## What the Notebook Does

1. **Configures App Insights** — Sets up OpenTelemetry counters (`llm.prompt_tokens`, `llm.completion_tokens`, `llm.total_tokens`, `llm.request_cost_usd`) and a logger that sends custom dimensions to the `traces` and `customMetrics` tables.
2. **Calls agents via APIM** — Sends HTTP POST requests to Foundry-hosted agents through the APIM gateway endpoint, authenticating with `DefaultAzureCredential` and an APIM subscription key.
3. **Tracks token usage** — Extracts `input_tokens`, `output_tokens`, and `total_tokens` from each response, estimates cost using built-in model pricing (GPT-4.1, GPT-4o, GPT-4o-mini), and logs everything as custom dimensions.
4. **Queries telemetry** — Runs KQL queries against the App Insights `traces` table to retrieve per-request token usage and cost.
5. **Aggregates metrics** — Computes average and total token consumption and cost **per agent per model** using a KQL `summarize` query.

## Notebook Sections

| # | Section | Description |
|---|---------|-------------|
| 1 | Configuration | Sets the App Insights connection string, APIM base URL, subscription key, and API version. |
| 2 | App Insights Setup & Agent Call | Configures OpenTelemetry, defines `track_llm_usage()` and `call_agent()`, then calls `WeatherAgent`. |
| 3 | Query Data | Queries the App Insights `traces` table for individual token-usage records. |
| 4 | Average & Total Metrics | Queries App Insights for aggregated metrics (avg tokens, avg cost, total cost) grouped by agent and model. |

## Prerequisites

- **Azure CLI** authenticated (`az login`)
- **Python 3.10+** with a virtual environment
- **Azure resources (pre-provisioned):**
  - Azure API Management instance with routes to Foundry-hosted agents
  - Azure AI Foundry project with **one or more agents already deployed and accessible via APIM** (this notebook does not create or deploy agents)
  - Application Insights resource
- **APIM subscription key** with access to the Foundry agent routes
- **App Insights connection string**

## Setup

1. **Clone the repo and create a virtual environment:**

   ```bash
   python -m venv .venv
   # Windows
   .venv\Scripts\Activate.ps1
   # macOS / Linux
   source .venv/bin/activate
   ```

2. **Install dependencies:**

   ```bash
   pip install httpx azure-identity azure-monitor-opentelemetry opentelemetry-api opentelemetry-sdk azure-monitor-query
   ```

   Or install all project dependencies:

   ```bash
   pip install -r requirements.txt
   ```

3. **Update the configuration cell** in the notebook with your own values:

   | Variable | Description |
   |----------|-------------|
   | `APP_INSIGHTS_CONN` | Application Insights connection string |
   | `APP_INSIGHTS_RESOURCE_ID` | Full ARM resource ID of your App Insights instance |
   | `APIM_BASE` | APIM gateway URL (e.g. `https://mygateway.azure-api.net`) |
   | `APIM_SUB_KEY` | APIM subscription key |
   | `API_VERSION` | Foundry API version (default: `2025-11-15-preview`) |

4. **Run the notebook cells in order.**

## Querying Telemetry in App Insights

Navigate to your App Insights resource → **Logs** and run:

```kql
traces
| where message == "llm.usage"
| extend cd_raw = tostring(customDimensions["custom_dimensions"])
| extend cd = parse_json(replace_string(cd_raw, "'", "\""))
| extend
    agent_name        = tostring(cd["agent_name"]),
    model             = tostring(cd["model"]),
    operation_id      = tostring(cd["operation_id"]),
    prompt_tokens     = toint(cd["prompt_tokens"]),
    completion_tokens = toint(cd["completion_tokens"]),
    total_tokens      = toint(cd["total_tokens"]),
    cost_usd          = todouble(cd["cost_usd"])
| project timestamp, message, agent_name, model, operation_id,
         prompt_tokens, completion_tokens, total_tokens, cost_usd
| order by timestamp desc
```

> **Note:** Telemetry takes **2–5 minutes** to appear in App Insights after execution.

## Model Pricing (built-in)

| Model | Input (per token) | Output (per token) |
|-------|-------------------|--------------------|
| gpt-4.1 | $0.000002 | $0.000008 |
| gpt-4o | $0.0000025 | $0.000010 |
| gpt-4o-mini | $0.00000015 | $0.0000006 |

## Adding Your Own Agents

Any agent hosted in Microsoft Foundry and exposed through APIM can be added to this notebook. To call a new agent, simply invoke `call_agent()` with the agent's route name and a user message:

```python
response = call_agent("YourAgentName", "Your prompt here")
print("YourAgentName:", response)
```

Token usage and cost telemetry will be tracked automatically — no additional configuration is needed.

## Key Functions

| Function | Purpose |
|----------|---------|
| `track_llm_usage()` | Records token counters to `customMetrics` and logs a structured `llm.usage` entry to the `traces` table with cost, model, and agent metadata. |
| `call_agent()` | Sends an authenticated POST request to a Foundry agent via APIM, extracts the response text, and calls `track_llm_usage()`. |
| `query_logs()` | Queries App Insights for individual token-usage records. |
| `query_avg_cost()` | Queries App Insights for per-agent, per-model aggregated metrics. |
