# Lesson 7: Analyzing Metrics (Additive, Semi-Additive, Non-Additive & Ratios)

> **Principal Mentor Note:** A data model exists to compute metrics accurately across arbitrary dimensions (`WHERE` / `GROUP BY`). If you put a non-additive ratio or a point-in-time balance into a fact table without understanding its mathematical behavior under `SUM()`, BI users will generate silent, catastrophic reporting errors. Principal candidates classify every metric by **additivity** and design schemas that make wrong aggregations hard to write.

---

## 1. The Three Classes of Fact Additivity

| Additivity Class | Mathematical Rule | Examples | Safe Aggregation Pattern |
| :--- | :--- | :--- | :--- |
| **1. Fully Additive** | Can be summed across **ALL** dimensions (Time, Geography, Product, Customer). | `gross_revenue_usd`, `quantity_sold`, `trip_distance_meters`, `ad_clicks` | `SUM(metric)` across any slice. |
| **2. Semi-Additive** | Can be summed across some dimensions (e.g., Warehouse, Product), but **NEVER across Time** (snapshot/balance metrics). | `inventory_on_hand`, `account_balance_usd`, `active_subscriber_count`, `cpu_allocated_cores` | `SUM()` across entities at a **single point in time**, then `AVG()`, `MIN()`, `MAX()`, or `LAST_VALUE()` across Time. |
| **3. Non-Additive** | Cannot be summed across **ANY** dimension. | Ratios (`take_rate`, `ctr`, `conversion_rate`), unit prices (`unit_price_usd`), distinct counts (`dau`, `mau`), percentages. | **Never store pre-computed ratios as facts to be summed.** Store the constituent **numerator and denominator** (`clicks`, `impressions`), sum them separately, and divide last: `SUM(clicks) / NULLIF(SUM(impressions), 0)`. |

---

## 2. Deep Dive: Avoiding the Top 4 Metric Modeling Pitfalls

### Pitfall 1: Averaging Averages (Simpson's Paradox in BI)
* **Scenario:** Store A has 1,000 orders with an Average Order Value (AOV) of `$20` (`$20,000` total). Store B has 10 orders with an AOV of `$100` (`$1,000` total).
* **Wrong Schema:** Storing `avg_order_value` in `agg_store_daily` and letting Tableau run `AVG(avg_order_value)` $\rightarrow$ `($20 + $100) / 2 = $60.00` (Wrong!).
* **Principal Schema:** Store **`total_order_revenue` (`$21,000`)** and **`order_count` (`1,010`)** in `agg_store_daily`. True AOV = `SUM(total_order_revenue) / SUM(order_count) = $20.79`.

### Pitfall 2: Distinct Counts (`DAU / WAU / MAU`) Across Rollups
* **Scenario:** A user opens the app on Monday, Wednesday, and Friday.
* **Trap:** Daily `DAU` is `1` on Mon, `1` on Wed, `1` on Fri. Summing `DAU` across the week gives `3`, when `WAU` is actually `1`.
* **Principal Solutions:**
  1. For interactive ad-hoc slicing up to ~1B rows: Keep user-grain daily activity (`fct_user_daily_activity` with `user_sk`, `activity_date_sk`) and run `COUNT(DISTINCT user_sk)` (or `APPROX_COUNT_DISTINCT` / HyperLogLog).
  2. For massive scale pre-aggregated rollups: Store **HyperLogLog (HLL) sketches** (`VARBINARY` / `HLL_SKETCH` in Snowflake/BigQuery/Spark) in summary tables so distinct user counts *become* algebraically mergeable (`HLL_COMBINE`) across days, regions, and platforms!

### Pitfall 3: Semi-Additive Balances Over Time
* **Scenario:** `fct_account_balances_daily` records every customer's end-of-day checking balance.
* **Trap:** An analyst queries Q1 balances by running `SELECT customer_id, SUM(ending_balance) FROM fct_account_balances_daily WHERE quarter = 'Q1'`, multiplying the customer's balance by 90 days!
* **Principal Solution:**
  * Explicitly name snapshot columns `eod_balance_usd` (End-of-Day) and document their semi-additive nature.
  * For high-frequency balance changes (e.g., ledger accounts with 500 transactions/day), store **additive net delta** (`net_balance_change_usd`) in `fct_ledger_entries` *and* end-of-day snapshots in `fct_account_balances_daily`. Summing `net_balance_change_usd` up to any timestamp yields the exact point-in-time balance!

