# Lesson 12: End-to-End Walkthrough — Business Requirements Gathering

> **Principal Mentor Note:** Let's put Lessons 5–11 into action in a realistic, end-to-end interview simulation. Watch how a Principal candidate takes a vague, 1-sentence prompt, runs a structured **6-minute discovery conversation**, translates messy business rules into crisp technical decisions, and produces a **Requirements-to-Schema Blueprint** before drawing a single DDL statement.

---

## 1. The Interview Prompt

> **Interviewer:** *"We operate a B2B/B2C hybrid food and grocery delivery marketplace (similar to Uber Eats or DoorDash). Leadership complains that Operations, Finance, and Marketing all report different numbers for daily orders, revenue, and delivery times. Design the core analytical data warehouse to become our single source of truth."*

---

## 2. The 6-Minute Discovery Conversation (Annotated)

### Round 1: Consumers, Core Processes & Metric Definitions (Minutes 0–2)
* **Candidate:** *"Before jumping into tables, I'd like to align on scope and definitions across those three teams:*
  1. *What are the primary business processes we need to cover in v1—is it the end-to-end order lifecycle from checkout through courier delivery and financial settlement, or also upstream restaurant search/impressions?*
  2. *You mentioned Operations, Finance, and Marketing report different numbers. Are those discrepancies driven by **order state definitions** (e.g., placed vs. delivered vs. refunded orders), **timezone cutoffs** (UTC vs. local restaurant city time), or **attribution rules**?*
  3. *For revenue, do we need to track gross merchandise value (GMV), platform net take-rate, courier payouts, and partial refunds/promo credits?"*

* **Interviewer:** *"Let's focus on checkout through delivery and financial settlement—skip search impressions for v1. Yes, the discrepancies come from all three: Marketing counts placed orders in UTC, Operations counts delivered orders in local city time, and Finance excludes canceled orders and subtracts refunds on the refund date. We definitely need GMV, net revenue, courier payouts, and partial refunds."*

> **Mentor Takeaway:** Look how much ground was covered in one turn! The candidate immediately identified **why** numbers diverge (lifecycle state + timezone + refund timing) and got explicit confirmation.

---

### Round 2: Edge Cases, History (SCD) & Granularity (Minutes 2–4)
* **Candidate:** *"That clarifies the core tension. Let me check three structural edge cases:*
  1. *Can a single customer order be fulfilled by **multiple couriers** (e.g., courier reassignment mid-trip) or batched where one courier delivers **two orders in a single trip**?*
  2. *When a merchant's commission rate or a courier's tier changes, does Finance need historical orders reported under the **commission rate active at the time of the order** (`as-was`), while Sales Ops wants current account ownership (`as-is`)?*
  3. *Do we need to capture granular milestone timestamps (`ordered_at`, `merchant_accepted_at`, `courier_assigned_at`, `picked_up_at`, `delivered_at`) to diagnose where delivery delays happen?"*

* **Interviewer:** *"Yes to all three: orders get reassigned to backup couriers and couriers batch orders; Finance strictly requires historical commission rates at order time; and Operations desperately needs milestone bottleneck durations."*

---

### Round 3: Scale, Latency & Compliance (Minutes 4–6)
* **Candidate:** *"Last quick check on physical constraints:*
  1. *What scale are we designing for—say, 5M to 20M orders/day globally across multiple currencies?*
  2. *What freshness SLA do Operations and Finance expect—hourly micro-batch for Ops control towers, and daily closed snapshots for Finance?*
  3. *Are there GDPR/CCPA erasure requirements for customer and courier PII, and how many years of history must we retain?"*

* **Interviewer:** *"Assume 10M orders/day globally in 25 currencies, 15-minute freshness for Operations dashboards, daily close for Finance, 5 years of analytical retention, and strict GDPR right-to-be-forgotten compliance."*

---

## 3. Synthesizing Requirements into the Architectural Blueprint (Minute 6)

Before writing any SQL or drawing ERD boxes, state your **Design Contract** out loud:

