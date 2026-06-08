# Chapter 5: Normalization — 3NF and Beyond: Completing the Model

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Chapter 4 addressed the first two normal forms: 1NF eliminated multi-valued cells and repeating groups; 2NF eliminated partial dependencies on composite keys. A table in 2NF is significantly better than one in UNF or 1NF — but it can still harbour a specific class of redundancy: facts that depend on non-key columns rather than on the key itself.

**Third Normal Form (3NF)** eliminates this remaining class of dependency — the **transitive dependency** — to produce a schema where every non-key fact depends directly on the primary key, the whole key, and nothing but the key.

This chapter also introduces **Boyce-Codd Normal Form (BCNF)**, a stricter variant of 3NF that closes a subtle loophole, and gives an accessible overview of 4NF and 5NF for completeness. It closes with the practical question that every designer eventually faces: **when to stop normalising**.

By the end of this chapter you will be able to:

- Define transitive dependency and identify instances of it in a table
- Define Third Normal Form and explain its relationship to 2NF
- Normalise a table from 2NF to 3NF by decomposing transitive dependencies
- Explain Boyce-Codd Normal Form and when it differs from 3NF
- Describe 4NF and 5NF at a conceptual level
- Explain the concept of denormalization and when it is appropriate
- Apply the complete normalization process (UNF → 1NF → 2NF → 3NF) to a realistic example
- Evaluate whether the CabotTrail OLTP is in 3NF

---

## Table of Contents