### Pitfall 4: Financial Metrics — Gross vs. Net, Refunds & Multi-Currency
Whenever money is involved in an interview, proactively define **four columns** instead of a single ambiguous `amount` column:
1. `gross_amount_local` (before discounts, taxes, refunds)
2. `discount_amount_local` & `tax_amount_local`
3. `net_revenue_local` (what the business actually recognizes as revenue)
4. `fx_rate_to_usd` + `net_revenue_usd` (locked at the transaction timestamp's exchange rate so historical USD reports never fluctuate when FX rates move next month).

---

## 3. Real Interview Questions & Model Answers

### Q1: "In a subscription SaaS warehouse, how do you model Monthly Recurring Revenue (MRR) so Finance can slice Ending MRR, New MRR, Expansion MRR, Contraction MRR, and Churned MRR across customer segments and regions?"

**Principal-Level Answer:**
> *"MRR is a classic **semi-additive stock metric** (Ending MRR cannot be summed across months), while MRR *movements* (New, Expansion, Contraction, Churn) are **fully additive flow metrics** within a period.*
>
> *To make both trivial and error-free to query, I model a **Monthly Periodic Snapshot Fact** (`fct_workspace_mrr_monthly`) at the grain of **one row per `(workspace_sk, month_end_date_sk)`** for every active workspace plus any workspace that churned in that exact month. Each row contains:*
> * *`beginning_mrr_usd` (Semi-additive: equals prior month's `ending_mrr_usd`)*
> * *`ending_mrr_usd` (Semi-additive: `0.00` on the churn month row, then excluded in subsequent months)*
> * *`new_mrr_usd`, `expansion_mrr_usd`, `contraction_mrr_usd`, `churned_mrr_usd` (All **fully additive** flows)*
> * *`net_mrr_movement_usd` = `ending_mrr_usd - beginning_mrr_usd` (Additive)*
>
> *This guarantees the accounting identity `SUM(beginning_mrr) + SUM(new) + SUM(expansion) - SUM(contraction) - SUM(churned) = SUM(ending_mrr)` holds cleanly across any segment or region slice."*

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "What if a customer upgrades on March 15th and downgrades on March 25th within the same month? How does your monthly MRR table reflect that?"**
   * **Expected Insight:** On a monthly snapshot, comparing Feb 28 (`$100`) vs. Mar 31 (`$120`) shows a net `+$20` Expansion MRR, hiding the intra-month volatility. To preserve intra-month auditability, pair `fct_workspace_mrr_monthly` with an underlying **Transaction Fact** (`fct_subscription_events`) capturing every mid-month state change (`+$50` on Mar 15, `-$30` on Mar 25).
2. **Top-Up 2: "How do you prevent BI analysts from accidentally summing semi-additive columns in tools like Looker, dbt Semantic Layer, or Tableau?"**
   * **Expected Insight:** Enforce metric definitions in a **Semantic Layer (dbt MetricFlow / LookML / Cube)** using `non_additive_dimension: { name: date_day, window_choice: max }` so the BI tool automatically applies `LAST_VALUE` / `MAX(date)` over time while allowing `SUM()` over regions and segments.

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Decomposes ratios into numerator + denominator** in fact and aggregate tables. | Stores `conversion_rate` or `ctr` as a float column in a fact table. |
| **Distinguishes stock (semi-additive) vs. flow (additive)** metrics and uses ledger deltas or snapshots appropriately. | Sums daily inventory or account balances across a 30-day month. |
| **Captures both local currency and point-in-time USD** (`amount_local`, `fx_rate_to_usd`, `amount_usd`). | Uses a single `price` column with no currency code or exchange rate timestamp. |