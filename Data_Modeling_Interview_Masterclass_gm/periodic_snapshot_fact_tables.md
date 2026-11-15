# Lesson 25: Periodic Snapshot Fact Tables

> **Principal Mentor Note:** Transaction fact tables only record what *happened*. They are completely blind to what *did not happen* (zero-sale SKUs, idle drivers, inactive subscribers, end-of-day bank balances). Whenever an interviewer asks about **inventory levels, account balances, SaaS MRR/churn, driver utilization, or DAU/WAU/MAU retention**, you must immediately reach for a **Periodic Snapshot Fact Table**.

---

## 1. Core Definition & Architectural Properties

A **Periodic Snapshot Fact Table** samples the state of an entity or business process at **regular, predictable time intervals** (hourly, daily, weekly, or month-end).

| Architectural Dimension | Transaction Fact Table | Periodic Snapshot Fact Table |
| :--- | :--- | :--- |
| **Grain** | One row per discrete event (`event_id`). | **One row per `(entity_sk, period_sk)`** (e.g., 1 row per account per month-end, or 1 row per SKU/warehouse per day). |
| **Sparsity vs. Density** | **Sparse** (no event = no row). | **Dense** (every active entity gets a row for every period, even if zero transactions occurred during that period!). |
| **Measure Additivity** | **Fully Additive** (`SUM` across all dimensions including time). | **Semi-Additive** for stock/balance measures (`SUM` across entities, `LAST_VALUE`/`AVG` across time) + **Fully Additive** for period-flow rollups (`daily_deposits`, `daily_units_sold`). |
| **Primary Analytical Superpower** | Detailed event forensics, conversion funnels, individual transaction auditing. | **Stock-and-Flow accounting**, zero-activity detection (churn, stockouts, idle supply), and fast longitudinal time-series queries without scanning years of raw transactions. |

---

## 2. Production DDL Pattern: Fintech Account Daily Snapshot

```sql
-- ============================================================================
-- PERIODIC SNAPSHOT FACT: DAILY CUSTOMER ACCOUNT BALANCE & ACTIVITY
-- Grain: One row per active (account_sk, snapshot_date_sk)
-- ============================================================================
CREATE TABLE fct_account_daily_snapshot (
    snapshot_date_sk        INT NOT NULL,          -- FK -> dim_date (YYYYMMDD)
    account_sk              BIGINT NOT NULL,       -- Historical SCD2 SK as of snapshot_date
    account_id              VARCHAR(64) NOT NULL,  -- Natural key for longitudinal queries
    customer_sk             BIGINT NOT NULL,
    product_tier_sk         INT NOT NULL,          -- CHECKING, HIGH_YIELD_SAVINGS, BUSINESS_PRO
    country_code            CHAR(2) NOT NULL,
    currency_code           CHAR(3) NOT NULL,
    -- 1. SEMI-ADDITIVE STOCK MEASURES (Point-in-Time Balances — NEVER SUM ACROSS DATES!)
    bod_balance_usd         DECIMAL(16,2) NOT NULL,-- Beginning-of-Day balance (= prior day EOD)
    eod_balance_usd         DECIMAL(16,2) NOT NULL,-- End-of-Day settled balance
    eod_pending_hold_usd    DECIMAL(16,2) NOT NULL,
    eod_available_credit_usd DECIMAL(16,2) NOT NULL,
    -- 2. FULLY ADDITIVE FLOW MEASURES (Period Activity Rollups for that single day)
    daily_inflow_usd        DECIMAL(16,2) NOT NULL,-- 0.00 if no deposits occurred
    daily_outflow_usd       DECIMAL(16,2) NOT NULL,-- 0.00 if no withdrawals occurred
    daily_net_change_usd    DECIMAL(16,2) NOT NULL,-- eod_balance - bod_balance
    daily_txn_count         INT NOT NULL,          -- 0 if dormant today!
    -- 3. LONGITUDINAL STATE & CHURN INDICATORS
    days_since_last_txn     INT NOT NULL,          -- Increments by +1 on zero-activity days!
    is_dormant_30d          BOOLEAN NOT NULL,      -- TRUE when days_since_last_txn >= 30
    PRIMARY KEY (snapshot_date_sk, account_sk)
)
PARTITION BY (snapshot_date_sk)
CLUSTER BY (account_id);
```

### Notice the Accounting Identity Built Into Every Row:
$$\text{bod\_balance\_usd} + \text{daily\_inflow\_usd} - \text{daily\_outflow\_usd} = \text{eod\_balance\_usd}$$
Building this invariant directly into the table allows automated **dbt / Great Expectations data quality tests** to verify 100% ledger reconciliation on every daily partition!

