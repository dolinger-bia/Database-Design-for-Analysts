# Chapter 8: OLTP vs OLAP — Two Design Philosophies

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Every concept in Chapters 1 through 7 has been applied to one kind of database: the operational, transactional database — the OLTP. But you have been working with two kinds of databases throughout this course and the prerequisite: the CabotTrail OLTP and the CabotTrail data warehouse and data marts. The contrast between them is not accidental. They are built for fundamentally different purposes, and those purposes drive radically different design decisions.

This chapter examines that contrast directly. It explains *why* the same information is structured so differently in `CabotTrailOutdoor` vs `CabotTrailOutdoorsSales`, why one is normalised and the other is deliberately not, and why both are correctly designed for their respective purposes.

This chapter also positions what you have learned in this course within the broader data architecture. As an analyst, you will work with both OLTP and OLAP systems throughout your career. Understanding the design philosophy behind each — and knowing which questions to ask of each — is a core professional competency.

By the end of this chapter you will be able to:

- Define OLTP and OLAP and describe the primary purpose of each
- Explain the workload characteristics that distinguish OLTP from OLAP
- Describe the star schema design pattern and justify its trade-offs
- Compare the CabotTrail OLTP and DW schemas and explain specific design differences
- Explain the role of surrogate keys, slowly changing dimensions, and degenerate dimensions in a DW
- Describe the ETL pipeline that connects OLTP to DW
- Explain the concept of conformed dimensions and why they matter
- Position OLAP within the broader analytics architecture including data marts and the executive summary database

---

## Table of Contents

