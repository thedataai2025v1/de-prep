# Module 1: Overview

## Lesson 1.4 — Data Modeling Fundamentals

### What Is Data Modeling?

Data modeling is the practice of creating a conceptual representation of how data is structured, related, and stored in a system. In the context of **data warehousing and analytics**, it focuses on organizing data for efficient querying and business analysis.

---

### Types of Data Models

#### 1. Conceptual Model
- **Purpose:** High-level view for business stakeholders
- **Content:** Key entities and their relationships
- **No technical details**

```
Example:
Customer -- places --> Order -- contains --> Product
```

#### 2. Logical Model
- **Purpose:** Detailed structure independent of technology
- **Content:** Entities, attributes, data types, relationships, cardinality
- **Used for:** Team alignment, requirements documentation

#### 3. Physical Model
- **Purpose:** Implementation-specific schema
- **Content:** Tables, columns, indexes, partitions, storage format
- **Depends on:** Specific database (BigQuery, Snowflake, Redshift, etc.)

---

### Dimensional Modeling (Kimball Approach)

The **Kimball methodology** is the industry standard for data warehouse design. It emphasizes:

1. **Business-driven** — Start from business processes
2. **Incremental** — Build in scopes, not big-bang
3. **Integrated** — Shared (conformed) dimensions across fact tables
4. **User-friendly** — Easy for business users to query

#### Core Concepts

| Concept | Description | Example |
|---|---|---|
| **Fact Table** | Numeric measures that can be aggregated | sales_amount, quantity, revenue |
| **Dimension Table** | Contextual attributes for filtering/grouping | customer_name, product_category, date |
| **Grain** | The level of detail of one row in a fact table | "One row = one line item in one order" |
| **Star Schema** | Fact table surrounded by denormalized dimensions | Simple, fast queries |
| **Snowflake Schema** | Normalized dimensions (dimensions have child dimensions) | More storage-efficient, complex joins |

---

### Star Schema vs Snowflake Schema

```
STAR SCHEMA:
┌──────────────────┐
│   fact_sales     │
│ (order_id, date_key, product_key, customer_key, qty, amount) │
└──────────────────┘
    │         │         │
    ▼         ▼         ▼
┌──────────┐ ┌────────────┐ ┌────────────┐
│ dim_date │ │ dim_product│ │ dim_customer│
│ (date,   │ │ (product_id│ │ (customer_id│
│  year,   │ │  name,     │ │  name,      │
│  month)  │ │  category, │ │  region,    │
│          │ │  subcat)   │ │  country)   │
└──────────┘ └────────────┘ └────────────┘

SNOWFLAKE SCHEMA:
┌──────────────────┐
│   fact_sales     │
└──────────────────┘
    │
    ▼
┌────────────┐     ┌────────────┐
│ dim_product│─────│ dim_category│
│ (product_id│     │ (category_id│
│  name,     │     │  name)      │
│  cat_id)   │     └────────────┘
└────────────┘
```

| Criteria | Star Schema | Snowflake Schema |
|---|---|---|
| Query complexity | Simple (fewer joins) | Complex (more joins) |
| Storage | More (denormalized) | Less (normalized) |
| Query performance | Generally faster | Slower (more joins) |
| Maintenance | Easier to understand | Harder to understand |
| When to use | **Default choice** for DW | When dimensions are very large with many shared attributes |

---

### Fact Table Types

#### 1. Transaction Fact Table
- **Grain:** One row per transaction event
- **Purpose:** Record each individual event
- **Example:** Sales, clicks, page views
- **Characteristics:** High volume, never updated

```sql
CREATE TABLE fact_orders (
    order_id BIGINT PRIMARY KEY,
    order_date_key INT,
    customer_key INT,
    product_key INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(10,2)
);
-- Grain: One row = one line item in one order
```

#### 2. Periodic Snapshot Fact Table
- **Grain:** One row per subject per time period
- **Purpose:** Track state at regular intervals (daily balance, monthly inventory)
- **Example:** Daily account balances, monthly inventory levels
- **Characteristics:** Semi-additive measures

```sql
CREATE TABLE fact_daily_balance (
    snapshot_date DATE,
    account_key INT,
    balance DECIMAL(12,2),
    PRIMARY KEY (snapshot_date, account_key)
);
-- Grain: One row = one account's balance on one date
```

#### 3. Accumulating Snapshot Fact Table
- **Grain:** One row per business process instance
- **Purpose:** Track milestones in a process with a lifecycle
- **Example:** Order lifecycle (ordered, shipped, delivered, returned)
- **Characteristics:** Updated as milestones occur

```sql
CREATE TABLE fact_order_lifecycle (
    order_id BIGINT PRIMARY KEY,
    customer_key INT,
    order_date TIMESTAMP,
    confirmed_date TIMESTAMP,
    shipped_date TIMESTAMP,
    delivered_date TIMESTAMP,
    returned_date TIMESTAMP,
    total_amount DECIMAL(10,2)
);
-- Grain: One row = one order through its lifecycle
```

