# Forecasting Electricity Consumption and CO₂ Intensity in DK1

**MSc Data Science Thesis — Syddansk Universitet (SDU), 2026**  

---

## Overview

This thesis evaluates one-hour-ahead forecasting of **hourly electricity consumption** and **CO₂ emission intensity** in Denmark's DK1 bidding zone (Western Denmark), using 2022–2024 hourly data from Energinet and DMI.

Three models are compared: naïve persistence baselines, linear regression, and gradient boosting. The central finding is an asymmetry — gradient boosting meaningfully improves consumption forecasts but provides no improvement over lag-1 persistence for CO₂ intensity. SHAP analysis is used to explain what drives each model's predictions.

---

## Research Questions

1. How accurately can hourly electricity consumption and CO₂ intensity in DK1 be forecast using ML models?
2. Do gradient boosting models improve forecast accuracy over linear regression and naïve baselines?
3. Which input variables contribute most to predictive performance?

---

## Data Sources

| Source | Content | Resolution |
|--------|---------|------------|
| Energinet | Production, consumption, cross-border exchange (28 variables) | Hourly |
| Energinet | CO₂ intensity (gCO₂/kWh) | 5-min → hourly |
| DMI Station 06072 | Temperature, wind speed, solar radiation | 10-min → hourly |

Final dataset: **26,135 rows × 28 model-ready columns** (Jan 2022 – Dec 2024)

---

## Methodology

### Train / Validation / Test Split
| Set | Period | Rows |
|-----|--------|------|
| Train | Jan 2022 – Sep 2023 | 15,143 |
| Validation | Oct – Dec 2023 | 2,208 |
| Test | All of 2024 | 8,784 |

Strict chronological splitting — no random shuffling to prevent leakage from autocorrelated time series (ACF lag-1 = 0.965).

### Two Modeling Specifications
- **Full model** — includes lag-1 of target variable. Maximises accuracy. Answers RQ1 & RQ2.
- **Diagnostic no-lag-1 model** — excludes lag-1. Forces other features to compete. Answers RQ3.

### Models
- Naïve lag-1 persistence (`ŷ(t+1) = y(t)`)
- Naïve lag-24 persistence (`ŷ(t+1) = y(t-23)`)
- Linear Regression (OLS)
- Gradient Boosting (`sklearn.GradientBoostingRegressor`)

### Feature Engineering
- Lag features: lag-1, lag-24, lag-168 for both targets
- Aggregated generation: TotalWind, TotalSolar, TotalRenewables, TotalConventional
- Exchange: NetExchange, TotalImports, TotalExports
- Calendar: hour, day_of_week, month, is_weekend, season (one-hot)
- Lagged system state: lag-1 versions of wind, solar, production, imports, exports

---

## Results

### Electricity Consumption

| Model | Test MAE (MWh) | Test RMSE (MWh) | vs Lag-1 |
|-------|---------------|----------------|----------|
| Lag-1 persistence | 102.63 | 133.46 | — |
| Lag-24 persistence | 206.90 | 285.46 | −101.6% |
| Linear Regression | 85.33 | 111.62 | +16.9% |
| Gradient Boosting (tuned) | **65.13** | **92.49** | **+36.5%** |

Average test consumption: 2,748.6 MWh. GB tuned error = **2.37% of average demand**.

### CO₂ Intensity

| Model | Test MAE (gCO₂/kWh) | Test RMSE | vs Lag-1 |
|-------|-------------------|-----------|----------|
| Lag-1 persistence | **15.52** | 24.06 | — |
| Lag-24 persistence | 52.47 | 68.65 | −238.2% |
| Linear Regression | 16.01 | 23.31 | −3.2% |
| Gradient Boosting (tuned) | 15.70 | **23.18** | −1.2% |

Every trained model is **worse than lag-1 in MAE**. This is the central negative result — not a modeling failure, but a reflection of CO₂ being governed by wind on atmospheric timescales that lagged features cannot anticipate.

---

## Key Finding: The Asymmetry

**Consumption** is shaped by human behaviour — heating, working hours, cooking — which follows weekly and daily rhythms. Gradient boosting exploits the U-shaped temperature–demand relationship and time-of-day interactions to improve over persistence.

**CO₂ intensity** is determined by instantaneous wind generation, which follows atmospheric dynamics, not the clock. Once you know last hour's CO₂ value, you already know the current renewable penetration state as accurately as any lagged feature can tell you. The information ceiling is inherent, not a model limitation.

---

## SHAP Analysis

- **Full model**: lag-1 dominates 95–98% of importance, masking all other features.
- **Diagnostic no-lag-1 model** (the meaningful result):
  - *Consumption*: lagged net exchange and total production emerge as dominant predictors
  - ![Electricity Consumption SHAP](https://github.com/sifatsami/dk1-electricity-forecasting/blob/main/Consumption.png?raw=true)
  - *CO₂*: `conventional_lag1` and `renewables_lag1` dominate — physically coherent with DK1's generation mix
  - ![CO2 Intensity SHAP](https://github.com/sifatsami/dk1-electricity-forecasting/blob/main/CO2.png?raw=true)

> ⚠️ SHAP values describe how the model uses its inputs — not causal relationships.


---

## Tech Stack

![Python](https://img.shields.io/badge/Python-3.10-blue)
![scikit-learn](https://img.shields.io/badge/scikit--learn-GradientBoosting-orange)
![SHAP](https://img.shields.io/badge/SHAP-Explainability-green)
![pandas](https://img.shields.io/badge/pandas-Data%20Processing-lightblue)

- **Python**, pandas, NumPy
- **scikit-learn** — GradientBoostingRegressor, LinearRegression
- **SHAP** — feature importance and explainability
- **Matplotlib / Seaborn** — visualization
- **Data**: Energinet Open Data, DMI Climate Data

---

## Limitations

- Observed weather values used instead of NWP forecast output — results represent an upper bound on operational performance
- Single chronological test split (rolling-origin evaluation is future work)
- Deep learning models (LSTM, TFT) and probabilistic forecasting not explored
- No spatial variation in weather across DK1

---

## Future Work

1. Use actual weather forecast output for true operational evaluation
2. Extend to longer horizons (2h, 4h) where lag-1 degrades faster
3. Probabilistic forecasting for prediction intervals
4. Test whether CO₂ findings generalise to other high-renewable bidding zones
