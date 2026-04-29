# Total Gasoline Stocks Model v6.2

A single-file, browser-based dashboard for forecasting weekly US **Total Motor Gasoline** inventory levels using live EIA petroleum data. Covers US Total and all five PADD regions with analyst adjustment journaling and forward-tracking accuracy scoring.

---

## Overview

This model fetches live weekly petroleum data from the EIA API v2 and runs a stacked ensemble forecast to predict gasoline stock levels **1–4 weeks forward**. Three fundamental models feed into a meta-learner that adapts its blending weights based on the current market regime.

**Product:** Total Motor Gasoline (EIA code: EPM0)

**Regions:**
- U.S. Total (NUS)
- PADD 1 — East Coast
- PADD 2 — Midwest
- PADD 3 — Gulf Coast
- PADD 4 — Rocky Mountain
- PADD 5 — West Coast

---

## Data Inputs

All data is pulled live from the `EIA API v2`:

| Series | Description |
|--------|-------------|
| Weekly Stocks | Primary target variable (kbbl) |
| Production | Finished gasoline output from refineries |
| Imports / Exports | Trade flows (gasoline, RBOB, CBOB) |
| Refinery Utilization | Capacity utilization rate (%) |
| Crude Inputs | Crude oil input to refineries |
| Blender Inputs | RBOB and CBOB blender net inputs |
| Demand (Product Supplied) | Implied weekly consumption (US-total only; PADD demand estimated via stock-share proportioning) |

Data is cached in-memory with a **30-minute TTL** and auto-refreshes when stale. Up to **8 S&D feature series** are loaded per region.

---

## Prediction Models

Three primary models form the base layer:

### 1. S&D Regression
Ridge regression (λ = 1.0) with **forward stepwise feature selection**. Starts with core features (seasonality harmonics, momentum, YoY change) and greedily adds S&D features only if they improve out-of-sample validation accuracy. Prevents overfitting to noisy inputs.

### 2. S&D Balance
Seasonally-aware flow model. Computes implied weekly stock change from Production + Imports – Exports – Demand, calibrated against actual stock changes over the last 26 weeks. Projects forward using **same-week-last-year** component values adjusted by current deviation from seasonal norms.

### 3. AR(6) Autoregressive
Pure time-series model using lags 1–6 plus a seasonal lag (week 52). Captures momentum and mean reversion. Acts as the baseline when S&D data is sparse.

**Fallback models** (activated only when S&D data is insufficient):
- **Seasonal Naive** — Weighted multi-year same-week average with trend adjustment
- **Holt-Winters** — Exponential smoothing with grid-searched α, β, γ parameters

---

## Regime Detection

Before blending, the model classifies the current market environment using 7 features:

| Feature | Description |
|---------|-------------|
| Season (4 one-hot) | **Draw** (wk 10–24), **Peak** (wk 25–36), **Build** (wk 37–46), **Shoulder** (wk 47–9) |
| Volatility ratio | Recent 8-week vol ÷ trailing 52-week vol (>1.5× = high volatility) |
| Turnaround flag | Refinery utilization dropped >3 pts in 4 weeks |
| Momentum | 4-week stock change acceleration/deceleration (kbbl) |

---

## Meta-Learner (Ensemble Blending)

A ridge regression (λ = 0.5) trained on 30-week rolling backtest data. Learns **10 coefficients**:

- 1 intercept
- 3 model blend weights — how much to trust each base model
- 7 regime bias adjustments — regime-conditional corrections in kbbl

A **sanity check** bounds the meta-learner output to ±50% of the model spread. If it extrapolates too far, the system falls back to inverse-MAE weighted averaging with exponential decay (most recent weeks weighted highest).

---

## Confidence Intervals

80% CIs are computed from a **52-week** rolling backtest window. The 10th and 90th percentile of historical ensemble residuals define the band at each forecast horizon. Falls back to scaled standard deviation when backtest sample is too small.

---

## Dashboard Tabs

### Forecast Tab
- **Forecast cards**: +1W through +4W predictions with % change badges, 80% CI ranges, and YoY comparisons
- **Interactive chart**: Historical stocks, ensemble line, individual model lines, adjusted ensemble (when active), and optional 5-year avg band
- **Cross-PADD Snapshot**: All 6 regions' latest stocks, weekly change, +1W and +4W forecasts (computed with full S&D features for each PADD), and position vs. 5-year average
- **Detailed forecast table**: All model outputs, CI bounds, with adjusted column when adjustments are active
- **CSV export**: Full forecast including adjusted values if applicable

### Backtest Tab
- **Rolling backtest**: Runs the full ensemble (including meta-learner and regime detection) at each historical point
- **Summary stats**: MAPE, MAE, directional accuracy, sample count
- **Per-model MAE comparison**: Visual bars comparing Ensemble vs. S&D Reg vs. S&D Balance vs. AR
- **Scatter plot**: Predicted vs. Actual with perfect-line reference
- Configurable horizon (+1W to +4W) and window (26, 52, or 104 weeks)

