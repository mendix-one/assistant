# Integration Architecture

## 1. Overview

This document describes how all AI components connect with the multi-dimensional analysis (MDA) layer, Elasticsearch, and enterprise systems to form a unified resource planning platform for Samsung DS Division.

## 2. End-to-End Architecture

```
+===========================================================================+
|                         PRESENTATION LAYER                                 |
|  +-------------+  +--------------+  +--------------+  +----------------+  |
|  | Kibana       |  | Planning     |  | Mobile       |  | LLM Chat      |  |
|  | Dashboards   |  | Portal (Web) |  | Alerts       |  | Interface     |  |
|  +------+------+  +------+-------+  +------+-------+  +-------+-------+  |
+=========|================|================|==================|============+
          |                |                |                   |
+=========|================|================|==================|============+
|                          API GATEWAY                                       |
|  +----------------------------------------------------------------------+ |
|  | REST API / GraphQL                                                    | |
|  | Authentication, Rate Limiting, Routing                                | |
|  +----------------------------------------------------------------------+ |
+===========================================================================+
          |                |                |                   |
+=========|================|================|==================|============+
|                       AI & ANALYTICS LAYER                                 |
|                                                                            |
|  +----------------+  +----------------+  +----------------+               |
|  | Scheduling     |  | Forecasting    |  | Scenario       |               |
|  | Service        |  | Service        |  | Service        |               |
|  |                |  |                |  |                |               |
|  | - RL Agent     |  | - TFT Model   |  | - Digital Twin |               |
|  | - GA Solver    |  | - XGBoost     |  | - What-if      |               |
|  | - Constraint   |  | - Ensemble    |  |   Engine       |               |
|  |   Validator    |  | - Reconciler  |  | - Simulator    |               |
|  +-------+--------+  +-------+--------+  +-------+--------+               |
|          |                    |                    |                        |
|  +-------+--------------------+--------------------+--------+              |
|  |                  AI Orchestrator                          |              |
|  |  - Model serving (MLflow / TorchServe)                   |              |
|  |  - Feature store (Feast)                                  |              |
|  |  - Experiment tracking                                    |              |
|  |  - A/B testing framework                                 |              |
|  +----------------------------------------------------------+              |
+===========================================================================+
          |                |                |                   |
+=========|================|================|==================|============+
|                       DATA LAYER                                           |
|                                                                            |
|  +-----------------+  +------------------+  +------------------+          |
|  | Elasticsearch   |  | Knowledge Graph  |  | Data Warehouse   |          |
|  | Cluster         |  | (Neo4j)          |  | (Snowflake       |          |
|  |                 |  |                  |  |  Schema)         |          |
|  | - resource-     |  | - Constraints    |  | - FactCapacity   |          |
|  |   allocation    |  | - Qualifications |  | - FactDemand     |          |
|  | - capacity-plan |  | - Process flows  |  | - FactAllocation |          |
|  | - demand-       |  | - Equipment      |  | - Dim tables     |          |
|  |   forecast      |  |   relationships  |  |   (normalized)   |          |
|  | - schedule-     |  | - SLA rules      |  |                  |          |
|  |   constraint    |  |                  |  |                  |          |
|  | - scenario-     |  |                  |  |                  |          |
|  |   results       |  |                  |  |                  |          |
|  +---------+-------+  +---------+--------+  +---------+--------+          |
|            |                    |                      |                   |
|  +---------+--------------------+----------------------+--------+         |
|  |                     ETL / Streaming                           |         |
|  |  Apache Kafka (real-time) + Airflow (batch)                  |         |
|  +--------------------------------------------------------------+         |
+===========================================================================+
          |                |                |                   |
+=========|================|================|==================|============+
|                    SOURCE SYSTEMS                                          |
|  +----------+  +----------+  +----------+  +----------+  +----------+    |
|  | SAP N-ERP|  | MES      |  | Equipment|  | CRM /    |  | Market   |    |
|  |          |  |          |  | Sensors  |  | Sales    |  | Data     |    |
|  |          |  |          |  | (IoT)    |  |          |  | Feeds    |    |
|  +----------+  +----------+  +----------+  +----------+  +----------+    |
+===========================================================================+
```

## 3. Data Flow Patterns

### 3.1 Operational Flow (Real-Time)

```
Equipment sensor event
  --> Kafka topic: "equipment-events"
  --> Stream processor (Flink/Kafka Streams):
      1. Update Elasticsearch (equipment status)
      2. Update Digital Twin state
      3. Trigger RL scheduler if state change is significant
  --> RL agent evaluates new state
  --> If action needed: push schedule update to MES
  --> Log decision to ES: "schedule-decisions" index
```

