# ML for Demand Forecasting

## 1. Why ML/DL for Semiconductor Demand?

Traditional statistical methods (ARIMA, Exponential Smoothing) fail when:
- **Multiple external drivers** influence demand simultaneously (AI boom, geopolitics, consumer cycles)
- **Non-linear relationships** exist between features (technology transitions, market disruptions)
- **Hierarchical reconciliation** is needed (SKU-level forecasts must align with BU-level strategy)
- **Cross-dimensional patterns** matter (product x customer x region x time interactions)

ML models capture these complexities by learning from **multi-dimensional feature spaces** — directly complementing the MDA framework.

## 2. Model Landscape

### 2.1 Architecture Comparison

| Model | Type | Strengths | Weaknesses | Best For |
|-------|------|-----------|------------|----------|
| **ARIMA/ETS** | Statistical | Interpretable, fast, well-understood | Univariate, linear, manual tuning | Stable products, short-term |
| **XGBoost/LightGBM** | Tree ensemble | Handles tabular features well, fast training | No native temporal structure | Feature-rich demand with many drivers |
| **LSTM** | Recurrent NN | Captures temporal dependencies | Slow training, vanishing gradients | Medium-length sequences |
| **Temporal Fusion Transformer (TFT)** | Attention-based | Multi-horizon, interpretable attention, handles static + temporal features | Complex, compute-intensive | Multi-step forecasting with explanations |
| **N-BEATS** | Deep NN | Pure time-series, no feature engineering | Univariate focus | High-accuracy point forecasts |
| **TimesFM / Chronos** | Foundation model | Zero/few-shot, general time-series | New, less validated for domain-specific | Rapid prototyping, cold-start products |
| **DeepAR** | Autoregressive RNN | Probabilistic forecasts, handles many related series | Needs large dataset | SKU-level across many products |

### 2.2 Recommended Stack for DS Division

```
Tier 1: Production Models (proven, deploy now)
+---------------------------------------------+
| XGBoost ensemble for short-term (0-3 month) |
| LSTM for medium-term (3-12 month)           |
+---------------------------------------------+

Tier 2: Advanced Models (pilot, deploy in 6 months)
+---------------------------------------------+
| Temporal Fusion Transformer for multi-       |
|   horizon with interpretability              |
| DeepAR for probabilistic SKU-level          |
+---------------------------------------------+

Tier 3: Experimental (research, evaluate)
+---------------------------------------------+
| Foundation models (TimesFM, Chronos)         |
| LLM-augmented forecasting                   |
+---------------------------------------------+
```

## 3. Feature Engineering from MDA Dimensions

The snowflake schema dimensions map directly to ML features:

### 3.1 Temporal Features (from DimTime)

```python
temporal_features = {
    "month":              int,    # 1-12
    "quarter":            int,    # 1-4
    "day_of_week":        int,    # 0-6
    "week_of_year":       int,    # 1-52
    "is_quarter_end":     bool,   # Q-end demand spikes
    "days_to_product_launch": int, # Ramp-up signal
    "seasonality_index":  float,  # Learned from history
    "trend_component":    float,  # Decomposed trend
    "holiday_flag":       bool,   # Regional holidays affect orders
}
```

### 3.2 Product Features (from DimProduct, DimTechNode)

```python
product_features = {
    "product_family":     categorical,  # DRAM, NAND, SoC, ...
    "tech_node":          categorical,  # 3nm, 5nm, 7nm, ...
    "months_since_launch": int,         # Product lifecycle position
    "generation_position": int,         # nth gen in product line
    "node_maturity":      float,        # Yield trend indicator
    "competitor_node_gap": int,         # Months ahead/behind competitor
    "asp_trend":          float,        # Price erosion rate
    "design_win_count":   int,          # Number of customer design-ins
}
```

### 3.3 Customer Features (from DimCustomer)

```python
customer_features = {
    "customer_tier":       categorical,  # Strategic, Standard, Internal
    "customer_region":     categorical,  # APAC, NA, EU
    "customer_industry":   categorical,  # Mobile, DC, Auto, IoT
    "historical_order_cv": float,        # Coefficient of variation (volatility)
    "order_pipeline_value": float,       # PO + committed backlog
    "customer_inventory_weeks": float,   # Their inventory level (if shared)
    "customer_market_share": float,      # Their position in end market
}
```

### 3.4 Market / External Features

```python
external_features = {
    "smartphone_shipment_forecast": float,  # IDC/Gartner data
    "server_build_forecast":        float,  # Hyperscaler signals
    "auto_production_forecast":     float,  # OICA data
    "dram_spot_price":              float,  # DRAMeXchange
    "nand_spot_price":              float,
    "gdp_growth_rate":              float,  # By region
    "pmi_index":                    float,  # Manufacturing index
    "competitor_capacity_news":     categorical, # Expansion/contraction signals
}
```

## 4. Temporal Fusion Transformer (TFT) — Recommended Architecture

TFT is the recommended primary model because it:
- Handles **static** (product, customer) + **temporal** (time-varying) features natively
- Produces **multi-horizon** forecasts (1-month to 12-month) in one pass
- Provides **interpretable attention weights** showing which features and time steps matter
- Generates **quantile forecasts** (P10, P50, P90) for confidence intervals

### 4.1 Architecture Overview

```
Inputs:                           TFT Architecture:
                                 +---------------------------+
Static features -----+           | Variable Selection        |
(product, customer)  |           | Network (per feature)     |
                     +---------->| (learns feature importance)|
                     |           +-------------+-------------+
Known future --------+                         |
(calendar, promotions)|          +-------------v-------------+
                     +---------->| LSTM Encoder-Decoder      |
                     |           | (temporal processing)     |
Observed past -------+           +-------------+-------------+
(historical demand,  |                         |
 prices, orders)     +---------->| Multi-Head Attention      |
                                 | (interpretable, temporal) |
                                 +-------------+-------------+
                                               |
                                 +-------------v-------------+
                                 | Quantile Output           |
                                 | P10, P50, P90 per horizon |
                                 +---------------------------+
```

