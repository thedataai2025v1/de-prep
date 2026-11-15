# Module 3: High-Level Model Diagrams
## Lesson 3.2 — Evolving Models Based on Changing Requirements

> *"The only thing constant in data is change."* — Every analyst who has lived through a product launch.

Models are never "done." Business priorities shift, new products launch, regulations change, and the metrics that matter today differ from those that mattered six months ago. A senior data modeler doesn't just build a good initial model — they design for change.

---

### 1. Why Models Evolve (and How to Handle Change Gracefully)

Models evolve for several reasons:

| Trigger | Example |
|---|---|
| **New business lines** | Adding a subscription service to an e-commerce platform |
| **Regulatory changes** | GDPR, CCPA require new PII tracking columns |
| **New metrics/KPIs** | CFO requests cohort-based lifetime value |
| **Performance issues** | Fact table growing too large; needs partitioning or splitting |
| **Data source changes** | Upstream system refactored, dropping old fields |

**Golden rule:** Design your schema so that *most* changes are additive (new columns, new tables) rather than destructive (renames, drops, grain changes). Non-breaking changes mean downstream dashboards and reports keep working.

---

### 2. Common Types of Changes

#### 2a. New Business Metrics → Add Columns to Existing Fact Tables

When a new measure fits the **existing grain** of a fact table, simply add a column.

```sql
-- Existing fact_orders (grain: one row per order line item)
ALTER TABLE fact_orders
  ADD COLUMN discount_amount DECIMAL(12,2),
  ADD COLUMN tax_amount DECIMAL(12,2);
```

This is the easiest, least risky change. Downstream consumers simply ignore the new columns if they don't need them.

#### 2b. New Dimensions → Create New Dimension Table + FK to Fact

When business asks for a new slicing capability (e.g., by *sales channel* or *fulfillment center*):

```sql
-- New dimension
CREATE TABLE dim_channel (
    channel_id     INT          PRIMARY KEY,
    channel_name   VARCHAR(50),
    channel_type   VARCHAR(20),   -- 'online', 'retail', 'wholesale'
    is_active      BOOLEAN
);

-- Link to existing fact
ALTER TABLE fact_orders
    ADD COLUMN channel_id INT REFERENCES dim_channel(channel_id);
```

#### 2c. New Fact Tables → New Business Process or Different Grain

Each fact table should represent **one business process at one grain**. When a new process emerges, create a new fact table.

```
-- E.g., the business starts tracking customer support tickets
-- This is a different process from orders → new fact table
CREATE TABLE fact_support_tickets (
    ticket_id        INT          PRIMARY KEY,
    customer_id      INT          REFERENCES dim_customer,
    product_id       INT          REFERENCES dim_product,
    created_date_key INT          REFERENCES dim_date,
    resolved_date_key INT         REFERENCES dim_date,
    ticket_category  VARCHAR(30),
    satisfaction_score INT
);
```

#### 2d. Splitting a Fact Table → When Grain Diverges

If two metrics in one fact table start recording at **different granularities**, split the table:

| Before (mixed grain) | After (separate grain) |
|---|---|
| `fact_order_summary` (one row per order header + some line-level fields) | `fact_orders` (per order line) + `fact_order_summary` (per order header) |

#### 2e. Merging Dimensions → When Consolidation Makes Sense

Two nearly-identical dimensions (e.g., `dim_store` and `dim_fulfillment_center`) can merge into `dim_location` if the business treats them as one entity.

---

### 3. Schema Evolution Strategies

| Strategy | Breaking? | Effort | Typical Use |
|---|---|---|---|
| **Add column** | No | Low | New metrics |
| **Add table** | No | Low–Medium | New process/dimension |
| **Rename column** | **Yes** | Medium | Poor initial naming; requires migration |
| **Drop column** | **Yes** | Medium | Field no longer needed |
| **Change grain** | **Yes** | High | Requires full rebuild + downstream fixes |

#### Backfilling Historical Data

Whenever you add a column or table, decide: **Do existing rows need data?**

