# Lesson 37: Principal Architectural Audit & L7 Advanced Addendum

> **Principal Reviewer Note:** In a Senior (L5) or Staff (L6) loop, a clean Star Schema with SCD Type 2 and date partitioning earns a Hire. In a **Principal (L7/L8) loop**, the bar shifts from *"Does this schema work?"* to *"Where does this schema silently corrupt data at 10B events/day, how does it serve ML feature pipelines without target leakage, how do we enforce upstream data contracts, and how do we govern FinOps compute costs?"*
>
> Below is a rigorous module-by-module audit of Lessons 1–36, fixing subtle edge-case bugs and adding L7 production depth.

## 1. Module-by-Module Principal Audit Summary

| Module | What Was Already Strong | Subtle Production Gap Identified | Principal Upgrade Added Below | 
| ----- | ----- | ----- | ----- | 
| **Module 1: Overview & Fundamentals** | Clear 45-min pacing, Bus Matrix, Medallion layering (`Bronze -> Silver -> Gold`). | Missing **Upstream Data Contracts** (schema enforcement at microservice producer boundaries) and **Semantic Layer Governance**. | Section 2: Shift-Left Data Contracts & CI/CD Breaking-Change Gates. | 
| **Module 2: Requirements & Scale** | 4-Layer Discovery Funnel, HLL sketches, Bi-Temporal modeling, GDPR Crypto-Shredding. | Missing **FinOps Unit Economics** (quantifying storage vs. scan vs. `MERGE` rewrite dollars). | Section 6: FinOps Cost Modeling & Query Governance Guardrails. | 
| **Module 3: High-Level Diagrams & Domains** | 6 domain blueprints (`E-Commerce`, `Social`, `Streaming`, `Ride-Share`, `Cloud`, `AdTech`). | Missing **ML Feature Store Point-in-Time (PIT) correctness** across those domains. | Section 4: Zero-Leakage ML Feature Engineering (`ASOF JOIN` on SCD2). | 
| **Module 4: Dimension Design** | SCD Types 0–7, `tracked_attr_hash`, Inferred Members, Junk/Mini/Closure tables. | **CRITICAL BUG FIX in Lesson 22:** Standard single-pass `MERGE` fails if `stg_merchant_cdc` contains **2+ changes for the same `merchant_id` within a single batch**! | Section 3: Multi-Version Intra-Batch CDC SCD Type 2 Algorithm. | 
| **Module 5: Fact Data Modeling** | Transaction, Periodic Snapshot, Accumulating Snapshot, Bridge & Coverage Factless Facts. | Missing **Transactional Outbox + CDC** pattern explaining how upstream services guarantee zero lost/phantom events. | Section 5: End-to-End Exactly-Once Ingestion & Outbox Pattern. | 
| **Module 6: Performance Optimization** | Iceberg Hidden Partitioning, Z-Order/Hilbert, Bloom Filters, SPJ, Rollup Pyramids. | Missing **Table Maintenance Lifecycle SLAs** (Compaction, Manifest Rewrites, Snapshot Expiration vs. Time-Travel) and **MV Incremental Refresh Rules**. | Section 6: Lakehouse Compaction & Materialized View Internals. | 
| **Module 7: Mock Interviews (31–36)** | 6 complete 45-min simulations (`Uber`, `Support`, `Airbnb`, `Stripe`, `Instagram`, `Amazon`). | Ready for L7 calibration when paired with the 5 architectural upgrades in this addendum. | Integrated into the upgraded Master Cheat Sheet (`00_master_*`). | 

## 2. \[Module 1 Upgrade\] Shift-Left Data Contracts & Semantic Governance

At the Principal level, you do not passively clean up broken upstream schemas in Silver—you establish **Data Contracts** between software engineering producers and the data platform:

1. **Producer-Side Schema Registry (Protobuf / Avro / JSON Schema):**

   * Enforce `BACKWARD` or `FULL` compatibility in Confluent/AWS Glue Schema Registry.

   * CI/CD pipelines block any microservice PR that renames a field, narrows a type (`BIGINT -> INT`), or removes a required business key without bumping the major event version (`OrderCompleted.v2`).

