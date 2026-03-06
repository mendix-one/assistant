# Capacity Planning with Multi-Dimensional Analysis

## 1. Overview

Capacity planning in Samsung DS Division determines **how much can be produced, when, and where** — balancing wafer starts, equipment throughput, workforce availability, and technology transitions across fabs worldwide.

Multi-dimensional analysis enables planners to view capacity from any angle: by facility, technology node, product family, time horizon, or planning scenario — and drill into bottlenecks at any level.

## 2. Capacity Planning Dimensions

```
                    Capacity Cube
                   /      |       \
                  /       |        \
           Facility    Product    Time
           /    \       /   \      |
        Site   Line  Family Node  Quarter
         |      |      |     |      |
       Country Cleanroom SKU  Process Month-->Week-->Shift
```

### Key Dimension Hierarchies

| Dimension | Hierarchy (top -> bottom) | Example |
|-----------|--------------------------|---------|
| Facility | Country > Site > Fab > Line > Bay | Korea > Pyeongtaek > P3 > Line17 > Bay-A |
| Product | BU > Family > SKU > Revision | Memory > DRAM > DDR5-16Gb > Rev.C |
| Technology | Generation > Node > Process Variant | EUV > 3nm > 3GAP |
| Time | Year > Quarter > Month > Week > Shift | 2026 > Q2 > Apr > W14 > DayShift |
| Equipment | Class > Type > Specific Tool | Litho > EUV Scanner > ASML NXE:3800 #3 |

## 3. Capacity Fact Tables

### FactCapacityPlan (Grain: facility-line + equipment-class + product-family + week)

| Column | Type | Description |
|--------|------|-------------|
| facility_line_id | FK | Specific fab line |
| equipment_class_id | FK | Equipment category |
| product_family_id | FK | Product being produced |
| time_id | FK | Week granularity |
| scenario_id | FK | Planning scenario |
| max_capacity_wpm | Measure | Max wafer starts per month (theoretical) |
| effective_capacity_wpm | Measure | Effective capacity (after maintenance, yield loss) |
| planned_load_wpm | Measure | Planned wafer starts |
| actual_load_wpm | Measure | Actual wafer starts (for past periods) |
| utilization_pct | Measure | planned_load / effective_capacity * 100 |
| bottleneck_flag | Measure | 1 = this is the bottleneck step |
| capex_required_usd | Measure | Capital expenditure if expansion needed |

### FactEquipmentAvailability (Grain: equipment + day)

| Column | Type | Description |
|--------|------|-------------|
| equipment_id | FK | Specific tool |
| time_id | FK | Day |
| total_hours | Measure | 24 (or shift hours) |
| production_hours | Measure | Hours in production |
| maintenance_hours | Measure | Planned maintenance |
| unplanned_downtime_hours | Measure | Unexpected downtime |
| setup_changeover_hours | Measure | Product changeover time |
| availability_pct | Measure | production / total * 100 |

## 4. Multi-Dimensional Capacity Analysis

### 4.1 Utilization Heatmap (Pivot: Facility x Time)

Roll up to site level, measure = AVG(utilization_pct):

```
                Q1-2026   Q2-2026   Q3-2026   Q4-2026
Hwaseong          87%       91%       94%       89%
Pyeongtaek        72%       78%       85%       92%    <-- ramp-up
Austin            65%       68%       70%       72%
Xi'an             88%       90%       88%       85%

Color scale: <70% = underutilized | 70-85% = healthy | 85-95% = high | >95% = critical
```

### 4.2 Bottleneck Analysis (Drill-Down)

Start from high utilization at site level, drill into the root cause:

```
Hwaseong (91% utilization, Q2-2026)
  |
  +-- Fab S8 (95%)        <-- hotspot
  |     |
  |     +-- Line 15, 3nm EUV (98%)     <-- near-critical
  |     |     |
  |     |     +-- EUV Litho step (99.2%)  <-- BOTTLENECK
  |     |     |     Tool: ASML NXE:3800 #2 (100% booked)
  |     |     |     Tool: ASML NXE:3800 #5 (98% booked)
  |     |     |
  |     |     +-- Etch step (85%)          <-- OK
  |     |
  |     +-- Line 16, 5nm (88%)            <-- OK
  |
  +-- Fab S6 (82%)                        <-- OK
```

