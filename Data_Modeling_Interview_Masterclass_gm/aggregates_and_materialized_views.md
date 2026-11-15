# Lesson 30: Aggregates, Pre-Aggregations & Materialized Views

> **Principal Mentor Note:** Even with perfect partitioning and Z-ordering, scanning `20 billion` atomic rows every time an executive opens a Tableau dashboard is economically unsustainable and violates sub-3-second latency SLAs. **Aggregate tables and Materialized Views (MVs)** are the ultimate force multipliers in data modeling—delivering **100x to 10,000x faster queries** while cutting warehouse compute spend by **80%+**. At the Principal level, you must design **strictly additive rollup schemas**, choose between **Declarative Materialized Views vs. Incremental ETL Aggregate Tables**, and leverage **automatic query rewrite**.

---

## 1. The Aggregation Pyramid: Serving Multiple SLAs from One Source of Truth

Never duplicate business logic across ad-hoc rollup scripts. Build a **layered aggregation hierarchy** fed deterministically from your atomic Gold fact table:

```
                   +-------------------------------+
                   | Tier 3: Executive Daily KPIs  |  ~50K rows/day | < 200ms SLA
                   | (agg_company_daily_executive) |  (Multi-year board trends)
                   +---------------+---------------+
                                   ^
                   +---------------+---------------+
                   | Tier 2: Domain Hourly/Daily   |  ~5M rows/day  | < 2s SLA
                   | (agg_merchant_daily_metrics)  |  (Partner portal & slice-and-dice)
                   +---------------+---------------+
                                   ^
                   +---------------+---------------+
                   | Tier 1: Atomic Fact Table     |  ~500M rows/day| 10s–60s SLA
                   | (fct_order_line_items)        |  (Ad-hoc forensics, ML, cohort deep-dives)
                   +-------------------------------+
```

---

## 2. Five Golden Rules for Designing Aggregate Tables

