# Data Sync & ETL - Oracle 19c to Elasticsearch

## 1. Executive Summary

IRIS operates a **dual-engine data architecture**: Oracle 19c as the transactional system of record (via Mendix) and Elasticsearch as the analytical query engine. This document defines the ETL data flow, consistency guarantees, failure handling, and performance strategies that keep both engines synchronized and performant.

### Why Two Engines?

| Concern | Oracle 19c (Mendix DB) | Elasticsearch |
|---------|----------------------|---------------|
| **Purpose** | Transactional CRUD, ACID compliance, referential integrity | Multi-dimensional aggregations, full-text search, analytical dashboards |
| **Data Shape** | Normalized (snowflake schema, 15+ tables) | Denormalized (flat documents, 4 indices) |
| **Query Pattern** | Single-entity lookups, joins, transactional writes | Aggregation across millions of records, slice/dice/drill |
| **Consistency** | Strong (ACID transactions) | Eventual (near real-time, ~1-5 seconds) |
| **Users** | Planners editing P/M plans, roadmaps, simulations | Analysts viewing dashboards, world map, reports |

**Architectural Rule:** Oracle is the **single source of truth**. Elasticsearch is a **read-optimized projection**. All writes go to Oracle first; ES is always derived from Oracle data.

---

## 2. Data Flow Overview

```
+=======================================================================+
|                         WRITE PATH                                     |
|                                                                        |
|  User Action (Mendix UI)                                               |
|       |                                                                |
|       v                                                                |
|  Mendix Runtime (Java)                                                 |
|       |                                                                |
|       v                                                                |
|  Oracle 19c  ---- COMMIT (ACID) ---->  Data persisted                  |
|       |                                                                |
|       +-- After-Commit Event -----+                                    |
|                                   |                                    |
|                                   v                                    |
|                          Sync Dispatcher                               |
|                          [M33 Module]                                   |
|                                   |                                    |
|                     +-------------+-------------+                      |
|                     |                           |                      |
|                     v                           v                      |
|              Sync Queue Table            (Fallback: Scheduled          |
|              (Oracle)                     Full Sync catches            |
|                     |                     any missed events)           |
|                     v                                                  |
|              Sync Worker                                               |
|              (Mendix Scheduled Event,                                  |
|               runs every 30 seconds)                                   |
|                     |                                                  |
|                     v                                                  |
|              Build Denormalized Document                               |
|              (Join 7+ tables into flat JSON)                           |
|                     |                                                  |
|                     v                                                  |
|              Elasticsearch Bulk API                                    |
|              (_bulk index/update/delete)                                |
|                     |                                                  |
|                     v                                                  |
|              ES Document Available                                     |
|              (searchable after refresh,                                 |
|               default 1 second)                                        |
+=======================================================================+

+=======================================================================+
|                         READ PATH                                      |
|                                                                        |
|  Transactional Reads          Analytical Reads                         |
|  (Mendix pages, forms)       (Dashboards, charts, reports)            |
|       |                            |                                   |
|       v                            v                                   |
|  Oracle 19c                   Elasticsearch                            |
|  (via Mendix ORM)            (via M33 REST client)                    |
|                                    |                                   |
|                                    v                                   |
|                              ECharts / Gantt / World Map               |
+=======================================================================+
```

---

## 3. Sync Strategies

### 3.1 Strategy Overview

| Strategy | Trigger | Latency | Scope | Purpose |
|----------|---------|---------|-------|---------|
| **Event-Driven** | After-Commit on Oracle | 1-5 seconds | Changed entities only | Real-time sync for user actions |
| **Scheduled Micro-Batch** | Every 30 seconds | 30 seconds max | Queued items | Process sync queue reliably |
| **Scheduled Full Sync** | Nightly (02:00 AM) | N/A (batch) | All active data | Catch-up, repair drift, master data refresh |
| **On-Demand Reindex** | Admin trigger | Minutes | Full or partial | Post-migration, post-import, recovery |

### 3.2 Event-Driven Sync (Primary)

This is the main sync path for all user-initiated changes.

