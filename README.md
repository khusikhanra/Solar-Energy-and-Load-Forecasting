<div align="center">

# ☀️ Solar Energy & Load Forecasting

**Time-series forecasting of national electricity load and solar generation, paired with an interactive grid-analytics dashboard.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Streamlit](https://img.shields.io/badge/Streamlit-App-FF4B4B?logo=streamlit&logoColor=white)](https://streamlit.io/)
[![statsmodels](https://img.shields.io/badge/statsmodels-ARIMA%20%2F%20SARIMAX-4C72B0)](https://www.statsmodels.org/)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-ML%20Pipeline-F7931E?logo=scikit-learn&logoColor=white)](https://scikit-learn.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

</div>

<p align="center">
  <img src="assets/architecture-overview.png" alt="Repository architecture: notebook pipeline and dashboard" width="850">
</p>

---

## Contents

- [Overview](#overview)
- [What's actually in this repo (read this first)](#whats-actually-in-this-repo-read-this-first)
- [Repository structure](#repository-structure)
- [Dataset](#dataset)
- [Notebook: modeling methodology](#notebook-modeling-methodology)
- [Dashboard: module tour](#dashboard-module-tour)
- [Getting started](#getting-started)
- [Roadmap](#roadmap)
- [Limitations & honest caveats](#limitations--honest-caveats)
- [Tech stack](#tech-stack)
- [Author](#author)
- [License](#license)

---

## Overview

This repository explores hourly electricity **load** and **solar generation** forecasting for Italy (2016) from two angles:

1. **`Time_Series_ARIMA.ipynb`** — a research notebook that takes the real dataset through stationarity testing, ACF/PACF diagnostics, classical time-series models (ARIMA, seasonal SARIMAX), and a gradient-boosted machine-learning pipeline with permutation-importance explainability. The trained pipeline is serialized to disk.
2. **`app.py`** — a 10-module Streamlit dashboard that demonstrates what a production grid-analytics console can look like: calibrated forecast intervals, Monte Carlo risk simulation, anomaly detection, battery-arbitrage optimization, a transformer digital twin, and a couple of exploratory extras (a backtested trading strategy and a from-scratch quantum-computing simulator).

## What's actually in this repo (read this first)

To keep this README honest rather than aspirational, here's exactly how the two pieces relate:

> **The dashboard does not currently consume the CSV or a trained model.** `app.py` generates its own seeded, synthetic 8,760-hour grid dataset at runtime (`build_grid_dataset`) and fits a lightweight ridge-regression model in-memory (`train_model`) purely so the app is self-contained and runs with zero setup. `TimeSeries_TotalSolarGen_and_Load_IT_2016.csv` is used **only** inside the notebook. Running the notebook end-to-end also serializes `forecasting_pipeline_model.pkl` / `model_features.pkl` to disk — those two files are build artifacts, not part of this repo (they're `.gitignore`-friendly and regenerated locally whenever you re-run the notebook).

This is a reasonable design for a demo (no external data dependency, fully reproducible via a random seed), but it means **the dashboard's numbers describe simulated grid behavior, not the real Italian 2016 dataset.** If you want the dashboard to score real data through the real trained pipeline, see [Roadmap](#roadmap).

## Repository structure

```
Solar-Energy-and-Load-Forecasting/
├── app.py                                        # Streamlit dashboard (10 analytics modules)
├── Time_Series_ARIMA.ipynb                       # ARIMA / SARIMAX / ML research notebook
├── TimeSeries_TotalSolarGen_and_Load_IT_2016.csv # Hourly Italy load + solar, 2016 (8,784 rows)
├── requirements.txt                              # Pinned dependencies for both components
├── LICENSE                                       # MIT
└── README.md

# Generated locally when you run the notebook (not committed to the repo):
#   forecasting_pipeline_model.pkl   — trained HistGradientBoostingRegressor
#   model_features.pkl               — feature list matching the pickled pipeline
```

## Dataset

`TimeSeries_TotalSolarGen_and_Load_IT_2016.csv` — hourly national telemetry for Italy, 2016.

| Column                | Description                                  | Units |
|-----------------------|-----------------------------------------------|-------|
| `utc_timestamp`       | Hourly timestamp, UTC                         | ISO 8601 |
| `IT_load_new`         | National electricity load                     | MW |
| `IT_solar_generation` | National solar generation                     | MW |

8,784 rows (2016 was a leap year → 366 × 24 h). This is the same shape of series published by the [Open Power System Data](https://open-power-system-data.org/) time-series project, a common source for European load/generation data.

## Notebook: modeling methodology

`Time_Series_ARIMA.ipynb` walks through:

1. **Preprocessing** — parse timestamps, set a `DatetimeIndex`, forward/back-fill gaps.
2. **Stationarity diagnostics** — Augmented Dickey-Fuller test on both series; ACF/PACF plots to inform model order.
3. **Classical forecasting**
   - `ARIMA(2, 0, 2)` on load, evaluated on a chronological 80/20 split with RMSE.
   - `SARIMAX(1,1,1)(1,1,1,24)` to capture the 24-hour seasonal cycle over a 14-day training window.
   - A monthly-resampled `SARIMAX(1,1,1)(1,1,1,12)` revenue model with a 12-month forecast and confidence band.
4. **Feature-engineered ML pipeline** — calendar features (hour/day/month, cyclical encodings), lag and rolling-window features, trained with `HistGradientBoostingRegressor` on a chronological 80/20 split.
5. **Explainability** — permutation importance ranks which features actually move the ML model's predictions.
6. **Persistence** — the fitted pipeline and its feature list are serialized with `joblib` to `forecasting_pipeline_model.pkl` / `model_features.pkl`, then reloaded and verified with a sample prediction.

Exact RMSE / MAE / R² values depend on your environment and aren't hardcoded here — run the notebook to reproduce them locally.

## Dashboard: module tour

`app.py` ships ten tabs, each backed by a real (if intentionally simplified) computation over the synthetic dataset described above:

| # | Module | What it actually does |
|---|--------|------------------------|
| 1 | **Real-Time Predictive Console** | Ridge-regression point forecast with a prediction interval calibrated from holdout residual quantiles (not an assumed z-score band). |
| 2 | **Dynamic Horizon Forecaster** | Recursive 24-hour forecast — each step's prediction feeds the next step's lag features, with an expanding uncertainty band. |
| 3 | **3D Explainable AI (XAI) Mesh** | Permutation importance (or coefficient-based fallback) as a 3D surface, plus a per-feature contribution waterfall (`coefᵢ × (xᵢ − x̄ᵢ)`). |
| 4 | **Monte Carlo Risk Simulator** | Correlated net-load simulation via Cholesky decomposition of the empirical demand/solar/wind correlation matrix, with outage shocks, VaR/CVaR. |
| 5 | **Anomaly Scanner & Stress Test** | `IsolationForest` (with a robust z-score fallback) for anomaly flags; peak-load stress extrapolation via the same cooling-degree response used to build the data. |
| 6 | **Financial & Tariff Optimization** | Greedy battery-arbitrage dispatch: discharge into the highest-priced hours first, recharge in the cheapest, with round-trip efficiency and depth-of-discharge limits. |
| 7 | **Digital Twin Real-Time Simulation** | A first-order thermal ODE (`dθ/dt = (θ_ss − θ)/τ`) for transformer winding temperature, plus a simple frequency/droop model. |
| 8 | **Algorithmic Trading Desk** | A seeded, backtested mean-reversion strategy on a synthetic price path — illustrative, not investment-grade. |
| 9 | **Quantum Computing Simulator** | A genuine from-scratch NumPy statevector simulator (H/X/Z/RX/RZ/CNOT gates, Grover's algorithm, Bloch-sphere rendering) — real linear algebra, clearly labeled as simulation, not connected to real quantum hardware. |
| 10 | **AI Copilot Assistant** | A transparent, rules-based Q&A layer that reads live session state (model metrics, recent peaks, current sliders) and answers with real numbers — not an LLM. |

Global sidebar controls (confidence interval, Monte Carlo run count, currency, random seed, and one of four visual themes) apply across every tab, and everything is deterministic for a given seed.

## Getting started

### Prerequisites
- Python 3.10+
- `pip`

### 1. Clone and install

```bash
git clone https://github.com/khusikhanra/Solar-Energy-and-Load-Forecasting.git
cd Solar-Energy-and-Load-Forecasting
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Run the notebook

```bash
jupyter notebook Time_Series_ARIMA.ipynb
```

Run all cells top to bottom — the last cells regenerate `forecasting_pipeline_model.pkl` and `model_features.pkl`.

### 3. Launch the dashboard

```bash
streamlit run app.py
```

Streamlit will open the app at `http://localhost:8501`. No dataset or model file is required to start it — the synthetic grid engine bootstraps everything on first load.

## Roadmap

- [ ] Wire the dashboard's **Real-Time Predictive Console** to optionally load `forecasting_pipeline_model.pkl` and score the real 2016 CSV, so the demo and the research artifact share one source of truth.
- [ ] Add a toggle to switch the dashboard's data source between "synthetic" and "uploaded CSV."
- [ ] Replace the in-app ridge regression with the notebook's `HistGradientBoostingRegressor` for consistency between the two components.
- [ ] Add automated tests around the feature-engineering functions (`build_grid_dataset`, `engineer_features`).
- [ ] Publish reproducible metric tables (RMSE/MAE/R² per model) generated by CI rather than by hand.

## Limitations & honest caveats

- **Synthetic ≠ real for the dashboard.** All dashboard tabs operate on procedurally generated data designed to *look* like realistic grid telemetry. Treat its numbers as illustrative, not as claims about the actual Italian grid.
- **Ridge regression is intentionally simple.** It's fast, interpretable, and dependency-light (with a pure-NumPy fallback if scikit-learn isn't installed), not a competitor to the notebook's gradient-boosted pipeline.
- **The trading desk and quantum simulator are educational demos**, included to show the underlying math (backtesting mechanics; statevector simulation) rather than to imply production trading or quantum-hardware capability.
- **No live data connection.** Everything runs on historical/synthetic data — there is no real-time SCADA, IoT, or utility API integration.

## Tech stack

| Layer | Tools |
|---|---|
| Statistical modeling | `statsmodels` (ARIMA, SARIMAX), ADF test |
| Machine learning | `scikit-learn` (`HistGradientBoostingRegressor`, `Ridge`, `IsolationForest`, permutation importance) |
| Dashboard | `streamlit`, `plotly` |
| Data handling | `pandas`, `numpy` |
| Persistence | `joblib` |

## Author

**Khusi Khanra**

[![GitHub](https://img.shields.io/badge/GitHub-khusikhanra-181717?style=flat-square&logo=github)](https://github.com/khusikhanra)

## License

Released under the [MIT License](LICENSE).
