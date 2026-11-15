# Lesson 31 — Advanced Dimension Design Techniques

## Learning Objectives
- Understand mini-dimensions and when to use them
- Design outrigger dimensions for shared reference data
- Model many-to-many relationships with bridge tables
- Handle hierarchical and role-playing dimensions
- Classify dimension attributes by change frequency and analytical purpose

---

## The Problem with Wide Dimensions

A customer dimension with 150 attributes that change at very different rates is hard to manage:
- Stable attributes (name, signup date) rarely change
- Volatile attributes (engagement band, risk level) change daily
- Mixing them in one Type 2 dimension causes row explosion

```
-- Bad: one massive Type 2 dimension
dim_customer (
    customer_sk,
    name, signup_date,         -- stable, rarely changes
    engagement_band,           -- changes daily
    risk_level,                -- changes weekly
    spend_tier,                -- changes monthly
    valid_from, valid_to, is_current
)
-- Every time engagement_band changes, you create a new row
-- even though name and signup_date haven't changed
```

### Solution: Decompose by Change Frequency

Split into dimensions that change at different rates:
- **Durable dimension** — stable identity, rarely changes
- **Mini-dimensions** — volatile attributes grouped by change rate

---

## 1. Mini-Dimensions

A mini-dimension isolates a group of low-cardinality, frequently-changing attributes into a separate small table.

### How It Works
```sql
-- Durable customer dimension (rarely changes)
CREATE TABLE dim_customer (
    customer_sk       BIGINT,
    customer_id       VARCHAR(50),
    name              VARCHAR(200),
    signup_date       DATE,
    acquisition_channel VARCHAR(50),
    valid_from        DATE,
    valid_to          DATE,
    is_current        BOOLEAN
);

-- Mini-dimension: customer profile (changes daily)
CREATE TABLE dim_customer_profile (
    profile_sk        BIGINT,
    engagement_band   VARCHAR(20),  -- high, medium, low
    risk_level        VARCHAR(20),  -- low, medium, high
    spend_tier        VARCHAR(20),  -- platinum, gold, silver, bronze
    valid_from        DATE,
    valid_to          DATE,
    is_current        BOOLEAN
);

-- Fact table references both
CREATE TABLE fact_order (
    order_sk          BIGINT,
    customer_sk       BIGINT,      -- points to dim_customer
    customer_profile_sk BIGINT,    -- points to dim_customer_profile at order time
    order_date        DATE,
    order_amount      DECIMAL(10,2)
);
```

**Key insight:** `customer_profile_sk` is a **Type 2** key — the fact captures which profile the customer had at the time of the order.

### When to Use Mini-Dimensions
- Attributes change frequently (daily or more)
- Low cardinality per attribute (e.g., 3–5 values each)
- Total combination space is manageable (3 × 4 × 4 = 48 possible profiles)
- You need historical snapshots of the volatile attributes

### Cardinality Problem
If a mini-dimension has too many attributes with too many values, the combination space explodes:
```
5 attributes × 10 values each = 10^5 = 100,000 rows
8 attributes × 10 values each = 10^8 = 100 million rows  ← too big
```

**Rule of thumb:** Keep mini-dimensions under ~10,000 rows. If they grow larger, split further.

---

## 2. Outrigger Dimensions

An outrigger (or satellite) dimension references a separately governed entity that is shared across multiple fact tables or domains.

### Example: Supplier Dimension
```
fact_purchase_order  →  dim_supplier  ←  fact_delivery
                         ↑
                   shared across domains
```

**When to use:**
- The entity has its own lifecycle and governance (e.g., supplier certification, vendor compliance)
- Multiple fact tables need to reference it
- It's maintained by a different team

**When NOT to use:**
- The entity is simple and rarely changes — just embed the attributes
- It's only used by one fact table — just include it in the dimension

### Pros and Cons
| Aspect | With Outrigger | Without (Embedded) |
|---|---|---|
| Joins | Extra join needed | No extra join |
| Governance | Centralized, shared | Duplicated across models |
| Maintenance | One source of truth | Multiple places to update |
| BI Complexity | Higher | Lower |

---

## 3. Many-to-Many: Bridge Tables

When one entity can have many values of another (e.g., a customer belongs to multiple loyalty programs), a simple foreign key doesn't work.

### Bridge Table Structure
```sql
CREATE TABLE bridge_customer_program (
    customer_sk   BIGINT,
    program_sk    BIGINT,
    start_date    DATE,
    end_date      DATE,
    allocation_weight DECIMAL(5,4)  -- for revenue attribution
);
```

