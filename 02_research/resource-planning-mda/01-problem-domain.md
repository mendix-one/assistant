# Problem Domain & Dimensional Model

## 1. Business Context

Samsung Electronics Device Solutions (DS) Division operates across three business units:

| Business Unit | Products | Resource Planning Challenge |
|---------------|----------|---------------------------|
| **Memory** | DRAM, NAND Flash, HBM | Fab capacity allocation, technology node migration |
| **System LSI** | Mobile SoC (Exynos), Image Sensors, DDI | Design team scheduling, tape-out planning |
| **Foundry** | Contract manufacturing | Multi-customer scheduling, yield optimization |

Resource planning must coordinate **equipment, engineers, materials, cleanroom capacity, and test infrastructure** across product lines with long lead times (6-18 months) and high capital intensity.

## 2. Why Multi-Dimensional Analysis?

Resource planning in semiconductor device solutions is inherently multi-dimensional:

```
Who builds it?    --> Resource Dimension (engineers, teams, equipment)
What is built?    --> Product Dimension (chip type, technology node, package)
When is it built? --> Time Dimension (quarter, month, week, shift)
Where is it built?--> Facility Dimension (fab, line, cleanroom)
How much?         --> Capacity Dimension (wafer starts, throughput, yield)
For whom?         --> Customer/Program Dimension (internal, foundry customer)
```

Traditional 2D reports (e.g., "engineers vs. weeks") cannot capture the interplay between facility constraints, product mix, technology transitions, and customer commitments simultaneously.

## 3. Snowflake Schema Design

### 3.1 Why Snowflake Over Star?

For device solutions resource planning, **snowflake schema** is preferred because:

1. **Deep hierarchies** - Equipment has type > class > specific tool chains; Products have division > family > SKU > revision
2. **Shared sub-dimensions** - "Technology Node" is shared across Product, Equipment, and Facility dimensions
3. **Data integrity** - Normalized dimension tables ensure a single source of truth for equipment specs, skill matrices, and facility capabilities
4. **Evolving structure** - New technology nodes, product families, and facilities are added frequently; normalization makes this easier

### 3.2 Fact Tables

#### FactResourceAllocation (Grain: resource + time slot + product + facility)

| Column | Type | Description |
|--------|------|-------------|
| allocation_id | PK | Unique allocation identifier |
| resource_id | FK | Engineer, equipment, or tool |
| product_id | FK | Chip/device being worked on |
| facility_id | FK | Fab or lab location |
| time_id | FK | Time period |
| scenario_id | FK | Planning scenario |
| allocated_hours | Measure | Hours allocated |
| available_hours | Measure | Hours available |
| utilization_pct | Measure | Utilization percentage |
| cost_usd | Measure | Cost of allocation |
| priority | Measure | Scheduling priority (1-5) |

#### FactCapacity (Grain: facility + equipment class + time period)

| Column | Type | Description |
|--------|------|-------------|
| capacity_id | PK | Unique record |
| facility_id | FK | Fab or line |
| equipment_class_id | FK | Equipment category |
| time_id | FK | Time period |
| scenario_id | FK | Planning scenario |
| max_wafer_starts | Measure | Maximum wafer starts per period |
| planned_wafer_starts | Measure | Planned wafer starts |
| actual_wafer_starts | Measure | Actual wafer starts |
| downtime_hours | Measure | Maintenance/downtime hours |
| yield_pct | Measure | Process yield percentage |

#### FactDemand (Grain: product + customer + time period)

| Column | Type | Description |
|--------|------|-------------|
| demand_id | PK | Unique record |
| product_id | FK | Device/chip |
| customer_id | FK | Internal BU or external customer |
| time_id | FK | Time period |
| scenario_id | FK | Planning scenario |
| forecast_qty | Measure | Forecasted demand (units) |
| committed_qty | Measure | Committed/contracted quantity |
| actual_qty | Measure | Actual demand realized |
| revenue_usd | Measure | Associated revenue |
| confidence_pct | Measure | Forecast confidence level |

### 3.3 Dimension Tables (Snowflaked)

```
DimResource
  |-- DimResourceType (Engineer, Equipment, Tool, Cleanroom)
  |     |-- DimSkillCategory (Design, Process, Test, Package)
  |-- DimTeam
  |     |-- DimDepartment
  |           |-- DimBusinessUnit (Memory, System LSI, Foundry)

DimProduct
  |-- DimProductFamily (DRAM, NAND, Exynos, ISOCELL, etc.)
  |     |-- DimProductLine
  |-- DimTechnologyNode (3nm, 5nm, 7nm, etc.)
  |-- DimPackageType (FOWLP, FC-BGA, etc.)

DimFacility
  |-- DimSite (Hwaseong, Pyeongtaek, Austin, Xi'an)
  |     |-- DimCountry
  |-- DimFabLine
  |-- DimCleanroomClass

DimTime
  |-- DimQuarter
  |     |-- DimYear
  |-- DimMonth
  |-- DimWeek
  |-- DimShift (Day, Night, Maintenance)

DimCustomer
  |-- DimCustomerTier (Strategic, Standard, Internal)
  |-- DimRegion

DimScenario
  |-- DimScenarioType (Baseline, Optimistic, Pessimistic, Stress)
```

### 3.4 Entity Relationship (Snowflake)

```
                          DimYear
                            |
                        DimQuarter
                            |
DimCountry---DimSite---DimFacility---DimTime---DimMonth
                |                       |
            DimFabLine              DimWeek---DimShift
                |
            DimCleanroom
                |                         DimScenarioType
                |                              |
DimBusinessUnit-+-DimDepartment          DimScenario
                |       |                     |
            DimTeam  DimSkillCat    +---------+----------+
                |       |          |          |          |
            DimResourceType    FactResource  FactCap  FactDemand
                |              Allocation              |
            DimResource            |              DimCustomer
                |                  |                   |
                +-------+----+----+              DimCustomerTier
                        |    |
                   DimProduct |
                     |    |   |
            DimProductFamily  DimTechNode
                     |
                DimProductLine    DimPackageType
```

## 4. Key Analytical Dimensions Mapping

| Planning Question | Dimensions Involved | OLAP Operation |
|-------------------|---------------------|----------------|
| "How are engineers distributed across fabs?" | Resource x Facility | Pivot |
| "What is Q2 utilization by technology node?" | Time x TechNode x Capacity | Slice + Roll-up |
| "Drill from yearly to weekly capacity for 3nm line" | Time hierarchy | Drill-down |
| "Compare baseline vs optimistic scenario" | Scenario | Dice |
| "Total demand across all DRAM products" | Product hierarchy | Roll-up |
| "Which teams are over-allocated next month?" | Resource x Time x Allocation | Slice + Filter |

---

## Sources

- [Device Solutions - Samsung US](https://www.samsung.com/us/about-us/our-business/device-solutions/)
- [Samsung DS Division - Samsung Semiconductor Newsroom](https://news.samsungsemiconductor.com/global/tag/samsung-electronics-ds-division/)
- [Enterprise-Wide Semiconductor Manufacturing Resource Planning](https://www.researchgate.net/publication/3284247_Enterprise-Wide_Semiconductor_Manufacturing_Resource_Planning)
- [Coordinating Strategic Capacity Planning in the Semiconductor Industry](https://ideas.repec.org/a/inm/oropre/v51y2003i6p839-849.html)
