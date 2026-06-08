# Chapter 7: Indexes and Physical Design — Making Queries Fast

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

The previous five chapters addressed the *logical* design of a database — what tables to create, how to normalise them, and what constraints to apply. This chapter shifts focus to the *physical* design — how the database stores and retrieves data, and how that physical organisation affects query performance.

At the heart of physical design is the **index**. An index is a separate data structure that SQL Server maintains alongside a table to enable fast lookups. Without indexes, every query that filters or joins must read every row of every table involved — a **full table scan**. For a table with a million rows, a full scan of every query is unacceptably slow. Indexes allow SQL Server to find qualifying rows without examining every row.

Understanding indexes is essential for both database designers and analysts. Designers create the indexes that make the schema perform well. Analysts write queries that either benefit from existing indexes or inadvertently prevent them from being used.

By the end of this chapter you will be able to:

- Explain how SQL Server stores table data using pages and the B-tree structure
- Describe the difference between a clustered and non-clustered index
- Choose the appropriate clustered index key for a table
- Design non-clustered indexes for common query patterns
- Explain covering indexes and when to use them
- Read and interpret a basic execution plan
- Identify queries that cannot use indexes (non-sargable conditions)
- Apply index design principles to the CabotTrail schema
- Explain the trade-offs between more indexes and write performance

---

## Table of Contents

