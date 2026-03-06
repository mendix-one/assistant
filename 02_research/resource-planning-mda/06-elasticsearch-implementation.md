# Elasticsearch Implementation for Resource Planning MDA

## 1. Overview

This document provides practical Elasticsearch index designs, mappings, and aggregation queries to implement the multi-dimensional resource planning model described in the previous documents.

### Why Elasticsearch?

| Requirement | Elasticsearch Capability |
|-------------|-------------------------|
| Real-time capacity monitoring | Near real-time indexing (<1s) |
| Multi-dimensional drill-down | Nested bucket aggregations (unlimited depth) |
| What-if scenario comparison | Filter by scenario_id + side-by-side aggs |
| Forecast vs actual tracking | Cross-index queries, transforms |
| Dashboard visualization | Kibana native integration |
| Scale to millions of records | Horizontally scalable, sharded architecture |

### Architecture

```
Data Sources                    Elasticsearch              Visualization
+-----------------+            +-------------------+       +-----------+
| ERP (SAP N-ERP) |--+        | resource-          |       | Kibana    |
| MES Systems     |  |  ETL   |   allocation-*     |------>| Dashboards|
| Equipment Logs  |--+------->| capacity-plan-*    |       |           |
| Sales Forecasts |  |        | demand-forecast-*  |       | Custom    |
| HR Systems      |--+        | schedule-          |------>| React App |
+-----------------+            |   constraint-*     |       | (ES DSL)  |
                               +-------------------+       +-----------+
```

## 2. Index Design

### Design Principle: Denormalized Documents (Star-Schema Style)

Elasticsearch does not support joins across indices. Denormalize snowflake dimensions into flat documents at index time. The snowflake schema from the data model is the **source-of-truth in the data warehouse**; Elasticsearch gets a **flattened, query-optimized** copy.

```
Snowflake (Warehouse)              Denormalized (Elasticsearch)
+----------+                       {
| FactCap  |                         "facility": "Pyeongtaek",
| fac_id=5 |--+                      "site": "Pyeongtaek",
+----------+  |                      "country": "Korea",
              |  +----------+        "fab_line": "Line 17",
              +--| DimFac   |        "cleanroom_class": "ISO 3",
                 | site=PYT |        "product_family": "DRAM",
                 | country  |        "product_sku": "DDR5-16Gb",
                 +----------+        "tech_node": "3nm",
                                     "time": "2026-04-14",
                                     "quarter": "Q2-2026",
                                     ...measures...
                                   }
```

### 2.1 Index: resource-allocation

```json
PUT resource-allocation
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "allocation_id":     { "type": "keyword" },
      "timestamp":         { "type": "date" },

      "resource": {
        "properties": {
          "id":            { "type": "keyword" },
          "name":          { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
          "type":          { "type": "keyword" },
          "skill_category": { "type": "keyword" },
          "team":          { "type": "keyword" },
          "department":    { "type": "keyword" },
          "business_unit": { "type": "keyword" }
        }
      },

      "product": {
        "properties": {
          "id":            { "type": "keyword" },
          "name":          { "type": "keyword" },
          "family":        { "type": "keyword" },
          "sku":           { "type": "keyword" },
          "tech_node":     { "type": "keyword" },
          "package_type":  { "type": "keyword" }
        }
      },

      "facility": {
        "properties": {
          "site":          { "type": "keyword" },
          "country":       { "type": "keyword" },
          "fab_line":      { "type": "keyword" },
          "cleanroom":     { "type": "keyword" }
        }
      },

      "time": {
        "properties": {
          "year":          { "type": "short" },
          "quarter":       { "type": "keyword" },
          "month":         { "type": "keyword" },
          "week":          { "type": "keyword" },
          "shift":         { "type": "keyword" }
        }
      },

      "scenario_id":       { "type": "keyword" },
      "scenario_name":     { "type": "keyword" },
      "scenario_type":     { "type": "keyword" },

      "allocated_hours":   { "type": "float" },
      "available_hours":   { "type": "float" },
      "utilization_pct":   { "type": "float" },
      "cost_usd":          { "type": "float" },
      "priority":          { "type": "byte" }
    }
  }
}
```

### 2.2 Index: capacity-plan

