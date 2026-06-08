# Chapter 4: Normalization — 1NF and 2NF: Removing Redundancy

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Chapter 1 introduced the three anomalies — update, insertion, and deletion — that arise from bad database design. Chapters 2 and 3 showed how to design and map a database correctly from the ground up. This chapter and the next address what happens when a database is *not* designed from scratch: how to take an existing flat, redundant structure and systematically improve it.

**Normalization** is the formal process of restructuring a relational schema to reduce data redundancy and eliminate the conditions that cause anomalies. It proceeds through a series of **normal forms** — increasingly strict structural requirements. Each normal form eliminates a specific class of problem. A table that satisfies a normal form is said to be "in" that form.

This chapter covers the first two normal forms: **First Normal Form (1NF)** and **Second Normal Form (2NF)**. Together, they eliminate the two most fundamental categories of structural problems: multi-valued cells and partial key dependencies.

By the end of this chapter you will be able to:

- Explain what functional dependency means and why it is the foundation of normalization
- Define First Normal Form and identify violations in any table
- Normalise a table to 1NF by removing multi-valued cells and repeating groups
- Define Second Normal Form and identify partial dependencies
- Normalise a table to 2NF by decomposing to remove partial dependencies
- Explain the preservation of data and lossless joins after normalization
- Apply normalization to realistic, messy source data from the CabotTrail context

---

## Table of Contents

