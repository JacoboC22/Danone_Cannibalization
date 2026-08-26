🥛 Innovation Cannibalization Analysis
A data science pipeline built at Danone to measure how newly launched products ("innovations") cannibalize sales of existing SKUs within the same category, using time series decomposition and causal impact estimation across retail store-level sales data.
📌 Project Overview
When a company launches a new product, part of its sales often comes from customers who would otherwise have bought a similar existing product — this is called cannibalization. This project builds an end-to-end pipeline that automatically:
Detects which SKUs behave like new product launches ("innovations") in sales data.
Identifies candidate "victim" SKUs in the same category that lose sales after the launch.
Quantifies the causal impact of the innovation on each victim using CausalImpact (with a regression-based fallback), controlling for category trend, seasonality, and day-of-week effects.
Aggregates results at the brand level to answer: which brands cannibalize which, and by how much?
The pipeline was tested across the top 20 highest-selling stores for the yogurt/dairy category in France.
> ⚠️ **Note:** This is a generalized, anonymized version of an internal Danone project. All retailer names, internal database schemas, file paths, and real sales figures have been removed or replaced with generic placeholders. The dataset itself is not included — this repository showcases the **methodology and code**, not proprietary business data.
🎯 Objectives
Automatically detect product launches from raw transactional sales data (no manual launch-date tagging needed).
Identify which existing products lose sales after a new launch, filtering out unrelated causes (discontinuations, insufficient data).
Estimate the causal, not just correlational, effect of a launch on victim SKUs — separating true cannibalization from category-wide trends or seasonality.
Roll up SKU-level effects into brand-vs-brand cannibalization insights across multiple stores.
🔍 Methodology
1. Data Preparation
Joined store-level sales fact tables with product, store, customer, supplier, brand, and calendar dimension tables.
Filtered to a specific market, category, and in-store sales channel.
Selected the top-selling stores by total sales volume.
Built an analytical SKU key (Supplier × Brand × Product Type) to analyze cannibalization at a meaningful business granularity, aggregating individual barcodes.
2. Signal Decomposition (STL)
Applied STL (Seasonal-Trend decomposition using Loess) to each SKU's daily sales to separate trend, weekly seasonality, and residual noise.
Built an availability matrix (`A`), flagging when a SKU was actively selling (trend above threshold) vs. likely out-of-stock.
Decomposed department/category-level totals to build control signals for the causal model (category trend, category seasonality).
3. Innovation Detection
Flagged a SKU as an "innovation" (new launch) if it had no sales in a configurable lookback window (default: 365 days) before its first sale.
Required a minimum post-launch sales strength and minimum number of active selling days to filter out noise/one-off spikes.
4. Victim Candidate Selection
For each detected innovation, scanned other SKUs in the same category for a meaningful, sustained sales drop after the launch date.
Filtered candidates using: minimum pre/post data availability, minimum active post-launch days (to exclude SKUs discontinued for unrelated reasons), and a minimum relative sales drop threshold.
5. Causal Impact Estimation
Built a control matrix per department: category trend, day-of-week dummies, and optional external regressors (e.g., temperature).
Estimated the effect of each innovation on each victim SKU using CausalImpact (Bayesian structural time series), with a linear regression + t-test (Difference-in-Differences style) fallback when CausalImpact wasn't available or usable.
Kept only statistically significant, negative effects (p-value below threshold) as confirmed cannibalization events.
6. Orchestration & Aggregation
Ran the full pipeline independently per store across the top 20 stores.
Consolidated all significant innovation→victim effects into a single results table.
Aggregated to the brand level, computing total units lost, number of affected stores, and average relative drop per brand pair — enabling both "who cannibalizes us" and "who do we cannibalize" views.
🛠️ Tech Stack
Language: Python (PySpark + pandas)
Big data processing: PySpark (Spark SQL, DataFrame joins/aggregations)
Time series decomposition: `statsmodels` (STL)
Causal inference: `causalimpact` (Bayesian structural time series), with `scikit-learn` (LinearRegression) + `scipy` (t-test) as a fallback method
Data manipulation: pandas, numpy
📁 Repository Structure
```
├── Baseline_innovations20stores.ipynb   # Full pipeline: data prep, decomposition, detection, causal impact, aggregation
└── README.md
```
🚀 How to Run
This notebook was originally built on a Spark-based data platform (e.g., Databricks). To adapt it to your own environment:
Install dependencies:
```bash
   pip install pyspark pandas numpy statsmodels scikit-learn scipy causalimpact
   ```
Replace the data loading cell (`spark.table(...)`) with your own data source — the pipeline expects a table with columns such as `store_id`, `barcode`, `period_date`, `total_qty`, `total_qty_promo`, category/brand descriptors, and store/product dimension attributes.
Run the notebook top to bottom. The core reusable functions (`build_core_matrices`, `detect_innovation_launches`, `candidate_victims_of_innovation`, `run_causal_effect`, `run_innovation_cannibalization_pipeline`) can be imported and applied to any retail sales dataset with a similar schema.
📈 Key Insights (Methodology-level)
Combining STL decomposition with an availability filter significantly reduces false positives from stockouts being mistaken for demand drops.
Using CausalImpact with category-level controls (rather than a simple before/after comparison) isolates the true effect of a launch from broader category trends and seasonality.
Aggregating SKU-level cannibalization to the brand level turns a large, noisy set of individual product pairs into actionable insights for portfolio and innovation strategy decisions.
📄 License
This repository shares an anonymized methodology developed during a professional engagement. No proprietary data, retailer names, or confidential business results are included.
