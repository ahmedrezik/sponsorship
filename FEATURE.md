# Sponsorship GAP Analysis — System Specification

> High-level feature description. All implementation must match this document.
> Claude Code reads this before touching any file.

---

## 1. System Overview

A two-model API pipeline that researches all sponsorship deals for a sports club,
then identifies gaps — industry verticals where the club previously had sponsors
but currently does not.

**Step 1 — o3-deep-research (Azure AI Foundry + Bing grounding)**
Fetches real-world sponsorship data with citations, grounded by live web search.

**Step 2 — claude-opus-4-6 (Claude on Azure)**
Segments by industry, identifies gaps, assesses exclusivity blockers.

These are the required models. Do not substitute them.
The API is stateless. No database. No caching. One request = one full pipeline run.
Endpoints and credentials are environment-driven.

---

## 2. Configuration — config.py

Load all values from environment via `python-dotenv`.
Expose as module-level typed constants. No classes, no validation logic.

| Constant                         | Type | Description                                  |
| -------------------------------- | ---- | -------------------------------------------- |
| AZURE_AI_FOUNDRY_ENDPOINT        | str  | Base URL for the Azure AI Foundry deployment |
| AZURE_AI_FOUNDRY_API_KEY         | str  | API key for Azure AI Foundry                 |
| AZURE_AI_FOUNDRY_DEPLOYMENT_NAME | str  | Deployment name for the Step 1 model         |
| ANTHROPIC_AZURE_ENDPOINT         | str  | Base URL for Claude on Azure                 |
| ANTHROPIC_AZURE_API_KEY          | str  | API key for Claude on Azure                  |
| ANTHROPIC_AZURE_DEPLOYMENT_NAME  | str  | Deployment name for the Step 2 model         |
| PORT                             | int  | Port the server binds to                     |

Model deployment names default to `o3-deep-research` and `claude-opus-4-6` respectively.
All other defaults live in `.env.example`.

---

## 3. Data Models — models.py

Use Pydantic v2. No inheritance chains. Keep models flat.

### 3.1 API Layer

```
GapAnalysisRequest
  club_name: str

HealthResponse
  status: str = "ok"

ErrorResponse
  error: str
  detail: str | None
  request_id: str | None
```

### 3.2 Deal (Step 1 output unit)

All fields optional except `club` and `partner_brand`.

```
Deal
  club: str
  partner_brand: str
  parent_company: str | None
  sponsorship_category: str | None
  asset_type: str | None          # see §6.2 for allowed values
  rights_summary: str | None
  exclusivity: str | None         # "yes/no/unknown + notes"
  geography: str | None
  start_date: str | None
  end_date: str | None            # YYYY-MM-DD, YYYY, or "ongoing"
  duration_years: float | None
  reported_annual_value: float | None
  reported_total_value: float | None
  currency: str | None
  variable_terms: str | None
  renewal_options: str | None
  source_1_title: str | None
  source_1_url: str | None
  source_2_title: str | None
  source_2_url: str | None
  confidence: str | None          # "High" | "Medium" | "Low"
  notes: str | None
```

### 3.3 GAP Analysis (Step 2 output)

```
CurrentSponsor
  brand: str
  asset_type: str
  since: str

HistoricalSponsor
  brand: str
  asset_type: str
  years: str

IndustrySegment
  vertical: str
  current_sponsors: list[CurrentSponsor]
  historical_sponsors: list[HistoricalSponsor]

Gap
  vertical: str
  last_sponsor: str
  last_active: str
  gap_status: str                 # "OPEN" | "PARTIALLY BLOCKED" | "BLOCKED"
  blocker_reason: str | None
  opportunity_notes: str

GapSummary
  total_verticals: int
  verticals_with_gaps: int
  open_gaps: int
  blocked_gaps: int
```

### 3.4 Final Response Shape (assembled dict, not a Pydantic model)

