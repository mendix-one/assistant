# What-If Scenario Analysis with Multi-Dimensional Analysis

## 1. Overview

What-if scenario analysis allows DS Division planners to model the impact of **hypothetical changes** — demand shifts, equipment failures, new product introductions, technology transitions — on resource plans **before committing** to decisions.

Multi-dimensional analysis powers what-if by enabling planners to **swap one dimension value** (e.g., Scenario = "Optimistic") and instantly see how all measures change across every other dimension.

## 2. Scenario Dimension Design (Snowflaked)

```
DimScenario
  |-- scenario_id (PK)
  |-- scenario_name
  |-- scenario_type_id (FK)
  |     |-- DimScenarioType
  |           |-- type_name (Baseline, Demand Shock, Supply Disruption,
  |           |               Tech Acceleration, Cost Optimization)
  |           |-- category (Strategic / Tactical / Operational)
  |-- created_by
  |-- created_date
  |-- status (Draft, Active, Approved, Archived)
  |-- parent_scenario_id (FK, self-reference for variant tracking)
  |-- description
  |-- assumptions_json (structured assumptions)
```

### Scenario Versioning

Scenarios form a tree — variants branch from a parent baseline:

```
Baseline 2026
  |-- Optimistic (Demand +15%)
  |     |-- Optimistic + New Fab (add Pyeongtaek P4)
  |     |-- Optimistic + Delayed EUV (3 month slip)
  |
  |-- Conservative (Demand -10%)
  |     |-- Conservative + Cost Cut (reduce shift to 2-shift)
  |
  |-- Disruption (Supply chain break)
        |-- Disruption + Alt Supplier
        |-- Disruption + Inventory Buffer
```

## 3. What-If Scenario Types for Device Solutions

### 3.1 Demand Scenarios

| Scenario | Change | Affected Dimensions |
|----------|--------|---------------------|
| AI/HPC boom | HBM demand +40% | Product (HBM), Customer (hyperscaler), Time (Q3+) |
| Mobile slowdown | Exynos demand -20% | Product (Mobile SoC), Customer, Revenue |
| New customer win | Foundry order +30K wafers/month | Customer, Facility, Capacity |
| Seasonal spike | Q4 consumer demand +25% | Time, Product (consumer devices) |

### 3.2 Supply / Capacity Scenarios

| Scenario | Change | Affected Dimensions |
|----------|--------|---------------------|
| New fab online | Pyeongtaek P4 live Q3 | Facility, Capacity, CapEx |
| Equipment delay | EUV tool delivery +3 months | Equipment, Capacity, Time |
| Yield improvement | 3nm yield 78% -> 85% | Product, Capacity (effective), Cost |
| Maintenance extension | Planned shutdown +2 weeks | Facility, Time, Capacity |

### 3.3 Resource / Cost Scenarios

| Scenario | Change | Affected Dimensions |
|----------|--------|---------------------|
| Hiring freeze | No new engineers in Q2-Q3 | Resource, Team, Utilization |
| Shift optimization | Move from 3-shift to 2-shift on legacy nodes | Resource, Time, Cost |
| Outsource packaging | External OSAT for mature products | Facility, Cost, Lead time |
| Skill retraining | 50 engineers retrained from 5nm to 3nm | Resource, Product, Qualification |

## 4. How OLAP Enables What-If

### 4.1 Scenario Comparison (Dice Operation)

Select two or more scenarios and compare measures side-by-side:

```
Measure: Total Wafer Capacity (K wafers/month)

                    Baseline    Optimistic    Conservative    Disruption
Hwaseong              320          320            300            250
Pyeongtaek            180          240            160            180
Austin                 90           90             80             70
Xi'an                 100          100            100             60
─────────────────────────────────────────────────────────────────────
TOTAL                 690          750            640            560

Delta vs Baseline:     --          +60            -50           -130
```

**OLAP Operation:** Dice(Scenario IN [Baseline, Optimistic, Conservative, Disruption]) + Roll-up(Facility -> Site) + Measure(SUM(effective_capacity))

### 4.2 Impact Propagation (Drill-Down)

A change in one dimension ripples through the cube. Trace the full impact:

```
SCENARIO: "EUV Tool Delayed 3 Months"

Impact Chain:
1. Equipment Dimension
   - ASML NXE:3800 #8 delivery: Q2 -> Q3

2. Capacity Dimension (direct impact)
   - Pyeongtaek Line 18 capacity: -15K wafers/month for Q2

3. Product Dimension (cascading)
   - HBM4 production: -12K wafers (primary product on Line 18)
   - DDR5 production: -3K wafers (secondary)

4. Customer/Demand Dimension (cascading)
   - Customer A committed delivery: at risk (-8K units)
   - Internal Memory BU forecast: shortfall for Q3 launch

5. Financial Dimension (cascading)
   - Revenue impact: -$45M in Q2
   - Penalty risk: $5M (customer SLA)

6. Mitigation Options (new scenarios branch):
   a. Overtime on existing tools (+5K wafers, +$2M cost)
   b. Redirect from Austin fab (+8K wafers, +$3M logistics)
   c. Customer negotiation (defer 4K units to Q3)
```

### 4.3 Sensitivity Analysis (Slice + Iterate)

Vary one parameter while holding others constant:

```
Variable: 3nm Yield (range 70% to 90%)
Fixed: Demand = Baseline, Facility = Hwaseong, Product = HBM4

Yield    Effective Capacity    Meets Demand?    Cost/Wafer
70%        210K                  No (-40K)       $4,800
75%        225K                  No (-25K)       $4,200
78%        234K                  No (-16K)       $3,900
80%        240K                  No (-10K)       $3,700
85%        255K                  Yes (+5K)       $3,300     <-- target
90%        270K                  Yes (+20K)      $3,000
```

### 4.4 Scenario Scoring Matrix

Evaluate scenarios across multiple KPIs simultaneously:

```
                Revenue   Utilization   Risk    CapEx    Customer     Weighted
Scenario        ($B)      (%)           Score   ($B)     Satisfaction Score
─────────────────────────────────────────────────────────────────────────────
Baseline        42.0       85%          Low      3.2       82%         7.2
Optimistic      48.5       92%          Med      5.1       88%         8.1
Conservative    38.0       78%          Low      2.0       75%         6.5
Disruption      35.0       72%          High     3.2       60%         4.8

Weights:        30%        15%          20%      15%       20%         100%
```

## 5. Implementation Pattern

### 5.1 Scenario Data Architecture

```
Shared Dimensions (same across scenarios)     Scenario-Specific Facts
+------------------+                          +---------------------------+
| DimTime          |                          | FactCapacity              |
| DimFacility      |                          |   scenario_id = 1 (Base)  |
| DimProduct       |  <-- shared keys -->     |   scenario_id = 2 (Opt)   |
| DimResource      |                          |   scenario_id = 3 (Cons)  |
| DimEquipment     |                          +---------------------------+
+------------------+                          | FactDemand                |
                                              |   scenario_id = 1,2,3...  |
                                              +---------------------------+
                                              | FactResourceAllocation    |
                                              |   scenario_id = 1,2,3...  |
                                              +---------------------------+
```

**Key principle:** Dimensions are shared. Only fact table measures vary per scenario. This keeps storage efficient and enables direct comparison via the scenario dimension.

### 5.2 Scenario Creation Workflow

```
1. Clone baseline fact data    -->  New scenario_id assigned
2. Apply parameter changes     -->  Modify specific measures
3. Run constraint solver       -->  Propagate impacts
4. Load results to fact tables -->  Available for MDA
5. Compare via OLAP            -->  Dashboard / report
6. Approve or iterate          -->  Status = Approved / Archive
```

## 6. Decision Framework

| Decision Type | Scenarios to Compare | Key Measures | Time Horizon |
|---------------|---------------------|--------------|--------------|
| CapEx approval | Baseline vs Growth + New Fab | ROI, capacity gap, payback | 2-5 years |
| Product launch timing | Multiple demand timelines | Revenue, resource readiness | 6-18 months |
| Emergency response | Disruption variants | Recovery time, cost, customer impact | Weeks |
| Technology transition | Migration variants | Yield ramp, capacity transfer | 1-3 years |
| Workforce planning | Hiring vs retraining | Cost, timeline, skill coverage | 6-12 months |

---

## Sources

- [What is OLAP? - AWS](https://aws.amazon.com/what-is/olap/)
- [OLAP Modeling and Dashboards - Wolters Kluwer](https://www.wolterskluwer.com/en/solutions/cch-tagetik/glossary/olap-modeling-and-dashboards)
- [OLAP Cubes: Multi-Dimensional Analysis - FasterCapital](https://fastercapital.com/content/Business-intelligence--OLAP-Cubes--OLAP-Cubes--Multi-Dimensional-Analysis-for-Better-Business-Insights.html)
- [Understanding OLAP Cubes - Keboola](https://www.keboola.com/blog/olap-cubes)
