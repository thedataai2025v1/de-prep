# Lesson 38 — Mock Interview: Stripe-Style Payments Platform

**Level:** Senior / Staff / Principal Data Engineer
**Duration:** 60-90 minutes
**Case type:** End-to-end source-to-warehouse data modeling

## How to Use This

This is a full mock interview based on a real Stripe-like payments platform. It covers:
- Multi-currency transactions, balance tracking, and idempotency
- Disputes, refunds, and chargeback flows
- High-volume event stream modeling with exactly-once guarantees
- Bridge tables for multi-merchant-to-multi-category mappings

---

## Scenario

You are the lead data engineer for a global payments platform (like Stripe). Merchants across multiple countries process millions of payments daily in different currencies.

### Business Requirements

1. **Track all payment lifecycle events** — authorization, capture, settlement, refund, dispute
2. **Balance tracking** — per-merchant, per-currency running balances
3. **Dispute management** — track disputes, evidence, outcomes, chargebacks
4. **Multi-currency support** — FX rates at time of transaction, reporting in base currency
5. **Merchant analytics** — revenue by merchant, by region, by payment method
6. **Idempotency** — handle duplicate API calls; same payment can be submitted multiple times

### Key Entities (from business team)

- **Merchants** — businesses that accept payments (SCD Type 2 for plan changes)
- **Customers** — end-users who pay
- **Payment Methods** — credit cards, ACH, wallets (linked to customers)
- **Transactions** — individual payment events with lifecycle states
- **Balances** — running account balances per merchant per currency
- **Disputes** — chargebacks filed by card issuers
- **Refunds** — partial or full returns
- **FX Rates** — currency conversion rates (daily, per currency pair)

---

## Phase 1: Requirements Clarification (10 min)

### Questions to Ask the Interviewer

**Business / Scope:**
- What are the key metrics the business cares about most? (GMV, net revenue, dispute rate, settlement time?)
- How many currencies do we support? Do we need historical FX or just settlement FX?
- What is the expected volume? (e.g., 10M transactions/day?)
- How do merchants get paid? (daily settle, weekly, threshold-based?)
- What is the dispute resolution timeline?

**Technical:**
- What is the data source? (Postgres OLTP, Kafka, both?)
- What is the target warehouse/lakehouse? (Snowflake, BigQuery, Databricks?)
- What is the required latency? (near-real-time dashboard vs. T+1 batch)
- Do we need exactly-once processing for financial data?

---

## Phase 2: Source-to-Warehouse Modeling (20 min)

### Dimension Tables

| Dimension | Grain | Key Columns | SCD Type |
|-----------|-------|-------------|----------|
| `dim_merchant` | One row per merchant version | `merchant_id`, `name`, `country`, `currency`, `plan_tier`, `status` | Type 2 (plan/status changes) |
| `dim_customer` | One row per customer version | `customer_id`, `merchant_id`, `email`, `country` | Type 2 |
| `dim_payment_method` | One row per method | `payment_method_id`, `customer_id`, `type`, `last4`, `expiry` | Type 2 |
| `dim_fx_rate` | One row per currency pair per day | `date_key`, `from_currency`, `to_currency`, `rate` | Type 1 (overwrite latest) |
| `dim_date` | Standard date dimension | `date_key`, `day`, `month`, `quarter`, `year`, `day_of_week` | — |

### Fact Tables

| Fact Table | Grain | Key Columns |
|------------|-------|-------------|
| `fct_transactions` | One row per transaction event | `transaction_id`, `merchant_key`, `customer_key`, `payment_method_key`, `date_key`, `amount_original`, `currency`, `amount_usd`, `status`, `created_at`, `settled_at` |
| `fct_balance_changes` | One row per balance event | `balance_change_id`, `merchant_key`, `date_key`, `currency`, `change_amount`, `running_balance`, `change_type` |
| `fct_disputes` | One row per dispute | `dispute_id`, `transaction_key`, `merchant_key`, `date_key`, `dispute_reason`, `amount`, `status`, `outcome` |
| `fct_refunds` | One row per refund | `refund_id`, `transaction_key`, `merchant_key`, `date_key`, `amount`, `currency`, `reason` |

### Key Design Decisions

