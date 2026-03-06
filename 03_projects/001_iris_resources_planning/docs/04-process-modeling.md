# Process Modeling - IRIS Resources Planning

## 1. Overview

This document defines the business process flows, state machines, integration sequences, and approval workflows for the IRIS Resources Planning system.

---

## 2. Core Process Map

```
+===========================================================================+
|                     IRIS Process Landscape                                 |
|                                                                           |
|  PLANNING PROCESSES          MANAGEMENT PROCESSES    ANALYSIS PROCESSES   |
|  +--------------------+     +-------------------+   +------------------+ |
|  | P01: P/M Planning  |     | P05: Version Mgmt |   | P08: Ad-Hoc     | |
|  | P02: Roadmap Mgmt  |     | P06: Approval Flow|   |      Analysis   | |
|  | P03: Simulation    |     | P07: Master Data  |   | P09: Periodic   | |
|  | P04: HC Planning   |     |      Sync         |   |      Reporting  | |
|  +--------------------+     +-------------------+   | P10: AI Report  | |
|                                                     +------------------+ |
|  INTEGRATION PROCESSES                                                    |
|  +--------------------------------------------------------------------+  |
|  | P11: PROMIS Delta Sync  |  P12: N-PLM Sync  |  P13: Smart Notify  |  |
|  +--------------------------------------------------------------------+  |
+===========================================================================+
```

---

## 3. P01: Standard P/M Planning Process

### 3.1 Process Flow

```
Start
  |
  v
[Planner opens P/M plan for site+dept+month]
  |
  v
<Is plan locked by another user?>
  |YES                    |NO
  v                       v
[Show "concurrent      [Acquire soft lock]
 edit" notice.           |
 Allow edit anyway]      v
  |                    [Load latest approved version]
  +--------+             |
           |             v
           +------> [Edit allocations in Gantt]
                      |
                      | (user modifies employee-project allocations)
                      v
                   [User clicks Save]
                      |
                      v
                   <Has base version changed since load?>
                      |NO                    |YES
                      v                      v
                   [Save as new          [Run auto-merge]
                    Draft version]           |
                      |                      v
                      |              <Conflicts exist?>
                      |                |NO            |YES
                      |                v              v
                      |             [Save merged   [Show Diff Viewer]
                      |              version]          |
                      |                |               v
                      |                |          <User choice?>
                      |                |           |Pass     |Overwrite
                      |                |           v         v
                      |                |        [Keep      [Keep user's
                      |                |         other's    changes,
                      |                |         changes]   discard other]
                      |                |           |         |
                      +--------+-------+-----------+---------+
                               |
                               v
                         [Draft Version Saved]
                               |
                               v
                         <User role?>
                          |Planner           |Manager
                          v                  v
                    [Save as Draft]    [Save as Permanent]
                          |                  |
                          v                  |
                    [Request Approval]       |
                    [Email to Manager]       |
                          |                  |
                          v                  |
                    (P06: Approval Flow) ----+
                          |
                          v
                    [Permanent Version Created]
                          |
                          v
                    (P11: PROMIS Delta Sync)
                          |
                          v
                        End
```

### 3.2 Concurrent Edit Sequence

```
Planner A               IRIS Server              Planner B
    |                        |                        |
    |-- Open PM Plan ------->|                        |
    |<-- Load V5, Lock(A) ---|                        |
    |                        |                        |
    |   [Editing...]         |<----- Open PM Plan ----|
    |                        |--- Load V5, Mark ------>|
    |                        |    concurrent          |
    |                        |                        |
    |-- Save (changes A) --->|                   [Editing...]
    |<-- V6 Created ---------|                        |
    |                        |                        |
    |                        |<---- Save (changes B) -|
    |                        |                        |
    |                   [Detect V6 > V5]              |
    |                   [Auto-merge A+B]              |
    |                        |                        |
    |                   <Conflict?>                   |
    |                    |NO     |YES                 |
    |                    v       v                    |
    |               [V7 saved] [Send diff to B] ----->|
    |                          |                      |
    |                          |<-- B resolves -------|
    |                          |    (Pass/Overwrite)  |
    |                          |                      |
    |                     [V7 saved]                  |
    |                          |-- V7 Saved ---------->|
```

---