### 4.2 Interpretability Output

```
Product: DDR5-16Gb | Customer: Customer A | Forecast: Q3-2026

Feature Importance (attention weights):
  smartphone_shipment_forecast:  0.28  ****
  historical_demand_lag_3m:      0.22  ****
  asp_trend:                     0.15  ***
  customer_inventory_weeks:      0.12  **
  server_build_forecast:         0.10  **
  seasonality_index:             0.08  *
  other:                         0.05  *

Temporal Attention (which past periods matter most):
  T-1 month:  0.35  (most recent trend)
  T-3 months: 0.25  (quarterly pattern)
  T-12 months: 0.20 (year-over-year)
  T-6 months: 0.15  (semi-annual cycle)
  other:      0.05

Insight: DDR5 demand for Customer A in Q3 is primarily driven by
         smartphone shipment outlook and recent order momentum.
```

## 5. Hierarchical Forecast Reconciliation

### 5.1 The Hierarchy Problem

Forecasts at different levels of the product/customer hierarchy don't naturally agree:

```
                  Level               Forecast Method
Division total    2.80B (top-down)    Market models
  |
  BU total        2.10B (middle-out)  Category analysis
  |
  Family total    1.20B (bottom-up)   ML model aggregation
  |
  SKU total       1.15B (bottom-up)   ML model direct

  Gap: 2.80B vs 1.15B aggregated from bottom = 22% discrepancy
```

### 5.2 Reconciliation Approaches

| Approach | Description | When to Use |
|----------|-------------|-------------|
| **Top-Down** | Forecast at top, disaggregate by historical proportions | Long-term strategic, new products with no history |
| **Bottom-Up** | Forecast each SKU, aggregate up | Short-term, mature products with good SKU-level data |
| **Middle-Out** | Forecast at product family, disaggregate down & aggregate up | Balanced, most common in practice |
| **Optimal Reconciliation (MinT)** | Statistical method that adjusts all levels simultaneously to minimize total variance | Best accuracy, requires covariance estimation |
| **ML Reconciliation** | Train a model to learn optimal adjustments across levels | When historical reconciliation patterns are complex |

### 5.3 Reconciliation in MDA

```
OLAP Cube Operation:

1. Load all forecast versions into FactDemandForecast
   - forecast_method = "bottom_up", "top_down", "middle_out", "reconciled"

2. Dice on forecast_method to compare all versions side by side

3. Roll-up bottom-up forecasts through product hierarchy
   SKU -> Family -> BU -> Division

4. Identify gaps at each level
   gap = abs(top_down - bottom_up_aggregated) / top_down

5. Apply reconciliation algorithm (MinT or ML)

6. Load reconciled forecast as new forecast_version

7. Dashboard: Show reconciled vs original at each hierarchy level
```

## 6. Model Training & Evaluation Pipeline

```
+------------------+     +------------------+     +------------------+
| Data Preparation |     | Model Training   |     | Evaluation       |
|                  |     |                  |     |                  |
| - Pull from ES   |     | - Train/val/test |     | - MAPE by        |
|   indices        |     |   split (time-   |     |   product family |
| - Feature eng    |---->|   based, not     |---->| - Bias detection |
|   from MDA dims  |     |   random!)       |     | - Coverage of    |
| - Handle missing |     | - Hyperparameter |     |   prediction     |
|   values         |     |   tuning (Optuna)|     |   intervals      |
| - Scale/encode   |     | - Cross-validate |     | - Forecast value |
+------------------+     |   on rolling     |     |   added (FVA)    |
                          |   windows        |     |   vs naive       |
                          +------------------+     +------------------+
                                                          |
                                                   +------v------+
                                                   | Production  |
                                                   | Deployment  |
                                                   | - Batch     |
                                                   |   weekly    |
                                                   | - Store in  |
                                                   |   ES index  |
                                                   | - Monitor   |
                                                   |   drift     |
                                                   +-------------+
```

### Key Evaluation Metrics

| Metric | Formula | Target (DS Division) |
|--------|---------|---------------------|
| **MAPE** | mean(abs(actual - forecast) / actual) | <15% at family level, <25% at SKU |
| **Bias** | mean((forecast - actual) / actual) | -5% to +5% (near zero) |
| **FVA** | 1 - (MAPE_model / MAPE_naive) | >20% improvement over naive |
| **Coverage** | % of actuals within P10-P90 interval | 80% target |
| **Hit Rate** | % of forecasts within +/-10% of actual | >60% at family level |

---

## Sources

- [AI Transforming Semiconductor Demand Forecasting](https://cbcinc.ai/ai-transforming-semiconductor-demand-forecasting-and-supply-chain/)
- [ML and DL Models for Demand Forecasting - Critical Review](https://www.mdpi.com/2571-5577/7/5/93)
- [Demand Forecasting in the Age of AI & ML](https://research.aimultiple.com/demand-forecasting/)
- [AI Demand Forecasting Trends 2025](https://indatalabs.com/blog/ai-demand-forecasting)
- [DRL for Selecting Demand Forecast Models (Industry 3.5)](https://www.tandfonline.com/doi/full/10.1080/00207543.2020.1733125)
- [Manufacturing Intelligence for Semiconductor Demand Forecast](https://www.sciencedirect.com/science/article/abs/pii/S092552731000263X)