```
{
  club_name
  generated_at          — UTC ISO 8601 timestamp
  research
    total_deals_found
    deals               — list of Deal objects (as dicts)
    supplementary_analysis
    raw_response_fallback  — null on clean extraction, raw text otherwise
  gap_analysis
    industry_segments
    gaps
    summary             — GapSummary fields
  meta
    research_model      — from config
    analysis_model      — from config
    pipeline_version    — from config
}
```

---

## 4. Pipeline — pipeline.py

### 4.1 Prompt loading

Prompts are loaded from `prompts/` at module import time using `string.Template`.
Variables use `$var` syntax. Never use `str.format()` on prompt strings.

### 4.2 JSON extraction

Two private helpers. No external dependencies.

**`_extract_json_array(text) -> (list, remainder_str)`**
Try fenced code blocks first, then scan for raw `[`.
Return `([], original_text)` on failure.

**`_extract_json_object(text) -> dict`**
Same approach, scan for `{`.
Return `{}` on failure.

### 4.3 step1_azure_research

Calls the Step 1 model via `AsyncOpenAI` pointed at the Azure AI Foundry endpoint.
No timeout. No retries. The model may take several minutes.

Behaviour:

- Raises `ValueError` if the response returns no choices
- If JSON extraction fails: log a warning, return the raw text as fallback — do not raise
- If extraction succeeds: return `(deals, supplementary_text, None)`

### 4.4 step2_claude_analysis

Calls the Step 2 model via `AsyncAnthropicFoundry`.
Passes the full deals list as JSON in the user prompt.
Output budget should be generous enough to cover a full GAP analysis.

Behaviour:

- If JSON extraction fails: log an error with a response snippet, raise `ValueError`
- Returns the parsed gap analysis dict

### 4.5 run_pipeline

Calls step1 then step2 in sequence.
Assembles and returns the §4.4 response shape.
Model names in `meta` come from config, not hardcoded strings.

---

## 5. Prompts — prompts/

Prompts are plain text files. They define the intent and output contract for each model.
The spec describes what each prompt must instruct the model to do — the exact wording
is authored in the file itself.

### 5.1 step1_system.txt

Establishes the model's role: a sports sponsorship intelligence analyst that
produces structured, cited research and returns valid JSON arrays when asked.
No placeholders.

### 5.2 step1_user.txt

Placeholder: `$club_name`

Instructs the model to build the most complete dataset of sponsorship deals
(past and present) for `$club_name`. Focus areas: kit suppliers, shirt sponsors,
sleeve sponsors, stadium naming rights, all other partners.

Output contract: JSON array first, supplementary text after.

Each array object must include:

| Field                 | Notes                                                                                                               |
| --------------------- | ------------------------------------------------------------------------------------------------------------------- |
| club                  |                                                                                                                     |
| partner_brand         |                                                                                                                     |
| parent_company        |                                                                                                                     |
| sponsorship_category  |                                                                                                                     |
| asset_type            | one of: kit supplier, front-of-shirt, sleeve, training kit, stadium, digital, regional, academy, womens team, other |
| rights_summary        | concise                                                                                                             |
| exclusivity           | yes / no / unknown + notes                                                                                          |
| geography             | global or region name                                                                                               |
| start_date            | YYYY-MM-DD or YYYY                                                                                                  |
| end_date              | YYYY-MM-DD, YYYY, or "ongoing"                                                                                      |
| duration_years        | number or null                                                                                                      |
| reported_annual_value | number or null                                                                                                      |
| reported_total_value  | number or null                                                                                                      |
| currency              |                                                                                                                     |
| variable_terms        |                                                                                                                     |
| renewal_options       | string or null                                                                                                      |
| source_1_title        |                                                                                                                     |
| source_1_url          |                                                                                                                     |
| source_2_title        | null if unavailable                                                                                                 |
| source_2_url          | null if unavailable                                                                                                 |
| confidence            | High / Medium / Low                                                                                                 |
| notes                 |                                                                                                                     |

After the JSON array:

