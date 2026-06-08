# Chapter 2: Entity-Relationship Diagrams — Drawing the Business

> **Database Design for Analysts**
> *From business requirements to relational structures*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Chapter 1 introduced the vocabulary of database design — entities, attributes, relationships, cardinality. This chapter puts that vocabulary to work. An **Entity-Relationship Diagram (ERD)** is the primary tool database designers use to capture and communicate the logical structure of a database before a single table is created.

An ERD is not a table diagram. It is not a list of SQL statements. It is a visual representation of the business's information model — a picture that shows what exists, what properties those things have, and how they are connected. A well-drawn ERD is readable by both a database developer and a business manager who has never written SQL.

This chapter covers the notation, the technique, and the thinking behind ERDs. The notation is straightforward to learn. The thinking — identifying the right entities, drawing the right relationships, getting cardinality correct — takes practice and is the core intellectual work of database design.

By the end of this chapter you will be able to:

- Identify entities, attributes, and relationships from a business narrative
- Distinguish between strong entities and weak entities
- Draw an ERD using Crow's Foot notation
- Correctly identify and represent cardinality for one-to-one, one-to-many, and many-to-many relationships
- Represent mandatory and optional participation
- Model derived attributes, multi-valued attributes, and composite attributes
- Apply supertypes and subtypes to model entity hierarchies
- Read and critique an ERD from an existing system

---

## Table of Contents