```json
PUT capacity-plan
{
  "mappings": {
    "properties": {
      "capacity_id":             { "type": "keyword" },
      "timestamp":               { "type": "date" },

      "facility": {
        "properties": {
          "site":                { "type": "keyword" },
          "country":             { "type": "keyword" },
          "fab_line":            { "type": "keyword" },
          "cleanroom_class":     { "type": "keyword" }
        }
      },

      "equipment": {
        "properties": {
          "class":               { "type": "keyword" },
          "type":                { "type": "keyword" },
          "tool_id":             { "type": "keyword" }
        }
      },

      "product": {
        "properties": {
          "family":              { "type": "keyword" },
          "tech_node":           { "type": "keyword" }
        }
      },

      "time": {
        "properties": {
          "year":                { "type": "short" },
          "quarter":             { "type": "keyword" },
          "month":               { "type": "keyword" },
          "week":                { "type": "keyword" }
        }
      },

      "scenario_id":             { "type": "keyword" },
      "scenario_name":           { "type": "keyword" },

      "max_wafer_starts":        { "type": "integer" },
      "effective_capacity":      { "type": "integer" },
      "planned_load":            { "type": "integer" },
      "actual_load":             { "type": "integer" },
      "utilization_pct":         { "type": "float" },
      "downtime_hours":          { "type": "float" },
      "yield_pct":               { "type": "float" },
      "bottleneck_flag":         { "type": "boolean" },
      "capex_required_usd":      { "type": "float" }
    }
  }
}
```

### 2.3 Index: demand-forecast

```json
PUT demand-forecast
{
  "mappings": {
    "properties": {
      "forecast_id":         { "type": "keyword" },
      "timestamp":           { "type": "date" },

      "product": {
        "properties": {
          "id":              { "type": "keyword" },
          "family":          { "type": "keyword" },
          "sku":             { "type": "keyword" },
          "tech_node":       { "type": "keyword" },
          "business_unit":   { "type": "keyword" }
        }
      },

      "customer": {
        "properties": {
          "id":              { "type": "keyword" },
          "name":            { "type": "keyword" },
          "tier":            { "type": "keyword" },
          "region":          { "type": "keyword" },
          "industry":        { "type": "keyword" }
        }
      },

      "market_segment":     { "type": "keyword" },

      "time": {
        "properties": {
          "year":            { "type": "short" },
          "quarter":         { "type": "keyword" },
          "month":           { "type": "keyword" }
        }
      },

      "scenario_id":         { "type": "keyword" },
      "forecast_version":    { "type": "keyword" },
      "forecast_method":     { "type": "keyword" },

      "forecast_qty":        { "type": "long" },
      "forecast_low":        { "type": "long" },
      "forecast_high":       { "type": "long" },
      "confidence_pct":      { "type": "float" },
      "actual_qty":          { "type": "long" },
      "forecast_error_pct":  { "type": "float" },
      "asp_usd":             { "type": "float" },
      "revenue_forecast":    { "type": "float" }
    }
  }
}
```

### 2.4 Index: schedule-constraint

```json
PUT schedule-constraint
{
  "mappings": {
    "properties": {
      "constraint_id":       { "type": "keyword" },
      "timestamp":           { "type": "date" },

      "resource": {
        "properties": {
          "id":              { "type": "keyword" },
          "type":            { "type": "keyword" },
          "team":            { "type": "keyword" }
        }
      },

      "task": {
        "properties": {
          "id":              { "type": "keyword" },
          "name":            { "type": "keyword" },
          "phase":           { "type": "keyword" }
        }
      },

      "product": {
        "properties": {
          "family":          { "type": "keyword" },
          "tech_node":       { "type": "keyword" }
        }
      },

      "facility": {
        "properties": {
          "site":            { "type": "keyword" },
          "fab_line":        { "type": "keyword" }
        }
      },

      "time": {
        "properties": {
          "quarter":         { "type": "keyword" },
          "month":           { "type": "keyword" },
          "week":            { "type": "keyword" }
        }
      },

      "constraint_type":     { "type": "keyword" },
      "constraint_category": { "type": "keyword" },

      "is_satisfied":        { "type": "boolean" },
      "violation_severity":  { "type": "byte" },
      "slack_hours":         { "type": "float" },
      "reschedule_cost":     { "type": "float" }
    }
  }
}
```

## 3. Aggregation Queries (OLAP Operations)

### 3.1 Capacity Utilization by Site and Quarter (Roll-Up + Pivot)

```json
GET capacity-plan/_search
{
  "size": 0,
  "query": {
    "term": { "scenario_id": "baseline-2026" }
  },
  "aggs": {
    "by_site": {
      "terms": { "field": "facility.site", "size": 10 },
      "aggs": {
        "by_quarter": {
          "terms": { "field": "time.quarter", "order": { "_key": "asc" } },
          "aggs": {
            "avg_utilization": { "avg": { "field": "utilization_pct" } },
            "total_planned": { "sum": { "field": "planned_load" } },
            "total_capacity": { "sum": { "field": "effective_capacity" } }
          }
        }
      }
    }
  }
}
```

