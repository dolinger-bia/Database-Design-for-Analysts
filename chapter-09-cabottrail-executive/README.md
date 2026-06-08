# Chapter 9: Putting It Together — Designing CabotTrailExecutive

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

This final chapter executes the complete database design process from start to finish. Every concept from the preceding eight chapters converges here: gathering requirements, identifying entities and relationships, drawing an ERD, normalising the model, mapping to DDL, applying constraints, designing indexes, and considering both OLTP and OLAP perspectives — all applied to one real design problem.

The problem is `CabotTrailExecutive` — an executive summary database that CabotTrail Outdoor's leadership team needs for monthly performance reporting. You have seen this database referenced throughout the course and the prerequisite SQL course. Now you will design it from requirements.

This chapter models the professional workflow: a business narrative leads to questions; questions refine requirements; requirements become a logical model; the logical model becomes a physical schema; the physical schema is populated and verified. The output is a complete, production-ready database design — and the skills to repeat the process for any future design challenge.

By the end of this chapter you will be able to:

- Extract entities, attributes, and relationships from a business narrative
- Surface and resolve design ambiguities through business questions
- Draw a complete ERD for a new database
- Apply the full normalization process to verify the design
- Write complete, production-quality DDL with all constraints
- Design an appropriate index strategy for the schema
- Populate the database from existing CabotTrail data
- Verify the design with reconciliation queries
- Describe how CabotTrailExecutive fits into the broader ETL architecture

---

## Table of Contents