```
Step 1: User saves data in Mendix
        |
        v
Step 2: Mendix commits to Oracle (ACID transaction)
        |
        |  After-Commit microflow fires (Mendix event handler)
        |
        v
Step 3: Insert record into SyncQueue table (SAME Oracle transaction
        is already committed, so this is a NEW micro-transaction)

        SyncQueue row:
        +----------------------------------------------------------+
        | queue_id | entity_type      | entity_id | action | status |
        |----------|------------------|-----------|--------|--------|
        | 10042    | PMPlanVersion    | V-2847    | UPSERT | PENDING|
        | 10043    | RoadmapVersion   | V-0091    | UPSERT | PENDING|
        +----------------------------------------------------------+
        | created_at          | retry_count | error_message         |
        |---------------------|-------------|----------------------|
        | 2026-03-05 14:22:01 | 0           | NULL                 |
        | 2026-03-05 14:22:01 | 0           | NULL                 |
        +----------------------------------------------------------+

Step 4: Sync Worker (scheduled event, every 30s) picks up PENDING items
        |
        v
Step 5: For each queue item:
        a. Query Oracle: fetch entity + all related dimension data
        b. Build denormalized ES document (join Employee, Project,
           Site, Department, Team, etc.)
        c. Call ES Bulk API to index
        d. On success: mark queue item as COMPLETED
        e. On failure: increment retry_count, mark as RETRY
        |
        v
Step 6: Document available in ES (~1s after index call)
```

#### SyncQueue Entity (Oracle)

```
SyncQueue
  |-- queue_id (PK, auto-increment)
  |-- entity_type           // PMPlanVersion, RoadmapVersion, SimulationVersion,
  |                         // HeadCountEntry, MasterData
  |-- entity_id             // PK of the changed entity
  |-- action                // UPSERT, DELETE
  |-- status                // PENDING, PROCESSING, COMPLETED, RETRY, FAILED
  |-- target_index          // iris-resource-allocation, iris-headcount, etc.
  |-- created_at            // Timestamp of queue insertion
  |-- processed_at          // Timestamp of processing
  |-- retry_count           // 0, 1, 2, 3 (max 3)
  |-- error_message         // NULL or error details
  |-- batch_id              // Groups items processed together
```

#### Which Events Trigger Sync?

| Mendix Event | Entity Type | ES Index Target | Action |
|-------------|-------------|-----------------|--------|
| P/M Plan version saved (Draft/Permanent) | PMPlanVersion + PMEntry | `iris-resource-allocation` | UPSERT all entries for this version |
| Roadmap version saved/approved | RoadmapVersion + RoadmapProjectAllocation | `iris-resource-allocation` | UPSERT all allocations for this version |
| Simulation version saved | SimulationVersion + SimulationAllocation | `iris-simulation-result` | UPSERT all simulation allocations |
| HeadCount plan saved | HeadCountEntry | `iris-headcount` | UPSERT all entries |
| Version approved (any) | Version entity | `iris-resource-allocation` | Update version.status field |
| Version archived | Version entity | `iris-resource-allocation` | Update version.status or DELETE if configured |
| Master data changed (N-PLM import) | Employee, Project, Site, Dept | ALL indices | FULL REINDEX (master data affects all documents) |

### 3.3 Denormalization Transform

The core ETL transformation: join normalized Oracle tables into flat ES documents.