```sql
-- Backfill new column from a source system
UPDATE fact_orders
SET discount_amount = src.discount_val
FROM staging.promotion_history src
WHERE fact_orders.order_id = src.order_id
  AND fact_orders.order_date_key BETWEEN '20240101' AND '20240630';
```

For new dimensions, join back through existing attributes to populate FKs for old data.

---

### 4. Communicating Changes to Stakeholders

Schema changes affect analysts, data scientists, BI developers, and application engineers. Communicate clearly:

| Audience | What They Need to Know |
|---|---|
| **BI developers** | New columns available; any old columns deprecated |
| **Data scientists** | New tables in the data catalog; training data may shift |
| **Product managers** | New metrics now available; old dashboard may need update |
| **Engineering** | ETL pipeline changes; upstream contract changes |

Use a **Data Change Log** (a simple Confluence page or markdown file):

```markdown
## Change #14 — Added subscription tracking (2024-06-15)
- New tables: `dim_subscription`, `fact_subscription_events`
- New column on `dim_customer`: `subscription_tier`
- Deprecated: `fact_orders.is_recurring` (moved to subscription model)
- Affected dashboards: Revenue Overview, Customer Cohort Analysis
```

---

### 5. Versioning Approaches

| Approach | How It Works | Pros | Cons |
|---|---|---|---|
| **Table versioning** | `fact_orders_v1`, `fact_orders_v2` | Full rollback; easy audit | Duplicates storage; consumers must switch |
| **Migration scripts** | Versioned SQL scripts (`V014__add_subscription.sql`) | Clean history; idempotent | Breaks if run out of order |
| **Dual-write period** | Write to both old and new schema for 1–2 weeks | Zero downtime; validate parity | Double compute cost during transition |

**Recommendation for most interview scenarios:** Use migration scripts + a short dual-write window, then switch consumption to the new schema.

```sql
-- Example migration script (Liquibase/Flyway style)
-- V014__add_subscription_tracking.sql

CREATE TABLE dim_subscription (
    subscription_id  INT PRIMARY KEY,
    plan_name        VARCHAR(50),
    monthly_price    DECIMAL(10,2),
    start_date_key   INT REFERENCES dim_date,
    end_date_key     INT REFERENCES dim_date
);

CREATE TABLE fact_subscription_events (
    event_id         BIGINT PRIMARY KEY,
    subscription_id  INT REFERENCES dim_subscription,
    customer_id      INT REFERENCES dim_customer,
    event_date_key   INT REFERENCES dim_date,
    event_type       VARCHAR(20),   -- 'signup', 'cancel', 'upgrade'
    event_amount     DECIMAL(12,2)
);

-- Backfill from legacy data
INSERT INTO dim_subscription (...)
SELECT ... FROM legacy.subscriptions;
```

---

### 6. Real Interview Scenario: Adding Subscription Tracking to an E-Commerce Model

**Scenario:** You built a star schema for an online store (`fact_orders`, `dim_product`, `dim_customer`, `dim_date`, `dim_store`). The business now launches a subscription box service. Walk me through the changes.

#### Step 1: Analyze Impact

| Question | Answer |
|---|---|
| Is this a new business process? | **Yes** — recurring payments differ from one-time purchases |
| Does it fit the existing grain? | **No** — subscriptions are per-member-month, not per-order |
| Do existing tables need changes? | `dim_customer` gains a `subscription_tier` column |
| What new dimensions are needed? | `dim_subscription` (plans), `dim_subscription_period` |

#### Step 2: Propose Changes

```
New/Modified Objects:
┌─────────────────────────────────────────┐
│ dim_customer  ───► add: subscription_tier│
│                                         │
│ +dim_subscription                       │
│   subscription_id (PK), plan_name,      │
│   monthly_price, start/end dates        │
│                                         │
│ +dim_subscription_period                │
│   period_id, year, month, is_current    │
│                                         │
│ +fact_subscription_events               │
│   event_id, subscription_id, customer_id│
│   event_date, event_type, amount        │
│                                         │
│ (fact_orders stays as-is)               │
└─────────────────────────────────────────┘
```

