# Store Sales Forecasting — Favorita (Ecuador)

**16-day-ahead daily demand forecasting across 1,782 store × product-family series**, built with the discipline of a production replenishment system — not a leaderboard sprint.

> **Headline:** validation RMSLE **0.384**, leaderboard **0.426** — a fully diagnosed climb from an initial leaking baseline of 0.515. Every basis point of that improvement is traceable to a specific, evidence-tested decision.

---

## Demand Classification (ADI/CV²)

![Demand classification](images/eda_16_demand_classification.png)

1,782 series segmented into SMOOTH / ERRATIC / INTERMITTENT / LUMPY using Syntetos-Boylan thresholds. ~57% of total forecast error concentrated in the LUMPY/INTERMITTENT classes — genuinely sparse demand where a single global regressor hits a hard ceiling.

---

## Why this project

Retail demand planning is where I've spent my career — building production forecasting systems for Alshaya and Landmark Group. This project brings that same rigor to a public forecasting problem: not "what score can I chase," but "what forecast could I actually *commit inventory* against." That distinction drove every decision here.

The dataset: Corporación Favorita, a large Ecuadorian grocery retailer — daily unit sales by store and product family, with promotions, oil prices, holidays, and store metadata. Scored on **RMSLE**.

---

## The core lesson: my validation was lying to me

My first pipeline scored **0.38 in validation but 0.51 on the leaderboard** — a gap that would be catastrophic in a real planning cycle. The cause was subtle: the model leaned on **recency features (recent lags) that don't exist at prediction time**. In backtest they were available (the holdout sat inside the training data); at true forecast time, they silently collapsed to static averages.

**The fix — and the centerpiece of this project — was a walled backtest ("harness")** that reproduces true forecast-time conditions: features rebuilt using only data available on or before the forecast wall. Rebuilding the entire feature set **horizon-safe** (lags ≥ 16 days, rolling windows offset ≥ 16 days) closed most of the gap and made the model's training and serving conditions identical.

---

## Model: what actually drives the forecast

![Feature importance](images/model_01_lgb_feature_importance.png)

Recent rolling sales (`sales_roll_7`) and `sales_lag_21` dominate — confirmed by a deliberate ablation test where I stripped the model down to *only* sales history and calendar skeleton (no oil, no promo, no holidays). That lean model scored **0.455**, meaningfully worse than the full 52-feature model's **0.384** — proving the "minor" contextual features carry real signal collectively, even though none dominates individually.

---

## Domain-specific signals

![Earthquake impact](images/eda_07_earthquake_impact.png)
![Payday effect](images/eda_08_payday_effect.png)

The April 2016 Ecuador earthquake caused a visible sales surge (ruled out as an August confound after checking the date — the quake was 4 months prior). Payday effects (15th / month-end) show clear spikes, consistent with Ecuador's bi-monthly wage cycle.

---

## Approach

| Step | Decision | Why |
|------|----------|-----|
| **Feature engineering** | Horizon-safe lags & rolling means; ADI/CV² demand classification | Eliminate train-vs-serve leakage; segment error by demand behaviour |
| **Validation** | Walled backtest reproducing test-time feature availability | Standard CV was optimistic; this gated every change |
| **Objective** | Train directly in **log space** (RMSLE ≡ RMSE on log1p) | Optimise the exact competition metric, not a Tweedie proxy |
| **Model** | LightGBM, **5-seed bagged**, refit through the last known day | Gradient boosting + variance reduction |

**Result:** validation RMSLE **0.384**.

---

## Data
Download from the [Kaggle competition page](https://www.kaggle.com/competitions/store-sales-time-series-forecasting/data) and place CSVs in a `Data/` folder to run the notebook locally.

---

## Diagnostic journey

| Stage | Score |
|-------|-------|
| Initial pipeline (ordering + leakage bugs) | ~3.5 |
| Leakage fixed, still recency-dependent | 0.515 |
| Horizon-safe features | 0.505 |
| Transaction features removed | 0.431 |
| Log-space training + 5-seed bagging | **0.426** |

---

## What I tested and rejected — with evidence

- **Transaction (foot-traffic) features** — *dropped, and removing them improved the score.* Transactions ≠ units (basket size confound).
- **Hierarchical reconciliation** — built Pareto store-grading, a parent forecast model, and an **oracle-ceiling test**. Proved Pareto tiers don't smooth enough to produce a trustworthy parent, so bottom-up dominates.
- **Recursive forecasting, product-family encoding, promo lead/lag features, grade×day-of-week seasonality, multi-origin training, zero-forcing, two-stage intermittent model, sales-history-only ablation** — each tested on the harness, each rejected for degrading it.

---

## A note on rigor: adaptive overfitting

The gap between the final backtest (0.39) and leaderboard (0.43) is instructive. Early, the harness tracked the board to within 0.002; after 15+ experiments validated against the *same* holdout window, that neutrality eroded — **adaptive overfitting to the validation set.** In production, the fix is rotating validation windows. Surfacing this mattered more than the rank.

---

## Tech stack

`Python` · `pandas` · `NumPy` · `LightGBM` · `gradient boosting` · `time-series forecasting` · `demand forecasting` · `feature engineering` · `leakage detection` · `cross-validation` · `hierarchical forecasting` · `RMSLE`

## About

**Sagar Sharma** — AI & Analytics consultant with deep retail demand & supply planning experience (ex-Landmark Group, ex-Alshaya Group). IBM Data Science Professional Certificate. [LinkedIn](https://www.linkedin.com/in/sagar-sharma-a6211589/) · [Kaggle](https://www.kaggle.com/code/sagarsharmafeb87/favorita-showcase-final)
