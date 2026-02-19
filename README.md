# Sports Sponsorship GAP Analysis API

A FastAPI service that performs deep, AI-driven sponsorship GAP analysis for any sports club. It combines **Azure AI Foundry o3** (with web search grounding) for deal research and **Claude claude-sonnet-4-5** for structured industry segmentation and opportunity identification.

---

## Stack

| Layer | Technology |
|-------|-----------|
| API framework | FastAPI + uvicorn |
| Deep research | Azure AI Foundry — `o3` with web search grounding |
| Structured analysis | Anthropic Claude `claude-sonnet-4-5` |
| Data validation | Pydantic v2 |
| Async HTTP | httpx (transitive, via azure-ai-inference) |
| Config | python-dotenv |

---

## Setup

### 1. Clone and create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Create your `.env` file

```bash
cp .env.example .env
```

Fill in your credentials:

```env
# Azure AI Foundry
# Endpoint format for MaaS: https://<resource>.services.ai.azure.com/models
# Endpoint format for Azure OpenAI: https://<resource>.openai.azure.com/
AZURE_AI_FOUNDRY_ENDPOINT=https://your-resource.services.ai.azure.com/models
AZURE_AI_FOUNDRY_API_KEY=your-azure-api-key-here
AZURE_AI_FOUNDRY_DEPLOYMENT_NAME=o3

# Anthropic
ANTHROPIC_API_KEY=your-anthropic-api-key-here

# Server
PORT=8000
```

### 3. Configure Azure AI Foundry web search grounding

The o3 deployment must have **Bing Grounding** enabled:

1. Open your Azure AI Foundry project in the [Azure portal](https://ai.azure.com).
2. Go to **Settings → Connected resources** and attach a **Bing Search** resource.
3. In your o3 deployment settings, enable **Grounding with Bing Search**.

Once configured, the service makes standard Chat Completions calls — Azure routes them through the grounding pipeline automatically.

### 4. Run the server

```bash
python main.py
# or
uvicorn main:app --host 0.0.0.0 --port 8000
```

---

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Liveness check — returns `{"status": "ok"}` |
| `POST` | `/gap-analysis` | Run full sponsorship GAP analysis |
| `GET` | `/docs` | Interactive Swagger UI |

---

## Example — Real Madrid

```bash
curl -X POST http://localhost:8000/gap-analysis \
  -H "Content-Type: application/json" \
  -d '{"club_name": "Real Madrid"}' \
  | jq .
```

Expected response shape:

```json
{
  "club_name": "Real Madrid",
  "generated_at": "2025-01-15T10:30:00.000000+00:00",
  "research": {
    "total_deals_found": 28,
    "deals": [
      {
        "club": "Real Madrid",
        "partner_brand": "Adidas",
        "parent_company": "Adidas AG",
        "sponsorship_category": "Kit supplier",
        "asset_type": "kit supplier",
        "rights_summary": "Long-term kit manufacturing and supply deal",
        "exclusivity": "yes — exclusive kit supplier rights globally",
        "geography": "global",
        "start_date": "1998",
        "end_date": "ongoing",
        "duration_years": null,
        "reported_annual_value": 110000000,
        "reported_total_value": null,
        "currency": "EUR",
        "variable_terms": "Performance bonuses linked to UCL/LaLiga titles",
        "activation_obligations": "Co-branded marketing campaigns",
        "renewal_options": "Extension clause active",
        "source_1_title": "Real Madrid renew Adidas deal",
        "source_1_publisher": "Reuters",
        "source_1_date": "2023-06-01",
        "source_1_url": "https://example.com/source1",
        "source_2_title": null,
        "source_2_publisher": null,
        "source_2_date": null,
        "source_2_url": null,
        "confidence": "High",
        "notes": "Figures are reported, not officially disclosed"
      }
    ],
    "supplementary_analysis": "## Timeline Summary\n...\n## Terms Playbook\n...\n## Source Index\n...",
    "raw_response_fallback": null
  },
  "gap_analysis": {
    "industry_segments": [
      {
        "vertical": "Sportswear / Kit",
        "current_sponsors": [
          {"brand": "Adidas", "asset_type": "kit supplier", "since": "1998"}
        ],
        "historical_sponsors": [
          {"brand": "Kelme", "asset_type": "kit supplier", "years": "1994–1998"}
        ]
      }
    ],
    "gaps": [
      {
        "vertical": "Cryptocurrency / Web3",
        "last_sponsor": "Socios.com",
        "last_active": "2022",
        "gap_status": "OPEN",
        "blocker_reason": null,
        "opportunity_notes": "Category fully open since Socios partnership concluded. High-value vertical with multiple tier-1 crypto brands actively seeking elite club partnerships."
      }
    ],
    "summary": {
      "total_verticals": 14,
      "verticals_with_gaps": 3,
      "open_gaps": 2,
      "blocked_gaps": 1
    }
  },
  "meta": {
    "research_model": "azure-ai-foundry/o3",
    "analysis_model": "claude-sonnet-4-5",
    "pipeline_version": "1.0.0"
  }
}
```

---

## Pipeline architecture

```
POST /gap-analysis  {"club_name": "..."}
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  STEP 1 — Azure AI Foundry (o3 + web search)        │
│  • Searches the web for all sponsorship deals       │
│  • Extracts deal JSON array + supplementary text    │
│  • Timeout: 10 minutes                              │
│  • On JSON parse failure: stores raw fallback       │
└────────────────────────┬────────────────────────────┘
                         │  deals[]
                         ▼
┌─────────────────────────────────────────────────────┐
│  STEP 2 — Claude claude-sonnet-4-5                  │
│  • Groups deals by industry vertical                │
│  • Identifies current vs historical sponsors        │
│  • Flags gaps + exclusivity blockers (OPEN /        │
│    PARTIALLY BLOCKED / BLOCKED)                     │
└────────────────────────┬────────────────────────────┘
                         │  gap_analysis{}
                         ▼
┌─────────────────────────────────────────────────────┐
│  STEP 3 — Assemble response                         │
│  • Merges research + gap_analysis + meta            │
│  • Returns full JSON                                │
└─────────────────────────────────────────────────────┘
```

---

## Logging

All log output is newline-delimited JSON (structured logging). Each line includes:

```json
{
  "timestamp": "2025-01-15T10:30:00.000000+00:00",
  "level": "INFO",
  "logger": "pipeline.step1",
  "message": "Step 1 complete: response received",
  "request_id": "a1b2c3d4-...",
  "response_chars": 18423
}
```

Every request carries a unique `request_id` (UUID v4), also returned in the `X-Request-ID` response header.

---

## Error handling

All errors return JSON — the server never returns an HTML error page:

```json
{
  "error": "Pipeline execution failed",
  "detail": "Azure AI Foundry returned an empty choices list",
  "request_id": "a1b2c3d4-..."
}
```
