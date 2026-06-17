# Retail Sales Forecasting Pipeline
### End-to-end data cleaning, outlier handling, and multi-model time series forecasting on 50,000+ retail transactions

---

## Overview

This project builds a complete forecasting pipeline on a modified Superstore retail dataset — from raw messy data to production-ready 2018 sales forecasts for the five most profitable product subcategories.

The pipeline covers three distinct stages: data cleaning and preparation, outlier detection and handling (with a quantitative method comparison), and model selection via rolling-origin backtesting across three competing forecasting approaches. A separate Excel-based implementation using `FORECAST.ETS` is also included as a baseline reference.

**Key result:** Python models reduced average MAPE by 2–5 percentage points versus the Excel baseline, with the largest improvement on Binders (43.66% → 41.82%) and Phones (28.24% → 22.62%).

---

## Repository Structure

```
superstore-sales-forecasting/
│
├── content/
│   └── Superstore_Raw_Data_with_Trade_terms_Second_Project_Need_Cleaning.xlsx
│
├── notebooks/
│   ├── 01_data_cleaning.ipynb          # Full cleaning pipeline with outlier evaluation
│   └── 02_forecasting_models.ipynb     # Model training, backtesting, and 2018 forecasts
│
├── excel/
│   └── Second_Project_ExcelSheet.xlsx  # Excel FORECAST.ETS baseline implementation
│
├── reports/
│   ├── data_cleaning_report.pdf        # Methodology and results for cleaning phase
│   └── forecasting_report.pdf          # Full forecasting analysis and managerial implications
│
├── outputs/                            # Generated plots and CSVs (populated on run)
│
└── README.md
```

---

## The Data

- **Source:** Modified Superstore retail dataset (29 columns, 50,004 rows)
- **Period:** January 2014 – December 2017 (192 rows from 2018 excluded as incomplete)
- **Key columns:** `Order Date`, `Ship Date`, `Category`, `Sub-Category`, `Sales`, `Profit`, `COGS`, `Ship Mode`, `Country of Origin`, `Tariff %`, `Transportation_Cost`, `Inventory_Cost`, `FX_Gain_Loss`

The dataset includes supply chain cost fields (tariffs, transportation, FX) not present in the standard public Superstore dataset, making it more representative of a real global retail operation.

---

## Stage 1 — Data Cleaning (`01_data_cleaning.ipynb`)

A deterministic, logged, end-to-end cleaning pipeline with the following steps:

| Step | Action | Result |
|------|--------|--------|
| Duplicate removal | Business key: `[Order ID, Product ID, Order Date]` | 170 rows removed |
| Missing value imputation | `sales`: median by sub-category; `ship_mode`: inferred from ship lag | 3 nulls resolved |
| 2018 exclusion | Dropped rows with any 2018 date | 192 rows removed |
| Type standardization | Dates → datetime; categories → string; sales/profit/COGS → float | — |
| Text normalization | Title-case, whitespace strip, canonical `ship_mode` labels | — |
| Discount clipping | Clipped to `[0, 1]`; negatives set to zero | — |
| **Final clean shape** | | **49,642 rows × 29 columns** |

### Outlier Detection

Outliers flagged using IQR ±1.5 (groups ≥ 20 rows) and MAD ±3σ (smaller groups), grouped by `(sub_category, region)`:

- `sales`: 5,861 flagged (~11.7%)
- `profit`: 5,849 flagged (~11.7%)
- `cogs`: 5,860 flagged (~11.8%)

### Outlier Handling — Method Comparison

Three correction methods were compared using a 5% randomly masked holdout (seed = 42):

| Method | MAE | MAPE | Skew After |
|--------|-----|------|------------|
| Winsorization (IQR/MAD cap) | 206.56 | 2.39% | 1.22 |
| Linear Interpolation | 263.04 | 7.37% | 1.43 |
| **LOWESS (selected)** | **206.52** | **2.31%** | 1.88 |

**LOWESS** was selected as the final method — it achieved the lowest MAE and MAPE while preserving seasonal curvature. Skewness was reduced from **12.98 → ~1.2** after cleaning.

---

## Stage 2 — Forecasting (`02_forecasting_models.ipynb`)

### Subcategory Selection

Top 5 subcategories selected by total historical profit:

