# WorldSim API Documentation

Programmatic access to WorldSim country simulations.

Base URL: `https://worldsimlab.com/api/v1`
Interactive docs: https://worldsimlab.com/api/v1/docs

## Authentication

All requests require a Bearer token in the `Authorization` header:

```
Authorization: Bearer wsim_your_api_key_here
```

API keys are managed from your [Account page](https://worldsimlab.com/billing/account/). Keys are shown once at creation -- copy and store securely.

## Tier Access

| Tier | API Access | Raw MC Paths | Rate Limit | Simulations/Run |
|------|-----------|-------------|------------|-----------------|
| Free | No | No | -- | 2,000 |
| Individual | No | No | -- | 2,000 |
| Professional | No | No | -- | 5,000 |
| Institutional | Yes | No | 100 req/hr | 5,000 |
| Institutional Pro | Yes | No | 700 req/hr | 10,000 |
| Enterprise | Yes | Yes | 10,000 req/hr | 10,000 |

## Endpoints

### GET /countries/

List all available countries.

**Response:**
```json
{
  "count": 261,
  "countries": [
    {"iso3": "DEU", "name": "Germany"},
    {"iso3": "USA", "name": "United States"}
  ]
}
```

### GET /kpis/

List all 26 KPIs available for bias input, grouped by card.

**Response:**
```json
{
  "count": 26,
  "kpis": [
    {"code": "gdp_per_capita", "card": "Income & Productivity"},
    {"code": "inflation_rate", "card": "Cost of Living"}
  ]
}
```

### POST /run/

Submit a simulation. Returns immediately with a `run_group_id` for polling.

**Request body:**
```json
{
  "country": "DEU",
  "horizon": 2050,
  "path": "average",
  "biases": {
    "inflation_rate": {"shift_sigma": 1.0, "persist_years": 5}
  },
  "raw_paths": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `country` | string | Yes | ISO3 country code (e.g. "DEU", "USA") |
| `horizon` | int | No | End year: 2030, 2035, 2040, or 2050. Default: 2050 |
| `path` | string | No | Scenario: "better", "average", or "shock". Default: "average" |
| `biases` | object | No | KPI bias overrides (Layer 2). Key = KPI code, value = `{shift_sigma, persist_years}` |
| `raw_paths` | bool | No | Store all MC sample paths as Parquet. Enterprise only. Default: false |

**Bias parameters:**
- `shift_sigma`: Standard deviation shift applied to the KPI baseline. Positive = increase, negative = decrease. Range: typically -3.0 to +3.0.
- `persist_years`: How many years the bias persists before reverting. 0 = single year.

**Response:**
```json
{
  "run_group_id": "a1b2c3d4-...",
  "status": "queued",
  "paths": ["better", "shock", "average"],
  "poll_url": "/api/v1/run/a1b2c3d4-.../"
}
```

### GET /run/{run_group_id}/

Poll for simulation status and results.

**Status values:** `queued` > `running` > `done` (or `failed`)

**Response (when done):**
```json
{
  "run_group_id": "a1b2c3d4-...",
  "status": "done",
  "country": "DEU",
  "horizon": 2050,
  "paths": [
    {"path": "average", "status": "done", "completed_at": "2026-03-31T08:00:00Z"},
    {"path": "better", "status": "done", "completed_at": "2026-03-31T08:00:01Z"},
    {"path": "shock", "status": "done", "completed_at": "2026-03-31T08:00:02Z"}
  ],
  "output": {
    "average": {
      "cards": {},
      "trajectory_index": {"score": 52.3, "label": "Moderate"},
      "narrative": {},
      "monte_carlo_meta": {"n_sims": 5000, "seed": 123456}
    }
  }
}
```

### GET /run/{run_group_id}/mc/

Get Monte Carlo quantile distributions (P10/P50/P90) per KPI per path.

Enterprise tier also receives `raw_paths_url` links to downloadable Parquet files.

**Response:**
```json
{
  "run_group_id": "a1b2c3d4-...",
  "raw_paths_included": true,
  "paths": {
    "average": {
      "quantiles": {
        "gdp_per_capita": {
          "years": [
            {"year": 2025, "p10": 45000, "p50": 48000, "p90": 51000},
            {"year": 2026, "p10": 45500, "p50": 48500, "p90": 52000}
          ]
        }
      },
      "raw_paths_url": "/media/raw_paths/a1b2c3d4-.../average.parquet"
    }
  }
}
```

## Raw MC Paths (Enterprise)

When `raw_paths: true` is set, the full Monte Carlo simulation matrix is saved as a Parquet file on the server. The `/mc/` endpoint returns a download URL per path.

**Parquet schema:**

| Column | Type | Description |
|--------|------|-------------|
| `kpi` | string | KPI code (e.g. "gdp_per_capita") |
| `sim_id` | int | Simulation index (0 to n_sims-1) |
| `year` | int | Year (2025 to horizon) |
| `value` | float | Simulated value |

**Download example:**
```python
import pandas as pd

mc = session.get(f"{BASE}/run/{run_id}/mc/").json()
url = mc["paths"]["average"]["raw_paths_url"]

resp = session.get(f"https://worldsimlab.com{url}")
with open("paths.parquet", "wb") as f:
    f.write(resp.content)

df = pd.read_parquet("paths.parquet")
# Pivot to get simulation paths for one KPI
gdp = df[df["kpi"] == "gdp_per_capita"].pivot(
    index="year", columns="sim_id", values="value"
)
```

## Quick Start (Python)

```python
import requests
import time

BASE = "https://worldsimlab.com/api/v1"
s = requests.Session()
s.headers["Authorization"] = "Bearer wsim_your_key_here"

# 1. Submit
r = s.post(f"{BASE}/run/", json={
    "country": "DEU",
    "horizon": 2050,
    "path": "average",
    "biases": {
        "inflation_rate": {"shift_sigma": 1.0, "persist_years": 5},
        "unemployment_rate": {"shift_sigma": -0.5, "persist_years": 3}
    }
}).json()
run_id = r["run_group_id"]

# 2. Poll
while True:
    r = s.get(f"{BASE}/run/{run_id}/").json()
    if r["status"] in ("done", "failed"):
        break
    time.sleep(3)

# 3. Results
for path, data in r["output"].items():
    ti = data["trajectory_index"]
    print(f"{path}: TI={ti['score']} ({ti['label']})")

# 4. MC Quantiles
mc = s.get(f"{BASE}/run/{run_id}/mc/").json()
gdp_q = mc["paths"]["average"]["quantiles"]["gdp_per_capita"]["years"]
for row in gdp_q[-5:]:
    print(f"  {row['year']}: P10={row['p10']:,.0f}  P50={row['p50']:,.0f}  P90={row['p90']:,.0f}")
```

## Error Responses

| Code | Meaning | Example |
|------|---------|---------|
| 400 | Invalid input | Bad country code, invalid horizon, unknown KPI |
| 401 | Unauthorized | Missing or invalid API key, or tier without API access |
| 404 | Not found | Unknown run_group_id |
| 429 | Rate limited | Hourly rate limit exceeded |

All errors return JSON:
```json
{
  "error": "invalid_country",
  "detail": "Country 'ZZZ' not found. Use GET /api/v1/countries/ for valid codes."
}
```

## Rate Limit Headers

Every response includes rate limit headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 97
X-RateLimit-Reset: 1711872000
```