1. [How SQL Server Stores Data](#1-how-sql-server-stores-data)
2. [The B-Tree Index Structure](#2-the-b-tree-index-structure)
3. [Clustered Indexes](#3-clustered-indexes)
4. [Choosing the Clustered Index Key](#4-choosing-the-clustered-index-key)
5. [Non-Clustered Indexes](#5-non-clustered-indexes)
6. [Covering Indexes and Included Columns](#6-covering-indexes-and-included-columns)
7. [Index Design for Common Query Patterns](#7-index-design-for-common-query-patterns)
8. [The Execution Plan](#8-the-execution-plan)
9. [Non-Sargable Conditions: When Indexes Cannot Help](#9-non-sargable-conditions-when-indexes-cannot-help)
10. [Indexes and Write Performance](#10-indexes-and-write-performance)
11. [Index Maintenance](#11-index-maintenance)
12. [Composite Indexes and Column Order](#12-composite-indexes-and-column-order)
13. [Indexes in the CabotTrail Schema](#13-indexes-in-the-cabottrail-schema)
14. [Chapter Summary](#14-chapter-summary)
15. [Review Questions](#15-review-questions)
16. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. How SQL Server Stores Data

Before understanding indexes, you need a mental model of how SQL Server stores data physically. This model explains why some queries are fast and others are slow.

### 1.1 Pages: The Unit of Storage

SQL Server stores all data in **pages** — fixed-size blocks of 8 kilobytes (8,192 bytes). Every read and write operation works at the page level, not the row level. When SQL Server needs to read one row, it reads the entire 8 KB page containing that row.

A typical data row in the CabotTrail `fact.Sales` table uses roughly 200–300 bytes. An 8 KB page holds approximately 25–40 rows. The 16,359 rows in `fact.Sales` span roughly 400–650 pages.

### 1.2 Extents: Groups of Pages

Eight consecutive pages form an **extent** (64 KB). SQL Server allocates and reads data in extents during large scans, which is more efficient than reading individual pages.

### 1.3 The Table Scan

When no index is available for a query's filter condition, SQL Server reads every page of the table from beginning to end — a **table scan** (or **heap scan** for unordered tables). For a 400-page table, this means 400 page reads minimum. For a 1-million-page table, it means 1 million page reads.

A table scan is not always bad — for small tables or queries that need most rows, it can be the right choice. The optimizer chooses between scans and index seeks based on estimated row counts and costs.

---

## 2. The B-Tree Index Structure

SQL Server indexes use a **Balanced Tree (B-tree)** structure — a hierarchical structure where every lookup starts at the root and follows a path down to the leaf level. The "balanced" property means every leaf is at the same depth, guaranteeing predictable lookup time regardless of which value is searched.

### 2.1 The Three Levels

```
Root page:  CustomerID 1–50 → Page A | CustomerID 51–100 → Page B
                                │
Intermediate:   1–25 → Page C | 26–50 → Page D
                        │
Leaf pages:  [1,Fundy Bay][2,Trail Hut]...[25,Cape Breton Supply]
```

- **Root page:** The top of the tree. Contains the highest key value on each sub-page, pointing to the next level.
- **Intermediate pages:** Navigation level(s) between root and leaves.
- **Leaf pages:** The bottom level. Contains the actual data (for clustered indexes) or row pointers (for non-clustered indexes).

### 2.2 The Lookup Process

Finding CustomerID = 42:
1. Read root page: "42 is between 26 and 50, follow the pointer to Page D"
2. Read intermediate page D: "42 is between 38 and 50, follow the pointer to leaf page G"
3. Read leaf page G: "Row with CustomerID = 42 found"

Three page reads to find one row out of a million. Contrast with a table scan: a million page reads for a million-row table. The B-tree is the mechanism that makes indexes fast.

---

## 3. Clustered Indexes

A **clustered index** determines the physical order of rows in the table. The leaf level of a clustered index *is* the table data — the rows are stored in the order of the index key.

### 3.1 One Per Table

A table can have **exactly one** clustered index, because the rows can only be physically sorted one way. When you define a PRIMARY KEY constraint, SQL Server creates a clustered index on those columns by default.

### 3.2 What Happens Without a Clustered Index

A table without a clustered index is called a **heap** — rows are stored in insertion order with no organisation. Lookups on a heap always require a full scan (unless a non-clustered index is present). For most tables, a clustered index is preferable to a heap.

### 3.3 Clustered Index Seeks vs Scans

A **clustered index seek** navigates the B-tree to find specific rows — efficient for equality searches and narrow range searches.

A **clustered index scan** reads all leaf pages of the clustered index — equivalent to a table scan but organised by key order. Efficient when the query needs most rows; inefficient when it needs few rows.

```sql
-- This benefits from a clustered index seek on CustomerID
SELECT CustomerName FROM Sales.Customers WHERE CustomerID = 42;

-- This performs a clustered index scan (needs all rows)
SELECT CustomerName FROM Sales.Customers ORDER BY CustomerName;
```

---

## 4. Choosing the Clustered Index Key

Because the clustered index determines physical row order, the choice of clustered index key significantly affects both query performance and storage efficiency.

### 4.1 Properties of a Good Clustered Index Key

**Narrow:** The clustered index key is stored in every non-clustered index as a row locator. A wide key (many columns, or wide data types) inflates every non-clustered index's size.

**Unique:** If the clustered key is not unique, SQL Server adds a 4-byte uniquifier to every duplicate, adding storage overhead and complicating lookups.

**Static:** Changing a clustered index key value requires moving the row to its new sorted position — an expensive operation. Keys that frequently change (email addresses, names) make poor clustered index keys.

**Ever-increasing:** Keys that always increase (IDENTITY integers, sequential GUIDs) cause new rows to be appended at the end of the table — efficient. Random keys (standard GUIDs, hash values) cause pages to split when new rows are inserted between existing ones — creating **page splits** that fragment the index.

### 4.2 Why IDENTITY Is an Excellent Clustered Key

IDENTITY integers satisfy all four properties:
- **Narrow:** 4 bytes (INT)
- **Unique:** Each value is unique by definition
- **Static:** Surrogate keys never change
- **Ever-increasing:** New values are always higher than existing ones

This is the strongest argument for surrogate IDENTITY keys — beyond their role as stable identifiers, they produce excellent clustered index performance.

### 4.3 When to Choose a Different Clustered Key

Sometimes the most common query pattern suggests a different clustered key than the PK:

```
A table frequently queried by date range:
    WHERE OrderDate BETWEEN '2024-01-01' AND '2024-03-31'
```

If most queries retrieve rows by `OrderDate`, clustering by `OrderDate` means that all rows for a given date range are physically adjacent on disk — a **range scan** retrieves them in one sequential read. Clustering by `OrderID` (IDENTITY) means date-range rows are scattered across the entire table.

For such tables, consider creating the clustered index on `OrderDate` and adding a separate unique non-clustered index on the PK (`OrderID`). This is a non-default choice that requires explicit DDL:

```sql
-- Cluster on OrderDate instead of OrderID
CREATE TABLE Sales.Orders
(
    OrderID     INT         NOT NULL IDENTITY(1,1),
    OrderDate   DATE        NOT NULL,
    CustomerID  INT         NOT NULL,
    ...
);

-- Create clustered index on OrderDate (not the PK)
CREATE CLUSTERED INDEX CX_Orders_OrderDate
    ON Sales.Orders (OrderDate);

-- Create unique non-clustered index to enforce PK uniqueness
CREATE UNIQUE NONCLUSTERED INDEX UX_Orders_OrderID
    ON Sales.Orders (OrderID);
```

This is an advanced design choice. For most tables, the default (clustered index on the surrogate PK) is correct and simple.

---

## 5. Non-Clustered Indexes

A **non-clustered index** is a separate B-tree structure that contains the index key columns and a pointer back to the table row. It does not change the physical order of the table.

### 5.1 Structure

The leaf level of a non-clustered index contains:
- The index key column(s)
- A **row locator**: for a heap, this is a physical row address; for a table with a clustered index, this is the clustered index key value

When SQL Server finds a matching row in the non-clustered index, it uses the row locator to retrieve the full row from the clustered index (or heap) — a **key lookup**.

### 5.2 Creating a Non-Clustered Index

```sql
-- Index on a single column
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
    ON Sales.Orders (CustomerID);

-- Index on a single column with explicit name and fill factor
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate
    ON Sales.Orders (OrderDate)
    WITH (FILLFACTOR = 80);  -- Leave 20% of each page empty for future inserts
```

### 5.3 When Non-Clustered Indexes Are Used

SQL Server uses a non-clustered index when:
1. The query filters on the index key column(s): `WHERE CustomerID = 42`
2. The estimated number of qualifying rows is small enough that a seek+lookup is faster than a scan

For large tables with selective filters (few rows per filter value), non-clustered indexes are essential. For filters that return most rows, the optimizer may prefer a scan even with an index available.

---

## 6. Covering Indexes and Included Columns

### 6.1 The Key Lookup Problem

After a non-clustered index seek, SQL Server must retrieve the full row from the clustered index to return any columns not in the non-clustered index. This **key lookup** is an additional I/O operation per row found — for large result sets, it negates the index's benefit.

```sql
-- Index on CustomerID only
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID ON Sales.Orders (CustomerID);

-- Query: uses the index to find rows, then key-looks up SalesRepID and OrderDate
SELECT CustomerID, SalesRepID, OrderDate
FROM   Sales.Orders
WHERE  CustomerID = 42;
-- Plan: Index Seek (IX_Orders_CustomerID) → Key Lookup (clustered) for SalesRepID, OrderDate
```

For a customer with 500 orders, this means 500 key lookups in addition to the index seek.

### 6.2 Covering Indexes

A **covering index** includes all the columns a query needs, so SQL Server can answer the query entirely from the index without touching the main table.

```sql
-- Covering index: includes all columns the query needs
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID_Covering
    ON Sales.Orders (CustomerID)
    INCLUDE (SalesRepID, OrderDate);
-- Now the query above can be answered from the index alone — no key lookups
```

The `INCLUDE` clause adds columns to the leaf level of the index without including them in the key (which would change the sort order and index size). The result: the index is larger but the query avoids key lookups.

### 6.3 When to Use Covering Indexes

Covering indexes are most valuable for:
- Frequently executed queries on large tables
- Queries that return a small number of columns for a filtered set of rows
- Reports that run on a fixed set of columns for a common filter

The cost: each additional `INCLUDE` column adds to the index size and slows INSERT/UPDATE/DELETE operations (more indexes to maintain).

---

## 7. Index Design for Common Query Patterns

### 7.1 The Index Design Process

Good index design starts from query patterns — the actual queries that will run against the table — not from theoretical principles. The process:

1. Identify the most frequent and most performance-critical queries
2. For each query, identify the WHERE clause columns (filter), JOIN columns, and SELECT columns
3. Design an index with the most selective filter columns first in the key
4. Add additional columns to the key if they help with range scans or ordering
5. Add `INCLUDE` columns if they eliminate key lookups for high-frequency queries

### 7.2 Selectivity: The Key Concept

**Selectivity** is the proportion of rows returned by a filter condition. A highly selective filter returns few rows; a low-selectivity filter returns many rows.

- `WHERE CustomerID = 42` — highly selective (one customer out of 100)
- `WHERE IsDiscontinued = 0` — low selectivity (most products are active)

Indexes provide the greatest benefit for high-selectivity conditions. An index on `IsDiscontinued` has limited value because most queries using it will retrieve a large fraction of the table — a scan may be just as fast.

```sql
-- High value: index on CustomerID (1 of 100 customers — high selectivity)
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID ON Sales.Orders (CustomerID);

-- Moderate value: index on OrderDate (useful for date range queries)
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate ON Sales.Orders (OrderDate);

-- Low value: index on IsDiscontinued (BIT — only 2 possible values)
-- SQL Server will often ignore this index in favour of a scan
-- CREATE NONCLUSTERED INDEX IX_Products_IsDiscontinued -- probably not worth it
```

### 7.3 A Standard Index Set for Fact Tables

For a fact table like `fact.Sales`, the most important indexes are on the FK columns (used in JOINs) and on the date key (used in filters):

```sql
-- Indexes for CabotTrailOutdoorsSales.fact.Sales
-- (Clustered index on SalesKey already created by PRIMARY KEY)

-- FK indexes (essential for JOIN performance)
CREATE NONCLUSTERED INDEX IX_FactSales_CustomerKey
    ON fact.Sales (CustomerKey);

CREATE NONCLUSTERED INDEX IX_FactSales_ProductKey
    ON fact.Sales (ProductKey);

CREATE NONCLUSTERED INDEX IX_FactSales_EmployeeKey
    ON fact.Sales (EmployeeKey);

-- Date key index (essential for date range filters)
CREATE NONCLUSTERED INDEX IX_FactSales_OrderDateKey
    ON fact.Sales (OrderDateKey)
    INCLUDE (CustomerKey, ProductKey, LineTotal, GrossProfit);
-- INCLUDE columns cover the most common analytical query pattern:
-- filter by date, join to Customer and Product, sum LineTotal and GrossProfit
```

---

## 8. The Execution Plan

The **execution plan** is SQL Server's recipe for executing a query — which tables to read, in what order, which indexes to use, and how to combine results. Reading execution plans is the primary tool for understanding and improving query performance.

### 8.1 Viewing an Execution Plan in SSMS

Two options in SSMS:
- **Estimated plan** (`Ctrl+L` or Query menu → Display Estimated Execution Plan): Shows the plan without running the query
- **Actual plan** (`Ctrl+M` toggles inclusion, then run with `F5`): Shows the plan with actual row counts from the execution

For performance analysis, the actual plan is more informative — estimated row counts can be wrong when statistics are stale.

### 8.2 Key Plan Operators

| Operator | What it means | Performance implication |
|---|---|---|
| **Clustered Index Seek** | Navigated B-tree to find specific rows | Efficient — few page reads |
| **Clustered Index Scan** | Read all pages of the clustered index | Expensive for large tables |
| **Non-Clustered Index Seek** | Navigated non-clustered B-tree | Efficient for selective queries |
| **Key Lookup** | Retrieved full row from clustered index after non-clustered seek | Extra I/O per row — consider covering index |
| **Hash Match** | Joined two inputs using a hash table | Memory-intensive; common for large joins |
| **Nested Loops** | For each row in outer input, searched inner input | Efficient when outer is small |
| **Merge Join** | Joined two sorted inputs simultaneously | Efficient when both inputs are sorted |
| **Sort** | Sorted input rows | Expensive; breaks streaming; indicates missing index |
| **Table Spool** | Spooled intermediate results to tempdb | Memory pressure; complex query |

### 8.3 Reading Plan Cost

Each operator in the execution plan has an estimated cost (as a percentage of total query cost). Operators with high percentages are the bottlenecks — candidates for index improvement.

```sql
-- Enable actual execution plan, then run this query
USE CabotTrailOutdoorsSales;

SELECT
    prod.CategoryName,
    SUM(fs.LineTotal)   AS Revenue
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName
ORDER BY Revenue DESC;

-- Look for: which table is scanned? Is there a Key Lookup?
-- Does it use an index seek or scan on fact.Sales?
```

### 8.4 The Missing Index Hint

When SQL Server determines that a query could benefit from an index that does not exist, it adds a **missing index suggestion** to the execution plan — shown as a green "Missing Index" banner at the top of the plan. The suggestion shows the exact `CREATE INDEX` statement to run.

Missing index suggestions are valuable but should not be applied blindly:
- Each suggested index has a cost (storage, write overhead)
- Multiple similar suggestions may be consolidatable into one index
- The query optimizer suggests indexes per-query; a database with many queries may have hundreds of suggestions, not all of which are worth implementing

---

## 9. Non-Sargable Conditions: When Indexes Cannot Help

Chapter 2 of *SQL for Analysts* and the Deeper Dive of Chapter 2 introduced **sargability**. This section deepens that concept from the designer's perspective.

### 9.1 What Non-Sargable Means

A **sargable** condition (Search ARGument ABLE) is one that can be resolved using an index — typically a direct comparison on a column. A **non-sargable** condition applies a function or expression to the column, forcing SQL Server to evaluate the expression for every row rather than using the B-tree.

### 9.2 Non-Sargable Patterns

```sql
-- NON-SARGABLE: function applied to column — full scan required
WHERE YEAR(OrderDate) = 2024
WHERE UPPER(CustomerName) = 'FUNDY BAY'
WHERE LEN(ProductCode) > 5

-- SARGABLE equivalents: comparison directly against column value
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
WHERE CustomerName = 'Fundy Bay'   -- (relies on case-insensitive collation)
WHERE ProductCode > 'AAAAA'        -- (approximate length filter via range)
```

### 9.3 The Designer's Responsibility

As a database designer, you can help analysts write sargable queries by:

**Storing computed values as separate columns.** If analysts frequently filter by year, add a `CalendarYear` column (or use the Calendar dimension) rather than making them apply `YEAR()` to a date column.

**Choosing data types that support direct comparison.** Storing dates as `DATE` (not `NVARCHAR`) allows `>= '2024-01-01'` to use a date index. Storing numbers as `DECIMAL` (not `NVARCHAR`) allows `> 100` to use a numeric index.

**Avoiding case-sensitive collations** for columns that will be searched with equality. Case-insensitive collation (the SQL Server default) allows `WHERE Name = 'value'` to be sargable without `UPPER()` transformation.

### 9.4 Computed Persisted Columns as an Index Workaround

If a non-sargable expression is frequently used in WHERE clauses, a **computed persisted column** plus an index on it provides a sargable alternative:

```sql
-- Add a persisted computed column for the year
ALTER TABLE Sales.Orders
ADD OrderYear AS YEAR(OrderDate) PERSISTED;

-- Index the computed column
CREATE NONCLUSTERED INDEX IX_Orders_OrderYear
    ON Sales.Orders (OrderYear);

-- Now this query can use the index
WHERE OrderYear = 2024   -- Sargable — directly comparing column value
```

The query `WHERE YEAR(OrderDate) = 2024` is still non-sargable. But `WHERE OrderYear = 2024` uses the index on the pre-computed column.

---

## 10. Indexes and Write Performance

Every index is a trade-off. Indexes accelerate reads; they add overhead to writes.

### 10.1 The Write Overhead

When a row is inserted, updated, or deleted, SQL Server must maintain every index on that table:

- **INSERT:** Add the new row's key value to every index. If a page split occurs in any index (the target page is full), SQL Server allocates a new page and redistributes the rows — expensive.
- **UPDATE:** If the updated column is part of an index key, update the index (remove the old key, insert the new key). If the updated column is only in `INCLUDE`, update the leaf-level entry.
- **DELETE:** Remove the row's key value from every index.

A table with 10 non-clustered indexes requires 11 B-tree updates for every single-row insert (1 clustered + 10 non-clustered).

### 10.2 The Index Design Trade-Off

```
More indexes → faster reads, slower writes, more storage
Fewer indexes → slower reads, faster writes, less storage
```

The right balance depends on the access pattern:
- **OLTP tables** (many writes, many targeted reads): conservative indexing — essential indexes only (PK, FKs, high-selectivity filter columns)
- **OLAP/DW fact tables** (few writes — nightly load — many reads): generous indexing — cover common query patterns aggressively

### 10.3 Filtering During Bulk Loads

When bulk-loading a large table (as in the CabotTrail ETL), temporarily disabling non-clustered indexes before the load and rebuilding them after is significantly faster than maintaining them row-by-row during the load:

```sql
-- Before bulk load: disable non-clustered indexes
ALTER INDEX IX_FactSales_CustomerKey ON fact.Sales DISABLE;
ALTER INDEX IX_FactSales_ProductKey  ON fact.Sales DISABLE;

-- [Bulk load operation]

-- After bulk load: rebuild (not re-enable — REBUILD also reorganises the pages)
ALTER INDEX IX_FactSales_CustomerKey ON fact.Sales REBUILD;
ALTER INDEX IX_FactSales_ProductKey  ON fact.Sales REBUILD;
```

---

## 11. Index Maintenance

Indexes degrade over time as rows are inserted, updated, and deleted. **Fragmentation** occurs when the logical order of index pages diverges from the physical order on disk, or when pages have excessive empty space.

### 11.1 Types of Fragmentation

**Logical fragmentation:** Index pages are not in sequential physical order. A range scan must hop between non-sequential pages rather than reading ahead.

**Page density (internal fragmentation):** Pages have significant empty space due to deletions or page splits. SQL Server reads more pages than necessary because each page holds fewer rows.

### 11.2 Detecting Fragmentation

```sql
-- Check fragmentation for all indexes on fact.Sales
SELECT
    i.name                      AS IndexName,
    s.avg_fragmentation_in_percent,
    s.avg_page_space_used_in_percent,
    s.page_count
FROM    sys.dm_db_index_physical_stats(
            DB_ID('CabotTrailOutdoorsSales'),
            OBJECT_ID('fact.Sales'),
            NULL, NULL, 'LIMITED') s
INNER JOIN sys.indexes i
    ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE   s.index_id > 0   -- Exclude heap
ORDER BY s.avg_fragmentation_in_percent DESC;
```

### 11.3 Fragmentation Thresholds and Actions

| Fragmentation | Action | SQL command |
|---|---|---|
| < 5% | No action needed | — |
| 5% – 30% | Reorganise (online, minimal locking) | `ALTER INDEX ... REORGANIZE` |
| > 30% | Rebuild (offline or online with Enterprise edition) | `ALTER INDEX ... REBUILD` |

```sql
-- Reorganise a fragmented index (light maintenance)
ALTER INDEX IX_FactSales_CustomerKey ON fact.Sales REORGANIZE;

-- Rebuild an index (full defragmentation, updates statistics)
ALTER INDEX IX_FactSales_CustomerKey ON fact.Sales REBUILD;

-- Rebuild all indexes on a table
ALTER INDEX ALL ON fact.Sales REBUILD;
```

Index maintenance is typically run as a scheduled SQL Server Agent job during off-peak hours — part of the ETL maintenance window described in Chapter 8 of the ETL textbook.

---

## 12. Composite Indexes and Column Order

When an index spans multiple columns, **column order matters** — it determines which queries can use the index and which cannot.

### 12.1 The Leftmost Prefix Rule

A composite index can be used by queries that filter on the **leftmost prefix** of the index key — a contiguous set of columns starting from the leftmost. Queries that skip the leftmost column(s) cannot use the index efficiently.

```sql
-- Composite index: (SalesTerritory, CalendarYear, CategoryName)
CREATE NONCLUSTERED INDEX IX_Sales_Composite
    ON fact.Sales (SalesTerritory, CalendarYear, CategoryName);

-- CAN use this index (matches leftmost prefix):
WHERE SalesTerritory = 'Atlantic — Nova Scotia'                          -- prefix: 1 col
WHERE SalesTerritory = 'Atlantic — Nova Scotia' AND CalendarYear = 2024  -- prefix: 2 cols
WHERE SalesTerritory = 'Atlantic — Nova Scotia' AND CalendarYear = 2024
      AND CategoryName = 'Sleeping'                                       -- prefix: 3 cols

-- CANNOT use this index efficiently (skips leftmost column):
WHERE CalendarYear = 2024                    -- SalesTerritory is missing
WHERE CategoryName = 'Sleeping'              -- Both SalesTerritory and CalendarYear missing
```

### 12.2 Choosing Column Order

Put the **most selective column first** when the index is used for equality searches — this narrows the range as quickly as possible.

Put the **range predicate column last** (among the key columns) for range searches — a range on an early column prevents the index from being useful for subsequent equality columns.

```sql
-- Good: equality columns before range column
-- Query: WHERE CustomerKey = 15 AND OrderDateKey BETWEEN 20240101 AND 20241231
CREATE NONCLUSTERED INDEX IX_FactSales_Cust_Date
    ON fact.Sales (CustomerKey, OrderDateKey);
-- CustomerKey equality narrows to one customer's rows; OrderDateKey range scans within those

-- Poor: range column before equality column (usually)
-- Query: WHERE OrderDateKey BETWEEN 20240101 AND 20241231 AND CustomerKey = 15
-- An index on (OrderDateKey, CustomerKey) would find all 2024 rows,
-- then filter by CustomerKey — less efficient than customer first
```

---

## 13. Indexes in the CabotTrail Schema

### 13.1 Querying Existing Indexes

```sql
-- All indexes in CabotTrailOutdoorsSales
SELECT
    s.name                  AS SchemaName,
    t.name                  AS TableName,
    i.name                  AS IndexName,
    i.type_desc             AS IndexType,
    i.is_unique,
    i.is_primary_key,
    -- Key columns
    STRING_AGG(
        CASE WHEN ic.is_included_column = 0
             THEN c.name + ' (' + CASE ic.is_descending_key WHEN 1 THEN 'DESC' ELSE 'ASC' END + ')'
             ELSE NULL
        END,
        ', '
    ) WITHIN GROUP (ORDER BY ic.key_ordinal)        AS KeyColumns,
    -- Included columns
    STRING_AGG(
        CASE WHEN ic.is_included_column = 1
             THEN c.name
             ELSE NULL
        END,
        ', '
    ) WITHIN GROUP (ORDER BY ic.index_column_id)    AS IncludedColumns
FROM    sys.indexes i
INNER JOIN sys.tables t     ON t.object_id  = i.object_id
INNER JOIN sys.schemas s    ON s.schema_id  = t.schema_id
INNER JOIN sys.index_columns ic ON ic.object_id = i.object_id
                               AND ic.index_id   = i.index_id
INNER JOIN sys.columns c    ON c.object_id  = ic.object_id
                           AND c.column_id  = ic.column_id
WHERE   i.type > 0
AND     s.name IN ('fact', 'dim')
GROUP BY s.name, t.name, i.name, i.type_desc, i.is_unique, i.is_primary_key
ORDER BY s.name, t.name, i.name;
```

### 13.2 What Good Index Coverage Looks Like

For `CabotTrailOutdoorsSales.fact.Sales`, a well-indexed configuration would include:

| Index | Type | Key columns | Purpose |
|---|---|---|---|
| `PK_Sales` | Clustered | `SalesKey` | Row identity and physical order |
| `IX_Sales_OrderDateKey` | Non-clustered | `OrderDateKey` | Date range queries |
| `IX_Sales_CustomerKey` | Non-clustered | `CustomerKey` | JOIN to dim.Customer |
| `IX_Sales_ProductKey` | Non-clustered | `ProductKey` | JOIN to dim.Product |
| `IX_Sales_EmployeeKey` | Non-clustered | `EmployeeKey` | JOIN to dim.Employee |
| `IX_Sales_DateKey_Covering` | Non-clustered | `OrderDateKey, CustomerKey` INCLUDE `LineTotal, GrossProfit` | Common analytical pattern |

### 13.3 Index Gaps in a Fresh Schema

A freshly created schema typically has only the PK clustered index. The FK indexes and covering indexes described above must be added as part of the physical design step. This is why physical design (this chapter) is a distinct step from logical design (normalization, constraints):

1. Create tables with correct structure and constraints (Chapters 3–6)
2. Add indexes based on expected query patterns (this chapter)
3. Load data and run representative queries
4. Use execution plans to identify remaining bottlenecks
5. Add targeted indexes for identified bottlenecks

---

## 14. Chapter Summary

- SQL Server stores data in **8 KB pages**. Index lookups read a small number of pages; table scans read all pages.

- **B-tree indexes** use a balanced tree structure to enable fast lookups: navigate from root to leaf in O(log n) page reads.

- A **clustered index** determines the physical row order of the table. Tables can have only one. The default is the PRIMARY KEY. An ideal clustered key is narrow, unique, static, and ever-increasing — `IDENTITY INT` satisfies all four.

- A **non-clustered index** is a separate B-tree with a row pointer (key lookup) to the clustered index. Multiple non-clustered indexes can exist per table.

- A **covering index** adds `INCLUDE` columns to the leaf level, allowing queries to be answered entirely from the index without key lookups.

- **Index design** starts from query patterns: most selective filter columns first in the key, range columns last among key columns, cover frequently queried columns with INCLUDE.

- The **execution plan** (`Ctrl+L` estimated, `Ctrl+M` actual) shows the operators SQL Server uses. Key operators: Index Seek (fast), Index Scan (expensive), Key Lookup (consider covering), Sort (missing index or ORDER BY mismatch).

- **Non-sargable conditions** (functions on columns in WHERE) prevent index use. Design schemas and query patterns to keep WHERE conditions sargable.

- **Indexes cost write performance.** Every INSERT/UPDATE/DELETE must maintain all indexes. OLTP tables: conservative indexing. OLAP/DW tables: generous indexing. Disable and rebuild during bulk loads.

- **The leftmost prefix rule**: composite indexes are usable for queries filtering on any contiguous prefix of the key, starting from the leftmost column.

---

## 15. Review Questions

1. Explain the difference between a clustered index scan and a non-clustered index seek with key lookup. Under what circumstances would the optimizer choose the scan even when an index exists?

2. The `dim.Customer` table has a clustered index on `CustomerKey` (the surrogate PK). A common analytical query filters by `SalesTerritory`:
   ```sql
   SELECT CustomerName, CreditLimit FROM dim.Customer WHERE SalesTerritory = 'Atlantic — Nova Scotia';
   ```
   What index would improve this query? Write the complete `CREATE INDEX` statement, including `INCLUDE` columns if appropriate.

3. You create the following composite index on `fact.Sales`:
   ```sql
   CREATE NONCLUSTERED INDEX IX_Sales_Combo ON fact.Sales (OrderDateKey, CustomerKey);
   ```
   For each of the following queries, state whether this index can be used efficiently and explain why:
   a. `WHERE OrderDateKey = 20240315`
   b. `WHERE CustomerKey = 15`
   c. `WHERE OrderDateKey = 20240315 AND CustomerKey = 15`
   d. `WHERE OrderDateKey BETWEEN 20240101 AND 20241231 AND CustomerKey = 15`

4. A fact table has no non-clustered indexes. An analyst runs a query joining it to `dim.Customer` and the execution plan shows a clustered index scan on the fact table with an estimated 16,359 rows. They add `CREATE NONCLUSTERED INDEX IX_Fact_CustomerKey ON fact.Sales (CustomerKey)` and re-run the query. Describe what the new execution plan is likely to show, and under what condition the optimizer might still choose the scan.

5. Write a query against `sys.dm_db_index_physical_stats` to check fragmentation for all indexes in `CabotTrailOutdoorsSales.fact.Sales`. Interpret hypothetical results: fragmentation is 8% for `IX_Sales_CustomerKey` and 45% for `IX_Sales_OrderDateKey`. What action would you take for each, and why?

6. The following WHERE clause is used in a frequently executed report:
   ```sql
   WHERE DATEPART(year, OrderDate) = 2024 AND DATEPART(quarter, OrderDate) = 2
   ```
   Explain why this is non-sargable. Propose two design changes — one at the query level and one at the schema level — that would make the filter sargable.

7. A table used in an OLTP transaction processing system has 15 non-clustered indexes. A DBA reports that INSERT performance has degraded significantly. Explain the relationship between the number of indexes and INSERT performance, and describe the process you would use to evaluate which indexes are actually being used.

8. Design the index strategy for the `Sales.LoyaltyTransactions` table created in Chapter 3. The most common queries are:
   - "What transactions has account 42 had in the last 30 days?" (filter by AccountID and TransactionDate)
   - "How many EARN transactions were there per month?" (filter by TransactionType, group by month)
   - "What rewards has account 42 redeemed?" (filter by AccountID and RewardID where not null)
   Write the CREATE INDEX statements for each pattern, explaining your column order and INCLUDE choices.

---

## 🔍 Deeper Dive

### Going Further with Indexes and Physical Design

#### Columnstore Indexes

Chapter 8 of the ETL textbook mentioned **columnstore indexes** as a performance option for large analytical fact tables. They deserve a fuller explanation here.

Traditional indexes store data **row by row** — a B-tree page contains complete rows. **Columnstore indexes** store data **column by column** — all values of a single column are stored together, heavily compressed.

For analytical queries that aggregate a few columns across millions of rows:

```sql
-- This query reads LineTotal and GrossProfit from all 16M rows
SELECT prod.CategoryName, SUM(fs.LineTotal), SUM(fs.GrossProfit)
FROM   fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
GROUP BY prod.CategoryName;
```

With a row-store index, SQL Server reads all columns of every qualifying row even though the query only needs two. With a columnstore index, only the `LineTotal` and `GrossProfit` columns (and the `ProductKey` for the JOIN) need to be read — dramatically reducing I/O.

```sql
-- Create a clustered columnstore index on a fact table
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales
    ON fact.Sales;

-- Or add a non-clustered columnstore alongside the existing row-store indexes
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactSales_Analytics
    ON fact.Sales (OrderDateKey, CustomerKey, ProductKey, LineTotal, GrossProfit);
```

The trade-off: columnstore indexes are optimised for batch reads, not for individual row lookups or frequent small updates. They are ideal for DW fact tables loaded nightly; they are less suited for OLTP tables with many individual row insertions.

Microsoft documentation:
[Columnstore Indexes — Overview](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview)

#### Filtered Indexes

A **filtered index** is a non-clustered index that indexes only a subset of rows — rows that satisfy a filter predicate. Filtered indexes are smaller, faster to maintain, and more efficient for queries that consistently filter on the same condition.

```sql
-- Only index active (non-discontinued) products
CREATE NONCLUSTERED INDEX IX_Products_Active
    ON Inventory.Products (ProductName, UnitPrice)
    WHERE IsDiscontinued = 0;

-- Only index open loyalty accounts
CREATE NONCLUSTERED INDEX IX_LoyaltyAccounts_Active
    ON Sales.LoyaltyAccounts (PointBalance, TierID)
    WHERE PointBalance > 0;
```

A filtered index on `IsDiscontinued = 0` covers 95%+ of analytical queries about products (which typically filter to active products) while indexing only the active rows. The resulting index is much smaller than a full-table index and requires less maintenance during updates to discontinued products.

Filtered indexes are particularly effective for:
- Tables with a status flag where most queries filter on the active/current status
- Sparse columns (columns that are NULL for most rows)
- Tables with a "soft delete" flag

Microsoft documentation:
[Create Filtered Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes)

#### Statistics and the Query Optimizer

SQL Server's query optimizer uses **statistics** — histograms of column value distributions — to estimate how many rows each operator will process. Based on these estimates, it chooses between different execution plan options (index seek vs scan, hash join vs nested loops, etc.).

Statistics are automatically created when an index is created and automatically updated when a significant fraction of rows change. But they can become stale between updates:

```sql
-- View statistics for a column
DBCC SHOW_STATISTICS ('fact.Sales', 'IX_FactSales_CustomerKey');

-- Manually update statistics (run when optimizer is making poor choices)
UPDATE STATISTICS fact.Sales;

-- Update all statistics in the database
EXEC sp_updatestats;
```

When statistics are stale, the optimizer may significantly underestimate or overestimate row counts — leading to poor plan choices (using a scan when a seek would be better, or vice versa). Regular statistics maintenance — typically as part of the same maintenance window as index maintenance — keeps the optimizer making good decisions.

---

### References and Further Reading

1. Fritchey, G. (2018). *SQL Server Query Performance Tuning* (5th ed.). Apress. — The most comprehensive practical guide to SQL Server indexes and execution plans. Chapters 3–7 cover index design; Chapters 9–12 cover execution plan reading.

2. Ben-Gan, I., Kollar, L., Sarka, D., & Talmage, R. (2015). *T-SQL Querying*. Microsoft Press. — Chapters 12–13 cover indexes and query optimisation at depth.

3. Kalen Delaney. (2014). *Microsoft SQL Server 2014 Internals*. Microsoft Press. — The authoritative reference for SQL Server storage internals including page structure, B-tree mechanics, and index architecture.

4. Microsoft. (2024). *Clustered and Non-Clustered Indexes Described*. [https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described)

5. Microsoft. (2024). *Columnstore Indexes Overview*. [https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview)

6. Microsoft. (2024). *Create Filtered Indexes*. [https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes)

7. Microsoft. (2024). *sys.dm_db_index_physical_stats (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql)

8. Microsoft. (2024). *Execution Plans*. [https://learn.microsoft.com/en-us/sql/relational-databases/performance/execution-plans](https://learn.microsoft.com/en-us/sql/relational-databases/performance/execution-plans)

---

*Previous chapter: [Chapter 6 — Constraints and Data Integrity: Encoding Business Rules](../chapter-06-constraints-integrity/README.md)*

*Next chapter: [Chapter 8 — OLTP vs OLAP: Two Design Philosophies](../chapter-08-oltp-vs-olap/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
