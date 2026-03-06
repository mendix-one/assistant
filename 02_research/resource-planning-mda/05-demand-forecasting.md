# Demand Forecasting with Multi-Dimensional Analysis

## 1. Overview

Demand forecasting in semiconductor device solutions is uniquely challenging due to:

- **Long lead times** (6-18 months from wafer start to finished product)
- **Technology cycle volatility** (new nodes, new architectures)
- **Customer concentration** (few large customers drive majority of demand)
- **Product life cycle compression** (especially in mobile/consumer)
- **Market segment diversity** (mobile, server, automotive, IoT each have different patterns)

Multi-dimensional analysis provides the framework to **decompose, analyze, and reconcile** forecasts across products, customers, time horizons, and confidence levels.

## 2. Forecasting Dimensions

### 2.1 Dimension Hierarchy for Demand Forecasting

```
DimProduct                      DimCustomer                  DimTime
  |-- BU (Memory/LSI/Foundry)    |-- Tier (Strategic/Std)     |-- Year
  |-- Family (DRAM/NAND/SoC)     |-- Region (APAC/NA/EU)      |-- Quarter
  |-- SKU (DDR5-16Gb)            |-- Industry (Mobile/DC/     |-- Month
  |-- Revision                   |     Auto/IoT/Consumer)     |-- Week
                                 |-- Account

DimMarketSegment                DimForecastType              DimConfidence
  |-- Segment (Mobile/Server/    |-- Method (Statistical/     |-- Level (High/
  |    Auto/IoT/Consumer)        |    ML/Expert/Consensus)    |    Medium/Low)
  |-- Application                |-- Horizon (Short/Med/Long) |-- Range (P10/P50/P90)
  |-- End-use                    |-- Source (Customer PO/      |-- Basis (Order/
                                 |    Sales Forecast/Market)  |    Commit/Forecast)
```

### 2.2 Forecast Granularity by Horizon

| Horizon | Time Grain | Product Grain | Typical Method | Confidence |
|---------|-----------|---------------|----------------|------------|
| Short-term (0-3 mo) | Weekly | SKU level | Customer POs + statistical | High (80-95%) |
| Medium-term (3-12 mo) | Monthly | Product family | ML + expert consensus | Medium (50-80%) |
| Long-term (1-3 yr) | Quarterly | BU/Market segment | Market models + strategic | Low (30-60%) |

## 3. Fact Table for Demand Forecasting

### FactDemandForecast (Grain: product-SKU + customer-account + month + forecast-version)

| Column | Type | Description |
|--------|------|-------------|
| product_id | FK | Product SKU |
| customer_id | FK | Customer account |
| market_segment_id | FK | End market |
| time_id | FK | Forecast period |
| forecast_type_id | FK | Forecast method |
| scenario_id | FK | Planning scenario |
| forecast_version_id | FK | Forecast iteration (v1, v2, ...) |
| forecast_qty | Measure | Point forecast (units) |
| forecast_low | Measure | P10 (pessimistic bound) |
| forecast_high | Measure | P90 (optimistic bound) |
| confidence_pct | Measure | Confidence level |
| actual_qty | Measure | Actual demand (filled in later) |
| forecast_error_pct | Measure | abs(actual - forecast) / actual |
| asp_usd | Measure | Average selling price |
| revenue_forecast | Measure | forecast_qty x asp_usd |

### FactForecastAccuracy (Grain: product-family + month + forecast-version)

| Column | Type | Description |
|--------|------|-------------|
| product_family_id | FK | Product family |
| time_id | FK | Period being forecasted |
| forecast_version_id | FK | Which forecast version |
| forecast_lead_months | Measure | How far in advance was forecast made |
| mape | Measure | Mean Absolute Percentage Error |
| bias | Measure | Systematic over/under-forecast |
| hit_rate_pct | Measure | % of forecasts within +/- 10% of actual |

## 4. Multi-Dimensional Forecast Analysis

### 4.1 Demand Decomposition (Drill-Down by Product Hierarchy)

```
Total DS Division Demand: 2.8B units (2026)
  |
  +-- Memory: 2.1B units (75%)
  |     +-- DRAM: 1.2B
  |     |     +-- DDR5: 800M  (ramping)
  |     |     +-- DDR4: 300M  (declining)
  |     |     +-- HBM:  100M  (high growth)
  |     |
  |     +-- NAND: 900M
  |           +-- 176L+: 600M
  |           +-- Legacy: 300M
  |
  +-- System LSI: 500M units (18%)
  |     +-- Exynos: 200M
  |     +-- Image Sensor: 250M
  |     +-- DDI: 50M
  |
  +-- Foundry: 200M units (7%)
        +-- 3nm: 50M
        +-- 5nm: 100M
        +-- 7nm+: 50M
```

### 4.2 Customer Demand Heatmap (Pivot: Customer x Product)

```
               DDR5    HBM4    NAND    Exynos   ISOCELL   Foundry
Customer A     120M     30M     80M      -        -        50M
Customer B      80M     25M     60M      -        -         -
Customer C       -       -       -      200M     100M       -
Customer D      50M     15M     40M      -        -        80M
Internal        40M      5M     30M      -       150M       -
Others         510M     25M    690M      -        -        70M
─────────────────────────────────────────────────────────────────
TOTAL          800M    100M    900M     200M     250M      200M
```