---

## 3. Solving the "Dense Table Explosion" Problem at Scale

**The Trap:** If you have `200M` registered users or `50M` SKUs $\times$ `200` warehouses (`10 billion` combinations) and generate a dense daily snapshot every day for 5 years (`18.25 trillion rows`), storage and compute costs will bankrupt the platform.

### Three Principal Techniques to Tame Snapshot Density
1. **Active-Lifecycle Bounding (Only Snapshot Active Entities):**
   * Do not generate daily rows before `account_opened_date` or after `account_closed_date` (emit one final `CLOSED` row with `eod_balance = 0` on the closure date, then stop).
2. **Sparse Delta / Run-Length Compaction (`valid_from_date` to `valid_to_date`):**
   * For entities whose state changes infrequently (e.g., warehouse inventory where 95% of SKUs have zero movement on a given day), store a **Compressed Daily Snapshot** `(valid_from_date_sk, valid_to_date_sk, warehouse_sk, product_sk, eod_on_hand_units)`—only writing a new row on days when inventory actually changes! Expose a thin SQL View joining to `dim_date` (`d.date_sk BETWEEN s.valid_from_date_sk AND s.valid_to_date_sk`) when analysts need dense daily series.
3. **Grain Coarsening Over Time (Daily $\rightarrow$ Monthly):**
   * Retain **Daily** snapshots for the trailing **90 days** (for operational volatility), and retain **Month-End** snapshots (`is_month_end = TRUE`) for **7+ years** (for multi-year YoY financial reporting).

---

## 4. Validation Query (Semi-Additive Balance & Dormancy Detection)

```sql
-- Query: Total AUM (Assets Under Management) & 30-Day Dormancy Rate at Month-End
SELECT
    d.year_month,
    a.country_code,
    COUNT(a.account_id)                                             AS total_active_accounts,
    -- Safe semi-additive aggregation: filtered to a single snapshot day per month!
    SUM(a.eod_balance_usd)                                          AS total_eod_aum_usd,
    SUM(CASE WHEN a.is_dormant_30d THEN 1 ELSE 0 END)               AS dormant_accounts,
    ROUND(
        SUM(CASE WHEN a.is_dormant_30d THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(a.account_id), 0),
    2)                                                              AS dormancy_rate_pct
FROM fct_account_daily_snapshot a
JOIN dim_date d
  ON a.snapshot_date_sk = d.date_sk
WHERE d.is_month_end = TRUE              -- Critical: prevents summing EOD balances across 30 days!
  AND d.calendar_year = 2026
GROUP BY 1, 2
ORDER BY 1, 4 DESC;
```

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "How do you build `fct_account_daily_snapshot` incrementally every night without re-scanning 5 years of `fct_payment_transactions`?"**
   * **Expected Insight:** Use **Recursive Incremental State Roll-Forward**: take yesterday's partition (`snapshot_date_sk = T - 1`) as the baseline `bod_balance_usd` and `days_since_last_txn`, `FULL OUTER JOIN` it with today's aggregated flows from `fct_payment_transactions` (`WHERE event_date_utc_sk = T`), add any newly opened accounts from `dim_account`, and write partition `T`. Compute cost is strictly $O(\text{Active Entities})$, independent of historical table size!
2. **Top-Up 2: "What if a late-arriving transaction from 3 days ago (`T - 3`) arrives today (`T`)? How does that impact your daily snapshots?"**
   * **Expected Insight:** Because `eod_balance` is cumulative, a `$100` deposit on `T - 3` shifts `eod_balance` on `T - 3`, `T - 2`, `T - 1`, and `T`. You must run a **cascading state recomputation** starting from the earliest backfilled partition (`T - 3` forward to `T`), which is fast when partitioned by `snapshot_date_sk` and bounded to a 7-to-14-day late-arrival window.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Combines semi-additive stock metrics (`eod_balance`) with additive period flows (`daily_inflow`, `daily_txn_count`)** and enforces the `BOD + Inflow - Outflow = EOD` invariant. | Tries to answer "How many accounts had $0 balance and 0 transactions last Tuesday?" using only a transaction table. |
| **Builds daily snapshots incrementally from `T - 1` + today's delta** rather than scanning all-time transaction history. | Recomputes cumulative balances from `SUM(transactions)` over 5 years every night. |
| **Solves dense explosion** via active-lifecycle bounding, run-length compression, or daily-to-monthly tiering. | Generates billions of zero-activity rows for closed accounts or discontinued SKUs forever. |