**Critical issue:** Joining through a bridge multiplies fact rows. If a customer belongs to 3 programs, one order becomes 3 rows.

### Revenue Attribution
When measures need to go through a bridge, you must define an allocation rule:
```sql
-- Equal split: each program gets 1/3 of the revenue
SELECT
    bp.program_sk,
    SUM(fo.order_amount / COUNT(bp.program_sk)) AS allocated_revenue
FROM fact_order fo
JOIN bridge_customer_program bp ON fo.customer_sk = bp.customer_sk
WHERE fo.order_date BETWEEN bp.start_date AND bp.end_date
GROUP BY bp.program_sk;
```

**Rule:** Never sum measures directly through a bridge without allocation. The business must approve the allocation rule.

---

## 4. Hierarchy Bridge Tables

Hierarchical rollups (org charts, product categories, geographic hierarchies) can be:
- Simple trees (each node has one parent)
- Ragged hierarchies (variable depth)
- Recursive (multiple paths to the root)

### Simple Parent Key
```sql
-- Good for flat, single-path hierarchies
CREATE TABLE dim_organization (
    org_sk      BIGINT,
    org_id      VARCHAR(50),
    org_name    VARCHAR(200),
    parent_sk   BIGINT,     -- points to its own table
    level       INT          -- depth in hierarchy
);
```

### Hierarchy Bridge (for complex, historical hierarchies)
```sql
CREATE TABLE bridge_org_hierarchy (
    org_sk          BIGINT,
    ancestor_sk     BIGINT,     -- can be any ancestor, not just parent
    depth           INT,         -- 1 = parent, 2 = grandparent, etc.
    valid_from      DATE,
    valid_to        DATE
);
```

This supports:
- Multiple reporting paths (an employee in two departments)
- Historical rollups (who reported to whom at a past date)
- Ragged hierarchies (no fixed depth)

---

## 5. Role-Playing Dimensions

A single dimension used in multiple roles in the same fact table:
```sql
CREATE TABLE fact_order (
    order_sk          BIGINT,
    order_date_sk     BIGINT,    -- dim_date as "order date"
    ship_date_sk      BIGINT,    -- dim_date as "ship date"
    due_date_sk       BIGINT,    -- dim_date as "due date"
    ...
);
```

The same `dim_date` table serves three roles. The distinction is which column in the fact table references it.

**In BI tools:** You define separate roles for the same dimension table so the semantic layer knows which one is "order date" vs "ship date."

---

## 6. Junk Dimensions

Group low-cardinality flags, status indicators, and yes/no columns into a single dimension:
```sql
CREATE TABLE dim_junk (
    junk_sk   INT,
    is_rushed BOOLEAN,
    is_gift   BOOLEAN,
    delivery_type VARCHAR(20),  -- standard, express, overnight
    payment_status  VARCHAR(20) -- paid, pending, refunded
);
```

**When to use:** When you have 5+ flag columns in a fact table
**When NOT to use:** Don't bundle unrelated attributes — only group related ones

---

## 7. Snowflaking (and When to Avoid It)

Snowflaking normalizes a dimension into multiple related tables:
```
dim_product → dim_category → dim_department
```

**Pros:** Less duplication, easier maintenance
**Cons:** More joins, harder for analysts to use

**Rule of thumb:** Snowflake only when:
- The referenced entity has independent lifecycle or governance
- Duplication would be excessive (thousands of products per category)
- The BI semantic layer handles the joins transparently

---

## 8. Temporal Accuracy

Distinguish two kinds of time:
- **Valid time** — when the attribute was true in the business
- **System time** — when the platform recorded the value

A **bitemporal** model tracks both, needed when corrections arrive after the fact and you must answer:
1. "What do we now believe was true at that time?"
2. "What did we believe at that time?"

```sql
CREATE TABLE dim_customer_bitemporal (
    customer_sk  BIGINT,
    engagement_band VARCHAR(20),
    valid_from    DATE,    -- when it was true in the business
    valid_to      DATE,
    sys_from      TIMESTAMP, -- when we recorded it
    sys_to        TIMESTAMP
);
```

---

## 9. Privacy Considerations

