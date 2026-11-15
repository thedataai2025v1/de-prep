# Lesson 29: Indexing, Clustering, Z-Ordering & Bloom Filters

> **Principal Mentor Note:** Candidates coming from OLTP backgrounds (Postgres/MySQL) often suggest *"creating a B-tree secondary index on `customer_id`, `merchant_id`, and `status`"* on a 50-billion-row analytical fact table. In modern columnar cloud warehouses and lakehouses (Snowflake, BigQuery, Iceberg, Delta), **traditional B-tree row indexes do not exist** because maintaining point-to-row pointers across immutable Parquet/columnar blocks would destroy bulk write throughput. Instead, Principal Data Engineers optimize selective filtering and joins using **Columnar Sort/Cluster Keys, Multi-Dimensional Space-Filling Curves (Z-Order / Hilbert), Zone Maps (Min/Max Stats), Bloom Filters, and Storage-Partitioned Joins (SPJ)**.

---

## 1. OLTP Indexing vs. OLAP Data Skipping Mechanisms

| Mechanism | Where Used | How It Works Under the Hood | Best Query Pattern | Write / Maintenance Cost |
| :--- | :--- | :--- | :--- | :--- |
| **1. Linear Clustering / Sort Key** | Snowflake `CLUSTER BY (col1, col2)`, BigQuery `CLUSTER BY`, Iceberg `WRITE ORDERED BY` | Physically sorts rows inside micro-partitions/Parquet row groups by `(col1, col2)`. Narrowing `[min, max]` ranges in file footers allows the engine to skip 95%+ of blocks (**Zone Maps**). | Point lookups & range scans on the **leading cluster column** (`WHERE merchant_id = 'M123'`). | Moderate (requires periodic background re-clustering / compaction as new data arrives). |
| **2. Multi-Dimensional Clustering (Z-Order / Hilbert Curve)** | Databricks `ZORDER BY` / `CLUSTER BY` (Liquid), Apache Iceberg `sort_order` | Interleaves the binary bits of 2–4 columns into a single 1D space-filling curve so spatial locality is preserved across **all clustered columns simultaneously** (not just the leading column!). | Queries filtering on **either** `merchant_id` **OR** `customer_id` **OR** `city_id` independently. | Higher CPU during `OPTIMIZE` compaction, but eliminates leading-column bias. |
| **3. Bloom Filters** | Apache Iceberg (`write.parquet.bloom-filter-enabled.column.*`), Databricks, ClickHouse | Compact probabilistic bit-array stored per Parquet row group (~1–2 MB/file) that answers *"Is value `X` definitely NOT in this row group?"* with 0% false negatives. | High-cardinality random UUID/hash lookups (`trace_id`, `order_id`, `promo_code`) where sorting is impossible. | Very low (~1–2% storage overhead); zero runtime maintenance after write. |
| **4. Search Optimization / Inverted Secondary Index** | Snowflake Search Optimization Service (SOS), ClickHouse `tokenbf_v1` / `set`, Pinot Inverted Index | Builds separate persistent micro-partition lookup structures mapping high-cardinality values or substrings to specific micro-partitions. | Sub-second point lookups (`WHERE order_id = 'ORD-99182'`) on 100TB+ tables for customer support portals. | Dedicated background compute & storage cost (use only on high-value lookup columns). |
| **5. Storage-Partitioned Joins (SPJ / Bucketing)** | Apache Iceberg (`bucket(N, join_key)`), Spark/Hive Bucketing | Pre-hashes both the Fact table and large Dimension table into the exact same `N` buckets (`bucket(256, user_id)`). | Joining two massive tables (`50B` fact $\bowtie$ `300M` dimension) with **zero network shuffle**! | Both tables must share identical bucket count & hash transform. |

---

## 2. Linear Sort vs. Z-Order / Hilbert Curve: Visual Intuition

Suppose analysts query `fct_trips` either by **`city_id`** (Operations) OR by **`driver_id`** (Driver Payouts):

* **Linear Sort `(city_id, driver_id)`:**
  * File 1: `city_id = 1`, `driver_id = [100 .. 999,999]`
  * File 2: `city_id = 2`, `driver_id = [100 .. 999,999]`
  * *Result:* Filtering `WHERE city_id = 1` skips File 2 completely (100% pruned!). Filtering `WHERE driver_id = 550` **must read BOTH File 1 and File 2** (0% pruned!) because `driver_id = 550` exists in every city's min/max range.
* **Z-Order / Hilbert Clustering `(city_id, driver_id)`:**
  * Co-locates rows where *both* `city_id` and `driver_id` are close in 2D space.
  * *Result:* Filtering `WHERE city_id = 1` prunes ~85% of files, **AND** filtering `WHERE driver_id = 550` *also* prunes ~85% of files!