## 4. P02: Resource Roadmap Management Process

```
Start
  |
  v
[Planner opens Roadmap for site+BU+FY]
  |
  v
[View current approved version in Gantt + Table]
  |
  v
[Planner clicks "Create New Version"]
  |
  v
[System clones current version data into new Draft]
  |
  v
[Planner edits in Gantt view]
  | - Add/remove project allocations
  | - Adjust employee assignments
  | - Modify timelines
  |
  v
[Save Draft]
  |
  v
[View Diff: Draft vs Current Approved]
  |
  v
<Satisfied?>
  |NO           |YES
  v              v
[Continue     [Submit for Approval]
 editing]         |
                  v
           (P06: Approval Flow)
                  |
            +-----+-----+
            |           |
         Approved    Rejected
            |           |
            v           v
      [Set as Active  [Return to
       Version]        Draft for edit]
            |
            v
      [Detect changed projects]
      [List: added, modified, removed]
            |
            v
      (P11: PROMIS Delta Sync - changed only)
            |
            v
          End
```

---

## 5. P03: Resource Simulation Process

```
Start
  |
  v
[Planner selects approved Roadmap Version]
  |
  v
[Click "Create Simulation"]
  |
  v
[System clones roadmap data into Simulation V1]
  |
  v
[Planner names simulation, sets scenario description]
  |
  v
[Edit simulation in Gantt view]
  | - Move resources between projects
  | - Add/remove allocations
  | - Test alternative staffing plans
  |
  v
[Save Simulation Version]
  |
  v
<Create another variant?>
  |YES                    |NO
  v                       v
[Clone current version  [View comparison dashboard]
 as new variant]           |
  |                        v
[Edit variant]           [ECharts: Simulation vs Roadmap]
  |                      [Gantt: Overlay view]
  +------>               [Delta Table: Changes summary]
                           |
                           v
                    <Promote to Roadmap?>
                      |NO            |YES
                      v              v
                    [Keep as       [Create new Roadmap
                     simulation]    Version from simulation]
                      |              |
                      v              v
                    End          (P02: Roadmap process
                                  from approval step)
```

---

## 6. P04: HeadCount Portfolio Process

```
Start
  |
  v
[HR Planner opens HC Portfolio for site+dept+quarter]
  |
  v
[View current headcount by grade x skill_category]
  |
  v
[Enter target headcount numbers]
  |
  v
[System calculates gaps: target - current]
  |
  v
[Assign actions per gap entry]
  | - Hire (new positions)
  | - Transfer (internal movement)
  | - Retain (keep current)
  | - Reduce (natural attrition / layoff)
  |
  v
[Save staffing plan]
  |
  v
[View analytics]
  | - ECharts stacked bar: Current vs Target
  | - ECharts trend: Historical headcount
  | - ECharts pie: Gap by action type
  |
  v
[Submit for approval]
  |
  v
(P06: Approval Flow)
  |
  v
End
```

---

## 7. P05: Version Management Process

### 7.1 Version State Machine

```
                    +-------+
                    | Draft |<-----------+
                    +---+---+            |
                        |                |
                  [Submit for            |
                   Approval]         [Rejected -
                        |             Return to
                        v             Draft]
                +-----------+            |
                | Pending   |------------+
                | Approval  |
                +-----+-----+
                      |
                [Manager Approves]
                      |
                      v
                +-----------+
                | Approved  |--------+
                | (Active)  |        |
                +-----+-----+     [New version
                      |            approved]
                [Newer version        |
                 approved]            |
                      |               |
                      v               |
                +-----------+         |
                | Archived  |<--------+
                +-----------+

Special: PROMIS_Synced flag is set independently on any Approved version
         after successful PROMIS delta sync.
```

### 7.2 Diff Engine Process

```
[User requests diff: Version A vs Version B]
  |
  v
[Load all allocation records for Version A]
[Load all allocation records for Version B]
  |
  v
[Match records by composite key: (employee_id + project_id + period)]
  |
  v
[Classify each record:]
  |
  +-- ADDED:     in B but not in A (new allocation)
  +-- REMOVED:   in A but not in B (deleted allocation)
  +-- MODIFIED:  in both, but values differ (hours, percentage changed)
  +-- UNCHANGED: in both, values identical
  |
  v
[Render in Diff Viewer Widget (W03)]
  | - Side-by-side table view
  | - Color coding: green=added, red=removed, yellow=modified
  | - Summary stats: X added, Y removed, Z modified
  |
  v
[Return diff data for PROMIS delta detection if needed]
```