Advanced dimensions often hold sensitive data (addresses, risk bands, health categories). Apply:
- **Data minimization** — store only what the analytical use case needs
- **Masking or tokenization** — hash sensitive identifiers
- **Retention and deletion policies** — honor "right to erasure"
- **Audit trails** — track changes to sensitive attributes
- **Access controls** — restrict who can see sensitive dimension attributes

---

## Pattern Selection Guide

| Requirement | Pattern | Watch Out For |
|---|---|---|
| Frequently changing profile bands | Mini-dimension | Profile combination cardinality |
| Shared separately governed reference | Outrigger | Extra joins and BI complexity |
| One entity has many memberships | Bridge table | Measure fan-out and allocation |
| Recursive or ragged hierarchy | Hierarchy bridge | Bridge growth and multiple paths |
| Same entity in multiple roles | Role-playing dimension | Role semantics must be explicit |
| Several low-cardinality flags | Junk dimension | Avoid unrelated attribute bundling |
| High-frequency attribute changes | Event/history or snapshot | Temporal query complexity |

---

## Worked Example: Customer Analytics Platform

A retailer needs to analyze orders by:
- Customer's stable acquisition channel
- Customer's engagement band at order time
- Loyalty programs the customer belonged to at order time
- Current customer profile for operational dashboards

### Model
1. `dim_customer` — durable identity and stable attributes
2. `dim_customer_profile` — engagement/risk/spend band combinations (Type 2)
3. `bridge_customer_program` — dated program memberships with allocation weights
4. `fact_order` — stores `customer_sk`, `customer_profile_sk`, event timestamp, measures
5. A current-state view — exposes latest profile for operational dashboards

**Key concerns:**
- Membership joins can multiply order rows
- Revenue attribution needs an approved allocation rule
- Engagement-band history requires the profile key resolved at order event time

---

## Interview Dialogue

### Interviewer: "A customer has 150 behavioral attributes, many change daily. Would you put all of them in a Type 2 customer dimension?"

**Candidate:**
> "Not automatically. I'd classify attributes by analytical purpose, change frequency, and historical requirement. Stable identity belongs in the durable customer dimension. Frequently changing banded attributes could be grouped into a mini-dimension, while raw behavioral changes may be better represented as events or periodic snapshots.
>
> I'd start by listing all 150 attributes, grouping them by how often they change, then deciding which group needs history and which doesn't."

### Interviewer: "What if a customer belongs to multiple loyalty programs?"

**Candidate:**
> "That's a many-to-many relationship, so I'd use a bridge table with customer key, program key, and effective dates. If we need to allocate revenue across programs, we need a business-approved allocation rule and weights in the bridge. Otherwise, a simple join multiplies revenue and produces misleading totals."

### Interviewer: "Why not snowflake every dimension to reduce duplication?"

**Candidate:**
> "Normalization reduces repeated attributes but increases joins and complexity for analysts. I'd snowflake only where the referenced entity has independent lifecycle or governance value, or where duplication is genuinely problematic. I'd test query patterns and the BI semantic layer before standardizing the pattern."

### Interviewer: "How would you represent an organization hierarchy that changes over time?"

**Candidate:**
> "If it's a simple tree and consumers only need current parent relationships, a parent key in the dimension suffices. For historical rollups, variable depth, or multiple reporting paths, I'd use an effective-dated hierarchy bridge. I'd explicitly define how multiple paths affect aggregation to prevent double counting."

---

## Key Takeaways

| Concept | Rule of Thumb |
|---|---|
| Mini-dimensions | Isolate rapidly changing profiles from durable identity |
| Outriggers | Centralize separately governed entities; accept extra joins |
| Bridge tables | Model many-to-many; always define measure allocation |
| Hierarchy bridges | Support recursive, ragged, historical rollups |
| Snowflaking | Only when the referenced entity has independent governance |
| Junk dimensions | Group related flags; don't bundle unrelated attributes |
| Bitemporal | When corrections arrive after the fact |
| Pattern selection | Solve a demonstrated problem, not add abstraction for its own sake |

---

## Related Lessons
- [Lesson 21 — Slowly Changing Dimensions Types 1–4](./21-slowly-changing-dimensions-types-1-4.md)
- [Lesson 22 — Degenerate and Junk Dimensions](./22-degenerate-and-junk-dimensions.md)
- [Lesson 23 — Dimension Design Practice](./23-dimension-design-practice.md)
- [Lesson 32 — Bridge Tables and Factless Fact Tables](./32-bridge-tables-and-factless-facts.md)
- [Lesson 24 — Fact Table Types](./24-fact-table-types.md)