### 4.3 Technology Node Migration View (Slice + Roll-Up)

Slice on dimension = Technology Node, roll up capacity across facilities:

```
              Total Capacity (K wafers/month)
              2026-Q1   Q2      Q3      Q4      2027-Q1
3nm GAP         45      52      58      65       72      <-- ramping up
5nm LPE        120     115     108     100       90      <-- declining
7nm LPP         80      75      70      65       60      <-- mature
14nm LPC        40      40      38      35       30      <-- legacy

Insight: 3nm ramp-up must absorb demand migrating from 5nm.
         Gap analysis: Is 3nm ramp fast enough?
```

### 4.4 Capacity vs. Demand Gap Analysis (Dice)

Cross-reference FactCapacity with FactDemand:

```
Product Family   Demand (K units)   Capacity (K units)   Gap      Action
DDR5 16Gb            850                 800             -50      Expand Line 17
HBM4                 200                 180             -20      Add test equipment
Exynos 2500          120                 150             +30      Available for foundry
ISOCELL HP3           95                 100              +5      Balanced
```

## 5. Capacity Planning Scenarios

Use the **DimScenario** dimension to maintain multiple plans:

| Scenario | Description | Key Assumptions |
|----------|-------------|-----------------|
| Baseline | Current approved plan | Current orders, no expansion |
| Growth | Market share gains | +15% demand, new fab line online Q3 |
| Conservative | Market slowdown | -10% demand, defer capex |
| Tech Transition | Accelerated 3nm migration | 3nm ramp +20%, 5nm wind-down faster |
| Supply Disruption | Equipment delivery delays | EUV tool delivery delayed 3 months |

All scenarios share the same dimensions but have different measure values in the fact tables, enabling side-by-side comparison via the **Dice** operation.

## 6. Key Capacity KPIs by Dimension

| KPI | Formula | Dimensions |
|-----|---------|------------|
| **Utilization Rate** | Planned Load / Effective Capacity | Facility x Time |
| **Throughput** | Actual Wafer Starts / Time Period | Facility x Product x Time |
| **OEE** | Availability x Performance x Quality | Equipment x Time |
| **Capacity Headroom** | Effective Capacity - Planned Load | Facility x Product x Time |
| **Ramp Readiness** | Current vs Target Capacity for New Node | TechNode x Facility x Time |
| **CapEx Efficiency** | Revenue per CapEx Dollar | Facility x Time x Scenario |

## 7. Roll-Up Path for Executive Reporting

```
Operational (Weekly)                 Tactical (Monthly)              Strategic (Quarterly)
+---------------------------+     +---------------------+     +-------------------+
| Equipment-level metrics   |     | Line-level summary  |     | Site/BU capacity  |
| - Tool availability       |---->| - Utilization by    |---->|   vs. demand      |
| - WIP by step             |     |   product family    |     | - CapEx decisions |
| - Constraint violations   |     | - Bottleneck report |     | - Node migration  |
+---------------------------+     +---------------------+     +-------------------+

OLAP: Drill-down (Strategic -> Tactical -> Operational)
      Roll-up   (Operational -> Tactical -> Strategic)
```

---

## Sources

- [Capacity Planning for Semiconductor Manufacturing](https://datacalculus.com/en/blog/semiconductor-manufacturing/applications-engineer/capacity-planning-for-semiconductor-manufacturing)
- [Capacity Planning in the Semiconductor Industry: Dual-Mode Procurement](https://pubsonline.informs.org/doi/10.1287/msom.1110.0361)
- [Robust Capacity Planning in Semiconductor Manufacturing](https://optimization-online.org/2001/10/379/)
- [OLAP Cubes for Better Business Insights](https://fastercapital.com/content/Business-intelligence--OLAP-Cubes--OLAP-Cubes--Multi-Dimensional-Analysis-for-Better-Business-Insights.html)