2. **Tiered Gold SLA Contracts (Tier-0 vs. Tier-1 vs. Exploratory):**

   * **Tier-0 (Financial & Regulatory Core — e.g., `fct_balance_transactions`, `dim_merchant`):** Strict PagerDuty alerting, `< 15 min` freshness SLA, automated row-count/reconciliation circuit breakers (**Write-Audit-Publish**), and a mandatory 60-day Expand-Contract deprecation policy.

   * **Tier-1 (Product & Operational Analytics):** Standard business-hours alerting, hourly freshness, automated dbt schema tests.

   * **Exploratory / Sandbox:** Self-service marts with 30-day TTL and zero downstream SLA guarantees.

## 3. \[Module 4 Critical Fix\] Multi-Version Intra-Batch CDC for SCD Type 2

### The Subtle Bug in Standard `MERGE`

In Lesson 22, our `MERGE` joined `stg_merchant_cdc` to `dim_merchant WHERE is_current = TRUE`.

* **What happens in production:** Suppose our SCD2 job runs every 1 hour. At `10:05 AM`, Merchant M-101 upgrades from `SMB` to `MID_MARKET`. At `10:42 AM`, M-101 upgrades again from `MID_MARKET` to `ENTERPRISE`.

* If `stg_merchant_cdc` contains **both** rows for `M-101`, a standard `MERGE` throws a non-deterministic cardinality error (`ON clause matched multiple source rows`)—or worse, deduplicates to `10:42 AM` and **erases the `10:05 AM – 10:42 AM` `MID_MARKET` version**, misclassifying orders placed at `10:20 AM`!

### Principal Fix: Windowed Timeline Construction Before `MERGE`

Instead of feeding raw multi-row CDC directly into `MERGE`, construct contiguous `[valid_from_utc, valid_to_utc)` intervals across **both** the existing current dimension row and all new intra-batch changes using `LEAD()`:

```
-- ============================================================================
-- BULLETPROOF MULTI-VERSION INTRA-BATCH SCD TYPE 2 PIPELINE
-- Handles 1, 2, or 50 changes for the same entity within a single micro-batch!
-- ============================================================================
WITH incoming_deduped_changes AS (
    -- Keep only rows where tracked business attributes actually changed from the prior event
    SELECT
        merchant_id,
        merchant_name,
        merchant_tier,
        commission_rate_pct,
        country_code,
        cdc_event_ts_utc,
        XXHASH64(CONCAT_WS('|', merchant_name, merchant_tier, CAST(commission_rate_pct AS STRING), country_code)) AS tracked_attr_hash,
        LAG(XXHASH64(CONCAT_WS('|', merchant_name, merchant_tier, CAST(commission_rate_pct AS STRING), country_code)))
            OVER (PARTITION BY merchant_id ORDER BY cdc_event_ts_utc ASC) AS prev_batch_hash
    FROM stg_merchant_cdc
),
true_state_transitions AS (
    SELECT c.*
    FROM incoming_deduped_changes c
    LEFT JOIN gold.dim_merchant cur
      ON c.merchant_id = cur.merchant_id
     AND cur.is_current = TRUE
    WHERE
        -- First event in batch differs from current warehouse state (or brand-new entity)
        (c.prev_batch_hash IS NULL AND (cur.merchant_id IS NULL OR c.tracked_attr_hash <> cur.tracked_attr_hash))
        -- Subsequent events in batch differ from preceding event in batch
        OR (c.prev_batch_hash IS NOT NULL AND c.tracked_attr_hash <> c.prev_batch_hash)
),
-- Combine existing current row + all new transitions for affected entities to compute exact LEAD() bounds
combined_timeline AS (
    SELECT
        cur.merchant_sk                                 AS existing_sk,
        cur.merchant_id,
        cur.merchant_name,
        cur.merchant_tier,
        cur.commission_rate_pct,
        cur.country_code,
        cur.tracked_attr_hash,
        cur.valid_from_utc,
        FALSE                                           AS is_new_version
    FROM gold.dim_merchant cur
    WHERE cur.is_current = TRUE
      AND cur.merchant_id IN (SELECT DISTINCT merchant_id FROM true_state_transitions)

    UNION ALL

    SELECT
        NULL                                            AS existing_sk,
        t.merchant_id,
        t.merchant_name,
        t.merchant_tier,
        t.commission_rate_pct,
        t.country_code,
        t.tracked_attr_hash,
        t.cdc_event_ts_utc                              AS valid_from_utc,
        TRUE                                            AS is_new_version
    FROM true_state_transitions t
),
final_intervals AS (
    SELECT
        COALESCE(
            existing_sk,
            XXHASH64(CONCAT(merchant_id, '|', CAST(valid_from_utc AS STRING)))
        )                                               AS merchant_sk,
        merchant_id,
        merchant_name,
        merchant_tier,
        commission_rate_pct,
        country_code,
        tracked_attr_hash,
        valid_from_utc,
        LEAD(valid_from_utc, 1, TIMESTAMP '9999-12-31 23:59:59')
            OVER (PARTITION BY merchant_id ORDER BY valid_from_utc ASC) AS valid_to_utc,
        CASE WHEN LEAD(valid_from_utc) OVER (PARTITION BY merchant_id ORDER BY valid_from_utc ASC) IS NULL
             THEN TRUE ELSE FALSE END                   AS is_current,
        is_new_version
    FROM combined_timeline
)
-- Now expire old current rows (is_new_version = FALSE) and insert all new intervals (is_new_version = TRUE)!
SELECT * FROM final_intervals;

```