```
Oracle (Normalized)                      Elasticsearch (Denormalized)

RoadmapProjectAllocation                 {
  allocation_id = "A-001"                  "allocation_id": "A-001",
  version_id = "V-091"                    "source_type": "Roadmap",
  project_id = "P-042"  ----+             "source_version_id": "V-091",
  employee_id = "E-108" -+  |             "timestamp": "2026-03-05T14:22:00Z",
  period = "2026-04"     |  |
  allocated_pct = 80     |  |             "employee": {
  allocated_hours = 128  |  |               "id": "E-108",
                         |  |               "name": "Kim Minjun",
Employee (E-108)  <------+  |               "job_title": "Senior Engineer",
  name = "Kim Minjun"       |               "job_grade": "G7",
  job_grade = "G7"           |               "skill_category": "Design",
  team_id = "T-22"  --------+---+           "team": "HBM Design Team",
  site_id = "S-01"  -----+  |   |           "department": "DRAM Development",
                          |  |   |           "business_unit": "Memory",
Team (T-22)         <-----+--+   |           "division": "DS"
  team_name = "HBM Design"  |   |         },
  dept_id = "D-05"  ---------+  |
                              |  |         "project": {
Department (D-05)       <-----+  |           "id": "P-042",
  dept_name = "DRAM Dev"     |   |           "name": "HBM4 Development",
  bu_id = "BU-01"  ----------+  |           "code": "HBM4-DEV",
                              |  |           "type": "R&D",
BusinessUnit (BU-01)    <-----+  |           "product_family": "DRAM",
  bu_name = "Memory"         |   |           "technology_node": "3nm",
  division_id = "DIV-01" ----+   |           "priority": 1,
                              |  |           "status": "Active"
Division (DIV-01)       <-----+  |         },
  division_name = "DS"           |
                                 |         "site": {
Project (P-042)         <--------+           "id": "S-01",
  project_name = "HBM4 Dev"                 "name": "Hwaseong",
  project_type = "R&D"                      "country": "Korea",
  product_family = "DRAM"                    "region": "APAC",
  technology_node = "3nm"                    "geo_point": {"lat":37.2,"lon":127.0}
                                           },
Site (S-01)
  site_name = "Hwaseong"                   "time": {
  country = "Korea"                          "period": "2026-04",
  region = "APAC"                            "year": 2026,
                                             "quarter": "Q2-2026",
RoadmapVersion (V-091)                       "month": "2026-04"
  version_number = "V3"                    },
  version_type = "Approved"
  status = "Active"                        "allocation": {
                                             "percentage": 80,
                                             "hours": 128,
                                             "planned_days": 16,
                                             "actual_days": null,
                                             "role": "Member"
                                           },

                                           "version": {
                                             "id": "V-091",
                                             "number": "V3",
                                             "type": "Approved",
                                             "status": "Active"
                                           }
                                         }
```

#### Denormalization Query (Oracle SQL)

```sql
SELECT
    rpa.allocation_id,
    'Roadmap' AS source_type,
    rv.version_id AS source_version_id,
    SYSTIMESTAMP AS sync_timestamp,
    -- Employee dimension (5 joins)
    e.employee_id, e.name AS employee_name,
    e.job_title, e.job_grade, e.skill_category,
    t.team_name,
    d.dept_name AS department,
    bu.bu_name AS business_unit,
    div.division_name AS division,
    -- Project dimension (2 joins)
    p.project_id, p.project_name, p.project_code,
    pt.type_name AS project_type,
    pf.family_name AS product_family,
    p.technology_node, p.priority, p.status AS project_status,
    -- Site dimension (1 join)
    s.site_id, s.site_name, c.country_name, c.region,
    s.latitude, s.longitude,
    -- Time dimension (derived)
    rpa.period,
    EXTRACT(YEAR FROM TO_DATE(rpa.period, 'YYYY-MM')) AS time_year,
    'Q' || TO_CHAR(TO_DATE(rpa.period, 'YYYY-MM'), 'Q') || '-'
        || EXTRACT(YEAR FROM TO_DATE(rpa.period, 'YYYY-MM')) AS time_quarter,
    -- Measures
    rpa.allocated_percentage, rpa.allocated_hours,
    rpa.role_in_project,
    -- Version
    rv.version_id, rv.version_number, rv.version_type, rv.version_type AS version_status
FROM RoadmapProjectAllocation rpa
    JOIN RoadmapVersion rv  ON rpa.version_id = rv.version_id
    JOIN Employee e         ON rpa.employee_id = e.employee_id
    JOIN Team t             ON e.team_id = t.team_id
    JOIN Department d       ON t.dept_id = d.dept_id
    JOIN BusinessUnit bu    ON d.bu_id = bu.bu_id
    JOIN Division div       ON bu.division_id = div.division_id
    JOIN Project p          ON rpa.project_id = p.project_id
    JOIN ProjectType pt     ON p.project_type_id = pt.type_id
    JOIN ProductFamily pf   ON p.product_family_id = pf.family_id
    JOIN Site s             ON e.site_id = s.site_id
    JOIN Country c          ON s.country_id = c.country_id
WHERE rv.version_id = :versionId
```

