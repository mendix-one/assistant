# Data Modeling - IRIS Resources Planning

## 1. Overview

IRIS uses a **dual-storage** architecture:
- **Mendix Oracle 19c** — Transactional domain model (planning, versioning, workflows, master data)
- **Elasticsearch** — Analytical model (multi-dimensional aggregations, reporting, analysis views)

The Mendix domain model follows a **snowflake schema** for normalized master data. Data is denormalized and synced to Elasticsearch for OLAP-like analytical queries.

---

## 2. Mendix Domain Model (Transactional)

### 2.1 Entity Relationship Overview

```
+==========================================+
|           MASTER DATA (from N-PLM)       |
|                                          |
|  Site ----+                              |
|    |      |                              |
|  Country  +-- Department ---+            |
|                |            |            |
|              Team        Division        |
|                |            |            |
|             Employee     BusinessUnit    |
+==========================================+
        |              |
        v              v
+==========================================+
|           PLANNING DOMAIN                |
|                                          |
|  ResourceRoadmap ----> RoadmapVersion    |
|       |                    |             |
|       v                    v             |
|  RoadmapProject ----> ProjectVersion     |
|       |                    |             |
|       v                    v             |
|  ResourceAllocation   AllocationVersion  |
|       |                                  |
|       v                                  |
|  ResourceSimulation --> SimulationVersion |
|                                          |
|  PMPlan ------------> PMPlanVersion      |
|       |                    |             |
|       v                    v             |
|  PMEntry             PMEntryVersion      |
+==========================================+
        |
        v
+==========================================+
|           WORKFLOW & CONTROL             |
|                                          |
|  ApprovalRequest --> ApprovalHistory     |
|  FactorControlSet -> FactorControlItem   |
|  SyncRecord (PROMIS/N-PLM tracking)     |
|  NotificationSchedule                    |
+==========================================+
```

### 2.2 Master Data Entities

#### Site (Snowflaked)

```
Site
  |-- site_id (PK)
  |-- site_name                    // Hwaseong, Pyeongtaek, Austin, Xi'an
  |-- site_code
  |-- country_id (FK)
  |     |-- Country
  |           |-- country_id (PK)
  |           |-- country_name
  |           |-- region           // APAC, Americas, EMEA
  |           |-- latitude, longitude  (for world map)
  |-- latitude, longitude          (for world map pin)
  |-- timezone
  |-- is_active
```

#### Organization (Snowflaked)

```
Division
  |-- division_id (PK)
  |-- division_name               // DS, DX, Harman, etc.
  |-- division_code

BusinessUnit
  |-- bu_id (PK)
  |-- bu_name                     // Memory, System LSI, Foundry
  |-- division_id (FK -> Division)

Department
  |-- dept_id (PK)
  |-- dept_name
  |-- dept_code
  |-- bu_id (FK -> BusinessUnit)
  |-- site_id (FK -> Site)
  |-- parent_dept_id (FK, self-reference for hierarchy)

Team
  |-- team_id (PK)
  |-- team_name
  |-- dept_id (FK -> Department)
  |-- team_lead_id (FK -> Employee)

Employee
  |-- employee_id (PK)
  |-- name
  |-- email
  |-- job_title
  |-- job_grade
  |-- team_id (FK -> Team)
  |-- site_id (FK -> Site)
  |-- hire_date
  |-- skill_category              // Design, Process, Test, Package, Mgmt
  |-- is_active
```

#### Project (Snowflaked)

```
Project
  |-- project_id (PK)
  |-- project_name
  |-- project_code
  |-- project_type_id (FK)
  |     |-- ProjectType
  |           |-- type_name       // R&D, Production, Support, Infrastructure
  |           |-- category        // Core, Sustaining, New
  |-- product_family_id (FK)
  |     |-- ProductFamily
  |           |-- family_name     // DRAM, NAND, Exynos, ISOCELL, Foundry
  |           |-- bu_id (FK -> BusinessUnit)
  |-- technology_node             // 3nm, 5nm, 7nm
  |-- priority                    // 1-5
  |-- start_date
  |-- target_end_date
  |-- status                      // Active, On-Hold, Completed, Cancelled
  |-- owner_id (FK -> Employee)
  |-- site_id (FK -> Site)
```

### 2.3 Planning Entities

#### Resource Roadmap & Versioning