## 4. \[Module 3 & 4 Upgrade\] Zero-Leakage ML Feature Engineering (`ASOF JOIN`)

A top Principal interview curveball: *"Data Scientists want to train a fraud or churn model using features from `dim_user` and `fct_user_rolling_metrics` as of the exact millisecond a transaction occurred. How does your data model prevent data leakage?"*

* **The Data Leakage Trap:** Joining `fct_transactions` to `dim_user WHERE is_current = TRUE` leaks future information into historical training rows (e.g., a user who was flagged `is_high_risk = TRUE` *after* committing fraud on Nov 10 appears `is_high_risk = TRUE` on their Nov 1 training row—giving the model 99% offline AUC and 50% production accuracy!).

* **Principal Solution (`ASOF JOIN` on SCD Type 2 / Feature Snapshot Tables):**
  Modern engines (Snowflake `ASOF JOIN`, Databricks/Spark, DuckDB, ClickHouse) natively support temporal `ASOF` joins against SCD Type 2 dimensions:

```
-- Point-in-Time (PIT) ML Training Dataset Generation (Zero Future Data Leakage!)
SELECT
    t.transaction_id,
    t.occurred_at_utc,
    t.amount_usd,
    t.is_fraud_label,                                   -- Target label
    u.account_tenure_days,                              -- Feature state strictly BEFORE transaction!
    u.kyc_verification_level,
    u.historical_chargeback_count
FROM gold.fct_transactions t
ASOF JOIN gold.dim_user u
  MATCH_CONDITION (t.occurred_at_utc >= u.valid_from_utc)
  ON t.user_id = u.user_id
WHERE t.occurred_at_utc >= TIMESTAMP '2026-06-01';

```

## 5. \[Module 5 Upgrade\] Transactional Outbox + CDC for Exactly-Once Events

When an interviewer asks *"How do you guarantee that `fct_orders` and `fct_balance_transactions` never miss an event or emit a phantom event when an upstream microservice crashes mid-request?"*:

* **The Dual-Write Failure:** If a checkout microservice writes `INSERT INTO orders` in Postgres and then calls `kafkaProducer.send(OrderCreated)` separately, a crash between the DB commit and Kafka ack causes a **silent missing order** in the warehouse (or a phantom Kafka event if the DB rolls back).

