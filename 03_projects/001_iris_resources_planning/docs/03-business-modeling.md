# Business Modeling - IRIS Resources Planning

## 1. Business Capability Map

```
+===========================================================================+
|                IRIS Resources Planning - Business Capabilities             |
+===========================================================================+
|                                                                           |
|  PLAN                          MANAGE                   ANALYZE           |
|  +------------------------+   +---------------------+   +---------------+ |
|  | P/M Planning           |   | Version Control     |   | Analysis      | |
|  |  - Monthly allocation  |   |  - Roadmap versions |   | Reporting     | |
|  |  - Concurrent editing  |   |  - PM versions      |   |  - Periodic   | |
|  |  - Auto-merge          |   |  - Simulation ver.  |   |  - Manual     | |
|  |  - Conflict resolution |   |  - Diff view        |   |  - AI-based   | |
|  +------------------------+   +---------------------+   +---------------+ |
|  | Resource Roadmap       |   | Approval Workflow   |   | World Map     | |
|  |  - Long-term planning  |   |  - Draft -> Review  |   | View          | |
|  |  - Project allocation  |   |  - Approve/Reject   |   |  - Site stats | |
|  |  - Versioned snapshots |   |  - Email notify     |   |  - Drill-down | |
|  +------------------------+   +---------------------+   +---------------+ |
|  | Resource Simulation    |   | Master Data Mgmt    |   | Personal View | |
|  |  - Clone from roadmap  |   |  - N-PLM sync       |   |  - Individual | |
|  |  - What-if scenarios   |   |  - Project registry |   |    workload   | |
|  |  - Version history     |   |  - Org hierarchy    |   |  - Allocation | |
|  +------------------------+   +---------------------+   +---------------+ |
|  | HeadCount Portfolio    |   | Factor Control      |   | HC Analysis   | |
|  |  - Staffing plan       |   |  - Dimension filter |   |  - Gap report | |
|  |  - Gap analysis        |   |  - Saved filter sets|   |  - Trend      | |
|  |  - Hire/transfer plan  |   |  - Role-based scope |   |  - Staffing   | |
|  +------------------------+   +---------------------+   +---------------+ |
|                                                                           |
|  INTEGRATE                     OPERATE                                    |
|  +------------------------+   +---------------------+                    |
|  | PROMIS Sync            |   | Home & Navigation   |                    |
|  |  - Delta project sync  |   |  - World map home   |                    |
|  |  - Changed-only export |   |  - Role-based menu  |                    |
|  +------------------------+   |  - Site/project nav  |                    |
|  | N-PLM Integration      |   +---------------------+                    |
|  |  - Master data import  |   | Smart Notify        |                    |
|  |  - Bidirectional sync  |   |  - Scheduled emails |                    |
|  +------------------------+   |  - Event alerts     |                    |
|  | Samsung AI Services    |   |  - Report delivery  |                    |
|  |  - AI Excel PIVOT      |   +---------------------+                    |
|  |  - Report generation   |   | Server & Infra      |                    |
|  +------------------------+   |  - QA/Prod setup    |                    |
|                               |  - ES cluster mgmt  |                    |
|                               +---------------------+                    |
+===========================================================================+
```

## 2. Actor Model

### 2.1 Actors & Roles

| Actor | Role | Key Actions |
|-------|------|-------------|
| **Planner** | Resource planner at site/dept level | Create/edit P/M plans, manage roadmap allocations, run simulations |
| **Manager** | Department or team manager | Approve P/M plans, review roadmap versions, approve draft-to-permanent |
| **Site Admin** | Site-level administrator | Manage site data, factor controls, user access per site |
| **Division Manager** | BU or division-level executive | View cross-site analytics, world map, headcount portfolio |
| **HR Planner** | Headcount portfolio manager | Manage staffing plans, gap analysis, hire/transfer planning |
| **Analyst** | Reporting and analysis user | Create analysis reports, configure periodic reports, use AI reporting |
| **System Admin** | Technical administrator | Server config, ES cluster management, sync monitoring, N-PLM config |
| **PROMIS System** | External system (automated) | Receive changed project data from IRIS |
| **N-PLM System** | External system (automated) | Provide/receive master data |
| **Samsung AI** | External service (automated) | Generate AI-based Excel PIVOT reports |

### 2.2 Actor-Feature Matrix

```
                          Planner  Manager  SiteAdmin  DivMgr  HRPlan  Analyst  SysAdmin
P/M Plan - Edit              X
P/M Plan - Approve                    X
P/M Plan - View              X        X        X         X
Resource Roadmap - Edit      X
Resource Roadmap - Approve            X
Resource Roadmap - View      X        X        X         X
Simulation - Create/Edit     X
Simulation - View            X        X        X         X
HeadCount - Edit                                                X
HeadCount - View             X        X        X         X       X
World Map View               X        X        X         X       X       X
Personal View                X        X                  X
Analysis Report - Create                                                 X
Analysis Report - View       X        X        X         X       X       X
AI Reporting                                                             X
Factor Control - Configure            X        X
Factor Control - Use         X        X        X         X       X       X
N-PLM Sync                                                                       X
PROMIS Sync - Monitor                          X                                 X
Server Management                                                                X
```