> **Note:** This query joins 12 tables. In Oracle 19c, ensure proper indexes exist on all FK columns and use execution plan analysis (see Section 7).

### 3.4 Scheduled Full Sync (Nightly Catch-Up)

```
[Scheduled Event: 02:00 AM daily]
  |
  v
[Query Oracle: all active versions across all entity types]
  |-- Active RoadmapVersions (status = 'Active' or 'Approved')
  |-- Active PMPlanVersions (status = 'Approved' or 'Draft' for current month)
  |-- Active SimulationVersions (status != 'Archived')
  |-- All HeadCountEntries (current fiscal year)
  |
  v
[For each entity set:]
  |
  +-- Step 1: Run denormalization query (see 3.3)
  |
  +-- Step 2: Build ES bulk payload
  |           (batch size: 500-1000 documents per bulk request)
  |
  +-- Step 3: Execute ES Bulk API
  |           POST _bulk
  |           { "index": { "_index": "iris-resource-allocation", "_id": "A-001" } }
  |           { ...document... }
  |           { "index": { "_index": "iris-resource-allocation", "_id": "A-002" } }
  |           { ...document... }
  |
  +-- Step 4: Verify counts
  |           Oracle count vs ES count per index
  |           If mismatch > threshold (0.1%): alert admin
  |
  +-- Step 5: Clean up
  |           - Purge COMPLETED SyncQueue records older than 7 days
  |           - Reset any RETRY items that exceeded max retries to FAILED
  |           - Log sync summary to SyncAuditLog
  |
  v
[End - Log total sync duration and document counts]
```

### 3.5 On-Demand Reindex

```
[Admin triggers reindex via IRIS Admin page]
  |
  v
[Select scope:]
  |-- Full reindex (all indices)
  |-- Single index (e.g., iris-resource-allocation only)
  |-- Single version (e.g., Roadmap V6 only)
  |
  v
[For full/single index:]
  |
  +-- Step 1: Create new index with timestamp suffix
  |           iris-resource-allocation-20260305
  |
  +-- Step 2: Bulk index all data from Oracle into new index
  |
  +-- Step 3: Verify document counts (Oracle vs new ES index)
  |
  +-- Step 4: Swap alias atomically
  |           POST _aliases
  |           { "actions": [
  |             { "remove": { "index": "iris-resource-allocation-20260301",
  |                           "alias": "iris-resource-allocation" } },
  |             { "add":    { "index": "iris-resource-allocation-20260305",
  |                           "alias": "iris-resource-allocation" } }
  |           ]}
  |
  +-- Step 5: Delete old index after confirmation
  |
  v
[Zero-downtime reindex complete]
```

---

## 4. Data Consistency Guarantees

### 4.1 Consistency Model

```
+-----------------------------------------------------------------------+
|                    Consistency Spectrum                                 |
|                                                                        |
|  Strong <<<-----------+---------+----------+---------->>> Eventual     |
|  Consistency           |         |          |            Consistency    |
|                        |         |          |                          |
|                   Oracle 19c   Event-     Nightly       ES query      |
|                   (ACID)       Driven     Full Sync     result        |
|                                Sync                                   |
|                                (1-5s)    (catches       (1s refresh   |
|                                           drift)        interval)     |
|                                                                        |
|  WRITES always      READS from ES may be 1-5 seconds behind Oracle    |
|  go to Oracle       This is ACCEPTABLE for analytical dashboards      |
+-----------------------------------------------------------------------+
```

### 4.2 Consistency Rules