---

### Dimension Table Concepts

#### Role-Playing Dimensions
A single dimension table used in multiple roles:

```sql
-- dim_date serves as both order_date and ship_date
SELECT d_order.date, d_ship.date, SUM(f.amount)
FROM fact_orders f
JOIN dim_date d_order ON f.order_date_key = d_order.date_key
JOIN dim_date d_ship  ON f.ship_date_key  = d_ship.date_key
GROUP BY d_order.date, d_ship.date;
```

#### Degenerate Dimensions
Attributes that have no separate dimension table — they live directly in the fact table:

```
Example: transaction_id, invoice_number in fact_payments
(They are used for filtering but don't need a dimension table)
```

#### Junk Dimensions
Grouping low-cardinality flag/indicator attributes into a single dimension:

```sql
CREATE TABLE dim_junk_order (
    junk_key INT PRIMARY KEY,
    rush_flag CHAR(1),
    gift_wrap_flag CHAR(1),
    promo_flag CHAR(1)
);
```

---

### Normalization Forms (Quick Reference)

| Form | Rule | Example |
|---|---|---|
| **1NF** | Atomic values, no repeating groups | Split "phone1, phone2" into separate rows |
| **2NF** | No partial dependency on composite key | Every attribute depends on the full key |
| **3NF** | No transitive dependency | Non-key attributes don't depend on other non-key attributes |

> **Key point:** Data warehouses are typically **de-normalized** (star schema) for query performance, while operational databases are normalized (3NF).

---

### Surrogate Keys vs Natural Keys

| Feature | Surrogate Key | Natural Key |
|---|---|---|
| Source | Generated by the data warehouse | Comes from the source system |
| Type | Usually sequential integer | Business key (email, product code) |
| Stability | Never changes | Can change (e.g., customer ID renumbering) |
| Use in DW | **Always use surrogate keys** for dimension tables | Used for mapping to source |

```sql
-- Good practice: keep both
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY,          -- surrogate key (auto-generated)
    customer_id VARCHAR(50),                -- natural key (from source)
    name VARCHAR(100),
    email VARCHAR(100),
    region VARCHAR(50)
);
```

---

### Interviewer Questions You'll Face

1. "What's the difference between a star schema and a snowflake schema?"
2. "Explain the grain of a fact table with an example."
3. "What are the three types of fact tables?"
4. "When would you use a surrogate key vs a natural key?"
5. "What is a role-playing dimension?"

---

### Top-Up Questions (Senior-Level Probes)

- "How do conformed dimensions enable cross-business-unit analysis?"
- "What is a bridge table and when do you need one?"
- "Explain the difference between additive, semi-additive, and non-additive measures."
- "How does Kimball's methodology differ from Inmon's approach?"
- "When would you choose a data vault approach over dimensional modeling?"

---

### Medallion Architecture & Lakehouse Context

In modern lakehouse architectures (Databricks, Snowflake), dimensional models often follow the **Medallion Architecture**:

```
Raw Zone (Bronze)         Staging & Cleansed (Silver)         Analytics (Gold)
┌──────────────────┐      ┌──────────────────────┐           ┌──────────────────┐
│ Raw source data   │────▶│ Cleansed, deduplicated│────────▶ │ Star schema      │
│ Change data       │      │ Conformed dimensions  │           │ Fact tables      │
│ No schema enforce │      │ SCD handling          │           │ Aggregates       │
└──────────────────┘      └──────────────────────┘           └──────────────────┘
```

| Layer | What It Holds | Data Modeling Role |
|---|---|---|
| **Bronze** | Raw source data, append-only | Minimal transformation; preserve history |
| **Silver** | Cleansed, deduplicated, conformed dimensions | Business rules applied; SCD logic lives here |
| **Gold** | Star schemas, aggregates, marts | Ready for BI and analytics |

> **Interview tip:** Mentioning medallion architecture shows you understand how dimensional models fit into modern data platforms beyond traditional warehouses.

---

### Practice Exercise

> **Prompt:** Design the core star schema for an online bookstore.
>
> 1. Define the grain of the main fact table.
> 2. List all dimension tables with 4–5 key columns each.
> 3. Identify which dimension is role-playing.
> 4. Identify any degenerate dimensions.
> 5. Draw the schema with relationships.

---

### Related Lessons

- ⬅️ [Lesson 3: Rubric for Data Modeling Questions](./03-rubric-for-data-modeling-questions.md) — what you'll be scored on
- ➡️ [Lesson 5: Introduction to Gathering Business Requirements](./05-introduction-to-gathering-business-requirements.md) — start the requirements module
- ➡️ [Lesson 13: Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md) — translate fundamentals to diagrams
- 🔗 [Lesson 22: Degenerate and Junk Dimensions](./22-degenerate-and-junk-dimensions.md) — advanced dimension patterns