---

## 8. P06: Approval Workflow Process

```
                    Requester                  System                    Approver
                        |                        |                         |
                   [Submit for              [Create                        |
                    Approval] ------------> ApprovalRequest]               |
                        |                   [Status = Pending]            |
                        |                        |                         |
                        |                   [Send email  ----------------->|
                        |                    notification]                 |
                        |                        |                    [Open IRIS]
                        |                        |                    [View pending]
                        |                        |                         |
                        |                        |                    [View Diff:
                        |                        |                     Current vs
                        |                        |                     Previous]
                        |                        |                         |
                        |                        |                    <Decision?>
                        |                        |                     |Approve  |Reject
                        |                        |                     v         v
                        |                   [Update status] <---  [Approve]  [Reject
                        |                        |                             + Comment]
                        |                        |
                        |                   <Approved?>
                        |                    |YES              |NO
                        |                    v                 v
                        |              [Mark version      [Return version
                        |               as Permanent]      to Draft status]
                        |              [Archive prev]     [Notify requester
                        |              [Trigger PROMIS     with rejection
                        |               delta sync]        reason]
                        |                    |
                   [Notification] <----------+
                        |
                       End
```

---

## 9. P07: Master Data Sync (N-PLM) Process

```
+------- Scheduled (Daily, 02:00 AM) -------+
|                                            |
|  [Trigger N-PLM sync job]                  |
|       |                                    |
|       v                                    |
|  [Call N-PLM REST API]                     |
|  [Fetch: Projects, Products, OrgStructure, |
|   Employees, Sites]                        |
|       |                                    |
|       v                                    |
|  [Compare with Mendix DB]                  |
|       |                                    |
|       +-- New records    -> INSERT         |
|       +-- Changed records -> UPDATE        |
|       +-- Removed records -> SOFT DELETE   |
|       |                                    |
|       v                                    |
|  [Log sync results in SyncRecord]          |
|       |                                    |
|       v                                    |
|  [Trigger ES reindex for changed masters]  |
|       |                                    |
|       v                                    |
|  End                                       |
+--------------------------------------------+
```

---

## 10. P11: PROMIS Delta Sync Process

```
[Triggered by: Version Approval (P06)]
  |
  v
[Load newly approved version V(n)]
  |
  v
[Find last PROMIS-synced version V(promis)]
  | (Query SyncRecord where target_system = 'PROMIS' AND status = 'Success')
  |
  v
[Run Diff Engine: V(n) vs V(promis)]
  |
  v
[Extract changed projects list]
  |-- added_projects: [{project_id, allocation_data}, ...]
  |-- modified_projects: [{project_id, old_data, new_data}, ...]
  |-- removed_projects: [{project_id}, ...]
  |
  v
<Any changes?>
  |NO                |YES
  v                  v
[Log: "No changes  [Build delta payload JSON]
 to sync"]           |
  |                  v
  |            [POST to PROMIS REST API]
  |                  |
  |                  v
  |            <API Response?>
  |              |200 OK           |Error
  |              v                 v
  |         [Mark V(n) as       [Log error in SyncRecord]
  |          PROMIS_Synced]     [Alert System Admin]
  |         [Log in SyncRecord] [Retry queue (3 attempts)]
  |              |                 |
  +-------+-----+-----------------+
          |
          v
        End
```

---

## 11. P09: Periodic Analysis Reporting Process

```
[Scheduled trigger: NotificationSchedule entity]
  |
  | (e.g., Every Monday 08:00 AM for weekly report,
  |  1st of month for monthly report)
  |
  v
[Load report configuration]
  |-- dimensions: site, department, project_type
  |-- measures: headcount, allocation_pct, gap
  |-- filters: Factor Control set
  |-- recipients: email list
  |-- format: chart types, layout
  |
  v
[Build ES aggregation query from config]
  |
  v
[Execute against Elasticsearch cluster]
  |
  v
[Generate report content]
  |-- ECharts rendered to image (server-side)
  |-- Data tables formatted
  |-- Summary text generated
  |
  v
[Store report in iris-analysis-report index]
  |
  v
[Smart Notify: Send email to recipients]
  |-- Email body: Summary + key metrics
  |-- Attachment: Report PDF or link to IRIS dashboard
  |
  v
[Log delivery status]
  |
  v
End
```