1. [Two Databases, Two Purposes](#1-two-databases-two-purposes)
2. [OLTP: Designed for Operations](#2-oltp-designed-for-operations)
3. [OLAP: Designed for Analysis](#3-olap-designed-for-analysis)
4. [The Design Consequences of Different Purposes](#4-the-design-consequences-of-different-purposes)
5. [The Star Schema: OLAP's Core Structure](#5-the-star-schema-olapts-core-structure)
6. [Dimension Tables in Depth](#6-dimension-tables-in-depth)
7. [Fact Tables in Depth](#7-fact-tables-in-depth)
8. [Surrogate Keys in the DW Context](#8-surrogate-keys-in-the-dw-context)
9. [Slowly Changing Dimensions](#9-slowly-changing-dimensions)
10. [Conformed Dimensions](#10-conformed-dimensions)
11. [The ETL Pipeline as the Bridge](#11-the-etl-pipeline-as-the-bridge)
12. [The CabotTrail Architecture End to End](#12-the-cabottrail-architecture-end-to-end)
13. [Chapter Summary](#13-chapter-summary)
14. [Review Questions](#14-review-questions)
15. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. Two Databases, Two Purposes

### 1.1 The Same Data, Different Structure

Consider the revenue for order number 1001. In `CabotTrailOutdoor` (OLTP), finding the total revenue for that order requires joining at least five tables:

```sql
-- OLTP: revenue for order 1001
USE CabotTrailOutdoor;

SELECT
    o.OrderID,
    c.CustomerName,
    SUM(il.LineTotal)   AS OrderRevenue
FROM    Sales.Orders o
INNER JOIN Sales.Customers c    ON c.CustomerID   = o.CustomerID
INNER JOIN Sales.Invoices i     ON i.OrderID      = o.OrderID
INNER JOIN Sales.InvoiceLines il ON il.InvoiceID  = i.InvoiceID
WHERE   o.OrderID = 1001
GROUP BY o.OrderID, c.CustomerName;
```

In `CabotTrailOutdoorsSales` (data mart), the same analysis is one table with a filter:

```sql
-- Data mart: revenue for order 1001
USE CabotTrailOutdoorsSales;

SELECT
    InvoiceID,
    SUM(LineTotal)  AS Revenue
FROM    fact.Sales
WHERE   OrderID = 1001
GROUP BY InvoiceID;
```

The data mart version is dramatically simpler. But this simplicity comes at a cost — the data mart cannot answer operational questions ("was this order picked yet? who approved the credit?") that the OLTP was designed for.

Neither design is better than the other. They are different tools for different jobs.

### 1.2 The Two Letters That Explain Everything

**OLTP** — Online Transaction Processing.
**OLAP** — Online Analytical Processing.

These acronyms were coined in the 1980s and 1990s to distinguish two patterns of database usage that had emerged in practice. They capture the core distinction:

- **Transaction processing:** Recording what the business *does* — one event at a time, many users simultaneously, requiring correctness above all
- **Analytical processing:** Understanding what the business *has done* — across large datasets, typically by one or few users, requiring query performance above all

---

## 2. OLTP: Designed for Operations

### 2.1 Characteristics of OLTP Workloads

An OLTP database is the system of record for a business's operational activity. It is designed to:

- **Process many small transactions simultaneously** — hundreds or thousands of users placing orders, updating records, and querying balances at the same time
- **Record each transaction quickly and accurately** — an order that takes 10 seconds to save is unacceptable; an order that saves the wrong data is catastrophic
- **Support current state queries** — "what is Customer 42's current credit balance?" not "what was Customer 42's average monthly spend over the past three years?"
- **Maintain data integrity** — every constraint, every FK check, every transaction boundary protects against incorrect data

### 2.2 OLTP Query Characteristics

| Characteristic | OLTP pattern |
|---|---|
| Rows accessed per query | Few — typically one record or a small set |
| Query complexity | Simple lookups, small JOINs |
| Query frequency | Very high — thousands per minute |
| Data currency | Must be real-time — freshness of seconds |
| Write-to-read ratio | High — many writes relative to reads |
| Typical users | Application systems, operational staff |

### 2.3 OLTP Design Principles

These workload characteristics drive the OLTP design principles covered throughout this course:

**Normalization to 3NF:** Each fact stored once. Updates to one row propagate correctly everywhere. No redundancy to maintain during the many small transactions.

**Many tables with targeted indexes:** Small tables that can be efficiently filtered and joined. Indexes on FK columns and commonly filtered columns ensure single-row lookups are fast.

**Strict constraints:** Every business rule enforced at the database layer. A transaction that violates a constraint is rejected before it can corrupt the system of record.

**Short transactions:** OLTP transactions lock rows briefly (one order, one customer update), released quickly so other transactions can proceed.

---

## 3. OLAP: Designed for Analysis

### 3.1 Characteristics of OLAP Workloads

An OLAP database (which includes data warehouses and data marts) is designed to:

- **Answer large, complex aggregation queries** — "what was total revenue by territory by product category for each quarter of 2023 and 2024?"
- **Support historical analysis** — data is typically accumulated over years; queries must scan millions of rows efficiently
- **Serve a smaller number of analytical users** — analysts, managers, BI tools — rather than hundreds of operational users simultaneously
- **Tolerate slightly stale data** — loaded nightly or hourly from the OLTP; real-time freshness is not required

### 3.2 OLAP Query Characteristics

| Characteristic | OLAP pattern |
|---|---|
| Rows accessed per query | Many — thousands to millions |
| Query complexity | Complex aggregations across multiple dimensions |
| Query frequency | Low to moderate — dozens per hour |
| Data currency | Acceptable latency — hours to one day |
| Write-to-read ratio | Very low — data loaded by batch ETL, read constantly |
| Typical users | Analysts, managers, BI tools (Power BI, Tableau) |

### 3.3 OLAP Design Principles

The OLAP workload characteristics drive opposite design decisions from OLTP:

**Denormalization:** Wide, flat tables that avoid joins during queries. A customer dimension contains `CustomerName`, `CityName`, `ProvinceName`, `CountryName`, and `SalesTerritory` in one row — data that in the OLTP spans four tables. This denormalization means analytical queries need fewer joins.

**Fewer, simpler tables:** A star schema with one fact table and five to ten dimension tables is far simpler to query than a 3NF schema with 30–50 tables.

**Optimised for reads:** Generous indexing (including columnstore indexes for large fact tables), no FK enforcement constraints that would slow query planning, no write-intensive update operations.

**Historical data:** The DW accumulates facts over time. The OLTP may only retain current state (active customers, current prices); the DW retains everything that ever happened.

---

## 4. The Design Consequences of Different Purposes

The contrast between OLTP and OLAP design is most visible when you look at the same business concept in both systems.

### 4.1 The Customer: OLTP vs DW

**In `CabotTrailOutdoor` (OLTP):**
```sql
-- Customer in OLTP: linked to geography through normalised hierarchy
SELECT
    c.CustomerID,
    c.CustomerName,
    ci.CityName,
    sp.StateProvinceName,
    co.CountryName
FROM    Sales.Customers c
INNER JOIN Application.Cities ci         ON ci.CityID            = c.DeliveryCityID
INNER JOIN Application.StateProvinces sp ON sp.StateProvinceID   = ci.StateProvinceID
INNER JOIN Application.Countries co      ON co.CountryID         = sp.CountryID
WHERE   c.CustomerID = 42;
-- 3 joins to get customer geography — 3NF design
```

**In `CabotTrailOutdoorDW` (Data Warehouse):**
```sql
-- Customer in DW: all geography attributes in one wide row
SELECT
    CustomerKey,
    CustomerID,
    CustomerName,
    CityName,
    ProvinceName,
    ProvinceCode,
    CountryName,
    SalesTerritory
FROM    Dimension.DimCustomer
WHERE   CustomerID = 42;
-- No joins needed — denormalised for query simplicity
```

The DW `DimCustomer` is deliberately in violation of 3NF. `ProvinceName` depends on `ProvinceCode` which depends on `CityName` which depends on `CustomerID`. There are transitive dependencies. This is not a mistake — it is intentional denormalization to eliminate joins from analytical queries.

### 4.2 The Sale: OLTP vs DW

**In `CabotTrailOutdoor` (OLTP):** A sale spans `Orders`, `OrderLines`, `Invoices`, `InvoiceLines`, and references `Customers`, `Products`, `Employees`, `DeliveryMethods`, `Cities`.

**In `CabotTrailOutdoorsSales` (Data Mart):** A sale is one row in `fact.Sales` with 25 pre-computed columns — all joins resolved, all measures pre-calculated, all FK lookups replaced with surrogate key references to five dimension tables.

The ETL pipeline performs the transformation between these two representations nightly — this is the fundamental purpose of ETL.

### 4.3 The Design Decision Matrix

| Design question | OLTP answer | OLAP answer |
|---|---|---|
| Normalise to 3NF? | Yes — mandatory | No — deliberately denormalised |
| How many tables? | Many (20–50+) | Few (5–15 per star schema) |
| Fact table exists? | No — data is spread across entities | Yes — central fact table with pre-computed measures |
| Store historical versions? | Generally no (current state) | Yes — multiple snapshots |
| Index aggressively? | Moderately (write overhead) | Aggressively (reads dominate) |
| FK constraints on fact table? | N/A | Usually not (ETL ensures integrity) |
| Primary consumer? | Applications, operational users | BI tools, analysts |

---

## 5. The Star Schema: OLAP's Core Structure

The **star schema** is the dominant design pattern for OLAP systems. It was formalised by Ralph Kimball in the 1990s and remains the standard for analytical database design worldwide.

### 5.1 The Star Structure

```
                    dim.Calendar
                         │
dim.Customer ─────── fact.Sales ─────── dim.Product
                         │
              dim.Employee ── dim.DeliveryMethod
```

One central **fact table** surrounded by multiple **dimension tables** — connected by FK relationships from the fact table to each dimension's PK. The visual structure resembles a star.

### 5.2 Why It Works for Analysis

The star schema is optimised for a specific class of questions: "What was [measure] for [dimension filter] by [dimension group]?"

- "What was **total revenue** (measure) for customers in **Nova Scotia** (dimension filter) **by product category** (dimension group)?"
- "What was the **return rate** (measure) for **Sleeping products** (dimension filter) in **Q2 2024** (dimension filter) **by sales rep** (dimension group)?"

Every analytical question about business performance follows this pattern. The star schema answers all of them with the same two-step structure:
1. Join the fact table to the relevant dimensions
2. Filter and aggregate

The simplicity of this two-step structure is why BI tools can auto-generate correct SQL for star schemas — and why analysts find star schemas intuitive to query.

### 5.3 Snowflake Schema: A Variation

A **snowflake schema** partially normalises the dimension tables. Instead of a flat `dim.Customer` with `CityName` and `ProvinceName` as direct attributes, the snowflake has separate `dim.City` and `dim.Province` tables connected to `dim.Customer` through FKs:

```
dim.Country ──── dim.Province ──── dim.City ──── dim.Customer ──── fact.Sales
```

This recovers some storage efficiency (city names are stored once) at the cost of additional joins in analytical queries. Kimball generally argues against snowflaking: the storage savings are modest, the query complexity increases, and BI tools handle star schemas more naturally than snowflakes.

The CabotTrail DW uses a hybrid: the customer dimension is flattened (star), but products have a `DimProductCategory` table separate from `DimProduct` (a limited snowflake for categories). This is a pragmatic decision — product category details are needed independently and the one additional join is acceptable.

---

## 6. Dimension Tables in Depth

Dimension tables provide the descriptive context for fact measurements. They answer the "who, what, when, where, and how" of every transaction.

### 6.1 Dimension Table Characteristics

- **Wide:** Many descriptive columns — a dimension table with 20–30 columns is normal
- **Small:** Relatively few rows compared to the fact table — `dim.Customer` has 100 rows; `fact.Sales` has 16,359
- **Text-heavy:** Most columns are descriptive strings — names, descriptions, categories, codes
- **Slowly changing:** Dimension data changes infrequently relative to fact data (customers change addresses occasionally; products change categories rarely)
- **Include hierarchy levels:** Multiple levels of a hierarchy (City, Province, Country) stored as flat columns in the same row — the denormalization that enables hierarchy drill-down in a single table

### 6.2 The Date Dimension: The Most Important Dimension

Every data warehouse has a **date dimension** — a table with one row per calendar date, containing every possible date attribute that any analytical query might need:

```sql
-- A few columns from CabotTrailOutdoorDW.Dimension.DimDate
SELECT TOP 5
    DateKey,                -- YYYYMMDD integer (e.g., 20240315)
    FullDate,               -- Actual DATE value
    DayName,                -- 'Friday'
    MonthName,              -- 'March'
    MonthNumber,            -- 3
    CalendarYear,           -- 2024
    CalendarQuarter,        -- 'Q1'
    FiscalYearNumber,       -- 2024 (or 2025 depending on fiscal year start)
    FiscalQuarterNumber,    -- 1
    IsWeekend,              -- 0
    IsHoliday               -- 0
FROM Dimension.DimDate
ORDER BY DateKey;
```

The date dimension is populated once for a wide range of dates (CabotTrail has 4,017 rows spanning 2022–2032) and never updated. Its integer `DateKey` (YYYYMMDD format) is used as the FK on the fact table — allowing date range filters without converting dates to integers at query time.

### 6.3 The Unknown Member

Every dimension table includes an **unknown member** row with surrogate key = 0 and descriptive values of 'Unknown' or similar. This row serves as the FK target for fact rows whose dimension value cannot be resolved:

```sql
-- The unknown member in dim.Customer
SELECT CustomerKey, CustomerID, CustomerName, SalesTerritory
FROM   dim.Customer
WHERE  CustomerKey = 0;
-- Returns: 0, 0, 'Unknown Customer', 'Unknown'
```

When a fact row cannot be matched to a dimension row (a referential integrity gap in the source data), it references `CustomerKey = 0` rather than being dropped. The fact is preserved; it is attributed to the unknown member. Analysts can identify and investigate these rows:

```sql
-- How many sales have an unresolved customer?
SELECT COUNT(*) AS UnknownCustomerSales
FROM   fact.Sales
WHERE  CustomerKey = 0;
-- Should always return 0 in a well-managed DW
```

---

## 7. Fact Tables in Depth

Fact tables store the measurements of business events. They are the largest tables in the DW — typically orders of magnitude larger than any dimension table.

### 7.1 Types of Facts

**Additive facts** can be summed across all dimensions. `LineTotal`, `GrossProfit`, `OrderedQuantity` are additive — they can be summed by territory, by year, by product, or any combination.

**Semi-additive facts** can be summed across some dimensions but not others. `QuantityOnHand` (inventory) is semi-additive: it can be summed across products (total stock) but NOT across time (adding yesterday's and today's stock counts double-counts).

**Non-additive facts** cannot be meaningfully summed across any dimension. `GrossProfitMarginPct` is non-additive — summing percentages produces meaningless results. The correct aggregate is always `SUM(GrossProfit) / SUM(LineTotal)`.

### 7.2 Fact Table Grain

The **grain** is the precise definition of what one row in the fact table represents. Declaring the grain before designing the fact table is the first and most critical design step.

CabotTrail `fact.Sales` grain: **one row per product line on each customer invoice**. Every design decision about which columns and measures belong in the fact table follows from this grain declaration.

A finer grain (one row per item within each line) would allow tracking individual units; a coarser grain (one row per invoice) would lose line-level detail. Kimball's advice: choose the finest grain the source data supports. Aggregates can always be computed upward; detail cannot be recovered downward.

### 7.3 Fact Table Columns

A fact table row contains:
- **Foreign keys** to every dimension (CustomerKey, ProductKey, EmployeeKey, etc.)
- **Degenerate dimensions** — transaction identifiers that have no corresponding dimension table (InvoiceID, OrderID — identifiers that add analytical context without needing a dimension of their own)
- **Additive measures** (LineTotal, GrossProfit, OrderedQuantity, TaxAmount)
- **Pre-computed derived measures** (LineTotalIncludingTax, GrossProfitMarginPct — computed during ETL and stored for query convenience)

```sql
-- fact.Sales column inventory
SELECT  COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM    INFORMATION_SCHEMA.COLUMNS
WHERE   TABLE_SCHEMA = 'fact' AND TABLE_NAME = 'Sales'
ORDER BY ORDINAL_POSITION;
```

---

## 8. Surrogate Keys in the DW Context

Chapter 3 of the ETL textbook introduced surrogate keys as the mechanism that links facts to dimensions in a data warehouse. This section explains *why* surrogate keys are used in the DW even when they are not strictly necessary in the OLTP.

### 8.1 The Problem with Natural Keys in the DW

The OLTP uses `CustomerID INT` as the natural key for customers. Could the DW use `CustomerID` directly as the FK on `fact.Sales`?

The answer is: technically yes, but practically no. Three reasons:

**1. Source system changes.** If a second source system is added (a second retail channel, an acquisition), its customers will have their own ID numbering — which may overlap with the OLTP's CustomerIDs. The DW needs a namespace-independent key that is unique across all sources.

**2. Slowly changing dimensions.** When a customer moves from Nova Scotia to Ontario, SCD Type 2 creates a new dimension row for the same customer — but with different attribute values. The surrogate key distinguishes the two versions (`CustomerKey = 15` for the old NS address, `CustomerKey = 87` for the new ON address). The natural key (`CustomerID = 42`) is the same for both.

**3. Performance.** Surrogate integer keys (4 bytes) are more compact than natural keys (which may be NVARCHAR(20), 40+ bytes). Every FK on every fact row uses the surrogate key — at millions of rows, the storage and join performance difference is significant.

### 8.2 The Surrogate Key as Insulation

The surrogate key insulates the DW from changes in the source system. If CabotTrail's OLTP resets customer numbering (unlikely but possible), the DW's CustomerKey values remain stable — the ETL mapping layer absorbs the change.

---

## 9. Slowly Changing Dimensions

Real-world dimension data changes over time. A customer moves. An employee is promoted. A product is recategorised. How the DW handles these changes is the **Slowly Changing Dimension (SCD)** design problem.

### 9.1 SCD Type 1: Overwrite (No History)

The simplest approach: when an attribute changes, update the dimension row in place. Historical fact rows now reflect the new attribute value — you cannot determine what the attribute was at the time of any specific transaction.

**When appropriate:** For corrections (fixing a typo in a customer name) or for attributes whose historical value is not analytically important.

**CabotTrail:** All dimensions in the CabotTrail DW use SCD Type 1. This is a deliberate design decision that simplifies the ETL — at the cost of historical dimension tracking.

### 9.2 SCD Type 2: Add New Row (Full History)

When an attribute changes, a new row is added to the dimension with the new attribute values. The old row is expired (its `ValidTo` date is set to today; its `IsCurrent` flag is set to 0). The new row gets a new surrogate key.

Historical fact rows continue to reference the old surrogate key — they preserve the attribute values that were current when those facts were recorded.

```sql
-- SCD Type 2 example: customer 42 moves from NS to ON
-- Before the move:
-- CustomerKey=15, CustomerID=42, ProvinceCode='NS', ValidFrom='2022-01-01', ValidTo='9999-12-31', IsCurrent=1

-- After the move, the old row is expired and a new row added:
-- CustomerKey=15, CustomerID=42, ProvinceCode='NS', ValidFrom='2022-01-01', ValidTo='2024-07-14', IsCurrent=0
-- CustomerKey=87, CustomerID=42, ProvinceCode='ON', ValidFrom='2024-07-15', ValidTo='9999-12-31', IsCurrent=1

-- Historical facts reference CustomerKey=15 (NS province at time of sale)
-- New facts reference CustomerKey=87 (ON province)
```

**When appropriate:** When the business needs to answer "what province was this customer in at the time of this sale?" — i.e., when historical accuracy of the dimension attribute matters for analysis.

### 9.3 SCD Type 6: The Hybrid

SCD Type 6 combines Types 1, 2, and 3. Each dimension row stores both the **historical** attribute value (preserved from when the row was created — Type 2) and the **current** attribute value (updated on every load — Type 1). This allows both "what was the province at time of sale?" and "what is the customer's current province?" from a single dimension join.

```sql
-- SCD Type 6 example
-- CustomerKey=15, CustomerID=42, ProvinceCode='NS'(historical), CurrentProvinceCode='ON'(current)
-- CustomerKey=87, CustomerID=42, ProvinceCode='ON'(historical), CurrentProvinceCode='ON'(current)
```

The ETL for Business Intelligence textbook covers all SCD types in detail in Chapter 7. For the database designer, the key point is: **the SCD type choice must be made before the DW schema is finalised** — it affects the dimension table structure.

---

## 10. Conformed Dimensions

A **conformed dimension** is a dimension table used consistently across multiple fact tables and multiple data marts, with the same structure, the same keys, and the same meaning.

### 10.1 Why Conformance Matters

If `dim.Customer` in the Sales data mart defines `SalesTerritory` differently from `dim.Customer` in the Returns data mart, analysts cannot directly compare "Sales by territory" with "Returns by territory" — the territories are defined differently.

Conformed dimensions make cross-mart comparisons possible. When both data marts share the same `DimCustomer` from the DW (same surrogate keys, same attribute definitions), a query joining Sales and Returns data by customer is meaningful.

### 10.2 The DW as the Conformance Layer

The CabotTrail architecture uses the DW (`CabotTrailOutdoorDW`) as the conformance layer. Dimensions are defined once in the DW with canonical structure and values. The data marts load their dimensions from the DW — not independently from the OLTP:

```
OLTP → DW (canonical dimensions defined here) → Data marts (copy from DW)
```

When the Sales mart loads `dim.Customer`, it copies from `Dimension.DimCustomer` in the DW — using the same surrogate keys, the same `SalesTerritory` derivation, the same `ProvinceCode` values. When the Returns mart loads its `dim.Customer`, it copies from the same source.

The result: `CustomerKey = 15` means the same customer in both marts. A query comparing sales and returns by customer can join on CustomerKey safely.

### 10.3 The Bus Matrix

Kimball's **bus matrix** is a planning tool that maps which dimensions are used by which fact tables. It visualises the conformance relationships:

| Fact table | Calendar | Customer | Product | Employee | Supplier | Delivery |
|---|---|---|---|---|---|---|
| FactSales | ✓ | ✓ | ✓ | ✓ | | ✓ |
| FactReturns | ✓ | ✓ | ✓ | ✓ | | |
| FactPurchasing | ✓ | | ✓ | ✓ | ✓ | |
| FactInventory | ✓ | | ✓ | | ✓ | |

Dimensions that appear in multiple fact tables (Calendar, Customer, Product) are the conformed dimensions — they must be defined consistently across all the fact tables that use them.

---

## 11. The ETL Pipeline as the Bridge

The ETL pipeline is the mechanism that transforms data from the normalised OLTP representation to the denormalised OLAP representation. Understanding this transformation bridges the two design philosophies.

### 11.1 What the ETL Does

The ETL pipeline for a single dimension load (`DimCustomer`) performs:

1. **Extract:** Read from `Sales.Customers`, `Application.Cities`, `Application.StateProvinces`, `Application.Countries` in the OLTP
2. **Transform:** Join the four tables, derive `SalesTerritory` from `ProvinceCode`, apply default values for missing data
3. **Load:** Write the combined, denormalised row to `Dimension.DimCustomer` in the DW

The ETL resolves the 3NF normalisation of the OLTP into the intentional denormalization of the DW. This is not a workaround — it is the designed behaviour of the architecture.

### 11.2 The S2T Mapping as the Link

The **source-to-target (S2T) mapping** — introduced in the ETL textbook — documents exactly how each OLTP column maps to each DW column. It is, in effect, the specification that describes how to bridge the two design philosophies for every column in every dimension and fact table.

For the database designer, the S2T mapping represents the answer to the question: "How does the OLTP design choices affect the DW design?" Every normalisation decision in the OLTP (separating customer geography into four tables) creates a corresponding structural gap in the DW S2T mapping (the gap must be resolved by JOINing four source tables during ETL).

### 11.3 The Design Implications

**OLTP design affects ETL complexity.** A well-normalised OLTP requires more complex ETL (more source tables to join, more derived columns to compute). A poorly-normalised OLTP may produce a simpler ETL extract but poorer data quality (redundant data in the source creates update anomaly risk).

**DW design affects query simplicity.** A wide, flat, well-structured DW star schema enables simple analytical queries. A snowflaked or partially normalised DW requires more joins in analytical queries.

The optimal architecture: a well-normalised OLTP (maintains data integrity), connected by a well-designed ETL, to a well-structured star schema DW (supports fast analytics). This is exactly the CabotTrail architecture.

---

## 12. The CabotTrail Architecture End to End

### 12.1 The Complete Pipeline

```
CabotTrailOutdoor (OLTP)
    • 3NF normalised
    • ~30 tables, 4 schemas
    • Designed for transactional correctness
    • Updated constantly (orders, invoices, picks)
         │
         │  ETL (SSIS packages, nightly 2:00 AM)
         │  Source-to-target mapping resolves:
         │    • Geography JOINs (4 tables → 1 flat column set)
         │    • Surrogate key generation
         │    • Measure derivations (UnitCost, GrossProfit)
         │    • SalesTerritory derivation
         ↓
CabotTrailOutdoorDW (Data Warehouse)
    • Dimensional model (star + limited snowflake)
    • 10 dimensions, 6 fact tables
    • Conformed dimensions for cross-mart analysis
    • Updated nightly (full reload)
         │
         │  ETL Layer 2 (datamart load packages)
         │  Simple projection from DW to each mart
         ↓
Data Marts:
    CabotTrailOutdoorsSales      — 6 tables — Sales analysis
    CabotTrailOutdoorsReturns    — 5 tables — Returns analysis
    CabotTrailOutdoorsPurchasing — 5 tables — Purchasing analysis
    CabotTrailOutdoorsInventory  — 5 tables — Inventory analysis
    CabotTrailOutdoorsTransactions — 6 tables — Financial transactions
    CabotTrailExecutive          — 4 tables — Executive summary (Chapter 9)
         │
         ↓
BI Tools (Power BI, Excel, SQL queries)
    • Star schema queries
    • Filtered by dimension attributes
    • Aggregated on fact measures
```

### 12.2 Reading the Architecture as a Designer

The CabotTrail architecture embodies every principle from this course:

- The OLTP reflects Chapters 1–7: proper entities, 3NF normalisation, complete constraints, appropriate indexes
- The ETL embodies Chapter 3's mapping rules (in reverse): denormalising the 3NF structure for OLAP
- The DW reflects Chapter 8: star schema, conformed dimensions, surrogate keys, SCD choices
- The data marts reflect Chapter 8's denormalization philosophy: flattened dimensions, pre-computed measures, read-optimised structure

### 12.3 What Each Database Is Good At

```sql
-- OLTP: operational question — is this order ready to ship?
USE CabotTrailOutdoor;
SELECT o.OrderID, o.IsUndersupplyBackordered, ol.PickedQuantity, ol.Quantity
FROM   Sales.Orders o
INNER JOIN Sales.OrderLines ol ON ol.OrderID = o.OrderID
WHERE  o.OrderID = 1001;

-- DW: analytical question — which sales reps had the highest margins last quarter?
USE CabotTrailOutdoorDW;
SELECT
    e.FullName,
    ROUND(SUM(fs.GrossProfit) / NULLIF(SUM(fs.LineTotal), 0) * 100, 2) AS MarginPct
FROM   Fact.FactSales fs
INNER JOIN Dimension.DimEmployee e ON e.EmployeeKey = fs.EmployeeKey
INNER JOIN Dimension.DimDate d     ON d.DateKey      = fs.OrderDateKey
WHERE  d.FiscalYearNumber = 2024 AND d.FiscalQuarterNumber = 1
GROUP BY e.FullName
ORDER BY MarginPct DESC;

-- Data mart: the same analytical question, even simpler query
USE CabotTrailOutdoorsSales;
SELECT
    emp.FullName,
    ROUND(SUM(fs.GrossProfit) / NULLIF(SUM(fs.LineTotal), 0) * 100, 2) AS MarginPct
FROM   fact.Sales fs
INNER JOIN dim.Employee emp ON emp.EmployeeKey = fs.EmployeeKey
INNER JOIN dim.Calendar cal ON cal.DateKey     = fs.OrderDateKey
WHERE  cal.FiscalYearNumber = 2024 AND cal.FiscalQuarterNumber = 1
GROUP BY emp.FullName
ORDER BY MarginPct DESC;
```

Each database type excels at the questions it was designed for.

---

## 13. Chapter Summary

- **OLTP** (Online Transaction Processing) databases support business operations: many simultaneous users, small transactions, real-time freshness, write-heavy, normalised to 3NF for data integrity.

- **OLAP** (Online Analytical Processing) databases support business analysis: complex aggregations, historical data, read-heavy, denormalised for query performance.

- The **star schema** — one central fact table surrounded by dimension tables — is the standard OLAP design pattern. It optimises for the analytical query pattern: "what was [measure] for [filter] by [group]?"

- **Dimension tables** are wide, text-heavy, and slowly changing. They store descriptive context (who, what, when, where). The date dimension is universal. Every dimension has an unknown member row.

- **Fact tables** store measurements at a defined grain. Additive facts can be summed across all dimensions; semi-additive only across some; non-additive cannot be summed.

- **Surrogate keys** in the DW insulate against source system changes, support SCD Type 2 (multiple versions of the same entity), and improve join performance.

- **SCD types** define how dimension changes are handled: Type 1 (overwrite — no history), Type 2 (new row — full history), Type 6 (hybrid — both current and historical values).

- **Conformed dimensions** are shared consistently across multiple fact tables and data marts, enabling cross-mart analysis. The DW serves as the conformance layer.

- The **ETL pipeline** transforms data from normalised OLTP to denormalised OLAP — resolving JOINs, computing derived measures, generating surrogate keys, and handling SCD logic.

- The CabotTrail architecture embodies the complete pipeline: OLTP → DW → data marts → BI tools, with each layer designed for its specific purpose.

---

## 14. Review Questions

1. Explain why the same customer information is stored in four tables in `CabotTrailOutdoor` (OLTP) but in one wide row in `Dimension.DimCustomer` (DW). Reference both normalization principles and OLAP design principles in your answer.

2. Write two queries that answer the same business question — "what is the total revenue by product category for calendar year 2024?" — one against the OLTP (`CabotTrailOutdoor`) and one against the Sales data mart (`CabotTrailOutdoorsSales`). Compare the number of joins and the structural complexity of each.

3. `fact.Sales` stores `GrossProfitMarginPct` as a column even though it is non-additive. Explain why storing a non-additive measure in a fact table can be acceptable, what risks it introduces, and what documentation should accompany it.

4. CabotTrail decides to implement SCD Type 2 for `DimCustomer` to track province changes. Describe the structural changes required to `dim.Customer` (new columns), what the ETL process must do when a province change is detected, and how historical fact rows will correctly reference the old province after the change.

5. The bus matrix shows that `Calendar`, `Customer`, and `Product` are conformed dimensions — used by multiple fact tables. Explain what "conformed" means technically: what specifically must be true about the `CustomerKey = 15` row in the Sales mart and the `CustomerKey = 15` row in the Returns mart for them to be considered conformed?

6. A new analyst joins the team and asks why `fact.Sales` does not have a FOREIGN KEY constraint from `CustomerKey` to `dim.Customer.CustomerKey`, since CustomerKey is clearly a FK relationship. Provide a complete explanation of why the DW intentionally does not enforce FK constraints on fact tables in most production environments.

7. Compare the index strategy appropriate for `Sales.Orders` (OLTP) versus `fact.Sales` (data mart). Explain the differences in terms of write frequency, query patterns, and the acceptable trade-off between read performance and write overhead.

8. A business analyst suggests: "We should store all our data in one big database and just have both normalised OLTP tables and denormalised analytical views — that way we don't need two databases." Evaluate this suggestion: what are its advantages, what are its risks, and under what circumstances might it be the right approach?

---

## 🔍 Deeper Dive

### Going Further with OLTP and OLAP Design

#### Kimball vs Inmon: The Data Warehouse Architecture Debate

Two competing approaches to data warehouse architecture dominated the field from the 1990s through the 2000s:

**Bill Inmon's Corporate Information Factory (CIF):** Build a centralised, normalised enterprise data warehouse first — a single integrated, normalised (3NF) repository of all enterprise data. Then derive denormalised departmental data marts from the enterprise DW. The DW is the single source of truth; marts are derived.

**Ralph Kimball's Dimensional Modelling / Data Warehouse Bus Architecture:** Build subject-area data marts first, designed around dimensional (star schema) models. Connect them through conformed dimensions (the "bus"). The enterprise DW emerges bottom-up as the collection of integrated marts.

**CabotTrail uses a hybrid approach:** The DW (`CabotTrailOutdoorDW`) is built directly from the OLTP using a dimensional model (Kimball-style), and the subject-area data marts are derived from the DW. This combines Inmon's idea of a centralised DW (before the marts) with Kimball's dimensional modelling methodology.

In practice, most modern enterprise DW implementations blend both approaches. Pure Inmon implementations require large upfront investment and long development cycles; pure Kimball bottom-up approaches can result in marts that are harder to integrate. The key is understanding both enough to make principled design decisions.

Inmon, W. H. (2005). *Building the Data Warehouse* (4th ed.). Wiley.
Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley.

#### The Modern Data Lakehouse

The traditional OLTP/OLAP separation described in this chapter is being disrupted by the **data lakehouse** architecture — a hybrid that combines the scale and flexibility of a data lake with the governance and query performance of a data warehouse.

In a data lakehouse:
- Raw data is stored in cloud object storage (S3, Azure Data Lake, Google Cloud Storage) in open formats (Parquet, Delta Lake, Apache Iceberg)
- A query engine (Databricks, Spark, DuckDB) reads directly from the object storage
- The same data can serve both operational queries and analytical queries
- The ETL pipeline becomes lighter — data is ingested as-is and transformed on read rather than on write (ELT pattern)

The star schema concepts from this chapter apply to the semantic layer of a data lakehouse — the logical model that BI tools and analysts use — even if the physical storage is different.

Tools: Delta Lake, Apache Iceberg, Databricks Lakehouse, Microsoft Fabric, Snowflake.

#### Operational Data Stores (ODS)

An **Operational Data Store (ODS)** is a middle tier between the OLTP and the DW — a normalised, subject-oriented, integrated, volatile store of current operational data from multiple source systems.

The ODS:
- Integrates data from multiple OLTP source systems (in a multi-system enterprise, the OLTP is rarely just one database)
- Is normalised (not dimensional) — it is an integration layer, not an analytical layer
- Contains current data only (no history) — it is operational, not historical
- Supports operational reporting (current-state analysis) that the OLTP cannot easily support (because it requires integrating multiple systems)

The ODS feeds the DW (which adds history and dimensional structure). Many enterprises have an ODS between their OLTP systems and their DW, though smaller organisations like CabotTrail can load directly from OLTP to DW.

---

### References and Further Reading

1. Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling* (3rd ed.). Wiley. — The primary reference for star schema design, dimensional modelling, and SCD types.

2. Inmon, W. H. (2005). *Building the Data Warehouse* (4th ed.). Wiley. — The foundational text for the enterprise data warehouse approach.

3. Kimball, R., & Caserta, J. (2004). *The Data Warehouse ETL Toolkit*. Wiley. — The ETL perspective on bridging OLTP and OLAP.

4. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Chapter 31 covers data warehousing and OLAP in the context of relational database design.

5. Microsoft. (2024). *Azure Synapse Analytics Documentation*. [https://learn.microsoft.com/en-us/azure/synapse-analytics/](https://learn.microsoft.com/en-us/azure/synapse-analytics/) — The cloud-native evolution of the OLTP/OLAP architecture described in this chapter.

6. Databricks. (2024). *Delta Lake Documentation*. [https://delta.io/](https://delta.io/) — The leading open-source data lakehouse format.

7. Dolinger, P. (2027). *ETL for Business Intelligence: A practical guide to data provisioning, dimensional modelling, and pipeline design*. NSCC Institute of Technology. CC BY 4.0. — The companion textbook covering the ETL pipeline that connects OLTP to DW in detail.

---

*Previous chapter: [Chapter 7 — Indexes and Physical Design: Making Queries Fast](../chapter-07-indexes-physical-design/README.md)*

*Next chapter: [Chapter 9 — Putting It Together: Designing CabotTrailExecutive](../chapter-09-cabottrail-executive/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