#### Step 3: Updated Diagram (Text Form)

```
        dim_customer              dim_subscription
    ┌──────────────┐          ┌──────────────────┐
    │ customer_id ◄─┤──+      │ subscription_id ◄─┤──┐
    │ name         │  │       │ plan_name        │  │
    │ email        │  │       │ monthly_price    │  │
    │ sub_tier ────┤──┘       │ start_date       │  │
    └──────────────┘          │ end_date         │  │
                              └────────┬─────────┘  │
                                       │            │
        dim_date                        │            │
    ┌──────────────┐                    │            │
    │ date_key     │                    │            │
    │ full_date    │                    ▼            │
    └──────┬───────┐          fact_subscription_events│
           │       └──────────┬──────────────────────┤
           │                  │ event_id             │
           │                  │ subscription_id (FK) │
           │                  │ customer_id (FK)     │
           │                  │ event_date (FK→date) │
           │                  │ event_type           │
           │                  │ amount               │
           ▼                  └──────────────────────┘
    fact_orders
    (unchanged — one-time purchases)
```

#### Step 4: Explain the Rationale to the Interviewer

> "I separated subscription tracking into its own fact table because the grain is different — one row per recurring billing event, not per purchase. I kept `fact_orders` for one-time transactions. I added a `subscription_tier` column to `dim_customer` so we can segment customers across both streams. This is fully additive — no existing queries break."

---

### 6a. Interviewer Questions You'll Face

1. **"If you added a column to `dim_customer`, what happens to existing ETL jobs that SELECT * from that table?"**
   → They will pick up the new column. If a downstream consumer expects a fixed schema, this could break. Always communicate changes and, for production systems, prefer `SELECT col1, col2...` over `SELECT *`.

2. **"How would you handle a case where the subscription service needs to retroactively calculate MRR for the past year?"**
   → Backfill `fact_subscription_events` from whatever historical billing data exists. Derive the period from the subscription start date and billing cadence. Validate counts against finance reports.

3. **"When would you merge `dim_subscription` into `dim_product` instead of keeping it separate?"**
   → Only if subscriptions are truly just a product variant (same grain, same attributes, always sold the same way). But subscriptions have unique attributes (cadence, churn, renewal) and a different lifecycle — keeping them separate is the cleaner design.

4. **"Walk me through the migration plan for this change in a production environment."**
   → (1) Create new tables in staging first. (2) Backfill historical data. (3) Run dual-write for one billing cycle to validate parity. (4) Update downstream dashboards to include subscription data. (5) Communicate deprecation of any legacy fields.

---

### 6b. Questions YOU Should Ask the Interviewer

| Question | Why It Matters |
|---|---|
| *"How frequently does the business add new metrics or data sources?"* | Reveals how much your design must prioritize extensibility. |
| *"Who are the primary consumers of this data model — analysts, apps, external partners?"* | Determines whether you can afford breaking changes. |
| *"Is there an existing data catalog or schema governance process?"* | Shows if versioning and communication are already formalized. |
| *"What's the typical lag between a business request and when the data needs to be available?"* | Helps you justify the right trade-off between speed and thoroughness. |
| *"Are there SLAs on dashboard availability during schema changes?"* | If zero downtime is required, dual-write becomes mandatory. |

---

### 6c. Top-Up Questions (Senior-Level Probes)

1. **"How do you handle slow-changing dimension types (SCD 1 vs SCD 2) when the business rules around historical tracking change mid-flight?"**
   → Look for: awareness that switching from SCD 1 (overwrite) to SCD 2 (history tracking) requires reprocessing all historical rows and updating every downstream query that assumed a single current value.

2. **"Describe a time you had to migrate from a flat-table or denormalized model to a proper star schema. What was the biggest risk?"**
   → The risk is breaking every existing report. Senior candidates discuss phased migration: build the new model in parallel, validate every metric, switch consumers one by one.