> **Principal Rule:** Do not Z-Order on more than **3–4 columns**—beyond 4 dimensions, the space-filling curve loses locality (**Curse of Dimensionality**) and degrades back to random distribution.

---

## 3. Production DDL: Combining Partitioning, Liquid/Z-Order Clustering & Bloom Filters

```sql
-- ============================================================================
-- HIGH-SCALE FACT TABLE OPTIMIZED FOR BOTH TIME SCANS & ENTITY POINT LOOKUPS
-- ============================================================================
CREATE TABLE gold.fct_orders (
    order_id                STRING NOT NULL,       -- High-cardinality point lookup key
    ordered_at_utc          TIMESTAMP NOT NULL,    -- Primary time scan key
    customer_id             STRING NOT NULL,       -- Frequent cohort/support lookup key
    merchant_id             STRING NOT NULL,       -- Frequent merchant portal filter key
    order_status            STRING NOT NULL,
    net_revenue_usd         DECIMAL(12,2) NOT NULL
)
USING iceberg
PARTITIONED BY (months(ordered_at_utc))            -- Coarse monthly partitions avoid small files
TBLPROPERTIES (
    -- 1. Enable Parquet Bloom Filters on high-cardinality point-lookup IDs
    'write.parquet.bloom-filter-enabled.column.order_id'    = 'true',
    'write.parquet.bloom-filter-max-bytes'                  = '1048576',
    -- 2. Configure Z-Order / Sort optimization during background compaction
    'write.distribution-mode'                               = 'range'
);

-- Configure multi-dimensional sort order in Iceberg metadata
ALTER TABLE gold.fct_orders WRITE ORDERED BY (merchant_id, customer_id, ordered_at_utc);
```

---

## 4. Optimizing Expensive Joins in Data Models

When an interviewer asks *"How do you optimize joins in your data model?"*, structure your answer around **eliminating shuffles**:

1. **Broadcast Hash Join (`Dimension < 100 MB`):**
   * Keep lookup dimensions (`dim_date`, `dim_promo`, `dim_device`, Junk & Mini-dimensions) compact so the optimizer broadcasts them to every worker node—**zero fact table shuffle**.
2. **Pre-Resolve Surrogate Keys at Write Time (Eliminate Inequality Joins):**
   * Never force BI queries to run `f.event_ts >= d.valid_from AND f.event_ts < d.valid_to`. Stamp `d.entity_sk` onto `f` during ETL so BI queries run integer **Equi-Joins** (`f.entity_sk = d.entity_sk`).
3. **Storage-Partitioned Join (SPJ) for Massive `Fact` $\bowtie$ `Fact` or `Fact` $\bowtie$ `300M-Row Dimension`:**
   * Partition both `fct_user_events` and `dim_user` using `bucket(256, user_id)` in Apache Iceberg. Spark/Trino reads matching bucket pairs locally on the same executor—**100% shuffle-free join at multi-terabyte scale**.

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Why do Parquet min/max Zone Maps fail completely on UUIDv4 strings (`'f47ac10b-58cc-...'`) even if you sort by them?"**
   * **Expected Insight:** Standard UUIDv4s are completely random, and while sorting makes their min/max contiguous, if they are *not* the leading sort key, every file's `[min, max]` spans `'0000...'` to `'ffff...'` (100% overlap!). Furthermore, **UUIDv7** (time-ordered UUIDs) preserves chronological monotonicity naturally so min/max stats prune time ranges even without sorting—and **Bloom Filters** solve non-leading point lookups on random UUIDv4s!
2. **Top-Up 2: "A Snowflake table `fct_events` is clustered on `(tenant_id, event_date)`, but queries filtering `WHERE event_date = CURRENT_DATE` are suddenly scanning 80% of the table. Why?"**
   * **Expected Insight:** Because `tenant_id` is the **leading** cluster key and has high cardinality (`50,000` tenants), rows for `CURRENT_DATE` are scattered inside every tenant's micro-partitions! Flipping the cluster key to `(event_date, tenant_id)`—or using multi-dimensional Z-Order/Hilbert clustering—ensures `CURRENT_DATE` micro-partitions are tightly grouped.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Explains Zone Maps (min/max pruning), Z-Order/Hilbert curves, and Bloom Filters** accurately for columnar lakehouses. | Proposes creating traditional OLTP B-tree indexes on a 50-billion-row Snowflake/Iceberg fact table. |
| **Uses Bloom Filters for un-sorted high-cardinality IDs** (`order_id`, `trace_id`) and Z-Order for 2–3 orthogonal filter columns. | Z-Orders on 12 columns at once (destroying spatial locality via the Curse of Dimensionality). |
| **Leverages Storage-Partitioned Joins (SPJ / Bucketing)** to eliminate shuffles when joining 100M+ row dimensions. | Leaves large dimension joins unbucketed, causing multi-terabyte network shuffles on every BI refresh. |