```
ResourceRoadmap
  |-- roadmap_id (PK)
  |-- roadmap_name
  |-- site_id (FK -> Site)
  |-- bu_id (FK -> BusinessUnit)
  |-- fiscal_year
  |-- status                      // Draft, Active, Archived
  |-- created_by (FK -> User)
  |-- created_date

RoadmapVersion
  |-- version_id (PK)
  |-- roadmap_id (FK -> ResourceRoadmap)
  |-- version_number              // V1, V2, V3...
  |-- version_type                // Draft, Approved, PROMIS_Synced
  |-- created_by (FK -> User)
  |-- created_date
  |-- approved_by (FK -> User)
  |-- approved_date
  |-- parent_version_id (FK, self) // For diff tracking
  |-- change_summary              // Auto-generated diff summary
  |-- promis_synced               // Boolean
  |-- promis_sync_date

RoadmapProjectAllocation
  |-- allocation_id (PK)
  |-- version_id (FK -> RoadmapVersion)
  |-- project_id (FK -> Project)
  |-- employee_id (FK -> Employee)
  |-- period                      // YYYY-MM
  |-- allocated_percentage        // 0-100
  |-- allocated_hours             // Monthly hours
  |-- role_in_project             // Lead, Member, Support
  |-- notes
```

#### Resource Simulation

```
ResourceSimulation
  |-- simulation_id (PK)
  |-- simulation_name
  |-- source_roadmap_version_id (FK -> RoadmapVersion)  // Created FROM
  |-- status                      // Draft, Running, Completed
  |-- created_by (FK -> User)
  |-- created_date

SimulationVersion
  |-- sim_version_id (PK)
  |-- simulation_id (FK -> ResourceSimulation)
  |-- version_number
  |-- parameters_json             // What-if parameters changed
  |-- created_date

SimulationAllocation
  |-- sim_alloc_id (PK)
  |-- sim_version_id (FK -> SimulationVersion)
  |-- project_id (FK -> Project)
  |-- employee_id (FK -> Employee)
  |-- period
  |-- allocated_percentage
  |-- allocated_hours
  |-- delta_vs_roadmap            // Difference from source roadmap
```

#### Standard P/M (Personal Monthly) Plan

```
PMPlan
  |-- pm_plan_id (PK)
  |-- site_id (FK -> Site)
  |-- dept_id (FK -> Department)
  |-- fiscal_year
  |-- fiscal_month
  |-- status                      // Editing, Draft, PendingApproval, Approved
  |-- lock_owner_id (FK -> User)  // Concurrent edit lock
  |-- lock_timestamp

PMPlanVersion
  |-- pm_version_id (PK)
  |-- pm_plan_id (FK -> PMPlan)
  |-- version_number
  |-- version_type                // AutoSave, Draft, Permanent
  |-- created_by (FK -> User)
  |-- created_date
  |-- approved_by
  |-- approved_date

PMEntry
  |-- entry_id (PK)
  |-- pm_version_id (FK -> PMPlanVersion)
  |-- employee_id (FK -> Employee)
  |-- project_id (FK -> Project)
  |-- period                      // YYYY-MM
  |-- planned_days                // Working days allocated
  |-- planned_hours
  |-- actual_days
  |-- actual_hours
  |-- status                      // Planned, InProgress, Completed
```

#### HeadCount Portfolio

```
HeadCountPlan
  |-- hc_plan_id (PK)
  |-- site_id (FK -> Site)
  |-- dept_id (FK -> Department)
  |-- fiscal_year
  |-- status

HeadCountEntry
  |-- hc_entry_id (PK)
  |-- hc_plan_id (FK -> HeadCountPlan)
  |-- job_grade
  |-- skill_category
  |-- period                      // YYYY-MM or YYYY-QN
  |-- current_count
  |-- planned_count
  |-- target_count
  |-- gap                         // target - planned
  |-- action                      // Hire, Transfer, Retain, Reduce
```

### 2.4 Workflow & Control Entities

```
ApprovalRequest
  |-- request_id (PK)
  |-- entity_type                 // PMPlan, Roadmap, Simulation
  |-- entity_id                   // FK to the versioned entity
  |-- version_id
  |-- requester_id (FK -> User)
  |-- approver_id (FK -> User)
  |-- status                      // Pending, Approved, Rejected
  |-- requested_date
  |-- resolved_date
  |-- comments

FactorControlSet
  |-- fcs_id (PK)
  |-- name
  |-- user_id (FK -> User)        // User's saved filter set
  |-- is_default

FactorControlItem
  |-- fci_id (PK)
  |-- fcs_id (FK -> FactorControlSet)
  |-- dimension                   // site, department, project_type, period, bu
  |-- operator                    // eq, in, between, not_in
  |-- value_json                  // Filter values

SyncRecord
  |-- sync_id (PK)
  |-- target_system               // PROMIS, N-PLM
  |-- entity_type
  |-- entity_id
  |-- version_id
  |-- sync_status                 // Pending, Success, Failed
  |-- sync_date
  |-- changed_projects_json       // List of project IDs that changed (for delta sync)
  |-- error_message
```