| Rule | Description | Enforcement |
|------|-------------|-------------|
| **R1: Oracle is truth** | All writes go to Oracle. ES is never written to directly by business logic. | Module M33 is the only ES writer. |
| **R2: Transactional reads from Oracle** | P/M plan editing, roadmap editing, simulation editing all read from Oracle (Mendix ORM). | Mendix data views always bound to domain model. |
| **R3: Analytical reads from ES** | Dashboards, charts, world map, reports, analysis views read from ES. | ECharts/Gantt widgets call ES via M33 REST endpoints. |
| **R4: Acceptable lag** | 1-5 second delay between Oracle commit and ES visibility is acceptable for analytics. | Users see "Data as of: [timestamp]" on dashboards. |
| **R5: Nightly reconciliation** | Full sync catches any missed events, corrects drift. | Scheduled at 02:00 AM, count verification. |
| **R6: Immutable versioning** | Once a version is approved, its data in both Oracle and ES is immutable (never updated, only new versions created). | Version state machine enforces this. |

### 4.3 Handling Inconsistency Scenarios

| Scenario | Detection | Resolution |
|----------|-----------|------------|
| **Sync queue item fails** | retry_count > 0 in SyncQueue | Auto-retry up to 3 times with exponential backoff (30s, 60s, 120s). After 3 failures: mark FAILED, alert admin. |
| **ES cluster temporarily unavailable** | ES connection error during sync | Queue items remain PENDING. Sync worker retries next cycle (30s). Items accumulate safely in Oracle queue. |
| **Oracle count != ES count after nightly sync** | Count verification step | Log warning. If delta > 0.1%: trigger automatic reindex for affected index. Alert admin. |
| **Master data changed (N-PLM import)** | N-PLM sync event | Trigger full reindex of ALL analytical indices (master data denormalized into every document). |
| **Stale data visible on dashboard** | User reports or timestamp check | Dashboard shows "Last synced: [timestamp]". Admin can trigger on-demand reindex. |
| **ES index corrupted** | Health check or query errors | Zero-downtime reindex via alias swap (Section 3.5). No data loss because Oracle is the source. |

### 4.4 Sync Audit Log

Every sync operation is logged for traceability:

```
SyncAuditLog
  |-- audit_id (PK)
  |-- sync_type              // EVENT_DRIVEN, SCHEDULED_FULL, ON_DEMAND
  |-- target_index           // iris-resource-allocation, iris-headcount, etc.
  |-- started_at
  |-- completed_at
  |-- duration_ms
  |-- documents_processed    // Number of docs indexed
  |-- documents_failed       // Number of failures
  |-- oracle_count           // Count in Oracle for verification
  |-- es_count               // Count in ES after sync
  |-- count_match            // Boolean: oracle_count == es_count
  |-- status                 // SUCCESS, PARTIAL_FAILURE, FAILED
  |-- error_details          // NULL or error summary
  |-- triggered_by           // SYSTEM, ADMIN (user_id)
```

---

## 5. Failure Handling & Recovery

### 5.1 Retry Strategy

```
SyncQueue Item Lifecycle:

  PENDING -----> PROCESSING -----> COMPLETED
                     |
                     | (ES error)
                     v
                   RETRY (retry_count += 1)
                     |
                     | (wait: 30s * 2^retry_count)
                     v
                  PROCESSING -----> COMPLETED
                     |
                     | (retry_count >= 3)
                     v
                   FAILED
                     |
                     v
              Alert Admin + will be caught by nightly full sync
```

### 5.2 Circuit Breaker Pattern

Prevent overwhelming a struggling ES cluster:

```
Sync Worker Logic:

  consecutive_failures = 0

  FOR EACH batch in SyncQueue:
      result = call_es_bulk(batch)

      IF result.success:
          consecutive_failures = 0
          mark_completed(batch)

      ELSE:
          consecutive_failures += 1
          mark_retry(batch)

          IF consecutive_failures >= 5:
              LOG "Circuit breaker OPEN - pausing sync for 5 minutes"
              SLEEP 300 seconds
              consecutive_failures = 0
              // Nightly sync will catch up anything missed
```

### 5.3 Recovery Procedures

