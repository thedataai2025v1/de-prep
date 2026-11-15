# Lesson 8: Analyzing Query Patterns & Read/Write Access Paths

> **Principal Mentor Note:** A logical schema that ignores how queries actually scan bytes is a performance disaster waiting to happen. Before finalizing any table structure, you must map the **top 5–8 read access patterns** (filters, joins, group-bys, window functions) and **write access patterns** (append-only vs. high-frequency upserts). Your physical layout—partitioning, clustering, pre-joins, and snapshots—is dictated 100% by these access paths.

---

## 1. The 5 Core Analytical Query Archetypes

Every analytical workload falls into one or more of these five query patterns. Ask the interviewer which ones dominate:

| Query Archetype | SQL Signature | Physical Bottleneck | Optimal Modeling & Storage Pattern |
| :--- | :--- | :--- | :--- |
| **1. Time-Series KPI Rollups** | `WHERE event_date BETWEEN ... GROUP BY region, category` | Full-table date scans; slow wide joins on high-cardinality dimensions. | **Date Partitioning** + **Star Schema** (or daily Gold rollup aggregate). |
| **2. Entity-Centric Point / History Lookup** | `WHERE user_id = 'U123'` or `WHERE merchant_id = 'M456' AND date >= ...` | Scanning 365 daily partitions just to find 10 rows for one customer! | **Clustering / Z-Ordering on `(entity_id, event_date)`** (or secondary projections/bucketing). |
| **3. Funnel & Sequential Behavioral Queries** | `LEAD() / LAG()`, sessionization, `step_1 -> step_2 -> step_3` within 30 mins | Massive self-joins or multi-pass shuffles on raw event logs. | **Sessionized Fact Tables** or **Bitmask / Array Funnel Summaries** per `(user_id, date)`. |
| **4. Cohort Retention & Survivorship** | `GROUP BY signup_month, months_since_signup` | Expensive `N x M` cross-joins between signup table and activity log. | **Conformed `cohort_month` attribute** stamped on `dim_user` + **User Monthly Activity Fact**. |
| **5. Point-in-Time (PIT) / As-Of Joins** | `f.event_ts >= d.valid_from AND f.event_ts < d.valid_to` | Range-join explosion (`Inequality Join`) in distributed engines (Spark/Snowflake). | Pre-resolve `dimension_sk` during ETL ingestion so BI queries use fast **equi-joins (`f.user_sk = d.user_sk`)**. |

---

## 2. Deep Dive: Optimizing for Specific Access Paths

### Access Path A: High-Cardinality Entity Lookups on Time-Partitioned Tables
* **The Problem:** Suppose `fct_transactions` has 50B rows, partitioned by `transaction_date`. A compliance dashboard or customer support tool runs `SELECT * FROM fct_transactions WHERE customer_id = 'cust_99812'` across the last 2 years. Even with date partitioning, the engine must open files in **all 730 partitions** because `cust_99812` appears randomly across dates!
* **The Principal Solution:**
  1. **Coarse Date Partitioning + Fine Entity Clustering:** Partition by `MONTH(transaction_date)` (reducing partition metadata overhead) and **Cluster / Liquid Cluster / Z-Order by `(customer_id, transaction_date)`**. Min/max column statistics in Parquet/Iceberg headers allow the engine to skip 99.9% of row groups where `customer_id` falls outside `[min_id, max_id]`.
  2. **Dedicated Entity-Partitioned Serving Table (if SLA < 500ms):** Materialize a narrow serving table or DynamoDB/Pinot index keyed by `customer_id` for sub-second customer support lookups.

### Access Path B: Eliminating Expensive Range Joins in BI (`SCD Type 2`)
* **The Problem:** If an analyst writes an SCD Type 2 join manually in BI:
  ```sql
  -- SLOW & ERROR-PRONE: Range join at query time
  SELECT d.tier, SUM(f.amount)
  FROM fct_orders f
  JOIN dim_customer d
    ON f.customer_id = d.customer_id
   AND f.order_ts >= d.valid_from
   AND f.order_ts <  d.valid_to
  GROUP BY 1;
  ```
  Range joins cannot use standard hash joins efficiently and frequently cause duplicate rows if `valid_from`/`valid_to` boundaries overlap by 1 millisecond.
* **The Principal Solution:** Resolve the exact historical surrogate key (`customer_sk`) **once during the Silver-to-Gold ETL pipeline**, and store `customer_sk` directly as a foreign key on `fct_orders`. Now every BI query becomes a blazing-fast **integer equi-join** (`ON f.customer_sk = d.customer_sk`), while also keeping `customer_id` (natural key) on `fct_orders` for `as-is` current-state joins (`ON f.customer_id = d.customer_id AND d.is_current = TRUE`).

