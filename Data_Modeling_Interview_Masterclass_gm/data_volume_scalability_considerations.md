# Lesson 10: Data Volume & Scalability Considerations

> **Principal Mentor Note:** A schema that runs in 400ms on 10 million rows can completely collapse at 100 billion rows—causing OOM (Out-of-Memory) shuffle spills, multi-hour `MERGE` jobs, skewed joins, and runaway cloud bills. In every Principal interview, you must do **back-of-the-envelope capacity math** in under 60 seconds and explicitly design your tables for **cardinality skew, key width, file pruning, and write amplification**.

---

## 1. Back-of-the-Envelope Capacity Math (60-Second Interview Habit)

Always anchor your physical modeling decisions on concrete numbers:

| Metric | Example Calculation (Consumer App: 50M DAU) | Architectural Implication |
| :--- | :--- | :--- |
| **Daily Event Volume** | `50M DAU * 40 events/user/day = 2B rows/day` | Cannot run full-table scans or unpartitioned `MERGE`s. Must partition by date/hour. |
| **Daily Storage (Compressed Parquet)** | `2B rows * ~120 bytes/row (ZSTD) ≈ 240 GB/day` | `~87 TB/year`, `~435 TB` over 5 years. Must tier cold data (>1 yr) and prune columns aggressively. |
| **Dimension Scale (`dim_user`)** | `300M total registered users` (`50M` active/day) | Too large to broadcast-join (`> 10GB` in memory) in Spark/Trino! Requires **Shuffle Hash Join** or **Bucketed/Co-located Join**. |
| **Dimension Scale (`dim_country`)** | `245 rows` (`< 50 KB`) | Fits easily in worker RAM; always **Broadcast-Joined** (`MapJoin`) with zero shuffle. |

---

## 2. Four Major Scalability Bottlenecks in Data Models (And How to Solve Them)

### Bottleneck 1: Data Skew ("The Celebrity / Hot Key Problem")
* **The Scenario:** In an e-commerce or social media warehouse (`fct_orders` or `fct_engagements`), 99.9% of merchants/creators have `< 1,000` rows/day, while 5 mega-merchants (Amazon Retail, Nike, Taylor Swift) have `50,000,000` rows/day.
* **What Breaks:** When joining `fct_orders` to `dim_merchant` on `merchant_id`, the distributed engine hashes `merchant_id` to assign reduce tasks. One unlucky worker node receives all 50M rows for Nike, spills to disk, and stalls the entire job for 2 hours (**Straggler Task**).
* **Principal Solutions:**
  1. **Exclude or Isolate Anonymous/Null Keys:** Never leave unattributed rows as `merchant_id = NULL` or `'UNKNOWN'` before a join—all NULLs hash to a single partition! Filter or salt sentinel keys (`UNKNOWN_0` .. `UNKNOWN_15`).
  2. **Key Salting for Skewed Joins:** Append a random salt (`0..N-1`) to the hot key in the fact table (`CONCAT(merchant_id, '_', FLOOR(RAND()*16))`) and explode the hot key `16x` in the dimension table before joining, distributing Nike across 16 parallel tasks.
  3. **Adaptive Query Execution (AQE):** Mention leveraging Spark/Databricks/Snowflake AQE skew-join optimization alongside smart physical clustering.