### Meta-Learner Tab
- **Current regime**: Season, volatility, turnaround, momentum
- **Regime feature vector**: All 7 features the meta-learner sees
- **Model blend weights chart**: Bar chart of the 3 base model coefficients
- **Regime bias chart**: All 7 regime coefficients with active features highlighted
- **Blending walkthrough**: Step-by-step table showing intercept + model contributions + regime biases = final output
- **Fallback weights**: Inverse-MAE weights used when meta-learner is inactive

### Analyst Adjustments Tab
- **Adjustment builder**: Add qualitative overlays (refinery outage, weather event, demand shock, etc.) with direction (Bullish/Bearish), magnitude in kbbl, and affected weeks
- **Smart presets**: Impact suggestions auto-scaled by region's share of national stocks
- **Active adjustments table**: Shows all current adjustments with net impact per forecast week
- **Adjusted vs. Model comparison**: Side-by-side table of model ensemble vs. adjusted forecast
- **Adjustment Journal**: Forward-tracking performance scorer. When you add an adjustment, it snapshots the model forecast. As actual EIA data arrives, the journal scores whether your adjusted forecast was closer to actuals than the model alone. Tracks win rate, MAE improvement, and per-entry detail.

---

## Usage

1. Open `Total_Gasoline_Stocks_Model_V2.html` in any modern browser (no install or server required)
2. Select a PADD region from the dropdown (defaults to U.S. Total)
3. Click **▶ Run Forecast** to generate the ensemble prediction
4. Explore individual model contributions in the chart legend
5. Run a **Backtest** to validate model accuracy over the last 26–104 weeks
6. Use the **Meta-Learner** tab to understand how models are being blended
7. Add **Analyst Adjustments** to overlay qualitative views on the forecast
8. Track your adjustment accuracy over time in the **Adjustment Journal**
9. Export forecasts to CSV with the ↓ button

> **PADD accuracy note:** Forecast accuracy is highest for U.S. Total due to the volume of EIA data available. Individual PADD regions show higher MAPE because stock levels are smaller and S&D data is sparser. PADD-level demand is estimated via stock-share proportioning of national product supplied data.

---

## V2 Changes

- **Cross-PADD Snapshot** now runs the full ensemble with real S&D features for each PADD, matching Forecast tab calculations exactly
- **Backtest engine** now uses the complete `runEnsemble()` pipeline (meta-learner + regime detection + inverse-MAE weighting) instead of a simple model average
- **Forecast metrics** (MAPE, MAE, direction accuracy) now use S&D features during their backtest validation, consistent with the live forecast
- All tabs share the same ensemble computation path — no divergence between what the Forecast tab shows and what Backtest or Cross-PADD reports

---

## Technical Notes

- **No build tools required.** Single HTML file with CDN dependencies (Chart.js 4.4.4).
- **No backend.** All computation runs client-side in the browser.
- **Data units:** Thousand Barrels (kbbl) throughout, consistent with EIA reporting.
- **Persistence:** Analyst adjustments and journal entries are stored in `localStorage`.

---

## Files

```
gasoline_stock_predictions/
├── Total_Gasoline_Stocks_Model_V2.html      # Total Gas focused model v6.2 (latest)
├── Total_Gasoline_Stocks_Model.html         # Total Gas focused model v6.0
├── gasoline_stock_predictions_v5.html       # Full dashboard (RBOB, CBOB, Total Gas)
├── gasoline_stock_predictions_v4.html       # Prior version
├── gasoline_stock_predictions_v3.html       # Prior version
└── .claude/
    └── launch.json                          # Local dev server config (port 8085)
```

---

## Version History

| Version | Key Changes |
|---------|-------------|
| Total Gas v6.2 | Cross-PADD snapshot uses real S&D features, backtest uses full ensemble pipeline, metrics use S&D-aware validation, weight cache key collision fix |
| Total Gas v6.0 | Total Motor Gasoline focused model with analyst adjustments, adjustment journal, forward-tracking accuracy scoring |
| v5 | Meta-learner with regime detection, stacked generalization, 52-wk CI window, auto-refresh on Run Forecast, Meta-Learner tab, weight cache collision fix |
| v4 | Holt-Winters fallback, inverse-MAE weighted ensemble |
| v3 | AR(6) model added, PADD support |
| v2 | S&D Balance model, 5-year average band |
| v1 | Initial release, S&D Regression only |

---

## Data Source

All data sourced from the **US Energy Information Administration (EIA) API v2**.
API documentation: [https://www.eia.gov/opendata/documentation.php](https://www.eia.gov/opendata/documentation.php)