### 3.2 Bottleneck Detection - Drill Down (Facility > Line > Equipment)

```json
GET capacity-plan/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "facility.site": "Hwaseong" } },
        { "term": { "time.quarter": "Q2-2026" } },
        { "range": { "utilization_pct": { "gte": 90 } } }
      ]
    }
  },
  "aggs": {
    "by_fab_line": {
      "terms": { "field": "facility.fab_line" },
      "aggs": {
        "by_equipment_class": {
          "terms": { "field": "equipment.class" },
          "aggs": {
            "avg_util": { "avg": { "field": "utilization_pct" } },
            "max_util": { "max": { "field": "utilization_pct" } },
            "bottleneck_count": {
              "filter": { "term": { "bottleneck_flag": true } }
            }
          }
        }
      }
    }
  }
}
```

### 3.3 Scenario Comparison - Capacity (Dice)

```json
GET capacity-plan/_search
{
  "size": 0,
  "query": {
    "terms": { "scenario_id": ["baseline-2026", "optimistic-2026", "conservative-2026"] }
  },
  "aggs": {
    "by_scenario": {
      "terms": { "field": "scenario_name" },
      "aggs": {
        "by_site": {
          "terms": { "field": "facility.site" },
          "aggs": {
            "total_capacity": { "sum": { "field": "effective_capacity" } },
            "total_planned": { "sum": { "field": "planned_load" } },
            "avg_util": { "avg": { "field": "utilization_pct" } }
          }
        },
        "overall_capacity": { "sum": { "field": "effective_capacity" } },
        "overall_util": { "avg": { "field": "utilization_pct" } }
      }
    }
  }
}
```

### 3.4 Demand vs Capacity Gap by Product (Cross-Index)

Use Elasticsearch transforms or application-level join:

```json
GET demand-forecast/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "scenario_id": "baseline-2026" } },
        { "term": { "time.quarter": "Q2-2026" } }
      ]
    }
  },
  "aggs": {
    "by_product_family": {
      "terms": { "field": "product.family", "size": 20 },
      "aggs": {
        "total_demand": { "sum": { "field": "forecast_qty" } },
        "demand_low": { "sum": { "field": "forecast_low" } },
        "demand_high": { "sum": { "field": "forecast_high" } },
        "avg_confidence": { "avg": { "field": "confidence_pct" } },
        "total_revenue": { "sum": { "field": "revenue_forecast" } }
      }
    }
  }
}
```

### 3.5 Constraint Violation Analysis (Slice + Pivot)

```json
GET schedule-constraint/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "is_satisfied": false } },
        { "term": { "time.quarter": "Q2-2026" } }
      ]
    }
  },
  "aggs": {
    "by_constraint_type": {
      "terms": { "field": "constraint_type" },
      "aggs": {
        "by_severity": {
          "terms": { "field": "violation_severity" },
          "aggs": {
            "count": { "value_count": { "field": "constraint_id" } },
            "total_cost": { "sum": { "field": "reschedule_cost" } }
          }
        },
        "by_facility": {
          "terms": { "field": "facility.site" },
          "aggs": {
            "violation_count": { "value_count": { "field": "constraint_id" } }
          }
        }
      }
    }
  }
}
```

### 3.6 Forecast Accuracy Trend (Pipeline Aggregation)

```json
GET demand-forecast/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "exists": { "field": "actual_qty" } },
        { "term": { "product.business_unit": "Memory" } }
      ]
    }
  },
  "aggs": {
    "by_month": {
      "terms": { "field": "time.month", "order": { "_key": "asc" }, "size": 24 },
      "aggs": {
        "avg_mape": { "avg": { "field": "forecast_error_pct" } },
        "total_forecast": { "sum": { "field": "forecast_qty" } },
        "total_actual": { "sum": { "field": "actual_qty" } }
      }
    },
    "mape_trend": {
      "avg_bucket": {
        "buckets_path": "by_month>avg_mape"
      }
    }
  }
}
```

### 3.7 Resource Over-Allocation Alert (Filter + Threshold)

```json
GET resource-allocation/_search
{
  "size": 10,
  "query": {
    "bool": {
      "filter": [
        { "range": { "utilization_pct": { "gt": 100 } } },
        { "term": { "time.month": "2026-04" } },
        { "term": { "scenario_id": "baseline-2026" } }
      ]
    }
  },
  "sort": [
    { "utilization_pct": "desc" }
  ],
  "_source": ["resource.name", "resource.team", "product.family",
              "facility.site", "utilization_pct", "allocated_hours", "available_hours"]
}
```