## 3. Use Cases

### 3.1 UC01: Standard P/M Planning (Concurrent Edit)

```
Actor:    Planner A, Planner B (same department)
Trigger:  Monthly planning cycle begins
Flow:
  1. Planner A opens P/M plan for April 2026, Dept: DRAM Dev, Site: Hwaseong
  2. System loads current version and acquires soft lock
  3. Planner A edits allocations in Gantt view (xDHTML Gantt widget)
  4. Planner B opens same P/M plan (concurrent edit detected, allowed)
  5. Planner A saves -> V2 created (auto-save as Draft)
  6. Planner B saves -> System detects V2 exists since B's load:
     a. Auto-merge non-conflicting changes (different employees/projects)
     b. Conflict detected on Employee Kim, Project DDR5:
        - Show Diff Viewer (Widget W03): A's version vs B's version
        - Planner B chooses: Pass (accept A's) or Overwrite (keep B's)
  7. Merged version saved as V3 (Draft)
  8. Planner A requests approval -> Email sent to Manager
  9. Manager reviews, sees diff (V3 vs V1), approves
  10. V3 marked as Permanent version
  11. Changed projects detected, delta synced to PROMIS

Postcondition: Approved P/M plan, PROMIS updated with changes only
```

### 3.2 UC02: Resource Roadmap with Version Control

```
Actor:    Planner
Trigger:  Annual/quarterly roadmap update
Flow:
  1. Planner opens Resource Roadmap for FY2026
  2. Views current approved version (V5) in Gantt view
  3. Creates new draft version (V6) by cloning V5
  4. Modifies project allocations:
     - Adds new project: "HBM4 Development"
     - Removes 3 engineers from legacy DDR4 project
     - Adjusts timeline for Exynos 2600
  5. Saves V6 as Draft
  6. Opens Diff View: V6 vs V5
     - Shows: added projects (green), removed allocations (red), changed (yellow)
  7. Submits for approval
  8. Manager approves -> V6 becomes Active
  9. System determines changed projects: HBM4 (new), DDR4 (modified), Exynos (modified)
  10. Only these 3 projects sent to PROMIS (not all 50+ projects)
  11. Only these 3 projects sent to N-PLM

Postcondition: New roadmap version active, external systems updated with delta only
```

### 3.3 UC03: Resource Simulation (What-If)

```
Actor:    Planner
Trigger:  Need to evaluate alternative resource allocation
Flow:
  1. Planner selects approved Roadmap Version V6
  2. Clicks "Create Simulation" -> system clones V6 data into new Simulation S1
  3. Simulation S1-V1 is an exact copy of Roadmap V6
  4. Planner modifies simulation:
     - Scenario: "What if we move 5 engineers from NAND to HBM4?"
     - Adjusts allocations in Gantt view
  5. Saves as S1-V2
  6. Views comparison:
     - Gantt overlay: Roadmap V6 (baseline) vs Simulation S1-V2
     - ECharts bar chart: headcount by project (before/after)
     - Delta table: employees affected, hours shifted
  7. Creates another variant: S1-V3 (move 8 engineers instead)
  8. Compares S1-V2 vs S1-V3 in Diff View
  9. Selects preferred simulation
  10. (Optional) Promotes simulation to new Roadmap Version V7

Postcondition: Simulation versions preserved, decision documented
```

### 3.4 UC04: World Map Home Screen

```
Actor:    Division Manager
Trigger:  Opens IRIS home page
Flow:
  1. System displays ECharts world map with Samsung DS site pins
  2. Each site pin shows summary bubble:
     - Total headcount
     - Allocation utilization %
     - Active project count
     - Headcount gap (if any)
  3. Manager clicks on Hwaseong pin -> drills into site view
  4. Site view shows:
     - Department breakdown (ECharts treemap)
     - Project allocation heatmap (ECharts heatmap: dept x project)
     - Headcount trend (ECharts line chart: 12-month trend)
  5. Manager applies Factor Control: filter to "Memory" BU only
  6. All charts update with filtered data (ES aggregation re-runs)
  7. Manager clicks specific department -> drills to team/person view

Postcondition: Executive has multi-level resource visibility
```

### 3.5 UC05: Analysis Reporting (Periodic + Manual)

```
Actor:    Analyst
Trigger:  Monthly report cycle or ad-hoc request

Periodic Flow:
  1. System runs scheduled analysis report (configured by Analyst)
  2. ES aggregation query executes against current approved data
  3. Report generated with ECharts visualizations
  4. Smart Notify sends report via email to configured recipients
  5. Report stored in iris-analysis-report index

Manual Flow:
  1. Analyst opens Analysis Reporting module
  2. Selects dimensions: Site x Department x Project Type
  3. Selects measures: Headcount, Allocation %, Gap
  4. Selects chart type: Heatmap
  5. Applies Factor Control filters
  6. System builds ES aggregation query, returns results
  7. ECharts widget renders interactive visualization
  8. Analyst exports to Excel or triggers AI-based PIVOT via Samsung AI Services

AI-Based Flow:
  1. Analyst clicks "Generate AI Report"
  2. System sends aggregated data + dimension config to Samsung AI Services
  3. AI service returns formatted Excel with PIVOT tables
  4. Analyst downloads the Excel file

Postcondition: Report delivered (email/screen/download)
```

