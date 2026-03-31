# WorldSim API

Run country simulations programmatically. Submit a scenario, poll for results, and download full Monte Carlo distributions.

**Base URL:** `https://worldsimlab.com/api/v1`
**Interactive docs:** [worldsimlab.com/api/v1/docs](https://worldsimlab.com/api/v1/docs)

## Getting Started

### 1. Get your API key

Log in at [worldsimlab.com](https://worldsimlab.com), go to **Account**, and click **Create Key**. API access requires Institutional tier or above.

### 2. Install dependencies

```bash
pip install requests pandas pyarrow
```

### 3. Connect

```python
import requests
import time

BASE = "https://worldsimlab.com/api/v1"

session = requests.Session()
session.headers["Authorization"] = "Bearer wsim_your_key_here"

# Verify connection
resp = session.get(f"{BASE}/countries/")
print(f"Connected: {resp.json()['count']} countries available")
```

---

## Full Example: Germany to 2050

Simulate Germany with a +1 sigma inflation shock that persists for 5 years.

### Step 1: Submit the simulation

```python
resp = session.post(f"{BASE}/run/", json={
    "country": "DEU",
    "horizon": 2050,
    "path": "average",
    "biases": {
        "inflation_rate": {"shift_sigma": 1.0, "persist_years": 5}
    }
})

run_id = resp.json()["run_group_id"]
print(f"Submitted: {run_id}")
```

Response:

```json
{
  "run_group_id": "a1b2c3d4-...",
  "status": "queued",
  "paths": ["better", "shock", "average"],
  "poll_url": "/api/v1/run/a1b2c3d4-.../"
}
```

The engine always runs all 3 paths (better, average, shock) regardless of which one you select. Your path choice determines the primary scenario direction.

### Step 2: Poll until done

```python
while True:
    resp = session.get(f"{BASE}/run/{run_id}/")
    data = resp.json()
    print(f"Status: {data['status']}")

    if data["status"] in ("done", "failed"):
        break
    time.sleep(3)
```

Typical runtime: 15-25 seconds.

### Step 3: Read the results

```python
data = session.get(f"{BASE}/run/{run_id}/").json()

for path_name, path_data in data["output"].items():
    ti = path_data["trajectory_index"]
    print(f"{path_name:10s}  TI: {ti['score']} ({ti['label']})  |  {len(path_data['cards'])} cards")
```

```
average     TI: 48.2 (Moderate)     |  9 cards
better      TI: 62.1 (Favourable)   |  9 cards
shock       TI: 31.5 (Challenging)  |  9 cards
```

Each path contains:

- **cards** -- 9 enriched KPI cards (Income, Cost of Living, Housing, Fiscal, Labor, Demographics, Social, Energy, Technology)
- **trajectory_index** -- composite score (0-100) with a label
- **narrative** -- generated scenario narrative
- **monte_carlo_meta** -- seed, number of simulations

### Step 4: Get Monte Carlo quantiles

```python
mc = session.get(f"{BASE}/run/{run_id}/mc/").json()

# GDP per capita quantiles for the average path
gdp = mc["paths"]["average"]["quantiles"]["gdp_per_capita"]["years"]

print(f"{'Year':>6}  {'P10':>10}  {'P50':>10}  {'P90':>10}")
for row in gdp:
    print(f"{row['year']:>6}  {row['p10']:>10,.0f}  {row['p50']:>10,.0f}  {row['p90']:>10,.0f}")
```

```
  Year         P10         P50         P90
  2025      46,200      48,400      50,600
  2030      47,100      51,300      55,800
  2040      49,500      57,200      66,100
  2050      51,800      63,400      78,300
```

P10/P50/P90 are the 10th, 50th, and 90th percentile of the Monte Carlo distribution.

---

## Raw Sample Paths (Enterprise)

Enterprise clients can download the full Monte Carlo simulation matrix as a Parquet file.

```python
# Submit with raw_paths=True
resp = session.post(f"{BASE}/run/", json={
    "country": "DEU",
    "horizon": 2050,
    "path": "average",
    "raw_paths": True
})
run_id = resp.json()["run_group_id"]

# ... poll until done ...

# Get download URLs
mc = session.get(f"{BASE}/run/{run_id}/mc/").json()
url = mc["paths"]["average"]["raw_paths_url"]

# Download and load into pandas
import pandas as pd

file = session.get(f"https://worldsimlab.com{url}")
with open("germany_average.parquet", "wb") as f:
    f.write(file.content)

df = pd.read_parquet("germany_average.parquet")
print(df.head())
```

```
              kpi  sim_id  year      value
   gdp_per_capita       0  2025  47823.41
   gdp_per_capita       0  2026  48156.22
   gdp_per_capita       0  2027  48934.87
   gdp_per_capita       0  2028  49201.55
   gdp_per_capita       0  2029  50112.30
```

Columns: `kpi`, `sim_id`, `year`, `value`. Each sim_id is one independent simulation path. Use this for custom risk models, VaR calculations, or distribution analysis.

```python
# Example: GDP distribution at 2050
gdp = df[df["kpi"] == "gdp_per_capita"].pivot(index="year", columns="sim_id", values="value")
terminal = gdp.loc[2050]
print(f"Mean:   {terminal.mean():>10,.0f}")
print(f"Median: {terminal.median():>10,.0f}")
print(f"P5:     {terminal.quantile(0.05):>10,.0f}")
print(f"P95:    {terminal.quantile(0.95):>10,.0f}")
```

---

## Bias Parameters

Biases shift KPI baselines using the `biases` field in POST /run/.

```python
"biases": {
    "inflation_rate": {"shift_sigma": 1.0, "persist_years": 5},
    "unemployment_rate": {"shift_sigma": -0.5, "persist_years": 3}
}
```

- **shift_sigma**: Standard deviations to shift the baseline. +1.0 = one sigma above, -2.0 = two sigma below. Typical range: -3.0 to +3.0.
- **persist_years**: How long the shift lasts before reverting. 0 = one year only.

Use `GET /kpis/` to see all 26 available KPI codes.

---

## Available Countries

```python
countries = session.get(f"{BASE}/countries/").json()["countries"]
# 261 entries: {"iso3": "DEU", "name": "Germany"}, ...
```

Use the `iso3` code in the `country` field of POST /run/.

## Available KPIs

```python
kpis = session.get(f"{BASE}/kpis/").json()["kpis"]
# 26 entries grouped by card
```

| Card | KPIs |
|------|------|
| Income & Productivity | gdp_per_capita |
| Cost of Living | inflation_rate, electricity_price_household_usd_per_kwh, interest_rate_policy_pct, petrol_price_usd_litre |
| Housing Affordability | rent_index, price_to_income, price_to_rent |
| Fiscal & Tax Structure | tax_wedge_avg_worker, gov_expenditure_pct_gdp, gov_revenue_pct_gdp, public_debt_pct_gdp |
| Labor Market | unemployment_rate, crime_rate_per_100k, ai_displacement_exposure_pct |
| Demographics | population_share_65_plus, total_fertility_rate, net_migration_rate_per_1000 |
| Social & Cultural | religion_share_christian, religion_share_muslim, religion_share_hindu, religion_share_other_atheist |
| Energy | energy_self_sufficiency_pct, renewable_energy_share_pct |
| Technology | internet_users_pct, rd_expenditure_pct_gdp, high_tech_exports_pct |

## Horizons and Paths

**Horizons:** `2030`, `2035`, `2040`, `2050`

**Paths:**

- **better** -- optimistic scenario
- **average** -- baseline scenario
- **shock** -- adverse scenario

---

## Error Handling

```python
# No auth
requests.get(f"{BASE}/countries/")                     # 401

# Bad country
session.post(f"{BASE}/run/", json={"country": "ZZZ"})  # 400

# Bad horizon
session.post(f"{BASE}/run/", json={"country": "DEU", "horizon": 2099})  # 400

# Unknown KPI in biases
session.post(f"{BASE}/run/", json={
    "country": "DEU",
    "biases": {"fake_kpi": {"shift_sigma": 1}}
})  # 400

# Run not found
session.get(f"{BASE}/run/00000000-0000-0000-0000-000000000000/")  # 404
```

All errors return JSON:

```json
{
  "error": "invalid_country",
  "detail": "Country 'ZZZ' not found. Use GET /api/v1/countries/ for valid codes."
}
```

---

## Tier Limits

| Tier | API | Raw Paths | Rate Limit | Sims/Run | Runs/Month |
|------|-----|-----------|------------|----------|------------|
| Institutional | Yes | No | 100/hr | 5,000 | 1,000 |
| Institutional Pro | Yes | No | 700/hr | 10,000 | 7,000 |
| Enterprise | Yes | Yes | 10,000/hr | 10,000 | Unlimited |

Rate limit headers are included in every response:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 97
X-RateLimit-Reset: 1711872000
```
