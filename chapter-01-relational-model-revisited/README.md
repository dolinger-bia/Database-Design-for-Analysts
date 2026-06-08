# Chapter 1: The Relational Model Revisited — From Querying to Designing

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

In DBAS 5010 you learned to read databases. You queried `dim.Customer`, joined it to `fact.Sales`, filtered by territory, and aggregated by category. You became fluent in the language of databases as a *user*. This course starts the same databases from a different angle: not as things to query, but as things to design.

That shift — from user to designer — is the conceptual leap this chapter bridges. A user asks "what does this table contain?" A designer asks "why is this table structured the way it is, and what rules governed that structure?" The answers to those questions are the content of this course.

This chapter revisits the foundations of the relational model — not to repeat what DBAS 5010 covered, but to deepen your understanding of *why* the relational model works the way it does. By the time you finish, you will see the CabotTrail databases not just as collections of tables, but as formal representations of a business's reality.

By the end of this chapter you will be able to:

- Describe the three perspectives on a database: external, conceptual, and internal
- Define the terms entity, attribute, and relationship in the context of database design
- Distinguish between a logical data model and a physical data model
- Explain what a relation is and how it differs from a simple table
- Read an existing database schema and identify its design decisions
- Explain the purpose of primary and foreign keys at a deeper level than their mechanical function
- Describe the CabotTrail OLTP schema and explain the design choices it reflects

---

## Table of Contents