```
================================================================================
REQUIREMENTS-TO-SCHEMA TRANSLATION CONTRACT
================================================================================
1. SCALE & STORAGE:
   - 10M orders/day (~3.65B orders/year; ~18.25B orders over 5 years).
   - ~25M order line items/day; ~12M courier delivery legs/day.
   - Format: Apache Iceberg / Snowflake partitioned by DATE(ordered_at_utc),
     clustered by (city_id, merchant_id).

2. MULTI-GRAIN FACT ARCHITECTURE (Zero Mixed Grains):
   - fct_orders (Transaction Fact | Grain: 1 row per customer order_id)
     -> Solves GMV, order-level fees, promos, multi-currency (local + USD),
        and BOTH ordered_at_utc AND ordered_at_local (restaurant timezone).
   - fct_order_line_items (Transaction Fact | Grain: 1 row per order_id + item_seq)
     -> Solves menu item / category performance without duplicating delivery fees.
   - fct_delivery_legs (Transaction Fact | Grain: 1 row per courier assignment leg)
     -> Solves courier reassignments, batched trips, distance, and courier payouts.
   - fct_order_lifecycle (Accumulating Snapshot | Grain: 1 row per order_id)
     -> Solves Operations milestone bottlenecks (placed -> accepted -> picked_up -> delivered).
   - fct_financial_adjustments (Transaction Fact | Grain: 1 row per refund/chargeback/tip_adj)
     -> Solves Finance auditability: closed periods stay immutable; refunds record
        on refund_date with FK back to original order_id.

3. TEMPORAL & SCD STRATEGY:
   - dim_merchant: SCD Type 2 for commission_rate_pct & merchant_tier (preserves
     Finance 'as-was' truth), plus Type 1 current_account_owner for Sales Ops.
   - dim_date & dim_time: Conformed role-playing dimensions supporting both UTC
     and Local City Time comparisons cleanly.

4. PRIVACY & COMPLIANCE (GDPR):
   - PII Vault (dim_user_pii & dim_courier_pii) isolated via surrogate keys +
     per-user crypto-shredding so GDPR erasure never rewrites 18B fact rows.
================================================================================
```

---

## 4. How This Blueprint Eliminates the Original Stakeholder Conflicts

| Team | Original Complaint | How Our Schema Solves It Cleanly |
| :--- | :--- | :--- |
| **Marketing** | Counted placed orders in UTC; numbers shifted when orders later canceled. | Queries `fct_orders` filtering `is_placed = TRUE` grouped by `ordered_date_utc_sk` (or net of cancellations via `order_status`). |
| **Operations** | Counted delivered orders in local city time; couldn't find delay bottlenecks. | Queries `fct_order_lifecycle` filtering `delivered_at_local IS NOT NULL` grouped by `delivered_date_local_sk`, using pre-computed `accept_to_pickup_sec` lags. |
| **Finance** | Needed immutable monthly close with refunds booked on refund date and historical commission rates. | Queries `fct_orders` (joined via historical `merchant_sk` for exact commission rate) UNIONED with `fct_financial_adjustments` grouped by `settlement_date_utc_sk`. |

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Your Operations team wants 15-minute freshness on `fct_order_lifecycle`, which is an Accumulating Snapshot updated via `MERGE`. How do you keep `MERGE` costs low at 10M orders/day?"**
   * **Expected Insight:** 99.5% of food delivery orders complete within **3 hours** of placement. Therefore, the 15-minute `MERGE` job only scans and merges target partitions where `ordered_at_utc >= CURRENT_TIMESTAMP - INTERVAL '6 hours'` (plus a small late-arrival catch-up partition), using Iceberg Merge-on-Read deletion vectors. Yesterday's partitions are never touched again by the 15-minute job!
2. **Top-Up 2: "What if a courier delivers a batched order containing 3 customer orders in a single trip (`batch_trip_id`), and you need to allocate the courier's base drive time and distance across those 3 orders?"**
   * **Expected Insight:** Keep `batch_trip_id` as a degenerate dimension on `fct_delivery_legs`, store actual unallocated trip metrics in `fct_courier_batches` (grain: 1 row per `batch_trip_id`), and if order-level unit economics require allocated courier cost, allocate proportionally by `straight_line_distance` or `order_gmv` into a clearly named `allocated_courier_cost_usd` column.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Diagnoses why stakeholders disagree** (state, timezone, refund timing) instead of assuming one team is "wrong". | Picks one team's definition arbitrarily and forces the other two teams to adapt. |
| **Produces a written Requirements-to-Schema Contract** at minute 6 before drawing tables. | Jumps straight from vague Q&A into DDL without synthesizing constraints. |
| **Designs for both UTC and Local City Time** natively in the fact/date model. | Stores only UTC timestamps and leaves every analyst to write error-prone `CONVERT_TIMEZONE()` joins. |