### 4.3 Forecast Accuracy Over Time (Slice by Forecast Lead)

How accurate are forecasts at different lead times?

```
Product Family    1-Month Lead   3-Month Lead   6-Month Lead   12-Month Lead
DRAM (DDR5)         MAPE: 5%      MAPE: 12%      MAPE: 22%      MAPE: 35%
DRAM (HBM)          MAPE: 8%      MAPE: 18%      MAPE: 30%      MAPE: 45%
NAND                MAPE: 6%      MAPE: 14%      MAPE: 20%      MAPE: 30%
Mobile SoC          MAPE: 10%     MAPE: 20%      MAPE: 35%      MAPE: 50%
Foundry             MAPE: 15%     MAPE: 25%      MAPE: 40%      MAPE: 55%

Insight: HBM and Foundry are hardest to forecast. Need wider confidence
         intervals and more frequent re-forecasting for these products.
```

### 4.4 Demand vs Capacity Alignment (Cross-Fact Analysis)

Join FactDemandForecast with FactCapacity to find gaps:

```
                  2026-Q2                    2026-Q3
Product      Demand   Capacity   Gap     Demand   Capacity   Gap
DDR5          200M     195M      -5M      220M     210M      -10M   !!
HBM4           28M      25M      -3M       35M      30M       -5M   !!
NAND          230M     250M     +20M      225M     250M      +25M
Exynos         50M      55M      +5M       55M      55M        0M
Foundry        48M      50M      +2M       55M      50M       -5M   !!

Action: DDR5 and HBM4 need capacity expansion or demand shaping.
        NAND has excess — consider foundry cross-allocation.
```

## 5. Forecasting Methods Integration

### 5.1 Multi-Model Approach

```
                Statistical         ML/Deep Learning        Expert/Market
                (Time Series)       (Multi-dimensional)     (Judgmental)
                    |                      |                      |
                    v                      v                      v
             +------------+        +-------------+        +-----------+
             | ARIMA/ETS  |        | LSTM/        |        | Sales     |
             | Seasonal   |        | Transformer  |        | consensus |
             | decompose  |        | with multi-  |        | Customer  |
             |            |        | dim features |        | signals   |
             +-----+------+        +------+------+        +-----+-----+
                   |                      |                      |
                   +----------+-----------+----------+-----------+
                              |                      |
                       +------v------+        +------v------+
                       | Ensemble /  |        | Forecast    |
                       | Weighted    |        | Reconcili-  |
                       | Average     |        | ation       |
                       +------+------+        | (top-down / |
                              |               | bottom-up)  |
                              |               +------+------+
                              +----------+-----------+
                                         |
                                  +------v------+
                                  | Final       |
                                  | Forecast    |
                                  | + Confidence|
                                  | Intervals   |
                                  +-------------+
                                         |
                                         v
                                  Load into FactDemandForecast
```

### 5.2 Dimensional Features for ML Models

| Feature Category | Dimensions Used | Examples |
|-----------------|----------------|----------|
| Temporal | Time | Month, quarter, seasonality index, trend |
| Product lifecycle | Product, Time | Months since launch, generation position |
| Customer behavior | Customer, Time | Order history, PO pipeline, inventory levels |
| Market signals | MarketSegment | Smartphone shipment forecasts, server build plans |
| Technology | Product, TechNode | Node maturity, competitor node timeline |
| Macroeconomic | External | GDP growth, exchange rates, industry indices |

## 6. Forecast Reconciliation

### Top-Down vs Bottom-Up

```
Top-Down:  Total Market -> Samsung Share -> BU -> Product Family -> SKU
           (Market research)

Bottom-Up: Customer POs -> SKU totals -> Product Family -> BU -> Total
           (Sales team input)

Middle-Out: Product Family level forecast, then:
            - Disaggregate down to SKU using historical proportions
            - Aggregate up to BU/Division for strategic planning

Reconciliation:  Compare all three in the OLAP cube.
                 If gap > threshold, trigger consensus meeting.
```

### Reconciliation in the Cube

```
Level          Top-Down    Bottom-Up    Middle-Out   Consensus    Gap
Division        2.80B       2.65B        2.72B          ?        5.7%
  Memory        2.10B       2.00B        2.05B          ?        5.0%
    DRAM        1.20B       1.15B        1.18B          ?        4.3%
      DDR5       800M        780M         790M          ?        2.6%
      HBM        100M         85M          92M          ?       17.6%  !!
```

> **HBM has 17.6% gap** between top-down and bottom-up. This is the highest variance item and needs executive attention.

---

## Sources

- [Manufacturing Intelligence for Semiconductor Demand Forecast](https://www.sciencedirect.com/science/article/abs/pii/S092552731000263X)
- [Multi-Model Fusion Demand Forecasting Framework](https://www.mdpi.com/2227-9717/12/11/2612)
- [Demand Forecasting and Financial Estimation - Semiconductor Supply Chain](https://www.sciencedirect.com/science/article/abs/pii/S036083521930573X)
- [Demand Forecast of Semiconductor](https://dl.acm.org/doi/pdf/10.5555/1516744.1517150)
- [Machine Learning Demand Forecasting and Supply Chain Performance](https://www.tandfonline.com/doi/full/10.1080/13675567.2020.1803246)