### Bottleneck 2: High-Cardinality Dimension Explosion (The "2-Billion-Row SCD Type 2" Trap)
* **The Scenario:** You have `dim_user` with `500M` users, and a mobile app sends updated `last_ip_address`, `app_version`, and `last_active_ts` every single day. If you put those volatile attributes inside an **SCD Type 2** `dim_user`, every active user generates a new row *every day* (`500M` rows $\rightarrow$ `50B` rows in 3 months!).
* **Principal Solutions (Split by Volatility):**
  1. **Type 1 Overwrite for Operational Metadata:** Keep `last_active_ts` and `app_version` as **SCD Type 1** (or don't put them in `dim_user` at all—leave them on the event fact!).
  2. **Mini-Dimension (`dim_user_profile_tier`):** Extract fast-changing analytical attributes (e.g., `loyalty_tier`, `credit_score_band`, `marketing_segment`) into a separate low-cardinality **Mini-Dimension** (`5 tiers * 10 bands * 8 segments = 400 static rows` total!) and stamp `user_profile_sk` directly onto the fact table at transaction time. `dim_user` shrinks by 99% and never needs expensive SCD2 updates for tier changes!

### Bottleneck 3: Join Key Width & Surrogate Key Overhead
* **The Scenario:** Joining a `100B`-row fact table to 5 dimensions using 36-byte UUID strings (`'550e8400-e29b-41d4-a716-446655440000'`) vs. 8-byte `BIGINT` surrogate keys.
* **Physical Impact:**
  * Storage & Network Shuffle: `36 bytes` (plus 4-byte length prefix = 40 bytes) vs. `8 bytes` = **5x more RAM and network I/O** during hash table builds and shuffles across 5 foreign keys (`200 bytes/row` vs. `40 bytes/row` = `16 TB` extra shuffle on 100B rows!).
  * Dictionary Encoding: Sequential `BIGINT` keys compress dramatically better with delta/bit-packing encodings in Parquet/ORC than random UUIDs or SHA-256 hex strings.
* **Principal Rule:** Use `INT` (4 bytes, up to 2.1B) or `BIGINT` (8 bytes) surrogate keys in Gold Star Schemas whenever possible; if deterministic hashing is required for distributed parallel loads, use **64-bit integer hashes (`XXHASH64` / `FARM_FINGERPRINT`)** or raw `BINARY(16)` instead of 32/64-character `VARCHAR` hex strings!

### Bottleneck 4: Unbounded `MERGE` Write Amplification
* **The Scenario:** Updating a `50TB` Iceberg/Delta table (`fct_orders`) to mark `0.1%` of rows as `REFUNDED` or `DELIVERED`.
* **Principal Solutions:**
  1. **Partition-Bounded `MERGE`:** Always include a date range filter on the target table (`target.order_date >= CURRENT_DATE - 30`) so Iceberg/Delta only opens manifests for the last 30 days instead of 5 years.
  2. **Merge-on-Read (MoR) with Deletion Vectors:** Configure Iceberg v2/v3 or Delta Lake with **Deletion Vectors** (Roaring Bitmaps) so row updates write tiny position-delete files instead of rewriting entire 512MB Parquet data files on every run, deferring physical file rewrites to off-peak compaction jobs.
  3. **Immutable Adjustment Facts:** For accounting/financial corrections older than 30 days, **never `UPDATE` old partitions**—insert a new reversing/adjustment transaction row in today's partition!

---

## 3. Real Interview Questions & Model Answers

### Q1: "We are modeling ad impressions for an ad network processing 15 billion impressions per day (~1.5 TB/day compressed). Analysts need to query CTR and Spend by `campaign_id`, `advertiser_id`, `country`, and `device_type` over trailing 90-day windows. How do you design the table hierarchy so queries return in under 5 seconds without blowing up our cloud compute bill?"

**Principal-Level Answer:**
> *"At 15B rows/day, a 90-day scan touches **1.35 trillion rows (~135 TB)**. Running ad-hoc BI queries directly against the raw impression grain for every dashboard refresh will destroy both latency SLAs and compute budgets.*
>
> *I would implement a **3-tier hierarchical rollup architecture**:*
> 1. ***Tier 1: Atomic Impression Fact (`fct_ad_impressions`, 15B rows/day):** Stored in Apache Iceberg, partitioned by `HOUR(event_ts)` and clustered by `(campaign_id, user_id)`. Retained in hot/warm storage for **30 days** (then tiered to S3 Glacier) strictly for fraud forensics, attribution modeling, and ML training.*
> 2. ***Tier 2: Hourly Aggregate Fact (`agg_campaign_hourly`, ~50M rows/day — 300x compression):** Pre-aggregated at the grain of `(event_hour, campaign_sk, advertiser_sk, geo_sk, device_sk)`. Stores **additive constituent metrics** (`impression_count`, `click_count`, `spend_micros_usd`), **never pre-divided CTR**. Partitioned by `DATE(event_hour)` and clustered by `(advertiser_sk, campaign_sk)`. A 90-day query now scans only **4.5B rows (~150 GB)**—returning in ~2–3 seconds.*
> 3. ***Tier 3: Daily Executive Rollup (`agg_advertiser_daily`, ~2M rows/day):** Powers executive portals and advertiser billing dashboards with sub-second response times across multi-year horizons."*

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "What if analysts also want `COUNT(DISTINCT user_id)` (Unique Reach) in your `agg_campaign_hourly` rollup table?"**
   * **Expected Insight:** You cannot sum `unique_users` across hours or countries because the same user sees ads multiple times. Store a **HyperLogLog (HLL) binary sketch** (`hll_user_sketch`) in `agg_campaign_hourly`. HLL sketches are tiny (~1–4 KB per row), algebraically mergeable (`HLL_UNION`), and allow sub-second approximate distinct reach (`~99%` accuracy) across any arbitrary rollup of hours, campaigns, or countries!
2. **Top-Up 2: "How do you handle a dimension table that grows from 1M rows to 500M rows (e.g., `dim_url` or `dim_device_id`) and suddenly breaks your warehouse's broadcast join threshold?"**
   * **Expected Insight:** First, question whether a 500M-row high-cardinality identifier belongs in a traditional star dimension at all—often it is a **Degenerate Dimension** (keep `device_id` directly on the fact table if it has few attributes) or requires **Bucketed / Storage-Partitioned Joins (SPJ in Iceberg)** so both the fact and dimension are pre-partitioned by `HASH(device_id, 256)`, eliminating network shuffles completely during joins.

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Performs 60-second capacity math** (`rows/day * bytes/row * retention`) before choosing storage tiers and rollups. | Designs a single atomic table for 20B events/day with 10-year retention and no aggregation layer. |
| **Identifies data skew (hot keys / NULL keys)** and proposes salting, AQE, or isolation. | Assumes all `merchant_id` or `user_id` keys distribute uniformly across worker nodes. |
| **Uses Mini-Dimensions or Type 1 splits** to prevent SCD Type 2 explosion on 100M+ row dimensions. | Tracks every minor status ping as an SCD Type 2 row on a 500M-row user dimension. |