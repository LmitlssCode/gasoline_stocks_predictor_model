# Gasoline Stock Predictions v5

A single-file, browser-based dashboard for forecasting weekly US gasoline inventory levels using EIA petroleum data. Built for commodity traders and analysts tracking RBOB, CBOB, and total motor gasoline stocks across US PADD regions.

---

## Overview

This tool fetches live weekly petroleum data from the EIA API v2 and runs a stacked ensemble forecast model to predict gasoline stock levels 1–4 weeks forward. The ensemble combines three fundamental models under a meta-learner that adapts its blending weights based on the current market regime.

**Products supported:**
- Total Motor Gasoline (EPM0)
- RBOB Gasoline (EPOBGRR)
- CBOB Gasoline (EPOBGCC)

**Regions supported:**
- US Total (NUS)
- PADD 1 (East Coast)
- PADD 2 (Midwest)
- PADD 3 (Gulf Coast)
- PADD 4 (Rocky Mountain)
- PADD 5 (West Coast)

---

## How It Works

### Data Inputs

All data is pulled from the EIA API v2 (`api.eia.gov/v2/petroleum/`):

| Series | Description |
|--------|-------------|
| Weekly stocks | Primary target variable (kbbl) |
| Production | Domestic refinery output |
| Imports / Exports | Trade flows |
| Refinery utilization | Capacity utilization rate |
| Demand (product supplied) | Implied consumption |

Data is cached in memory with a 30-minute TTL and auto-refreshes when you click **Run Forecast**.

---

### Prediction Models

Three primary models form the base layer of the ensemble:

#### 1. S&D Regression
A ridge regression (λ = 1.0) trained on supply-and-demand features. Uses forward stepwise feature selection to choose the most predictive subset from ~10 candidate features: lagged production, import delta, export delta, refinery utilization change, demand delta, and their recent trends. When fewer than 52 weeks of data are available, falls back to a simpler OLS on available data.

#### 2. S&D Balance
A seasonally-aware projection model. Computes the implied weekly stock change from net supply minus demand, then anchors the forecast to same-week-last-year stock levels, blending recent build/draw trajectory with seasonal norms. This model is particularly useful during large seasonal turning points.

#### 3. AR(6) Autoregressive Model
An autoregressive model using the prior 6 weeks of stock levels. Captures momentum and mean reversion. Acts as a pure time-series baseline and serves as the fallback when S&D data is sparse.

**Fallback models** (used when S&D data is unavailable):
- **Seasonal Naive**: Projects same-week-last-year stock level forward
- **Holt-Winters**: Exponential smoothing with trend and seasonality components

---

### Regime Detection

Before blending, the model detects the current market regime using five signals:

| Feature | Description |
|---------|-------------|
| Season | Categorical: build (Oct–Mar), draw (Apr–Sep), peak (Jul–Aug), shoulder (Sep/Apr) |
| Volatility ratio | 8-week vs. 52-week realized stock volatility |
| Turnaround flag | Binary: refinery utilization dropped >3 pts over 4 weeks |
| Momentum | Direction and magnitude of recent 4-week stock change |
| Season × Volatility | Interaction term |

These seven regime features are passed to the meta-learner as additional inputs alongside the three base model predictions.

---

### Meta-Learner (Ensemble Model)

The meta-learner is a ridge regression (λ = 0.5) trained on a rolling 30-week backtest. It learns 10 coefficients:

- **1 intercept**
- **3 model blend weights** — how much to trust S&D Regression, S&D Balance, and AR(6) respectively
- **7 regime bias adjustments** — how each regime feature should shift the final prediction up or down

Each week, the meta-learner combines all three base model predictions with the current regime features to produce the final **Ensemble** forecast. It automatically adapts over time as new weekly data is added — no manual tuning is needed across seasons.

When fewer than 30 backtest samples exist (e.g., sparse PADD/product combinations), the meta-learner falls back to an **inverse-MAE weighted average** with exponential decay, giving more recent forecasts higher weight.

---

### Confidence Intervals

80% confidence intervals are computed from a 52-week rolling backtest window. For each historical week in that window, the model generates an out-of-sample forecast and measures the residual. The 10th and 90th percentile of those residuals define the CI band, scaled forward by forecast horizon.

---

## Dashboard Features

### Main Forecast Tab
- Interactive Chart.js time series with 5-year seasonal average band
- Historical stocks, ensemble forecast, and 80% CI shading
- Hover tooltips — the last historical point shows only the actual stock value; model values only appear on forecast dates
- Forecast table: Ensemble, S&D Reg, S&D Balance, AR Model, CI bounds, week-over-week change (all in kbbl)
- CSV export of full forecast table
- Scatter plot: Predicted vs. Actual for backtest window

### Meta-Learner Tab
Provides transparency into how the ensemble is making decisions:
- **Summary stats**: R², MAPE, directional accuracy from 30-week backtest
- **Current regime**: Active regime features and their values
- **Feature vector**: Full 10-feature input into the meta-learner for the current forecast
- **Model blend weights chart**: Coefficients for S&D Reg, S&D Balance, AR(6)
- **Regime bias chart**: All 7 regime coefficients, with active features highlighted
- **Blending walkthrough**: Step-by-step table showing how each component contributes to the final prediction
- **Fallback weights**: Inverse-MAE weights used when meta-learner has insufficient training data

---

## Usage

1. Open `gasoline_stock_predictions_v5.html` in any modern browser
2. Enter your EIA API key (free at [eia.gov/opendata](https://www.eia.gov/opendata/))
3. Select product and PADD region
4. Click **Run Forecast** — data loads and the model runs automatically
5. Adjust forecast horizon (1–4 weeks) as needed
6. Use the **Meta-Learner** tab to inspect model internals
7. Export forecast to CSV if needed

> **Note on performance by region/product:** Forecast accuracy is highest for US Total Motor Gasoline due to the volume of EIA data available. Individual PADD regions and sub-products (RBOB, CBOB) show higher MAPE and lower directional accuracy because stock levels are smaller in absolute terms (amplifying percentage errors), EIA S&D coverage is less complete, and the signals are choppier with more data revisions.

---

## Technical Notes

- **No build tools required.** All dependencies load from CDN (Chart.js, Papa Parse).
- **No backend.** All computation runs client-side in the browser.
- **Weight cache** is keyed by `product_PADD_lastDate_seriesLength` to prevent cross-product/region collisions.
- Data units throughout are **Thousand Barrels (kbbl)**, consistent with EIA reporting.

---

## Files

```
gasoline_model_predictive/
├── gasoline_stock_predictions_v5.html   # Main dashboard (single file)
├── gasoline_stock_predictions_v4.html   # Prior version
├── gasoline_stock_predictions_v3.html   # Prior version
└── .claude/
    └── launch.json                      # Local dev server config (port 8085)
```

---

## Version History

| Version | Key Changes |
|---------|-------------|
| v5 | Meta-learner with regime detection, stacked generalization, 52-wk CI window, auto-refresh on Run Forecast, Meta-Learner tab, weight cache collision fix |
| v4 | Holt-Winters fallback, inverse-MAE weighted ensemble |
| v3 | AR(6) model added, PADD support |
| v2 | S&D Balance model, 5-year average band |
| v1 | Initial release, S&D Regression only |

---

## Data Source

All data sourced from the **US Energy Information Administration (EIA) API v2**.  
API documentation: [https://www.eia.gov/opendata/documentation.php](https://www.eia.gov/opendata/documentation.php)