---

## 3. Elasticsearch Analytical Model (Denormalized)

### 3.1 Design Principle

```
Mendix Oracle 19c (Snowflake)         Elasticsearch (Denormalized)
+-----------------------+             +-----------------------+
| Site                  |             | {                     |
| Country               |   Sync     |   "site": "Hwaseong", |
| Department            |   Job      |   "country": "Korea", |
| BusinessUnit          |  =======>  |   "dept": "DRAM Dev", |
| Employee              |            |   "bu": "Memory",     |
| Project               |            |   "employee": "Kim",  |
| RoadmapProjectAlloc   |            |   "project": "DDR5",  |
+-----------------------+            |   "period": "2026-04",|
7+ tables, normalized                |   "alloc_pct": 80,   |
                                     |   ...                 |
                                     | }                     |
                                     +-----------------------+
                                     1 document, flat
```

### 3.2 Index: iris-resource-allocation

Primary index for all multi-dimensional analysis of resource allocation.

```json
{
  "mappings": {
    "properties": {
      "allocation_id":        { "type": "keyword" },
      "source_type":          { "type": "keyword" },
      "source_version_id":    { "type": "keyword" },
      "timestamp":            { "type": "date" },

      "employee": {
        "properties": {
          "id":               { "type": "keyword" },
          "name":             { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
          "job_title":        { "type": "keyword" },
          "job_grade":        { "type": "keyword" },
          "skill_category":   { "type": "keyword" },
          "team":             { "type": "keyword" },
          "department":       { "type": "keyword" },
          "business_unit":    { "type": "keyword" },
          "division":         { "type": "keyword" }
        }
      },

      "project": {
        "properties": {
          "id":               { "type": "keyword" },
          "name":             { "type": "keyword" },
          "code":             { "type": "keyword" },
          "type":             { "type": "keyword" },
          "product_family":   { "type": "keyword" },
          "technology_node":  { "type": "keyword" },
          "priority":         { "type": "byte" },
          "status":           { "type": "keyword" }
        }
      },

      "site": {
        "properties": {
          "id":               { "type": "keyword" },
          "name":             { "type": "keyword" },
          "country":          { "type": "keyword" },
          "region":           { "type": "keyword" },
          "geo_point":        { "type": "geo_point" }
        }
      },

      "time": {
        "properties": {
          "period":           { "type": "keyword" },
          "year":             { "type": "short" },
          "quarter":          { "type": "keyword" },
          "month":            { "type": "keyword" }
        }
      },

      "allocation": {
        "properties": {
          "percentage":       { "type": "float" },
          "hours":            { "type": "float" },
          "planned_days":     { "type": "float" },
          "actual_days":      { "type": "float" },
          "role":             { "type": "keyword" }
        }
      },

      "version": {
        "properties": {
          "id":               { "type": "keyword" },
          "number":           { "type": "keyword" },
          "type":             { "type": "keyword" },
          "status":           { "type": "keyword" }
        }
      }
    }
  }
}
```

### 3.3 Index: iris-headcount

```json
{
  "mappings": {
    "properties": {
      "hc_entry_id":          { "type": "keyword" },
      "timestamp":            { "type": "date" },

      "site":                 { "type": "keyword" },
      "country":              { "type": "keyword" },
      "region":               { "type": "keyword" },
      "department":           { "type": "keyword" },
      "business_unit":        { "type": "keyword" },
      "job_grade":            { "type": "keyword" },
      "skill_category":       { "type": "keyword" },

      "period":               { "type": "keyword" },
      "year":                 { "type": "short" },
      "quarter":              { "type": "keyword" },

      "current_count":        { "type": "integer" },
      "planned_count":        { "type": "integer" },
      "target_count":         { "type": "integer" },
      "gap":                  { "type": "integer" },
      "action":               { "type": "keyword" }
    }
  }
}
```

### 3.4 Index: iris-simulation-result

```json
{
  "mappings": {
    "properties": {
      "simulation_id":        { "type": "keyword" },
      "simulation_name":      { "type": "keyword" },
      "sim_version_id":       { "type": "keyword" },
      "source_roadmap":       { "type": "keyword" },
      "timestamp":            { "type": "date" },

      "employee_id":          { "type": "keyword" },
      "employee_name":        { "type": "keyword" },
      "project_id":           { "type": "keyword" },
      "project_name":         { "type": "keyword" },
      "product_family":       { "type": "keyword" },
      "site":                 { "type": "keyword" },
      "department":           { "type": "keyword" },
      "period":               { "type": "keyword" },

      "allocated_percentage": { "type": "float" },
      "allocated_hours":      { "type": "float" },
      "delta_vs_roadmap":     { "type": "float" }
    }
  }
}
```