---

## 12. Key Process Integration Map

```
                    +------------------+
                    |   N-PLM (MDM)    |
                    +--------+---------+
                             |
                        P07: Sync Master Data (daily)
                             |
                             v
+----------+        +--------+---------+        +-----------+
|          |  P01   |                  |  P11   |           |
| Planners |------->|   IRIS Platform  |------->|  PROMIS   |
|          |  P02   |                  | (delta)|  (Profit) |
|          |  P03   |  P05: Versioning |        |           |
|          |  P04   |  P06: Approval   |        +-----------+
+----------+        |                  |
                    |  P08: Analysis   |        +-----------+
+----------+        |  P09: Periodic   |  P10   |           |
|          |  P06   |  P10: AI Report  |------->| Samsung   |
| Managers |------->|                  |        | AI Svc    |
|          |        |                  |        |           |
+----------+        +--------+---------+        +-----------+
                             |
                        P13: Smart Notify
                             |
                             v
                    +--------+---------+
                    |   Email System   |
                    +------------------+
```

---

## 13. Gantt Widget (xDHTML Gantt) Data Flow

```
User Action                 Mendix                    xDHTML Gantt
    |                         |                            |
    |-- Open planning page -->|                            |
    |                         |-- Load Gantt config ------>|
    |                         |   (columns, scale,         |
    |                         |    resources, tasks)       |
    |                         |                            |
    |                         |-- REST: GET /gantt/tasks ->|
    |                         |   {tasks: [...],           |
    |                         |    links: [...]}           |
    |                         |                       [gantt.parse()]
    |                         |                       [Render timeline]
    |<----------------------------------------------------|
    |                                                      |
    |-- Drag task bar (change allocation) ----------------->|
    |                                                      |
    |                         |<-- onAfterTaskDrag event --|
    |                         |    {task_id, new_start,    |
    |                         |     new_end, resource_id}  |
    |                         |                            |
    |                         |-- Nanoflow: validate       |
    |                         |   & save to Mendix DB      |
    |                         |                            |
    |                         |-- WebSocket: broadcast --->| (other users)
    |                         |   change to concurrent     |
    |                         |   editors                  |
```

---

## 14. ECharts Widget Data Flow

```
User Action                 Mendix                    ECharts.js
    |                         |                            |
    |-- Select analysis ---->>|                            |
    |   dimensions/measures   |                            |
    |                         |                            |
    |-- Apply Factor Control ->|                           |
    |                         |                            |
    |                         |-- Build ES agg query       |
    |                         |   from dimension config    |
    |                         |                            |
    |                         |-- POST to ES cluster       |
    |                         |   (via M33 ES Client)      |
    |                         |                            |
    |                         |<-- ES aggregation result   |
    |                         |                            |
    |                         |-- Transform to ECharts -->>|
    |                         |   option format            |
    |                         |   {xAxis, yAxis, series}   |
    |                         |                       [chart.setOption()]
    |                         |                       [Render chart]
    |<----------------------------------------------------|
    |                                                      |
    |-- Click chart element (drill-down) ----------------->|
    |                                                      |
    |                         |<-- onClick event ---------|
    |                         |    {dimension: "Hwaseong", |
    |                         |     level: "site"}         |
    |                         |                            |
    |                         |-- Nanoflow: drill-down     |
    |                         |   Add sub-dimension        |
    |                         |   Re-query ES              |
    |                         |                            |
    |                         |-- Updated option --------->|
    |                         |                       [chart.setOption()]
    |<----------------------------------------------------|
```

---

## Sources

- Samsung IRIS Customer Note (requirements 0-12)
- [Research: Constraint Scheduling](../../02_research/resource-planning-mda/02-constraint-scheduling.md)
- [Research: What-If Scenario Analysis](../../02_research/resource-planning-mda/04-what-if-scenario.md)
- [Research: Integration Architecture](../../02_research/ai-resource-planning/07-integration-architecture.md)