1. [The Design Brief](#1-the-design-brief)
2. [Requirements Analysis](#2-requirements-analysis)
3. [Identifying Entities and Attributes](#3-identifying-entities-and-attributes)
4. [Drawing the ERD](#4-drawing-the-erd)
5. [Normalization Verification](#5-normalization-verification)
6. [Mapping to DDL](#6-mapping-to-ddl)
7. [Constraint Design](#7-constraint-design)
8. [Index Strategy](#8-index-strategy)
9. [The Complete DDL Script](#9-the-complete-ddl-script)
10. [Populating from Existing Data](#10-populating-from-existing-data)
11. [Verification and Reconciliation](#11-verification-and-reconciliation)
12. [Positioning in the ETL Architecture](#12-positioning-in-the-etl-architecture)
13. [Looking Back, Looking Forward](#13-looking-back-looking-forward)
14. [Chapter Summary](#14-chapter-summary)
15. [Review Questions](#15-review-questions)
16. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. The Design Brief

### 1.1 The Business Narrative

> CabotTrail Outdoor's executive team meets monthly to review company performance. Currently, the CFO prepares a report manually by running queries against multiple data marts and assembling the results in Excel. This takes three days and the report is often out of date by the time it circulates.
>
> The executive team needs a dedicated database — `CabotTrailExecutive` — that contains pre-aggregated monthly summary data they can query directly. The database should support the following key questions:
>
> - "What was total revenue by territory and product category for each month?"
> - "How does this month compare to the same month last year?"
> - "Which territories are growing and which are declining?"
> - "What is our gross profit margin by category?"
> - "How many invoices and customers did we serve in each territory?"
>
> The database should be refreshable from existing data — updated monthly from the data marts that already exist. It does not need to track individual transactions; it should contain only monthly summaries at the territory × category × month grain.

### 1.2 Reading the Brief

Before designing anything, read the brief carefully and identify what is explicit and what is ambiguous.

**Explicit:**
- Monthly grain (one row per month per territory per category)
- Key measures: revenue, gross profit margin, invoice count, customer count
- Territory and product category as dimensions
- Time comparison needed (year-over-year)

**Ambiguous:**
- Does "territory" mean `SalesTerritory` (derived) or `ProvinceCode` (source)? If CabotTrail opens in a new territory, does it appear immediately or need to be pre-defined?
- Does "category" mean the 13 existing product categories? Are all 13 included, or only major ones?
- Does "customer count" mean distinct customers who placed orders, or total customer records?
- Should the database be refreshable incrementally (append new months) or fully replaced each month?
- Does "same month last year" mean the calculation should be pre-computed, or computed at query time?

These questions must be answered before the design can be finalised. The next section shows how.

---

## 2. Requirements Analysis

A good database designer does not proceed on ambiguous requirements. This section demonstrates the question-and-answer process that resolves ambiguities.

### 2.1 Clarifying Questions and Decisions

**Q: What is the territory definition?**
*A:* Use `SalesTerritory` as derived in `dim.Customer` — the 10-value derived field ('Atlantic — Nova Scotia', 'Ontario', etc.). New territories will be added to the derivation logic and the database will pick them up on the next refresh.

**Q: What categories are included?**
*A:* All 13 product categories currently in `dim.Product`. The database should accommodate new categories without structural change.

**Q: What does "customer count" mean?**
*A:* Distinct customers who placed at least one order in that month, in that territory. Not total customer records.

**Q: Incremental or full refresh?**
*A:* Full replacement monthly — truncate and reload from the current state of the data marts. The CFO wants the cleanest possible data; partial updates would complicate verification.

**Q: Should year-over-year be pre-computed?**
*A:* No — the query can compute it at runtime by joining the same summary table to itself with a one-year offset. Pre-computing it would either require storing an extra column (and recomputing it on every refresh) or creating a separate view. A view is the better approach.

**Q: Is a calendar/period dimension needed?**
*A:* Yes — a separate `Period` table with CalendarYear and MonthNumber as the PK, plus MonthName and other calendar attributes, will make queries cleaner than storing year and month as plain integers in the summary table.

**Q: Should the database include all time or only recent data?**
*A:* All available history — 2022 through the current month. The database must be able to support year-over-year comparisons going back to the first full year of data.

### 2.2 The Refined Requirements

After clarification, the requirements are:

1. **Grain:** One row per (SalesTerritory, CategoryName, CalendarYear, MonthNumber)
2. **Measures per row:** TotalRevenue, TotalGrossProfit, InvoiceCount, CustomerCount, SalesLineCount
3. **Dimensions:** Period (CalendarYear + MonthNumber), SalesTerritory (string, from DW), ProductCategory (CategoryName)
4. **History:** All available data (2022–current)
5. **Refresh:** Monthly full replacement (TRUNCATE + INSERT)
6. **YoY comparison:** Delivered as a view, not stored columns
7. **No individual transaction data** — summaries only

---

## 3. Identifying Entities and Attributes

### 3.1 Entities from the Requirements

Reading the refined requirements for nouns that have multiple instances:

- **Period** — each unique (CalendarYear, MonthNumber) combination; has MonthName and other calendar attributes
- **Territory** — each unique SalesTerritory value; could have descriptive attributes
- **ProductCategory** — each unique CategoryName; could have margin tier or description attributes
- **MonthlySummary** — the central fact: one row per (Period, Territory, Category) with measures

### 3.2 Is Territory a Full Entity?

Should `SalesTerritory` be a separate table with its own attributes, or just a column in the summary?

Arguments for a separate Territory table:
- Allows adding territory descriptors (regional manager, geographic description) without changing the summary table
- Makes territory name changes manageable (update once, reflects everywhere)
- More normalized

Arguments against:
- The territory is a derived string value, not a business entity with independent existence
- A Territory table would have very few rows (10) and no attributes beyond the name
- Adding a JOIN to a 10-row lookup table for a summary database adds complexity without much value

**Decision:** Keep `SalesTerritory` as a column in the summary table. Its cardinality (10 values) is small enough that a separate table adds more complexity than value. The period and category are the true independent entities.

### 3.3 The Entity Set

Three entities in the final design:

**Period** — calendar period for grouping:
- PeriodKey (PK, surrogate)
- CalendarYear (SMALLINT)
- MonthNumber (TINYINT)
- MonthName (NVARCHAR)
- YearMonthLabel (derived: e.g., 'Mar 2024')
- FiscalYearNumber (SMALLINT)
- FiscalQuarterNumber (TINYINT)
- QuarterLabel (e.g., 'Q1 2024/25')

**ProductCategory** — product category reference:
- CategoryKey (PK, surrogate)
- CategoryName (NVARCHAR, unique)
- MarginTier (derived: 'High', 'Medium', 'Low' based on typical margins)
- IsActive (BIT — future-proofing for discontinued categories)

**ExecutiveSummary** — the central fact/summary:
- SummaryKey (PK, surrogate)
- PeriodKey (FK → Period)
- CategoryKey (FK → ProductCategory)
- SalesTerritory (NVARCHAR — stored inline, not a FK)
- TotalRevenue (DECIMAL)
- TotalGrossProfit (DECIMAL)
- InvoiceCount (INT)
- CustomerCount (INT)
- SalesLineCount (INT)

---

## 4. Drawing the ERD

### 4.1 The ERD in Text Notation

```
┌───────────────────┐                      ┌───────────────────────┐
│    PERIOD         │                      │   PRODUCT_CATEGORY    │
├───────────────────┤                      ├───────────────────────┤
│ PK PeriodKey      │──|──O<────────────>O──|│ PK CategoryKey        │
│    CalendarYear   │                      │    CategoryName        │
│    MonthNumber    │         ↓            │    MarginTier          │
│    MonthName      │  EXECUTIVE_SUMMARY   │    IsActive            │
│    FiscalYear     │  ────────────────    └───────────────────────┘
│    QuarterLabel   │  PK SummaryKey
└───────────────────┘  FK PeriodKey
                       FK CategoryKey
                          SalesTerritory
                          TotalRevenue
                          TotalGrossProfit
                          InvoiceCount
                          CustomerCount
                          SalesLineCount
```

**Relationships:**
- `Period` to `ExecutiveSummary`: 1:N — one period has many summary rows (one per territory × category combination)
- `ProductCategory` to `ExecutiveSummary`: 1:N — one category has many summary rows (one per period × territory)
- `SalesTerritory` is embedded in `ExecutiveSummary` as a column, not a FK relationship

**Cardinality note:** For any given (PeriodKey, CategoryKey, SalesTerritory) combination, there should be exactly one `ExecutiveSummary` row — enforced by a UNIQUE constraint on those three columns.

### 4.2 Design Decisions Recorded

| Decision | Choice | Rationale |
|---|---|---|
| Territory as separate entity | No — inline string | Too small and too simple to justify a lookup table |
| Period as separate entity | Yes — lookup table | Enables clean calendar attributes without redundancy in summary table |
| Category as separate entity | Yes — lookup table | Allows category descriptors; `MarginTier` has independent meaning |
| SCD handling | None — full monthly replacement | Simplifies ETL; no history within the executive database needed |
| YoY as stored or computed | View — computed at query time | Avoids redundant storage; view is the right tool for derived comparative data |

---

## 5. Normalization Verification

### 5.1 Checking ExecutiveSummary

Let PK = `SummaryKey` (single-column surrogate).

Check for transitive dependencies in non-key attributes:

- `PeriodKey` — describes the summary row (which period). ✓ Direct FK
- `CategoryKey` — describes the summary row (which category). ✓ Direct FK
- `SalesTerritory` — describes the summary row (which territory). ✓ Direct
- `TotalRevenue` — a fact about the (period, category, territory) combination. ✓ Direct
- `TotalGrossProfit` — same. ✓ Direct
- `InvoiceCount`, `CustomerCount`, `SalesLineCount` — same. ✓ Direct

**No transitive dependencies in ExecutiveSummary.** ✓ 3NF.

### 5.2 Checking Period

PK = `PeriodKey`.

- `CalendarYear`, `MonthNumber` — describe the period directly. ✓
- `MonthName` — is it a fact about the period? Yes — MonthName is determined by MonthNumber, but MonthNumber is also a direct attribute of the period. This is a transitive-ish dependency: `MonthNumber → MonthName`. But since `MonthNumber` is part of the logical period identity, not a separate entity, and MonthName adds no redundancy (it is a canonical attribute of months), this is acceptable. In strict 3NF, `MonthName` would belong in a separate `Month` lookup. For a small reference table, the pragmatic choice is to keep it in `Period`.
- `FiscalYearNumber`, `FiscalQuarterNumber` — facts about the period's position in the fiscal calendar. ✓ Direct
- `QuarterLabel` — derived from `FiscalYearNumber` and `FiscalQuarterNumber`. This is a derived attribute. Design choice: compute it as a computed column or store it for query convenience.

**Period is in 3NF with a pragmatic exception for MonthName.** Acceptable for a reference table with 48 rows.

### 5.3 Checking ProductCategory

PK = `CategoryKey`.

- `CategoryName` — describes the category. ✓ Direct
- `MarginTier` — is it a fact about the category? Yes, if it is a derived classification applied per category. It could also be a derived value from average margins in the fact data — in which case it is not a property of the category entity itself, but a computed aggregate.

**Design clarification:** `MarginTier` will be loaded from the ETL as a static classification derived from `dim.Product.CategoryName` in the DW — it is treated as a property of the category in this design, not a computed aggregate. ✓ 3NF.

---

## 6. Mapping to DDL

### 6.1 Applying the Mapping Rules

**Rule 1 (Strong entities → tables):** Three tables: `dim.Period`, `dim.ProductCategory`, `fact.ExecutiveSummary`.

**Rule 2 (Attributes → columns):** Each attribute maps to a column with an appropriate data type.

**Rule 5 (1:N relationships → FK on many side):** `ExecutiveSummary` carries `PeriodKey` and `CategoryKey` as FKs.

**No composite PK:** `SummaryKey` is a surrogate PK for `ExecutiveSummary`. The business uniqueness rule (one row per period × category × territory) is enforced by a UNIQUE constraint.

### 6.2 Data Type Decisions

| Column | Type | Rationale |
|---|---|---|
| PeriodKey, CategoryKey, SummaryKey | `INT NOT NULL IDENTITY(1,1)` | Surrogate PKs |
| CalendarYear, FiscalYearNumber | `SMALLINT NOT NULL` | 2-byte; year range 2000–2100 easily fits |
| MonthNumber, FiscalQuarterNumber | `TINYINT NOT NULL` | 1-byte; 1–12 or 1–4 |
| MonthName, QuarterLabel, CategoryName | `NVARCHAR(20)` | Short descriptive labels |
| SalesTerritory | `NVARCHAR(60) NOT NULL` | Matches dim.Customer.SalesTerritory |
| TotalRevenue, TotalGrossProfit | `DECIMAL(18,2) NOT NULL DEFAULT 0` | Financial amounts; 2 decimal places |
| InvoiceCount, CustomerCount, SalesLineCount | `INT NOT NULL DEFAULT 0` | Counts; non-negative |
| MarginTier | `NVARCHAR(10) NOT NULL` | 'High', 'Medium', 'Low' |
| IsActive | `BIT NOT NULL DEFAULT 1` | Boolean flag |

---

## 7. Constraint Design

For each table, identify all constraints needed to enforce the business rules:

**`dim.Period` constraints:**
- PK on PeriodKey
- UNIQUE on (CalendarYear, MonthNumber) — each calendar month appears once
- CHECK: MonthNumber BETWEEN 1 AND 12
- CHECK: CalendarYear BETWEEN 2000 AND 2100
- CHECK: FiscalQuarterNumber BETWEEN 1 AND 4

**`dim.ProductCategory` constraints:**
- PK on CategoryKey
- UNIQUE on CategoryName — no duplicate category names
- CHECK: MarginTier IN ('High', 'Medium', 'Low')
- CHECK: IsActive IN (0, 1) — redundant with BIT but explicit

**`fact.ExecutiveSummary` constraints:**
- PK on SummaryKey
- UNIQUE on (PeriodKey, CategoryKey, SalesTerritory) — one row per combination
- FK: PeriodKey → dim.Period.PeriodKey
- FK: CategoryKey → dim.ProductCategory.CategoryKey
- CHECK: TotalRevenue >= 0
- CHECK: TotalGrossProfit >= 0 (could also be negative if returns exceed sales — clarify with business)
- CHECK: InvoiceCount >= 0, CustomerCount >= 0, SalesLineCount >= 0

---

## 8. Index Strategy

`CabotTrailExecutive` is a read-heavy analytical database. The ETL loads it monthly (full reload); analysts query it constantly. The index strategy favours reads.

**`dim.Period`:**
- Clustered on PeriodKey (PK default)
- UNIQUE non-clustered on (CalendarYear, MonthNumber) — supports queries filtering by year and month
- Non-clustered on (FiscalYearNumber, FiscalQuarterNumber) — supports fiscal period queries

**`dim.ProductCategory`:**
- Clustered on CategoryKey (PK default)
- UNIQUE non-clustered on CategoryName — supports joins and lookups by name

**`fact.ExecutiveSummary`:**
- Clustered on SummaryKey (PK default)
- UNIQUE non-clustered on (PeriodKey, CategoryKey, SalesTerritory) — enforces grain, also supports the most common query pattern
- Non-clustered on (PeriodKey) INCLUDE (SalesTerritory, TotalRevenue, TotalGrossProfit) — covers time-series queries
- Non-clustered on (SalesTerritory, PeriodKey) INCLUDE (TotalRevenue, TotalGrossProfit) — covers territory trend queries

---

## 9. The Complete DDL Script

```sql
-- ============================================================
-- CabotTrailExecutive Database
-- Executive summary database for monthly performance reporting
-- Author: Patrick Dolinger, NSCC Institute of Technology
-- Version: 1.0
-- Date: 2027-01-06
-- ============================================================

-- Create the database
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = 'CabotTrailExecutive')
BEGIN
    CREATE DATABASE CabotTrailExecutive;
END;
GO

USE CabotTrailExecutive;
GO

-- ── Create schemas ─────────────────────────────────────────
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'dim')
    EXEC ('CREATE SCHEMA dim');
GO
IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'fact')
    EXEC ('CREATE SCHEMA fact');
GO

-- ── Drop existing tables (idempotent) ──────────────────────
DROP TABLE IF EXISTS fact.ExecutiveSummary;
DROP TABLE IF EXISTS dim.Period;
DROP TABLE IF EXISTS dim.ProductCategory;
GO

-- ── dim.Period ─────────────────────────────────────────────
CREATE TABLE dim.Period
(
    PeriodKey           INT             NOT NULL IDENTITY(1,1),
    CalendarYear        SMALLINT        NOT NULL,
    MonthNumber         TINYINT         NOT NULL,
    MonthName           NVARCHAR(10)    NOT NULL,
    YearMonthLabel      NVARCHAR(10)    NOT NULL,   -- e.g., 'Mar 2024'
    FiscalYearNumber    SMALLINT        NOT NULL,
    FiscalQuarterNumber TINYINT         NOT NULL,
    QuarterLabel        NVARCHAR(12)    NOT NULL,   -- e.g., 'Q1 2024/25'
    IsCurrentMonth      BIT             NOT NULL DEFAULT 0,

    CONSTRAINT PK_Period                PRIMARY KEY (PeriodKey),
    CONSTRAINT UQ_Period_CalYearMonth   UNIQUE (CalendarYear, MonthNumber),
    CONSTRAINT CK_Period_MonthNum       CHECK (MonthNumber BETWEEN 1 AND 12),
    CONSTRAINT CK_Period_CalYear        CHECK (CalendarYear BETWEEN 2000 AND 2100),
    CONSTRAINT CK_Period_FiscalQtr      CHECK (FiscalQuarterNumber BETWEEN 1 AND 4)
);
GO

-- ── dim.ProductCategory ────────────────────────────────────
CREATE TABLE dim.ProductCategory
(
    CategoryKey         INT             NOT NULL IDENTITY(1,1),
    CategoryName        NVARCHAR(60)    NOT NULL,
    MarginTier          NVARCHAR(10)    NOT NULL DEFAULT 'Medium',
    IsActive            BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_ProductCategory           PRIMARY KEY (CategoryKey),
    CONSTRAINT UQ_ProductCategory_Name      UNIQUE (CategoryName),
    CONSTRAINT CK_ProductCategory_Margin    CHECK (MarginTier IN ('High', 'Medium', 'Low'))
);
GO

-- ── fact.ExecutiveSummary ──────────────────────────────────
CREATE TABLE fact.ExecutiveSummary
(
    SummaryKey          INT             NOT NULL IDENTITY(1,1),
    PeriodKey           INT             NOT NULL,
    CategoryKey         INT             NOT NULL,
    SalesTerritory      NVARCHAR(60)    NOT NULL,
    TotalRevenue        DECIMAL(18,2)   NOT NULL DEFAULT 0,
    TotalGrossProfit    DECIMAL(18,2)   NOT NULL DEFAULT 0,
    InvoiceCount        INT             NOT NULL DEFAULT 0,
    CustomerCount       INT             NOT NULL DEFAULT 0,
    SalesLineCount      INT             NOT NULL DEFAULT 0,

    CONSTRAINT PK_ExecutiveSummary          PRIMARY KEY (SummaryKey),
    CONSTRAINT UQ_ExecutiveSummary_Grain    UNIQUE (PeriodKey, CategoryKey, SalesTerritory),
    CONSTRAINT FK_ExecSummary_Period
        FOREIGN KEY (PeriodKey)    REFERENCES dim.Period (PeriodKey),
    CONSTRAINT FK_ExecSummary_Category
        FOREIGN KEY (CategoryKey)  REFERENCES dim.ProductCategory (CategoryKey),
    CONSTRAINT CK_ExecSummary_Revenue
        CHECK (TotalRevenue >= 0),
    CONSTRAINT CK_ExecSummary_InvoiceCount
        CHECK (InvoiceCount >= 0),
    CONSTRAINT CK_ExecSummary_CustomerCount
        CHECK (CustomerCount >= 0),
    CONSTRAINT CK_ExecSummary_SalesLineCount
        CHECK (SalesLineCount >= 0)
);
GO

-- ── Indexes ────────────────────────────────────────────────
CREATE NONCLUSTERED INDEX IX_Period_FiscalYear
    ON dim.Period (FiscalYearNumber, FiscalQuarterNumber);

CREATE NONCLUSTERED INDEX IX_ExecSummary_Period
    ON fact.ExecutiveSummary (PeriodKey)
    INCLUDE (SalesTerritory, TotalRevenue, TotalGrossProfit, InvoiceCount, CustomerCount);

CREATE NONCLUSTERED INDEX IX_ExecSummary_Territory
    ON fact.ExecutiveSummary (SalesTerritory, PeriodKey)
    INCLUDE (TotalRevenue, TotalGrossProfit, InvoiceCount);
GO

-- ── YoY View ───────────────────────────────────────────────
CREATE OR ALTER VIEW fact.vw_ExecutiveSummaryYoY AS
SELECT
    es.SummaryKey,
    p.CalendarYear,
    p.MonthNumber,
    p.MonthName,
    p.YearMonthLabel,
    p.FiscalYearNumber,
    p.QuarterLabel,
    cat.CategoryName,
    cat.MarginTier,
    es.SalesTerritory,
    es.TotalRevenue,
    es.TotalGrossProfit,
    es.InvoiceCount,
    es.CustomerCount,
    es.SalesLineCount,
    -- Gross profit margin percentage (computed, non-additive)
    ROUND(
        es.TotalGrossProfit / NULLIF(es.TotalRevenue, 0) * 100,
    2)                                      AS GrossMarginPct,
    -- Same period last year: join back to the same table 12 months ago
    prior.TotalRevenue                      AS PriorYearRevenue,
    prior.TotalGrossProfit                  AS PriorYearGrossProfit,
    -- Year-over-year revenue change
    ROUND(
        (es.TotalRevenue - ISNULL(prior.TotalRevenue, 0))
        / NULLIF(prior.TotalRevenue, 0) * 100,
    1)                                      AS YoYRevenueChangePct,
    -- Revenue growth flag
    CASE
        WHEN prior.TotalRevenue IS NULL     THEN 'New'
        WHEN es.TotalRevenue > prior.TotalRevenue THEN 'Growth'
        WHEN es.TotalRevenue < prior.TotalRevenue THEN 'Decline'
        ELSE                                     'Flat'
    END                                     AS YoYTrend
FROM    fact.ExecutiveSummary es
INNER JOIN dim.Period p      ON p.PeriodKey   = es.PeriodKey
INNER JOIN dim.ProductCategory cat ON cat.CategoryKey = es.CategoryKey
-- Self-join: find the same territory+category combination 12 months prior
LEFT JOIN (
    SELECT
        es2.CategoryKey,
        es2.SalesTerritory,
        p2.CalendarYear,
        p2.MonthNumber,
        es2.TotalRevenue,
        es2.TotalGrossProfit
    FROM   fact.ExecutiveSummary es2
    INNER JOIN dim.Period p2 ON p2.PeriodKey = es2.PeriodKey
) prior
    ON  prior.CategoryKey   = es.CategoryKey
    AND prior.SalesTerritory = es.SalesTerritory
    AND prior.CalendarYear  = p.CalendarYear - 1
    AND prior.MonthNumber   = p.MonthNumber;
GO
```

---

## 10. Populating from Existing Data

With the schema in place, populate `CabotTrailExecutive` from the existing data in `CabotTrailOutdoorDW`.

### 10.1 Populate dim.Period

```sql
USE CabotTrailExecutive;

-- Load all distinct year/month combinations from the DW fact table
INSERT INTO dim.Period
(
    CalendarYear, MonthNumber, MonthName, YearMonthLabel,
    FiscalYearNumber, FiscalQuarterNumber, QuarterLabel, IsCurrentMonth
)
SELECT DISTINCT
    d.YearNumber            AS CalendarYear,
    d.MonthNumber,
    d.MonthName,
    -- Build the label: 'Mar 2024'
    LEFT(d.MonthName, 3) + ' ' + CAST(d.YearNumber AS NVARCHAR(4)) AS YearMonthLabel,
    d.FiscalYearNumber,
    d.FiscalQuarterNumber,
    -- Build the quarter label: 'Q1 2024/25'
    'Q' + CAST(d.FiscalQuarterNumber AS NVARCHAR(1))
        + ' ' + CAST(d.FiscalYearNumber AS NVARCHAR(4))
        + '/' + RIGHT(CAST(d.FiscalYearNumber + 1 AS NVARCHAR(4)), 2) AS QuarterLabel,
    -- Flag the current month
    CASE WHEN d.YearNumber = YEAR(GETDATE())
          AND d.MonthNumber = MONTH(GETDATE())
         THEN 1 ELSE 0 END AS IsCurrentMonth
FROM    CabotTrailOutdoorDW.Dimension.DimDate d
WHERE   d.YearNumber BETWEEN 2022 AND YEAR(GETDATE())
GROUP BY d.YearNumber, d.MonthNumber, d.MonthName,
         d.FiscalYearNumber, d.FiscalQuarterNumber
ORDER BY d.YearNumber, d.MonthNumber;
```

### 10.2 Populate dim.ProductCategory

```sql
-- Load product categories from the DW
INSERT INTO dim.ProductCategory (CategoryName, MarginTier, IsActive)
SELECT
    pc.ProductCategoryName  AS CategoryName,
    -- MarginTier derived from known category margin rates
    CASE
        WHEN pc.ProductCategoryName IN
            ('Accessories','Hydration','Navigation','Safety and First Aid')
        THEN 'High'
        WHEN pc.ProductCategoryName IN
            ('Climbing and Rappelling','Cooking and Food',
             'Apparel - Outerwear','Apparel - Tops','Backpacks and Bags')
        THEN 'Medium'
        ELSE 'Low'
    END                     AS MarginTier,
    1                       AS IsActive
FROM    CabotTrailOutdoorDW.Dimension.DimProductCategory pc
WHERE   pc.ProductCategoryID <> 0   -- Exclude unknown member
ORDER BY pc.ProductCategoryName;
```

### 10.3 Populate fact.ExecutiveSummary

```sql
-- The main summary load: aggregate from DW FactSales
-- This is the source-to-target script for the executive summary
INSERT INTO fact.ExecutiveSummary
(
    PeriodKey, CategoryKey, SalesTerritory,
    TotalRevenue, TotalGrossProfit,
    InvoiceCount, CustomerCount, SalesLineCount
)
SELECT
    p.PeriodKey,
    cat.CategoryKey,
    cust.SalesTerritory,
    ROUND(SUM(fs.LineTotal),        2) AS TotalRevenue,
    ROUND(SUM(fs.GrossProfit),      2) AS TotalGrossProfit,
    COUNT(DISTINCT fs.InvoiceID)       AS InvoiceCount,
    COUNT(DISTINCT fs.CustomerKey)     AS CustomerCount,
    COUNT(*)                           AS SalesLineCount
FROM    CabotTrailOutdoorDW.Fact.FactSales fs
-- Join to DW dimensions
INNER JOIN CabotTrailOutdoorDW.Dimension.DimDate d
    ON d.DateKey = fs.OrderDateKey
INNER JOIN CabotTrailOutdoorDW.Dimension.DimCustomer cust
    ON cust.CustomerKey = fs.CustomerKey
    AND cust.IsCurrent  = 1      -- Current version of customer (SCD Type 1 — all current)
INNER JOIN CabotTrailOutdoorDW.Dimension.DimProduct dp
    ON dp.ProductKey = fs.ProductKey
    AND dp.ProductID <> 0
INNER JOIN CabotTrailOutdoorDW.Dimension.DimProductCategory dpc
    ON dpc.ProductCategoryKey = dp.ProductCategoryKey
    AND dpc.ProductCategoryID <> 0
-- Join to Executive dimension tables to get surrogate keys
INNER JOIN dim.Period p
    ON p.CalendarYear = d.YearNumber
    AND p.MonthNumber = d.MonthNumber
INNER JOIN dim.ProductCategory cat
    ON cat.CategoryName = dpc.ProductCategoryName
-- Exclude unknown members and 2026 open orders
WHERE   fs.CustomerKey <> 0
AND     fs.ProductKey  <> 0
AND     d.YearNumber   < 2026     -- Exclude open/incomplete year
GROUP BY
    p.PeriodKey,
    cat.CategoryKey,
    cust.SalesTerritory;
```

---

## 11. Verification and Reconciliation

After populating the database, verify that the data is correct and complete.

### 11.1 Row Count Verification

```sql
USE CabotTrailExecutive;

-- How many periods, categories, territories, and summary rows?
SELECT 'dim.Period'          AS TableName, COUNT(*) AS Rows FROM dim.Period
UNION ALL
SELECT 'dim.ProductCategory', COUNT(*) FROM dim.ProductCategory
UNION ALL
SELECT 'fact.ExecutiveSummary', COUNT(*) FROM fact.ExecutiveSummary;

-- Expected:
-- dim.Period: 48 rows (12 months × 4 years)
-- dim.ProductCategory: 13 rows
-- fact.ExecutiveSummary: up to 48 × 13 × 10 = 6,240 rows
--   (fewer if some territory/category combinations have no sales in some months)
```

### 11.2 Revenue Reconciliation

```sql
-- Total revenue in ExecutiveSummary must match total revenue in the DW FactSales
SELECT
    'DW FactSales'              AS Source,
    ROUND(SUM(LineTotal), 2)    AS TotalRevenue
FROM    CabotTrailOutdoorDW.Fact.FactSales
WHERE   CustomerKey <> 0
AND     ProductKey  <> 0
AND     OrderDateKey / 10000 < 2026   -- Match the WHERE filter used in the load

UNION ALL

SELECT
    'CabotTrailExecutive',
    ROUND(SUM(TotalRevenue), 2)
FROM    fact.ExecutiveSummary;

-- Both values must be equal (within $0.01 tolerance for rounding)
```

### 11.3 Testing the YoY View

```sql
-- Test the year-over-year view
SELECT TOP 20
    YearMonthLabel,
    CategoryName,
    SalesTerritory,
    TotalRevenue,
    PriorYearRevenue,
    YoYRevenueChangePct,
    YoYTrend
FROM    fact.vw_ExecutiveSummaryYoY
WHERE   CalendarYear = 2024
ORDER BY CalendarYear, MonthNumber, CategoryName, SalesTerritory;
```

### 11.4 The Executive Report Query

The executive report query is now a simple SELECT against the view — exactly what the CFO needs:

```sql
-- Monthly territory performance: YTD 2024 vs prior year
SELECT
    QuarterLabel,
    SalesTerritory,
    SUM(TotalRevenue)               AS Revenue,
    SUM(PriorYearRevenue)           AS PriorYearRevenue,
    ROUND(
        (SUM(TotalRevenue) - SUM(ISNULL(PriorYearRevenue, 0)))
        / NULLIF(SUM(PriorYearRevenue), 0) * 100,
    1)                              AS YoYChangePct,
    ROUND(
        SUM(TotalGrossProfit) / NULLIF(SUM(TotalRevenue), 0) * 100,
    2)                              AS MarginPct
FROM    fact.vw_ExecutiveSummaryYoY
WHERE   CalendarYear = 2024
GROUP BY QuarterLabel, FiscalYearNumber, FiscalQuarterNumber, SalesTerritory
ORDER BY FiscalYearNumber, FiscalQuarterNumber, SalesTerritory;
```

This is the query the CFO was previously assembling manually over three days. With `CabotTrailExecutive`, it returns in under a second.

---

## 12. Positioning in the ETL Architecture

### 12.1 Where CabotTrailExecutive Fits

```
CabotTrailOutdoor (OLTP)
        ↓ ETL Layer 1 (nightly, 2:00 AM)
CabotTrailOutdoorDW (Data Warehouse)
        ↓ ETL Layer 2 (nightly, 2:05 AM)
Data Marts (Sales, Returns, Purchasing, etc.)
        ↓ ETL Layer 3 (monthly, 1st of month, 3:00 AM)
CabotTrailExecutive
```

`CabotTrailExecutive` is loaded from the DW after the nightly DW load completes. The monthly load runs on the first of each month:

1. **Truncate** all three tables
2. **Reload** `dim.Period` from the DW calendar
3. **Reload** `dim.ProductCategory` from the DW product dimension
4. **Reload** `fact.ExecutiveSummary` from the DW fact table
5. **Verify** revenue reconciliation
6. **Alert** on failure

### 12.2 The SSIS Package Design

In DBAS 2103, you will build the SSIS package for this load. Its structure follows the master/child package pattern from the ETL textbook:

```
Master_Executive_Load.dtsx
    ↓
[Execute SQL: TRUNCATE dim.Period]
[Execute SQL: TRUNCATE dim.ProductCategory]
[Execute SQL: TRUNCATE fact.ExecutiveSummary]
    ↓
[Data Flow: Load dim.Period]
    ↓
[Data Flow: Load dim.ProductCategory]
    ↓
[Data Flow: Load fact.ExecutiveSummary]
    ↓
[Execute SQL: Reconciliation test — Revenue match]
    ↓
[On failure: Send alert email]
```

The T-SQL source-to-target scripts in section 10 of this chapter are the direct specification for what each Data Flow Task implements.

---

## 13. Looking Back, Looking Forward

### 13.1 What This Course Covered

`CabotTrailExecutive` was designed using every concept in this course:

| Chapter | Skill applied in this chapter |
|---|---|
| 1 — The Relational Model | Understanding entities, attributes, relationships; the designer's perspective |
| 2 — ERD | Drawing the three-entity ERD; recording design decisions |
| 3 — Mapping to DDL | Seven mapping rules applied; FK placement; surrogate PKs |
| 4 — 1NF and 2NF | Verifying no multi-valued attributes, no partial dependencies |
| 5 — 3NF | Verifying no transitive dependencies; pragmatic MonthName decision |
| 6 — Constraints | Named constraints for every business rule |
| 7 — Indexes | Read-optimised index strategy; covering indexes for common patterns |
| 8 — OLTP vs OLAP | Positioned as OLAP; denormalised SalesTerritory; summary grain |

### 13.2 The Three Courses Together

This course is the middle of a three-course sequence:

**DBAS 5010 (SQL for Analysts)** taught you to *read* databases — SELECT, JOIN, GROUP BY, window functions. You worked with the CabotTrail databases as existing structures.

**DBAS 2010 (Database Design for Analysts)** taught you to *design* databases — ERDs, normalization, constraints, indexes, OLTP vs OLAP. You learned why the CabotTrail databases are the way they are, and built `CabotTrailExecutive`.

**DBAS 2103 (Data Provisioning with ETL)** will teach you to *connect* databases — the SSIS pipelines that move data from OLTP through DW to data marts to `CabotTrailExecutive`. The designs you built in this course are the targets you will load in the next.

The three courses together give you a complete picture of the data lifecycle: design the structures, query the data, build the pipelines that keep the structures current.

---

## 14. Chapter Summary

- The complete database design process proceeds: **narrative → clarifying questions → refined requirements → entity identification → ERD → normalization verification → DDL mapping → constraints → indexes → population → verification**.

- **Requirements analysis** is not optional — ambiguities in the brief (territory definition, customer count meaning, refresh strategy, YoY pre-computation) must be resolved before design begins.

- `CabotTrailExecutive` has three tables: `dim.Period`, `dim.ProductCategory`, and `fact.ExecutiveSummary`, connected by two 1:N relationships. SalesTerritory is stored inline rather than as a separate entity.

- The **ERD** documents three entities with two 1:N relationships. Design decisions (territory inline, period as entity, YoY as view) are recorded and justified.

- **Normalization verification** confirms 3NF with one pragmatic exception (MonthName in dim.Period), explicitly documented.

- The **complete DDL** includes all three tables, all constraints (named), indexes, and the YoY view — 100+ lines of production-quality SQL.

- **Population** uses cross-database INSERT INTO ... SELECT from `CabotTrailOutdoorDW`, with three sequential loads in dependency order (Period first, Category second, Summary last).

- **Reconciliation** verifies row counts and total revenue match between source (DW) and target (Executive).

- `CabotTrailExecutive` feeds into the ETL architecture as Layer 3 — loaded from the DW monthly, positioned as the terminal layer before BI tools.

---

## 15. Review Questions

1. The design brief asked for "customer count" — the number of customers served in each territory/category/month. The design uses `COUNT(DISTINCT fs.CustomerKey)` in the population query. Describe a scenario where this count could be misleading, and propose an alternative definition that would be clearer for executive reporting.

2. `SalesTerritory` was stored as an inline column in `fact.ExecutiveSummary` rather than as a separate entity with its own table. Write the alternative DDL for a `dim.Territory` table and the modified `fact.ExecutiveSummary` with a FK to it. Then justify the original decision (inline) vs the alternative (separate table) — under what future requirement would the separate table become the better choice?

3. The `vw_ExecutiveSummaryYoY` view computes year-over-year revenue by joining the summary table to itself. Explain the JOIN logic: what columns are joined, why a LEFT JOIN is used rather than INNER JOIN, and what a NULL `PriorYearRevenue` means in the context of executive reporting.

4. Run the revenue reconciliation query from section 11.2 against your loaded `CabotTrailExecutive` database. If the totals do not match exactly (within $0.01), identify three possible causes and show the diagnostic query you would run for each.

5. The `dim.Period` table stores `MonthName` even though it is transitively dependent on `MonthNumber`. Explain why this is an acceptable pragmatic exception to strict 3NF for this specific case, and describe the conditions under which this pragmatism would become a maintenance problem.

6. Design the monthly SSIS package structure (in text notation, following the process flow diagram format from Chapter 6 of the ETL book) for `Master_Executive_Load.dtsx`. Include all Control Flow tasks, the error paths, and the reconciliation steps. Which package from DBAS 2103 is this most similar to?

7. A month after deploying `CabotTrailExecutive`, a new territory is added to CabotTrail's operations: 'Northern Canada' (covering Yukon, Northwest Territories, and Nunavut). Describe every place in the architecture — from the OLTP through the DW to the Executive database — that would need to change to support the new territory, and in what order those changes must be made.

8. The executive team asks: "Could we have built this in Power BI directly from the data marts, without a separate database?" Evaluate this suggestion technically: what would Power BI need to do to replicate the YoY comparison, what are the limitations, and under what scale or complexity would a dedicated database like `CabotTrailExecutive` become clearly preferable?

---

## 🔍 Deeper Dive

### The Design Process in Professional Practice

#### Agile Database Development

Traditional database design assumes requirements are fully known before design begins. In practice, requirements emerge and change throughout a project. **Agile database development** adapts the design process to accommodate iterative requirements:

- Start with the entities and relationships that are clearly understood; design those first
- Use evolutionary design — add tables and columns incrementally as requirements are clarified
- Version control the schema (as SQL migration scripts) so changes are tracked
- Use non-destructive changes where possible (add columns as nullable before constraining)

Tools like **Liquibase** and **Flyway** manage database schema migrations in version-controlled environments — tracking which migration scripts have been applied to each environment and applying new ones in sequence.

The tension between "design everything upfront" and "evolve as you go" is resolved by: design what you know clearly, leave designed flexibility where you anticipate change, and use tools that make changes safe and trackable.

#### Dimensional Modelling for the Executive Layer

`CabotTrailExecutive` is a small dimensional model — one fact table and two dimension tables. Kimball's guidance for executive summary databases emphasises:

- **Keep the grain coarse.** Monthly territory × category grain is appropriate for executive reporting; finer grain is for operational or tactical analysis.
- **Pre-compute everything the executives will always need.** Gross margin percentage belongs in the view (or as a stored derived measure) — executives should not have to write `SUM(GP)/SUM(Rev)*100` themselves.
- **Make the schema self-evident.** An executive database should be queryable by a senior analyst who has not read any documentation. Column names, table names, and the YoY view should be self-explaining.

Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley. — Chapter 5 covers enterprise data warehouse bus architecture including summary and executive databases.

#### Database Design Reviews

A professional database design is reviewed before implementation — by a peer developer, a senior DBA, or a business stakeholder, depending on the organisation. Common review checkpoints:

**Correctness review:** Is the ERD accurate? Does the DDL correctly implement the ERD? Are all constraints named? Are all FKs indexed?

**Business review:** Are the entity names and column names recognisable to business stakeholders? Does the grain match the business requirement? Are the measures defined correctly?

**Performance review:** Are the indexes appropriate for the expected query patterns? Are there any non-sargable patterns baked into the schema? Should the clustered index key be reconsidered?

**Standards review:** Do all names follow the established conventions? Are all constraints named? Is the DDL idempotent (safe to re-run)?

For `CabotTrailExecutive`, a business review would confirm: does the executive team agree with the territory definition, the category list, and the monthly grain? A single wrong assumption here invalidates months of data.

---

### References and Further Reading

1. Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley. — The entire book; Chapter 5 specifically covers enterprise DW bus architecture and summary databases.

2. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Chapter 10 covers the end-to-end database design methodology including requirements analysis and review processes.

3. Date, C. J. (2019). *Database Design and Relational Theory: Normal Forms and All That Jazz* (2nd ed.). Apress. — Chapter 12 covers pragmatic normalisation decisions — when and how to deviate from strict forms.

4. Dolinger, P. (2027). *ETL for Business Intelligence: A practical guide to data provisioning, dimensional modelling, and pipeline design*. NSCC Institute of Technology. CC BY 4.0. — Chapter 9 covers the complete ETL pipeline including the executive summary layer.

5. Liquibase. (2024). *Liquibase Documentation*. [https://docs.liquibase.com/](https://docs.liquibase.com/) — Database change management and migration tracking.

6. Microsoft. (2024). *CREATE DATABASE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-transact-sql)

7. Microsoft. (2024). *CREATE VIEW (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/create-view-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-view-transact-sql)

---

*Previous chapter: [Chapter 8 — OLTP vs OLAP: Two Design Philosophies](../chapter-08-oltp-vs-olap/README.md)*

*Return to: [Database Design for Analysts — Table of Contents](../README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
>
> *Thank you for reading. If you use or adapt this material, please attribute as:*
> *Dolinger, P. (2027). Database Design for Analysts: From business requirements to relational structures. NSCC Institute of Technology. CC BY 4.0.*