## 4. Kibana Dashboard Design

### 4.1 Capacity Planning Dashboard

| Panel | Visualization | Aggregation |
|-------|--------------|-------------|
| Utilization Heatmap | Heatmap (Site x Quarter) | terms(site) > terms(quarter) > avg(util) |
| Capacity vs Demand | Dual-axis bar chart | terms(product.family) > sum(capacity), sum(demand) |
| Bottleneck Tracker | Data table with conditional formatting | filter(util>90) > terms(fab_line) > terms(equipment) |
| Trend Line | Line chart | date_histogram(month) > avg(utilization_pct) |
| Scenario Toggle | Controls panel | Filter on scenario_id |

### 4.2 Scheduling Dashboard

| Panel | Visualization | Aggregation |
|-------|--------------|-------------|
| Violation Summary | Metric tiles | filter(is_satisfied=false) > value_count |
| Violations by Type | Donut chart | terms(constraint_type) > count |
| Severity Timeline | Stacked bar | terms(week) > terms(severity) > count |
| Resource Conflicts | Data table | filter(overbooking) > terms(resource.name) |

### 4.3 Demand Forecasting Dashboard

| Panel | Visualization | Aggregation |
|-------|--------------|-------------|
| Demand by Product | Treemap | terms(BU) > terms(family) > sum(forecast_qty) |
| Forecast vs Actual | Line chart (dual) | date_histogram(month) > sum(forecast), sum(actual) |
| Accuracy by Product | Bar chart | terms(family) > avg(forecast_error_pct) |
| Customer Demand | Horizontal bar | terms(customer) > sum(forecast_qty) |
| Confidence Bands | Area chart | date_histogram > percentiles(forecast) |

## 5. Performance Optimization

### 5.1 Index Lifecycle Management (ILM)

```json
PUT _ilm/policy/resource-planning-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_size": "30gb", "max_age": "30d" }
        }
      },
      "warm": {
        "min_age": "90d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "365d",
        "actions": {
          "freeze": {}
        }
      }
    }
  }
}
```

### 5.2 Transforms for Pre-Aggregated Rollups

```json
PUT _transform/capacity-monthly-rollup
{
  "source": { "index": "capacity-plan-*" },
  "dest": { "index": "capacity-plan-monthly-rollup" },
  "pivot": {
    "group_by": {
      "facility_site": { "terms": { "field": "facility.site" } },
      "product_family": { "terms": { "field": "product.family" } },
      "tech_node": { "terms": { "field": "product.tech_node" } },
      "month": { "terms": { "field": "time.month" } },
      "scenario": { "terms": { "field": "scenario_id" } }
    },
    "aggregations": {
      "avg_utilization": { "avg": { "field": "utilization_pct" } },
      "total_planned": { "sum": { "field": "planned_load" } },
      "total_capacity": { "sum": { "field": "effective_capacity" } },
      "avg_yield": { "avg": { "field": "yield_pct" } },
      "bottleneck_count": { "sum": { "field": "bottleneck_flag" } }
    }
  },
  "frequency": "1h",
  "sync": { "time": { "field": "timestamp", "delay": "60s" } }
}
```

### 5.3 Tips

| Tip | Details |
|-----|---------|
| Use `keyword` for all dimension fields | Enables efficient terms aggregations |
| Avoid `text` type for dimension fields | Use `text` + `.keyword` sub-field only when full-text search is needed |
| Separate hot/warm/cold tiers | Current quarter = hot, historical = warm/cold |
| Pre-aggregate for executive dashboards | Use transforms for monthly/quarterly rollups |
| Shard sizing | Target 10-30GB per shard for aggregation-heavy workloads |
| Use `composite` aggregation | For paginating through large multi-dimensional result sets |
| Cache query results | Enable `request_cache` for repeated scenario comparisons |

---

## Sources

- [Elasticsearch Aggregations - Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)
- [Elasticsearch Aggregations - Opster](https://opster.com/guides/elasticsearch/glossary/elasticsearch-aggregations/)
- [Pipeline Aggregations - Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-pipeline.html)
- [Capacity Planning for Elasticsearch - Medium](https://medium.com/codex/capacity-planning-for-elasticsearch-cde3c0693add)
- [Elasticsearch Capacity Planning - Opster](https://opster.com/guides/elasticsearch/capacity-planning/elasticsearch-capacity-planning/)