1. [What an ERD Is and Is Not](#1-what-an-erd-is-and-is-not)
2. [ERD Notation: The Crow's Foot Standard](#2-erd-notation-the-crows-foot-standard)
3. [Drawing Entities and Attributes](#3-drawing-entities-and-attributes)
4. [Special Attribute Types](#4-special-attribute-types)
5. [Relationships and Cardinality](#5-relationships-and-cardinality)
6. [Participation: Mandatory and Optional](#6-participation-mandatory-and-optional)
7. [Strong and Weak Entities](#7-strong-and-weak-entities)
8. [Many-to-Many Relationships](#8-many-to-many-relationships)
9. [Supertypes and Subtypes](#9-supertypes-and-subtypes)
10. [From Narrative to ERD: A Worked Example](#10-from-narrative-to-erd-a-worked-example)
11. [Reading the CabotTrail ERD](#11-reading-the-cabottrail-erd)
12. [Chapter Summary](#12-chapter-summary)
13. [Review Questions](#13-review-questions)
14. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. What an ERD Is and Is Not

### 1.1 What an ERD Is

An ERD is a **conceptual model** — a formal representation of the logical structure of a business's information. It answers three questions:

1. **What exists?** The entities — the things the business tracks
2. **What do those things look like?** The attributes — the properties of each entity
3. **How are those things connected?** The relationships — the associations between entities

An ERD is drawn *before* any SQL is written. It is the specification that SQL implementations are derived from. If you change the ERD, the SQL changes. If you write SQL without an ERD, you have skipped the design step — and you will spend time discovering structural problems during implementation rather than on paper, where changes are free.

### 1.2 What an ERD Is Not

An ERD is **not** a table diagram. Tables have SQL data types, IDENTITY columns, DEFAULT constraints, and indexes. An ERD has none of these — they belong to the physical model, not the conceptual model.

An ERD is **not** a schema diagram (like the ones SSMS generates). Schema diagrams show physical tables with their actual columns. ERDs show logical entities with their semantic attributes.

An ERD is **not** a finished product. It is a working document — drawn in pencil, revised through conversation with stakeholders, and updated as understanding deepens. An ERD that has never been revised is either a very simple system or a design that was never questioned.

### 1.3 The ERD as a Communication Tool

The most important property of an ERD is that it is understandable to non-technical stakeholders. A business manager who has never seen SQL can look at a well-drawn ERD and say "yes, that's how our orders work" or "wait — that's not right, an order can have multiple delivery addresses."

This conversation — between the designer and the business — is what the ERD enables. The diagram externalises the designer's understanding of the business, making it verifiable by the people who know the business best.

---

## 2. ERD Notation: The Crow's Foot Standard

Several ERD notations exist. This book uses **Crow's Foot notation**, which is the most widely used in industry, supported by Microsoft Visio, Lucidchart, draw.io, and most database design tools.

### 2.1 The Basic Symbols

**Entity:** A rectangle containing the entity name. Names are singular nouns in uppercase or title case — `CUSTOMER`, `ORDER`, or `Customer`, `Order`.

**Attribute:** An oval connected to its entity with a line, or (in simplified notation) listed inside the entity rectangle. The primary key attribute is underlined.

**Relationship:** A line connecting two entities, with Crow's Foot symbols at each end indicating cardinality.

### 2.2 Crow's Foot Cardinality Symbols

The "crow's foot" is the symbol that looks like a three-pronged fork at the end of a relationship line. The symbols at each end of the line represent the minimum and maximum number of instances that can participate:

```
Minimum symbols (closest to entity):
  ─|─   Exactly one (mandatory)
  ─O─   Zero (optional)

Maximum symbols (farther from entity):
  ─|─   One
  ─<─   Many (the "crow's foot")
```

These are combined to express four cardinality patterns:

```
  ──|──|──   Exactly one (mandatory one)
  ──O──|──   Zero or one (optional one)
  ──|──<──   One or many (mandatory many — at least one)
  ──O──<──   Zero or many (optional many — none, one, or more)
```

### 2.3 Reading a Relationship Line

A relationship line is read in both directions, with the entity at each end as the subject:

```
CUSTOMER ──|──<── ORDER
```

Reading left to right: "One Customer can have many Orders"
Reading right to left: "One Order must belong to exactly one Customer"

The symbols at the Customer end (`|`) mean: from Order's perspective, exactly one Customer.
The symbols at the Order end (`|<`) mean: from Customer's perspective, one or many Orders.

Always read both directions. Both statements express the complete cardinality rule.

### 2.4 Notation in Practice

In simplified ERD notation (used in most tools), attributes are listed inside the entity rectangle rather than as separate ovals. Primary key attributes are marked `PK`, foreign key attributes are marked `FK`:

```
┌─────────────────────┐         ┌─────────────────────┐
│ CUSTOMER            │         │ ORDER               │
├─────────────────────┤         ├─────────────────────┤
│ PK CustomerID (INT) │──|──O<──│ PK OrderID (INT)    │
│    CustomerName     │         │ FK CustomerID (INT)  │
│    CreditLimit      │         │    OrderDate         │
│    AccountOpenedDate│         │    DeliveryAddress   │
└─────────────────────┘         └─────────────────────┘
```

The relationship line at the Order end shows `O<` (zero or many) — a customer may exist with no orders yet. The line at the Customer end shows `|` (exactly one) — an order must always reference a customer.

---

## 3. Drawing Entities and Attributes

### 3.1 Identifying Entities

From a business narrative, entities are typically:

- **Nouns** that the business tracks over time: Customer, Product, Order, Supplier
- **Things that appear on business documents**: an invoice references customers, products, quantities, prices
- **Things with multiple instances**: there are many customers, many products, many orders
- **Things the business makes decisions about**: which supplier to use for which product

**Not entities:**
- Single-value facts (the company's tax number, the current tax rate) — these are better stored as configuration, not as entities
- Actions or processes (placing an order) — the *result* of the action is the entity (Order), not the action itself
- Relationships between entities — the relationship is captured as a line, not a box

**The entity identification test:** Can you have multiple distinct instances of this thing? Can two instances have the same attribute values? If yes to both, it is an entity.

### 3.2 Naming Entities

Entity names should be:
- **Singular** not plural: `Customer` not `Customers`, `Order` not `Orders`
- **Specific** to the domain: `PurchaseOrder` not `Order` when the system has both sales orders and purchase orders
- **Business-language** not technical: `Customer` not `UserAccount`, `Product` not `Item`

### 3.3 Identifying Attributes

For each entity, attributes are the facts the business needs to store about instances of that entity. Good questions to surface attributes:

- What information does the business record when a new [entity] is created?
- What does the business need to know about [entity] to do its work?
- What columns would appear on a report about [entity]?

For `Customer`, the CabotTrail attributes are: CustomerID, CustomerName, CustomerGroupName (category), CreditLimit, AccountOpenedDate.

For `Order`, the attributes are: OrderID, OrderDate, ExpectedDeliveryDate, IsUndersupplyBackordered — and a reference to CustomerID (the FK that links to Customer).

### 3.4 Choosing the Primary Key Attribute

Every entity must have a primary key — the attribute(s) that uniquely identify each instance. Mark the PK attribute with an underline (in traditional notation) or `PK` label (in simplified notation).

If no natural attribute is a reliable unique identifier, introduce a surrogate key (`CustomerID INT IDENTITY`). The discussion from Chapter 1 applies: surrogate keys are stable and index-efficient; natural keys carry business meaning but may change.

---

## 4. Special Attribute Types

Some attributes have properties that cannot be represented as simple single-valued columns. Recognising these in the design phase prevents structural problems later.

### 4.1 Multi-Valued Attributes

A **multi-valued attribute** is an attribute that can hold more than one value per entity instance. In the relational model, multi-valued attributes cannot be stored as a single column — they require a separate table.

**Example:** A `Product` can belong to multiple `ProductCategories`. `CategoryName` is a multi-valued attribute of `Product`.

**Solution:** Create a separate `ProductCategoryAssignment` entity (a junction table) with a row for each (Product, Category) pair. This is the standard resolution for multi-valued attributes and for many-to-many relationships (section 8).

In ERD notation, multi-valued attributes are shown with a **double oval**. They are always resolved to a separate table in the physical model.

### 4.2 Derived Attributes

A **derived attribute** is one that can be computed from other stored attributes. It does not need to be stored, because it can always be recalculated.

**Example:** `CustomerAge` can be derived from `DateOfBirth` and `GETDATE()`. `OrderTotal` can be derived from the sum of `InvoiceLines.LineTotal`.

In ERD notation, derived attributes are shown with a **dashed oval**. In the physical model, you have a design decision:
- **Compute at query time** — cheaper to store, always accurate, requires computation each query
- **Store as a computed column** (`AS expression PERSISTED`) — stored on disk, automatically maintained by SQL Server
- **Store as a regular column populated by ETL** — fast to query, but requires synchronisation logic

The CabotTrail DW stores `DaysUntilInvoice` and `GrossProfitMarginPct` as populated columns — the ETL computes and stores them. This is documented in the S2T mapping as derived attributes.

### 4.3 Composite Attributes

A **composite attribute** is one that is made up of multiple components that each have meaning independently.

**Example:** `FullName` is a composite attribute — it consists of `FirstName` and `LastName`. If the business only ever needs the full name as a unit, `FullName` is appropriate. If the business needs to sort by last name, search by first name, or generate personalised greetings, `FirstName` and `LastName` must be stored separately.

In ERD notation, composite attributes are shown with sub-ovals branching from the main oval. In the physical model, they become multiple columns.

The CabotTrail OLTP stores `Application.People.FullName` and `Application.People.PreferredName` as flat attributes — a decision that implies the business does not need to sort or filter by first name and last name independently. If that assumption is wrong, the schema would need to change.

---

## 5. Relationships and Cardinality

### 5.1 Naming Relationships

Relationships should be named with a verb phrase that describes the association. Read the name in both directions:

```
CUSTOMER ──places──> ORDER
ORDER ──is placed by──> CUSTOMER
```

Good relationship names make the ERD readable as prose: "A Customer *places* zero or many Orders. An Order *is placed by* exactly one Customer."

### 5.2 One-to-One (1:1)

A 1:1 relationship means each instance of entity A is associated with at most one instance of entity B, and vice versa.

```
EMPLOYEE ──|──|── EMPLOYEE_CONTRACT
```

One employee has exactly one contract; one contract belongs to exactly one employee.

1:1 relationships are rare because if two entities are always in a 1:1 relationship, they are usually the same entity. When 1:1 relationships do appear, they often reflect:
- **Security partitioning**: sensitive attributes are split to a separate table with tighter access control
- **Optional extension**: attributes that only apply to some instances are moved to a separate table rather than using NULLs
- **Legacy integration**: two systems that should share an entity do not, and a join table bridges them

**CabotTrail example:** `Application.People` and `Application.Employees` have a near-1:1 relationship — each employee is also a person. The separation allows the `People` table to be shared across multiple entities (employees, contacts, perhaps customers in future).

### 5.3 One-to-Many (1:N)

The most common relationship type. One instance of A is associated with many instances of B; each instance of B is associated with exactly one instance of A.

```
CUSTOMER ──|──O<── ORDER
```

One Customer has zero or many Orders. Each Order belongs to exactly one Customer.

The FK goes on the "many" side — `Orders.CustomerID` references `Customers.CustomerID`. This is the invariant rule: **the foreign key always lives on the many side of a one-to-many relationship.**

### 5.4 Many-to-Many (M:N)

Many instances of A are associated with many instances of B.

```
PRODUCT ──O<──>O── PRODUCT_CATEGORY
```

One Product belongs to zero or many Categories. One Category contains zero or many Products.

Many-to-many relationships **cannot be directly implemented** in the relational model with just two tables and a foreign key. They require a **junction table** (also called an associative entity, bridge table, or link table). Section 8 covers this in depth.

### 5.5 Self-Referencing Relationships

An entity can have a relationship with itself — a **recursive** or **unary** relationship. The classic example is a hierarchy: an Employee has a Manager who is also an Employee.

```
EMPLOYEE ──|──O<── EMPLOYEE (manages/is managed by)
```

In the physical model, this becomes a self-referencing foreign key: `Employees.ManagerID` references `Employees.EmployeeID`.

The CabotTrail `Application.Employees` table includes a `ReportsTo` or similar column referencing the same table. Recursive relationships underpin org charts, bill-of-materials structures, and category trees.

---

## 6. Participation: Mandatory and Optional

**Participation** defines whether every instance of an entity *must* participate in a relationship, or whether participation is optional.

### 6.1 Mandatory Participation

**Mandatory participation** (also called **total participation**): every instance of the entity must be involved in at least one instance of the relationship.

In Crow's Foot notation: the `|` symbol (vertical bar) on the relationship line nearest the entity indicates mandatory participation (minimum cardinality = 1).

**Example:** Every Order must have a Customer. An order cannot exist without a customer. This is mandatory participation — the minimum on the Customer side of the Order–Customer relationship is 1 (not 0).

In the physical model, mandatory participation translates to a `NOT NULL` foreign key: `Orders.CustomerID NOT NULL`. If the FK can be NULL, the participation is optional.

### 6.2 Optional Participation

**Optional participation** (also called **partial participation**): some instances of the entity may exist without participating in any instance of the relationship.

In Crow's Foot notation: the `O` symbol (circle/zero) on the relationship line nearest the entity indicates optional participation (minimum cardinality = 0).

**Example:** A Customer may or may not have placed any Orders. Participation of Customer in the Customer–Order relationship is optional from the Customer's perspective. A customer can exist in the database before placing their first order.

In the physical model, optional participation can mean either a nullable FK (on the "many" side) or simply that the referenced entity row may have no referencing rows.

### 6.3 Reading Participation in Both Directions

Every relationship has participation from both entities' perspectives:

```
CUSTOMER ──|──O<── ORDER
```

From Order's perspective: Every Order *must* have exactly one Customer (`|` at Customer end — mandatory).
From Customer's perspective: A Customer *may* have zero or many Orders (`O<` at Order end — optional many).

Both participation decisions are business rules:
- "An order cannot exist without a customer" → mandatory on Order's side
- "A customer can exist before placing any orders" → optional on Customer's side

These two rules together define the complete semantics of the relationship.

---

## 7. Strong and Weak Entities

### 7.1 Strong Entities

A **strong entity** has its own primary key — it can be uniquely identified independently of any other entity. Most entities in a well-designed system are strong entities.

`Customer` is a strong entity — `CustomerID` uniquely identifies it without reference to anything else.

### 7.2 Weak Entities

A **weak entity** cannot be uniquely identified by its own attributes alone. It depends on a related **owner entity** for part of its identity.

**Example:** `OrderLine`. An order line is identified by its `OrderLineID` — but that ID only makes sense within the context of a specific `Order`. If two different orders both have an `OrderLineID = 1`, they are different line items. The uniqueness of `OrderLine` depends on knowing which Order it belongs to.

In the physical model, weak entities have a composite primary key that includes the FK to the owner entity:

```sql
-- OrderLine's identity depends on its Order
CONSTRAINT PK_OrderLines PRIMARY KEY (OrderID, OrderLineID)
-- or with a surrogate key:
-- The OrderLineID alone is unique system-wide (if using IDENTITY)
-- but conceptually, its meaning depends on the order context
```

In ERD notation, weak entities are shown in a **double rectangle** and the identifying relationship (the relationship to the owner entity) is shown as a **double diamond**.

The CabotTrail `Sales.OrderLines` is a weak entity — its rows are identified in the context of a specific order. `Sales.InvoiceLines` is similarly weak — each line is identified within the context of an invoice.

### 7.3 When Surrogate Keys Change the Picture

Surrogate keys complicate the strong/weak distinction. If every table has a system-generated IDENTITY column as its PK (as CabotTrail does), every entity *looks* like a strong entity from a physical standpoint — `OrderLineID IDENTITY(1,1)` makes each order line uniquely identifiable on its own.

The *conceptual* weakness remains: an order line without an order has no business meaning. The surrogate key makes it physically independent, but the business relationship still requires the owner entity. This is why the conceptual model (ERD) and the physical model (SQL) need to be understood separately.

---

## 8. Many-to-Many Relationships

Many-to-many relationships require special handling in both ERDs and physical models.

### 8.1 Why M:N Relationships Need Resolution

Consider a direct M:N relationship:

```
PRODUCT ──O<──>O── PRODUCT_CATEGORY
```

In the physical model, you cannot implement this with just two tables. There is no single column on either table that holds all the relationships. You cannot put `CategoryID` on `Products` (a product has many categories — you would need multiple `CategoryID` columns, with an unknown maximum). You cannot put `ProductID` on `Categories` (same problem).

### 8.2 The Junction Table (Associative Entity)

The solution is to introduce a **junction table** — an entity that represents the relationship itself. It has a row for each (Product, Category) pair:

```
PRODUCT ──|──O<── PRODUCT_CATEGORY_ASSIGNMENT ──>O──|── PRODUCT_CATEGORY
```

The junction table `PRODUCT_CATEGORY_ASSIGNMENT` has:
- `ProductID` FK → references `Products.ProductID`
- `ProductCategoryID` FK → references `ProductCategories.ProductCategoryID`
- A composite PK: `(ProductID, ProductCategoryID)`

This resolves the M:N relationship into two 1:N relationships — one from Product to the junction, one from Category to the junction.

### 8.3 Junction Tables with Their Own Attributes

Sometimes the relationship itself has attributes. A junction table can carry those attributes.

**Example:** If CabotTrail tracked the date a product was first assigned to each category, the `ProductCategoryAssignment` would have a `DateAssigned` column — an attribute of the relationship, not of either entity.

```sql
CREATE TABLE Inventory.ProductCategoryAssignments
(
    ProductID           INT     NOT NULL,
    ProductCategoryID   INT     NOT NULL,
    DateAssigned        DATE    NULL,       -- Attribute of the relationship

    CONSTRAINT PK_ProductCategoryAssignments
        PRIMARY KEY (ProductID, ProductCategoryID),
    CONSTRAINT FK_PCA_Product
        FOREIGN KEY (ProductID) REFERENCES Inventory.Products (ProductID),
    CONSTRAINT FK_PCA_Category
        FOREIGN KEY (ProductCategoryID) REFERENCES Inventory.ProductCategories (ProductCategoryID)
);
```

When a junction table has its own attributes, it has elevated to a full **associative entity** — a concept that bridges between entity and relationship.

### 8.4 Identifying M:N Relationships

The signal for a potential M:N relationship: if you find yourself saying "one [A] can be associated with many [B]s, AND one [B] can be associated with many [A]s" — that is an M:N relationship requiring a junction table.

Common M:N relationships in business systems:
- Products ↔ Categories
- Employees ↔ Projects
- Students ↔ Courses
- Orders ↔ Promotions (an order can have multiple promotions; a promotion applies to many orders)
- Ingredients ↔ Recipes

---

## 9. Supertypes and Subtypes

### 9.1 The Problem They Solve

Sometimes multiple entities share a set of common attributes but also have their own distinct attributes. The brute-force solution — duplicate the common attributes in both tables — violates normalization. The elegant solution is a **supertype/subtype relationship**.

**Example:** CabotTrail has `Employees` and (potentially) `Customers` — both are `People` with a name, an address, a contact method. But employees have `JobTitle`, `HireDate`, `Department`; customers have `CreditLimit`, `AccountOpenedDate`, `CustomerGroupName`.

### 9.2 The Supertype

The **supertype** is the parent entity that holds attributes common to all subtypes. In CabotTrail, `People` is the supertype:

```
PERSON
├── PersonID (PK)
├── FullName
├── PreferredName
├── EmailAddress
└── PhoneNumber
```

### 9.3 The Subtypes

**Subtypes** are child entities that hold attributes specific to each subtype, plus a FK to the supertype:

```
EMPLOYEE                          CUSTOMER
├── EmployeeID (PK, FK→Person)    ├── CustomerID (PK, FK→Person — if applicable)
├── JobTitle                      ├── CreditLimit
├── HireDate                      ├── AccountOpenedDate
├── Department                    └── CustomerGroupName
└── IsActive
```

The supertype/subtype relationship is drawn with a triangle or an arc connecting the subtype entities to the supertype.

### 9.4 Disjoint vs Overlapping Subtypes

**Disjoint subtypes**: a person can be either an employee or a customer, but not both. (Exclusive membership.)

**Overlapping subtypes**: a person can be both an employee *and* a customer simultaneously. (Shared membership.)

The CabotTrail case is likely overlapping — an employee could also be a CabotTrail customer.

### 9.5 Physical Implementation Options

Option 1 — **Table per subtype** (CabotTrail's approach): `People` table + `Employees` table + `Customers` table. Each subtype table has a FK to `People`. Queries joining all attributes require a join between the supertype and subtype tables.

Option 2 — **Single table with NULLs**: All columns for all subtypes in one table. Non-applicable columns are NULL. Simple queries; wastes storage; difficult to enforce subtype-specific constraints.

Option 3 — **Table per subtype, no supertype table**: Each subtype table contains all common and specific attributes. Duplication of common attributes; simpler queries within a subtype.

Each option has trade-offs. Option 1 is the most normalized and most maintainable.

---

## 10. From Narrative to ERD: A Worked Example

This section demonstrates the complete process of building an ERD from a business narrative — the same process you will practice with the CabotTrail Loyalty Program in the labs.

### 10.1 The Narrative

> CabotTrail Outdoor is launching a customer loyalty programme. Customers earn points on each purchase based on the amount spent. Points can be redeemed for rewards (discount vouchers, free products, gift items). Each customer has a loyalty account, and the account tracks their current point balance and their tier status (Bronze, Silver, Gold, Platinum). Every point transaction — whether earned or redeemed — must be recorded with a date and the number of points.

### 10.2 Step 1: Identify Entities

Read the narrative and find the nouns:
- **Customer** — already exists; we extend it
- **LoyaltyAccount** — one per customer; tracks balance and tier
- **LoyaltyTransaction** — each point earn or redemption
- **Reward** — the things points can be redeemed for
- **Tier** — Bronze, Silver, Gold, Platinum (could be a lookup table)

### 10.3 Step 2: Identify Attributes

For each entity, list the attributes mentioned or implied:

```
LOYALTY_ACCOUNT
├── AccountID (PK)
├── CustomerID (FK → Customer)
├── PointBalance
├── TierID (FK → Tier)
└── AccountOpenedDate

LOYALTY_TRANSACTION
├── TransactionID (PK)
├── AccountID (FK → LoyaltyAccount)
├── TransactionDate
├── PointsAmount (positive = earned, negative = redeemed)
├── TransactionType ('EARN' or 'REDEEM')
└── SalesLineID (FK → fact.Sales — nullable, for earn transactions)

REWARD
├── RewardID (PK)
├── RewardName
├── RewardDescription
├── PointCost
└── IsAvailable

TIER
├── TierID (PK)
├── TierName
├── MinimumPoints
└── BenefitDescription
```

### 10.4 Step 3: Identify Relationships

- `Customer` *has* one `LoyaltyAccount` (1:1 — each customer has exactly one account)
- `LoyaltyAccount` *has* many `LoyaltyTransactions` (1:N)
- `LoyaltyTransaction` *may redeem* one `Reward` (optional — earn transactions have no reward) (1:N from Reward)
- `LoyaltyAccount` *is in* one `Tier` (N:1 from LoyaltyAccount to Tier)

### 10.5 Step 4: Check Cardinality

For each relationship, ask in both directions:

**Customer ↔ LoyaltyAccount:**
- "Can a customer have more than one loyalty account?" — per the narrative, no: one account per customer.
- "Can a loyalty account exist without a customer?" — no: the account belongs to a customer.
- Result: 1:1, mandatory both sides.

**LoyaltyAccount ↔ LoyaltyTransaction:**
- "Can an account have zero transactions?" — yes, a new account has no transactions yet.
- "Can a transaction exist without an account?" — no.
- Result: 1:N from Account, optional on Account side, mandatory on Transaction side.

### 10.6 The Questions That Surface

A real design session would generate questions the narrative did not answer:

- Can points be transferred between accounts?
- What happens to points if a customer closes their account?
- Can a transaction earn points on a return/refund (negative purchase)?
- Is there an expiry date on points?
- Can a single sales transaction earn points for more than one customer (e.g., a shared account)?

Each of these questions may change the ERD. Surfacing them before implementation is the value of the design process.

---

## 11. Reading the CabotTrail ERD

### 11.1 Sales Domain ERD

The core Sales domain can be read from the OLTP schema:

```
COUNTRY ──|──O<── STATE_PROVINCE ──|──O<── CITY ──|──O<── CUSTOMER
                                                          │
                                                  ──|──O<──
                                                          │
                                                        ORDER ──|──O<── ORDER_LINE
                                                          │                   │
                                                          └──────────────┐    │
                                                                    INVOICE    │
                                                                         │     │
                                                                INVOICE_LINE──┘
                                                                         │
                                                                      RETURN
```

Reading this ERD:
- A Country has zero or many State/Provinces (countries with no provinces tracked = optional)
- A Province belongs to exactly one Country (mandatory)
- A City belongs to exactly one Province (mandatory)
- A Customer has a delivery city — one city has zero or many customers
- A Customer places zero or many Orders
- An Order has one or many Order Lines
- An Order generates zero or one Invoice (an order may not be invoiced immediately)
- An Invoice covers exactly one Order
- An Invoice Line references exactly one Order Line
- An Order Line may have zero or one Return

### 11.2 What the ERD Reveals

The ERD makes visible things that are easy to miss in SQL:

- **The separation of Orders and Invoices** is explicit in the diagram — it shows these are distinct entities with a 1:1 (or 1:0) relationship
- **The geography hierarchy** shows three levels of nesting before reaching Customer — making the normalization rationale visible
- **The Return entity** depends on InvoiceLine (weak entity) — a return cannot exist without an invoiced line

### 11.3 ERD as Documentation

The ERD is the most important documentation artifact for a database. A developer who reads the ERD before querying the database will understand the schema much faster than one who discovers it through trial-and-error SQL.

For the CabotTrail OLTP, an ERD generated from the schema (using SSMS's Database Diagrams feature or a tool like Lucidchart) shows the complete structure at a glance — every entity, every relationship, every cardinality constraint — in a form that is faster to read than dozens of CREATE TABLE statements.

---

## 12. Chapter Summary

- An ERD is a **conceptual model** of the business's information structure — readable by both technical and non-technical stakeholders. It is drawn before SQL and revised through stakeholder conversation.

- **Crow's Foot notation** uses rectangles (entities), lines with crow's foot symbols (relationships), and oval or in-box attributes. The symbols at each end of a relationship line express minimum and maximum cardinality.

- **Entities** are things the business tracks; identified from nouns in the narrative. **Attributes** are properties of entities; they become columns. Special attribute types: multi-valued (require a separate table), derived (computable from others), composite (multiple sub-components).

- **Cardinality** defines the numerical constraint on a relationship: 1:1, 1:N, M:N. **Participation** defines whether every instance must participate (mandatory) or may participate (optional). Both are business rules.

- **Weak entities** cannot be identified without their owner entity. They have a composite PK that includes the owner's PK. Surrogate keys make this partially invisible in physical models.

- **Many-to-many relationships** require a **junction table** (associative entity) that resolves the M:N into two 1:N relationships. Junction tables may carry their own attributes.

- **Supertypes and subtypes** model entity hierarchies where some entities share common attributes. The supertype holds shared attributes; subtypes hold specialised attributes plus a FK to the supertype.

- The ERD process: identify entities → identify attributes → identify relationships → determine cardinality → validate with stakeholders → surface unanswered questions.

---

## 13. Review Questions

1. Explain the difference between an ERD and a schema diagram. A colleague shows you a screenshot of SSMS's Database Diagram tool for `CabotTrailOutdoor` and calls it "the ERD." What is correct about this description and what is missing?

2. For each of the following pairs, determine the cardinality (1:1, 1:N, or M:N) and the participation (mandatory/optional for each side) of the relationship. Justify each answer with a business reason:
   a. `Supplier` ↔ `Product` (a supplier supplies products to CabotTrail)
   b. `Invoice` ↔ `InvoiceLine`
   c. `Employee` ↔ `SalesOrder` (employees take sales orders)
   d. `Product` ↔ `Color`

3. In the Loyalty Programme ERD from section 10, `LoyaltyTransaction` was identified as a weak entity. Explain why, and describe what the composite primary key would look like in the physical model.

4. Draw (or describe in text notation) the ERD for the following narrative:
   > "CabotTrail sells products. Each product belongs to at least one category, and each category may contain many products. A product has a name, a unit price, and a weight. Each product is supplied by exactly one supplier, but a supplier may supply many products."
   Include entities, attributes, relationships, cardinality, and participation.

5. Explain the difference between a multi-valued attribute and a one-to-many relationship. Give a specific example of each from the CabotTrail domain, and show how each is represented differently in the ERD.

6. The CabotTrail OLTP has `Application.People`, `Application.Employees`, and `Sales.Customers`. Identify the supertype/subtype relationship and explain which option was chosen for physical implementation (table per subtype, single table with NULLs, or combined). What are the trade-offs of this choice?

7. A new business requirement: CabotTrail wants to offer bundle deals — a bundle consists of two or more products sold together at a combined price. The bundle itself is treated as a product that can be ordered. Draw (or describe) the ERD extension needed to model this. Identify any junction tables required.

8. Run the FK relationship query from Chapter 1 against `CabotTrailOutdoor`. Pick the `Sales.Returns` entity and draw its immediate ERD neighbourhood — the entity itself, all entities it directly relates to, the cardinality, and participation of each relationship. What does the ERD tell you about the business rules governing returns?

---

## 🔍 Deeper Dive

### Going Further with Entity-Relationship Modelling

#### The Original Chen Notation

The ERD notation introduced by Peter Chen in his 1976 paper uses different symbols from Crow's Foot:

- **Rectangles** for entities
- **Diamonds** for relationships (with the relationship name inside)
- **Ovals** for attributes
- **Lines** from entities to diamonds to show participation
- **Double diamonds** for identifying relationships (with weak entities)
- **Double rectangles** for weak entities

Chen notation is more visually expressive — relationships are first-class objects with their own symbol — but it becomes cluttered for large schemas. Crow's Foot notation, developed later, is more compact and is now the industry standard.

Understanding both notations is useful because academic textbooks (Elmasri & Navathe, Connolly & Begg) use Chen notation, while industry tools (Visio, Lucidchart, draw.io, ERDPlus) use Crow's Foot or variations of it.

Chen, P. P. (1976). The entity-relationship model — Toward a unified view of data. *ACM Transactions on Database Systems*, 1(1), 9–36.

#### UML Class Diagrams vs ERDs

**UML (Unified Modelling Language) class diagrams** are sometimes used in place of ERDs for database design, especially in object-oriented development contexts. A UML class diagram shows:

- Classes (equivalent to entities)
- Attributes (with types)
- Associations (equivalent to relationships)
- Multiplicity (equivalent to cardinality, written as `0..1`, `1..*`, `0..*`)

The key differences from ERDs:
- UML class diagrams include **methods** (operations) — not relevant to relational databases
- UML uses different multiplicity notation (`0..1` instead of Crow's Foot symbols)
- UML supports **inheritance** notation more explicitly than ERDs

In practice, both notations are used. Database professionals tend to use ERDs; software architects tend to use UML. Knowing both is an advantage in teams that mix both disciplines.

#### When ERDs Are Not Enough: The Relational Schema Diagram

An ERD captures logical structure. For communicating the complete physical schema — including all columns, data types, constraint names, and index definitions — a **relational schema diagram** (what SSMS Database Diagrams generates) is more appropriate.

The two documents serve different audiences at different stages:
- ERD: design phase, stakeholder communication, conceptual review
- Schema diagram: implementation phase, developer reference, DBA documentation

Both should exist for a production database. The ERD captures intent; the schema diagram captures implementation.

#### Entity Identification Anti-Patterns

Some common mistakes in entity identification:

**Verbs as entities:** "Purchasing" is not an entity — "Purchase Order" is. "Delivery" as a standalone entity may be premature — the delivery information may belong as attributes of `Order` (delivery address, expected delivery date).

**Attributes as entities:** "Product Colour" is probably an attribute of `Product`, not an entity in its own right — unless the business needs to track colour codes independently (e.g., for colour management across product lines). The test: does `Colour` have multiple attributes? Does it exist independently of any product?

**Redundant entities:** "RegularCustomer" and "WholesaleCustomer" as separate entities, when both are instances of `Customer` with a `CustomerType` attribute.

**Missing entities:** The classic example — missing the `OrderLine` entity by jumping straight from `Order` to `Product` with a many-to-many. The ordered quantity, unit price at time of order, and pick status belong to the line, not to either `Order` or `Product`.

---

### References and Further Reading

1. Chen, P. P. (1976). The entity-relationship model — Toward a unified view of data. *ACM Transactions on Database Systems*, 1(1), 9–36. — The original ERD paper.

2. Connolly, T., & Begg, C. (2015). *Database Systems: A Practical Approach to Design, Implementation, and Management* (6th ed.). Pearson. — Chapters 12–14 cover ER modelling in depth with many worked examples.

3. Elmasri, R., & Navathe, S. (2016). *Fundamentals of Database Systems* (7th ed.). Pearson. — Chapters 3–4 cover the ER model using Chen notation; Chapter 9 covers enhanced ER (supertypes/subtypes).

4. Hoffer, J. A., Ramesh, V., & Topi, H. (2019). *Modern Database Management* (13th ed.). Pearson. — Strong practical coverage of ERD drawing techniques and case studies.

5. Microsoft. (2024). *Database Diagrams (SSMS)*. [https://learn.microsoft.com/en-us/sql/ssms/visual-db-tools/design-database-diagrams-visual-database-tools](https://learn.microsoft.com/en-us/sql/ssms/visual-db-tools/design-database-diagrams-visual-database-tools)

6. Lucidchart. (2024). *ERD Tutorial*. [https://www.lucidchart.com/pages/er-diagrams](https://www.lucidchart.com/pages/er-diagrams) — Practical Crow's Foot notation reference and drawing guide.

---

*Previous chapter: [Chapter 1 — The Relational Model Revisited: From Querying to Designing](../chapter-01-relational-model-revisited/README.md)*

*Next chapter: [Chapter 3 — From ERD to Tables: Mapping Diagrams to DDL](../chapter-03-erd-to-tables/README.md)*

---

> **Database Design for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