### 3.5 Index: iris-analysis-report

Stores pre-aggregated report data for periodic analysis reporting.

```json
{
  "mappings": {
    "properties": {
      "report_id":            { "type": "keyword" },
      "report_name":          { "type": "keyword" },
      "report_type":          { "type": "keyword" },
      "generated_date":       { "type": "date" },
      "schedule_id":          { "type": "keyword" },

      "dimensions":           { "type": "object", "dynamic": true },
      "measures":             { "type": "object", "dynamic": true },

      "site":                 { "type": "keyword" },
      "department":           { "type": "keyword" },
      "period":               { "type": "keyword" }
    }
  }
}
```

## 4. Multi-Dimensional Analysis Mapping

### 4.1 Dimension-to-Field Mapping

| OLAP Dimension | Hierarchy | ES Field Path | Mendix Entity |
|---------------|-----------|---------------|---------------|
| **Site** | Region > Country > Site | `site.region` > `site.country` > `site.name` | Site > Country |
| **Organization** | Division > BU > Dept > Team | `employee.division` > `employee.business_unit` > `employee.department` > `employee.team` | Division > BU > Department > Team |
| **Employee** | Grade > SkillCategory > Employee | `employee.job_grade` > `employee.skill_category` > `employee.name` | Employee |
| **Project** | ProductFamily > ProjectType > Project | `project.product_family` > `project.type` > `project.name` | ProductFamily > ProjectType > Project |
| **Time** | Year > Quarter > Month | `time.year` > `time.quarter` > `time.month` | (derived from period) |
| **Version** | VersionType > VersionNumber | `version.type` > `version.number` | RoadmapVersion / PMPlanVersion |

### 4.2 Key Aggregation Queries

#### Headcount by Site (World Map View)

```json
GET iris-resource-allocation/_search
{
  "size": 0,
  "query": { "term": { "version.status": "Approved" } },
  "aggs": {
    "by_site": {
      "terms": { "field": "site.name", "size": 20 },
      "aggs": {
        "unique_employees": { "cardinality": { "field": "employee.id" } },
        "total_allocation_hours": { "sum": { "field": "allocation.hours" } },
        "avg_allocation_pct": { "avg": { "field": "allocation.percentage" } },
        "site_location": { "top_hits": {
          "size": 1, "_source": ["site.geo_point", "site.country"]
        }}
      }
    }
  }
}
```

#### Resource Allocation by Department x Project (Pivot)

```json
GET iris-resource-allocation/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "site.name": "Hwaseong" } },
        { "term": { "time.quarter": "Q2-2026" } }
      ]
    }
  },
  "aggs": {
    "by_department": {
      "terms": { "field": "employee.department", "size": 50 },
      "aggs": {
        "by_project": {
          "terms": { "field": "project.name", "size": 50 },
          "aggs": {
            "headcount": { "cardinality": { "field": "employee.id" } },
            "total_hours": { "sum": { "field": "allocation.hours" } }
          }
        }
      }
    }
  }
}
```

#### Simulation vs Roadmap Comparison

```json
GET iris-simulation-result/_search
{
  "size": 0,
  "query": { "term": { "simulation_id": "SIM-001" } },
  "aggs": {
    "by_project": {
      "terms": { "field": "project_name", "size": 50 },
      "aggs": {
        "total_delta": { "sum": { "field": "delta_vs_roadmap" } },
        "sim_hours": { "sum": { "field": "allocated_hours" } },
        "affected_people": { "cardinality": { "field": "employee_id" } }
      }
    }
  }
}
```

## 5. Data Sync Architecture

```
Mendix Domain Model (Oracle 19c)
        |
        | [M33] ElasticsearchClient Module
        |
        +-- Event-driven sync (on version save/approve):
        |   1. Mendix After-Commit microflow triggers
        |   2. Build denormalized document from related entities
        |   3. Bulk index to Elasticsearch
        |
        +-- Scheduled sync (nightly full refresh):
        |   1. Scheduled event queries all active versions
        |   2. Rebuilds ES indices from scratch
        |   3. Handles any missed event-driven updates
        |
        +-- On-demand sync:
            1. Admin triggers manual reindex
            2. Used after N-PLM master data import
```

---

## Sources

- [Research: Schema Patterns](../../02_research/multi-dimensional-analysis/02-schema-patterns.md)
- [Research: Problem Domain & Dimensional Model](../../02_research/resource-planning-mda/01-problem-domain.md)
- [Research: ES Implementation](../../02_research/resource-planning-mda/06-elasticsearch-implementation.md)
