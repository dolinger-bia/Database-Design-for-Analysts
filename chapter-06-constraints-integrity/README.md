# Chapter 6: Constraints and Data Integrity — Encoding Business Rules

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Normalization ensures that data is organized correctly — that facts live in the right places and redundancy is eliminated. But organization alone is not enough. A normalized table can still store absurd or impossible values: a negative price, a future hire date for a retired employee, an order quantity of zero, a customer with a credit limit that exceeds any business rationale.

**Constraints** are the database's mechanism for enforcing business rules at the storage layer — rules that must be true for data to be valid. When a constraint is violated, the database rejects the operation before any data is changed. The application cannot bypass it, the ETL process cannot bypass it, and — most importantly — human error cannot bypass it.

This chapter covers every constraint type available in SQL Server, the business rules they encode, and the design decisions behind them. It also covers NULL handling in depth, default values, identity columns, and the catalog queries that reveal what constraints already exist.

By the end of this chapter you will be able to:

- Explain the purpose of each SQL Server constraint type and when to use each
- Write PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK, and DEFAULT constraints with proper naming
- Design CHECK constraints that encode complex business rules
- Explain the difference between NULL and NOT NULL and choose correctly
- Configure IDENTITY columns appropriately
- Query the system catalog to discover existing constraints
- Describe the cascading referential integrity options and choose among them
- Explain computed columns and when to use them

---

## Table of Contents

