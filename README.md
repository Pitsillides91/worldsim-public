# WorldSim

**Probabilistic socio-economic simulation platform for 195 countries to 2050.**

WorldSim runs Monte Carlo simulations across multiple socio-economic indicators, producing full probability distributions (P10/P50/P90) instead of single-point forecasts. Over 100 structural coupling rules connect variables causally, so when you shock one variable, the cascading effects propagate automatically.

[Try it free](https://worldsimlab.com) | [Methodology](https://worldsimlab.com/methodology/) | [Pricing](https://worldsimlab.com/pricing/)

---

## What It Does

- **195 countries** with historical data from 2000-2025 and probabilistic forecasts to 2050
- **26 KPIs** across 9 domains (income, cost of living, housing, fiscal, labor, demographics, social, energy, technology)
- **100+ structural coupling rules** that fire inside each simulation loop
- **Monte Carlo simulation** with 2,000-10,000 trajectories per scenario
- **Three scenario paths:** Better, Average, Shock
- **Full P10/P50/P90 distributions**, not single-point estimates
- **Reproducible:** every run is deterministic per configuration and seed

---

## How It Works

WorldSim uses a 5-layer simulation engine:

| Layer | Name | Purpose |
|-------|------|---------|
| L1 | Baseline | Load P10/P50/P90 forecast distributions from canonical sources |
| L2 | User Bias | Apply sigma shifts to any KPI (user-controlled) |
| L3 | Path Tilt | Apply scenario direction (Better / Average / Shock) |
| L4 | Coupling Rules | Structural KPI interactions via System Coupling Engine + Black Swan Engine |
| L5 | Personal Overlay | Translate macro outcomes to household-level projections |

Each layer modifies the previous. The final distributions are sampled via Monte Carlo and aggregated into quantile outputs.

---

## KPI Taxonomy

### 9 Domains, 26 Indicators

| Domain | KPIs |
|--------|------|
| **Income & Productivity** | GDP per capita |
| **Cost of Living** | Inflation rate, Electricity price (USD/kWh), Interest rate (policy %), Petrol price (USD/litre) |
| **Housing Affordability** | Rent index, Price-to-income ratio, Price-to-rent ratio |
| **Fiscal & Tax Structure** | Tax wedge (avg worker), Gov. expenditure (% GDP), Gov. revenue (% GDP), Public debt (% GDP) |
| **Labor Market** | Unemployment rate, Crime rate (per 100k), AI displacement exposure (%) |
| **Demographics** | Population share 65+, Total fertility rate, Net migration rate (per 1,000) |
| **Social & Cultural** | Religion shares (Christian, Muslim, Hindu, Other/Atheist) |
| **Energy** | Energy self-sufficiency (%), Renewable energy share (%) |
| **Technology** | Internet users (%), R&D expenditure (% GDP), High-tech exports (%) |

---

## Structural Coupling Rules

WorldSim's engine includes 100+ empirically calibrated coupling rules organized into two systems:

**System Coupling Engine (SCE):** Gradual structural interactions between KPIs. When GDP declines, unemployment rises. When unemployment rises, net migration falls. When migration falls, the tax base erodes. These cascades are modelled explicitly.

**Black Swan Engine (BSE):** Threshold-triggered crisis rules. When sovereign debt exceeds critical levels while inflation is elevated, a crisis cascade activates with amplified effects across multiple domains simultaneously.

All rules include:
- Explicit trigger conditions with empirical thresholds
- Floor and ceiling guards to prevent unrealistic outputs
- Asymmetric effects (crises hit harder than recoveries)
- Persistence and decay parameters
- Academic references and empirical grounding

[View the public rule catalog](https://worldsimlab.com/coupling-rules/)

---

## Data Sources

| Source | Coverage |
|--------|----------|
| **World Bank** | GDP, population, fertility, migration, education, technology indicators |
| **IMF** | Fiscal indicators, debt, government expenditure and revenue |
| **OECD** | Tax wedge, housing indices, labour market indicators |
| **Eurostat** | Energy prices, renewable energy shares, EU-specific demographic data |

Historical data: 2000-2025. Baseline forecasts: 2025-2050.

---

## Use Cases

| Audience | Example Scenario |
|----------|-----------------|
| **Governments** | Stress-test fiscal plans under energy price shocks and demographic pressure |
| **Hedge Funds** | Quantify sovereign tail risk with full P10/P50/P90 debt trajectories |
| **Policy Institutions** | Compare structural resilience across EU member states under identical conditions |
| **Corporate Strategy** | Evaluate 15-year cost structures for site selection (energy, tax, labor, demographics) |
| **Research** | Reproducible probabilistic scenarios for peer-reviewed structural analysis |
| **AI & Data Teams** | Structurally coherent synthetic macro data for model training and stress testing |
| **EU AI Act Compliance** | Environmental robustness testing across diverse socio-economic conditions |

[Explore all use cases](https://worldsimlab.com/use-cases/governments/)

---

## Output Structure

Every simulation produces:

- **Baseline distributions** (P10/P50/P90 from canonical forecasts)
- **Prepared baseline** (after user bias and path tilt)
- **Tilted path** (scenario-adjusted trajectory)
- **Monte Carlo quantiles** (P10/P50/P90 from N simulated trajectories)
- **Trajectory Index** (single 0-1 score aggregating all KPI outcomes)
- **Regime classification** per domain (e.g. "Fiscal Stress", "Demographic Winter", "Renewable Surge")
- **Rule fire log** (full audit trail of which coupling rules activated, when, and why)

---

## API (Coming Soon)

Programmatic access to WorldSim simulations. Features planned:

- RESTful endpoints for scenario configuration and run submission
- API key authentication with tier-based rate limiting
- Batch scenario submissions
- JSON response with full quantile outputs
- Raw Monte Carlo paths (Enterprise tier)

Documentation will be published here when available.

---

## Links

- **Platform:** [worldsimlab.com](https://worldsimlab.com)
- **Methodology:** [worldsimlab.com/methodology](https://worldsimlab.com/methodology/)
- **Coupling Rules:** [worldsimlab.com/coupling-rules](https://worldsimlab.com/coupling-rules/)
- **Pricing:** [worldsimlab.com/pricing](https://worldsimlab.com/pricing/)
- **Substack:** [worldsimlab.substack.com](https://worldsimlab.substack.com)

---

## License

WorldSim is a proprietary platform. This repository contains public documentation only, not source code.

Copyright 2026 YP Analytics. All rights reserved.
