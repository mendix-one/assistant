# Solution Architecture - IRIS Resources Planning

## 1. Executive Summary

IRIS (Intelligent Resource Information System) is a resource planning platform for Samsung Electronics Device Solutions (DS) Division. It manages personnel/monthly (P/M) planning, resource roadmaps, simulations, headcount portfolios, and multi-dimensional analysis reporting across global semiconductor sites (Hwaseong, Pyeongtaek, Austin, Xi'an, etc.).

The system is built on **Mendix 10** low-code platform with **Elasticsearch** for multi-dimensional analytics, **xDHTML Gantt** for timeline visualization, and **ECharts.js** for analytical charts and dashboards.

---

## 2. Architecture Principles

| Principle | Description |
|-----------|-------------|
| **Low-Code First** | Maximize Mendix for business logic, UI, and workflows; custom code only where Mendix cannot reach |
| **Multi-Dimensional Analytics** | Elasticsearch aggregations power all analytical views (OLAP-like slice/dice/drill/roll-up) |
| **Version-Controlled Planning** | Every planning artifact (roadmap, simulation, P/M) is versioned with diff capability |
| **Integration-Ready** | Clean APIs to PROMIS, N-PLM, Samsung AI Services, and email notification systems |
| **Clustered & Scalable** | HA cluster architecture for all layers including ES master-data node topology |

---

## 3. System Context

```
+------------------------------------------------------------------+
|                        External Systems                           |
|                                                                   |
|  +----------+  +--------+  +------------+  +------------------+  |
|  | PROMIS   |  | N-PLM  |  | Samsung AI |  | Email / Smart    |  |
|  | (Profit) |  | (MDM)  |  | Services   |  | Notify           |  |
|  +----+-----+  +---+----+  +-----+------+  +--------+---------+  |
|       |             |             |                   |           |
+-------|-------------|-------------|-------------------|----------+
        |             |             |                   |
+-------v-------------v-------------v-------------------v----------+
|                                                                   |
|                    IRIS Platform (Mendix 10)                      |
|                                                                   |
|  +-------------------------------------------------------------+ |
|  |                     API Gateway Layer                        | |
|  |  REST APIs | OData | WebSocket (real-time Gantt updates)     | |
|  +-------------------------------------------------------------+ |
|                              |                                    |
|  +-------------------------------------------------------------+ |
|  |                  Application Layer (Mendix)                  | |
|  |                                                              | |
|  | +-------------+ +-------------+ +--------------+            | |
|  | | P/M Planner | | Resource    | | HeadCount    |            | |
|  | | Module      | | Roadmap &   | | Portfolio    |            | |
|  | |             | | Simulation  | | Module       |            | |
|  | +-------------+ +-------------+ +--------------+            | |
|  |                                                              | |
|  | +-------------+ +-------------+ +--------------+            | |
|  | | Analysis &  | | Factor      | | Home &       |            | |
|  | | Reporting   | | Control     | | Navigation   |            | |
|  | | Module      | | Module      | | Module       |            | |
|  | +-------------+ +-------------+ +--------------+            | |
|  |                                                              | |
|  | +-------------+ +--------------+                            | |
|  | | Version &   | | Approval &   |                            | |
|  | | Diff Engine | | Workflow     |                            | |
|  | +-------------+ +--------------+                            | |
|  +-------------------------------------------------------------+ |
|                              |                                    |
|  +-------------------------------------------------------------+ |
|  |                    Data Layer                                | |
|  |                                                              | |
|  | +------------------+  +-------------------+                 | |
|  | | Mendix Database  |  | Elasticsearch     |                 | |
|  | | (Oracle 19c)     |  | Cluster           |                 | |
|  | |                  |  |                   |                 | |
|  | | - Domain model   |  | - Resource indices |                 | |
|  | | - Version store  |  | - Capacity indices |                 | |
|  | | - User/Role      |  | - Demand indices   |                 | |
|  | | - Workflow state  |  | - Analysis indices |                 | |
|  | +------------------+  +-------------------+                 | |
|  +-------------------------------------------------------------+ |
+------------------------------------------------------------------+
|                                                                   |
|  +---------------------+  +---------------------+               |
|  | xDHTML Gantt         |  | ECharts.js          |               |
|  | (Custom Widget)      |  | (Custom Widget)     |               |
|  | - Resource timeline  |  | - Analytical charts |               |
|  | - P/M Gantt          |  | - World map         |               |
|  | - Simulation Gantt   |  | - Pivot tables      |               |
|  +---------------------+  +---------------------+               |
+------------------------------------------------------------------+
```

## 4. Technology Stack

### 4.1 Core Platform

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Application** | Mendix | 10.x | Low-code platform, UI, business logic, workflows |
| **Database** | Oracle Database | 19c | Mendix domain model persistence, transactional data |
| **Search & Analytics** | Elasticsearch | 8.x | Multi-dimensional aggregations, full-text search, analytics |
| **Runtime** | Mendix Runtime (Java) | 10.x | Server-side execution |

### 4.2 Frontend Visualization (Custom Widgets)

| Widget | Technology | Purpose |
|--------|-----------|---------|
| **Gantt Chart** | xDHTML Gantt | Resource timelines, P/M planning, roadmap visualization, simulation |
| **Analytical Charts** | ECharts.js | Bar, line, pie, heatmap, treemap, radar, sankey charts |
| **World Map** | ECharts.js (map module) | Global site visualization with resource statistics overlay |
| **Pivot Table** | ECharts.js + custom | Multi-dimensional data pivoting for analysis reporting |
| **Diff Viewer** | Custom Mendix widget | Version comparison (roadmap, P/M, simulation diffs) |

### 4.3 Integration

| System | Protocol | Direction | Data |
|--------|----------|-----------|------|
| **PROMIS** (Profit System) | REST API | IRIS -> PROMIS | Changed project data only (delta sync) |
| **N-PLM** (Master Data) | REST API | Bidirectional | Master data (projects, products, org structure) |
| **Samsung AI Services** | REST API | IRIS -> AI -> IRIS | AI-based Excel PIVOT generation, reporting |
| **Email / Smart Notify** | SMTP / API | IRIS -> Mail | Approval notifications, periodic analysis reports |

### 4.4 Infrastructure

| Component | Configuration |
|-----------|--------------|
| **Mendix App Server** | Clustered (2+ nodes), load balanced |
| **Oracle Database** | Primary + Standby (Oracle Data Guard) |
| **Elasticsearch** | Cluster: 3 master-eligible + N data nodes (hot/warm) |
| **Reverse Proxy** | Nginx / HAProxy for SSL termination + LB |
| **Environments** | QA + Production (new servers April-July 2026) |

## 5. Elasticsearch Cluster Architecture

```
                        Load Balancer
                            |
              +-------------+-------------+
              |             |             |
         +----v----+  +----v----+  +----v----+
         | Master  |  | Master  |  | Master  |
         | Node 1  |  | Node 2  |  | Node 3  |
         | (coord) |  | (coord) |  | (coord) |
         +---------+  +---------+  +---------+
              |             |             |
    +---------+------+------+------+------+---------+
    |         |      |      |      |      |         |
+---v---+ +--v---+ +v----+ +v----+ +v---+ +---v---+
| Data  | | Data | | Data | | Data | | Data| | Data  |
| Hot 1 | | Hot 2| | Hot 3| | Hot 4| |Warm1| | Warm2 |
+-------+ +------+ +------+ +------+ +-----+ +-------+

Hot nodes:  Current quarter data, active planning indices
Warm nodes: Historical data, archived versions, analysis reporting
```

### Index Strategy

| Index Pattern | Lifecycle | Shard Strategy |
|---------------|-----------|----------------|
| `iris-resource-allocation-*` | Hot: current quarter, Warm: 90d+, Delete: never | 3 primary, 1 replica |
| `iris-capacity-plan-*` | Hot: current, Warm: after approval | 2 primary, 1 replica |
| `iris-pm-planning-*` | Hot: active version, Warm: archived versions | 3 primary, 1 replica |
| `iris-simulation-*` | Hot: active, Warm: 30d after close | 2 primary, 1 replica |
| `iris-headcount-*` | Hot: current, Warm: historical | 2 primary, 1 replica |
| `iris-analysis-report-*` | Hot: last 90d, Warm: 90d+ | 2 primary, 1 replica |

## 6. Application Module Architecture

### 6.1 Module Breakdown

```
IRIS Application (Mendix 10)
|
+-- Core Modules
|   +-- [M01] UserManagement       - Roles, departments, site access, SSO
|   +-- [M02] MasterData           - Projects, products, org hierarchy, sites (synced from N-PLM)
|   +-- [M03] VersionEngine        - Generic versioning, diff engine, snapshot management
|   +-- [M04] ApprovalWorkflow     - Draft/review/approve states, email notifications
|   +-- [M05] FactorControl        - Filter dimensions (site, dept, project type, time period)
|
+-- Planning Modules
|   +-- [M10] PMPlanner            - Standard P/M planning (concurrent edit, auto-merge, conflict resolve)
|   +-- [M11] ResourceRoadmap      - Long-term resource roadmap management (versioned)
|   +-- [M12] ResourceSimulation   - Simulation created from roadmap, version history
|   +-- [M13] HeadCountPortfolio   - Staffing plan, headcount analysis views
|
+-- Analysis Modules
|   +-- [M20] AnalysisEngine       - Multi-dimensional aggregation via Elasticsearch
|   +-- [M21] AnalysisReporting    - Periodic + manual reports, chart widgets, Excel export
|   +-- [M22] AIReporting          - Samsung AI Services integration for PIVOT generation
|   +-- [M23] WorldMapView         - ECharts world map with site-level resource statistics
|   +-- [M24] PersonalView         - Individual resource analysis dashboard
|
+-- Integration Modules
|   +-- [M30] PROMISConnector      - Delta sync of changed projects to PROMIS
|   +-- [M31] NPLMConnector        - Master data sync with N-PLM
|   +-- [M32] SmartNotify          - Email notification engine (scheduled + event-triggered)
|   +-- [M33] ElasticsearchClient  - ES query builder, index management, sync engine
|
+-- UI Modules (Custom Widgets)
    +-- [W01] GanttWidget          - xDHTML Gantt wrapper for Mendix
    +-- [W02] EChartsWidget        - ECharts.js wrapper for Mendix
    +-- [W03] DiffViewerWidget     - Side-by-side version comparison
    +-- [W04] WorldMapWidget       - ECharts map with interactive drill-down
    +-- [W05] PivotTableWidget     - Multi-dimensional pivot table
```

### 6.2 Custom Widget Architecture

```
Mendix Page
  |
  +-- Mendix Data View (context: PlanningSession entity)
       |
       +-- [W01] GanttWidget (xDHTML Gantt)
       |    |
       |    +-- Props from Mendix:
       |    |   - dataSource: REST endpoint /api/gantt/tasks
       |    |   - config: columns, scale, editable, resources
       |    |   - onTaskUpdate: Mendix nanoflow (save changes)
       |    |   - onConflict: Mendix nanoflow (show diff dialog)
       |    |
       |    +-- Internal:
       |         - gantt.init(container, config)
       |         - gantt.parse(data, "json")
       |         - gantt.attachEvent("onAfterTaskDrag", callback)
       |         - WebSocket for concurrent edit sync
       |
       +-- [W02] EChartsWidget
            |
            +-- Props from Mendix:
            |   - chartType: "bar" | "line" | "pie" | "heatmap" | "treemap" | "map"
            |   - dataSource: REST endpoint /api/analysis/aggregate
            |   - dimensions: ["site", "department", "project"]
            |   - measures: ["headcount", "allocation_pct"]
            |   - drillDownEnabled: true
            |   - onDrillDown: Mendix nanoflow
            |
            +-- Internal:
                 - echarts.init(container, theme)
                 - chart.setOption(buildOption(data, config))
                 - chart.on('click', handleDrillDown)
```

## 7. Integration Architecture

### 7.1 N-PLM Integration (Master Data)

```
N-PLM (Source of Truth)              IRIS
+-----------------+                  +-----------------+
| Projects        |  REST/Scheduled  | MasterData      |
| Products        |  Pull (daily)    | Module [M02]    |
| Org Structure   |  ==============> |                 |
| Site Hierarchy  |                  | Oracle 19c      |
| Employee Data   |                  | (Mendix DB)     |
+-----------------+                  +--------+--------+
                                              |
                                     Sync to ES indices
                                     (denormalized for analytics)
```

### 7.2 PROMIS Integration (Delta Sync)

```
IRIS                                    PROMIS
+-------------------+                   +-----------+
| VersionEngine     |                   |           |
| [M03]             |                   | Profit    |
|                   |  Only changed     | System    |
| Compare current   |  projects         |           |
| version vs last   |  ===============> | Receives  |
| synced version    |  REST POST        | project   |
|                   |  (delta payload)  | updates   |
| Mark synced       |                   |           |
+-------------------+                   +-----------+

Delta Detection Logic:
  1. On version approval, diff current vs last-PROMIS-synced version
  2. Extract list of changed projects (added, modified, removed)
  3. Send only changed project records to PROMIS API
  4. Mark current version as "PROMIS-synced"
```

### 7.3 Samsung AI Services Integration

```
IRIS                                    Samsung AI
+-------------------+                   +-----------+
| AIReporting [M22] |                   |           |
|                   |  POST /analyze    | AI Engine |
| User requests     |  {                |           |
| AI-based report   |    data: [...],   | Generate  |
|                   |    format: "pivot",| Excel     |
| Send aggregated   |    dimensions:    | PIVOT     |
| data + config     |    [...],         |           |
|                   |    measures: [...] | Return    |
| Receive Excel     |  }                | .xlsx     |
| Download          |  <=============== |           |
+-------------------+                   +-----------+
```

## 8. Concurrent Editing Architecture (P/M Planner)

```
User A (editing Project X)         User B (editing Project X)
    |                                   |
    v                                   v
+--------+                         +--------+
| Mendix |                         | Mendix |
| Client |                         | Client |
+---+----+                         +---+----+
    |                                   |
    |  WebSocket (real-time sync)       |
    +----------->+----------+<----------+
                 | Mendix   |
                 | Runtime  |
                 +----+-----+
                      |
              +-------v-------+
              | Conflict       |
              | Detection      |
              | Engine [M03]   |
              +-------+-------+
                      |
          +-----------+-----------+
          |                       |
    +-----v-----+          +-----v-----+
    | Auto-Merge |          | Conflict  |
    | (no        |          | Detected  |
    | conflict)  |          |           |
    +-----+------+          +-----+-----+
          |                       |
     Save merged             Show Diff View
     version                 [W03 Widget]
                                  |
                           User chooses:
                           Pass / Overwrite
```

### Conflict Resolution Flow

1. User A opens P/M plan -> system creates **edit lock record** with timestamp
2. User B opens same plan -> system detects concurrent edit, allows but marks as "concurrent"
3. User A saves -> **base version V1** created
4. User B saves -> system detects V1 changed since B loaded it:
   - **Auto-merge** if changes are in different projects/rows
   - **Conflict dialog** if same project/row modified by both
   - User B sees diff view (B's changes vs A's saved version)
   - User B chooses: **Pass** (discard own) or **Overwrite** (keep own)

## 9. Deployment Topology

### 9.1 Production Environment (Post July 2026)

```
                     Internet / Intranet
                          |
                    +-----v------+
                    | Load       |
                    | Balancer   |
                    | (HAProxy)  |
                    +--+------+--+
                       |      |
              +--------v--+ +-v---------+
              | Mendix    | | Mendix    |
              | App Node 1| | App Node 2|
              | (Active)  | | (Active)  |
              +-----+-----+ +-----+-----+
                    |              |
          +---------+--------------+---------+
          |                                  |
   +------v------+                  +--------v--------+
   | Oracle 19c  |                  | Elasticsearch   |
   | Primary     |                  | Cluster         |
   |             |                  | (3M + 4D hot    |
   | Standby     |                  |  + 2D warm)     |
   | (Data Guard)|                  |                 |
   +-------------+                  +-----------------+
```

### 9.2 Environment Strategy

| Environment | Timeline | Purpose |
|-------------|----------|---------|
| **DEV** | Current | Development and unit testing |
| **QA** | April 2026 | Integration testing, UAT |
| **PROD** | July 2026 | Production with cluster architecture |

## 10. Security Architecture

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | Samsung SSO (SAML 2.0 / OIDC) via Mendix SSO module |
| **Authorization** | Role-based: Admin, Manager, Planner, Viewer per site/department |
| **Data Isolation** | Factor Control filters enforce site/department visibility |
| **API Security** | JWT tokens for REST APIs, API key for ES cluster |
| **Audit Trail** | Version engine tracks all changes with user, timestamp, diff |
| **Network** | HTTPS everywhere, ES cluster on private subnet, no public ES access |

---

## Sources

- Samsung IRIS Customer Note (requirements)
- [Research: Resource Planning MDA - Integration Architecture](../../02_research/ai-resource-planning/07-integration-architecture.md)
- [Research: Elasticsearch Implementation](../../02_research/resource-planning-mda/06-elasticsearch-implementation.md)
