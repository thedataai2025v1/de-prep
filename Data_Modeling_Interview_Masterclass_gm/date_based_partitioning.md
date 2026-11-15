# Lesson 28: Date-Based Partitioning & Modern Lakehouse Partition Pruning

> **Principal Mentor Note:** Partitioning is the single highest-leverage physical optimization in analytical data warehouses and lakehouses—and also the easiest way to accidentally ruin query performance via **over-partitioning (the Small File Problem)** or **partition key mismatch**. At the Principal level, you must compare **legacy Hive-style directory partitioning** against **Apache Iceberg Hidden Partitioning**, **Snowflake automatic micro-partitions**, and **BigQuery ingestion/column partitioning**, and defend exact partition granularity (`HOUR` vs. `DAY` vs. `MONTH` vs. `YEAR`) using back-of-the-envelope file-size math.

---

## 1. How Partitioning Works Across Modern Engines

| Engine / Table Format | Partitioning Mechanism | How Pruning Works | Key Principal Gotcha |
| :--- | :--- | :--- | :--- |
| **Apache Iceberg** | **Hidden Partitioning** (`PARTITIONED BY (days(occurred_at_utc))` or `months(...)`) | Catalog manifest lists (`manifest-list.avro`) store min/max partition bounds; engine skips entire manifest files before touching S3/GCS Parquet footers. | Analysts query `WHERE occurred_at_utc >= '...'` directly; **never** create a redundant `event_date` string column just for partitioning! |
| **Delta Lake / Databricks** | **Liquid Clustering** (`CLUSTER BY (occurred_at_utc, merchant_id)`) or traditional `PARTITIONED BY` | Transaction log (`_delta_log`) tracks per-file min/max stats; Liquid Clustering dynamically reorganizes data via Hilbert curves without rigid directory locks. | Databricks now recommends **Liquid Clustering over Hive-style `PARTITION BY`** for all tables under several terabytes to prevent small files. |
| **Snowflake** | **Automatic Micro-Partitions** (`50MB–500MB` uncompressed contiguous storage blocks) + optional `CLUSTER BY` | Cloud Services layer maintains min/max metadata per column for every micro-partition in RAM; prunes without user-defined partitions. | Defining manual `CLUSTER BY` on high-cardinality columns triggers continuous background Serverless Auto-Clustering credits—monitor `SYSTEM$CLUSTERING_INFORMATION`. |
| **Google BigQuery** | **Time-Unit Column Partitioning** (`PARTITION BY DATE(occurred_at_utc)`) + `CLUSTER BY` (up to 4 columns) | Segment metadata prunes partitions prior to slot allocation (`bytes billed` drops proportionally). | Hard limit of **10,000 partitions per table** (`~27 years` of daily partitions). Always set `require_partition_filter = TRUE` on 10TB+ tables! |

---

## 2. Choosing Partition Granularity: The 256MB–1GB File Size Rule

**Golden Physical Law of Columnar Storage (Parquet / ORC):**
Optimal analytical scan throughput requires Parquet files between **256 MB and 1 GB** compressed.
* **Too Small (`< 10 MB` per file):** Cloud object store `LIST`/`GET` API latency, Iceberg manifest bloat, and Parquet footer parsing dominate CPU time.
* **Too Large (`> 4 GB` per file):** Poor parallelism (fewer workers can read a single non-splittable row group) and massive `MERGE` rewrite amplification.

### Granularity Sizing Cheat Sheet

| Daily Compressed Data Volume | Optimal Partition Granularity | Why? |
| :--- | :--- | :--- |
| **`< 500 MB / day`** (e.g., `1M` orders/day) | **`YEAR(event_ts)`** or **`MONTH(event_ts)`** | Daily partitioning creates tiny `10–50 MB` files (`3,650` partitions over 10 yrs). Monthly yields clean `15 GB` partitions (`~30` optimal 512MB files/month). |
| **`500 MB – 50 GB / day`** (e.g., `50M` trips/day) | **`DAY(event_ts)`** (or `MONTH` + Z-Order) | Daily partitions hold `1–100` optimally sized `512MB` Parquet files; 90-day queries prune cleanly. |
| **`50 GB – 5 TB / day`** (e.g., `15B` ad impressions/day) | **`HOUR(event_ts)`** | Hourly partitions hold `2 GB – 200 GB` (`4–400` files/hour); enables fast 1-hour micro-batch compaction and retention drops. |
| **`> 5 TB / day`** (e.g., `100B` IoT/network logs/day) | **`HOUR(event_ts)` + Hash Bucket (`bucket(16, entity_id)`)** | Prevents single-hour manifest bottlenecks while keeping each bucket at `~10–20 GB`. |

---

## 3. Production DDL: Iceberg Hidden Partitioning & Partition Evolution