| Failure | Impact | Recovery |
|---------|--------|----------|
| **ES node failure** | Reduced cluster capacity, replicas serve reads | ES self-heals via shard reallocation. Sync queue items retry. No action needed if replicas exist. |
| **ES cluster down** | No analytical dashboards | Queue accumulates in Oracle. When ES recovers, sync worker drains queue. If downtime > 24h: trigger full reindex. |
| **Oracle downtime** | No writes, no new sync events | ES continues serving stale-but-valid analytical data. Sync resumes when Oracle recovers. |
| **Mendix runtime restart** | Sync worker temporarily stopped | Scheduled event auto-restarts with Mendix. Pending queue items processed on restart. |
| **Corrupt ES index** | Query errors on specific index | Zero-downtime reindex via alias swap. No data loss. |
| **SyncQueue table full** | Queue insert fails | Monitor queue depth. Alert if PENDING count > 10,000. Increase nightly sync frequency or add parallel workers. |

---

## 6. Data Consistency Verification

### 6.1 Automated Health Checks

Run as part of nightly sync and available on-demand from admin panel:

```
Health Check Suite:

CHECK 1: Document Count Match
  FOR EACH index IN [iris-resource-allocation, iris-headcount,
                     iris-simulation-result, iris-analysis-report]:
      oracle_count = SELECT COUNT(*) FROM {source_table}
                     WHERE version_status IN ('Active', 'Approved', 'Draft')
      es_count     = GET {index}/_count

      IF abs(oracle_count - es_count) / oracle_count > 0.001:
          ALERT "Count mismatch on {index}: Oracle={oracle_count}, ES={es_count}"
          ACTION: trigger targeted reindex

CHECK 2: Sample Verification (Spot Check)
  SELECT 100 random allocation records from Oracle
  FOR EACH record:
      es_doc = GET iris-resource-allocation/_doc/{allocation_id}
      COMPARE record.allocated_hours vs es_doc.allocation.hours
      COMPARE record.allocated_pct vs es_doc.allocation.percentage

      IF mismatch:
          LOG "Data mismatch: {allocation_id}"
          INCREMENT mismatch_counter

  IF mismatch_counter > 5:
      ALERT "Significant data drift detected"
      ACTION: trigger full reindex

CHECK 3: SyncQueue Health
  pending_count = SELECT COUNT(*) FROM SyncQueue WHERE status = 'PENDING'
  failed_count  = SELECT COUNT(*) FROM SyncQueue WHERE status = 'FAILED'
  oldest_pending = SELECT MIN(created_at) FROM SyncQueue WHERE status = 'PENDING'

  IF pending_count > 5000: ALERT "Sync queue backlog"
  IF failed_count > 100:   ALERT "Excessive sync failures"
  IF oldest_pending < SYSDATE - INTERVAL '1' HOUR: ALERT "Stale pending items"

CHECK 4: Freshness Check
  latest_oracle_update = SELECT MAX(modified_date) FROM RoadmapProjectAllocation
  latest_es_update     = GET iris-resource-allocation/_search
                         { "size":1, "sort":[{"timestamp":"desc"}] }

  lag = latest_oracle_update - latest_es_update.timestamp
  IF lag > 5 minutes: ALERT "Sync lag exceeds threshold: {lag}"
```

### 6.2 Admin Dashboard (Mendix)

| Panel | Data Source | Shows |
|-------|-----------|-------|
| Sync Queue Status | Oracle: SyncQueue | PENDING / PROCESSING / FAILED counts, queue depth chart |
| Last Sync Summary | Oracle: SyncAuditLog | Last event-driven, last nightly, last on-demand results |
| Count Comparison | Oracle + ES | Side-by-side row counts per index with match/mismatch indicator |
| Sync Lag | Oracle + ES | Current delay between latest Oracle change and latest ES document |
| Failed Items | Oracle: SyncQueue | List of FAILED items with error messages, manual retry button |

---

## 7. Performance Optimization

### 7.1 Oracle 19c Performance

| Area | Strategy | Detail |
|------|----------|--------|
| **Denormalization query** | Indexed FK columns | Create indexes on all FK columns used in the 12-table join (employee.team_id, team.dept_id, dept.bu_id, etc.) |
| **Denormalization query** | Materialized view (optional) | For nightly full sync, consider an Oracle Materialized View that pre-joins allocation + dimension tables. Refresh on-demand before sync. |
| **SyncQueue throughput** | Index on (status, created_at) | Sync worker queries `WHERE status = 'PENDING' ORDER BY created_at` — this must be fast |
| **SyncQueue cleanup** | Partition by month | Partition SyncQueue table by `created_at` month. Drop old partitions instead of DELETE (faster, no undo generation) |
| **Connection pooling** | Mendix built-in | Mendix manages Oracle connection pool. Ensure pool size >= sync worker concurrency + normal user load. Recommend: 50-100 connections. |
| **Batch reads** | Fetch size 1000 | When reading allocation records for sync, use JDBC fetch size of 1000 to reduce round trips |

