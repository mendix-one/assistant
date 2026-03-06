# Constraint Scheduling with Multi-Dimensional Analysis

## 1. Overview

Constraint scheduling in semiconductor device solutions involves assigning **resources** (engineers, equipment, cleanrooms) to **tasks** (design, fabrication, test, packaging) while satisfying hard and soft constraints across multiple dimensions.

Multi-dimensional analysis enables planners to **visualize, analyze, and resolve** scheduling conflicts by examining constraints from multiple angles simultaneously.

## 2. Types of Constraints

### 2.1 Hard Constraints (Must Satisfy)

| Constraint | Dimension | Description |
|-----------|-----------|-------------|
| Equipment availability | Resource x Time | A lithography tool cannot run two lots simultaneously |
| Cleanroom class | Facility x Product | EUV processes require specific cleanroom specs |
| Skill qualification | Resource x Product | Only certified engineers can operate specific tools |
| Technology node compatibility | Equipment x Product | 3nm process requires specific equipment set |
| Regulatory compliance | Facility x Time | Maintenance windows, safety shutdowns |

### 2.2 Soft Constraints (Optimize)

| Constraint | Dimension | Description |
|-----------|-----------|-------------|
| Priority ordering | Product x Customer | Strategic customers get higher scheduling priority |
| Team continuity | Resource x Product x Time | Keep same team on a product across phases |
| Load balancing | Resource x Time | Distribute workload evenly across shifts |
| Minimize changeovers | Equipment x Product x Time | Reduce setup time between product runs |
| Cost optimization | Resource x Facility | Prefer lower-cost facilities when possible |

## 3. Dimensional Model for Constraint Scheduling

### 3.1 Constraint Fact Table

**FactScheduleConstraint** (Grain: one row per constraint evaluation per time slot)

| Column | Type | Description |
|--------|------|-------------|
| constraint_id | PK | Unique constraint record |
| resource_id | FK | Resource being scheduled |
| task_id | FK | Task to be performed |
| product_id | FK | Product/device |
| facility_id | FK | Location |
| time_id | FK | Time slot |
| constraint_type_id | FK | Type of constraint |
| is_satisfied | Measure | 1 = met, 0 = violated |
| violation_severity | Measure | 0 = none, 1-5 = severity |
| slack_hours | Measure | Available buffer time |
| reschedule_cost | Measure | Cost to reschedule if violated |

### 3.2 Supporting Dimension: DimConstraintType (Snowflaked)

```
DimConstraintType
  |-- constraint_id (PK)
  |-- constraint_name
  |-- constraint_category_id (FK)
  |     |-- DimConstraintCategory
  |           |-- category_name (Hard/Soft)
  |           |-- enforcement_level
  |-- priority_weight
  |-- applicable_resource_type_id (FK -> DimResourceType)
```

### 3.3 Task Dimension (Snowflaked)

```
DimTask
  |-- task_id (PK)
  |-- task_name
  |-- task_phase_id (FK)
  |     |-- DimTaskPhase
  |           |-- phase_name (Design, Fab, Test, Package, Qualification)
  |           |-- phase_order
  |-- estimated_hours
  |-- required_skill_id (FK -> DimSkillCategory)
  |-- predecessor_task_id (self-reference for dependency chains)
```

## 4. OLAP Operations for Constraint Analysis

### 4.1 Constraint Violation Heatmap (Pivot)

Analyze constraint violations across **Resource x Time** to find bottleneck periods.

```
              Week 10   Week 11   Week 12   Week 13
EUV Tool #1    0         2         3         0       <-- violations
EUV Tool #2    0         0         1         0
Etch Bay A     1         1         0         0
Test Lab 3     0         0         4         5       <-- critical
```

**OLAP:** Pivot on Resource (rows) x Time (columns), measure = SUM(violation_severity)

### 4.2 Conflict Resolution Drill-Down

When violations are detected, drill down to identify root causes:

```
Facility: Pyeongtaek Fab
  --> FabLine: Line 17 (3nm EUV)
    --> Equipment: EUV Scanner #3
      --> Time: Week 12, Night Shift
        --> Conflicting tasks:
            - Task A: HBM4 lot #2847 (Priority: 1, Customer: Internal)
            - Task B: Foundry Customer X lot #1052 (Priority: 2)
        --> Resolution: Defer Task B to Week 13 (slack = 48hrs)
```

### 4.3 Resource Qualification Matrix (Slice)

Slice on constraint_type = "Skill Qualification" to see which resources are qualified for which products:

```
                  DRAM    NAND    Exynos   ISOCELL   Foundry
Team Alpha         Y       Y       -        -         -
Team Beta          -       -       Y        Y         -
Team Gamma         -       -       -        -         Y
Equipment Set A    Y       Y       -        -         Y
Equipment Set B    -       -       Y        Y         Y
```

## 5. Scheduling Algorithm Integration

Multi-dimensional analysis works alongside scheduling algorithms, not as a replacement:

```
+-------------------+       +------------------+       +-------------------+
| Multi-Dimensional |       | Scheduling       |       | Multi-Dimensional |
| Analysis          |------>| Algorithm        |------>| Analysis          |
| (Understand)      |       | (Optimize)       |       | (Validate)        |
+-------------------+       +------------------+       +-------------------+

Phase 1: Analyze         Phase 2: Solve           Phase 3: Review
- Current utilization    - Constraint              - Visualize schedule
- Constraint violations    propagation            - Verify no violations
- Bottleneck detection   - Priority-based          - Scenario comparison
- Historical patterns      assignment             - Stakeholder reports
                         - Heuristic/LP/CP
                           solver
```

### Integration Points

| Phase | MDA Role | Data Flow |
|-------|----------|-----------|
| **Pre-scheduling** | Identify constraints, current state, historical patterns | MDA -> Solver input parameters |
| **During scheduling** | Real-time monitoring of solver progress | Solver metrics -> MDA dashboard |
| **Post-scheduling** | Validate results, compare scenarios, detect remaining conflicts | Solver output -> Fact tables -> MDA |

## 6. Practical Queries

### "Show all constraint violations for next quarter by severity"
- **Dimensions:** Time (Q2-2026), ConstraintType, Severity
- **Measure:** COUNT(violations), SUM(reschedule_cost)
- **Operation:** Slice(Time=Q2) + Roll-up(by severity)

### "Which equipment is double-booked in Hwaseong next month?"
- **Dimensions:** Facility (Hwaseong), Resource (Equipment), Time (April)
- **Measure:** COUNT(concurrent_tasks) WHERE concurrent > 1
- **Operation:** Slice(Facility=Hwaseong, Time=April) + Filter(overbooking)

### "Compare scheduling feasibility across baseline vs aggressive plan"
- **Dimensions:** Scenario (Baseline, Aggressive), Resource, Time
- **Measure:** SUM(violation_count), AVG(utilization_pct)
- **Operation:** Dice(Scenarios) + Pivot(Resource x Scenario)

---

## Sources

- [Production Planning for Semiconductor Manufacturing](https://www.sciencedirect.com/science/article/abs/pii/S0360835224005242)
- [Tool Capacity Planning in Semiconductor Manufacturing](https://www.academia.edu/18749191/Tool_capacity_planning_in_semiconductor_manufacturing)
- [Robust Capacity Planning in Semiconductor Manufacturing](https://ideas.repec.org/a/wly/navres/v52y2005i5p459-468.html)