### Access Path C: Funnel & Pathing Queries at Scale
* **The Problem:** Product managers constantly ask: *"Of users who viewed a product, how many added to cart within 1 hour, and how many purchased within 24 hours?"* Running 3-way self-joins on a 5B-row `fct_clickstream` table times out.
* **The Principal Solution:** Build a **Daily User Funnel Fact** (`fct_user_daily_funnel`, grain: 1 row per `(user_sk, date_sk)`) storing milestone timestamps (`first_view_ts`, `first_cart_ts`, `first_purchase_ts`) and step counts (`view_count`, `cart_count`, `purchase_count`). Now a 3-step conversion funnel becomes a single-pass `COUNT(CASE WHEN ...)` scan with zero self-joins!

---

## 3. Write Access Patterns: Append vs. Upsert (`MERGE`) Amplification

Never ignore how data *enters* the model:
* **Append-Only (`INSERT`):** Immutable facts (`fct_clickstream`, `fct_payments`). Zero file rewrites; optimal for Iceberg/Delta/BigQuery.
* **Mutable Upserts (`MERGE INTO`):** Accumulating snapshots (`fct_order_fulfillment`) and SCD Type 2 dimensions.
  * **Principal Warning:** If you run a `MERGE` across an unpartitioned 20TB table to update 50,000 rows that changed today, copy-on-write engines will rewrite gigabytes of untouched Parquet files (**Write Amplification**).
  * **Mitigation:** Always bound `MERGE` target scans using a partition predicate (e.g., `WHERE target.order_date >= CURRENT_DATE - INTERVAL '14 days'`) and use **Merge-on-Read (deletion vectors)** in Apache Iceberg v2/v3 or Delta Lake for high-churn tables.

---

## 4. Real Interview Questions & Model Answers

### Q1: "We have a 10-billion-row `fct_orders` table. Half our queries filter by `order_date` (Executive BI), and the other half filter by `merchant_id` without a date filter (Merchant Portal Analytics). How do you design the physical layout and table strategy?"

**Principal-Level Answer:**
> *"A single physical sort order cannot simultaneously optimize full-table `order_date` scans and unbounded `merchant_id` lookups at 10B rows without trade-offs. I would solve this in two tiers:*
> 1. ***Core Fact Table (`fct_orders`):** Partition by `MONTH(order_date)` and apply **multi-dimensional clustering (Liquid Clustering / Z-Order) on `(merchant_id, order_date)`**. Using monthly rather than daily partitions prevents small-file proliferation while keeping `merchant_id` values tightly grouped inside Parquet row groups, allowing metadata pruning to skip ~95%+ of files for single-merchant queries.*
> 2. ***Merchant Portal Serving Layer (`agg_merchant_daily_kpis`):* Since the Merchant Portal displays pre-defined time-series KPIs (daily GMV, order count, refund rate), I materialize a daily aggregate table **partitioned/clustered primarily by `merchant_id`**. At ~1M merchants $\times$ 1,000 days, that table is only 1B tiny rows—serving portal queries in under 150ms without touching the 10B-row atomic fact table."*

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "What happens when a query filters on a column that is neither the partition key nor the cluster key—say, `promo_code`?"**
   * **Expected Insight:** The query engine falls back to scanning all matching date partitions, though Parquet column chunk min/max stats and **Bloom filters** (which you can explicitly enable on high-cardinality string columns like `promo_code` or `trace_id` in Iceberg/Databricks/Snowflake Search Optimization) can still prune files where that `promo_code` does not exist.
2. **Top-Up 2: "Why can over-partitioning (e.g., partitioning by `date` AND `country` AND `device_type`) destroy query performance?"**
   * **Expected Insight:** **The Small File Problem.** `365 days * 200 countries * 5 devices = 365,000 partitions` per year. Each partition contains tiny 50 KB Parquet files, causing metadata bottlenecks (Hive Metastore / Iceberg manifest bloat), excessive S3/GCS `LIST`/`GET` API calls, and poor columnar compression ratios. Target **128 MB – 1 GB per Parquet file** and keep partition cardinality coarse.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Maps top read AND write patterns** before finalizing partitioning and clustering keys. | Picks `PARTITION BY date` blindly for every table regardless of entity-centric query patterns. |
| **Pre-resolves SCD Type 2 surrogate keys at write time** so BI queries run fast integer equi-joins. | Forces BI tools to execute complex timestamp inequality joins (`>=` and `<`) at runtime. |
| **Understands write amplification on `MERGE`** and bounds target scans with partition predicates + deletion vectors. | Runs unbounded `MERGE INTO` across multi-year tables every 15 minutes. |