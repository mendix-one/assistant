# Schema Patterns for Multi-Dimensional Analysis

## Summary

Star schema and snowflake schema are the two primary data modeling patterns used in multi-dimensional analysis. They define how fact tables (measures) relate to dimension tables (context) in data warehouses and OLAP systems.

---

## 1. Star Schema

The star schema is the simplest form of dimensional model. A central **fact table** connects directly to **denormalized dimension tables**.

```
              +------------+
              | DimProduct |
              +------------+
                    |
+----------+  +----------+  +-----------+
| DimTime  |--| FactSales|--| DimRegion |
+----------+  +----------+  +-----------+
                    |
              +-------------+
              | DimCustomer |
              +-------------+
```

### Characteristics

- Dimension tables are **denormalized** (all attributes in a single table)
- Fewer joins needed at query time (fact + dimension = 1 join each)
- Some data redundancy in dimension tables
- Simple, intuitive design

### Example: Product Dimension (Denormalized)

| product_id | product_name | brand  | category    | department  |
|------------|-------------|--------|-------------|-------------|
| 1          | iPhone 15   | Apple  | Smartphones | Electronics |
| 2          | Galaxy S24  | Samsung| Smartphones | Electronics |
| 3          | MacBook Pro | Apple  | Laptops     | Electronics |

> "Electronics > Smartphones > Apple" is repeated across rows.

---

## 2. Snowflake Schema

The snowflake schema **normalizes** dimension tables into sub-dimension tables, creating a hierarchy that resembles a snowflake shape.

```
                  +----------+
                  | DimBrand |
                  +----------+
                       |
+----------+    +------------+    +-------------+
| DimMonth |    | DimProduct |    | DimCategory |
+----------+    +------------+    +-------------+
     |                |                  |
+----------+    +----------+    +--------------+
| DimTime  |----| FactSales|----| DimDepartment|
+----------+    +----------+    +--------------+
     |                |
+-----------+   +-------------+
| DimYear   |   | DimCustomer |
+-----------+   +-------------+
                       |
                +-------------+
                | DimSegment  |
                +-------------+
```

### Characteristics

- Dimension tables are **normalized** (broken into sub-tables)
- Reduces data redundancy and storage
- More joins required at query time
- Supports complex hierarchies naturally

### Example: Product Dimension (Normalized)

**DimProduct:**

| product_id | product_name | brand_id | category_id |
|------------|-------------|----------|-------------|
| 1          | iPhone 15   | 1        | 1           |
| 2          | Galaxy S24  | 2        | 1           |

**DimBrand:**

| brand_id | brand_name |
|----------|-----------|
| 1        | Apple     |
| 2        | Samsung   |

**DimCategory:**

| category_id | category_name | department_id |
|-------------|--------------|---------------|
| 1           | Smartphones  | 1             |
| 2           | Laptops      | 1             |

**DimDepartment:**

| department_id | department_name |
|---------------|----------------|
| 1             | Electronics    |

---

## 3. Comparison

| Aspect | Star Schema | Snowflake Schema |
|--------|------------|-----------------|
| **Normalization** | Denormalized dimensions | Normalized dimensions |
| **Query Performance** | Faster (fewer joins) | Slower (more joins) |
| **Storage** | More (data duplication) | Less (no redundancy) |
| **Complexity** | Simple, easy to understand | More complex relationships |
| **ETL** | Simpler to load | More complex transformations |
| **Maintenance** | Easier | Harder (more tables) |
| **Hierarchy Support** | Implicit in single table | Explicit via sub-tables |
| **Best For** | Query-heavy, BI dashboards | Storage-constrained, complex hierarchies |

## 4. Galaxy Schema (Fact Constellation)

A variation where multiple fact tables share dimension tables. Useful when analyzing multiple business processes (sales + inventory + shipping) that share common dimensions.

```
+----------+    +----------+    +----------+
| FactSales|----| DimTime  |----| FactShip |
+----------+    +----------+    +----------+
     |          | DimProduct|         |
     +----------+----------+---------+
```

## 5. When to Use Which

### Use Star Schema When:
- Query performance is the top priority
- The analytics team prefers simpler SQL
- Dimensional hierarchies are straightforward
- Using BI tools that work best with star schemas (most do)

### Use Snowflake Schema When:
- Storage efficiency is important
- Dimensions have deep, complex hierarchies
- Data integrity and normalization are priorities
- ETL processes can handle the additional complexity

### Practical Recommendation

Most modern data warehouses favor **star schema** because:
1. Modern storage is cheap (redundancy cost is minimal)
2. Query performance matters more than storage
3. BI tools optimize for star schema patterns
4. Simpler for analysts to understand and query

---

## Sources

- [Star Schema vs Snowflake Schema - ThoughtSpot](https://www.thoughtspot.com/data-trends/data-modeling/star-schema-vs-snowflake-schema)
- [Snowflake Schema - Wikipedia](https://en.wikipedia.org/wiki/Snowflake_schema)
- [Star Schema vs Snowflake Schema - Integrate.io](https://www.integrate.io/blog/snowflake-schemas-vs-star-schemas-what-are-they-and-how-are-they-different/)
- [Snowflake Schema - Databricks](https://www.databricks.com/glossary/snowflake-schema)
- [Snowflake Schema in Data Warehouse - GeeksforGeeks](https://www.geeksforgeeks.org/dbms/snowflake-schema-in-data-warehouse-model/)
- [Star Schema vs Snowflake Schema - DataCamp](https://www.datacamp.com/blog/star-schema-vs-snowflake-schema)
