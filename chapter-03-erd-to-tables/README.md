# Chapter 3: From ERD to Tables — Mapping Diagrams to DDL

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Chapter 2 produced a logical data model — an entity-relationship diagram that represents the business's information structure. This chapter makes the crossing from logical to physical: translating every entity, attribute, and relationship in the ERD into concrete SQL Server DDL.

The translation is governed by a set of **mapping rules** — a systematic procedure that converts each ERD element into its physical equivalent. The rules are not arbitrary; each one reflects a deliberate design principle. Understanding the rules deeply means you can apply them confidently to new designs and adapt them when business requirements are unusual.

By the end of this chapter you will be able to:

- Apply the seven ERD-to-table mapping rules to produce a complete relational schema from any ERD
- Map strong entities, weak entities, and associative entities to tables with correct primary and foreign keys
- Choose between surrogate and natural primary keys with justification
- Map 1:1, 1:N, and M:N relationships to their correct physical implementations
- Map supertypes and subtypes to tables, choosing among implementation options
- Write complete DDL for a mapped schema including all constraints
- Verify the mapping by tracing every ERD element to a physical object
- Describe the design decisions made when building the CabotTrail OLTP schema

---

## Table of Contents

1. [The Mapping Process](#1-the-mapping-process)
2. [Rule 1: Strong Entities](#2-rule-1-strong-entities)
3. [Rule 2: Simple Attributes](#3-rule-2-simple-attributes)
4. [Rule 3: Multi-Valued Attributes](#4-rule-3-multi-valued-attributes)
5. [Rule 4: Weak Entities](#5-rule-4-weak-entities)
6. [Rule 5: One-to-Many Relationships](#6-rule-5-one-to-many-relationships)
7. [Rule 6: One-to-One Relationships](#7-rule-6-one-to-one-relationships)
8. [Rule 7: Many-to-Many Relationships](#8-rule-7-many-to-many-relationships)
9. [Mapping Supertypes and Subtypes](#9-mapping-supertypes-and-subtypes)
10. [Choosing Data Types During Mapping](#10-choosing-data-types-during-mapping)
11. [Naming Conventions](#11-naming-conventions)
12. [A Complete Mapping: The Loyalty Programme](#12-a-complete-mapping-the-loyalty-programme)
13. [Verifying the Mapping](#13-verifying-the-mapping)
14. [Chapter Summary](#14-chapter-summary)
15. [Review Questions](#15-review-questions)
16. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. The Mapping Process

The ERD-to-table mapping is systematic, not creative. Given an ERD, two designers following the same mapping rules should produce the same relational schema. This is by design — the logical model (ERD) captures the creative and analytical decisions; the mapping is the mechanical step that converts those decisions to SQL.

### 1.1 What the Mapping Produces

For each ERD element, the mapping produces one or more physical objects:

| ERD element | Physical result |
|---|---|
| Strong entity | One table |
| Weak entity | One table with composite or FK-assisted PK |
| Simple attribute | One column |
| Multi-valued attribute | One new table with FK back to the parent |
| Derived attribute | One computed column, or no column (compute at query time) |
| Composite attribute | Multiple columns (one per component) |
| 1:N relationship | FK column on the "many" table |
| 1:1 relationship | FK on one table (choose based on participation/optionality) |
| M:N relationship | One new junction table with FKs to both entities |
| Supertype/subtype | Two or more tables depending on strategy chosen |

### 1.2 The Mapping is Not the Only Output

The mapping produces DDL — `CREATE TABLE` statements. But the designer must also decide:

- What data type does each attribute map to?
- What constraints enforce which business rules?
- What indexes are needed for query performance?
- What naming conventions are used?

Data types and constraints are covered in Chapters 6 and 7. This chapter focuses on the structural mapping — getting the right tables with the right keys — and introduces data type selection as part of the practical worked example.

---

## 2. Rule 1: Strong Entities

**Every strong entity becomes one table.**

The table takes the entity name. If the entity has a natural candidate key that is stable, unique, and not null, use it as the primary key. If not, introduce a surrogate key.

```sql
-- ENTITY: Customer
-- Strong entity → one table
CREATE TABLE Sales.Customers
(
    CustomerID      INT             NOT NULL IDENTITY(1,1),
    CustomerName    NVARCHAR(100)   NOT NULL,
    CreditLimit     DECIMAL(18,2)   NOT NULL DEFAULT 0,
    AccountOpenedDate DATE          NULL,

    CONSTRAINT PK_Customers PRIMARY KEY (CustomerID)
);
```

The entity name (`Customer`) becomes the table name (`Customers` — pluralised in the physical model; convention varies but CabotTrail uses plural table names).

The `CustomerID IDENTITY(1,1)` is a surrogate key — introduced because `CustomerName` is not reliably unique (two customers could have the same name) and business-meaningful attributes like email are not always available or stable.

### 2.1 When to Use a Natural Key as PK

Use a natural key as the primary key when:
- It is **guaranteed unique** across all past and future instances
- It will **never change** for the life of the entity
- It is **always known** when a new instance is created
- It is **compact** (short enough to be efficient as a join key)

Strong candidates in CabotTrail:
- `StateProvinceCode` for provinces (2-char code, stable, unique, always known)
- `CountryName` or an ISO country code for countries

Weak candidates: `CustomerName` (not guaranteed unique), `EmailAddress` (may change), `ProductCode` (may be NULL for some products).

When in doubt, use a surrogate key.

---

## 3. Rule 2: Simple Attributes

**Every simple, single-valued attribute of an entity becomes a column in the entity's table.**

The column inherits the attribute name and gets an appropriate data type:

```sql
-- ENTITY: Product attributes mapped to columns
CREATE TABLE Inventory.Products
(
    ProductID               INT             NOT NULL IDENTITY(1,1),
    ProductCode             NVARCHAR(20)    NULL,       -- Optional attribute
    ProductName             NVARCHAR(200)   NOT NULL,
    UnitPrice               DECIMAL(18,2)   NOT NULL DEFAULT 0,
    RecommendedRetailPrice  DECIMAL(18,2)   NULL,
    TypicalWeightPerUnit    DECIMAL(18,3)   NULL,
    IsDiscontinued          BIT             NOT NULL DEFAULT 0,

    CONSTRAINT PK_Products PRIMARY KEY (ProductID)
);
```

**Derived attributes** (attributes computable from other stored values) have two mapping options:

Option 1 — **Omit from the physical model:** Do not store it. Compute it at query time using a SQL expression. This is the most storage-efficient approach and guarantees accuracy, but requires computation on every query.

```sql
-- Derived: LineTotal = Quantity × UnitPrice
-- Option 1: Compute in every query, do not store
SELECT Quantity * UnitPrice AS LineTotal FROM InvoiceLines;
```

Option 2 — **Store as a computed column:** Define it as a `AS expression` column, optionally `PERSISTED` to store the result physically:

```sql
-- Derived: LineTotalIncludingTax stored as a computed column
LineTotalIncludingTax AS (LineTotal + TaxAmount) PERSISTED
-- SQL Server computes and stores it; automatically maintained on UPDATE
```

The CabotTrail OLTP uses computed columns for `LineTotal` in `Sales.InvoiceLines`. The CabotTrail DW stores derived columns (`GrossProfit`, `GrossProfitMarginPct`) as regular populated columns loaded by ETL — a different approach appropriate for a read-optimised warehouse.

---

## 4. Rule 3: Multi-Valued Attributes

**Every multi-valued attribute becomes a new table.** The new table has:
- A FK referencing the parent entity's PK
- One column for the attribute value
- A PK (either the parent FK alone if each parent can have only one of each value, or a surrogate key if duplicate values per parent are allowed)

```
Entity: PRODUCT
Multi-valued attribute: CategoryName (a product can belong to many categories)
```

This becomes two tables plus a junction:

```sql
-- The multi-valued attribute becomes a separate entity
CREATE TABLE Inventory.ProductCategories
(
    ProductCategoryID   INT             NOT NULL IDENTITY(1,1),
    ProductCategoryName NVARCHAR(60)    NOT NULL,

    CONSTRAINT PK_ProductCategories PRIMARY KEY (ProductCategoryID),
    CONSTRAINT UQ_ProductCategoryName UNIQUE (ProductCategoryName)
);

-- And a junction table to resolve the M:N
CREATE TABLE Inventory.ProductCategoryAssignments
(
    ProductID           INT NOT NULL,
    ProductCategoryID   INT NOT NULL,

    CONSTRAINT PK_ProductCategoryAssignments
        PRIMARY KEY (ProductID, ProductCategoryID),
    CONSTRAINT FK_PCA_Product
        FOREIGN KEY (ProductID) REFERENCES Inventory.Products (ProductID),
    CONSTRAINT FK_PCA_Category
        FOREIGN KEY (ProductCategoryID)
        REFERENCES Inventory.ProductCategories (ProductCategoryID)
);
```

Notice: the multi-valued attribute `CategoryName` becomes a full entity (`ProductCategories`) with its own PK, plus a junction table linking it to `Products`. This is because the categories have their own identity — they are not just tags, they are entities that can be described independently (names, descriptions, margins).

If the multi-valued attribute had no independent existence — for example, a product having multiple phone numbers — the mapping is simpler:

```sql
-- No independent entity: just a value table
CREATE TABLE dbo.ProductPhoneNumbers
(
    PhoneID     INT             NOT NULL IDENTITY(1,1),
    ProductID   INT             NOT NULL,
    PhoneNumber NVARCHAR(20)    NOT NULL,
    PhoneType   NVARCHAR(20)    NOT NULL DEFAULT 'Main',

    CONSTRAINT PK_ProductPhoneNumbers PRIMARY KEY (PhoneID),
    CONSTRAINT FK_ProductPhone_Product
        FOREIGN KEY (ProductID) REFERENCES Inventory.Products (ProductID)
);
```

---

## 5. Rule 4: Weak Entities

**A weak entity becomes a table with a composite primary key that includes the FK to the owner entity.**

Or, if a surrogate key is used, the composite logical identity is enforced with a UNIQUE constraint rather than being the PK itself.

```
Entity: ORDER_LINE (weak — identity depends on ORDER)
Owner: ORDER
```

```sql
-- Option A: Composite PK (pure relational approach)
CREATE TABLE Sales.OrderLines
(
    OrderID             INT             NOT NULL,
    OrderLineID         INT             NOT NULL,   -- Line number within order
    ProductID           INT             NOT NULL,
    Quantity            INT             NOT NULL,
    PickedQuantity      INT             NOT NULL DEFAULT 0,
    UnitPrice           DECIMAL(18,2)   NOT NULL,

    CONSTRAINT PK_OrderLines
        PRIMARY KEY (OrderID, OrderLineID),         -- Composite PK
    CONSTRAINT FK_OrderLines_Order
        FOREIGN KEY (OrderID) REFERENCES Sales.Orders (OrderID)
);

-- Option B: Surrogate PK + UNIQUE constraint (CabotTrail's actual approach)
CREATE TABLE Sales.OrderLines
(
    OrderLineID         INT             NOT NULL IDENTITY(1,1),  -- Surrogate PK
    OrderID             INT             NOT NULL,
    ProductID           INT             NOT NULL,
    Quantity            INT             NOT NULL,
    PickedQuantity      INT             NOT NULL DEFAULT 0,
    UnitPrice           DECIMAL(18,2)   NOT NULL,

    CONSTRAINT PK_OrderLines
        PRIMARY KEY (OrderLineID),                  -- Surrogate PK
    CONSTRAINT FK_OrderLines_Order
        FOREIGN KEY (OrderID) REFERENCES Sales.Orders (OrderID)
    -- No explicit UNIQUE(OrderID, LineNumber) unless duplicate lines
    -- in the same order are prevented
);
```

CabotTrail uses Option B — surrogate keys throughout. This makes joins slightly simpler (single-column FKs) at the cost of the composite identity being invisible in the schema structure.

---

## 6. Rule 5: One-to-Many Relationships

**In a 1:N relationship, place a FK in the table on the "many" side, referencing the PK of the "one" side.**

This is the most frequently applied rule — most relationships in a relational database are 1:N.

```
Relationship: CUSTOMER ──|──O<── ORDER
One side: Customer
Many side: Order
```

```sql
-- FK goes on the Order table (the "many" side)
CREATE TABLE Sales.Orders
(
    OrderID             INT     NOT NULL IDENTITY(1,1),
    CustomerID          INT     NOT NULL,   -- FK to Customer
    OrderDate           DATE    NOT NULL,
    ExpectedDeliveryDate DATE   NULL,
    SalesRepID          INT     NULL,       -- FK to Employee (nullable — optional)
    CityID              INT     NOT NULL,   -- FK to Cities

    CONSTRAINT PK_Orders PRIMARY KEY (OrderID),
    CONSTRAINT FK_Orders_Customer
        FOREIGN KEY (CustomerID) REFERENCES Sales.Customers (CustomerID),
    CONSTRAINT FK_Orders_SalesRep
        FOREIGN KEY (SalesRepID) REFERENCES Application.Employees (EmployeeID),
    CONSTRAINT FK_Orders_City
        FOREIGN KEY (CityID) REFERENCES Application.Cities (CityID)
);
```

### 6.1 Mandatory vs Optional in the FK

The participation rules from the ERD translate directly to `NOT NULL` vs `NULL` on the FK:

| ERD participation (many side) | FK in table |
|---|---|
| Mandatory (every order has a customer) | `CustomerID INT NOT NULL` |
| Optional (an order may have no sales rep) | `SalesRepID INT NULL` |

If `CustomerID` could be NULL, an order could exist without a customer — violating the mandatory participation rule from the ERD. The `NOT NULL` constraint enforces the ERD's business rule.

### 6.2 Multiple Relationships to the Same Entity

`Sales.Orders` has FKs to `Customers`, `Employees` (as SalesRep), and `Cities`. Multiple FKs to different entities on the same table are normal and common.

A table can also have multiple FKs to the **same** entity — called **role relationships** or **multiple relationships**. A classic example: an `Invoice` references both a `Customer` (for billing) and a different `Customer` (for delivery, if different). Each FK must have a distinct name and a distinct column:

```sql
-- Multiple FKs to the same entity with different roles
CREATE TABLE Sales.Invoices
(
    InvoiceID           INT NOT NULL IDENTITY(1,1),
    BillingCustomerID   INT NOT NULL,   -- FK to Customer (billing)
    DeliveryCustomerID  INT NULL,       -- FK to Customer (delivery — may differ)
    OrderID             INT NOT NULL,
    InvoiceDate         DATE NOT NULL,
    DueDate             DATE NOT NULL,
    ...

    CONSTRAINT FK_Invoices_BillingCustomer
        FOREIGN KEY (BillingCustomerID) REFERENCES Sales.Customers (CustomerID),
    CONSTRAINT FK_Invoices_DeliveryCustomer
        FOREIGN KEY (DeliveryCustomerID) REFERENCES Sales.Customers (CustomerID)
);
```

---

## 7. Rule 6: One-to-One Relationships

**In a 1:1 relationship, place the FK on the table whose participation is optional — or on the table that logically extends the other.**

1:1 relationships require a judgment call about which table carries the FK. The decision is guided by:

1. **Participation**: If participation is mandatory on one side and optional on the other, put the FK on the optional side. The optional side may or may not have a corresponding row, so the FK on that table can be NULL without breaking the other.

2. **Logical extension**: If one entity is the "extension" of the other (a weak specialisation), put the FK on the extension table.

3. **Query frequency**: If queries almost always access the two entities together, consider merging them into one table.

### 7.1 Mapping 1:1 with Optional Participation

```
Relationship: EMPLOYEE ──|──O── EMPLOYEE_PARKING_PERMIT (optional — not all employees have parking)
```

```sql
-- FK on the optional side (Employee_Parking_Permit)
CREATE TABLE HR.EmployeeParkingPermits
(
    PermitID        INT     NOT NULL IDENTITY(1,1),
    EmployeeID      INT     NOT NULL UNIQUE,   -- UNIQUE enforces the 1:1
    PermitNumber    NVARCHAR(20) NOT NULL,
    ExpiryDate      DATE    NOT NULL,

    CONSTRAINT PK_EmployeeParkingPermits PRIMARY KEY (PermitID),
    CONSTRAINT FK_Permit_Employee
        FOREIGN KEY (EmployeeID) REFERENCES Application.Employees (EmployeeID),
    CONSTRAINT UQ_Permit_Employee UNIQUE (EmployeeID)
    -- The UNIQUE constraint on EmployeeID enforces the "at most one permit per employee" rule
);
```

The `UNIQUE` constraint on `EmployeeID` is what enforces the "one" in the 1:1 relationship. Without it, multiple parking permits per employee would be allowed — making it effectively 1:N.

### 7.2 The UNIQUE Constraint as a 1:1 Enforcer

This is a crucial physical design point: a FK alone does not enforce 1:1. A FK on `PermitID` → `EmployeeID` allows multiple permits for one employee (the FK allows repeated values). Adding `UNIQUE(EmployeeID)` to the permit table ensures each employee appears at most once — enforcing the 1:1 maximum.

---

## 8. Rule 7: Many-to-Many Relationships

**Every M:N relationship becomes a junction table.** The junction table:
- Has a FK referencing the PK of each participating entity
- Has a PK that is the composite of both FKs (or a surrogate key plus a UNIQUE constraint on the FK combination)
- May carry attributes of the relationship as additional columns

```
Relationship: PRODUCT ──O<──>O── PRODUCT_CATEGORY
```

```sql
CREATE TABLE Inventory.ProductCategoryAssignments
(
    ProductID           INT NOT NULL,
    ProductCategoryID   INT NOT NULL,

    CONSTRAINT PK_ProductCategoryAssignments
        PRIMARY KEY (ProductID, ProductCategoryID),
    CONSTRAINT FK_PCA_Product
        FOREIGN KEY (ProductID)
        REFERENCES Inventory.Products (ProductID),
    CONSTRAINT FK_PCA_ProductCategory
        FOREIGN KEY (ProductCategoryID)
        REFERENCES Inventory.ProductCategories (ProductCategoryID)
);
```

### 8.1 Adding a Surrogate Key to the Junction Table

Some designs add a surrogate key to the junction table for convenience — particularly when the junction is referenced by other tables:

```sql
-- Junction with surrogate key: useful when other tables need to FK to the assignment
CREATE TABLE Inventory.ProductCategoryAssignments
(
    AssignmentID        INT     NOT NULL IDENTITY(1,1),
    ProductID           INT     NOT NULL,
    ProductCategoryID   INT     NOT NULL,

    CONSTRAINT PK_ProductCategoryAssignments PRIMARY KEY (AssignmentID),
    CONSTRAINT UQ_PCA_Combination UNIQUE (ProductID, ProductCategoryID),
    CONSTRAINT FK_PCA_Product
        FOREIGN KEY (ProductID) REFERENCES Inventory.Products (ProductID),
    CONSTRAINT FK_PCA_ProductCategory
        FOREIGN KEY (ProductCategoryID)
        REFERENCES Inventory.ProductCategories (ProductCategoryID)
);
```

The `UNIQUE(ProductID, ProductCategoryID)` constraint preserves the M:N resolution even with the surrogate key.

### 8.2 When the Junction Becomes a Full Entity

Sometimes a junction table grows attributes until it becomes a full entity in its own right. The test: does the junction table have attributes that are not derivable from either parent entity? If yes, it is an **associative entity** and should be named and treated as a first-class entity.

In CabotTrail, if `ProductCategoryAssignments` needed to track which product manager approved the category assignment and when, it would become a full entity — not just a structural bridge.

---

## 9. Mapping Supertypes and Subtypes

Chapter 2 identified three physical implementation options. This section shows the DDL for each, using `People`, `Employees`, and `Customers` as the example.

### 9.1 Option 1: Table per Subtype (CabotTrail's Approach)

```sql
-- Supertype table: common attributes
CREATE TABLE Application.People
(
    PersonID        INT             NOT NULL IDENTITY(1,1),
    FullName        NVARCHAR(100)   NOT NULL,
    PreferredName   NVARCHAR(60)    NOT NULL,
    EmailAddress    NVARCHAR(256)   NULL,
    PhoneNumber     NVARCHAR(20)    NULL,

    CONSTRAINT PK_People PRIMARY KEY (PersonID)
);

-- Subtype 1: Employee-specific attributes
CREATE TABLE Application.Employees
(
    EmployeeID      INT             NOT NULL IDENTITY(1,1),
    PersonID        INT             NOT NULL UNIQUE,   -- 1:1 link to People
    JobTitle        NVARCHAR(100)   NOT NULL,
    Department      NVARCHAR(60)    NULL,
    HireDate        DATE            NULL,
    IsActive        BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_Employees PRIMARY KEY (EmployeeID),
    CONSTRAINT FK_Employees_Person
        FOREIGN KEY (PersonID) REFERENCES Application.People (PersonID),
    CONSTRAINT UQ_Employees_Person UNIQUE (PersonID)
);

-- Subtype 2: Customer-specific attributes
CREATE TABLE Sales.Customers
(
    CustomerID          INT             NOT NULL IDENTITY(1,1),
    CustomerName        NVARCHAR(100)   NOT NULL,   -- May differ from PersonID.FullName
    CustomerGroupName   NVARCHAR(100)   NULL,
    CreditLimit         DECIMAL(18,2)   NOT NULL DEFAULT 0,
    AccountOpenedDate   DATE            NULL,

    CONSTRAINT PK_Customers PRIMARY KEY (CustomerID)
    -- Note: CabotTrail Customers does not directly FK to People
    -- (wholesale customers are businesses, not individuals)
);
```

Note the design decision: CabotTrail `Customers` does not reference `People` because wholesale customers are organisations, not individuals. `Employees` references `People` because employees are individuals. This asymmetry reflects business reality.

### 9.2 Option 2: Single Table with NULLs

```sql
-- All attributes in one table; NULLs for non-applicable subtypes
CREATE TABLE dbo.AllPersons
(
    PersonID            INT             NOT NULL IDENTITY(1,1),
    PersonType          NVARCHAR(20)    NOT NULL,   -- 'Employee' or 'Customer'
    FullName            NVARCHAR(100)   NOT NULL,
    EmailAddress        NVARCHAR(256)   NULL,
    -- Employee-specific (NULL for customers)
    JobTitle            NVARCHAR(100)   NULL,
    HireDate            DATE            NULL,
    -- Customer-specific (NULL for employees)
    CreditLimit         DECIMAL(18,2)   NULL,
    AccountOpenedDate   DATE            NULL,

    CONSTRAINT PK_AllPersons PRIMARY KEY (PersonID)
);
```

Drawbacks:
- NULL columns are semantically meaningless for non-applicable subtypes
- Cannot enforce subtype-specific NOT NULL constraints (e.g., employees must have `JobTitle`, but the column must allow NULL for customers)
- Table grows wide as subtypes multiply

### 9.3 Option 3: Table per Concrete Class (No Supertype Table)

```sql
-- No shared table: each subtype is fully self-contained
CREATE TABLE Application.Employees
(
    EmployeeID  INT NOT NULL IDENTITY(1,1),
    FullName    NVARCHAR(100) NOT NULL,      -- Duplicated from supertype
    EmailAddress NVARCHAR(256) NULL,         -- Duplicated from supertype
    JobTitle    NVARCHAR(100) NOT NULL,
    HireDate    DATE NULL,
    ...
);

CREATE TABLE Sales.Customers
(
    CustomerID  INT NOT NULL IDENTITY(1,1),
    FullName    NVARCHAR(100) NOT NULL,      -- Duplicated from supertype
    EmailAddress NVARCHAR(256) NULL,         -- Duplicated from supertype
    CreditLimit DECIMAL(18,2) NOT NULL DEFAULT 0,
    ...
);
```

Advantage: simple queries — no joins needed to get all attributes.
Disadvantage: common attributes are duplicated; changes to common attributes require updating multiple tables.

**CabotTrail uses Option 1 (table per subtype)** — the most normalized and maintainable approach.

---

## 10. Choosing Data Types During Mapping

When mapping attributes to columns, every attribute requires a data type decision. Chapter 3 of *SQL for Analysts* covered the type families; this section applies them to design decisions.

### 10.1 The Data Type Selection Checklist

For each attribute, ask:

1. **What is the domain?** Numbers, text, dates, true/false values?
2. **What is the range of valid values?** Affects numeric type size.
3. **What is the maximum length?** Affects character type size.
4. **Is precision critical?** Financial amounts need DECIMAL, not FLOAT.
5. **Will this be compared or sorted?** Date types sort correctly; date-as-string does not.
6. **Will this be NULLable?** Mandatory attributes should be NOT NULL.

### 10.2 Common Mapping Decisions

| Attribute type | Recommended SQL Server type | Notes |
|---|---|---|
| Surrogate key | `INT NOT NULL IDENTITY(1,1)` | Use `BIGINT` if > 2 billion rows possible |
| Short code (e.g., province code) | `NCHAR(2) NOT NULL` | Fixed-length for known-length codes |
| Name or description | `NVARCHAR(n) NOT NULL` | Choose n based on maximum expected length |
| Long text | `NVARCHAR(MAX)` | Only if truly variable, large content |
| Price / financial amount | `DECIMAL(18,2) NOT NULL` | Never FLOAT for money |
| Rate or percentage | `DECIMAL(8,4)` | 4 decimal places for percentages/rates |
| Date only | `DATE` | Not DATETIME unless time is needed |
| Date and time | `DATETIME2(0-7)` | Prefer over legacy DATETIME |
| Boolean flag | `BIT NOT NULL DEFAULT 0` | 1=yes/true, 0=no/false |
| Count or quantity | `INT NOT NULL DEFAULT 0` | TINYINT/SMALLINT for small ranges |
| Weight / measurement | `DECIMAL(10,3)` | Appropriate precision for measurement |

### 10.3 Mapping the Loyalty Programme Attributes

Applying the checklist to each `LoyaltyAccount` attribute:

| Attribute | Decision | Rationale |
|---|---|---|
| AccountID | `INT NOT NULL IDENTITY(1,1)` | Surrogate PK |
| CustomerID | `INT NOT NULL` | FK — must always reference a customer |
| PointBalance | `INT NOT NULL DEFAULT 0` | Points are whole numbers; cannot go below 0 (add CHECK) |
| TierID | `TINYINT NOT NULL` | Four tiers; TINYINT (0–255) is sufficient |
| AccountOpenedDate | `DATE NOT NULL DEFAULT GETDATE()` | Date only; defaults to today |

---

## 11. Naming Conventions

Consistent naming makes a database self-documenting. CabotTrail follows conventions worth adopting.

### 11.1 Table Names

- **Plural** noun describing the entity: `Customers`, `Orders`, `Products`
- **PascalCase** (each word capitalised): `OrderLines`, `ProductCategories`, `StateProvinces`
- **Schema-qualified**: `Sales.Orders`, `Inventory.Products`, `Application.Employees`
- **No abbreviations** unless universal: `Qty` is ambiguous; `Quantity` is clear

### 11.2 Column Names

- **PascalCase**: `CustomerName`, `AccountOpenedDate`, `IsDiscontinued`
- **Descriptive**: `CreditLimit` not `Limit`; `AccountOpenedDate` not `OpenDate`
- **Consistent FK naming**: FK columns share the name of the referenced PK column: `CustomerID` in `Orders` references `CustomerID` in `Customers`
- **Boolean columns start with `Is` or `Has`**: `IsDiscontinued`, `IsActive`, `HasLoyaltyAccount`
- **Date columns end with `Date`**: `OrderDate`, `HireDate`, `AccountOpenedDate`

### 11.3 Constraint Names

Explicit constraint names follow a pattern that makes them identifiable in error messages:

| Constraint | Pattern | Example |
|---|---|---|
| Primary key | `PK_TableName` | `PK_Customers` |
| Foreign key | `FK_TableName_ReferencedTable` | `FK_Orders_Customers` |
| Unique | `UQ_TableName_Column(s)` | `UQ_Customers_Email` |
| Check | `CK_TableName_Column` | `CK_Products_UnitPrice` |
| Default | `DF_TableName_Column` | `DF_Orders_OrderDate` |

When SQL Server generates constraint names automatically (unnamed constraints), the generated names are cryptic: `PK__Customers__A1B2C3D4E5`. These are impossible to reference in ALTER TABLE statements. **Always name constraints explicitly**.

---

## 12. A Complete Mapping: The Loyalty Programme

Applying all seven rules to the Loyalty Programme ERD from Chapter 2.

### 12.1 Step 1: Map Strong Entities

```sql
USE CabotTrailOutdoor;
GO

-- Rule 1: Strong entity TIER
CREATE TABLE Sales.LoyaltyTiers
(
    TierID              TINYINT         NOT NULL IDENTITY(1,1),
    TierName            NVARCHAR(20)    NOT NULL,
    MinimumPoints       INT             NOT NULL DEFAULT 0,
    BenefitDescription  NVARCHAR(200)   NULL,

    CONSTRAINT PK_LoyaltyTiers      PRIMARY KEY (TierID),
    CONSTRAINT UQ_LoyaltyTiers_Name UNIQUE (TierName),
    CONSTRAINT CK_LoyaltyTiers_MinPts CHECK (MinimumPoints >= 0)
);
GO

-- Rule 1: Strong entity REWARD
CREATE TABLE Sales.LoyaltyRewards
(
    RewardID            INT             NOT NULL IDENTITY(1,1),
    RewardName          NVARCHAR(100)   NOT NULL,
    RewardDescription   NVARCHAR(500)   NULL,
    PointCost           INT             NOT NULL,
    IsAvailable         BIT             NOT NULL DEFAULT 1,

    CONSTRAINT PK_LoyaltyRewards        PRIMARY KEY (RewardID),
    CONSTRAINT CK_LoyaltyRewards_Cost   CHECK (PointCost > 0)
);
GO
```

### 12.2 Step 2: Map LoyaltyAccount (1:1 with Customer)

```sql
-- Rule 6: 1:1 relationship — LoyaltyAccount extends Customer
-- FK on the optional side (not every customer has an account — yet)
CREATE TABLE Sales.LoyaltyAccounts
(
    AccountID           INT             NOT NULL IDENTITY(1,1),
    CustomerID          INT             NOT NULL,   -- FK → Customers
    PointBalance        INT             NOT NULL DEFAULT 0,
    TierID              TINYINT         NOT NULL,   -- FK → LoyaltyTiers
    AccountOpenedDate   DATE            NOT NULL DEFAULT GETDATE(),

    CONSTRAINT PK_LoyaltyAccounts       PRIMARY KEY (AccountID),
    CONSTRAINT UQ_LoyaltyAccounts_Cust  UNIQUE (CustomerID),   -- Enforces 1:1
    CONSTRAINT FK_LoyaltyAccounts_Cust
        FOREIGN KEY (CustomerID) REFERENCES Sales.Customers (CustomerID),
    CONSTRAINT FK_LoyaltyAccounts_Tier
        FOREIGN KEY (TierID) REFERENCES Sales.LoyaltyTiers (TierID),
    CONSTRAINT CK_LoyaltyAccounts_Bal   CHECK (PointBalance >= 0)
);
GO
```

### 12.3 Step 3: Map LoyaltyTransaction (1:N from LoyaltyAccount)

```sql
-- Rule 5: 1:N relationship — LoyaltyAccount has many Transactions
-- Rule 4: Weak entity — Transaction identity depends on Account context
CREATE TABLE Sales.LoyaltyTransactions
(
    TransactionID       INT             NOT NULL IDENTITY(1,1),
    AccountID           INT             NOT NULL,   -- FK → LoyaltyAccounts
    TransactionDate     DATE            NOT NULL DEFAULT GETDATE(),
    PointsAmount        INT             NOT NULL,   -- Positive = earn, negative = redeem
    TransactionType     NVARCHAR(10)    NOT NULL,   -- 'EARN' or 'REDEEM'
    RewardID            INT             NULL,        -- FK → Rewards (nullable — earn txns have none)
    InvoiceID           INT             NULL,        -- FK → Invoices (nullable — redeem txns have none)
    Notes               NVARCHAR(200)   NULL,

    CONSTRAINT PK_LoyaltyTransactions PRIMARY KEY (TransactionID),
    CONSTRAINT FK_LoyaltyTxn_Account
        FOREIGN KEY (AccountID) REFERENCES Sales.LoyaltyAccounts (AccountID),
    CONSTRAINT FK_LoyaltyTxn_Reward
        FOREIGN KEY (RewardID) REFERENCES Sales.LoyaltyRewards (RewardID),
    CONSTRAINT FK_LoyaltyTxn_Invoice
        FOREIGN KEY (InvoiceID) REFERENCES Sales.Invoices (InvoiceID),
    CONSTRAINT CK_LoyaltyTxn_Type
        CHECK (TransactionType IN ('EARN', 'REDEEM')),
    CONSTRAINT CK_LoyaltyTxn_EarnPos
        CHECK (
            (TransactionType = 'EARN'   AND PointsAmount > 0) OR
            (TransactionType = 'REDEEM' AND PointsAmount < 0)
        )
);
GO
```

The CHECK constraint on `PointsAmount` encodes a business rule: earn transactions must have positive points, redeem transactions must have negative points. This level of constraint design is covered in Chapter 6.

---

## 13. Verifying the Mapping

After completing the DDL, verify that every element of the ERD has been accounted for.

### 13.1 The Mapping Verification Checklist

```
□ Every entity in the ERD has a corresponding table
□ Every attribute in the ERD is a column in the appropriate table
□ Every derived attribute is either a computed column or intentionally omitted
□ Every multi-valued attribute has a separate table
□ Every 1:N relationship has a FK on the "many" table
□ Every 1:1 relationship has a FK with a UNIQUE constraint
□ Every M:N relationship has a junction table
□ Every weak entity has its ownership relationship enforced by FK + composite PK or UNIQUE
□ Every mandatory participation rule is enforced by NOT NULL on the FK
□ Every optional participation is modeled as a nullable FK where appropriate
□ Every constraint is explicitly named
```

### 13.2 Tracing Back to the ERD

For each table in the physical model, identify its ERD origin:

| Physical table | ERD origin | Rule applied |
|---|---|---|
| `Sales.LoyaltyTiers` | `TIER` entity | Rule 1 (strong entity) |
| `Sales.LoyaltyRewards` | `REWARD` entity | Rule 1 (strong entity) |
| `Sales.LoyaltyAccounts` | `LOYALTY_ACCOUNT` entity + 1:1 relationship | Rules 1 + 6 |
| `Sales.LoyaltyTransactions` | `LOYALTY_TRANSACTION` entity + 1:N relationships | Rules 4 + 5 |

No ERD element was skipped. No physical table exists without an ERD origin. The mapping is complete.

### 13.3 Querying the Schema to Verify

After creating the tables, use the system catalog to confirm the structure:

```sql
-- Verify all Loyalty tables exist
SELECT TABLE_SCHEMA, TABLE_NAME
FROM   INFORMATION_SCHEMA.TABLES
WHERE  TABLE_NAME LIKE 'Loyalty%'
ORDER BY TABLE_SCHEMA, TABLE_NAME;

-- Verify all FK relationships are in place
SELECT
    fk.name                         AS RelationshipName,
    SCHEMA_NAME(tp.schema_id) + '.' + tp.name AS ParentTable,
    cp.name                         AS ParentColumn,
    SCHEMA_NAME(tr.schema_id) + '.' + tr.name AS ReferencedTable,
    cr.name                         AS ReferencedColumn
FROM    sys.foreign_keys fk
INNER JOIN sys.foreign_key_columns fkc ON fkc.constraint_object_id = fk.object_id
INNER JOIN sys.tables tp    ON tp.object_id = fkc.parent_object_id
INNER JOIN sys.tables tr    ON tr.object_id = fkc.referenced_object_id
INNER JOIN sys.columns cp   ON cp.object_id = fkc.parent_object_id
                           AND cp.column_id  = fkc.parent_column_id
INNER JOIN sys.columns cr   ON cr.object_id = fkc.referenced_object_id
                           AND cr.column_id  = fkc.referenced_column_id
WHERE   tp.name LIKE 'Loyalty%'
ORDER BY tp.name, fk.name;
```

---

## 14. Chapter Summary

- The ERD-to-table mapping is **systematic and deterministic** — each ERD element maps to a physical object by a defined rule. Given the same ERD, two designers applying the same rules produce the same schema.

- **Rule 1:** Strong entities → tables. **Rule 2:** Simple attributes → columns. **Rule 3:** Multi-valued attributes → new tables with FK to parent. **Rule 4:** Weak entities → tables with composite PK or FK-assisted identity.

- **Rule 5 (1:N):** FK on the "many" side. Mandatory participation → NOT NULL FK. Optional participation → nullable FK.

- **Rule 6 (1:1):** FK on the optional/extension side, with a `UNIQUE` constraint to enforce the maximum cardinality. Without the UNIQUE constraint, a FK alone allows 1:N.

- **Rule 7 (M:N):** Junction table with FKs to both entities, composite PK, and optional surrogate key + UNIQUE constraint.

- **Supertype/subtype mapping:** Three options — table per subtype (most normalized), single table with NULLs (simplest queries), table per concrete class (no joins but duplication). CabotTrail uses table per subtype.

- **Data type selection** follows a checklist: domain, range, length, precision needs, sort behaviour, and nullability.

- **Naming conventions** — plural tables, PascalCase, descriptive names, explicit constraint names following `PK_/FK_/UQ_/CK_/DF_` patterns — make the schema self-documenting.

- **Verify the mapping** by tracing every ERD element to a physical object and every physical object back to an ERD element. Use system catalog queries to confirm the physical structure matches the design.

---

## 15. Review Questions

1. The mapping rules state that in a 1:N relationship, the FK goes on the "many" side. Explain *why* this is the rule — what would happen structurally if you put the FK on the "one" side instead? Use the Customer–Order relationship as your example.

2. A developer creates the following DDL for a 1:1 relationship between `Employee` and `ParkingPermit`:
   ```sql
   CREATE TABLE HR.ParkingPermits (
       PermitID   INT NOT NULL IDENTITY(1,1) PRIMARY KEY,
       EmployeeID INT NOT NULL,
       PermitCode NVARCHAR(20) NOT NULL,
       FOREIGN KEY (EmployeeID) REFERENCES Application.Employees (EmployeeID)
   );
   ```
   Identify the missing design element that should enforce the 1:1 maximum cardinality, explain what is wrong without it, and write the corrected DDL.

3. Write the complete DDL (applying all mapping rules with named constraints) for the following ERD narrative:
   > A **Warehouse** has a name, a city, and a capacity in cubic metres. A warehouse stores many **ProductBatches**. Each batch belongs to exactly one warehouse, has a batch number (unique within a warehouse), a product ID, a quantity, and a received date. A product can have many batches across multiple warehouses.

4. The `LoyaltyTransaction` table in section 12.3 has two nullable FKs: `RewardID` and `InvoiceID`. Explain the business rules these nullable FKs represent. Now write a CHECK constraint that enforces: "An EARN transaction must reference an InvoiceID, and a REDEEM transaction must reference a RewardID."

5. For the supertype/subtype relationship between `People`, `Employees`, and `Customers`, compare Option 1 (table per subtype) and Option 2 (single table with NULLs). Write a specific query against each design that retrieves an employee's full name, email address, and job title — and compare the complexity of each version.

6. A business analyst tells you: "We need to store the delivery address for each order — street, city, province, and postal code." Describe two different ways to model this in an ERD and their corresponding physical implementations. What business question determines which approach is correct?

7. Convert the following unnormalised description to an ERD and then to DDL, applying all mapping rules:
   > "Each **sales rep** has an ID, a name, and an email. A sales rep manages many **accounts**. Each account has an account number, a company name, and a credit limit. An account can have many **contacts** — people at the company — each with a name, a phone number, and a primary contact flag."

8. Explain the relationship between a well-named FK constraint and the ability to maintain a production database. Give a specific example of a maintenance task that becomes easier when FK constraints are named explicitly following the `FK_TableName_ReferencedTable` pattern.

---

## 🔍 Deeper Dive

### Going Further with the Mapping Process

#### The Formal Relational Schema Notation

Academic database texts express a relational schema in formal notation rather than DDL. The notation uses an underline for primary key attributes and an asterisk (or italic) for foreign keys:

```
Customers(<u>CustomerID</u>, CustomerName, CreditLimit, AccountOpenedDate)
Orders(<u>OrderID</u>, *CustomerID, OrderDate, ExpectedDeliveryDate)
OrderLines(<u>OrderLineID</u>, *OrderID, *ProductID, Quantity, PickedQuantity, UnitPrice)
```

The underlined attributes are the primary key. The asterisked attributes are foreign keys. This notation is useful for quickly communicating a schema without full DDL — for whiteboard discussions, exam answers, and academic papers.

Understanding the formal notation helps when reading database textbooks (Connolly & Begg, Elmasri & Navathe) which use it throughout.

#### The Chen-to-Relational Mapping Algorithm

The formal mapping algorithm from ER model to relational schema is described in Elmasri & Navathe as a 9-step procedure:

1. Map regular (strong) entity types
2. Map weak entity types
3. Map binary 1:1 relationship types
4. Map binary 1:N relationship types
5. Map binary M:N relationship types
6. Map multivalued attributes
7. Map N-ary relationship types (three or more entities)
8. Map specialisation/generalisation (supertypes/subtypes)
9. Map union types (categories — a single entity that can be one of several types)

This chapter covers steps 1–8 in the context of the Crow's Foot notation. N-ary relationships (step 7) and union types (step 9) are less common in business systems but are covered in Elmasri & Navathe Chapter 9.

#### Referential Integrity Actions in Depth

Chapter 1 introduced `NO ACTION`, `CASCADE`, `SET NULL`, and `SET DEFAULT` as referential integrity actions on FK constraints. These deserve more depth:

```sql
-- ON DELETE CASCADE: deleting a Customer deletes all their Orders automatically
CONSTRAINT FK_Orders_Customer
    FOREIGN KEY (CustomerID) REFERENCES Sales.Customers (CustomerID)
    ON DELETE CASCADE

-- ON DELETE SET NULL: deleting a SalesRep sets the FK to NULL (order keeps record)
CONSTRAINT FK_Orders_SalesRep
    FOREIGN KEY (SalesRepID) REFERENCES Application.Employees (EmployeeID)
    ON DELETE SET NULL

-- ON UPDATE CASCADE: if CustomerID changes, propagate to all Orders
CONSTRAINT FK_Orders_Customer
    FOREIGN KEY (CustomerID) REFERENCES Sales.Customers (CustomerID)
    ON UPDATE CASCADE
```

**Cascade delete** is powerful but dangerous — a single DELETE can remove thousands of rows across multiple tables. Use it only when child rows are truly meaningless without the parent (e.g., invoice lines without an invoice).

**Set NULL on delete** is appropriate when the referenced entity is optional — an order placed by a sales rep who has since left the company should not be deleted; the FK should become NULL.

**No action (default)** prevents deletion of referenced rows — safest default. Requires the application to handle the deletion of dependents before deleting the parent.

The choice of referential action is a **business rule**: what does the business want to happen when a customer is deleted? This question must be answered by the business, not decided unilaterally by the designer.

Microsoft documentation:
[FOREIGN KEY Constraints](https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints)

---

### References and Further Reading

1. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Chapter 16 covers the ER-to-relational mapping with comprehensive worked examples.

2. Elmasri, R., & Navathe, S. (2016). *Fundamentals of Database Systems* (7th ed.). Pearson. — Chapter 9 covers the formal 9-step ER-to-relational mapping algorithm.

3. Date, C. J. (2019). *Database Design and Relational Theory: Normal Forms and All That Jazz* (2nd ed.). Apress. — Chapter 2 covers the mapping from logical to physical model from a rigorous theoretical perspective.

4. Microsoft. (2024). *FOREIGN KEY Constraints*. [https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints](https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints)

5. Microsoft. (2024). *Primary and Unique Key Constraints*. [https://learn.microsoft.com/en-us/sql/relational-databases/tables/unique-constraints-and-check-constraints](https://learn.microsoft.com/en-us/sql/relational-databases/tables/unique-constraints-and-check-constraints)

6. Microsoft. (2024). *Computed Columns*. [https://learn.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table](https://learn.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table)

---

*Previous chapter: [Chapter 2 — Entity-Relationship Diagrams: Drawing the Business](../chapter-02-entity-relationship-diagrams/README.md)*

*Next chapter: [Chapter 4 — Normalization: 1NF and 2NF — Removing Redundancy](../chapter-04-normalization-1nf-2nf/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
