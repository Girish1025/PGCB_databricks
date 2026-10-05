# PGCB notebook cleanup notes

## Recommended execution order
1. `01_bronze_ingestion.ipynb`
2. `02_silver_cleaning.ipynb`
3. `03_gold_aggregation.ipynb`
4. `04_demand_forecasting.ipynb`
5. `05_agent_tools.ipynb`
6. `06_main.ipynb`

## Key cleanup decisions

### 01 Bronze
- Consolidated many inspection-only cells into a compact ingestion + validation flow.
- Kept the raw Excel ingestion and ingestion metadata.
- Removed empty/redundant cells.

### 02 Silver
- Consolidated repeated one-column-at-a-time quality code into centralized rules.
- Added exact-row deduplication.
- Quarantines conflicting duplicate timestamps instead of arbitrarily keeping one.
- Preserves source NULLs rather than globally filling them with zero.
- Centralized the original project's plausibility thresholds so they are reviewable.
- Removed unused `Window` logic from the old version and then reintroduced it only where duplicate-timestamp detection needs it.

### 03 Gold
- Removed `pgcb.gold.forecast_features` because no downstream notebook used it.
- Kept only the two curated products actually consumed downstream: `daily_demand` and `grid_operations`.
- Added observation count to the daily table.

### 04 Forecasting
- Removed duplicate imports, duplicate final-model MLflow runs, hard-coded validation metrics, and unused ARIMA/Holt imports.
- Fixed a model inconsistency: the selected Holt-Winters candidate used multiplicative seasonality, while the old final model accidentally retrained with additive seasonality.
- Replaced the misleading native statsmodels signature (`days_ahead` did not match native `predict`) with a custom MLflow PyFunc whose actual interface is `days_ahead -> dated demand forecasts`.
- Registers the PyFunc model in Unity Catalog and assigns `@champion`.

### 05 Agent tools
- Removed multiple duplicate definitions of `get_grid_data`, `calculate_supply_gap`, `forecast_daily_demand`, and repeated model loading.
- Removed experimental/manual test scenarios that were useful during development but should not live in the final tool-definition notebook.
- Keeps one numeric `forecast_next_day_demand` tool to support type-safe chaining into the DOUBLE parameters of the UC calculation function.
- Keeps UC-tool creation and the combined `tools` list in one place.

### 06 Main
- Removed all model-endpoint debugging, failed Claude attempts, duplicate LLM definitions, duplicate agent creation, `%run ./04_demand_forecasting`, and training/model-registration code.
- `06_main` now does only what a main application notebook should do: load tools, initialize the LLM, define system instructions, create the agent, and run smoke tests.

## Important project assumptions
- The Silver data-quality bounds are project-defined and must not be presented as official PGCB operational limits.
- The current ML model forecasts **daily average demand**, not hourly/7 PM demand.
- `expected_generation_mw` and `expected_imports_mw` in the agent scenario are planning assumptions supplied by the user unless a future generation/import forecasting tool is added.
- The current recommendation tool is rule-based decision support, not autonomous grid control.