1. **Rule 1: Store Constituent Numerators & Denominators — NEVER Pre-Computed Ratios or Averages:**
   * Store `total_revenue_usd`, `total_orders`, `total_clicks`, `total_impressions`, `total_delivery_seconds`.
   * Never store `avg_order_value`, `ctr_pct`, or `avg_delivery_mins` as columns to be averaged downstream (prevents **Simpson's Paradox / Averaging Averages**).
2. **Rule 2: Use HyperLogLog (HLL) Sketches for Non-Additive Distinct Counts:**
   * If executives slice `Unique Customers` or `Unique Devices` by day, week, month, and region, store `HLL_ACCUMULATE(customer_id)` (`VARBINARY` / `HLL_SKETCH`) in your daily aggregate table so `HLL_ESTIMATE(HLL_COMBINE(sketch))` computes exact-ish distinct counts across any rollup in milliseconds!
3. **Rule 3: Conform Aggregate Dimensions to the Base Fact's Dimensions:**
   * Every foreign key in `agg_merchant_daily_metrics` (`date_sk`, `merchant_sk`, `city_sk`, `channel_sk`) must map 1:1 to the exact same conformed dimensions (`dim_date`, `dim_merchant`, `dim_city`, `dim_channel`) used by `fct_orders`.
4. **Rule 4: Shrink Dimension Keys via Shrunken / Rollup Dimensions:**
   * If `fct_orders` joins to `dim_date` (day grain) and `dim_product` (SKU grain), a monthly category aggregate (`agg_category_monthly`) joins to **Shrunken Dimensions** (`dim_month` and `dim_product_category`)—reducing join cardinality by 1,000x!
5. **Rule 5: Always Partition/Cluster Aggregates by Their Primary Consumer Access Path:**
   * If `fct_orders` is partitioned by `order_date` (for internal analytics), partition/cluster `agg_merchant_daily_metrics` by `(merchant_id, order_date)` so external Merchant Portal queries hit a single tenant block in `< 100ms`.

---

## 3. Declarative Materialized Views (MVs) vs. Incremental ETL Aggregate Tables

A frequent Principal interview question: *"Should we implement this rollup as a database Materialized View (Snowflake MV / BigQuery MV / Databricks MV) or as an explicit incremental dbt/Spark aggregate table?"*

| Evaluation Criterion | Declarative Materialized View (Engine-Managed) | Incremental ETL Aggregate Table (`dbt` / Spark `MERGE`) |
| :--- | :--- | :--- |
| **Automatic Query Rewrite** | **Yes** — BI tools query the base table `fct_orders`, and the optimizer transparently rewrites the plan to read the MV! | **No** (unless routed via a Semantic Layer like dbt MetricFlow / Looker `aggregate_awareness`). |
| **Maintenance & Freshness** | Automatically refreshed incrementally by warehouse background workers whenever base partitions change. | Scheduled via orchestrator (Airflow/Dagster) on explicit SLA cadence (e.g., hourly). |
| **SQL Complexity Limits** | **Strict restrictions:** Most engines forbid multi-table joins, `WINDOW` functions, `UDFs`, `UNION ALL`, or non-deterministic functions inside auto-refreshing MVs. | **Unlimited SQL/Spark power:** Supports complex multi-fact joins, SCD2 lookups, HLL sketches, and custom business rules. |
| **Cost Predictability** | Background auto-refresh credits can spike unexpectedly if upstream micro-batches update partitions every 60 seconds. | **100% predictable compute cost** tied to scheduled batch runs. |
| **Principal Verdict** | Best for **single-table projections/rollups** (`SELECT date_sk, merchant_id, SUM(amount) FROM fct_orders GROUP BY 1, 2`) where transparent query rewrite shines. | Best for **multi-table star-schema rollups**, complex KPIs, HLL sketches, and high-frequency streaming upstream tables. |

---

## 4. Production DDL & Query Pattern: Incremental Rollup with HLL Sketches

```sql
-- ============================================================================
-- GOLD DAILY AGGREGATE TABLE WITH HYPERLOGLOG DISTINCT REACH
-- Grain: One row per (order_date_sk, merchant_sk, channel_sk)
-- Compresses 50M daily order rows -> ~400K daily aggregate rows (125x reduction!)
-- ============================================================================
CREATE TABLE gold.agg_merchant_daily_kpis (
    order_date_sk           INT NOT NULL,
    merchant_sk             BIGINT NOT NULL,
    merchant_id             VARCHAR(64) NOT NULL,  -- Denormalized for fast portal pruning
    channel_sk              INT NOT NULL,
    -- Fully Additive Numerators & Denominators
    total_orders            BIGINT NOT NULL,
    completed_orders        BIGINT NOT NULL,
    canceled_orders         BIGINT NOT NULL,
    gross_gmv_usd           DECIMAL(16,2) NOT NULL,
    net_revenue_usd         DECIMAL(16,2) NOT NULL,
    total_delivery_seconds  BIGINT NOT NULL,
    -- Algebraically Mergeable HyperLogLog Sketch for Distinct Customers!
    unique_customer_hll     BINARY NOT NULL,
    PRIMARY KEY (order_date_sk, merchant_sk, channel_sk)
)
PARTITION BY (order_date_sk)
CLUSTER BY (merchant_id, order_date_sk);

-- ============================================================================
-- EXECUTIVE ROLLUP QUERY ACROSS ARBITRARY DATE RANGES & CHANNELS
-- Computes exact-ish Unique Customers across 90 days WITHOUT scanning raw orders!
-- ============================================================================
SELECT
    d.year_month,
    SUM(a.completed_orders)                                         AS completed_orders,
    SUM(a.net_revenue_usd)                                          AS net_revenue_usd,
    ROUND(SUM(a.net_revenue_usd) / NULLIF(SUM(a.completed_orders), 0), 2) AS true_aov_usd,
    ROUND(SUM(a.total_delivery_seconds) / 60.0 / NULLIF(SUM(a.completed_orders), 0), 1) AS avg_delivery_mins,
    HLL_ESTIMATE(HLL_COMBINE(a.unique_customer_hll))                AS est_unique_customers
FROM gold.agg_merchant_daily_kpis a
JOIN gold.dim_date d ON a.order_date_sk = d.date_sk
WHERE a.merchant_id = 'MERCH-8821'
  AND d.calendar_date BETWEEN DATE '2026-08-01' AND DATE '2026-10-31'
GROUP BY 1
ORDER BY 1;
```

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "How do you keep `agg_merchant_daily_kpis` consistent when late-arriving orders or refunds arrive 3 days after the original `order_date_sk`?"**
   * **Expected Insight:** Use **Partition-Replace Incremental Overwrite (`INSERT OVERWRITE` / dynamic partition replacement)** over a trailing lookback window (e.g., recompute the last `7` days of partitions from `fct_orders` on every scheduled run). Re-aggregating 7 days of `fct_orders` takes seconds and is **100% idempotent**, avoiding brittle `+=` delta arithmetic bugs on `MERGE`.
2. **Top-Up 2: "How do BI tools know whether to query `agg_merchant_daily_kpis` or `fct_orders` without forcing analysts to memorize 15 different aggregate table names?"**
   * **Expected Insight:** Use **Aggregate Awareness in a Semantic Layer** (Looker `aggregate_table`, dbt MetricFlow, Cube, or Snowflake/BigQuery Materialized View query rewrite). Analysts always build charts against a single logical model (`orders`), and the semantic layer automatically routes high-level queries to `agg_merchant_daily_kpis` while seamlessly falling back to `fct_orders` when an analyst adds a fine-grained filter like `promo_code` or `sku_id`.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Stores additive numerators/denominators + HLL sketches** in aggregate tables, never pre-divided ratios or `COUNT(DISTINCT)` integers. | Stores `avg_price`, `conversion_rate`, and `distinct_users` integers in daily rollup tables and averages/sums them across weeks. |
| **Articulates the exact trade-offs between engine-managed Materialized Views and incremental ETL aggregate tables** (join limits, auto-refresh cost spikes vs. query rewrite). | Recommends a Materialized View with a 6-table join and `COUNT(DISTINCT)` and assumes every warehouse supports incremental auto-refresh on it. |
| **Uses idempotent partition overwrite (`INSERT OVERWRITE` over a 7-day lookback)** to keep aggregates synced with late-arriving data. | Uses fragile `SUM += delta` incremental updates that drift silently on pipeline retries. |