#### Recommended Oracle Indexes

```sql
-- FK indexes for denormalization query performance
CREATE INDEX idx_rpa_version_id ON RoadmapProjectAllocation(version_id);
CREATE INDEX idx_rpa_employee_id ON RoadmapProjectAllocation(employee_id);
CREATE INDEX idx_rpa_project_id ON RoadmapProjectAllocation(project_id);
CREATE INDEX idx_employee_team_id ON Employee(team_id);
CREATE INDEX idx_employee_site_id ON Employee(site_id);
CREATE INDEX idx_team_dept_id ON Team(dept_id);
CREATE INDEX idx_dept_bu_id ON Department(bu_id);
CREATE INDEX idx_bu_division_id ON BusinessUnit(division_id);
CREATE INDEX idx_project_type_id ON Project(project_type_id);
CREATE INDEX idx_project_family_id ON Project(product_family_id);
CREATE INDEX idx_site_country_id ON Site(country_id);

-- SyncQueue performance
CREATE INDEX idx_syncqueue_status_created ON SyncQueue(status, created_at);
CREATE INDEX idx_syncqueue_entity ON SyncQueue(entity_type, entity_id);

-- Version lookup
CREATE INDEX idx_roadmap_version_status ON RoadmapVersion(version_type, roadmap_id);
CREATE INDEX idx_pm_version_status ON PMPlanVersion(version_type, pm_plan_id);
```

#### Optional: Materialized View for Full Sync

```sql
CREATE MATERIALIZED VIEW mv_resource_allocation_flat
BUILD DEFERRED
REFRESH ON DEMAND
AS
SELECT
    rpa.allocation_id,
    rv.version_id, rv.version_number, rv.version_type,
    e.employee_id, e.name AS employee_name, e.job_grade, e.skill_category,
    t.team_name, d.dept_name, bu.bu_name, div.division_name,
    p.project_id, p.project_name, p.project_code,
    pt.type_name AS project_type, pf.family_name AS product_family,
    p.technology_node, p.priority, p.status AS project_status,
    s.site_name, c.country_name, c.region, s.latitude, s.longitude,
    rpa.period, rpa.allocated_percentage, rpa.allocated_hours,
    rpa.role_in_project
FROM RoadmapProjectAllocation rpa
    JOIN RoadmapVersion rv  ON rpa.version_id = rv.version_id
    JOIN Employee e         ON rpa.employee_id = e.employee_id
    JOIN Team t             ON e.team_id = t.team_id
    JOIN Department d       ON t.dept_id = d.dept_id
    JOIN BusinessUnit bu    ON d.bu_id = bu.bu_id
    JOIN Division div       ON bu.division_id = div.division_id
    JOIN Project p          ON rpa.project_id = p.project_id
    JOIN ProjectType pt     ON p.project_type_id = pt.type_id
    JOIN ProductFamily pf   ON p.product_family_id = pf.family_id
    JOIN Site s             ON e.site_id = s.site_id
    JOIN Country c          ON s.country_id = c.country_id;

-- Refresh before nightly full sync:
EXEC DBMS_MVIEW.REFRESH('mv_resource_allocation_flat', 'C');
```

### 7.2 Elasticsearch Performance

