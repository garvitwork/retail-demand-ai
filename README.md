# AI-Powered Retail Demand Forecasting & Inventory Optimization

An end-to-end data pipeline that predicts product-level demand for a real UK-based online retailer and flags restock risk — built with Python (ML), MySQL (storage), and Power BI (visualization & DAX).

---

## Table of Contents
- [Problem Statement](#problem-statement)
- [Why This Matters](#why-this-matters)
- [Tech Stack](#tech-stack)
- [Architecture / Data Flow](#architecture--data-flow)
- [Dataset](#dataset)
- [Approach](#approach)
- [Key Metrics](#key-metrics)
- [Challenges Faced & How They Were Solved](#challenges-faced--how-they-were-solved)
- [Dashboard Pages](#dashboard-pages)
- [Repository Structure](#repository-structure)
- [How to Run This Project](#how-to-run-this-project)
- [Future Improvements](#future-improvements)
- [Author](#author)

---

## Problem Statement

Retailers routinely lose revenue in two opposite ways: **stockouts** (a popular product runs out, and the sale is lost to a competitor) and **overstock** (capital gets tied up in slow-moving inventory that eventually gets marked down or written off). Most small-to-mid retailers still make reorder decisions on gut feel or simple last-month averages, not on demand patterns.

This project builds a demand forecasting system that:
1. Learns historical sales patterns per product using machine learning
2. Predicts near-future demand
3. Flags which products need restocking vs. which have sufficient stock
4. Presents everything in a decision-ready Power BI dashboard for a retail operations manager

---

## Why This Matters

This isn't a toy dataset exercise — it mirrors a real operational workflow used in retail, e-commerce, and FMCG supply chains: **transactional data → forecasting engine → inventory decision support**. The same pipeline structure (raw data → database → ML layer → BI layer) is what production analytics teams run in industry, just at a larger scale.

---

## Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| Data Source | Online Retail II (UCI ML Repository) | Real transactional sales data, 2009–2011 |
| Data Cleaning & Feature Engineering | Python (pandas, numpy) | Cleaning, aggregation, time-series features |
| Machine Learning | XGBoost, scikit-learn | Demand forecasting model |
| Database | MySQL | Central storage layer between Python and Power BI |
| Automation / Connectivity | SQLAlchemy, PyMySQL | Python ↔ MySQL data transfer |
| Visualization | Power BI Desktop | Dashboard, DAX measures, AI visuals |
| Version Control | Git & GitHub | Project tracking and portfolio hosting |

---

## Architecture / Data Flow

```
                  ┌───────────────────────┐
                  │  Raw Excel Data       │
                  │  (Online Retail II)   │
                  └──────────┬────────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │  Python: Cleaning     │
                  │  - Remove nulls       │
                  │  - Remove cancels     │
                  │  - Fix invalid rows   │
                  └──────────┬────────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │  MySQL: sales_clean   │
                  └──────────┬────────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │  Python: Feature Eng. │
                  │  - Daily/weekly demand│
                  │  - Lag & rolling feats│
                  └──────────┬────────────┘
                             │
                             ▼
                  ┌────────────────────────────┐
                  │  MySQL: daily_demand_       │
                  │  features                   │
                  └──────────┬───────────────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │  Python: XGBoost      │
                  │  Model Training +     │
                  │  Time-Series CV       │
                  └──────────┬────────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │  MySQL: demand_        │
                  │  forecast              │
                  └──────────┬────────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │  Power BI              │
                  │  - Star schema model   │
                  │  - DAX measures        │
                  │  - AI visuals          │
                  │  - 3 dashboard pages   │
                  └───────────────────────┘
```

---

## Dataset

**Source:** [Online Retail II — UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/502/online+retail+ii)

- Real transactional data from a UK-based online retailer
- Time period: December 2009 – December 2011
- ~1 million rows across 2 sheets (`Year 2009-2010`, `Year 2010-2011`)
- Fields: Invoice, StockCode, Description, Quantity, InvoiceDate, Price, Customer ID, Country

---

## Approach

**1. Data Cleaning (Python)**
- Removed rows with missing `Customer ID` (can't attribute sales without a customer)
- Removed cancelled orders (`Invoice` starting with "C")
- Removed invalid rows (negative/zero quantity or price)
- Engineered `total_sales = quantity × price`

**2. Feature Engineering (Python)**
- Aggregated transactions into **weekly demand per product** (daily was too volatile for reliable prediction — explained below)
- Filtered to top-selling products by volume (removes noise from rarely-sold items, mirrors real business prioritization)
- Built lag features (`lag_1`, `lag_2`, `lag_4`), rolling mean/std, and exponentially weighted mean
- Log-transformed the target variable to handle demand spikes

**3. Model Training (Python + XGBoost)**
- Time-based train/test split (never random — this is a time-series problem)
- Hyperparameter tuning via grid search combined with `TimeSeriesSplit` cross-validation
- Evaluated using **WAPE-based accuracy** (Weighted Absolute Percentage Error), more business-relevant than raw MAE
- Only pushed results to production (MySQL) once accuracy met an acceptable business threshold

**4. Storage Layer (MySQL)**
- Three structured tables: `sales_clean`, `daily_demand_features`, `demand_forecast`
- Acts as the single source of truth between the Python ML layer and the Power BI reporting layer

**5. Business Logic (Python)**
- Reorder flag: a product is marked **"Restock Needed"** if predicted demand exceeds its recent rolling average by more than 20%

**6. Dashboard (Power BI)**
- Star-schema data model (`dim_products`, `DateTable`, 3 fact tables)
- 20+ DAX measures covering sales KPIs, forecast accuracy, and inventory risk
- AI visuals (Key Influencers, Decomposition Tree) to explain *why* products are flagged for restock
- 3 report pages: Executive Overview, Inventory & Restock Insights, AI Forecast Deep Dive

---

## Key Metrics

| Metric | Value |
|---|---|
| Forecast Accuracy (1 − WAPE) | ~80%+ |
| Products Forecasted | Top-selling SKUs by volume |
| Forecast Granularity | Weekly, per product |
| Restock Detection Logic | Predicted demand > 1.2 × rolling average |

*(Update this table with your final numbers once the model run is finalized.)*

---

## Challenges Faced & How They Were Solved

| Challenge | Root Cause | Solution |
|---|---|---|
| Special character in MySQL password broke the connection string | `@` in password was misread as part of the host URL | Used `urllib.parse.quote_plus()` to URL-encode the password |
| `dim_products` relationship failed with duplicate key error | Same `stockcode` had multiple inconsistent `description` text values | Used `Table.Group` in Power Query to guarantee one row per `stockcode` |
| Matrix visual couldn't relate `sales_clean` and `demand_forecast` | Both tables only connect indirectly through `dim_products`, which Power BI couldn't auto-resolve | Built an explicit `CALCULATE` + `CROSSFILTER` DAX measure to force the correct filter path |
| Key Influencers AI visual returned no results | Fields being compared lived in different tables — this visual requires everything in one table | Added calculated columns (`Day of Week`, `Month`) directly inside `demand_forecast` |
| Initial forecast accuracy was too low (~50–60%) on raw daily data | Daily per-SKU demand is extremely volatile; too few features; unfiltered long-tail products added noise | Aggregated to weekly demand, filtered to top-selling products, added stronger lag/rolling/EWM features, log-transformed the target, and tuned hyperparameters with time-series cross-validation |

This section exists intentionally — a genuine project always hits real obstacles. Documenting them (rather than hiding them) demonstrates actual hands-on debugging, not a copy-pasted tutorial.

---

## Dashboard Pages

**1. Executive Overview** — Total sales, orders, average order value, monthly trend, top products, sales by country.

**2. Inventory & Restock Insights** — Restock rate, stockout/overstock risk counts, product-level reorder table with conditional formatting, demand volatility scatter plot, restock pressure by country and month.

**3. AI Forecast Deep Dive** — Forecast accuracy, actual vs. predicted demand trend, Key Influencers (what drives restock risk), Decomposition Tree (demand breakdown by country/product/month).

---

## Repository Structure

```
retail-demand-ai/
├── data/
│   ├── raw/                     # Original Excel dataset
│   └── processed/               # (optional) intermediate exports
├── notebooks/
│   ├── 01_explore.ipynb         # Data cleaning
│   ├── 02_eda_features.ipynb    # EDA + feature engineering
│   └── 03_forecast_model.ipynb  # Model training + MySQL push
├── powerbi/
│   └── retail_demand_dashboard.pbix
├── .env                          # MySQL credentials (not committed)
├── .gitignore
└── README.md
```

---

## How to Run This Project

1. Clone this repository
2. Install dependencies:
   ```
   pip install pandas numpy xgboost scikit-learn sqlalchemy pymysql python-dotenv openpyxl
   ```
3. Set up a local MySQL database named `retail_demand_ai`
4. Create a `.env` file with your MySQL credentials (see `.env.example`)
5. Download the [Online Retail II dataset](https://archive.ics.uci.edu/dataset/502/online+retail+ii) into `data/raw/`
6. Run notebooks in order: `01 → 02 → 03`
7. Open `powerbi/retail_demand_dashboard.pbix` in Power BI Desktop, connect to your local MySQL instance, and refresh

---

## Future Improvements

- Automate the pipeline end-to-end with a scheduler (Airflow or Windows Task Scheduler)
- Add a live Power BI Service refresh via a Gateway connection to MySQL
- Extend forecasting to a full product catalog using a hierarchical/grouped model instead of per-SKU
- Add external features (holidays, promotions) to improve seasonal accuracy
- Deploy the model as an API endpoint for real-time reorder recommendations

---

## Author

Built as an end-to-end portfolio project demonstrating the full data lifecycle: raw data → database engineering → machine learning → business intelligence.

Connect with me on [LinkedIn] — happy to walk through the technical decisions behind this project.
