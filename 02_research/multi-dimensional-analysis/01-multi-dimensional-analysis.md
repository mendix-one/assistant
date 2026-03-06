# Multi-Dimensional Analysis

## Summary

Multi-dimensional analysis is a data analysis technique that allows users to view and analyze data across multiple dimensions (categories) simultaneously. It is the foundation of OLAP (Online Analytical Processing) systems used in business intelligence and data warehousing.

---

## 1. What is Multi-Dimensional Analysis?

Multi-dimensional analysis goes beyond traditional two-dimensional (rows/columns) analysis by representing information across multiple categories. It allows users to look at different dimensions of the same data interactively.

**Example:** Sales data can be analyzed across dimensions like:
- **Time** (year, quarter, month, day)
- **Geography** (country, region, city)
- **Product** (category, brand, SKU)
- **Customer** (segment, age group)

## 2. The OLAP Cube

An OLAP cube is a multi-dimensional array of data (also called a hypercube when dimensions > 3). It can be considered a generalization of a two-dimensional spreadsheet into multiple dimensions.

```
                  Time
                  /
                 /
    Product ----+---- Geography
                |
                |
              Measures
           (Revenue, Qty, etc.)
```

### Key Components

| Component | Description |
|-----------|-------------|
| **Fact Table** | Central table storing quantitative measures (revenue, quantity, cost) |
| **Dimension Tables** | Surrounding tables providing context (time, product, location) |
| **Measures** | Numeric values placed at intersections of dimensions |
| **Hierarchies** | Parent-child relationships within dimensions (year > quarter > month) |

## 3. Core OLAP Operations

### 3.1 Roll-Up (Consolidation)

Aggregates data by climbing up a concept hierarchy or reducing dimensions.

```
City-level sales  -->  Country-level sales
(Tokyo, Osaka)         (Japan)
```

### 3.2 Drill-Down

Reverse of roll-up. Navigates from general to detailed data.

```
Yearly sales  -->  Quarterly sales  -->  Monthly sales
```

### 3.3 Slice

Selects a single dimension value, creating a sub-cube.

```
All Products x All Regions x Q1-2026
                                ^^^^
                             (slice on Time = Q1-2026)
```

### 3.4 Dice

Selects a sub-cube by specifying value ranges on two or more dimensions.

```
Products: [Electronics, Clothing]
Regions:  [Asia, Europe]
Time:     [Q1-2026, Q2-2026]
```

### 3.5 Pivot (Rotate)

Rotates the data axes to provide an alternative view of the same data.

## 4. Types of OLAP

| Type | Description | Pros | Cons |
|------|-------------|------|------|
| **MOLAP** (Multidimensional) | Pre-aggregated data stored in multidimensional cubes | Fastest query performance, optimized indexing/caching | Limited scalability, high storage for large datasets |
| **ROLAP** (Relational) | Operates on relational databases using SQL | Handles large datasets, leverages existing RDBMS | Slower query performance, complex SQL generation |
| **HOLAP** (Hybrid) | Combines MOLAP and ROLAP | Balances speed and scalability | Added complexity |

## 5. OLAP System Architecture

```
Data Sources          ETL Pipeline          Storage            Analysis
+-----------+       +------------+       +----------+       +----------+
| Databases |------>|            |------>| OLAP     |------>| Dashboards|
| Files     |       | Extract    |       | Cube     |       | Reports  |
| APIs      |       | Transform  |       | (Star/   |       | Ad-hoc   |
| Logs      |       | Load       |       | Snowflake|       | queries  |
+-----------+       +------------+       +----------+       +----------+
```

## 6. Use Cases

- **Business Reporting** - Sales dashboards, customer behavior, financial analysis
- **Financial Forecasting** - Budgeting, profit analysis across time periods
- **Supply Chain** - Inventory monitoring, demand forecasting
- **Healthcare** - Patient data analysis, treatment outcome tracking
- **What-if Analysis** - Impact analysis of decisions across departments

## 7. Query Language

OLAP systems typically use **MDX (Multidimensional Expressions)** for querying cubes, analogous to SQL for relational databases.

```mdx
SELECT
  {[Measures].[Revenue], [Measures].[Quantity]} ON COLUMNS,
  {[Product].[Category].Members} ON ROWS
FROM [SalesCube]
WHERE [Time].[2026].[Q1]
```

---

## Sources

- [OLAP - Wikipedia](https://en.wikipedia.org/wiki/Online_analytical_processing)
- [What is OLAP? - AWS](https://aws.amazon.com/what-is/olap/)
- [OLAP Cube - Wikipedia](https://en.wikipedia.org/wiki/OLAP_cube)
- [MOLAP - GeeksforGeeks](https://www.geeksforgeeks.org/dbms/molap-multidimensional-olap/)
- [The Multidimensional Data Model - Oracle](https://docs.oracle.com/cd/B13789_01/olap.101/b10333/multimodel.htm)
- [OLAP Basics - Galaktika](https://galaktika-soft.com/blog/overview-of-olap-technology.html)