3. **"How do you decide when to denormalize vs. keep a normalized star schema in a cloud data warehouse?"**
   → Cost of JOINs vs. storage. In modern DWs (Snowflake, BigQuery), denormalization for performance matters less than it used to — but wide tables can still cause query-planning issues. Discuss hybrid approaches: semi-additive facts, bridge tables.

4. **"What's your strategy for schema evolution in a lakehouse (Delta Lake / Iceberg / Hudi) vs. a traditional RDBMS?"**
   → Lakehouse formats support schema evolution natively (add columns, evolve types). Discuss `evolve schema` commands, schema compatibility modes, and how backfilling works differently when you can't use simple `ALTER TABLE`.

---

### 7. Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Better Approach |
|---|---|---|
| **Constantly rebuilding the model from scratch** | Wastes time; loses trust; breaks everything downstream | Evolve incrementally; use migration scripts |
| **No versioning** | You can't roll back; can't audit what changed | Use Flyway/Liquibase; version the migration scripts |
| **Breaking downstream queries without notice** | BI dashboards go dark; stakeholders panic | Communicate ahead; dual-write; deprecate slowly |
| **Adding every new metric to the same fact table** | Table grows wide; unrelated columns bloat; grain drifts | New process → new fact table; respect grain discipline |
| **Hard-coding assumptions in ETL** | `WHERE status = 'active'` breaks when a new status appears | Parameterize ETL; use lookup tables; design for unknowns |
| **Ignoring backfill** | New columns are NULL for old data; analysts get wrong aggregates | Always backfill; document which rows are populated |

---

### 8. Practice Exercise

**Scenario:** You built a data model for a ride-sharing platform with `fact_trips`, `dim_driver`, `dim_rider`, `dim_location`, `dim_date`. Now the business launches **carpooling** — where one driver can carry multiple riders on the same trip, and each rider pays a share of the fare.

**Your task:**

1. **Analyze the impact** — What changes to the existing model are needed?
2. **Propose new or modified tables** — Sketch the updated schema.
3. **Decide the grain** of the new carpool fact — one row per trip, or one row per rider per trip? Justify.
4. **Write a migration plan** — How would you introduce this in production?
5. **Identify one anti-pattern** your design avoids.

<details>
<summary>💡 Click for a sample answer outline</summary>

1. **Impact:** The existing `fact_trips` (one row per trip) cannot express per-rider fare shares. Carsepool introduces a many-to-many relationship between trips and riders.

2. **New objects:**
   - `dim_carpool_route` — reusable route (origin, destination, typical stops)
   - `fact_carpool_trips` — one row per trip, includes driver, route, total fare, capacity used
   - `fact_carpool_rider_events` — one row per rider per trip, includes fare_share, seat_assigned

3. **Grain choice:** Two fact tables because the grain diverges. Trip-level metrics (total revenue, utilization) go in `fact_carpool_trips`. Rider-level metrics (individual charge, wait time) go in `fact_carpool_rider_events`. This avoids repeating rider-specific data N times in a single-trip row and avoids losing trip-level aggregates.

4. **Migration:** Create new tables in staging → backfill from existing carpool pilot data → dual-write for one week → update dashboards → communicate to consumers.

5. **Anti-pattern avoided:** Throwing rider-level data into the existing `fact_trips` and using repeating groups or semi-colon-delimited rider IDs — that would violate 1NF and break all aggregation queries.
</details>

---

### Quick Recap

- **Models always change** — design for additive evolution, not rebuilds.
- **New metric at existing grain** → add a column. **New process or grain** → new fact table.
- **Communicate every change** — downstream consumers are your stakeholders.
- **Version your migrations** — dual-write when downtime is not an option.
- **Avoid the anti-patterns** — especially mixing grains and silent schema breaks.

> *"A good data model is like a good city plan: you can add new neighborhoods and roads without destroying what already works."*

---

### Related Lessons

- [Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md)
- [Introduction to Data Modeling Questions](./01-introduction-to-data-modeling-questions.md)
- [Data Modeling Fundamentals](./04-data-modeling-fundamentals.md)
- [Practice: E-Commerce Platform](./15-practice-ecommerce-platform.md)