1. [The Residual Problem After 2NF](#1-the-residual-problem-after-2nf)
2. [Transitive Dependency: The Definition](#2-transitive-dependency-the-definition)
3. [Identifying Transitive Dependencies](#3-identifying-transitive-dependencies)
4. [Third Normal Form (3NF)](#4-third-normal-form-3nf)
5. [Normalising to 3NF: Worked Examples](#5-normalising-to-3nf-worked-examples)
6. [The 3NF Test: Kent's Mnemonic](#6-the-3nf-test-kents-mnemonic)
7. [Boyce-Codd Normal Form (BCNF)](#7-boyce-codd-normal-form-bcnf)
8. [4NF and 5NF: Brief Overview](#8-4nf-and-5nf-brief-overview)
9. [The Complete Normalization Process](#9-the-complete-normalization-process)
10. [CabotTrail in 3NF](#10-cabottrail-in-3nf)
11. [Denormalization: When to Break the Rules](#11-denormalization-when-to-break-the-rules)
12. [Chapter Summary](#12-chapter-summary)
13. [Review Questions](#13-review-questions)
14. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. The Residual Problem After 2NF

After applying 2NF, the `Orders_2NF` table has only a single-column primary key (`OrderID`) and non-key attributes about the order:

```
Orders_2NF
OrderID | CustomerID | CustomerName  | CustomerCity | OrderDate
1001    | 42         | Fundy Bay     | Halifax      | 2024-03-01
1002    | 17         | Trail Hut     | Truro        | 2024-03-05
1003    | 42         | Fundy Bay     | Halifax      | 2024-04-10
1004    | 17         | Trail Hut     | Truro        | 2024-04-15
```

This table is in 2NF — there is no composite key, so no partial dependencies are possible. But it still has redundancy:

- `CustomerName = 'Fundy Bay'` appears on every order placed by customer 42
- `CustomerCity = 'Halifax'` appears on every order placed by customer 42
- If Fundy Bay's name changes, every one of their order rows must be updated
- If all of customer 42's orders are deleted, the fact that CustomerID 42 exists and is based in Halifax disappears

These are exactly the update and deletion anomalies from Chapter 1 — but they survived 2NF. The cause is a **transitive dependency**.

---

## 2. Transitive Dependency: The Definition

A **transitive dependency** occurs when a non-key attribute depends on another non-key attribute — rather than depending directly on the primary key.

Formally: given primary key A and non-key attributes B and C, a transitive dependency exists when:
```
A → B   and   B → C   (therefore A → C transitively, not directly)
```

In `Orders_2NF`:
```
OrderID → CustomerID       (knowing the order, you know the customer)
CustomerID → CustomerName  (knowing the customer, you know their name)
CustomerID → CustomerCity  (knowing the customer, you know their city)

Therefore (transitively):
OrderID → CustomerName  (but only through CustomerID — not directly)
OrderID → CustomerCity  (but only through CustomerID — not directly)
```

`CustomerName` and `CustomerCity` depend on `OrderID` — but only *through* `CustomerID`. They are facts about the customer, not about the order. The order merely knows which customer it belongs to; it does not "own" the customer's name or city.

### 2.1 Visualising the Chain

The transitive dependency forms a chain:

```
OrderID ──────→ CustomerID ──────→ CustomerName
                              └───→ CustomerCity
```

The arrow from `OrderID` to `CustomerName` is the transitive path. It is not a direct dependency — `CustomerName` would not change if you changed `OrderID` while keeping `CustomerID` the same. The direct dependency runs through `CustomerID`.

### 2.2 The Intermediate Non-Key Column

The distinguishing feature: the intermediate step in the chain (`CustomerID`) is a **non-key column** in the orders table (it is a FK, but it is not the PK). When a non-key column determines other non-key columns, a transitive dependency exists.

---

## 3. Identifying Transitive Dependencies

The systematic process for identifying transitive dependencies in a table:

**Step 1:** List all non-key attributes.

**Step 2:** For each non-key attribute, ask: "What does this attribute describe — the primary key entity, or something else?"

**Step 3:** If it describes "something else" — check whether that "something else" is another non-key column. If yes, you have a transitive dependency.

### 3.1 Applied to Orders_2NF

Non-key attributes: `CustomerID`, `CustomerName`, `CustomerCity`, `OrderDate`

- `CustomerID` — describes the order (which customer placed it). ✓ Direct dependency on `OrderID`
- `CustomerName` — describes the customer, not the order. The customer's name is a fact about `CustomerID`, not about `OrderID`. ✗ Transitive via `CustomerID`
- `CustomerCity` — describes the customer, not the order. ✗ Transitive via `CustomerID`
- `OrderDate` — describes the order (when it was placed). ✓ Direct dependency on `OrderID`

`CustomerName` and `CustomerCity` are transitively dependent on `OrderID` through `CustomerID`. They violate 3NF.

### 3.2 The "Describes" Test

The practical test: **"Does this attribute describe the primary key entity, or does it describe something referenced by a FK?"**

If it describes something referenced by a FK — it belongs in the table for that FK's entity, not in the current table.

`CustomerName` describes the customer (`CustomerID` entity), not the order (`OrderID` entity). It belongs in a `Customers` table, not in `Orders`. Storing it in `Orders` creates redundancy because the same customer places many orders.

---

## 4. Third Normal Form (3NF)

A table is in **Third Normal Form** if and only if:

1. It is in **Second Normal Form**
2. Every non-key attribute is **non-transitively dependent on the primary key** — that is, every non-key attribute depends directly on the PK, and on nothing other than the PK

Equivalently: no non-key attribute depends on another non-key attribute.

### 4.1 The Classic Formulation

Edgar Codd's original 1971 definition states: a table is in 3NF if every non-key attribute depends on "the key, the whole key, and nothing but the key."

This mnemonic is famous. Read it literally:
- **"The key"** → not on part of the key (eliminates 2NF violations — partial dependencies)
- **"The whole key"** → not on something derived from the key through intermediaries
- **"Nothing but the key"** → not on any non-key attribute (eliminates 3NF violations — transitive dependencies)

### 4.2 Why Transitive Dependencies Cause Anomalies

The three anomalies, demonstrated for `Orders_2NF`:

**Update anomaly:** Fundy Bay Outfitters changes their trading name to 'Fundy Bay Ltd.' Every row in `Orders` where `CustomerID = 42` must be updated. Missing even one produces inconsistent data — some orders show 'Fundy Bay Outfitters', others show 'Fundy Bay Ltd.'

**Insertion anomaly:** A new customer is created (CustomerID = 55, CustomerName = 'Cape Breton Supply'). The customer cannot be recorded until they place their first order — because `CustomerName` only exists in the `Orders` table, and an order is needed to create a row.

**Deletion anomaly:** All orders for customer 17 are deleted (perhaps the customer had a dispute and all records are to be purged). The fact that CustomerID 17 is 'Trail Hut' in 'Truro' disappears along with the orders.

---

## 5. Normalising to 3NF: Worked Examples

### 5.1 The Decomposition Step

To remove a transitive dependency A → B → C:

1. Create a **new table** for the transitively dependent attributes, keyed by the intermediate non-key attribute (B)
2. **Remove the transitively dependent attributes** (C) from the original table
3. **Keep the intermediate attribute** (B) in the original table as a FK to the new table

```
Before 3NF (transitive dependency):
Orders_2NF (OrderID, CustomerID, CustomerName, CustomerCity, OrderDate)
Transitive chain: OrderID → CustomerID → CustomerName, CustomerCity
```

```
After 3NF decomposition:
Table 1: Customers_3NF (CustomerID, CustomerName, CustomerCity)
         PK: CustomerID
         (Facts about customers — no longer in Orders)

Table 2: Orders_3NF (OrderID, CustomerID, OrderDate)
         PK: OrderID
         FK: CustomerID → Customers_3NF.CustomerID
         (Facts about orders — CustomerID is a FK, not carrying customer details)
```

```sql
-- 3NF: Customers table
CREATE TABLE Customers_3NF
(
    CustomerID      INT             NOT NULL,
    CustomerName    NVARCHAR(100)   NOT NULL,
    CustomerCity    NVARCHAR(60)    NOT NULL,

    CONSTRAINT PK_Customers_3NF PRIMARY KEY (CustomerID)
);

-- 3NF: Orders table
CREATE TABLE Orders_3NF
(
    OrderID         INT     NOT NULL,
    CustomerID      INT     NOT NULL,
    OrderDate       DATE    NOT NULL,

    CONSTRAINT PK_Orders_3NF PRIMARY KEY (OrderID),
    CONSTRAINT FK_Orders_Customer
        FOREIGN KEY (CustomerID) REFERENCES Customers_3NF (CustomerID)
);
```

### 5.2 The 3NF Data

```sql
-- One row per customer
INSERT INTO Customers_3NF VALUES
    (42, 'Fundy Bay Outfitters', 'Halifax'),
    (17, 'Trail Hut Ltd',        'Truro');

-- Orders reference customers by ID only
INSERT INTO Orders_3NF VALUES
    (1001, 42, '2024-03-01'),
    (1002, 17, '2024-03-05'),
    (1003, 42, '2024-04-10'),
    (1004, 17, '2024-04-15');
```

Customer details are stored once. The update anomaly is eliminated: change 'Fundy Bay Outfitters' to 'Fundy Bay Ltd' in one row in `Customers_3NF` — all orders referencing CustomerID 42 automatically reflect the change through the FK relationship.

### 5.3 A Second Example: Employee and Department

Consider:

```
EMPLOYEE (EmployeeID, EmployeeName, DepartmentID, DepartmentName, DepartmentHead)
PK: EmployeeID
```

Check for transitive dependencies:
- `EmployeeName` — describes the employee. ✓ Direct
- `DepartmentID` — describes the employee (which department they are in). ✓ Direct
- `DepartmentName` — describes the department, not the employee. `DepartmentID → DepartmentName`. ✗ Transitive
- `DepartmentHead` — describes the department's head, not the employee. `DepartmentID → DepartmentHead`. ✗ Transitive

3NF decomposition:

```sql
CREATE TABLE Departments_3NF
(
    DepartmentID    INT             NOT NULL,
    DepartmentName  NVARCHAR(60)    NOT NULL,
    DepartmentHead  NVARCHAR(100)   NULL,

    CONSTRAINT PK_Departments_3NF PRIMARY KEY (DepartmentID)
);

CREATE TABLE Employees_3NF
(
    EmployeeID      INT             NOT NULL,
    EmployeeName    NVARCHAR(100)   NOT NULL,
    DepartmentID    INT             NOT NULL,

    CONSTRAINT PK_Employees_3NF PRIMARY KEY (EmployeeID),
    CONSTRAINT FK_Emp_Dept FOREIGN KEY (DepartmentID)
        REFERENCES Departments_3NF (DepartmentID)
);
```

### 5.4 Recognising 3NF in ERDs

In hindsight, a correct ERD avoids transitive dependencies automatically. The `Customer` entity in the ERD has its own box and its own attributes. An `Order` entity has its own attributes and a relationship line to `Customer`. The ERD makes it visually obvious that customer attributes belong in the Customer entity — not duplicated on the Order entity.

This is why the ERD process (Chapter 2) and the mapping rules (Chapter 3) naturally produce a 3NF schema when applied correctly. Normalization analysis becomes necessary when starting from existing flat data (like a spreadsheet) rather than from a clean ERD.

---

## 6. The 3NF Test: Kent's Mnemonic

William Kent's 1983 paper introduced a memorable phrasing for testing 3NF:

> *"Every non-key attribute must provide a fact about the key, the whole key, and nothing but the key."*

Each phrase targets a specific normal form:

| Phrase | Normal form tested | Eliminates |
|---|---|---|
| "…a fact about the key" | 2NF | Partial dependencies (depends on part of a composite key) |
| "…the whole key" | 2NF | (same — the entire composite key, not just a part) |
| "…nothing but the key" | 3NF | Transitive dependencies (depends on a non-key column) |

### 6.1 Applying the Mnemonic

For each non-key column, ask: does this column provide a fact about the key (the PK entity), and only about the key?

`CustomerName` in `Orders`: Does it provide a fact about the order (`OrderID`)? No — it provides a fact about the customer (`CustomerID`). Fails "nothing but the key." → 3NF violation.

`OrderDate` in `Orders`: Does it provide a fact about the order? Yes — when the order was placed is a fact about the order itself. Passes all three parts of the mnemonic. → No violation.

---

## 7. Boyce-Codd Normal Form (BCNF)

**Boyce-Codd Normal Form** (named after Raymond Boyce and Edgar Codd, who proposed it in 1974) is a strengthened version of 3NF that closes a subtle loophole.

### 7.1 The Loophole in 3NF

3NF requires that non-key attributes depend on "the key, the whole key, and nothing but the key." But it says nothing about what happens when a *key attribute* (part of a candidate key) depends on a non-key column.

In tables with multiple overlapping candidate keys, this situation can arise. A table can be in 3NF but still have a determinant that is not a candidate key.

### 7.2 BCNF Definition

A table is in **BCNF** if and only if for every non-trivial functional dependency X → Y, X is a **superkey** (a candidate key or a superset of a candidate key).

In other words: the only determinants allowed are superkeys. No non-key column may determine any column — including key columns.

### 7.3 A BCNF Violation Example

Consider a university course scheduling scenario:

```
COURSE_SCHEDULE (StudentID, CourseID, InstructorID, Room)
Business rules:
- Each student takes each course from exactly one instructor
- Each instructor teaches only one course (though many students)
- Each room hosts only one course at a time, but a course may be in multiple rooms

Candidate keys: (StudentID, CourseID) and (StudentID, InstructorID)
Functional dependencies:
- (StudentID, CourseID) → InstructorID
- InstructorID → CourseID       ← This is the BCNF violation!
```

`InstructorID → CourseID` is a functional dependency where the determinant (`InstructorID`) is **not** a superkey — it alone cannot identify a row. Yet it determines `CourseID` (a key attribute). This violates BCNF.

### 7.4 When BCNF Differs from 3NF

BCNF and 3NF produce the same result in most practical cases. The difference only arises in tables with:

- Multiple overlapping candidate keys
- A functional dependency where the determinant is part of one candidate key and the dependent is part of another

Such tables are relatively rare in typical business database design. When they do arise, BCNF decomposition sometimes sacrifices dependency preservation — a trade-off that must be evaluated.

### 7.5 BCNF in Practice

For CabotTrail and most business databases: if a table is in 3NF, it is almost certainly in BCNF. The distinction only matters in schemas with multiple complex candidate keys — more common in academic examples than in transactional business databases.

The practical guidance: **design to 3NF. Evaluate BCNF compliance when the schema has multiple overlapping candidate keys. If a BCNF decomposition sacrifices dependency preservation, weigh the trade-off carefully.**

---

## 8. 4NF and 5NF: Brief Overview

For completeness, the higher normal forms address increasingly specialised cases.

### 8.1 Fourth Normal Form (4NF)

**4NF** addresses **multi-valued dependencies** — a situation where one attribute independently determines two or more other attributes, creating spurious combinations when stored in the same table.

**Example:** A `ProductSupplierRegion` table stores which suppliers can provide which products, and in which regions a product is sold:

```
ProductID | SupplierID | Region
83        | SUP-01     | Atlantic
83        | SUP-01     | Ontario
83        | SUP-02     | Atlantic
83        | SUP-02     | Ontario
```

`ProductID` has two **independent** multi-valued facts: it determines a set of suppliers, and it determines a set of regions. These are independent — which suppliers can provide a product has nothing to do with which regions it is sold in. Storing them together forces every (Supplier, Region) combination to be represented, creating redundant rows.

**4NF decomposition:**
```
ProductSuppliers: (ProductID, SupplierID)
ProductRegions:   (ProductID, Region)
```

4NF is violated when there are independent multi-valued facts about the same entity in the same table. It is relevant primarily when designing junction-like tables with multiple independent M:N relationships.

### 8.2 Fifth Normal Form (5NF)

**5NF** (also called **Project-Join Normal Form**) addresses **join dependencies** — situations where a table can only be reconstructed losslessly from three or more projections (sub-tables), not just two.

5NF violations are extremely rare in practice — they arise only when three or more entities have a constrained three-way relationship where knowing any two pairs does not imply the third. Most practical business databases do not encounter 5NF issues.

### 8.3 Practical Perspective

For business database design:

| Normal form | Address in practice? | Notes |
|---|---|---|
| 1NF | Always | Eliminate multi-valued cells |
| 2NF | Always | Eliminate partial dependencies |
| 3NF | Always | Eliminate transitive dependencies |
| BCNF | Evaluate when relevant | Only differs from 3NF with overlapping candidate keys |
| 4NF | Occasionally | Relevant when same entity has multiple independent M:N facts |
| 5NF | Rarely | Complex join dependencies; extremely uncommon in business systems |

The target for this course — and for most professional database work — is **3NF**. BCNF is a refinement worth understanding; 4NF and 5NF are theoretical completeness.

---

## 9. The Complete Normalization Process

This section applies the complete UNF → 1NF → 2NF → 3NF process to a single realistic example, showing all intermediate steps.

### 9.1 Starting Point: UNF

A product order flat file from a legacy system:

```
ORDER_FLAT (UNF)
OrderNum | OrderDate  | CustID | CustName      | CustProvince | Items (repeating)
         |            |        |               |              | ProdCode | ProdName | SuppID | SuppName | Qty | Price
1001     | 2024-03-01 | C42    | Fundy Bay     | NS           | P083     | Slp. Bag | S-01   | MtnGear  | 2   | 249.99
         |            |        |               |              | P101     | Tent     | S-01   | MtnGear  | 1   | 649.99
1002     | 2024-03-05 | C17    | Trail Hut     | NS           | P045     | Hk. Boot | S-02   | TrlEquip | 3   | 159.99
1003     | 2024-04-10 | C42    | Fundy Bay     | NS           | P083     | Slp. Bag | S-01   | MtnGear  | 1   | 249.99
```

### 9.2 Step 1: To 1NF

Identify the repeating group: `(ProdCode, ProdName, SuppID, SuppName, Qty, Price)`.

Remove it to a new table. The composite PK of the new table: `(OrderNum, ProdCode)` — together they uniquely identify an order line.

```
ORDER_1NF:
OrderNum | OrderDate  | CustID | CustName  | CustProvince
1001     | 2024-03-01 | C42    | Fundy Bay | NS
1002     | 2024-03-05 | C17    | Trail Hut | NS
1003     | 2024-04-10 | C42    | Fundy Bay | NS

ORDERLINE_1NF:
OrderNum | ProdCode | ProdName | SuppID | SuppName  | Qty | Price
1001     | P083     | Slp. Bag | S-01   | MtnGear   | 2   | 249.99
1001     | P101     | Tent     | S-01   | MtnGear   | 1   | 649.99
1002     | P045     | Hk. Boot | S-02   | TrlEquip  | 3   | 159.99
1003     | P083     | Slp. Bag | S-01   | MtnGear   | 1   | 249.99
```

### 9.3 Step 2: To 2NF

Check `ORDERLINE_1NF` for partial dependencies. PK: `(OrderNum, ProdCode)`.

- `Qty` — depends on `(OrderNum, ProdCode)`. ✓ Full
- `Price` — depends on `(OrderNum, ProdCode)` if it is the price charged on this order. ✓ Full (assuming historical price capture)
- `ProdName` — depends on `ProdCode` alone. ✗ Partial dependency
- `SuppID` — depends on `ProdCode` alone (each product has one supplier). ✗ Partial dependency
- `SuppName` — depends on `ProdCode` alone (through `SuppID`). ✗ Partial dependency

Remove partial dependencies to a new Products table:

```
PRODUCT_2NF:
ProdCode | ProdName  | SuppID | SuppName
P083     | Slp. Bag  | S-01   | MtnGear
P101     | Tent      | S-01   | MtnGear
P045     | Hk. Boot  | S-02   | TrlEquip

ORDERLINE_2NF:
OrderNum | ProdCode | Qty | Price
1001     | P083     | 2   | 249.99
1001     | P101     | 1   | 649.99
1002     | P045     | 3   | 159.99
1003     | P083     | 1   | 249.99
```

`ORDER_1NF` remains unchanged after this step — its PK `OrderNum` is single-column, so no partial dependencies are possible.

### 9.4 Step 3: To 3NF

Check both tables for transitive dependencies.

**`ORDER_1NF`:** Non-key attributes: `OrderDate`, `CustID`, `CustName`, `CustProvince`.
- `OrderDate` — describes the order. ✓ Direct
- `CustID` — describes the order (which customer). ✓ Direct
- `CustName` — describes the customer, not the order. `CustID → CustName`. ✗ Transitive via `CustID`
- `CustProvince` — describes the customer, not the order. `CustID → CustProvince`. ✗ Transitive via `CustID`

**`PRODUCT_2NF`:** Non-key attributes: `ProdName`, `SuppID`, `SuppName`.
- `ProdName` — describes the product. ✓ Direct
- `SuppID` — describes the product (which supplier provides it). ✓ Direct
- `SuppName` — describes the supplier, not the product. `SuppID → SuppName`. ✗ Transitive via `SuppID`

Remove transitive dependencies:

```
CUSTOMER_3NF:
CustID | CustName  | CustProvince
C42    | Fundy Bay | NS
C17    | Trail Hut | NS

ORDER_3NF:
OrderNum | CustID | OrderDate
1001     | C42    | 2024-03-01
1002     | C17    | 2024-03-05
1003     | C42    | 2024-04-10

SUPPLIER_3NF:
SuppID | SuppName
S-01   | MtnGear
S-02   | TrlEquip

PRODUCT_3NF:
ProdCode | ProdName  | SuppID
P083     | Slp. Bag  | S-01
P101     | Tent      | S-01
P045     | Hk. Boot  | S-02

ORDERLINE_3NF (unchanged from 2NF — no transitive dependencies):
OrderNum | ProdCode | Qty | Price
```

### 9.5 Final Schema Summary

From one flat UNF table, normalization produced five tables in 3NF:

| Table | PK | FKs |
|---|---|---|
| `CUSTOMER_3NF` | CustID | — |
| `SUPPLIER_3NF` | SuppID | — |
| `PRODUCT_3NF` | ProdCode | SuppID → SUPPLIER |
| `ORDER_3NF` | OrderNum | CustID → CUSTOMER |
| `ORDERLINE_3NF` | (OrderNum, ProdCode) | OrderNum → ORDER, ProdCode → PRODUCT |

Every non-key attribute now depends on exactly and only the primary key of its table — directly, not transitively. The schema is in 3NF.

---

## 10. CabotTrail in 3NF

### 10.1 Does the OLTP Pass the 3NF Test?

The CabotTrail OLTP was designed with normalization in mind. A systematic check confirms it:

```sql
-- Check the structure of Sales.Orders
SELECT  c.name          AS ColumnName,
        c.is_nullable,
        tp.name         AS DataType
FROM    sys.tables t
INNER JOIN sys.schemas s  ON s.schema_id = t.schema_id
INNER JOIN sys.columns c  ON c.object_id = t.object_id
INNER JOIN sys.types tp   ON tp.user_type_id = c.user_type_id
WHERE   s.name = 'Sales' AND t.name = 'Orders'
ORDER BY c.column_id;
```

`Sales.Orders` columns: `OrderID`, `CustomerID`, `SalesRepID`, `PickedByPersonID`, `ContactPersonID`, `BackorderOrderID`, `OrderDate`, `ExpectedDeliveryDate`, `CustomerPurchaseOrderNumber`, `IsUndersupplyBackordered`, `Comments`, `DeliveryInstructions`, `InternalComments`, `PickingCompletedWhen`, `LastEditedBy`, `LastEditedWhen`, `CityID`, `DeliveryMethodID`.

Check for transitive dependencies:
- `CustomerID` — which customer placed the order. ✓ Direct (the FK, not the customer's name)
- `OrderDate` — when the order was placed. ✓ Direct
- `CityID` — delivery city for this order. ✓ Direct (the FK, not the city's name)
- `DeliveryMethodID` — how the order is delivered. ✓ Direct (the FK)

**No customer name, city name, or delivery method name appears on `Sales.Orders`.** Only FKs. This is 3NF — the transitive dependencies have been resolved by referencing dimension tables through FKs rather than duplicating their attributes.

### 10.2 Where 3NF Would Be Violated in a Less Careful Design

If `Sales.Orders` stored `CustomerName` (from the customer record), that would be a transitive dependency: `OrderID → CustomerID → CustomerName`. The CabotTrail design correctly stores only `CustomerID` and retrieves `CustomerName` by joining to `Sales.Customers`.

If `Sales.Orders` stored `DeliveryMethodName` directly (instead of `DeliveryMethodID`), that would be another transitive dependency. The design correctly references the method through a FK.

**Every fact in `Sales.Orders` describes the order itself or identifies a related entity through a FK. There are no attributes that describe the related entity directly.** This is the 3NF pattern in production.

### 10.3 The Geography Hierarchy as 3NF

The four-table geography hierarchy (`Countries → StateProvinces → Cities → Customers.DeliveryCityID`) is a 3NF design. Consider what would happen if `Customers` stored `CityName` and `ProvinceCode` directly:

```
Sales.Customers (bad design):
CustomerID | CustomerName | CityName | ProvinceCode
42         | Fundy Bay    | Halifax  | NS

Transitive chain: CustomerID → CityName → ProvinceCode
(Every city in Halifax belongs to NS — ProvinceCode is a fact about
the city, not directly about the customer)
```

The correct 3NF design:
```
Customers: CustomerID, DeliveryCityID (FK only)
Cities:    CityID, CityName, StateProvinceID (FK only)
StateProvinces: StateProvinceID, StateProvinceCode, CountryID (FK only)
Countries: CountryID, CountryName
```

Each fact lives in exactly the right table. The province code is stored once in `StateProvinces` and referenced everywhere — not duplicated on every customer and every city record.

---

## 11. Denormalization: When to Break the Rules

Normalization produces the most maintainable, least redundant structure. But it is not always the right structure for every purpose.

### 11.1 What Denormalization Is

**Denormalization** is the deliberate introduction of redundancy — the reversal of some normalization steps — to improve read performance at the cost of write complexity and storage.

Denormalization is appropriate when:
- The schema is used primarily for **reading** (analytical queries) rather than writing (transactional updates)
- **Query performance** is the primary concern, and the joins required by a normalised schema are too slow
- The **data changes infrequently** — the update anomaly risk is low
- The team has the **discipline and processes** to maintain the redundant data correctly

### 11.2 The DW as an Example of Intentional Denormalization

The CabotTrail data warehouse dimensions are intentionally denormalised. `dim.Customer` in `CabotTrailOutdoorsSales` stores `CustomerName`, `CityName`, `ProvinceName`, `ProvinceCode`, `CountryName`, and `SalesTerritory` all in one wide table — information that, in the normalised OLTP, is spread across four tables.

This denormalization is deliberate and justified:
- **The DW is read-only** for analysts — BI tools query it but do not update customer records
- **Joins are expensive** in queries scanning millions of fact rows — flattening the geography hierarchy into the customer dimension avoids three joins on every analytical query
- **Updates are managed by ETL** — when customer data changes in the OLTP, the ETL reloads the denormalised DW dimension

This is the Kimball star schema design philosophy: normalise the OLTP for writes; denormalise the DW for reads.

### 11.3 Other Denormalization Patterns

**Storing aggregates:** Adding a `TotalOrderValue` column to the `Orders` table to avoid summing `OrderLines` on every report. Update: the total must be recalculated when any line changes.

**Duplicating a lookup value:** Storing `ProvinceName` on `Customers` for display purposes even though it is available via join. Update: if the province name changes, every customer row must be updated.

**Collapsing a hierarchy:** Merging `Cities`, `StateProvinces`, and `Countries` into `Customers` as flat columns. Update: every customer in Halifax must be updated if Halifax's province changes.

### 11.4 The Denormalization Decision

Denormalization is not wrong — it is a trade-off. The question is always: **is the performance gain worth the maintenance cost and the risk of anomalies?**

For OLTP transactional systems: almost never. Normalization to 3NF is almost always the right choice because consistency and data integrity are paramount.

For OLAP analytical systems and data warehouses: frequently. The read-heavy access pattern, the managed ETL update process, and the need for query performance justify controlled denormalization.

**The discipline:** Know what normal form your schema is in. Know which rules you have broken and why. Document the denormalization decisions in the data dictionary so future developers understand the trade-offs.

---

## 12. Chapter Summary

- **Transitive dependency** occurs when a non-key attribute depends on another non-key attribute — forming a chain A → B → C where B is not a key. It causes update, insertion, and deletion anomalies.

- **Third Normal Form (3NF)** requires: in 2NF, plus every non-key attribute depends directly on the PK — "the key, the whole key, and nothing but the key."

- To identify 3NF violations: ask "does this attribute describe the primary key entity, or does it describe something referenced by a FK?" If the latter, it is transitively dependent.

- To normalise to 3NF: create a new table for the transitively dependent attributes, keyed by the intermediate non-key determinant. Keep the intermediate attribute as a FK in the original table.

- **BCNF** strengthens 3NF: every determinant must be a superkey. BCNF only differs from 3NF in tables with multiple overlapping candidate keys.

- **4NF** eliminates independent multi-valued dependencies. **5NF** eliminates join dependencies. Both are rare in practice; 3NF is the practical target.

- The **complete normalization process** (UNF → 1NF → 2NF → 3NF) transforms a flat table into a set of smaller, well-structured tables. Each step eliminates a specific class of redundancy.

- The CabotTrail OLTP is in 3NF: every column on every table describes only the entity that table represents, referencing related entities through FKs rather than copying their attributes.

- **Denormalization** deliberately introduces redundancy to improve read performance. It is appropriate for OLAP and DW designs but requires discipline, documentation, and managed update processes.

---

## 13. Review Questions

1. A table `EMPLOYEE_PROJECT (EmployeeID, ProjectID, ProjectName, ProjectManagerID, ProjectManagerName, HoursWorked)` has PK `(EmployeeID, ProjectID)`. After applying 2NF, you have `EMPLOYEE_PROJECT_2NF (EmployeeID, ProjectID, HoursWorked)` and `PROJECT_2NF (ProjectID, ProjectName, ProjectManagerID, ProjectManagerName)`. Does `PROJECT_2NF` contain any 3NF violations? Identify them and show the 3NF decomposition.

2. Explain the transitive dependency chain in the following table, draw the functional dependency diagram, and show the complete 3NF decomposition:
   `INVOICE (InvoiceID, CustomerID, CustomerName, CustomerCity, CustomerProvince, InvoiceDate, TotalAmount)`

3. Apply Kent's mnemonic ("the key, the whole key, and nothing but the key") to each non-key column in `Sales.Orders` from `CabotTrailOutdoor`. For each column, state which part of the mnemonic it satisfies and why. Use the FK catalog query from Chapter 1 to identify the columns and their relationships.

4. The following is a claim about BCNF vs 3NF: "In practice, if a table is in 3NF, it is almost always also in BCNF." Explain the conditions under which this claim is TRUE, and construct a specific example where a table can be in 3NF but NOT in BCNF.

5. Walk through the complete normalization process (UNF → 1NF → 2NF → 3NF) for the following flat data:
   ```
   STAFF_COURSE (StaffID, StaffName, DeptID, DeptName, CourseID, CourseName, DateCompleted, Score)
   Business rules: One staff member can complete many courses.
   Each course is offered by one department. Each staff member belongs to one department.
   ```
   Show intermediate tables at each step with PKs and FKs.

6. The CabotTrail data warehouse `dim.Customer` stores `CityName`, `ProvinceName`, and `CountryName` as flat columns — data that is spread across four normalised tables in the OLTP. Explain why this is acceptable in the DW but would be unacceptable in the OLTP. Reference the denormalization principles from section 11.

7. Write a SQL query against `CabotTrailOutdoor.Sales.Customers` that verifies there are no 3NF violations — that is, that no customer-level columns are storing facts that belong to other entities (cities, provinces, countries). Hint: check what FKs exist on this table using the system catalog, and confirm that geographical information is stored as FKs, not as direct attribute columns.

8. A designer argues: "Normalization is for academic exercises. Real-world databases need to be fast, so we should always denormalize." Construct a balanced response that acknowledges the valid points in this argument while explaining the risks and the contexts where each approach is appropriate.

---

## 🔍 Deeper Dive

### Going Further with 3NF and Beyond

#### The Synthesis Algorithm for 3NF

Rather than decomposing an existing unnormalised table, it is possible to **synthesise** a 3NF schema directly from a set of functional dependencies. The synthesis algorithm (Bernstein 1976) proceeds as follows:

1. Compute the minimal cover of the functional dependencies
2. For each dependency in the minimal cover, create a table with the determinant as PK and the dependent(s) as non-key columns
3. If no table contains a candidate key of the original relation, add a table containing a candidate key
4. Eliminate redundant tables (tables whose attributes are a subset of another table's attributes)

The synthesis algorithm **guarantees**:
- The result is in 3NF
- All functional dependencies are preserved
- The decomposition is lossless

It is the theoretical basis for the practical normalization steps this chapter describes manually.

Bernstein, P. A. (1976). Synthesizing third normal form relations from functional dependencies. *ACM Transactions on Database Systems*, 1(4), 277–298.

#### Heath's Theorem and Lossless Decomposition

The formal guarantee of lossless decomposition comes from **Heath's Theorem** (1971):

Given a relation R with attributes ABC... if A → B holds in R, then R can be losslessly decomposed into R1 (A, B) and R2 (A, C...).

The condition: A → B must hold (A must functionally determine B, and A must be in both tables). When this condition is met, joining R1 and R2 on A reconstructs R exactly — no spurious tuples, no missing tuples.

This theorem underlies every normalization decomposition in this and the previous chapter. Each step removes a functional dependency (partial or transitive) by creating a new table keyed by the determinant — precisely the structure Heath's Theorem requires for losslessness.

Heath, I. J. (1971). Unacceptable file operations in a relational database. *Proceedings of the 1971 ACM SIGFIDET Workshop on Data Description, Access and Control*, 19–33.

#### Multivalued Dependencies and 4NF in Practice

Fourth Normal Form, while theoretically important, appears in practical database design more often than students expect — specifically in product catalogues, inventory systems, and configuration data.

A real-world 4NF scenario: a software product that supports multiple languages AND multiple operating systems. If language support and OS support are **independent** facts about the product (supporting English does not imply anything about Windows support), storing both in a single table forces a cross-product:

```
ProductID | Language | OS
P001      | English  | Windows
P001      | English  | Mac
P001      | French   | Windows
P001      | French   | Mac
```

Adding a third language requires adding two rows (one for each OS). Adding a third OS requires adding two rows (one for each language). The table has a **multivalued dependency** — `ProductID →→ Language` and `ProductID →→ OS` are independent of each other.

4NF decomposition:
```
ProductLanguages: (ProductID, Language)
ProductOS:        (ProductID, OS)
```

This reduces the N×M table to two N+M tables — significantly reducing redundancy and eliminating the coupling between language and OS management.

---

### References and Further Reading

1. Kent, W. (1983). A simple guide to five normal forms in relational database theory. *Communications of the ACM*, 26(2), 120–125. — The paper introducing the "key, the whole key, and nothing but the key" mnemonic. Required reading; freely available.

2. Date, C. J. (2019). *Database Design and Relational Theory: Normal Forms and All That Jazz* (2nd ed.). Apress. — Chapters 5–9 cover 3NF, BCNF, 4NF, and 5NF with the mathematical rigour the subject deserves.

3. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Chapter 14 covers 3NF and BCNF; Chapter 15 covers higher normal forms.

4. Codd, E. F. (1971). Further normalization of the data base relational model. *IBM Research Report*, RJ909. — The original 3NF paper. Codd introduced 3NF as an extension of his relational model.

5. Bernstein, P. A. (1976). Synthesizing third normal form relations from functional dependencies. *ACM Transactions on Database Systems*, 1(4), 277–298. — The synthesis algorithm.

6. Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley. — Chapter 1 explains the denormalization philosophy of dimensional modelling — the "why" behind star schema design.

7. Elmasri, R., & Navathe, S. (2016). *Fundamentals of Database Systems* (7th ed.). Pearson. — Chapter 14–15 cover all normal forms with formal proofs and the synthesis algorithm.

---

*Previous chapter: [Chapter 4 — Normalization: 1NF and 2NF — Removing Redundancy](../chapter-04-normalization-1nf-2nf/README.md)*

*Next chapter: [Chapter 6 — Constraints and Data Integrity: Encoding Business Rules](../chapter-06-constraints-integrity/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