1. [The Designer's Perspective](#1-the-designers-perspective)
2. [The Three-Schema Architecture](#2-the-three-schema-architecture)
3. [Entities, Attributes, and Relationships](#3-entities-attributes-and-relationships)
4. [Relations, Not Just Tables](#4-relations-not-just-tables)
5. [Logical vs Physical Models](#5-logical-vs-physical-models)
6. [Keys: Identity and Connection](#6-keys-identity-and-connection)
7. [Reading the CabotTrail OLTP Schema](#7-reading-the-cabot-trail-oltp-schema)
8. [Design Decisions in the Wild](#8-design-decisions-in-the-wild)
9. [The Cost of Bad Design](#9-the-cost-of-bad-design)
10. [Chapter Summary](#10-chapter-summary)
11. [Review Questions](#11-review-questions)
12. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. The Designer's Perspective

When you queried `CabotTrailOutdoorsSales` in DBAS 5010, you worked with a finished product. The tables were there. The columns had names. The foreign keys connected the dimensions to the fact table. You wrote SQL against a structure someone else had already decided on.

Database design is the process of making those decisions. Before any table existed, a designer answered hundreds of questions:

- What information does the business actually need to store?
- What are the natural groupings of that information?
- How do different pieces of information relate to each other?
- What rules must data satisfy to be valid?
- What queries will analysts need to run, and how do we structure data to support them efficiently?

Some of those decisions are visible in the final schema — you can see that `fact.Sales` has a `CustomerKey` column that links to `dim.Customer.CustomerKey`. Some are invisible — the absence of a column is also a design decision. Why does `fact.Sales` not have a `CustomerName` column? Because that information is in `dim.Customer` and referencing it there via a key is more efficient, more consistent, and more maintainable.

Understanding why things are the way they are is what separates a database analyst from a database designer.

### 1.1 Design as Translation

Database design is fundamentally a translation activity. The input is a description of a business — usually given in natural language by people who know the business but not databases. The output is a formal specification in the language of the relational model.

This translation is not mechanical. Natural language is ambiguous. People describe things differently. Business rules are sometimes implicit, sometimes contradictory, sometimes simply unknown. A designer must ask questions, probe ambiguities, and make defensible decisions about things that are not clearly specified.

Consider this business description:

> "CabotTrail sells outdoor gear products to wholesale and retail customers across Canada. Each customer can place multiple orders, and each order can include multiple products. We need to track what was ordered, at what price, and when."

This description sounds complete. It is not. Questions a designer would ask:

- Can one order span multiple invoices, or is one order always one invoice?
- Can a product appear on an order more than once (two separate lines for the same product)?
- Is "price" the price at the time of the order, or the current catalogue price?
- What does "when" mean — the order date, the invoice date, or the delivery date?
- What happens to historical orders if a product is discontinued?
- What happens to historical invoices if a customer is deleted?

None of these are answerable from the description alone. Every one of them affects the database design. Chapter 2 covers techniques for surfacing these questions through entity-relationship diagramming.

### 1.2 The Designer Is Not the Last Word

An important professional truth: the database designer does not decide what a business does. The designer represents the business's reality in a formal structure. If the business says "one order can have one customer," the designer models that rule — even if the designer privately suspects it will change. If it does change, the design changes.

This is why database design requires close collaboration with business stakeholders. The designer brings technical expertise. The stakeholder brings business knowledge. Neither can produce a good design alone.

---

## 2. The Three-Schema Architecture

Database systems separate concerns into three levels, a framework called the **ANSI/SPARC three-schema architecture**. Understanding these levels explains why databases are designed the way they are.

### 2.1 The External Schema (View Level)

The **external schema** is what individual users and applications see — their personalised view of the data. Different users may see different subsets of the database, with different column names, different levels of detail, and different aggregations.

In the CabotTrail environment, the external schemas are the data marts: `CabotTrailOutdoorsSales`, `CabotTrailOutdoorsReturns`, `CabotTrailOutdoorsPurchasing`. Each mart presents a curated view of the data warehouse optimised for a specific audience. A sales analyst sees `fact.Sales` with customer and product dimensions. A purchasing manager sees `fact.Purchasing` with supplier and product dimensions.

These are not the "real" database. They are views onto it.

### 2.2 The Conceptual Schema (Logical Level)

The **conceptual schema** is the unified, complete description of the database — all entities, all relationships, all constraints — independent of how the data is physically stored and independent of which user sees what.

In the CabotTrail environment, the conceptual schema is the data warehouse (`CabotTrailOutdoorDW`) and, at a more fundamental level, the OLTP (`CabotTrailOutdoor`). These represent the full, integrated picture of the business's data.

The conceptual schema is what entity-relationship diagrams (Chapter 2) capture. It is the logical model — the definition of *what* the data is, not *how* it is stored.

### 2.3 The Internal Schema (Physical Level)

The **internal schema** is how the data is physically stored on disk — file organisation, index structures, page sizes, storage engine details. This level is largely invisible to SQL developers and analysts; it is the concern of database administrators and storage engineers.

In SQL Server, the internal schema includes the clustered index structure of each table, the non-clustered index definitions, the filegroup allocation, and the page compression settings. Chapter 7 touches the edge of the internal schema when discussing index design.

### 2.4 Why the Separation Matters

The three-schema architecture enforces **data independence** — the ability to change one level without breaking the others. You can add an index (internal schema change) without changing any queries (external schema). You can add a column to a table (conceptual schema change) without changing the physical storage format (internal schema). You can create a new data mart view (external schema change) without changing the underlying tables (conceptual schema).

This independence is one of the most practically important properties of relational databases. It is what makes it possible to optimise a query by adding an index without breaking the application that uses the database.

---

## 3. Entities, Attributes, and Relationships

The vocabulary of database design has precise meaning. These three terms — entity, attribute, relationship — are the building blocks of every data model.

### 3.1 Entities

An **entity** is a distinguishable thing in the real world about which the business needs to store information. Entities become tables.

An entity is not a specific instance — it is a category. "CabotTrail Outdoor" is not an entity; "Customer" is. "Cape Breton 3-Season Sleeping Bag" is not an entity; "Product" is.

Identifying entities is the first and most important step in database design. Good questions to ask:

- What are the main nouns in the business description?
- What things does the business track over time?
- What things have multiple attributes associated with them?
- What things would appear on a business document (invoice, purchase order, employee record)?

For CabotTrail, the core entities include: **Customer**, **Order**, **Product**, **Invoice**, **Supplier**, **Employee**, **City**, **Province**, **Country**.

### 3.2 Attributes

An **attribute** is a property of an entity — a piece of information that describes the entity. Attributes become columns.

The **Customer** entity has attributes: CustomerName, CreditLimit, AccountOpenedDate, DeliveryCity. Each attribute describes a fact about the customer.

Not everything that could be an attribute should be. A key design question: does this piece of information belong to this entity, or does it belong to a related entity? Customer's city is an attribute of Customer — but the city's province is not an attribute of Customer. It is an attribute of City. If you store `ProvinceCode` directly on the `Customers` table, you have duplicated information that belongs to `Cities`. When a city's province changes (unusual but possible), you must update every customer in that city rather than the one city record.

This reasoning is the foundation of normalization, covered in Chapters 4 and 5.

### 3.3 Relationships

A **relationship** describes how entities are associated with each other. Relationships become foreign keys (sometimes also tables, when the relationship has its own attributes).

Relationships have a **cardinality** — the numerical constraint on how many instances of one entity can be associated with instances of another:

| Cardinality | Symbol | Meaning | CabotTrail example |
|---|---|---|---|
| **One-to-one** | 1:1 | One A is associated with exactly one B | Employee ↔ EmployeeContactDetails |
| **One-to-many** | 1:N | One A is associated with many Bs | Customer → Orders (one customer, many orders) |
| **Many-to-many** | M:N | Many As are associated with many Bs | Products ↔ ProductCategories (one product, many categories; one category, many products) |

Cardinality is a **business rule**, not a technical constraint. "One customer can place many orders" is a statement about how CabotTrail operates — it is not a physical limitation of the database. The database *encodes* that business rule by putting `CustomerID` as a foreign key on the `Orders` table (enforcing the one-to-many). If the business decided "each order must be placed by exactly one customer," that too is a business rule encoded in the design.

---

## 4. Relations, Not Just Tables

The word "relational" in relational database does not mean "things are related to each other." It refers to the mathematical concept of a **relation** — which is what most people would call a table with some additional formal properties.

### 4.1 The Formal Definition

A **relation** is a two-dimensional structure with these properties:

1. **Each column has a unique name** — no two columns in the same table may have the same name
2. **All values in a column are of the same type** — a column cannot mix integers and text
3. **Each row is unique** — no two rows in the same table are identical in all columns
4. **The order of rows is irrelevant** — a relation has no inherent row order; ORDER BY is something you apply to a query, not a property of the data
5. **The order of columns is irrelevant** — a relation is defined by its column names, not their sequence
6. **Each cell contains exactly one value** — a column cannot store a list of values in a single cell

These properties are formal. Understanding them explains several SQL behaviours:

- **Why ORDER BY is always optional:** Because a relation has no inherent order — the default return order of rows is genuinely undefined.
- **Why SELECT * is imprecise:** Because column order is irrelevant to the relation; a query relying on column position is fragile.
- **Why a table without a primary key violates relational principles:** Because if rows are not unique, the relation property (each row is distinguishable) is violated.

### 4.2 What Makes a Table Not a Relation

A SQL table is a physical object that approximates a relation. It can violate relational properties in ways that cause problems:

**Violation: duplicate rows.** A table without a primary key can have two identical rows. Queries against such a table produce unpredictable aggregate results and JOIN behaviours.

**Violation: multi-valued attributes.** A column that stores comma-separated lists (`'NS, NB, PE'`) violates property 6 — each cell does not contain exactly one value. This makes it impossible to filter by a single value using `=`.

**Violation: inconsistent types.** A `VARCHAR` column used to store phone numbers, dates, and codes indiscriminately violates property 2.

Normalization (Chapters 4 and 5) is the process of eliminating these violations from a database design.

### 4.3 The Tuple and Domain

The formal vocabulary of the relational model uses the term **tuple** for what SQL calls a row, and **domain** for the set of valid values for a column.

A domain is not just a data type — it is a semantic constraint. The domain for `MonthNumber` is the integers 1 through 12. The domain for `ProvinceCode` is a specific set of two-letter codes. A `TINYINT` column that stores month numbers has the right data type, but only a `CHECK (MonthNumber BETWEEN 1 AND 12)` constraint enforces the domain.

This distinction — between the physical type and the logical domain — explains why data types alone are insufficient for data integrity. Constraints (Chapter 6) are what enforce domains.

---

## 5. Logical vs Physical Models

Database design happens in two stages: logical design and physical design. The distinction is fundamental.

### 5.1 The Logical Model

A **logical data model** describes *what* data the system needs to store and how it is logically organised — independent of any specific database technology. A logical model answers:

- What entities exist?
- What attributes do they have?
- How are the entities related?
- What constraints apply?

A logical model is expressed as an entity-relationship diagram (Chapter 2). It should be understandable by business stakeholders without any database knowledge. It does not mention SQL Server, indexes, or storage.

### 5.2 The Physical Model

A **physical data model** describes *how* the logical model is implemented in a specific database system. A physical model answers:

- What SQL data type is each attribute stored as?
- What is the exact DDL for each table?
- Which columns are primary keys? Which are foreign keys?
- What indexes exist for performance?
- What constraints enforce which business rules?

A physical model is expressed as DDL — `CREATE TABLE` statements with data types, constraints, and index definitions. It is technology-specific: the physical model for SQL Server differs from the physical model for PostgreSQL or Oracle in syntax, data type names, and constraint options.

### 5.3 The Flow from Logical to Physical

The design process flows:

```
Business requirements
        ↓
[Analysis: identify entities, attributes, relationships]
        ↓
Logical model (ERD)
        ↓
[Normalization: eliminate redundancy]
        ↓
Normalized logical model
        ↓
[Physical design: choose data types, constraints, indexes]
        ↓
Physical model (DDL)
        ↓
[Implementation: CREATE TABLE statements]
        ↓
Populated database
```

This course follows this flow. Chapters 2–5 cover the logical design steps. Chapters 6–7 cover physical design. Chapter 8 applies both to the OLAP context. Chapter 9 executes the complete flow for a new database.

---

## 6. Keys: Identity and Connection

In DBAS 5010 you used primary and foreign keys mechanically — join on this key, look up that surrogate key. This section gives those concepts the depth they deserve.

### 6.1 Candidate Keys

A **candidate key** is any column or combination of columns that could serve as a unique identifier for rows in the table — no two rows have the same value, and no part of the key is redundant.

A table can have multiple candidate keys. For a `Customers` table, both `CustomerID` (a system-assigned number) and `CustomerEmail` (if unique) could be candidate keys.

### 6.2 The Primary Key

The **primary key** is the candidate key that the designer chooses as the official row identifier. The choice of primary key has implications:

- SQL Server creates a clustered index on the primary key by default, ordering the physical storage of rows by PK value
- All foreign keys in other tables that reference this table reference the primary key
- The primary key constraint enforces uniqueness and NOT NULL

**Natural keys** are candidate keys derived from real-world attributes: email address, tax ID, ISBN. They have meaning outside the database. The risk: real-world attributes sometimes change. A customer changes their email address; the natural key must be updated everywhere it is referenced.

**Surrogate keys** are system-generated identifiers with no meaning outside the database: `IDENTITY(1,1)` integer columns like `CustomerID`. They never change. They have no business meaning, but they are stable, compact, and index-efficient. CabotTrail uses surrogate keys throughout both the OLTP and the DW.

**The surrogate vs natural key debate** is one of the most persistent in database design. The CabotTrail databases use surrogate keys consistently — a deliberate design decision. Chapter 3 covers when each approach is appropriate.

### 6.3 Foreign Keys

A **foreign key** is a column in one table whose value must match the primary key of another table — or be NULL. It enforces the relationship between entities.

```
Sales.Orders.CustomerID  →  Sales.Customers.CustomerID
```

This foreign key enforces: "every order must reference a customer that exists." Attempting to insert an order with a `CustomerID` that does not exist in `Customers` fails.

Foreign keys also define **referential integrity actions** — what happens when the referenced PK is updated or deleted:

| Action | Meaning |
|---|---|
| `NO ACTION` (default) | Block the update/delete if referencing rows exist |
| `CASCADE` | Propagate the update/delete to referencing rows |
| `SET NULL` | Set the FK column to NULL when the referenced row is deleted |
| `SET DEFAULT` | Set the FK column to its default value |

The choice of referential action is a business rule. "What happens to orders when a customer is deleted?" is a question only the business can answer. The designer encodes the answer as a referential action.

### 6.4 Composite Keys

A **composite key** is a primary key made of two or more columns. Their combined values are unique; neither alone is sufficient.

```sql
-- Junction table with a composite primary key
CREATE TABLE Inventory.ProductCategoryAssignments
(
    ProductID           INT NOT NULL,
    ProductCategoryID   INT NOT NULL,
    CONSTRAINT PK_ProductCategoryAssignments
        PRIMARY KEY (ProductID, ProductCategoryID)
);
```

Neither `ProductID` alone nor `ProductCategoryID` alone is unique — a product can appear in multiple categories, and a category contains multiple products. Only the combination is unique. This is the standard pattern for implementing many-to-many relationships.

---

## 7. Reading the CabotTrail OLTP Schema

You have queried this database. Now read it as a designer.

### 7.1 Discovering the Schema

```sql
-- List all tables by schema in the OLTP
USE CabotTrailOutdoor;

SELECT
    s.name              AS SchemaName,
    t.name              AS TableName,
    p.rows              AS RowCount
FROM    sys.tables t
INNER JOIN sys.schemas s    ON s.schema_id = t.schema_id
INNER JOIN sys.partitions p ON p.object_id = t.object_id
                           AND p.index_id IN (0, 1)
ORDER BY s.name, t.name;
```

Run this. You will see tables organised into four schemas: `Sales`, `Purchasing`, `Inventory`, and `Application`. This organisation is itself a design decision — schemas group related tables, reflecting business domains.

### 7.2 Reading the Relationships

```sql
-- All foreign key relationships in the OLTP
SELECT
    fk.name                     AS RelationshipName,
    SCHEMA_NAME(tp.schema_id) + '.' + tp.name  AS ParentTable,
    cp.name                     AS ParentColumn,
    SCHEMA_NAME(tc.schema_id) + '.' + tc.name  AS ReferencedTable,
    cc.name                     AS ReferencedColumn
FROM    sys.foreign_keys fk
INNER JOIN sys.foreign_key_columns fkc ON fkc.constraint_object_id = fk.object_id
INNER JOIN sys.tables tp    ON tp.object_id = fkc.parent_object_id
INNER JOIN sys.tables tc    ON tc.object_id = fkc.referenced_object_id
INNER JOIN sys.columns cp   ON cp.object_id = fkc.parent_object_id
                           AND cp.column_id  = fkc.parent_column_id
INNER JOIN sys.columns cc   ON cc.object_id = fkc.referenced_object_id
                           AND cc.column_id  = fkc.referenced_column_id
ORDER BY SCHEMA_NAME(tp.schema_id), tp.name;
```

Study the output. For each relationship, ask:
- Which table is the "one" side and which is the "many" side?
- What business rule does this relationship enforce?
- What would happen if this FK constraint did not exist?

### 7.3 Reading the Constraints

```sql
-- All CHECK constraints in the OLTP
SELECT
    SCHEMA_NAME(t.schema_id) + '.' + t.name    AS TableName,
    cc.name                                     AS ConstraintName,
    cc.definition                               AS CheckCondition
FROM    sys.check_constraints cc
INNER JOIN sys.tables t ON t.object_id = cc.parent_object_id
ORDER BY TableName, ConstraintName;
```

Each check constraint is a business rule encoded in DDL. Reading them tells you something about what the database designer knew about the business — what values are valid, what ranges are permitted.

### 7.4 The CabotTrail OLTP Entity Map

Reading the schema systematically, the OLTP has these core entity groups:

**Sales domain:**
```
Countries ──── StateProvinces ──── Cities
                                        │
                                   Customers ──── Orders ──── OrderLines
                                        │              │
                                   Invoices ──────────┘
                                        │
                                   InvoiceLines ──── Returns
```

**Purchasing domain:**
```
Suppliers ──── PurchaseOrders ──── PurchaseOrderLines
```

**Inventory domain:**
```
ProductCategories ─── ProductCategoryAssignments ─── Products ─── ProductHoldings
```

**Application domain (shared lookups):**
```
Employees ── People ── DeliveryMethods ── PackageTypes ── Colors ── PaymentMethods
```

Notice that `People` is a shared entity — both `Customers` and `Employees` reference it (or share its attributes). This is a **supertype/subtype** relationship — a design pattern covered in Chapter 2.

---

## 8. Design Decisions in the Wild

Every structural choice in the CabotTrail OLTP reflects a design decision. Identifying these decisions and understanding their implications is the core skill this course develops.

### 8.1 The Geography Hierarchy

Customer delivery addresses are stored across four tables:

```
Countries → StateProvinces → Cities → Customers.DeliveryCityID
```

A simpler design would store `City`, `Province`, and `Country` directly on the `Customers` table. Why was this not done?

**Reason:** Multiple customers share the same city. If `CityName` and `ProvinceCode` were columns on `Customers`, every customer in Halifax would store 'Halifax' and 'NS' redundantly. If Nova Scotia's province code ever changed, every customer record would need updating. Storing the geography in separate tables and linking by ID means:
- City data is stored once
- Changes propagate automatically
- Queries can aggregate by city or province without scanning customer records

This is the normalization principle in action — covered formally in Chapters 4 and 5.

### 8.2 The Junction Table for Categories

Products can belong to multiple categories. Categories contain multiple products. This many-to-many relationship is resolved with a junction table:

```
Products ──── ProductCategoryAssignments ──── ProductCategories
```

Why not store `CategoryID` directly on the `Products` table? Because a product can be in more than one category. A single column can hold only one value. To allow multiple categories per product requires a separate table — one row per (product, category) pair.

### 8.3 The Separation of Orders and Invoices

Orders and Invoices are separate tables even though they often have a one-to-one relationship. Why?

**Reason:** The business model allows for partial fulfilment — an order might be partially shipped and partially backordered, potentially resulting in multiple invoices per order. Even if this does not happen frequently, the design accommodates it without structural change. This is the principle of designing for the business's possible future, not just its current practice.

This is also why there are `OrderLines` *and* `InvoiceLines` — the quantity picked and invoiced may differ from the quantity ordered.

---

## 9. The Cost of Bad Design

Understanding why good design matters is motivating. Here are the specific costs that bad design inflicts.

### 9.1 Update Anomalies

When the same fact is stored in multiple places, updating it requires updating all copies — and missing one creates an inconsistency.

```
-- Bad design: storing city name directly on customer records
Customers table:
CustomerID | CustomerName          | CityName  | Province
42         | Fundy Bay Outfitters  | Halifax   | NS
17         | Atlantic Gear Ltd     | Halifax   | NS
91         | East Coast Supply     | Halifax   | NS

-- If Halifax is renamed to 'New Halifax' (hypothetical):
-- Must update 3 rows minimum, more if other customers are in Halifax
-- Missing any one row creates data inconsistency
```

With a proper geography hierarchy, the city name is stored once in `Cities`. Renaming it updates one row everywhere.

### 9.2 Insertion Anomalies

A bad design may make it impossible to record certain facts without simultaneously recording other facts you do not yet have.

```
-- Bad design: one table for customer + order information
-- Cannot record a new customer until they place their first order
-- (What order details do you fill in for a customer with no orders yet?)
CustomerOrderTable:
CustomerID | CustomerName | OrderID | OrderDate | ProductID | ...
```

With proper entity separation, you can create a Customer record before any orders exist.

### 9.3 Deletion Anomalies

A bad design may cause the loss of one piece of information when another is deleted.

```
-- Same bad design: if the last order for a customer is deleted,
-- all information about that customer disappears too
```

### 9.4 These Anomalies Have Names for a Reason

**Update, insertion, and deletion anomalies** are the formal names for the symptoms of under-normalization. Normalization (Chapters 4 and 5) is the process of eliminating these anomalies by structuring data correctly. Every normalization rule corresponds to eliminating one of these anomaly types.

---

## 10. Chapter Summary

- Database design is a **translation discipline** — converting business requirements into formal relational structures. It requires collaboration between technical designers and business stakeholders.

- The **three-schema architecture** separates the database into external (user views), conceptual (logical structure), and internal (physical storage) levels — enabling changes at one level without breaking others.

- The building blocks of data models are **entities** (things the business tracks), **attributes** (properties of entities), and **relationships** (associations between entities with cardinality).

- A **relation** is a formally defined structure with properties beyond a simple table: unique column names, consistent column types, unique rows, and single-valued cells. Violations of these properties cause anomalies.

- **Logical models** describe *what* data exists; **physical models** describe *how* it is stored. The flow is: requirements → ERD (logical) → normalization → DDL (physical).

- **Primary keys** identify rows; **foreign keys** enforce relationships. **Natural keys** have business meaning; **surrogate keys** are system-generated. **Composite keys** implement many-to-many relationships.

- Reading the CabotTrail OLTP schema reveals specific design decisions: the geography hierarchy (normalization), the junction table for categories (M:N resolution), and the separation of orders and invoices (future-proofing).

- **Update, insertion, and deletion anomalies** are the costs of bad design. Normalization eliminates them.

---

## 11. Review Questions

1. Explain the difference between an entity and an attribute. For each of the following, state whether it is an entity or an attribute, and explain your reasoning: **ProductName**, **Supplier**, **OrderDate**, **PaymentMethod**, **Province**, **TaxRate**.

2. The three-schema architecture defines external, conceptual, and internal schemas. Using the CabotTrail environment as your example, identify a specific object that belongs to each level and explain why it belongs there.

3. A colleague proposes storing customer orders in a spreadsheet where each row represents one customer, and multiple orders are stored as separate columns: `Order1Date`, `Order2Date`, `Order3Date`. Identify which relational property this design violates, what happens when a customer places a fourth order, and how you would redesign this correctly.

4. Explain the difference between a natural key and a surrogate key. Give a specific example of a column from `CabotTrailOutdoor` that could serve as a natural key for the `Customers` entity. What is the risk of using it as the primary key instead of the system-assigned `CustomerID`?

5. The CabotTrail OLTP stores geography across four tables: `Countries`, `StateProvinces`, `Cities`, and `Customers.DeliveryCityID`. Write out the specific update anomaly that would occur if city names were stored directly on the `Customers` table instead, using Halifax as a specific example.

6. Run the foreign key query from section 7.2 against `CabotTrailOutdoor`. Pick three foreign key relationships from the results and for each one: (a) state the business rule it enforces and (b) explain what would happen operationally if the constraint did not exist.

7. Describe the difference between a logical data model and a physical data model. For the CabotTrail `Customers` entity, give one example of a design decision that belongs in the logical model and one that belongs only in the physical model.

8. `Sales.InvoiceLines` has a composite primary key. Without looking it up, reason from the business context: which two columns form that composite key, and why is neither column alone sufficient to uniquely identify a row?

---

## 🔍 Deeper Dive

### Going Further with the Relational Model

#### E.F. Codd's Twelve Rules

In 1985, E.F. Codd (who had proposed the relational model in 1970) published what became known as **Codd's Twelve Rules** — a set of criteria that a database management system must satisfy to be considered truly relational. They are:

0. **The information rule:** All information must be represented as data values in tables.
1. **The guaranteed access rule:** Every datum must be accessible via table name, primary key, and column name.
2. **Systematic treatment of NULL:** NULL must represent missing or inapplicable information, distinct from any other value.
3. **Active online catalogue:** The database schema must be stored in tables that can be queried using the same query language as user data.
4. **Comprehensive data sublanguage:** At least one language must support data definition, data manipulation, transaction management, and integrity constraints.
5. **View updating rule:** All views that are theoretically updatable must be updatable by the system.
6. **High-level insert, update, delete:** These operations must be applicable to sets of rows, not just individual rows.
7. **Physical data independence:** Changes to physical storage must not require changes to applications.
8. **Logical data independence:** Changes to the logical schema must not require changes to applications (beyond the changed objects).
9. **Integrity independence:** Integrity constraints must be definable in the data sublanguage and stored in the catalogue, not in application code.
10. **Distribution independence:** A distributed database must behave identically to a centralised one from the application's perspective.
11. **Non-subversion rule:** If a low-level access method exists, it must not be able to bypass the integrity constraints.

No commercial DBMS fully satisfies all twelve rules. SQL Server satisfies most of them. Understanding the rules helps explain why some SQL behaviours seem odd — they are the result of partial compliance with a strict theoretical framework.

Codd, E. F. (1985, October 14). Is your DBMS really relational? *Computerworld*.

#### The Relational Model vs the Object-Relational Model

The relational model treats data as two-dimensional tables of atomic values. Modern databases extend this with the **object-relational model** — allowing complex types, arrays, JSON, and hierarchical structures within columns.

SQL Server's support for XML columns, JSON stored in NVARCHAR, spatial data types, and user-defined CLR types are all object-relational extensions. They are useful when the strict relational model is impractical — storing a variable-length list of tags on a product, for example.

The trade-off: object-relational extensions are powerful but sacrifice some of the formal guarantees that make the relational model trustworthy. A JSON column containing a list is not queryable with standard SQL operators. Values inside it cannot have FK constraints enforced against them.

The principle: use the relational model by default. Reach for extensions only when the relational approach is genuinely impractical for a specific use case.

#### The Entity-Relationship Model vs the Relational Model

A subtle distinction that confuses many students: the **Entity-Relationship (ER) model** and the **Relational model** are two different things, though they work together.

The ER model, proposed by Peter Chen in 1976, is a conceptual modelling framework — a way of describing the real world in terms of entities and relationships. It produces entity-relationship diagrams.

The relational model, proposed by Codd in 1970, is a mathematical framework for data storage and query. It produces tables and SQL.

ER modelling is a technique for *designing* a relational database. The ER diagram is the input; the relational schema (tables, keys, constraints) is the output. Chapter 2 covers ER modelling. Chapter 3 covers the mapping from ER model to relational tables.

Chen, P. P. (1976). The entity-relationship model — Toward a unified view of data. *ACM Transactions on Database Systems*, 1(1), 9–36.

---

### References and Further Reading

1. Date, C. J. (2019). *Database Design and Relational Theory: Normal Forms and All That Jazz* (2nd ed.). Apress. — The most rigorous treatment of the relational model's theoretical foundations.

2. Codd, E. F. (1970). A relational model of data for large shared data banks. *Communications of the ACM*, 13(6), 377–387. — The foundational paper.

3. Chen, P. P. (1976). The entity-relationship model — Toward a unified view of data. *ACM Transactions on Database Systems*, 1(1), 9–36. — The paper that introduced ERDs.

4. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Comprehensive textbook covering all topics in this course in depth.

5. Elmasri, R., & Navathe, S. (2016). *Fundamentals of Database Systems* (7th ed.). Pearson. — Academic standard for database design courses.

6. Microsoft. (2024). *SQL Server data types*. [https://learn.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql)

7. Microsoft. (2024). *sys.foreign_keys (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-foreign-keys-transact-sql](https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-foreign-keys-transact-sql)

---

*Next chapter: [Chapter 2 — Entity-Relationship Diagrams: Drawing the Business](../chapter-02-entity-relationship-diagrams/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