* **Principal Pattern (Transactional Outbox + Debezium CDC):**

  1. Inside the **same local ACID database transaction**, the microservice inserts the business row into `orders` AND inserts the canonical event payload (including a deterministic `event_id` UUIDv7) into an `outbox_events` table.

  2. **Debezium / CDC** tails the Postgres WAL (`Write-Ahead Log`), publishes `outbox_events` to Kafka, and the Lakehouse ingestion job deduplicates by `event_id` during `MERGE`—achieving **end-to-end deterministic exactly-once semantics** without distributed 2PC locks.

## 6. \[Module 6 Upgrade\] Lakehouse Maintenance SLAs & MV Incremental Refresh Internals

### A. Table Maintenance Lifecycle (Iceberg / Delta)

When designing any 10TB+ Iceberg or Delta table, explicitly state the **3 background maintenance jobs** and their cadence:

1. **Bin-Pack & Sort Compaction (`rewrite_data_files` / `OPTIMIZE ZORDER`):** Runs every `1–6 hours` on active partitions to merge tiny streaming files (`< 32 MB`) into optimal `512 MB` Parquet files and apply Z-Order/Hilbert clustering.

2. **Manifest Rewriting (`rewrite_manifests`):** Runs daily to compact fragmented Avro manifest files so query planning (`metadata scan`) stays under `< 200ms`.

3. **Snapshot Expiration & Orphan File Cleanup (`expire_snapshots` + `remove_orphan_files`):** Runs daily with a `7-day` retention window (matching your time-travel & GDPR erasure SLA) so S3/GCS storage bills don't balloon from historical copy-on-write/delete-vector snapshots.

### B. Why Declarative Materialized Views Break Incremental Refresh

If an interviewer asks *"Why didn't you just use an auto-refreshing Materialized View for that complex 4-table Star Schema rollup?"*, cite the exact engine internals:

* **Incremental MV refresh** (Snowflake, BigQuery, Databricks) relies on **algebraic delta propagation** (maintaining running `SUM` and `COUNT` states per group key).

* Operations that are **non-algebraic**—such as `COUNT(DISTINCT user_id)`, `PERCENTILE_CONT(0.95)`, `WINDOW` functions, `OUTER JOIN`s, or `NON-DETERMINISTIC` functions (`CURRENT_TIMESTAMP()`)—force the engine to fall back to a **full recompute** on every refresh, spiking compute costs.

* **Principal Workaround:** Replace `COUNT(DISTINCT user_id)` inside the MV with `HLL_ACCUMULATE(user_id)` (HyperLogLog state is algebraically mergeable!), or use an explicit incremental dbt/Spark partition-overwrite pipeline.

## 7. \[Module 2 & 6 Upgrade\] FinOps Compute & Storage Cost Governance

Demonstrate executive ownership by quantifying the **three biggest cloud warehouse cost multipliers** and how your data model neutralizes each:

| Cost Driver | What Causes Runaway Bills | Principal Model & Platform Control | 
| ----- | ----- | ----- | 
| **1. Unbounded Table Scans** | Analysts running `SELECT *` without date filters on 50TB fact tables. | `require_partition_filter = TRUE`, coarse `MONTH`/`DAY` partitioning, and Semantic Layer default lookback filters (`last 30 days`). | 
| **2. Copy-on-Write `MERGE` Amplification** | Updating 1,000 rows in a 10TB table rewrites 500GB of Parquet files every 15 minutes. | Partition by immutable creation date, bound `MERGE` target scan (`is_terminal_state = FALSE`), and enable **Merge-on-Read Deletion Vectors**. | 
| **3. Serverless Auto-Clustering Churn** | Defining `CLUSTER BY` on high-cardinality random UUIDs or volatile columns. | Cluster strictly on 2–3 stable, high-filter dimensions (`merchant_id`, `city_id`), and use **Bloom Filters** instead of sorting for random UUID point lookups. | 