- Timeline summary of major sponsorship eras
- Terms playbook: recurring clauses (bonuses, rev share, exclusivity, digital rights)

Rules the prompt must state:

- Record exact dates. Distinguish disclosed vs reported figures.
- Do not invent numbers. Use "not disclosed" when unknown.
- Return valid JSON array first. Supplementary sections after.

### 5.3 step2_user.txt

Placeholders: `$club_name`, `$deals_json`

Three numbered steps:

**STEP 1 — SEGMENTATION**
Group all deals by industry vertical.
For each: list current sponsors (active today) and historical sponsors (no longer active).

**STEP 2 — GAP IDENTIFICATION**
Find verticals where `$club_name` had historical sponsors but has no current sponsor.

**STEP 3 — EXCLUSIVITY FILTER**
For each gap, assess whether it is blocked by:

- An existing sponsor's exclusivity clause in an adjacent category
- A competing brand already sponsoring a direct rival with exclusivity
- Any other known contractual barrier

Label each gap: OPEN, PARTIALLY BLOCKED, or BLOCKED.

Output contract: return ONLY valid JSON with this structure:

```
{
  industry_segments: [ { vertical, current_sponsors, historical_sponsors } ]
  gaps: [ { vertical, last_sponsor, last_active, gap_status, blocker_reason, opportunity_notes } ]
  summary: { total_verticals, verticals_with_gaps, open_gaps, blocked_gaps }
}
```

End with: `Deals data: $deals_json`

---

## 7. API — main.py

### 7.1 Logging

Every log record is emitted as a single-line JSON object containing:
`timestamp` (UTC), `level`, `logger`, `message`
— plus any of these context fields when present on the record:
`request_id`, `club_name`, `deal_count`, `response_chars`, `snippet`

Two handlers: console (stdout) and rotating file (`logs/app.log`).
Both use the same JSON formatter. Rotation thresholds are an implementation detail.
`logs/` is created at startup if missing.
Root logger at INFO. Existing handlers cleared before setup.
Configured before the app is created.

### 7.2 Middleware

One middleware per request:

- Generate a UUID request ID, attach to `request.state`
- Log `request.received` with method and path
- Add `X-Request-ID` to every response

### 7.3 Exception handler

Global handler for unhandled exceptions:

- Log `request.error` at ERROR with traceback and request ID
- Return 500 JSON: `{ error, detail, request_id }`

### 7.4 Routes

```
GET  /health        → HealthResponse             tag: ops
POST /gap-analysis  → JSONResponse (§4.4 shape)  tag: analysis
```

`/gap-analysis`:

- Pulls request ID from `request.state`
- Calls `run_pipeline`
- Catches pipeline exceptions locally, returns 500 JSON
- Docstring must note Step 1 can take several minutes

### 7.5 Entrypoint

Server binds to `0.0.0.0` on `PORT` from config.
`reload=False`. Uvicorn's default log config is disabled — we own logging.

---

## 8. Error Handling Rules

| Location              | Condition                       | Action                                     |
| --------------------- | ------------------------------- | ------------------------------------------ |
| step1_azure_research  | empty choices                   | raise ValueError                           |
| step1_azure_research  | JSON extraction fails           | log WARN, return raw fallback, don't raise |
| step2_claude_analysis | JSON extraction fails           | log ERROR with snippet, raise ValueError   |
| /gap-analysis route   | any exception from run_pipeline | log ERROR, return 500 JSON                 |
| global handler        | anything not caught above       | log ERROR, return 500 JSON                 |

Step 1 failing silently is intentional — the raw text still has value and is
returned to the caller for manual inspection.

---

## 9. Out of Scope — v1.0

- Authentication on the FastAPI side
- Rate limiting
- Request timeouts (Step 1 can genuinely take 10+ minutes)
- Retries (disabled intentionally on the Azure client)
- Persistent storage
- Streaming responses

Do not implement any of the above unless the spec is updated first.