### 3.2 Planning Flow (Batch - Daily/Weekly)

```
1. Airflow DAG triggers nightly:
   a. Pull latest demand data from CRM/ERP
   b. Run demand forecast models (XGBoost, TFT)
   c. Store forecasts in ES: "demand-forecast" index
   d. Store forecasts in Data Warehouse: FactDemandForecast

2. Weekly planning cycle:
   a. Run GA solver for next-week schedule optimization
   b. Validate against Knowledge Graph constraints
   c. Run Digital Twin simulation for validation
   d. Store results in ES: "capacity-plan", "scenario-results"
   e. Generate Kibana dashboard refresh
   f. Alert planners if constraint violations detected
```

### 3.3 Strategic Flow (Monthly/Quarterly)

```
1. Monthly capacity review:
   a. Aggregate ES data into MDA cube rollups (ES transforms)
   b. Run long-term demand forecast (12-24 month)
   c. Generate what-if scenarios via Digital Twin
   d. Compare scenarios in Kibana dashboard
   e. Executive presentation with Pareto analysis

2. Quarterly CapEx planning:
   a. Run NSGA-II for capacity allocation optimization
   b. Simulate top 5 scenarios in Digital Twin
   c. Financial impact analysis per scenario
   d. Load to ES with scenario_ids for comparison
```

## 4. Component Interaction Matrix

| Producer | Consumer | Data | Transport | Frequency |
|----------|----------|------|-----------|-----------|
| MES | ES, Digital Twin | Lot events, equipment status | Kafka (real-time) | Continuous |
| ERP | Data Warehouse, ES | Orders, inventory, financials | Airflow (batch) | Daily |
| Forecast Service | ES, Data Warehouse | Demand forecasts | Airflow | Weekly |
| RL Scheduler | MES, ES | Schedule decisions | API (real-time) | On-demand |
| GA Solver | ES, Digital Twin | Optimized schedules | Airflow | Weekly |
| Digital Twin | ES | Simulation results | API (batch) | On-demand |
| Knowledge Graph | Scheduler, Validator | Constraints, rules | API (sync) | On-demand |
| Market Data | Forecast Service | External signals | Airflow | Daily |

## 5. Service Contracts

### 5.1 Scheduling Service API

```
POST /api/v1/schedule/optimize
{
  "facility": "Hwaseong",
  "time_horizon": "2026-W12",
  "method": "ga",          // "ga" | "rl" | "hybrid"
  "objectives": ["minimize_tardiness", "maximize_utilization"],
  "constraints": "from_knowledge_graph",
  "scenario_id": "baseline-2026"
}

Response:
{
  "schedule_id": "SCH-2026-W12-001",
  "status": "optimized",
  "fitness": { "tardiness": 12.5, "utilization": 0.91 },
  "violations": [],
  "assignments": [ ... ],
  "stored_in_es": true,
  "es_index": "capacity-plan"
}
```

### 5.2 Forecast Service API

```
POST /api/v1/forecast/generate
{
  "product_families": ["DDR5", "HBM4"],
  "horizon_months": 12,
  "model": "tft",           // "xgboost" | "lstm" | "tft" | "ensemble"
  "quantiles": [0.1, 0.5, 0.9],
  "scenario_id": "baseline-2026"
}

Response:
{
  "forecast_id": "FCT-2026-03-001",
  "forecasts": [
    {
      "product_family": "DDR5",
      "period": "2026-04",
      "p10": 185000000, "p50": 200000000, "p90": 218000000,
      "confidence": 0.82,
      "top_drivers": ["smartphone_forecast", "asp_trend"]
    },
    ...
  ],
  "stored_in_es": true
}
```

### 5.3 Scenario Service API

```
POST /api/v1/scenario/simulate
{
  "base_scenario": "baseline-2026",
  "modifications": {
    "equipment": { "add": [{ "type": "EUV", "facility": "Hwaseong", "available_from": "2026-Q3" }] },
    "demand": { "HBM4": "+40%" }
  },
  "simulation_runs": 1000,
  "horizon_months": 12
}

Response:
{
  "scenario_id": "sim-euv-add-hbm-growth",
  "status": "completed",
  "summary": {
    "throughput_change": "+28.5%",
    "utilization_change": "-15.2%",
    "revenue_impact": "+$380M",
    "new_bottleneck": "Etch Bay (91%)"
  },
  "confidence_interval": { "lower": "+22%", "upper": "+35%", "level": 0.95 },
  "stored_in_es": true,
  "kibana_dashboard_url": "/app/dashboards#/view/scenario-compare"
}
```

