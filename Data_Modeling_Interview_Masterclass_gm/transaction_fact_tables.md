# Lesson 24: Transaction Fact Tables

> **Principal Mentor Note:** Transaction fact tables are the foundational workhorses of every analytical warehouse—representing **80%+ of data volume** in modern lakehouses. While any mid-level engineer can define a transaction fact table, Principal Data Engineers distinguish themselves by enforcing **strict single-grain purity**, handling **late-arriving facts and retroactive restatements** cleanly, designing **idempotent ingestion pipelines**, and choosing when to **consolidate multiple event types into a single fact table** versus splitting them.

---

## 1. Core Definition & Architectural Properties

A **Transaction Fact Table** records a discrete business event that occurs at a **specific point in space and time**.

| Architectural Dimension | Rule / Best Practice |
| :--- | :--- |
| **Grain** | **One row per atomic business event** (e.g., 1 order line item, 1 payment authorization attempt, 1 ad click, 1 bank ledger entry). |
| **Mutability** | **Immutable / Append-Only** once recorded. If a business event is reversed or corrected later (e.g., refund, void, chargeback), insert a **new compensating/reversing transaction row** rather than updating historical rows in-place. |
| **Sparsity** | **Naturally Sparse:** If an entity (customer, product, driver) has zero events on a given day, **zero rows** are inserted for that entity. |
| **Measure Additivity** | Almost exclusively **Fully Additive** (`SUM` across all dimensions: time, geo, customer, product). |
| **Date/Time Representation** | Always stores **both** a conformed `date_sk` (`YYYYMMDD` integer for partition pruning and fast joins) **and** a high-precision `occurred_at_utc` `TIMESTAMP` (plus `ingested_at_utc` for watermark auditing). |

---

## 2. Production DDL Pattern: Payment Ledger Transaction Fact

```sql
-- ============================================================================
-- TRANSACTION FACT: PAYMENT AUTHORIZATIONS & SETTLEMENTS
-- Grain: One row per discrete payment state event (payment_event_id)
-- ============================================================================
CREATE TABLE fct_payment_transactions (
    -- 1. Natural / Degenerate Event Keys
    payment_event_id        VARCHAR(64) NOT NULL,  -- Idempotency deduplication key
    payment_intent_id       VARCHAR(64) NOT NULL,  -- Degenerate Dimension (groups auth/capture/refund)
    order_id                VARCHAR(64) NOT NULL,  -- Degenerate Dimension
    -- 2. Temporal Keys (Event Time vs. Ingestion Time)
    event_date_utc_sk       INT NOT NULL,          -- FK -> dim_date (partition key)
    event_date_local_sk     INT NOT NULL,          -- FK -> dim_date (merchant local timezone)
    occurred_at_utc         TIMESTAMP NOT NULL,
    ingested_at_utc         TIMESTAMP NOT NULL,
    -- 3. Conformed Dimension Foreign Keys (Pre-resolved SCD2 SKs + Natural Keys)
    merchant_sk             BIGINT NOT NULL,       -- Historical SCD2 SK (captures fee rate at event time)
    merchant_id             VARCHAR(64) NOT NULL,  -- Durable natural key for 'as-is' current joins
    customer_sk             BIGINT NOT NULL,       -- -1 sentinel for guest checkout
    payment_method_sk       INT NOT NULL,          -- Junk/Lookup Dim (VISA, MASTERCARD, UPI, APPLE_PAY)
    geo_sk                  INT NOT NULL,
    -- 4. Event Classification
    event_type              VARCHAR(24) NOT NULL,  -- AUTH_SUCCEEDED, AUTH_FAILED, CAPTURED, REFUNDED, CHARGEBACK
    failure_reason_code     VARCHAR(32),           -- NULL on success; 'INSUFFICIENT_FUNDS', 'FRAUD_BLOCK'
    currency_code           CHAR(3) NOT NULL,
    fx_rate_to_usd          DECIMAL(18,8) NOT NULL,
    -- 5. Fully Additive Measures (Signed for ledger math + Unsigned gross for volume)
    gross_volume_usd        DECIMAL(14,2) NOT NULL,-- Always >= 0 (for total processed volume)
    net_cash_flow_usd       DECIMAL(14,2) NOT NULL,-- +Amount on CAPTURED, -Amount on REFUNDED/CHARGEBACK, 0 on AUTH
    interchange_fee_usd     DECIMAL(12,4) NOT NULL,
    platform_fee_usd        DECIMAL(12,4) NOT NULL,
    net_take_usd            DECIMAL(12,4) NOT NULL -- platform_fee - interchange_fee
)
PARTITION BY (event_date_utc_sk)
CLUSTER BY (merchant_id, event_type);
```