| Area | Strategy | Detail |
|------|----------|--------|
| **Bulk indexing** | Batch size 500-1000 | ES Bulk API processes 500-1000 docs per request. Larger batches reduce HTTP overhead. |
| **Refresh interval** | 1s (default) for hot, 30s for warm | Hot indices: 1s refresh for near real-time visibility. Warm indices: 30s saves resources. During full reindex: set to -1 (disable) then refresh manually after. |
| **Index settings during reindex** | Disable replicas temporarily | Set `number_of_replicas: 0` during full reindex, restore to 1 after. Cuts indexing time ~50%. |
| **Shard sizing** | Target 10-30GB per shard | iris-resource-allocation: 3 primary shards handles up to ~90GB of allocation data |
| **Field types** | `keyword` for all dimensions | Never use `text` type for dimension fields used in aggregations. `keyword` is ~10x faster for terms aggs. |
| **Doc values** | Enabled by default | Oracle numeric/keyword fields map to ES `keyword`/`float` which use columnar doc_values by default — optimized for aggregations. |
| **Query caching** | Request cache enabled | Repeated dashboard queries (same filters, same time range) served from cache. Cache invalidated on index refresh. |

#### Bulk Indexing During Full Sync

```
Optimization for nightly full sync:

1. Set target index replica count to 0
   PUT iris-resource-allocation/_settings
   { "index": { "number_of_replicas": 0, "refresh_interval": "-1" } }

2. Bulk index all documents (500 per batch)

3. Force refresh
   POST iris-resource-allocation/_refresh

4. Restore replicas
   PUT iris-resource-allocation/_settings
   { "index": { "number_of_replicas": 1, "refresh_interval": "1s" } }

Result: Full sync of 500K documents: ~3-5 minutes vs ~15-20 minutes without optimization
```

### 7.3 End-to-End Latency Budget

```
User saves P/M plan
  |
  | Oracle COMMIT:                  ~50ms
  | After-Commit event fire:        ~10ms
  | SyncQueue INSERT:               ~5ms
  |                                 ------
  | Total write path:               ~65ms  (user sees "Saved" immediately)
  |
  | Sync Worker picks up:           0-30s  (polls every 30s)
  | Denormalization query:          ~100-500ms (12-table join)
  | Build JSON document:            ~5ms
  | ES Bulk API call:               ~50-200ms
  | ES refresh (auto, 1s):          0-1s
  |                                 ------
  | Total sync latency:             1-32s  (ES reflects change)
  |
  | Dashboard refresh:              User triggers or auto-refresh (configurable)

Typical end-to-end: User saves -> Dashboard shows new data in 2-10 seconds
```

---

## 8. Operational Procedures

### 8.1 Monitoring Checklist

| Metric | Threshold | Alert |
|--------|-----------|-------|
| SyncQueue PENDING count | > 5,000 | Warning: queue backlog |
| SyncQueue FAILED count | > 100 | Error: persistent failures |
| Oldest PENDING item age | > 60 minutes | Error: sync stalled |
| Nightly sync duration | > 30 minutes | Warning: performance degradation |
| Oracle-ES count delta | > 0.1% | Warning: data drift |
| ES cluster health | Yellow or Red | Error: cluster degraded |
| Sync lag (freshness) | > 5 minutes | Warning: stale data |

### 8.2 Runbook: Manual Recovery

```
SCENARIO: ES cluster was down for 2 hours, now recovered

1. Check SyncQueue:
   SELECT status, COUNT(*) FROM SyncQueue GROUP BY status;
   -- Expect: large PENDING count, some RETRY

2. Verify ES cluster health:
   GET _cluster/health
   -- Wait for status: "green"

3. Let sync worker drain the queue naturally (watch PENDING count decrease)

4. If PENDING count not decreasing:
   -- Check Mendix logs for sync worker errors
   -- Restart Mendix scheduled event if needed

5. After queue is drained, run health checks:
   -- Trigger count verification from Admin Dashboard
   -- If counts mismatch: trigger on-demand reindex

6. If data is critically stale and queue is too large:
   -- Skip queue processing
   -- Trigger full on-demand reindex (Section 3.5)
   -- Clear old queue items after reindex completes
```

---

## Sources

- [Solution Architecture - Data Layer](./01-solution-architecture.md)
- [Data Modeling - ES Indices & Sync Architecture](./02-data-modeling.md)
- [Research: Elasticsearch Implementation](../../02_research/resource-planning-mda/06-elasticsearch-implementation.md)
- [Research: Integration Architecture](../../02_research/ai-resource-planning/07-integration-architecture.md)
