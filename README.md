# Database Design for Analysts
### *From business requirements to relational structures*

> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## About This Book

This book teaches relational database design — the discipline of turning business requirements into structured, efficient, trustworthy data stores. It is the second course in a three-course sequence:

- **DBAS 5010 — SQL for Analysts** *(prerequisite)*: Writing queries, filtering, joining, aggregating
- **DBAS 2010 — Database Design for Analysts** *(this book)*: Designing the structures that hold data
- **DBAS 2103 — Data Provisioning with ETL**: Building pipelines that move data between structures

The first course taught you how to *read* data from databases someone else designed. This book teaches you how to *design* those databases — how to look at a business problem and produce a relational model that accurately represents the business's reality, enforces its rules, and supports the queries that analysts need.

The skills here are not simply technical. Database design is a translation discipline: you receive vague, ambiguous descriptions of how a business works, ask careful questions, and produce a precise, unambiguous specification in the form of an entity-relationship diagram and a set of SQL table definitions. That translation — from human language to relational structure — is what this course is about.

---

## Who This Book Is For

This book is written for students who have completed DBAS 5010 (or equivalent). You should be comfortable writing SELECT statements, joining tables, and understanding how primary and foreign keys link tables together. You do not need prior experience with database design — that is what this course teaches.

---

## The CabotTrail Outdoor Environment

All examples use the CabotTrail Outdoor database environment you worked with in DBAS 5010. By this point you know these databases from the analyst's side — you have queried their tables, joined their dimensions, and aggregated their facts. This course turns the lens around: you will learn to *explain why* those tables are designed the way they are, and ultimately design new structures of your own.

The course culminates in Chapter 9 with the design and implementation of **`CabotTrailExecutive`** — an executive summary database built from requirements, designed from scratch, and populated with data from the existing environment. This is the same database referenced in the ETL course that follows.

| Database | Used in chapters |
|---|---|
| `CabotTrailOutdoor` (OLTP) | 1–9 (primary design reference) |
| `CabotTrailOutdoorDW` (Data Warehouse) | 7–9 (OLAP design comparison) |
| `CabotTrailOutdoorsSales` (Data Mart) | 7–9 (star schema reference) |
| `CabotTrailExecutive` (design project) | 9 (built in this course) |

---

## How to Use This Book

Each chapter follows the same structure as the companion textbooks in this series:

| Section | Purpose |
|---|---|
| **Chapter Overview** | Learning outcomes and what to expect |
| **Concept sections** | Core content with CabotTrail examples and SQL |
| **Chapter Summary** | Key takeaways in condensed form |
| **Review Questions** | Eight questions for consolidation and self-assessment |
| **🔍 Deeper Dive** | Extended concepts, theory, industry context, and references |

---

## Chapters

| # | Chapter | Topics | Course weeks |
|---|---|---|---|
| 1 | [The Relational Model Revisited — From Querying to Designing](./chapter-01-relational-model-revisited/README.md) | Entities, attributes, relationships, three-schema architecture, logical vs physical models, keys, reading CabotTrail OLTP schema, design anomalies | Week 1 |
| 2 | [Entity-Relationship Diagrams — Drawing the Business](./chapter-02-entity-relationship-diagrams/README.md) | Crow's Foot notation, entities, attributes, cardinality, participation, strong/weak entities, M:N junction tables, supertypes/subtypes, Loyalty Programme worked example | Week 1 |
| 3 | [From ERD to Tables — Mapping Diagrams to DDL](./chapter-03-erd-to-tables/README.md) | Seven mapping rules, strong/weak/associative entities, 1:N FK placement, 1:1 UNIQUE enforcement, M:N junction tables, supertype/subtype DDL, data type selection, naming conventions, complete Loyalty Programme mapping | Week 2 |
| 4 | [Normalization: 1NF and 2NF — Removing Redundancy](./chapter-04-normalization-1nf-2nf/README.md) | Functional dependencies, UNF starting point, 1NF atomicity violations, 2NF partial dependencies, FD diagrams, lossless decomposition, CabotTrail analysis | Week 3 |
| 5 | [Normalization: 3NF and Beyond — Completing the Model](./chapter-05-normalization-3nf/README.md) | Transitive dependencies, 3NF definition, Kent's mnemonic, BCNF, 4NF/5NF overview, complete UNF→3NF worked example, CabotTrail 3NF verification, denormalization | Week 3 |
| 6 | [Constraints and Data Integrity — Encoding Business Rules](./chapter-06-constraints-integrity/README.md) | NOT NULL/NULL design, PRIMARY KEY, IDENTITY, FOREIGN KEY, referential actions, UNIQUE, CHECK, DEFAULT, computed columns, constraint naming, catalog queries | Week 4 |
| 7 | [Indexes and Physical Design — Making Queries Fast](./chapter-07-indexes-physical-design/README.md) | Pages and B-tree structure, clustered vs non-clustered, covering indexes, execution plans, sargability, write trade-offs, composite index column order, CabotTrail index strategy | Week 4 |
| 8 | [OLTP vs OLAP — Two Design Philosophies](./chapter-08-oltp-vs-olap/README.md) | OLTP vs OLAP workload characteristics, star schema, dimension and fact table design, surrogate keys in DW context, SCD types, conformed dimensions, ETL as the bridge, CabotTrail end-to-end architecture | Week 5 |
| 9 | [Putting It Together — Designing CabotTrailExecutive](./chapter-09-cabottrail-executive/README.md) | Complete design process, requirements analysis, ERD, normalization verification, DDL with all constraints, index strategy, population from DW, reconciliation, ETL architecture positioning | Weeks 5–6 |

---

## Licence

This work is licensed under the [Creative Commons Attribution 4.0 International Licence (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

**Suggested citation:**
> Dolinger, P. (2027). *Database Design for Analysts: From business requirements to relational structures*. NSCC Institute of Technology. Licensed under CC BY 4.0. https://creativecommons.org/licenses/by/4.0/

---

## About the Author

**Patrick Dolinger** teaches Business Intelligence and Data Science courses at NSCC Institute of Technology as part of the one-year graduate certificate programs in Data Analytics and Business Intelligence.

Contact: Patrick.Dolinger@nscc.ca

---

*This is the second book in a three-part series:*
- *[SQL for Analysts](../sql-for-analysts/README.md)*
- ***Database Design for Analysts*** *(this book)*
- *[ETL for Business Intelligence](../etl-for-business-intelligence/README.md)*

---

*NSCC Institute of Technology | Halifax, Nova Scotia, Canada*