---

## 3. Consolidated vs. Separate Transaction Fact Tables

A classic Principal design question: *"Should we put `AUTH_ATTEMPT`, `CAPTURE`, `REFUND`, and `CHARGEBACK` into one unified `fct_payment_transactions` table, or into four separate tables?"*

### Decision Matrix

| Factor | Favor **Consolidated Fact Table** | Favor **Separate Fact Tables** |
| :--- | :--- | :--- |
| **Dimensional Overlap** | All event types share **>80% of the same dimensions** (`merchant`, `customer`, `payment_method`, `geo`, `date`). | Event types have completely different dimensions (e.g., `fct_warehouse_shipments` vs. `fct_marketing_ad_clicks`). |
| **Cross-Event KPIs** | Analysts frequently compute ratios across event types in one pass (e.g., `Capture Rate = Captures / Auths`, `Net Cash = Captures - Refunds`). | Event types are queried by completely separate teams with zero shared metrics. |
| **Column Sparsity** | Event-specific columns are few (`< 5` nullable columns like `failure_reason_code`). | Each event type has 30+ unique columns that would leave 85% of cells `NULL`. |
| **Volume Skew** | Event volumes are within 1–2 orders of magnitude. | One event type is **1,000x larger** than another (e.g., `50B ad_impressions` vs. `40M conversions`—keep separate so conversion queries don't scan 50B rows!). |

---

## 4. Handling Late-Arriving Facts & Idempotent Ingestion

In distributed event architectures (Kafka $\rightarrow$ Lakehouse), events routinely arrive out of order (e.g., a mobile payment queued offline at `23:57 UTC` on Nov 14 arrives at `02:15 UTC` on Nov 15).

### Three-Step Principal Protocol for Late-Arriving Facts
1. **Never Bucket by Ingestion Time:** Always assign `event_date_utc_sk` from `occurred_at_utc` (`20261114`), not `ingested_at_utc` (`20261115`).
2. **Point-in-Time Dimension Key Resolution:** Look up `merchant_sk` in `dim_merchant` using `occurred_at_utc >= valid_from_utc AND occurred_at_utc < valid_to_utc`—so even if the merchant's fee tier changed at `00:05 UTC` on Nov 15, the late Nov 14 transaction gets the Nov 14 `merchant_sk`!
3. **Idempotent Incremental Upsert (`MERGE`):** Deduplicate incoming micro-batches by `payment_event_id` (using `QUALIFY ROW_NUMBER() OVER (PARTITION BY payment_event_id ORDER BY ingested_at_utc ASC) = 1`) and `MERGE` into `fct_payment_transactions` bounded by the lookback window (`WHERE target.event_date_utc_sk >= 20261107`). If the orchestrator retries a failed job 5 times, `payment_event_id` guarantees zero duplicate rows.

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Suppose Finance closes the October accounting books on November 3rd. On November 10th, a late transaction with `occurred_at_utc = Oct 31` arrives. How do you prevent silent restatement of closed October reports while still capturing the transaction?"**
   * **Expected Insight:** Store both `event_date_utc_sk` (actual business date: `Oct 31`) and `accounting_date_utc_sk` (open ledger posting date: `Nov 10`, computed as `GREATEST(DATE(occurred_at_utc), current_open_accounting_period_start)`). Product & Ops slice by `event_date_utc_sk` for accurate cohort behavior, while Finance slices by `accounting_date_utc_sk` so closed October books never change!
2. **Top-Up 2: "When should you NEVER put `NULL` in a numeric measure column of a transaction fact table?"**
   * **Expected Insight:** Distinguish between **"Zero value"** (`0.00` discount applied $\rightarrow$ store `0.00` so `AVG()` and arithmetic expressions `gross - discount` don't evaluate to `NULL`!) and **"Not Applicable"** (`comment_char_length` on a `LIKE` event $\rightarrow$ store `NULL` so `AVG(comment_char_length)` averages only across actual comments rather than being diluted by millions of `0`s!).

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Models refunds, voids, and chargebacks as new signed ledger rows** rather than overwriting historical transactions. | Runs `UPDATE fct_payments SET status = 'REFUNDED', amount = 0` on historical partitions, destroying audit trails. |
| **Separates `event_date` from `accounting_date`** to solve Finance closed-period immutability alongside late-arriving data. | Overwrites closed prior-month financial partitions silently whenever late events arrive. |
| **Applies volume-skew and sparsity criteria** when deciding whether to consolidate multiple events into one fact table. | Dumps 50B impressions and 10M purchases into a single fact table, forcing purchase queries to scan 5,000x more rows. |