1. [Why Constraints Belong in the Database](#1-why-constraints-belong-in-the-database)
2. [NOT NULL: The Simplest Constraint](#2-not-null-the-simplest-constraint)
3. [NULL: What It Means and When to Allow It](#3-null-what-it-means-and-when-to-allow-it)
4. [PRIMARY KEY Constraints](#4-primary-key-constraints)
5. [IDENTITY Columns](#5-identity-columns)
6. [FOREIGN KEY Constraints](#6-foreign-key-constraints)
7. [Referential Integrity Actions](#7-referential-integrity-actions)
8. [UNIQUE Constraints](#8-unique-constraints)
9. [CHECK Constraints](#9-check-constraints)
10. [DEFAULT Constraints](#10-default-constraints)
11. [Computed Columns](#11-computed-columns)
12. [Constraint Naming and Documentation](#12-constraint-naming-and-documentation)
13. [Discovering Constraints in the System Catalog](#13-discovering-constraints-in-the-system-catalog)
14. [Constraints in the CabotTrail OLTP](#14-constraints-in-the-cabottrail-oltp)
15. [Chapter Summary](#15-chapter-summary)
16. [Review Questions](#16-review-questions)
17. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. Why Constraints Belong in the Database

### 1.1 The Defence-in-Depth Principle

Business rules can be enforced at multiple layers:
- **Application layer:** The web form validates that price is positive before submitting
- **ETL layer:** The transformation script checks for negative quantities before loading
- **Database layer:** A CHECK constraint rejects negative prices at the storage level

Application and ETL validation is important — but it is not sufficient. Applications change, bugs are introduced, multiple applications access the same database, direct SQL connections bypass application validation entirely, and ETL processes sometimes receive corrected data that re-violates rules.

Database constraints are the last line of defence. They are always active, always enforced, and immune to application-layer bugs. A database with proper constraints will never store data that violates the rules it encodes, regardless of where that data comes from.

### 1.2 Constraints as Documentation

Every constraint is a documented business rule. A CHECK constraint that reads `CHECK (UnitPrice > 0)` says: "CabotTrail has decided that unit prices are always positive. This is a business rule." A future developer reading the schema understands this rule without consulting any document outside the database.

A constraint that exists but is undocumented outside the database is still better than a rule enforced only in application code — the constraint is discoverable through the system catalog; application code may be buried in a codebase no one fully understands.

### 1.3 The Cost of Missing Constraints

Consider what happens without constraints:

- A data entry error creates a product with `UnitPrice = -5.00`. Reports show negative revenue.
- An ETL script loads invoice lines with `Quantity = 0`. Average unit price calculations divide by zero.
- A duplicate customer is created with the same email address. Marketing sends duplicate communications; customer service cannot determine which record is authoritative.
- An order line references `ProductID = 9999`, which does not exist. Lookups fail silently or return NULLs.

Each of these would be caught immediately by the appropriate constraint. Without them, the errors may propagate undetected for months.

---

## 2. NOT NULL: The Simplest Constraint

`NOT NULL` is the most frequently applied constraint. It declares that a column must always contain a value — the column cannot be empty.

### 2.1 Syntax

```sql
-- Column-level NOT NULL
CREATE TABLE Sales.Customers
(
    CustomerID      INT             NOT NULL,
    CustomerName    NVARCHAR(100)   NOT NULL,   -- Cannot be blank
    CreditLimit     DECIMAL(18,2)   NOT NULL,   -- Cannot be blank
    AccountOpenedDate DATE          NULL,        -- Can be blank (optional)
    EmailAddress    NVARCHAR(256)   NULL         -- Can be blank
);
```

`NOT NULL` is the default in some database systems but not in SQL Server — SQL Server columns are nullable by default unless `NOT NULL` is explicitly specified. Always state nullability explicitly.

### 2.2 When to Apply NOT NULL

Apply `NOT NULL` to every column where:
- The attribute is **mandatory** for the entity to have meaning — `CustomerName` without a name is not a meaningful customer record
- The value is **always known** at the time the row is created
- A NULL would be **misleading** — a product with NULL `UnitPrice` could be interpreted as "free" or "unknown" or "error," creating ambiguity

### 2.3 Adding NOT NULL to an Existing Table

Adding `NOT NULL` to a column in a table that already contains rows requires that all existing rows have a non-NULL value in that column first:

```sql
-- Step 1: Populate existing NULLs before adding the constraint
UPDATE Sales.Customers
SET    EmailAddress = 'unknown@cabot-trail.ca'
WHERE  EmailAddress IS NULL;

-- Step 2: Add the NOT NULL constraint
ALTER TABLE Sales.Customers
ALTER COLUMN EmailAddress NVARCHAR(256) NOT NULL;
```

If any NULL remains when you attempt to add NOT NULL, SQL Server raises an error and rejects the change.

---

## 3. NULL: What It Means and When to Allow It

Chapter 2 of *SQL for Analysts* covered NULL's query behaviour. Here the design question: **when should a column be nullable?**

### 3.1 Three Meanings of NULL

SQL Server's NULL has multiple possible interpretations:

| Interpretation | Example | Correct use of NULL? |
|---|---|---|
| **Unknown** | `AccountOpenedDate` — we do not know when this account was opened | Yes — record exists but fact is unknown |
| **Not applicable** | `MiddleName` for a customer who has no middle name | Debatable — could use empty string, could use NULL |
| **Not yet assigned** | `PickedByPersonID` — order not yet picked | Yes — fact will exist later but not now |
| **Deliberately absent** | `EmailAddress` — customer has not provided one | Yes — optional attribute |

The important distinction: NULL means "no value here" — it does not mean zero, empty string, 'N/A', or 'Unknown'. Using NULL consistently for its intended purpose and documenting its meaning in the data dictionary prevents misinterpretation.

### 3.2 NULL in Aggregate Functions

As covered in Chapter 6 of *SQL for Analysts*, aggregate functions ignore NULLs. This can produce unexpected results when designing nullable columns:

```sql
-- If DeliveryDays is nullable, AVG ignores NULL rows
SELECT AVG(DeliveryDays) FROM Sales.Orders;
-- Returns the average only for orders WITH a delivery date — not all orders
-- Is that the intended analysis?
```

If the business needs to include all orders in the average (treating undelivered orders as 0 days or as a separate category), storing NULL may not be the right design. Sometimes a sentinel value (e.g., 0 or -1) is more appropriate, though it reduces the clarity that NULL provides.

### 3.3 The Three-Valued Logic Reminder

NULL in a comparison produces UNKNOWN (not TRUE or FALSE). This is why `WHERE column = NULL` returns no rows — `= NULL` always produces UNKNOWN. Use `IS NULL` and `IS NOT NULL`.

For design purposes: a nullable column in a WHERE condition requires extra care in queries. Document nullable columns in the data dictionary with their intended interpretation.

---

## 4. PRIMARY KEY Constraints

The PRIMARY KEY constraint enforces two rules simultaneously: uniqueness and NOT NULL. No two rows may have the same PK value, and no row may have a NULL PK.

### 4.1 Syntax

```sql
-- Single-column PK
CREATE TABLE Sales.Customers
(
    CustomerID  INT NOT NULL,
    CustomerName NVARCHAR(100) NOT NULL,
    ...
    CONSTRAINT PK_Customers PRIMARY KEY (CustomerID)
);

-- Composite PK
CREATE TABLE Inventory.ProductCategoryAssignments
(
    ProductID           INT NOT NULL,
    ProductCategoryID   INT NOT NULL,
    ...
    CONSTRAINT PK_ProductCategoryAssignments
        PRIMARY KEY (ProductID, ProductCategoryID)
);
```

### 4.2 What SQL Server Creates Behind the Scenes

When you define a PRIMARY KEY constraint, SQL Server automatically creates a **clustered index** on the PK column(s). This means:
- The physical data pages of the table are sorted by PK value
- Lookups by PK are extremely fast — O(log n) binary search on the clustered index
- Only one clustered index can exist per table

This has implications: if a table is most commonly queried by a non-PK column, the clustered index on the PK may not be optimal. Chapter 7 covers this trade-off in depth.

### 4.3 One Table, One Primary Key

A table has exactly one primary key — though the key may span multiple columns. If you need uniqueness enforced on a non-PK column, use a UNIQUE constraint (section 8).

---

## 5. IDENTITY Columns

`IDENTITY(seed, increment)` is not technically a constraint — it is a column property. But it is closely associated with primary key design and belongs in this discussion.

### 5.1 What IDENTITY Does

An `IDENTITY` column automatically generates a unique integer value when a new row is inserted. The database manages the counter — applications do not supply the value.

```sql
CustomerID INT NOT NULL IDENTITY(1, 1)
-- IDENTITY(seed, increment):
--   seed = starting value (1)
--   increment = step between values (1)
-- Result: 1, 2, 3, 4, 5, ...
```

### 5.2 Common Variations

```sql
-- Start at 1000 (useful when reserving low numbers for special rows)
CustomerID INT NOT NULL IDENTITY(1000, 1)

-- Start at 0 (for the unknown member / default row)
-- Then reseed to 1 after inserting the unknown row
DBCC CHECKIDENT ('Sales.Customers', RESEED, 0);
INSERT INTO Sales.Customers (CustomerID, CustomerName, ...)
VALUES (0, 'Unknown Customer', ...);
DBCC CHECKIDENT ('Sales.Customers', RESEED, 1);
-- Next auto-generated value will be 2
```

### 5.3 Retrieving the Generated Value

After inserting a row with an IDENTITY column, retrieve the generated value using `SCOPE_IDENTITY()`:

```sql
INSERT INTO Sales.Customers (CustomerName, CreditLimit)
VALUES ('New Customer Ltd', 5000.00);

SELECT SCOPE_IDENTITY() AS NewCustomerID;
-- Returns the CustomerID generated for this insert
```

`SCOPE_IDENTITY()` returns the last IDENTITY value generated in the current scope (the current query or stored procedure). Use it rather than `@@IDENTITY` (which can return values generated by triggers in other scopes).

### 5.4 IDENTITY Gaps

IDENTITY values are not reused when rows are deleted. Gaps in the sequence are normal and not a problem — the IDENTITY column is a key, not a business sequence number. If business requirements demand gapless sequences (invoice numbers, cheque numbers), use a separate sequence object or a dedicated counter table, not IDENTITY.

---

## 6. FOREIGN KEY Constraints

A FOREIGN KEY constraint enforces **referential integrity** — every value in the FK column must either be NULL (if the column is nullable) or must match a value in the referenced PK column.

### 6.1 Syntax

```sql
CREATE TABLE Sales.Orders
(
    OrderID     INT NOT NULL IDENTITY(1,1),
    CustomerID  INT NOT NULL,
    SalesRepID  INT NULL,       -- Nullable FK: order may have no assigned rep

    CONSTRAINT PK_Orders PRIMARY KEY (OrderID),

    CONSTRAINT FK_Orders_Customers
        FOREIGN KEY (CustomerID)
        REFERENCES Sales.Customers (CustomerID),

    CONSTRAINT FK_Orders_SalesRep
        FOREIGN KEY (SalesRepID)
        REFERENCES Application.Employees (EmployeeID)
);
```

### 6.2 What SQL Server Enforces

A FK constraint enforces referential integrity in both directions:

**On INSERT/UPDATE:** A new order with `CustomerID = 999` is rejected if CustomerID 999 does not exist in `Sales.Customers`.

**On DELETE/UPDATE of the referenced row:** Attempting to delete `Sales.Customers` row with `CustomerID = 42` while `Sales.Orders` has rows referencing CustomerID 42 is blocked (with default `NO ACTION` behaviour).

### 6.3 FK Without an Index on the FK Column

SQL Server creates an index on the referenced PK automatically (as part of the PK constraint). But it does **not** automatically create an index on the FK column itself — the referencing side.

This matters for performance: when SQL Server enforces a FK, it must look up the referenced PK. When the referenced table's rows are deleted or updated, SQL Server must find all referencing rows. Without an index on the FK column, this lookup requires a full table scan.

**Always create a non-clustered index on FK columns:**

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
    ON Sales.Orders (CustomerID);

CREATE NONCLUSTERED INDEX IX_Orders_SalesRepID
    ON Sales.Orders (SalesRepID);
```

Chapter 7 covers index design in detail. The FK-index practice is so important that it is introduced here.

---

## 7. Referential Integrity Actions

The `ON DELETE` and `ON UPDATE` clauses of a FK constraint define what happens when the referenced row changes.

### 7.1 The Four Actions

```sql
CONSTRAINT FK_Orders_Customers
    FOREIGN KEY (CustomerID)
    REFERENCES Sales.Customers (CustomerID)
    ON DELETE NO ACTION     -- Default
    ON UPDATE NO ACTION     -- Default
```

**`NO ACTION` (default):** Reject the delete/update if referencing rows exist. The most conservative and safest option. The application must explicitly handle the dependents before the parent can be changed.

**`CASCADE`:** Propagate the change. Deleting a customer cascades to delete all their orders. Updating a CustomerID cascades to update all referencing order rows.

```sql
ON DELETE CASCADE   -- Deleting the parent deletes all children automatically
ON UPDATE CASCADE   -- Updating the parent's PK propagates to all FK references
```

**`SET NULL`:** Set the FK to NULL when the referenced row is deleted/updated. Only valid on nullable FK columns.

```sql
ON DELETE SET NULL  -- Deleting the SalesRep sets Orders.SalesRepID = NULL
                    -- The order still exists; it just has no assigned rep
```

**`SET DEFAULT`:** Set the FK to its column-level default value. Rarely used — requires a DEFAULT constraint on the FK column pointing to a valid "fallback" row (like the unknown member, CustomerID = 0).

### 7.2 Choosing the Right Action

The referential action choice is a **business rule** about what the business wants to happen in each scenario:

| Scenario | Business question | Appropriate action |
|---|---|---|
| Customer is deleted | Should all their orders be deleted too? | `CASCADE` only if yes; `NO ACTION` if orders must be preserved |
| Sales rep leaves the company | Should their orders lose the rep assignment? | `SET NULL` if order must survive; mark order as unassigned |
| Product is discontinued | Should all order lines for that product be deleted? | Almost never `CASCADE` — use `NO ACTION` and soft-delete the product |
| PK value is updated | Should all references update automatically? | `CASCADE` is safest for PK changes if they are allowed at all |

### 7.3 The Cascade Danger

`ON DELETE CASCADE` should be used with caution. A single DELETE statement on a parent table can trigger cascading deletes across many child tables, potentially removing thousands of rows silently. For operational data that must be preserved for audit or historical analysis, `NO ACTION` with application-level dependency management is far safer.

The CabotTrail OLTP uses `NO ACTION` throughout — an appropriate choice for a transactional system where historical integrity is paramount.

---

## 8. UNIQUE Constraints

A UNIQUE constraint enforces that no two rows have the same value(s) in the specified column(s). It differs from the PRIMARY KEY in two ways:
- A table can have **multiple** UNIQUE constraints
- UNIQUE columns **can be nullable** (NULL is treated as a distinct value — two NULL values are allowed, because NULL ≠ NULL)

### 8.1 Syntax

```sql
CREATE TABLE Sales.Customers
(
    CustomerID      INT             NOT NULL IDENTITY(1,1),
    CustomerName    NVARCHAR(100)   NOT NULL,
    EmailAddress    NVARCHAR(256)   NULL,

    CONSTRAINT PK_Customers             PRIMARY KEY (CustomerID),
    CONSTRAINT UQ_Customers_Email       UNIQUE (EmailAddress)
    -- No two customers can have the same email address
    -- (NULL email addresses are allowed and do not conflict with each other)
);
```

### 8.2 Composite UNIQUE Constraints

UNIQUE can span multiple columns. The combination must be unique; individual columns may repeat:

```sql
-- A product can be assigned to each category at most once
-- (same as the composite PK in ProductCategoryAssignments — but here
-- with a surrogate PK, the UNIQUE constraint enforces the combination rule)
CONSTRAINT UQ_ProductCategoryAssignments
    UNIQUE (ProductID, ProductCategoryID)
```

### 8.3 UNIQUE as a 1:1 Enforcer

As covered in Chapter 3, a UNIQUE constraint on the FK column of a 1:1 relationship enforces the maximum cardinality of one:

```sql
-- An employee has at most one loyalty account
CONSTRAINT UQ_LoyaltyAccounts_CustomerID
    UNIQUE (CustomerID)
-- Without this, multiple accounts per customer would be possible
```

### 8.4 UNIQUE vs PRIMARY KEY

| Feature | PRIMARY KEY | UNIQUE |
|---|---|---|
| Limit per table | One | Many |
| NULL allowed | Never | Yes (treated as distinct) |
| Clustered index created | Yes (by default) | No (creates non-clustered index) |
| Purpose | Row identifier | Business uniqueness rule |

---

## 9. CHECK Constraints

A CHECK constraint tests a condition against every row being inserted or updated. If the condition evaluates to FALSE, the operation is rejected. If it evaluates to TRUE or UNKNOWN (NULL), the operation proceeds.

### 9.1 Syntax

```sql
CONSTRAINT CK_constraintname CHECK (condition)
```

The condition is any Boolean expression involving columns of the same table. Subqueries are not allowed — CHECK constraints cannot reference other tables.

### 9.2 Simple Value Range Checks

```sql
-- Unit price must be positive
CONSTRAINT CK_Products_UnitPrice
    CHECK (UnitPrice > 0)

-- Month number must be between 1 and 12
CONSTRAINT CK_Calendar_Month
    CHECK (MonthNumber BETWEEN 1 AND 12)

-- Hire date cannot be in the future
CONSTRAINT CK_Employees_HireDate
    CHECK (HireDate <= CAST(GETDATE() AS DATE))

-- Credit limit must be zero or positive
CONSTRAINT CK_Customers_CreditLimit
    CHECK (CreditLimit >= 0)
```

### 9.3 Cross-Column Checks

CHECK constraints can compare multiple columns within the same row:

```sql
-- The due date must be after the invoice date
CONSTRAINT CK_Invoices_DueDate
    CHECK (DueDate >= InvoiceDate)

-- Picked quantity cannot exceed ordered quantity
CONSTRAINT CK_OrderLines_PickedQty
    CHECK (PickedQuantity <= Quantity)

-- Discount percentage must be between 0 and 100
CONSTRAINT CK_Promotions_Discount
    CHECK (DiscountPct BETWEEN 0.00 AND 100.00)
```

### 9.4 Domain Value Checks

CHECK can enforce that a column contains only specific allowed values:

```sql
-- Transaction type must be EARN or REDEEM
CONSTRAINT CK_LoyaltyTxn_Type
    CHECK (TransactionType IN ('EARN', 'REDEEM'))

-- Credit rating must be one of the valid ratings
CONSTRAINT CK_Customers_Rating
    CHECK (CreditRating IN ('AAA', 'AA', 'A', 'BBB', 'BB', 'B', 'NR'))

-- Province code must be a valid Canadian province/territory code
CONSTRAINT CK_Addresses_Province
    CHECK (ProvinceCode IN ('AB','BC','MB','NB','NL','NS','NT','NU','ON','PE','QC','SK','YT'))
```

### 9.5 Conditional Checks

CHECK constraints can express conditional business rules:

```sql
-- If TransactionType is EARN, PointsAmount must be positive
-- If TransactionType is REDEEM, PointsAmount must be negative
CONSTRAINT CK_LoyaltyTxn_PointsDirection
    CHECK (
        (TransactionType = 'EARN'   AND PointsAmount > 0) OR
        (TransactionType = 'REDEEM' AND PointsAmount < 0)
    )

-- If IsActive = 1, TerminationDate must be NULL
-- If IsActive = 0, TerminationDate must be populated
CONSTRAINT CK_Employees_TerminationDate
    CHECK (
        (IsActive = 1 AND TerminationDate IS NULL) OR
        (IsActive = 0 AND TerminationDate IS NOT NULL)
    )
```

### 9.6 NULL Behaviour in CHECK Constraints

A CHECK constraint passes (does not reject the row) when the condition is NULL (UNKNOWN), not just when it is TRUE. This means:

```sql
CONSTRAINT CK_Products_Price CHECK (UnitPrice > 0)
-- This constraint PASSES for:
-- UnitPrice = 100.00 (TRUE)
-- UnitPrice = NULL   (UNKNOWN — not FALSE, so it passes)
-- This constraint FAILS for:
-- UnitPrice = -5.00  (FALSE)
-- UnitPrice = 0      (FALSE)
```

If NULL should also be rejected, combine NOT NULL with the CHECK:

```sql
UnitPrice   DECIMAL(18,2) NOT NULL,  -- Rejects NULL
CONSTRAINT CK_Products_Price CHECK (UnitPrice > 0)  -- Rejects zero/negative
```

The two constraints together enforce: UnitPrice must be a positive number.

---

## 10. DEFAULT Constraints

A DEFAULT constraint provides a value for a column when a row is inserted without explicitly specifying that column's value.

### 10.1 Syntax

```sql
-- Column-level default (inline)
OrderDate   DATE NOT NULL DEFAULT GETDATE()

-- Named constraint (preferred for production)
OrderDate   DATE NOT NULL,
CONSTRAINT DF_Orders_OrderDate DEFAULT GETDATE() FOR OrderDate
```

Named DEFAULT constraints can be dropped by name with `ALTER TABLE DROP CONSTRAINT`. Unnamed defaults require finding the system-generated name first.

### 10.2 Common Default Patterns

```sql
-- Default current date
CreatedDate     DATE        NOT NULL DEFAULT GETDATE()

-- Default to 0 (for counts, amounts that start at zero)
InvoiceCount    INT         NOT NULL DEFAULT 0
TotalRevenue    DECIMAL(18,2) NOT NULL DEFAULT 0.00

-- Default to FALSE (for bit flags)
IsDiscontinued  BIT         NOT NULL DEFAULT 0
IsActive        BIT         NOT NULL DEFAULT 1

-- Default to a specific lookup value
StatusID        INT         NOT NULL DEFAULT 1  -- 1 = 'Pending' in a Status lookup table

-- Default NULL (explicit — makes intent clear even though NULL is already the default)
TerminationDate DATE        NULL DEFAULT NULL
```

### 10.3 Defaults and NOT NULL Together

A common and important pattern: combine NOT NULL with a DEFAULT to ensure a column always has a value while making inserts that omit the column simpler:

```sql
-- Without DEFAULT, every INSERT must supply CreatedDate
INSERT INTO Sales.Orders (CustomerID, OrderDate, CreatedDate)
VALUES (42, '2024-03-01', GETDATE());

-- With DEFAULT GETDATE(), the column can be omitted
INSERT INTO Sales.Orders (CustomerID, OrderDate)
VALUES (42, '2024-03-01');
-- CreatedDate is automatically set to the current date
```

---

## 11. Computed Columns

A **computed column** is a column whose value is automatically derived from an expression involving other columns in the same row. It is not stored by default — it is recalculated each time it is referenced.

### 11.1 Syntax

```sql
-- Computed column: LineTotal = Quantity * UnitPrice (recalculated on each query)
LineTotal AS (Quantity * UnitPrice)

-- Persisted computed column: stored physically, maintained automatically
LineTotalIncludingTax AS (LineTotal + TaxAmount) PERSISTED
```

`PERSISTED` stores the computed value on disk and updates it automatically when any component column changes. Persisted computed columns:
- Can be indexed (non-persisted cannot)
- Are faster to query (pre-computed)
- Use more storage
- Can be used in CHECK constraints and foreign keys

### 11.2 CabotTrail Examples

```sql
-- Sales.InvoiceLines computed columns (from the CabotTrail OLTP)
LineTotal               AS (Quantity * UnitPrice),
TaxAmount               AS (LineTotal * TaxRate / 100),
LineTotalIncludingTax   AS (LineTotal + TaxAmount) PERSISTED
```

The computed column `LineTotal` is recalculated each time it is queried. `LineTotalIncludingTax` is persisted — it is stored and indexed.

### 11.3 Computed Columns vs Application Logic

Computed columns enforce derivation rules at the database layer — they cannot be set to an incorrect value by an INSERT or UPDATE. The application cannot accidentally store `LineTotal = -1` when `Quantity = 2` and `UnitPrice = 249.99`. The database always computes the correct value.

This is the constraint philosophy applied to derived values: encode the rule in the database, where it is always enforced.

---

## 12. Constraint Naming and Documentation

Consistent constraint naming is not just a style preference — it is a professional practice that makes database maintenance dramatically easier.

### 12.1 The Naming Conventions

From Chapter 3, the CabotTrail convention:

| Constraint | Pattern | Example |
|---|---|---|
| Primary key | `PK_TableName` | `PK_Customers` |
| Foreign key | `FK_ChildTable_ParentTable` | `FK_Orders_Customers` |
| Unique | `UQ_TableName_Column` | `UQ_Customers_Email` |
| Check | `CK_TableName_Column` | `CK_Products_UnitPrice` |
| Default | `DF_TableName_Column` | `DF_Orders_OrderDate` |

### 12.2 Why Names Matter in Production

**Error messages:** When a constraint violation occurs, SQL Server's error message includes the constraint name:

```
The INSERT statement conflicted with the FOREIGN KEY constraint "FK_Orders_Customers".
```

With a named constraint, the error message is immediately informative — it tells you exactly which relationship was violated. With an unnamed constraint, the system-generated name (`FK__Orders__Customer__4AB81AF0`) tells you nothing.

**Maintenance operations:** Dropping a constraint requires knowing its name:

```sql
-- Named constraint: trivial to drop
ALTER TABLE Sales.Orders DROP CONSTRAINT FK_Orders_Customers;

-- Unnamed constraint: must query sys.foreign_keys to find the generated name first
SELECT name FROM sys.foreign_keys WHERE parent_object_id = OBJECT_ID('Sales.Orders');
-- Then use the cryptic name in the DROP statement
```

**Documentation:** Named constraints are self-documenting. Reading `CK_LoyaltyTxn_PointsDirection` tells a developer what the constraint governs. Reading `CK__LoyaltyTrans__Points__7D2E4F12` tells them nothing.

---

## 13. Discovering Constraints in the System Catalog

The system catalog contains complete constraint information for every object in the database. These queries are essential for understanding an existing schema.

### 13.1 All Constraints on a Table

```sql
-- Every constraint on Sales.Orders
SELECT
    cc.name         AS ConstraintName,
    cc.type_desc    AS ConstraintType,
    -- For check constraints, the definition
    CASE WHEN cc.type = 'C' THEN cc.definition ELSE NULL END AS CheckDefinition
FROM    sys.objects cc
WHERE   cc.parent_object_id = OBJECT_ID('Sales.Orders')
AND     cc.type IN ('PK', 'UQ', 'C', 'D', 'F')
ORDER BY cc.type_desc, cc.name;
```

### 13.2 All Foreign Keys in the Database

```sql
USE CabotTrailOutdoor;

SELECT
    fk.name                         AS FKName,
    SCHEMA_NAME(tp.schema_id) + '.' + tp.name AS ChildTable,
    cp.name                         AS ChildColumn,
    SCHEMA_NAME(tr.schema_id) + '.' + tr.name AS ParentTable,
    cr.name                         AS ParentColumn,
    fk.delete_referential_action_desc AS OnDelete,
    fk.update_referential_action_desc AS OnUpdate,
    fk.is_disabled                  AS IsDisabled
FROM    sys.foreign_keys fk
INNER JOIN sys.foreign_key_columns fkc
    ON fkc.constraint_object_id = fk.object_id
INNER JOIN sys.tables tp  ON tp.object_id = fkc.parent_object_id
INNER JOIN sys.tables tr  ON tr.object_id = fkc.referenced_object_id
INNER JOIN sys.columns cp ON cp.object_id = fkc.parent_object_id
                         AND cp.column_id  = fkc.parent_column_id
INNER JOIN sys.columns cr ON cr.object_id = fkc.referenced_object_id
                         AND cr.column_id  = fkc.referenced_column_id
ORDER BY SCHEMA_NAME(tp.schema_id), tp.name, fk.name;
```

### 13.3 All CHECK Constraints

```sql
SELECT
    SCHEMA_NAME(t.schema_id) + '.' + t.name    AS TableName,
    cc.name                                     AS ConstraintName,
    cc.definition                               AS CheckExpression,
    cc.is_disabled                              AS IsDisabled
FROM    sys.check_constraints cc
INNER JOIN sys.tables t ON t.object_id = cc.parent_object_id
ORDER BY TableName, ConstraintName;
```

### 13.4 All DEFAULT Constraints

```sql
SELECT
    SCHEMA_NAME(t.schema_id) + '.' + t.name    AS TableName,
    c.name                                      AS ColumnName,
    dc.name                                     AS ConstraintName,
    dc.definition                               AS DefaultValue
FROM    sys.default_constraints dc
INNER JOIN sys.columns c ON c.object_id = dc.parent_object_id
                        AND c.column_id  = dc.parent_column_id
INNER JOIN sys.tables  t ON t.object_id = dc.parent_object_id
ORDER BY TableName, ColumnName;
```

---

## 14. Constraints in the CabotTrail OLTP

Running the catalog queries above against `CabotTrailOutdoor` reveals the constraint design choices.

### 14.1 What the OLTP Enforces

```sql
-- Count constraints by type across the entire OLTP
SELECT
    o.type_desc     AS ConstraintType,
    COUNT(*)        AS Count
FROM    sys.objects o
INNER JOIN sys.tables t ON t.object_id = o.parent_object_id
INNER JOIN sys.schemas s ON s.schema_id = t.schema_id
WHERE   o.type IN ('PK', 'UQ', 'C', 'D', 'F')
AND     s.name IN ('Sales', 'Purchasing', 'Inventory', 'Application')
GROUP BY o.type_desc
ORDER BY o.type_desc;
```

Run this query. You will find:
- Primary key constraints on every table
- Foreign key constraints linking all related tables
- Check constraints on key business rules
- Default constraints on dates, flags, and numeric columns
- Unique constraints on candidate keys (email addresses, product codes)

### 14.2 Specific Constraint Examples

```sql
-- Find all CHECK constraints in the OLTP and read their business rules
SELECT
    SCHEMA_NAME(t.schema_id) + '.' + t.name    AS TableName,
    cc.name                                     AS ConstraintName,
    cc.definition                               AS BusinessRule
FROM    sys.check_constraints cc
INNER JOIN sys.tables t ON t.object_id = cc.parent_object_id
WHERE   SCHEMA_NAME(t.schema_id) IN ('Sales', 'Purchasing', 'Inventory', 'Application')
ORDER BY TableName, ConstraintName;
```

Each check constraint you see is a business rule the original designer encoded permanently in the schema. Reading them is a form of system documentation — the constraints tell you what the designer knew about valid data.

### 14.3 The Unknown Member Rows

Several tables in the CabotTrail OLTP contain a row with PK = 0 — the **unknown member** row:

```sql
-- Verify the unknown member pattern
SELECT CustomerID, CustomerName FROM Sales.Customers WHERE CustomerID = 0;
SELECT ProductID,  ProductName  FROM Inventory.Products WHERE ProductID = 0;
SELECT EmployeeID, FullName     FROM Application.Employees WHERE EmployeeID = 0;
```

These rows are placeholder records that FK columns can reference when the actual value is unknown or not applicable. They prevent the need for nullable FKs in cases where the relationship is conceptually mandatory but the specific value is temporarily unknown.

The CHECK constraint and NOT NULL constraint on these FK columns enforce that they always have a value — even if that value is the unknown member.

---

## 15. Chapter Summary

- **Constraints** enforce business rules at the storage layer — they are always active, immune to application bugs, and discoverable through the system catalog.

- **NOT NULL** enforces mandatory attributes. All columns should explicitly declare nullability. Adding NOT NULL to an existing column requires existing NULLs to be resolved first.

- **NULL** represents missing, unknown, or not-applicable values. It is not zero or empty string. NULL in comparisons produces UNKNOWN. Three-valued logic means CHECK constraints pass on NULL unless NOT NULL is also applied.

- **PRIMARY KEY** enforces uniqueness and NOT NULL; creates a clustered index by default; one per table.

- **IDENTITY(seed, increment)** auto-generates surrogate key values. Use `SCOPE_IDENTITY()` to retrieve the generated value. Gaps are normal and not a problem.

- **FOREIGN KEY** enforces referential integrity. SQL Server does not auto-index the FK column — always create a non-clustered index on FK columns manually.

- **Referential integrity actions** (`NO ACTION`, `CASCADE`, `SET NULL`, `SET DEFAULT`) define what happens when a referenced row changes. The choice is a business rule.

- **UNIQUE** enforces uniqueness on non-PK columns; allows NULLs; multiple per table; essential for enforcing 1:1 maximum cardinality via FK + UNIQUE combination.

- **CHECK** validates column values against a condition. Can span multiple columns in the same row; cannot reference other tables. NULL makes a CHECK pass (UNKNOWN is not FALSE) unless NOT NULL is also applied.

- **DEFAULT** provides a fallback value when none is supplied. Combined with NOT NULL it simplifies inserts while ensuring mandatory columns always have a value.

- **Computed columns** enforce derivation rules at the database layer. `PERSISTED` stores the result physically and enables indexing.

- **Always name constraints** — named constraints produce informative error messages and are trivially dropped by name.

---

## 16. Review Questions

1. A developer creates a `Promotions` table with this column: `DiscountPct DECIMAL(5,2)`. They want to ensure the discount is between 0% and 50%. Write the complete column definition (including data type, nullability, default, and constraint) that correctly enforces this rule.

2. Explain why the following CHECK constraint does not reject NULL values for `ExpiryDate`, and write the corrected DDL that ensures `ExpiryDate` is always a date on or after `IssueDate` and is never NULL:
   ```sql
   ExpiryDate DATE,
   CONSTRAINT CK_Permits_Expiry CHECK (ExpiryDate > IssueDate)
   ```

3. The `Sales.LoyaltyAccounts` table has a FK from `CustomerID` to `Sales.Customers`. Describe what happens in each of the following scenarios under the default `NO ACTION` referential behaviour, and then explain which referential action (`CASCADE`, `SET NULL`, or `SET DEFAULT`) you would choose for each and why:
   a. A customer record is deleted
   b. A customer's `CustomerID` is updated

4. Write the complete DDL for a `Suppliers` table that includes: a surrogate PK, a supplier name (required), a contact email (unique if provided), a country code (must be one of 'CA', 'US', 'GB', 'DE', 'CN'), a credit terms value (must be 30, 60, or 90 days), and a created date (defaults to today). Name all constraints.

5. Run the CHECK constraint catalog query from section 13.3 against `CabotTrailOutdoor`. Find three CHECK constraints and for each: (a) state the table and column it applies to, (b) translate the check expression into a plain English business rule, (c) describe a specific INSERT that would be rejected.

6. The `Sales.Orders` table currently has `SalesRepID INT NULL` with no index on that column. Explain why this is a performance risk, and write the SQL to add both an appropriate index and a referential integrity constraint (assuming `SalesRepID` references `Application.Employees.EmployeeID` and a sales rep may or may not be assigned).

7. Write a computed column definition for `dbo.OrderSummary` that calculates `GrossMarginPct` from `TotalRevenue` and `TotalCost`, handling the divide-by-zero case. Should this column be PERSISTED? Justify your decision based on the likely query patterns for a summary table.

8. A colleague argues: "CHECK constraints are redundant — the application validates all inputs before they reach the database." Write a specific scenario from the CabotTrail context that demonstrates why this argument is wrong and why a database-level CHECK constraint would catch an error that application validation would miss.

---

## 🔍 Deeper Dive

### Going Further with Constraints and Data Integrity

#### Deferred Constraint Checking

SQL Server enforces all constraints **immediately** — when an INSERT, UPDATE, or DELETE statement is executed, every constraint is checked before the statement completes. If any constraint is violated, the statement is rolled back.

Other database systems (PostgreSQL, Oracle) support **deferrable constraints** — constraints that can be checked at the end of a transaction rather than at the end of each statement. This is useful for loading data where temporary constraint violations within a transaction are acceptable (e.g., loading a set of rows where FK references within the batch are resolved by the end of the batch but not at each individual INSERT).

SQL Server does not support deferrable constraints. The workaround: load data in the correct order (parent tables before child tables), or temporarily disable FK constraints during bulk loads and re-enable them after:

```sql
-- Disable FK checking during bulk load
ALTER TABLE Sales.Orders NOCHECK CONSTRAINT FK_Orders_Customers;

-- [Bulk load operation]

-- Re-enable and verify all constraints
ALTER TABLE Sales.Orders WITH CHECK CHECK CONSTRAINT FK_Orders_Customers;
-- WITH CHECK re-enables AND validates existing data
-- WITHOUT: ALTER TABLE Sales.Orders CHECK CONSTRAINT -- re-enables but doesn't validate existing rows
```

The `WITH CHECK CHECK` syntax (not a typo — `WITH CHECK` is the option; `CHECK CONSTRAINT` is the command) validates all existing rows against the constraint after re-enabling it. If existing data violates the constraint, the command fails. Use this to ensure data integrity after bulk loads.

Microsoft documentation:
[ALTER TABLE — Enabling Constraints](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-table-transact-sql)

#### Assertion Constraints (What SQL Server Lacks)

Standard SQL defines **assertion** constraints — named constraints that can span multiple tables and reference the entire database state:

```sql
-- Hypothetical standard SQL assertion (not supported in SQL Server)
CREATE ASSERTION Total_Revenue_Positive
    CHECK (
        (SELECT SUM(LineTotal) FROM Sales.InvoiceLines) >= 0
    );
```

SQL Server does not support assertions. When a constraint needs to check conditions across multiple tables (e.g., "an order's total cannot exceed the customer's credit limit"), the options in SQL Server are:

1. **Application-layer validation** — check in the application before inserting
2. **Stored procedure** — encapsulate the logic in a stored procedure that performs the validation before the DML
3. **INSTEAD OF trigger** — a trigger that intercepts the INSERT/UPDATE and performs the cross-table check

None of these is as clean as a database-level assertion, which is why assertions are included in the SQL standard. SQL Server's lack of support is a known gap.

#### Row-Level Security as a Constraint

SQL Server's **Row-Level Security (RLS)** is sometimes described as a form of constraint — it limits which rows a user can see or modify based on their identity. While not a traditional constraint, RLS enforces data access rules at the database layer:

```sql
-- Security predicate: only show rows matching the current user's territory
CREATE FUNCTION Security.fn_TerritoryFilter(@SalesTerritory NVARCHAR(60))
RETURNS TABLE WITH SCHEMABINDING AS
RETURN
    SELECT 1 AS IsAllowed
    WHERE @SalesTerritory = (
        SELECT UserTerritory
        FROM   Security.UserTerritoryMapping
        WHERE  Username = USER_NAME()
    );

CREATE SECURITY POLICY TerritoryPolicy
    ADD FILTER PREDICATE Security.fn_TerritoryFilter(SalesTerritory)
    ON dim.Customer;
```

The ETL book's Chapter 5 covers RLS in detail. For database designers, understanding that access control can be encoded at the database layer (rather than only in application code) is an important design principle.

Microsoft documentation:
[Row-Level Security](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security)

---

### References and Further Reading

1. Date, C. J. (2019). *Database Design and Relational Theory: Normal Forms and All That Jazz* (2nd ed.). Apress. — Chapter 10 covers constraints in the context of the relational model's integrity rules.

2. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Chapter 4 covers integrity constraints systematically; Chapter 6 covers SQL DDL for constraints.

3. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapters 11–12 cover DDL including constraints, computed columns, and sequence objects.

4. Microsoft. (2024). *PRIMARY AND FOREIGN KEY Constraints*. [https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints](https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints)

5. Microsoft. (2024). *Unique Constraints and Check Constraints*. [https://learn.microsoft.com/en-us/sql/relational-databases/tables/unique-constraints-and-check-constraints](https://learn.microsoft.com/en-us/sql/relational-databases/tables/unique-constraints-and-check-constraints)

6. Microsoft. (2024). *Specify Computed Columns in a Table*. [https://learn.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table](https://learn.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table)

7. Microsoft. (2024). *IDENTITY (Property)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql-identity-property](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql-identity-property)

8. Microsoft. (2024). *Row-Level Security*. [https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security)

---

*Previous chapter: [Chapter 5 — Normalization: 3NF and Beyond — Completing the Model](../chapter-05-normalization-3nf/README.md)*

*Next chapter: [Chapter 7 — Indexes and Physical Design: Making Queries Fast](../chapter-07-indexes-physical-design/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