1. [What Normalization Is and Why It Matters](#1-what-normalization-is-and-why-it-matters)
2. [Functional Dependency: The Foundation](#2-functional-dependency-the-foundation)
3. [The Unnormalised Form: Starting Point](#3-the-unnormalised-form-starting-point)
4. [First Normal Form (1NF)](#4-first-normal-form-1nf)
5. [Identifying 1NF Violations](#5-identifying-1nf-violations)
6. [Normalising to 1NF: Worked Examples](#6-normalising-to-1nf-worked-examples)
7. [Second Normal Form (2NF)](#7-second-normal-form-2nf)
8. [Functional Dependency Diagrams](#8-functional-dependency-diagrams)
9. [Identifying Partial Dependencies](#9-identifying-partial-dependencies)
10. [Normalising to 2NF: Worked Examples](#10-normalising-to-2nf-worked-examples)
11. [Lossless Decomposition](#11-lossless-decomposition)
12. [CabotTrail in 1NF and 2NF](#12-cabottrail-in-1nf-and-2nf)
13. [Chapter Summary](#13-chapter-summary)
14. [Review Questions](#14-review-questions)
15. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. What Normalization Is and Why It Matters

### 1.1 The Problem Normalization Solves

Imagine you receive a spreadsheet from a colleague representing CabotTrail's sales data. It has been assembled by combining information from several sources, and it looks like this:

```
OrderID | CustomerName | CustomerCity | ProductCodes          | ProductNames              | Quantities
1001    | Fundy Bay    | Halifax      | CT-083, CT-101        | Sleeping Bag, Tent        | 2, 1
1002    | Trail Hut    | Truro        | CT-045                | Hiking Boots              | 3
1003    | Fundy Bay    | Halifax      | CT-083, CT-200, CT-210| Sleeping Bag, Pad, Stove  | 1, 2, 1
```

This spreadsheet has several structural problems:

- `ProductCodes`, `ProductNames`, and `Quantities` contain comma-separated lists — multiple values in a single cell
- `CustomerName` and `CustomerCity` are repeated on every order from the same customer
- If Halifax's spelling changes, multiple rows must be updated
- You cannot query "how many units of CT-083 were sold" without parsing comma-separated strings

These problems do not arise from bad data — they arise from bad **structure**. Normalization provides a systematic method for identifying and fixing structural problems.

### 1.2 The Normal Form Levels

Normal forms are numbered: 1NF, 2NF, 3NF, BCNF, 4NF, 5NF. Each higher form satisfies all lower forms plus an additional requirement. In practice, most business databases aim for **Third Normal Form (3NF)** — which eliminates the three main categories of redundancy anomalies. 4NF and 5NF address more exotic cases covered in Chapter 5's Deeper Dive.

```
UNF (Unnormalised Form)
  ↓  Eliminate multi-valued attributes and repeating groups
1NF (First Normal Form)
  ↓  Eliminate partial dependencies
2NF (Second Normal Form)
  ↓  Eliminate transitive dependencies
3NF (Third Normal Form)
  ↓  Eliminate all remaining non-trivial dependencies on non-candidate keys
BCNF (Boyce-Codd Normal Form)
```

### 1.3 Normalization Is Analysis, Not Redesign

Normalization does not change the *information* in the database — it reorganises how that information is stored. After normalization, the same facts can be retrieved (by joining the normalised tables). The data is preserved; the redundancy is removed.

This is the **lossless join** property: a properly normalised decomposition can be reconstructed to the original by joining the parts. Section 11 demonstrates this.

---

## 2. Functional Dependency: The Foundation

Before any normal form can be defined, a concept must be established: **functional dependency**. It is the mathematical backbone of normalization theory.

### 2.1 The Definition

A column B is **functionally dependent** on column A (written `A → B`, read "A determines B") if, for every possible value of A, there is exactly one corresponding value of B.

Knowing A tells you B without ambiguity.

```
CustomerID → CustomerName
```

If you know `CustomerID = 42`, you know exactly one `CustomerName`: 'Fundy Bay Outfitters'. No two customers have the same `CustomerID` with different names. So `CustomerName` is functionally dependent on `CustomerID`.

```
CustomerID → CustomerCity
```

`CustomerID = 42` always maps to the same city: 'Halifax'. So `CustomerCity` is functionally dependent on `CustomerID`.

### 2.2 Counter-Example: Not a Functional Dependency

```
CustomerCity → CustomerName   (NOT a functional dependency)
```

Knowing the city 'Halifax' does not tell you a single customer name — there may be many customers in Halifax. Multiple customers share the same city. So `CustomerName` is NOT functionally dependent on `CustomerCity`.

### 2.3 Full vs Partial Functional Dependency

When the primary key is **composite** (two or more columns), a non-key attribute may depend on:

- **The whole key** — a **full functional dependency**
- **Only part of the key** — a **partial functional dependency**

This distinction is the basis of Second Normal Form. A table that has attributes depending on only part of a composite key is in 1NF but not 2NF.

### 2.4 Notation Summary

```
A → B          A functionally determines B (B is fully dependent on A)
AB → C         The combination of A and B determines C
A → B, A → C   A determines both B and C
```

When writing functional dependencies, the left side is the **determinant** and the right side is the **dependent**.

---

## 3. The Unnormalised Form: Starting Point

The **Unnormalised Form (UNF)** is a table with no formal structure requirements — it may have multi-valued cells, repeating groups, or no defined primary key. This is the starting point for normalization.

### 3.1 A Realistic UNF Example

The sales spreadsheet from section 1.1 is typical UNF data. Here it is expressed as a SQL table (though SQL Server would struggle to store it correctly because of the multi-valued cells):

```
SALES_FLAT (unnormalised)
OrderID | CustomerID | CustomerName  | CustomerCity | Products (repeating group)
        |            |               |              | ProductID | ProductName | Qty | UnitPrice
1001    | 42         | Fundy Bay     | Halifax      | 83        | Sleeping Bag| 2   | 249.99
        |            |               |              | 101       | Tent        | 1   | 649.99
1002    | 17         | Trail Hut     | Truro        | 45        | Hiking Boots| 3   | 159.99
1003    | 42         | Fundy Bay     | Halifax      | 83        | Sleeping Bag| 1   | 249.99
        |            |               |              | 200       | Sleeping Pad| 2   | 89.99
        |            |               |              | 210       | Camp Stove  | 1   | 119.99
```

Problems visible immediately:
- Each order row appears to span multiple sub-rows (the repeating product group)
- `CustomerName` and `CustomerCity` repeat for customer 42 across orders 1001 and 1003
- No clear primary key (OrderID alone doesn't identify a row if there are multiple product lines)

---

## 4. First Normal Form (1NF)

A table is in **First Normal Form** if and only if:

1. Every column contains **atomic (single-valued)** values — no lists, arrays, or repeating groups
2. Every column contains values from the **same domain** (consistent data type)
3. Every row is **unique** — there is a defined primary key
4. The **order of rows** is irrelevant (no row ordering dependency)

Condition 1 is the critical one in practice. Conditions 2–4 are properties of the relational model itself (Chapter 1); a properly defined SQL table satisfies them automatically with data types and PRIMARY KEY constraints.

### 4.1 The Atomicity Requirement

"Atomic" means indivisible — a single value that cannot be meaningfully subdivided for query purposes.

**Not atomic:**
- `'CT-083, CT-101'` in a ProductCodes column (a list of codes)
- `'Halifax, NS'` in a Location column (city and province combined)
- `'2, 1'` in a Quantities column (a list of quantities)

**Atomic:**
- `'CT-083'` — a single product code
- `'Halifax'` — a single city name
- `2` — a single quantity

The test for atomicity: does the business need to query, filter, or operate on individual components of this value? If yes, it must be split into separate columns or a separate table.

---

## 5. Identifying 1NF Violations

Three patterns consistently signal 1NF violations.

### 5.1 Pattern 1: Comma-Separated or Delimited Values

Any column storing multiple values separated by a delimiter violates 1NF.

```sql
-- 1NF violation: ProductCodes contains multiple values
ProductCodes = 'CT-083, CT-101, CT-200'

-- How would you query "all orders containing product CT-101"?
WHERE ProductCodes = 'CT-101'                  -- Misses 'CT-083, CT-101'
WHERE ProductCodes LIKE '%CT-101%'             -- Works but is fragile and non-sargable
```

The LIKE workaround is a common band-aid. It signals a structural problem: if you need LIKE to extract individual values from a column, the column is not atomic.

### 5.2 Pattern 2: Repeating Column Groups

A table with columns like `Product1`, `Product2`, `Product3` is structurally equivalent to a multi-valued attribute — just expressed as multiple columns rather than a list:

```
OrderID | Product1ID | Product1Name | Product1Qty | Product2ID | Product2Name | Product2Qty
1001    | 83         | Sleeping Bag | 2           | 101        | Tent         | 1
1002    | 45         | Hiking Boots | 3           | NULL       | NULL         | NULL
```

Problems:
- What if order 1003 has 5 products? The table would need `Product5ID`, `Product5Name`, `Product5Qty`
- NULLs represent non-applicable values — structurally messy
- Queries aggregating across all product columns are complex and change as columns are added

### 5.3 Pattern 3: No Primary Key

A table without a primary key cannot guarantee unique rows — it violates the uniqueness requirement of 1NF. Even if rows happen to be unique, there is no enforcement mechanism. This is a 1NF violation.

---

## 6. Normalising to 1NF: Worked Examples

### 6.1 Converting Comma-Separated Values

Starting table (1NF violation — comma-separated products):

```
ORDERS_FLAT
OrderID | CustomerID | ProductCodes     | ProductNames               | Quantities
1001    | 42         | CT-083, CT-101   | Sleeping Bag, Tent         | 2, 1
1002    | 17         | CT-045           | Hiking Boots               | 3
```

Step 1 — Identify the repeating group: `ProductCode`, `ProductName`, `Quantity` repeat per order.

Step 2 — Move the repeating group to a new table, with a FK back to the original table and a PK on the new combination:

```sql
-- After 1NF: split into two tables
-- Table 1: Order-level information (no repeating attributes)
CREATE TABLE Orders_1NF
(
    OrderID     INT NOT NULL,
    CustomerID  INT NOT NULL,
    CONSTRAINT PK_Orders_1NF PRIMARY KEY (OrderID)
);

-- Table 2: Order line items (the repeating group becomes rows)
CREATE TABLE OrderLines_1NF
(
    OrderID     INT NOT NULL,
    ProductCode NVARCHAR(20) NOT NULL,
    ProductName NVARCHAR(100) NOT NULL,
    Quantity    INT NOT NULL,
    CONSTRAINT PK_OrderLines_1NF PRIMARY KEY (OrderID, ProductCode),
    CONSTRAINT FK_OrderLines_Order FOREIGN KEY (OrderID) REFERENCES Orders_1NF (OrderID)
);
```

The comma-separated list is gone. Each product on each order is now its own row. Order 1001 produces two rows in `OrderLines_1NF` (one for CT-083, one for CT-101).

### 6.2 Converting Repeating Column Groups

Starting table (1NF violation — repeating column groups):

```
ORDERS_COLUMNS
OrderID | CustomerID | Prod1Code | Prod1Qty | Prod2Code | Prod2Qty | Prod3Code | Prod3Qty
1001    | 42         | CT-083    | 2        | CT-101    | 1        | NULL      | NULL
1002    | 17         | CT-045    | 3        | NULL      | NULL     | NULL      | NULL
```

Step 1 — Identify the repeating column group: `ProdNCode`, `ProdNQty` (where N = 1, 2, 3).

Step 2 — Same resolution: move to a new table, one row per (order, product) pair:

```sql
-- Same solution as comma-separated values: separate table for line items
CREATE TABLE OrderLines_1NF
(
    OrderID     INT             NOT NULL,
    ProductCode NVARCHAR(20)    NOT NULL,
    Quantity    INT             NOT NULL,
    CONSTRAINT PK_OrderLines_1NF PRIMARY KEY (OrderID, ProductCode),
    CONSTRAINT FK_OL_Order FOREIGN KEY (OrderID) REFERENCES Orders_1NF (OrderID)
);
```

The NULL-filled repeating columns become absent rows — clean and extensible.

### 6.3 Inserting the 1NF Data

The data transforms from the flat structure to the normalised rows:

```sql
-- Original flat row: Order 1001 with products CT-083 (qty 2) and CT-101 (qty 1)
-- Becomes:
INSERT INTO Orders_1NF VALUES (1001, 42);

INSERT INTO OrderLines_1NF VALUES
    (1001, 'CT-083', 2),
    (1001, 'CT-101', 1);

-- Original flat row: Order 1002 with one product CT-045 (qty 3)
INSERT INTO Orders_1NF VALUES (1002, 17);

INSERT INTO OrderLines_1NF VALUES
    (1002, 'CT-045', 3);
```

No information is lost. The original data can be reconstructed with a JOIN:

```sql
SELECT o.OrderID, o.CustomerID, ol.ProductCode, ol.Quantity
FROM   Orders_1NF o
INNER JOIN OrderLines_1NF ol ON ol.OrderID = o.OrderID;
```

---

## 7. Second Normal Form (2NF)

A table is in **Second Normal Form** if and only if:

1. It is in **First Normal Form**
2. Every non-key attribute is **fully functionally dependent on the entire primary key** — no partial dependencies

2NF is only relevant when the primary key is **composite** — a key made of two or more columns. If a table has a single-column primary key, it is automatically in 2NF (there is no "part of the key" to partially depend on).

### 7.1 Partial Dependency: The Violation

A **partial dependency** occurs when a non-key attribute is functionally dependent on *only part* of a composite key, rather than the whole key.

After 1NF, the `OrderLines_1NF` table has:

```
OrderLines_1NF (in 1NF)
PK: (OrderID, ProductCode)

OrderID | ProductCode | Quantity | ProductName         | UnitPrice
1001    | CT-083      | 2        | Sleeping Bag        | 249.99
1001    | CT-101      | 1        | Tent                | 649.99
1002    | CT-045      | 3        | Hiking Boots        | 159.99
1003    | CT-083      | 1        | Sleeping Bag        | 249.99
```

Check each non-key attribute:
- `Quantity` — depends on `(OrderID, ProductCode)` together: how many of CT-083 were on order 1001 is a fact about the specific combination. ✓ **Full dependency**
- `ProductName` — depends on `ProductCode` alone: 'CT-083' is always 'Sleeping Bag' regardless of which order it appears on. ✗ **Partial dependency** on just `ProductCode`
- `UnitPrice` — depends on `ProductCode` alone: the price of CT-083 is always $249.99 regardless of the order. ✗ **Partial dependency** on just `ProductCode`

`ProductName` and `UnitPrice` partially depend on `ProductCode` — only part of the composite PK. This violates 2NF.

### 7.2 Why Partial Dependencies Cause Anomalies

**Update anomaly:** If the price of CT-083 changes to $259.99, every row with `ProductCode = 'CT-083'` must be updated. Missing any one row means inconsistent prices across order lines.

**Insertion anomaly:** You cannot record information about a new product (its name and price) until it appears on at least one order. The product's name and price are facts about the product, not about any specific order line — but they can only be stored once an order line exists.

**Deletion anomaly:** If the only order containing CT-045 (Hiking Boots) is deleted, the fact that CT-045 is called 'Hiking Boots' and costs $159.99 is also deleted.

---

## 8. Functional Dependency Diagrams

A **functional dependency diagram** (FD diagram) visualises all the functional dependencies in a table, making partial and transitive dependencies visible.

### 8.1 Drawing an FD Diagram

Conventions:
- Draw all columns as named boxes
- Draw arrows from determinant to dependent
- Show the primary key (or composite key components) clearly
- Identify partial dependencies (arrows from part of the PK) vs full dependencies (arrows from the whole PK)

For `OrderLines_1NF`:

```
                    ┌──────────┐
OrderID ──────────→ │          │
         (together) │ Quantity │
ProductCode ──────→ │          │
                    └──────────┘

ProductCode ──────────────────→ ProductName
ProductCode ──────────────────→ UnitPrice
```

The arrows from `ProductCode` alone (without `OrderID`) to `ProductName` and `UnitPrice` are the partial dependencies. They violate 2NF.

### 8.2 Identifying Full vs Partial Dependencies

The systematic check: for each non-key attribute, ask "Could this attribute's value change if I changed only one component of the composite key, while keeping the other constant?"

- Can `Quantity` change if I change `OrderID` while keeping `ProductCode` fixed? Yes — a different order for the same product could have a different quantity. So `Quantity` needs **both** key components. ✓ Full dependency.
- Can `ProductName` change if I change `OrderID` while keeping `ProductCode` fixed? No — the product name depends only on the product, not which order it's on. ✗ Partial dependency.

---

## 9. Identifying Partial Dependencies

### 9.1 The Standard Partial Dependency Pattern

Partial dependencies always follow the same pattern:
1. There is a composite PK: `(A, B)`
2. Some non-key attribute C depends only on A (or only on B)
3. C is a fact about A (or B) alone, not about the combination (A, B)

### 9.2 Practical Questions to Ask

When examining a table with a composite PK for 2NF violations, ask for each non-key column:

**"Is this a fact about [A], [B], or [A and B together]?"**

If the answer is "[A] alone" or "[B] alone" → partial dependency → 2NF violation.
If the answer is "[A and B together]" → full dependency → no violation.

For `OrderLines_1NF`:
- `Quantity`: "Is this a fact about the order, the product, or the specific combination of order and product?" → The combination. ✓
- `ProductName`: "Is this a fact about the order, the product, or the combination?" → The product alone. ✗
- `UnitPrice`: "Is this a fact about the order, the product, or the combination?" → The product alone (assuming catalogue prices). ✗

### 9.3 A Subtler Case: Price at Time of Order

Is `UnitPrice` truly a fact about the product alone? Only if prices never change, or if the business does not need to know what price was charged on a specific order.

In reality, prices change. CabotTrail's invoices record the `UnitPrice` at the time of the sale — which may differ from today's catalogue price. So for historical invoice data, `UnitPrice` IS a fact about the specific order line (the combination), not just the product.

This is the key insight: **the correct functional dependencies depend on business rules, not just on the data that currently exists in the table.** If the business rule is "record the price at time of sale," then `UnitPrice` on `InvoiceLines` is a full dependency (on the line, which identifies the specific sale). If the business rule is "always use the current catalogue price," then `UnitPrice` is a partial dependency on the product.

The CabotTrail OLTP records `UnitPrice` on both `OrderLines` and `InvoiceLines` — correctly treating it as a fact about the specific transaction, not about the product.

---

## 10. Normalising to 2NF: Worked Examples

### 10.1 The Decomposition Step

To remove a partial dependency:

1. Create a **new table** for the partially dependent attributes, with the partial determinant as its PK
2. **Remove the partially dependent attributes** from the original table
3. **Keep the partial determinant column** in the original table as a FK to the new table

```
Before 2NF:
OrderLines_1NF (OrderID, ProductCode, Quantity, ProductName, UnitPrice)
PK: (OrderID, ProductCode)

Partial dependencies: ProductCode → ProductName, UnitPrice
```

```
After 2NF decomposition:
Table 1: Products_2NF (ProductCode, ProductName, UnitPrice)
         PK: ProductCode
         (Captures the facts about products)

Table 2: OrderLines_2NF (OrderID, ProductCode, Quantity)
         PK: (OrderID, ProductCode)
         FK: ProductCode → Products_2NF.ProductCode
         (Captures the facts about specific order lines)
```

```sql
-- 2NF: Products table (facts about products)
CREATE TABLE Products_2NF
(
    ProductCode     NVARCHAR(20)    NOT NULL,
    ProductName     NVARCHAR(100)   NOT NULL,
    UnitPrice       DECIMAL(18,2)   NOT NULL,

    CONSTRAINT PK_Products_2NF PRIMARY KEY (ProductCode)
);

-- 2NF: OrderLines table (facts about specific order lines)
CREATE TABLE OrderLines_2NF
(
    OrderID         INT             NOT NULL,
    ProductCode     NVARCHAR(20)    NOT NULL,
    Quantity        INT             NOT NULL,

    CONSTRAINT PK_OrderLines_2NF PRIMARY KEY (OrderID, ProductCode),
    CONSTRAINT FK_OL_Product FOREIGN KEY (ProductCode)
        REFERENCES Products_2NF (ProductCode),
    CONSTRAINT FK_OL_Order   FOREIGN KEY (OrderID)
        REFERENCES Orders_1NF (OrderID)
);
```

### 10.2 Populating the 2NF Tables

```sql
-- Products from the flat data
INSERT INTO Products_2NF VALUES
    ('CT-083', 'Sleeping Bag', 249.99),
    ('CT-101', 'Tent', 649.99),
    ('CT-045', 'Hiking Boots', 159.99);

-- Order lines (quantity only — product details are in Products_2NF)
INSERT INTO OrderLines_2NF VALUES
    (1001, 'CT-083', 2),
    (1001, 'CT-101', 1),
    (1002, 'CT-045', 3),
    (1003, 'CT-083', 1);
```

### 10.3 The 2NF Benefits Demonstrated

**Update anomaly eliminated:** To change CT-083's price, update one row in `Products_2NF`. All order lines referencing CT-083 automatically see the new price through the FK.

**Insertion anomaly eliminated:** A new product can be added to `Products_2NF` before it appears on any order line.

**Deletion anomaly eliminated:** Deleting all order lines containing CT-045 does not delete 'Hiking Boots' from `Products_2NF`.

### 10.4 A Second 2NF Example: Customer Data

The original flat `ORDERS_FLAT` also has a partial dependency if it has a composite PK:

```
ORDERS_FLAT (OrderID, CustomerID, CustomerName, CustomerCity, OrderDate)

If PK is OrderID alone: no partial dependency issue
If PK were somehow (OrderID, CustomerID): CustomerName and CustomerCity
  depend on CustomerID alone → partial dependency
```

In practice, `OrderID` is sufficient as the PK for orders. The issue with `CustomerName` and `CustomerCity` is a different problem — **transitive dependency** — which is the subject of 3NF (Chapter 5).

---

## 11. Lossless Decomposition

Every normalization step must be **lossless** — you must be able to reconstruct the original data by joining the decomposed tables. If the join produces more or fewer rows than the original, the decomposition introduced errors.

### 11.1 The Lossless Join Test

```sql
-- Original flat data (before normalization):
-- OrderID 1001 had CT-083 (qty 2) and CT-101 (qty 1)
-- OrderID 1002 had CT-045 (qty 3)

-- After 2NF decomposition, JOIN to reconstruct:
SELECT
    ol.OrderID,
    ol.ProductCode,
    p.ProductName,
    ol.Quantity,
    p.UnitPrice
FROM    OrderLines_2NF ol
INNER JOIN Products_2NF p ON p.ProductCode = ol.ProductCode
ORDER BY ol.OrderID, ol.ProductCode;
```

The result should match the original flat data exactly — same rows, same values. No rows added, no rows lost.

### 11.2 What Makes a Decomposition Lossless

A decomposition is lossless if the **join column is the primary key of one of the resulting tables**. In the 2NF decomposition:

- `OrderLines_2NF` and `Products_2NF` are joined on `ProductCode`
- `ProductCode` is the primary key of `Products_2NF`
- Therefore the join is lossless — each order line's `ProductCode` maps to exactly one product row

If the join column were not the PK of either table, the join could produce spurious (extra, incorrect) rows — a lossy decomposition.

### 11.3 Dependency Preservation

Along with lossless joins, a good normalization preserves **dependencies** — after decomposition, every functional dependency from the original table can still be enforced by a constraint in the new tables.

In the 2NF decomposition: `ProductCode → ProductName` is enforced by the UNIQUE constraint on `ProductCode` in `Products_2NF`. The dependency is preserved in the new table.

---

## 12. CabotTrail in 1NF and 2NF

### 12.1 Is the CabotTrail OLTP Already Normalised?

Yes — the CabotTrail OLTP was designed with normalization in mind. Examining it confirms this and reinforces the principles.

**1NF compliance:**

```sql
-- Check: are there any multi-valued columns?
-- Query for columns of XML, JSON, or text types that might store lists
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM   INFORMATION_SCHEMA.COLUMNS
WHERE  DATA_TYPE IN ('xml', 'ntext', 'text')
AND    TABLE_CATALOG = 'CabotTrailOutdoor';
-- Expected: no results (or none storing lists)
```

Every column in `CabotTrailOutdoor` stores atomic values. There are no repeating groups, no comma-separated lists. **The OLTP is in 1NF.**

**2NF compliance:**

```sql
-- Check: does Sales.InvoiceLines have any partial dependencies?
-- PK: (InvoiceID, OrderLineID) — let's verify this is indeed the PK
SELECT
    c.name          AS ColumnName,
    c.is_nullable,
    CASE WHEN pk.column_id IS NOT NULL THEN 'PK' ELSE '' END AS KeyType
FROM    sys.tables t
INNER JOIN sys.schemas s    ON s.schema_id = t.schema_id
INNER JOIN sys.columns c    ON c.object_id = t.object_id
LEFT JOIN (
    SELECT ic.object_id, ic.column_id
    FROM   sys.index_columns ic
    INNER JOIN sys.indexes i ON i.object_id = ic.object_id
                             AND i.is_primary_key = 1
) pk ON pk.object_id = t.object_id AND pk.column_id = c.column_id
WHERE  s.name = 'Sales' AND t.name = 'InvoiceLines'
ORDER BY c.column_id;
```

For `Sales.InvoiceLines`, the non-key columns include `ProductID`, `Quantity`, `UnitPrice`, `LineTotal`, `TaxRate`, `TaxAmount`. Checking each:

- `ProductID` — depends on the invoice line (which product was on this line). ✓ Full dependency
- `Quantity` — depends on the invoice line (how many were invoiced). ✓ Full dependency
- `UnitPrice` — depends on the invoice line (price **at time of invoice**). ✓ Full dependency on the line

This is the key design decision: `UnitPrice` on `InvoiceLines` is the price charged for this specific invoice line — not the product's current catalogue price. It is a fact about the invoice line, not about the product. **This is what makes it a full dependency and not a 2NF violation.**

### 12.2 Where 2NF Would Be Violated

If CabotTrail had stored `ProductName` or `CategoryName` on `InvoiceLines` (as the UNF spreadsheet did), that would be a 2NF violation — those facts belong to the Product entity, not to any specific invoice line.

The absence of `ProductName` from `InvoiceLines` is not an oversight — it is correct design. When a report needs product names on invoice lines, the query joins `InvoiceLines` to `Products` (or in the DW, `FactSales` to `DimProduct`). The join is the mechanism that assembles normalised facts into readable output.

---

## 13. Chapter Summary

- **Normalization** eliminates data redundancy and anomalies (update, insertion, deletion) by restructuring tables according to formal rules.

- **Functional dependency** (A → B) means knowing A uniquely determines B. It is the mathematical foundation of every normal form.

- **First Normal Form (1NF)** requires: atomic (single-valued) cells, consistent column types, unique rows (primary key), and irrelevant row order. Violations include comma-separated values, repeating column groups, and missing primary keys.

- To normalise to 1NF: move repeating groups to a new table with a FK back to the original table. One row per atomic fact.

- **Second Normal Form (2NF)** requires: being in 1NF **plus** every non-key attribute is fully functionally dependent on the entire composite primary key. Tables with a single-column PK are automatically in 2NF.

- A **partial dependency** is when a non-key attribute depends on only part of a composite key. It causes update, insertion, and deletion anomalies.

- To normalise to 2NF: move partially dependent attributes to a new table keyed by the partial determinant. Keep the partial determinant column as a FK in the original table.

- Every normalization decomposition must be **lossless** — the original data can be reconstructed by joining the decomposed tables. This is guaranteed when the join column is the PK of one of the tables.

- The CabotTrail OLTP is already in at least 2NF — `UnitPrice` on `InvoiceLines` is a full dependency on the invoice line (price at time of sale), not a partial dependency on the product.

---

## 14. Review Questions

1. A table `COURSE_REGISTRATION` has columns: `(StudentID, CourseID, StudentName, CourseName, Grade, InstructorName)` with PK `(StudentID, CourseID)`. Identify all functional dependencies, classify each as full or partial, and explain which ones violate 2NF.

2. The following table stores supplier invoices in UNF:
   ```
   InvoiceNum | SupplierName | SupplierCity | Items (repeating)
              |              |              | ProductCode | Description | Qty | Cost
   ```
   (a) Normalise to 1NF — write the resulting tables with PKs.
   (b) Identify any 2NF violations in your 1NF result.
   (c) Normalise to 2NF — write the final tables with PKs and FKs.

3. Explain why 2NF is only relevant when a table has a composite primary key. What happens if a table with a single-column surrogate key has a column that logically depends on only a "part" of the data? Does this violate 2NF? If not, which normal form addresses this?

4. The CabotTrail OLTP stores `UnitPrice` on `Sales.InvoiceLines`. Explain why this is **not** a 2NF violation, even though `UnitPrice` is also an attribute of the `Products` entity. What business rule makes the difference?

5. Write the lossless join test for the following 2NF decomposition. Show the query that reconstructs the original data and explain why it produces exactly the same rows as the original flat table:
   - Original: `ENROLMENT(StudentID, CourseID, StudentName, CourseName, Grade)`
   - After 2NF: `STUDENT(StudentID, StudentName)`, `COURSE(CourseID, CourseName)`, `ENROLMENT(StudentID, CourseID, Grade)`

6. A table `PRODUCT_SUPPLIER(ProductID, SupplierID, ProductName, SupplierName, LeadTimeDays)` has PK `(ProductID, SupplierID)`. Identify all functional dependencies. Draw the FD diagram. Which attributes violate 2NF, and what tables result from the 2NF decomposition?

7. Consider this business rule: "The unit price charged on an order line may differ from the catalogue price because of negotiated discounts." Does this change the functional dependency analysis for `UnitPrice` in `OrderLines`? Explain your reasoning.

8. After normalising to 2NF, a developer complains that queries now require a JOIN that was not needed before. "You've made the queries more complicated," they say. Construct a counter-argument that explains the benefits of 2NF using a specific example of an anomaly the join eliminates.

---

## 🔍 Deeper Dive

### Going Further with Functional Dependencies and 1NF/2NF

#### Armstrong's Axioms

The rules for deriving functional dependencies from known ones are called **Armstrong's Axioms** (after William Ward Armstrong, who formalised them in 1974). They are the logical foundation for computing the closure of a set of functional dependencies:

**Reflexivity:** If B is a subset of A, then A → B. (Trivial: {StudentID, CourseID} → StudentID)

**Augmentation:** If A → B, then AC → BC for any C. (If you already know A determines B, adding C to both sides preserves the dependency)

**Transitivity:** If A → B and B → C, then A → C. (This is the basis of transitive dependencies — the subject of 3NF)

From these three axioms, two additional rules are derived:

**Union:** If A → B and A → C, then A → BC.
**Decomposition:** If A → BC, then A → B and A → C.

Understanding Armstrong's Axioms explains *why* normalization decompositions work: the functional dependencies in the original table are preserved (through derivation) in the decomposed tables.

Armstrong, W. W. (1974). Dependency structures of data base relationships. *Proceedings of IFIP Congress*, 74, 580–583.

#### The Minimal Cover

For a given set of functional dependencies, there may be **redundant dependencies** — ones that can be derived from the others. The **minimal cover** (also called the **canonical cover**) is the smallest set of functional dependencies from which all others can be derived.

Computing the minimal cover is the basis of automated normalization algorithms used by some database design tools. It eliminates:
- Redundant dependencies (A → B when B can already be derived from other dependencies)
- Redundant attributes on the left side of dependencies (AB → C when A → C alone)

For the purposes of this course, identifying and removing partial and transitive dependencies manually is sufficient. The minimal cover becomes important in advanced normalization (4NF, 5NF) and in algorithm-based schema synthesis.

#### 1NF and the NoSQL Debate

First Normal Form's atomicity requirement — that every cell contains a single value — is contested in modern database practice. JSON document databases (MongoDB, Couchbase), key-value stores (Redis), and column-family databases (Cassandra) explicitly allow multi-valued, nested, and structured column values.

The argument for non-atomic columns in some modern systems:
- If you always retrieve all parts of a composite value together (e.g., a customer's complete address as a unit), storing them as a structured document is more efficient than normalising them into separate columns and joining
- Application-level access is often more natural with documents than with joins
- For systems with rapidly changing schemas, the rigidity of 1NF can be a liability

The argument for 1NF:
- Individual components of a non-atomic value cannot be efficiently filtered, sorted, or aggregated without parsing
- Referential integrity cannot be enforced across JSON values
- SQL's set-based operations assume atomic values

The practical position: relational databases with 1NF are the right choice for transactional systems with well-defined schemas, complex relationships, and integrity requirements. Document stores are appropriate for applications with flexible schemas, document-centric access patterns, and less complex relationships. The relational model remains the dominant choice for business data management precisely because of these guarantees.

---

### References and Further Reading

1. Date, C. J. (2019). *Database Design and Relational Theory: Normal Forms and All That Jazz* (2nd ed.). Apress. — The most rigorous and complete treatment of normal forms, functional dependencies, and the theory underlying normalization.

2. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Chapters 14–15 cover normalization from 1NF through 5NF with extensive worked examples.

3. Elmasri, R., & Navathe, S. (2016). *Fundamentals of Database Systems* (7th ed.). Pearson. — Chapter 14 covers functional dependencies and normal forms rigorously; Chapter 15 covers the relational design algorithms.

4. Armstrong, W. W. (1974). Dependency structures of data base relationships. *Proceedings of IFIP Congress*, 74, 580–583. — The original paper establishing the inference axioms for functional dependencies.

5. Kent, W. (1983). A simple guide to five normal forms in relational database theory. *Communications of the ACM*, 26(2), 120–125. — A remarkably accessible explanation of the normal forms, freely available and widely cited.

6. Microsoft. (2024). *Database normalization description*. [https://learn.microsoft.com/en-us/office/troubleshoot/access/database-normalization-description](https://learn.microsoft.com/en-us/office/troubleshoot/access/database-normalization-description) — Microsoft's practical introduction to normalization.

---

*Previous chapter: [Chapter 3 — From ERD to Tables: Mapping Diagrams to DDL](../chapter-03-erd-to-tables/README.md)*

*Next chapter: [Chapter 5 — Normalization: 3NF and Beyond — Completing the Model](../chapter-05-normalization-3nf/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
