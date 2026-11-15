# Module 5: Fact Data Modeling
## Lesson 5.1 — Fact Table Types (Transactional, Periodic Snapshot, Accumulating Snapshot)

---

### 1. Why Fact Tables Matter

In dimensional modeling, **fact tables are the core measure container**. Every business process generates events — sales, shipments, returns, clicks, account openings — and fact tables capture the numeric "measures" produced by those events.

| | Fact Table | Dimension Table |
|---|---|---|
| **Purpose** | Store measurable business metrics | Store descriptive context |
| **Contents** | Numeric measures + foreign keys | Text/attributes + surrogate key |
| **Rows** | Millions or billions (grows fast) | Hundreds to thousands (grows slowly) |
| **Nature** | Mostly additive (can sum across dimensions) | Rarely additive |
| **Updates** | Append-heavy (mostly INSERTs) | Slowly-changing (occasional UPDATEs) |
| **Grain** | Explicitly defined per table | One row per entity |

> **Interview insight:** "A fact without a grain is a mistake in progress." Always define the grain first.

**The three fundamental fact table types** handle different measurement needs. Choosing the wrong one produces slow queries, wasted storage, or analytical blind spots.

---

### 2. Transactional Fact Tables

**Definition:** One row per transaction event — every measurable event gets its own row.

**When to use:** Events that happen **sporadically** — an order is placed, a click is recorded, a call is logged, a payment is processed.

**Example: `fact_orders`**

| order_key | date_key | customer_key | product_key | quantity | unit_price | total_amount |
|---|---|---|---|---|---|---|
| 1001 | 2025-01-05 | 42 | 7 | 2 | 19.99 | 39.98 |
| 1002 | 2025-01-05 | 15 | 3 | 1 | 49.99 | 49.99 |
| 1003 | 2025-01-06 | 42 | 7 | 1 | 19.99 | 19.99 |

**Grain:** One row = one order (or one order line item).

**Pros:**
- ✅ **Most granular** — you can drill down to individual events
- ✅ **Most flexible** — supports any aggregation (SUM, COUNT, AVG, MIN, MAX)
- ✅ **Fully additive** — measures can be summed across any dimension
- ✅ **Best for trend analysis** over time

**Cons:**
- ❌ **Large row counts** — can grow into billions quickly
- ❌ **Multiple records per entity over time** — finding the "latest" requires a window function or subquery
- ❌ **No snapshot of state** — cannot easily answer "what was the total inventory at midnight?"

**SQL example:**
```sql
-- Total revenue by product category for last quarter
SELECT
    dp.category_name,
    SUM(fo.total_amount) AS total_revenue,
    COUNT(DISTINCT fo.order_key) AS distinct_orders
FROM fact_orders fo
JOIN dim_product dp ON fo.product_key = dp.product_key
JOIN dim_date dd ON fo.date_key = dd.date_key
WHERE dd.quarter = '2025-Q1'
GROUP BY dp.category_name
ORDER BY total_revenue DESC;
```

#### ❓ Interviewer Questions You'll Face

| Question | Model Answer |
|---|---|
| "What is the grain of a transactional fact table?" | "One row per single transaction event — the lowest level detail captured by the source system, such as one order line item or one payment." |
| "Why are transactional facts considered most flexible?" | "Because they preserve the raw event data, so you can aggregate measures in any combination of dimensions — daily, weekly, by customer, by product, etc. Periodic snapshots lose intra-period detail; accumulating snapshots hard-code milestone columns." |
| "When would you NOT use a transactional fact table?" | "When you need to query the *state* of an entity at a given time — for example, the end-of-day account balance — rather than the events that changed that state. A transactional fact of deposits and withdrawals can't easily tell you the balance at midnight." |

#### 💬 Questions YOU Should Ask the Interviewer

| Question | Why It Matters |
|---|---|
| "How many raw events per day does the source system generate?" | Determines storage, partitioning, and whether a transactional fact is viable or if you need aggregation/rollups |
| "Do business users need to see individual transactions, or only daily aggregates?" | If only aggregates, you could pre-aggregate into a periodic snapshot as a performance layer |
| "What is the expected data retention period?" | Transactional facts grow large; older data may need to be purged, rolled up, or moved to cold storage |