### 3.6 UC06: HeadCount Portfolio & Staffing

```
Actor:    HR Planner
Trigger:  Quarterly headcount review
Flow:
  1. HR Planner opens HeadCount Portfolio for Hwaseong, Q2-2026
  2. Views current staffing by department, grade, skill category
  3. Enters target headcount numbers (hiring plan)
  4. System calculates gaps: target - current = gap per category
  5. HR Planner assigns actions: Hire, Transfer, Retain, Reduce
  6. Saves staffing plan
  7. Views analysis:
     - ECharts stacked bar: Current vs Planned vs Target by department
     - ECharts line: Headcount trend over last 8 quarters
     - ECharts pie: Gap distribution by action type
  8. Submits plan for Manager approval

Postcondition: Staffing plan documented with gap analysis
```

## 4. Feature-to-Requirement Traceability

| Req# | Samsung Note Requirement | Features | Modules |
|------|--------------------------|----------|---------|
| 0 | Enhance P/M Planner, upgrade Mendix 10, custom widgets | Mendix 10 platform, W01 Gantt, W02 ECharts | M10, W01, W02 |
| 1 | Separate Resource Roadmap & Simulation, simulation from roadmap, version history | Roadmap module, Simulation module, version engine | M11, M12, M03 |
| 2 | Version history per project, delta sync to PROMIS & N-PLM, diff view | Version engine, delta detection, diff viewer | M03, M30, M31, W03 |
| 3 | Standard P/M: concurrent save, auto-merge, conflict resolve, draft/permanent, approval | Concurrent edit engine, auto-merge, approval workflow | M10, M03, M04 |
| 4 | Resource Roadmap & Simulation save with versioning | Version engine applied to roadmap + simulation | M11, M12, M03 |
| 5 | N-PLM integration (MDM) | N-PLM connector, master data sync | M31, M02 |
| 6 | Factor Control filtering | Factor control dimension filters | M05 |
| 7 | Main screen renewal (world map, role-based menu) | World map widget, navigation module | W04, M01 |
| 8 | Analysis views (world map, personal) | World map analytics, personal dashboard | M23, M24 |
| 9 | HeadCount Portfolio (staffing, analysis) | HC planning, gap analysis, charts | M13, W02 |
| 10 | Analysis Reporting (periodic + manual, chart widgets) | Report engine, smart notify, ECharts | M21, M32, W02 |
| 11 | AI-Based Reporting (Excel PIVOT via Samsung AI) | Samsung AI connector, Excel export | M22 |
| 12 | New server install & config (QA/Prod, ES cluster) | Infrastructure, cluster config | Infra |

## 5. Business Rules

### BR01: Version Lifecycle

```
Roadmap/PM Version States:
  Draft -> PendingApproval -> Approved -> Archived
                |
                v
             Rejected (back to Draft)

Rules:
- Only Planners can create Draft versions
- Only Managers can Approve/Reject
- Approved version becomes the "current" version
- Previous approved version auto-archived
- Archived versions are read-only but viewable for diff
```

### BR02: PROMIS Delta Sync

```
On version approval:
  1. Compare new version V(n) with last PROMIS-synced version V(promis)
  2. Detect changed projects:
     - Added:    in V(n) but not in V(promis)
     - Modified: in both but values differ
     - Removed:  in V(promis) but not in V(n)
  3. Send ONLY changed project records to PROMIS
  4. Do NOT send unchanged projects
  5. Mark V(n) as PROMIS-synced on success
```

### BR03: Draft vs Permanent (Role-Based)

```
When a user with "Planner" role saves a P/M plan:
  -> Saved as DRAFT version
  -> Email notification sent to Manager
  -> Manager must approve to promote to PERMANENT

When a user with "Manager" role saves:
  -> Saved directly as PERMANENT version (no approval needed)
```

### BR04: Factor Control Scoping

```
Factor Control applies to ALL analytical views and data queries:
  - World Map: Filter sites
  - Gantt: Filter projects/employees
  - ECharts: Filter aggregation scope
  - Reports: Filter report data
  - HeadCount: Filter portfolio scope

Factor Control dimensions:
  - Site (multi-select)
  - Department (multi-select, respects site selection)
  - Business Unit (multi-select)
  - Project Type (multi-select)
  - Time Period (range)
  - Status (active/all)
```

---

## Sources

- Samsung IRIS Customer Note (requirements 0-12)
- [Research: What-If Scenario Analysis](../../02_research/resource-planning-mda/04-what-if-scenario.md)
- [Research: Capacity Planning](../../02_research/resource-planning-mda/03-capacity-planning.md)