## 6. Deployment Architecture

```
Kubernetes Cluster
+----------------------------------------------------------+
| Namespace: ai-services                                    |
| +------------------+  +------------------+               |
| | scheduling-svc   |  | forecast-svc     |               |
| | replicas: 3      |  | replicas: 2      |               |
| | GPU: 1 per pod   |  | GPU: 1 per pod   |               |
| +------------------+  +------------------+               |
| +------------------+  +------------------+               |
| | scenario-svc     |  | llm-agent-svc    |               |
| | replicas: 2      |  | replicas: 2      |               |
| | CPU: 8 per pod   |  | API: Claude      |               |
| +------------------+  +------------------+               |
+----------------------------------------------------------+
| Namespace: data                                           |
| +------------------+  +------------------+               |
| | elasticsearch    |  | neo4j            |               |
| | 3 master, 6 data |  | 3 node cluster   |               |
| | 12 hot, 6 warm   |  |                  |               |
| +------------------+  +------------------+               |
| +------------------+  +------------------+               |
| | kafka            |  | redis            |               |
| | 5 brokers        |  | 3 node sentinel  |               |
| +------------------+  +------------------+               |
+----------------------------------------------------------+
| Namespace: ml-platform                                    |
| +------------------+  +------------------+               |
| | mlflow           |  | feast            |               |
| | model registry   |  | feature store    |               |
| +------------------+  +------------------+               |
| +------------------+                                      |
| | airflow          |                                      |
| | DAG scheduler    |                                      |
| +------------------+                                      |
+----------------------------------------------------------+
```

## 7. Technology Summary

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Presentation** | Kibana, React, Claude Chat | Dashboards, planning UI, NL interface |
| **API** | FastAPI (Python) | Service APIs, gateway |
| **Scheduling** | Stable-Baselines3 (RL), pymoo (GA), OR-Tools | Optimization engines |
| **Forecasting** | PyTorch (TFT), XGBoost, NeuralForecast | Demand prediction |
| **Simulation** | SimPy / AnyLogic | Digital twin DES |
| **Knowledge** | Neo4j + OWL | Constraint graph, reasoning |
| **Search/Analytics** | Elasticsearch + Kibana | MDA, real-time analytics |
| **Streaming** | Apache Kafka | Event streaming |
| **Batch** | Apache Airflow | ETL orchestration |
| **ML Platform** | MLflow, Feast | Model management, features |
| **Infrastructure** | Kubernetes, Helm | Container orchestration |
| **LLM** | Claude API (Anthropic) | NL planning interface, KG queries |

## 8. MDA + AI Synergy Map

```
+-------------------------------------------------------------------+
|                    Planning Decision Lifecycle                      |
|                                                                    |
|  UNDERSTAND          PREDICT            OPTIMIZE        ACT        |
|  (MDA)               (AI/ML)            (AI/Solver)    (System)   |
|                                                                    |
|  OLAP drill-down     Demand forecast    RL scheduler   MES update |
|  Utilization heatmap Yield prediction   GA allocation  ERP sync   |
|  Scenario compare    Bottleneck alert   What-if sim    Alert      |
|  Trend analysis      Risk scoring       Pareto front   Report     |
|                                                                    |
|  ES aggregations     TFT, XGBoost      PPO, NSGA-II   Kafka      |
|  Kibana dashboards   PyTorch            Stable-BL3     Airflow    |
|                                                                    |
|  "What happened?"    "What will         "What should   "Do it."   |
|  "Why?"              happen?"           we do?"                    |
+-------------------------------------------------------------------+
```

---

## Sources

- [Samsung Electronics Expands N-ERP System to 120 Offices](https://news.samsung.com/global/samsung-electronics-expands-the-next-generation-erp-system-to-120-offices-worldwide)
- [AI in Production Planning and Scheduling 2025](https://www.rapidinnovation.io/post/an-overview-of-production-planning-and-scheduling-with-ai)
- [LLM Agents Enterprise Architecture Guide 2025](https://aisera.com/blog/llm-agents/)
- [LLMs in Automated Planning and Scheduling](https://arxiv.org/abs/2401.02500)
- [LLMs for Manufacturing Scheduling](https://www.nature.com/articles/s44334-025-00061-w)