#### 🔥 Top-Up Questions (Senior-Level Probes)

1. **"How would you handle a transactional fact table that grows beyond 10 billion rows per month?"**
   *Model answer:* Partition by date (monthly), use columnar compression, consider pre-aggregated rollup tables (e.g., `fact_orders_daily`), and migrate older partitions to cheaper storage tiers (S3/Blob + external table).

2. **"What happens when the source system has late-arriving transactions (e.g., yesterday's click reported today)?"**
   *Model answer:* Design for idempotent reloads using merge/upsert logic. Use a "batch_id" or "last_updated" column for incremental processing. The fact table must handle updates to the same event without duplicating.

3. **"Can a transactional fact table contain non-additive measures?"**
   *Yes — e.g., unit_price is typically non-additive (you don't sum prices across products), but it is semi-additive (it can be averaged). Non-additive measures are fine in transactional facts as long as the grain is clear.*

---

### 3. Periodic Snapshot Fact Tables

**Definition:** One row per time period per entity — a regular "photograph" of cumulative or state-based values.

**When to use:** Regular snapshots of **cumulative or state values** — account balances, inventory levels, employee headcount, sensor readings at fixed intervals.

**Example: `fact_account_balances`**

| account_key | date_key | balance | interest_ytd | minimum_balance_mtd |
|---|---|---|---|---|
| 101 | 2025-01-01 | 5,000.00 | 12.50 | 4,800.00 |
| 101 | 2025-01-02 | 5,200.00 | 13.10 | 5,000.00 |
| 102 | 2025-01-01 | 15,000.00 | 45.00 | 14,500.00 |

**Grain:** One row = one account per day (or per week, per month).

**Pros:**
- ✅ **Captures state at a point in time** — perfect for "what was the balance at midnight?"
- ✅ **Good for balances, levels, and thresholds** that change frequently
- ✅ **Simpler queries** for point-in-time analysis — no window functions needed
- ✅ **Predictable row count** (entities × periods), not dependent on event volume

**Cons:**
- ❌ **Redundant storage** — if nothing changed, you still store a row (or need a "no-change" skip policy)
- ❌ **Can't see intra-period events** — if you snapshot daily, you miss hourly fluctuations
- ❌ **Measures are often semi-additive** — summing "balance" across time makes no sense (you can sum across accounts, not days)

**SQL example:**
```sql
-- Accounts with balance below $100 as of Jan 15, 2025
SELECT
    da.account_id,
    da.account_type,
    fab.balance
FROM fact_account_balances fab
JOIN dim_account da ON fab.account_key = da.account_key
WHERE fab.date_key = '2025-01-15'
  AND fab.balance < 100.00
ORDER BY fab.balance;
```

> **Semi-additive measure reminder:** Balance is additive across accounts (sum balances to get total assets) but NOT across time (you don't add Monday's balance to Tuesday's to get $X). Your SQL must filter to one date.

#### ❓ Interviewer Questions You'll Face

| Question | Model Answer |
|---|---|
| "When would you use a periodic snapshot over a transactional fact?" | "When the business question is about the *state* of something at a point in time, not the events that changed it. Account balances, inventory levels, and headcount are classic cases — trying to derive these from transactional events is expensive, error-prone, and requires complex window functions." |
| "What grain would you choose for a sales target fact table?" | "One row per sales rep per month — a periodic snapshot that stores the monthly quota and actual-to-date. This is a natural periodic snapshot because targets are set per period, not per transaction." |
| "How do you handle the 'nothing changed' redundancy?" | "Two valid approaches: (1) store every period anyway — the storage cost is usually negligible — for simplicity; (2) use a 'no-change skip' policy and fill gaps at query time with `LAG()` — but this adds query complexity and can confuse business tools that expect a row per period." |

#### 💬 Questions YOU Should Ask the Interviewer

| Question | Why It Matters |
|---|---|
| "What is the natural snapshot period — daily, weekly, monthly?" | Determines row count and query latency trade-offs; daily is common for most banking/inventory use cases |
| "Do business users ever need intra-period detail?" | If yes, you may need both a transactional fact (for detail) and a periodic snapshot (for point-in-time queries) |
| "How long do we retain full-detail transactional data?" | If retention is short, the periodic snapshot becomes the only historical record — you must store it carefully |

#### 🔥 Top-Up Questions (Senior-Level Probes)

1. **"Design a periodic snapshot for an inventory system with 100,000 SKUs, snapshotted hourly. What's the storage impact?"**
   *Model answer: 100K SKUs × 24 hours × 365 days = ~876M rows/year. With ~8 columns, this could be 50-80 GB/year compressed in a columnar store. Partitioning by month and using clustered columnstore indexing is essential. Consider lower-frequency snapshots (daily) for older data via a tiered retention policy.*

2. **"Can a periodic snapshot fact table have a degenerate dimension?"**
   *Yes. Example: a `batch_id` for a daily ETL run that applies to all rows in a snapshot. It's a degenerate dimension because it has no corresponding dimension table — it's just an ID stored in the fact.*

3. **"How would you detect and handle gaps in a periodic snapshot (e.g., a day missing due to ETL failure)?"**
   *Model answer: Build a date dimension cross-join skeleton to guarantee one row per entity per period. After the snapshot load, run a reconciliation query to find missing periods. Backfill with the last-known value or NULL, and document the gap.*

---

### 4. Accumulating Snapshot Fact Tables

**Definition:** One row per entity **throughout its lifecycle**, updated as milestones occur — columns are added or populated as the process advances.

**When to use:** Processes with a **finite, well-defined set of milestones** — order fulfillment (ordered→shipped→delivered→returned), loan application (submitted→approved→funded→paid), claim processing (filed→reviewed→settled).

**Example: `fact_order_fulfillment`**

| order_key | customer_key | ordered_date_key | paid_date_key | shipped_date_key | delivered_date_key | order_amount | shipping_cost | status |
|---|---|---|---|---|---|---|---|---|
| 1001 | 42 | 2025-01-05 | 2025-01-05 | 2025-01-06 | 2025-01-08 | 39.98 | 5.99 | Delivered |
| 1002 | 15 | 2025-01-05 | 2025-01-07 | NULL | NULL | 49.99 | 0.00 | Pending Shipment |

**Grain:** One row = one entity instance (one order, one loan, one claim) for its entire life.

**Pros:**
- ✅ **Easy lifecycle analysis** — all milestones are in one row
- ✅ **Excellent for tracking durations** between milestones (time to ship, time to deliver)
- ✅ **Single-row query** — fast, no joins needed to get the full picture
- ✅ **Good for funnel analysis** — how many orders are stuck at each stage?

**Cons:**
- ❌ **Requires updates** — the row must be updated as milestones complete (not append-only)
- ❌ **Only works for processes with a definite endpoint** — open-ended processes don't fit
- ❌ **Schema is rigid** — adding a new milestone (e.g., "out_for_delivery_date_key") requires an ALTER TABLE and backfill
- ❌ **Multiple milestones per stage** (e.g., multiple shipment attempts) requires extra design (outrigger or separate fact)

**SQL example:**
```sql
-- Average time from order to delivery, by month
SELECT
    dd.year_month,
    AVG(
        DATEDIFF(day, fof.ordered_date_key, fof.delivered_date_key)
    ) AS avg_delivery_days
FROM fact_order_fulfillment fof
JOIN dim_date dd ON fof.ordered_date_key = dd.date_key
WHERE fof.delivered_date_key IS NOT NULL  -- exclude in-flight orders
GROUP BY dd.year_month
ORDER BY dd.year_month;
```

> **Key design rule for accumulating snapshots:** Every milestone column (date or key) is nullable and gets populated only when the milestone occurs. The **status** column (current step) enables filtering on in-flight rows.

#### ❓ Interviewer Questions You'll Face

| Question | Model Answer |
|---|---|
| "Explain accumulating snapshot with an example." | "An accumulating snapshot tracks a single entity through a defined series of milestones, with one row per entity that is updated as events happen. For example, an order fulfillment fact: one row per order with columns for ordered date, paid date, shipped date, and delivered date. The row starts with only ordered_date populated and gets progressively updated as the order moves through the pipeline." |
| "What happens if an order is split into multiple shipments?" | "You have two options: (1) keep one row per order with the first and last shipment dates, losing per-shipment detail; or (2) change the grain to one row per order-shipment pair, making it a transactional fact with a 'shipment' dimension. The choice depends on whether the business cares about individual shipments." |
| "How do you handle late-arriving milestones (e.g., a return that comes in months after delivery)?" | "The accumulating snapshot row must be updated again — this is acceptable as long as the ETL process can handle late data correctly. You add the return_date_key and a return_status. Consider also keeping an audit trail (change data capture) if the business needs to track historical versions of the snapshot." |

#### 💬 Questions YOU Should Ask the Interviewer

| Question | Why It Matters |
|---|---|
| "How many distinct milestones does this process have?" | Accumulating snapshots become unwieldy beyond ~10 milestones; at that point, consider a transactional fact with a milestone dimension |
| "Can a milestone happen multiple times?" | If yes (e.g., multiple reshipments), you may need a more flexible design — the single-row accumulating snapshot doesn't repeat well |
| "Do we need to track *when* the milestone dates changed (historical versions of the snapshot)?" | If yes, you may need SCD Type 2 on the accumulating snapshot itself, or keep a separate audit table |

#### 🔥 Top-Up Questions (Senior-Level Probes)

1. **"Design an accumulating snapshot for a loan application process with 8 stages. How would you handle a loan that is 're-aged' (sent back to an earlier stage)?"**
   *Model answer: Use a pattern of "stage_started_date" and "stage_completed_date" columns for each stage, plus a stage_key pointing to a dim_process_stage dimension. When a loan is re-aged, you NULL out the completed_date for the current stage and set the started_date for the previous stage. The status column reflects the current active stage. For audit, maintain a process_history table tracking every stage transition.*

2. **"What is the 'outrigger' pattern for accumulating snapshots, and when would you use it?"**
   *An outrigger is a dimension table that attaches to a fact table (not to another dimension). For example, dim_shipment_attempt would be an outrigger joined to fact_order_fulfillment to capture multiple delivery attempts. Use outriggers when a 1:many relationship exists between the core entity and a set of repeated milestones.*

3. **"Can an accumulating snapshot fact table contain measure columns that are computed from its own date columns?"**
   *Absolutely — and it's a best practice. Pre-compute `days_to_ship = DATEDIFF(day, ordered_date, shipped_date)` as a stored column. It makes queries faster and lets business users avoid date arithmetic. Just keep the raw date columns too for flexibility.*

---

### 5. Choosing the Right Fact Table Type

**Decision matrix:**

| Scenario | Business Question | Recommended Fact Type |
|---|---|---|
| An order is placed | "How much revenue did we generate yesterday?" | **Transactional** |
| Account balance at end of day | "Which accounts went below $100 this month?" | **Periodic Snapshot** |
| Order lifecycle | "How long does it take from order to delivery?" | **Accumulating Snapshot** |
| Website click | "How many users clicked the checkout button?" | **Transactional** |
| Inventory at midnight | "What is the current stock level for SKU-123?" | **Periodic Snapshot** |
| Loan application pipeline | "How many applications are stuck at 'Document Review'?" | **Accumulating Snapshot** |

**Real-world e-commerce platform uses all three:**

| Fact Table | Type | Purpose | Grain |
|---|---|---|---|
| `fact_orders` | Transactional | Each purchase event | One row per order line item |
| `fact_daily_inventory` | Periodic Snapshot | End-of-day stock levels | One row per SKU per day |
| `fact_order_fulfillment` | Accumulating Snapshot | Order lifecycle tracking | One row per order, updated as milestones happen |

> **Senior architect tip:** It's completely normal (and recommended) to have all three fact types for the same business process. Transactional facts give you raw event analysis, periodic snapshots give you point-in-time state, and accumulating snapshots give you cycle/funnel analysis. They complement each other.

#### ❓ Interviewer Questions You'll Face

| Question | Model Answer |
|---|---|
| "How would you decide between a transactional fact and a periodic snapshot?" | "Ask: Does the business need to know *what happened* (events) or *what the state was* (snapshot)? If you need to analyze patterns in individual events, use transactional. If you need to report balances at specific cut-off times, use periodic snapshot. Sometimes you need both." |
| "Can you use all three types in one data warehouse?" | "Yes — this is standard practice for complex business processes. A single e-commerce platform may have fact_orders (transactional), fact_daily_inventory (periodic), and fact_order_fulfillment (accumulating) co-existing in the same star schema and sharing conformed dimensions." |

#### 💬 Questions YOU Should Ask the Interviewer

| Question | Why It Matters |
|---|---|
| "What existing reporting queries run slowest today?" | Identifies which fact type would deliver the most immediate value |
| "Is the business more focused on event analysis, state analysis, or process analysis?" | Determines the priority fact type to build first |
| "How frequently do new milestones get added to the business process?" | Frequent additions suggest accumulating snapshots may need periodic redesign |

#### 🔥 Top-Up Questions (Senior-Level Probes)

1. **"When would you choose a periodic snapshot over an accumulating snapshot for order tracking?"**
   *Model answer: If orders can go through an unbounded or highly variable number of stages (e.g., a custom manufacturing process with review loops), accumulating snapshots become unwieldy. A periodic snapshot of "order status by day" would be more maintainable.*

2. **"You have all three fact types for the same process. A user reports that numbers from fact_orders don't match fact_order_fulfillment. How do you troubleshoot?"**
   *Model answer: Check grain differences (transactional = per line item vs. accumulating = per order), filter conditions (incomplete updates in the accumulating snapshot), and timing (transactional may capture events in real-time while the accumulating snapshot runs nightly). Data governance — define the "source of truth" per metric.*

---

### 6. Practice Exercise

**Build the fact table schema for a **ridesharing platform** (Uber/Lyft-style).**

The platform has:
- **Trips** — each ride has a pickup and drop-off, distance, fare, driver and rider info
- **Driver earnings** — drivers are paid weekly; their daily balance is tracked
- **Driver shift lifecycle** — drivers sign on, accept trips, go offline; the goal is to measure average time from sign-on to first trip

**Your task:**

1. Propose **one transactional fact table** — define the grain, list 3-5 measures, and write a `CREATE TABLE` SQL statement
2. Propose **one periodic snapshot fact table** — define the grain, period, and list 3-5 measures
3. Propose **one accumulating snapshot fact table** — define the milestones/columns
4. For each fact table, write one query that would answer a realistic business question
5. Explain *why* you chose that fact type for each scenario

> **Try it before checking the sample solution** (at the end of this module). The best way to prepare for a data modeling interview is to design schemas from scratch, on a whiteboard.

---

### Quick Recap

| Fact Type | Grain | Row Behavior | Best For |
|---|---|---|---|
| **Transactional** | One row per event | Append-only | Event analysis, additive measures |
| **Periodic Snapshot** | One row per entity per period | Append-only, batch load | Point-in-time state, semi-additive measures |
| **Accumulating Snapshot** | One row per entity lifecycle | Updated | Process/funnel tracking, duration analysis |

> **The best data modelers don't just know the three types — they know when to use each, when to combine them, and when to push back on requirements that don't fit.**
---

### Related Lessons

- [Practice: E-Commerce Platform](./15-practice-ecommerce-platform.md)
- [Additive, Semi-Additive, Non-Additive Measures](./25-additive-semi-additive-non-additive-measures.md)
- [Multi-Fact Query Strategies](./26-multi-fact-query-strategies.md)
- [Fact Modeling Practice](./27-fact-modeling-practice.md)