```sql
-- ============================================================================
-- APACHE ICEBERG PRODUCTION FACT TABLE WITH HIDDEN PARTITIONING
-- ============================================================================
CREATE TABLE gold.fct_payment_transactions (
    payment_event_id        STRING NOT NULL,
    occurred_at_utc         TIMESTAMP NOT NULL,    -- Single source of truth timestamp
    ingested_at_utc         TIMESTAMP NOT NULL,
    merchant_id             STRING NOT NULL,
    customer_sk             BIGINT NOT NULL,
    event_type              STRING NOT NULL,
    net_cash_flow_usd       DECIMAL(14,2) NOT NULL
)
USING iceberg
PARTITIONED BY (
    days(occurred_at_utc),                         -- Hidden partition transform (no extra date column!)
    bucket(16, merchant_id)                        -- Secondary hash bucket for 100B+ scale
)
TBLPROPERTIES (
    'write.format.default'               = 'parquet',
    'write.parquet.compression-codec'    = 'zstd',
    'write.target-file-size-bytes'       = '536870912', -- 512 MB target file size
    'write.delete.mode'                  = 'merge-on-read',
    'write.update.mode'                  = 'merge-on-read',
    'write.merge.mode'                   = 'merge-on-read'
);

-- ============================================================================
-- QUERY ANTI-PATTERN THAT BREAKS PARTITION PRUNING!
-- ============================================================================
-- BAD (Wraps partition column in a function or timezone cast -> Full Table Scan!):
SELECT SUM(net_cash_flow_usd)
FROM gold.fct_payment_transactions
WHERE DATE(CONVERT_TIMEZONE('UTC', 'America/New_York', occurred_at_utc)) = DATE '2026-11-15';

-- GOOD (Sargable UTC range predicate -> Prunes to exact 2 UTC day partitions!):
SELECT SUM(net_cash_flow_usd)
FROM gold.fct_payment_transactions
WHERE occurred_at_utc >= TIMESTAMP '2026-11-15 05:00:00'
  AND occurred_at_utc <  TIMESTAMP '2026-11-16 05:00:00';
```

---

## 4. Partitioning Traps That Sink Senior Candidates

1. **Trap 1: Partitioning by Mutable `updated_at` Timestamp:**
   * If a row is inserted on Nov 1 (`updated_at = Nov 1`) and updated on Nov 10 (`updated_at = Nov 10`), partitioning by `updated_at` forces the engine to delete the row from the Nov 1 partition and re-insert it into the Nov 10 partition, breaking idempotency and historical cohort queries. **Always partition by immutable event creation time (`occurred_at_utc` or `created_at_utc`).**
2. **Trap 2: Multi-Column Cartesian Partitioning (`PARTITION BY (date, country, device_type)`):**
   * `365 days * 200 countries * 5 devices = 365,000 partitions/year`. Low-traffic countries (e.g., Luxembourg on Smart TV) end up with thousands of `4 KB` Parquet files. Keep coarse dimensions like `country` and `device_type` in **Clustering / Z-Ordering**, never in directory partitions!
3. **Trap 3: Non-Sargable Predicates (`WHERE YEAR(event_ts) = 2026` or `CAST(event_ts AS STRING) LIKE '2026-11%'`):**
   * Wrapping the partition column in a function prevents the query optimizer from pushing down partition bounds on many engines. Always write half-open range predicates (`>=` and `<`).

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Suppose 95% of your BI dashboards filter by `merchant_local_date`, while your table is partitioned by `occurred_at_utc`. How do you ensure queries filtering on `merchant_local_date` still get fast partition pruning?"**
   * **Expected Insight:** Even if the physical partition key is `days(occurred_at_utc)`, Iceberg/Parquet metadata still tracks **min/max statistics on `merchant_local_date` inside every data file**! Because any local calendar date globally falls within `[local_date - 14 hours, local_date + 38 hours]` in UTC, file-level min/max pruning on `merchant_local_date` automatically skips all files outside those 2–3 UTC days! Alternatively, if local-date queries are 95% of the workload, partition directly by `merchant_local_date` and accept that ingestion writes concurrently into 2 adjacent date partitions across timezone boundaries.
2. **Top-Up 2: "How do you prevent an analyst from accidentally running `SELECT * FROM fct_ad_impressions` without a date filter and burning $5,000 in warehouse compute?"**
   * **Expected Insight:** Enable **`require_partition_filter = TRUE`** (BigQuery) or table governance policies / semantic layer default date filters, plus enforce query timeout & byte-scanned resource monitors at the warehouse/catalog layer.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Calculates target Parquet file size (`256MB–1GB`)** from daily volume to choose `HOUR` vs. `DAY` vs. `MONTH` partitioning. | Blindly writes `PARTITION BY (event_date, country_id, product_category)` on every table. |
| **Uses Apache Iceberg Hidden Partitioning** (`days(occurred_at_utc)`) and explains partition evolution without table rewrites. | Creates redundant `year`, `month`, `day` string columns and forces analysts to filter all three in `WHERE` clauses. |
| **Recognizes non-sargable predicate traps** (`DATE(CONVERT_TIMEZONE(...))`) that silently disable partition pruning. | Assumes any `WHERE` clause mentioning a timestamp column automatically prunes partitions. |