**1. Multi-Currency FX:**
- Store `amount_original` + `currency` in `fct_transactions`
- Join `dim_fx_rate` at settlement date to get `amount_usd` (or any base currency)
- Use the FX rate at the time of **settlement** (not authorization) since rates change

**2. Idempotency:**
- Source systems use idempotency keys; deduplicate in Bronze layer by `transaction_id`
- Use `MERGE` with `event_id` dedup in Silver

**3. Balance Tracking:**
- `fct_balance_changes` is a **cumulative fact table** (running balance at each change)
- Supports point-in-time balance lookups without re-aggregation

**4. Disputes as Separate Fact:**
- Disputes have a different lifecycle than transactions
- Linked to `fct_transactions` via `transaction_key`
- Can have multiple evidence events per dispute (use a satellite fact if needed)

---

## Phase 3: Advanced Questions (15 min)

### Q: How do you handle exactly-once processing for financial data?

**Answer (Principal Level):**
- **Transactional Outbox + CDC:** Each microservice writes business rows and an `outbox_events` row in the same ACID transaction. Debezium tails the WAL, publishes to Kafka, and the lakehouse `MERGE` deduplicates by deterministic `event_id` (UUIDv7). This gives end-to-end exactly-once semantics without distributed 2PC.

### Q: How do you handle refunds that partially reverse a transaction?

**Answer:**
- `fct_refunds` is a separate fact with its own grain
- A refund links to the original `transaction_key`
- Net revenue = `SUM(amount)` from transactions - `SUM(amount)` from refunds
- Partial refunds create separate refund rows; multiple refunds per transaction are allowed

### Q: How do you model merchants that span multiple categories?

**Answer:**
- Use a **bridge table** (`bridge_merchant_category`) between `dim_merchant` and a category dimension
- Many-to-many relationship: one merchant can have multiple categories; one category can have multiple merchants
- Fact tables join through the bridge for category-level rollups

### Q: What is the top aggregate table you would pre-build?

**Answer:**
```
agg_merchant_daily_revenue
- merchant_key, date_key
- total_gmv_usd, transaction_count, refund_total_usd, net_revenue_usd
- dispute_count, dispute_amount_usd
```
Pre-aggregate at merchant-day grain to avoid scanning billions of transaction rows for daily dashboards.

---

## Phase 4: Architecture & Optimization (10 min)

### Medallion Architecture

| Layer | Purpose | Key Operations |
|-------|---------|----------------|
| **Bronze** | Raw events from Kafka/CDC | `MERGE` by `event_id`, append-only |
| **Silver** | Conformed dimensions + transactions | SCD Type 2 for merchants, FX join, idempotent `MERGE` |
| **Gold** | Aggregates for dashboards | `agg_merchant_daily_revenue`, `agg_global_metrics` |

### Performance

- **Partition** `fct_transactions` by `date_key` (day-level)
- **Cluster** by `merchant_key` for merchant-scoped queries
- **Materialize** `agg_merchant_daily_revenue` with incremental refresh
- **Bloom filters** on `transaction_id` for point lookups

### Cost Controls

- `require_partition_filter = TRUE` on fact tables
- Bound `MERGE` to non-terminal rows only (`is_terminal_state = FALSE`)
- Daily `expire_snapshots` with 7-day retention for GDPR

---

## Evaluation Rubric

| Level | What They Show |
|-------|---------------|
| **Senior** | Correct star schema, identifies FX & idempotency issues, handles disputes as separate fact |
| **Staff** | Medallion layers, bridge tables for many-to-many, balance tracking as cumulative fact, cost controls |
| **Principal** | Exactly-once via outbox + CDC, explains non-algebraic MV pitfalls, FinOps quantification, GDPR retention |

---

## Related Lessons

- [Lesson 24 — Fact Table Types](./24-fact-table-types.md) — cumulative fact tables for balance tracking
- [Lesson 25 — Additive, Semi-Additive, Non-Additive Measures](./25-additive-semi-additive-non-additive-measures.md) — running balance is semi-additive
- [Lesson 28 — Date-Based Partitioning](./28-date-based-partitioning.md)
- [Lesson 32 — Bridge Tables and Factless Facts](./32-bridge-tables-and-factless-facts.md) — many-to-many merchant categories
- [Lesson 10 — Data Volume and Scalability](./10-data-volume-and-scalability-considerations.md)