| Rank | Subcategory | Total Profit (USD) |
|------|-------------|-------------------|
| 1 | Phones | $427,270 |
| 2 | Chairs | $365,413 |
| 3 | Storage | $331,206 |
| 4 | Binders | $302,054 |
| 5 | Machines | $239,108 |

### Models Compared

| Model | Description |
|-------|-------------|
| **Seasonal Naïve** | Repeats same-month value from prior year — benchmark |
| **Holt-Winters** | Additive trend + additive 12-month seasonality, parameters auto-optimized |
| **XGBoost** | Lags 1–12, rolling means (3/6/12 months), calendar features (year, month, sin/cos encodings) |

### Backtesting Strategy

Rolling-origin cross-validation across 3 folds within 2014–2017:
- Fold 1: Train → 2015-12, Test → 2016 H1
- Fold 2: Train → 2016-12, Test → 2017 H1
- Fold 3: Train → 2017-06, Test → 2017 H2

Final model selected per subcategory by **lowest average Test RMSE** (MAPE as tiebreaker).

### Results

| Subcategory | Selected Model | MAE | RMSE | MAPE |
|-------------|---------------|-----|------|------|
| Phones | Holt-Winters | 7,601 | 8,705 | 22.62% |
| Chairs | Seasonal Naïve | 9,495 | 10,530 | 25.69% |
| Storage | Holt-Winters | 5,012 | 6,482 | 22.89% |
| Binders | Holt-Winters | 8,401 | 9,510 | 41.82% |
| Machines | Holt-Winters | 9,956 | 13,632 | 43.68% |

**XGBoost** showed near-zero Train RMSE (strong overfitting on short sequences) and performed worst on test folds across all subcategories despite hyperparameter tuning — a useful reminder that low training error does not indicate model quality.

---

## Stage 3 — Excel Baseline (`excel/`)

An independent `FORECAST.ETS` implementation in Excel covering the same five subcategories with 95% confidence intervals over a 24-month horizon. This was completed prior to the Python implementation and serves as a direct accuracy benchmark.

**Excel vs Python MAPE comparison:**

| Subcategory | Excel MAPE | Python MAPE | Improvement |
|-------------|-----------|-------------|-------------|
| Phones | 28.24% | 22.62% | −5.62 pp |
| Storage | 24.26% | 22.89% | −1.37 pp |
| Chairs | 24.47% | 25.69% | +1.22 pp |
| Binders | 43.66% | 41.82% | −1.84 pp |
| Machines | 54.54% | 43.68% | −10.86 pp |

---

## Tech Stack

```
Python 3.x
├── pandas, numpy
├── statsmodels (Holt-Winters / ETS)
├── xgboost
├── scikit-learn (MAE, MSE)
├── matplotlib
└── scipy / statsmodels (LOWESS)

Excel
└── FORECAST.ETS, FORECAST.ETS.CONFINT, PivotTables, Power Query
```

---

## How to Run

```bash
# Clone the repo
git clone https://github.com/Manaswini-Gopala/superstore-sales-forecasting.git
cd superstore-sales-forecasting

# Install dependencies
pip install pandas numpy matplotlib statsmodels xgboost scikit-learn

# Open notebooks in order
jupyter notebook notebooks/01_data_cleaning.ipynb
jupyter notebook notebooks/02_forecasting_models.ipynb
```

> The notebooks were originally developed in Google Colab. The data folder in this repo is named `content/` to match the `/content/` paths used inside the notebooks — no code changes needed to run locally.

---

## Key Takeaways

- Data cleaning took more effort than modeling — business key deduplication, Excel serial date parsing, ship mode inference, and 2018 exclusion were all non-trivial
- LOWESS outperformed both winsorization and linear interpolation on a masked holdout, but only marginally — the real value was in seasonal curvature preservation
- XGBoost overfits aggressively on monthly subcategory series with only 48 observations per fold; classical methods like Holt-Winters are more appropriate at this scale
- Holt-Winters won 4 of 5 subcategories; Seasonal Naïve won Chairs — suggesting Chairs follows a highly stable year-over-year pattern with minimal structural drift

---

## Academic Context

Developed as part of the **Global Supply Chain Analytics** course, MS in Management Information Systems, San Diego State University (2025).

---
