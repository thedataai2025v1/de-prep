# Master Principal Data Modeling Cheat Sheet & Interview-Day Playbook

> **How to Use This Guide:** Review this document 60 minutes before any Staff/Principal Data Engineering interview. It condenses all **7 Modules (36 Lessons)** into instant pattern-matching tables, copy-ready SQL idioms, physical storage rules, and the exact talking points hiring committees look for in an L6/L7 debrief.

## 0. Principal Bar-Raiser Review & Hardened Edge Cases

Upon peer-reviewing this playbook against real L6/L7 (Staff/Principal) interview loops at Tier-1 tech and fintech companies, **five high-signal additions** separate a "Good Senior" from an "Unanimous Principal Hire":

1. **Cross-Dialect Bitwise Precision:** In Snowflake (`BITAND` / `BITSHIFTLEFT`), BigQuery (`&` / `<<` on `INT64`), and Spark SQL (`&` / `shiftleft`), 32-bit signed `INT` overflows on bit 31. Always type bitmasks as **`BIGINT` (`INT64`)** (supporting up to 60-day or 64-day rolling windows cleanly without sign-bit bugs!).

2. **ML Feature Store Point-in-Time (PIT) Correctness:** Principal loops increasingly test whether your warehouse model can feed ML training sets **without target/future leakage**. Always emphasize `ASOF JOIN` (natively supported in Snowflake, Databricks, DuckDB, ClickHouse) or pre-resolved SCD2 `_sk` joins so training features reflect exact entity state *prior to* the prediction timestamp.

3. **Shift-Left Data Contracts (Producer-to-Gold):** Don't treat schema evolution purely as a reactive warehouse problem. Mention **Schema Registry (Protobuf/Avro) + CI/CD Breaking-Change Gates** at the microservice producer boundary so upstream teams cannot silently break downstream Silver/Gold contracts.

4. **Domain Ownership & Data Mesh Boundaries:** In large enterprises (Amazon, Uber, Stripe), a single monolithic warehouse team becomes a bottleneck. Explain how **Conformed Dimensions (`dim_customer`, `dim_date`, `dim_geo`)** act as the federated governance spine connecting domain-owned Gold data products (`Payments Domain`, `Fulfillment Domain`, `Growth Domain`).

5. **FinOps / Query Cost Observability:** Tie every physical design choice (partitioning, Liquid/Z-order clustering, HLL sketches, MoR deletion vectors) back to **unit economics** (bytes scanned per query, warehouse credits saved, and SLA tail latency `P95`/`P99`).

## 1. The 45-Minute Execution Checklist

```
[00 - 06 min] 1. CLARIFY & CONTRACT
                 [ ] Consumers (BI, Product, Finance, ML Training/Serving)?
                 [ ] Top 3-5 KPIs + edge cases (refunds, cancels, timezones, FX)?
                 [ ] Historical truth (SCD1 vs SCD2 'as-was' vs 'as-is')?
                 [ ] Volume/day, freshness SLA, late-arrival window, retention & GDPR?
                 [ ] State explicit Design Contract & get verbal buy-in!

[06 - 12 min] 2. BUS MATRIX & GRAIN LOCK
                 [ ] Draw Enterprise Bus Matrix (Processes x Conformed Dimensions).
                 [ ] Write GRAIN in bold at the top of every fact table ("1 row per ...").
                 [ ] Separate incompatible grains (Header vs Item; Event vs Lifecycle).

[12 - 26 min] 3. SCHEMA & PRODUCTION DDL
                 [ ] Fact tables first: Degenerate keys, Role-playing date FKs, SCD2 _sk + natural _id,
                     additive numerators/denominators, occurred_at_utc + ingested_at_utc.
                 [ ] Dimensions: Flat (no snowflaking), -1/-2 sentinel rows, SCD2 validity bounds,wflaking), -1/-2 sentnds
                 high-churn attributes, Junk Dimensions for flag soup.

[26 - 33 min] 4. PROACTIVE SQL DRY-RUN
                 [ ] Write 1-2 queries solving the hardest KPI (SCD2 join, cohort retention, ROAS).
                 [ ] Verify zero Fan Traps (1:N multiplication) and zero Chasm Traps (Fact x Fact).

[33 - 41 min] 5. PHYSICAL LAYOUT- 41 min] 5. PHYSICAL LAYOUT, MAINTENANCE & OPERATIONAL RESILIENCE
            file      [ ] Partitioning (256MB-1GB Parquet target) + Z-Order/Liquid Clustering + Bloom Filters.
                 Idempotent MERGE (pa Late-arriving data.letion vectors) + Mul] Com PII Vault /paction & Snapshot  Write-Audit-Publish (WAP) backfills.

[41 - 45 min] 6. TRADE-OFFS, COST. FINOPS TRADE-OFFS & EVOLUTION
           e FinOps controls & deferred opti aggregates, HLL sketches, streaming serving).

```

## 2. Instant Fact Table Pattern Selector

| If the Business Question Sounds Like... | Choose This Fact Archetype | Exact Grain | Mutability & Write Pattern | 
| ----- | ----- | ----- | ----- | 
| *"How much did we sell / charge / stream / click today?"* | **Transaction Fact (`fct_*`)** | 1 row per atomic event (`line_item_id`, `balance_txn_id`) | **Immutable Append-Only** (reversals/refunds are new signed rows). | 
| *"What was our inventory, bank balance, MRR, or active user count at day/month end?"* | **Periodic Snapshot Fact (`fct_*_daily` / `_monthly`)** | 1 row per `(entity_sk, period_sk)` | **Incremental State Roll-Forward** (`T-1` state + `T` delta). | 
| *"Where are orders/loans/tickets getting stuck in our multi-step funnel, and what is our SLA breach rate?"* | **Accumulating Snapshot (`fct_*_lifecycle`)** | 1 row per workflow instance (`order_id`, `ticket_id`) | **Bounded `MERGE`** (`WHERE is_terminal_state = FALSE`). | 
| *"Which promoted items had ZERO sales? / Which drivers were online with ZERO trips?"* | **Coverage Factless Fact (`fct_*_coverage`)** | 1 row per valid eligible combination | Append-only schedule; queried via `LEFT ANTI JOIN` against sales/trips. | 
| *"How do we attribute revenue/watch-hours across multiple diagnoses, genres, or joint account owners?"* | **Weighted Bridge Table (`bridge_*`)** | 1 row per `(group_sk, member_sk)` with `weight` summing to `1.0` | Keyed by deterministic group hash; also keep `primary_member_sk` on fact. | 

## 3. Instant Dimension &D & ttern Selector

| Challenge | Pattern to Deploy | Why It Wins at Scale | 
| ----- | ----- | ----- | 
| Need historical attribute at event time (`as-was`) AND current attribute (`as-is`) without rewriting Iceberg files | **SCD Type 7 (Dual Keys on Fact)** | Fact stores both `entity_sk` (historical SCD2 row) and `entity_id` (joins `WHERE is_current = TRUE`). Zero multi-row dimension updates! | 
| EntM-row `dim_user` has attributes changing daily (`follower_tier`, `churn_risk_band`, `ltv_decile`) | **Mini-Dimension (SCD Type 4)** | Bucket volatile attributes into a static \~500-row table `dim_user_tier_mini` and stamp `tier_sk` onto facts at write time. | 
|  Prevents SCD2 row explosion! | 
| 15 boolean/low-cardinality flags cluttering a 20B-row fact table | **Junk Dimension (`dim_*_flags`)** | Combines flags into one compact table (`< 5,000` rows) keyed by a single 4-byte `INT` FK. | 
| High-cardinality operational ID (`order_id`, `pod_id`, `trace_id`) with no standalone attributes | **Degenerate Dimension** | Stored directly on the fact table; avoids building a 500M-row 1:1 dimension table. | 
| Variable-depth recursive hierarchy (Org chart, Category tree) | **Closure Bridge Table (`bridge_hierarchy`)** | `(ancestor_sk, descendant_sk, depth_levels)` enables single-join subtree rollups without recursive CTEs. | 
| Fact arrives 15 minutes before new entity CDC record lands in dimension | **Inferred Member (Stub Row)** | Insert placeholder row (`is_inferred_member = TRUE`, `valid_from = '1900-01-01'`), assign valid `_sk`, overwrite Type 1 when CDC arrives. | 
| GDPR "Right to be Forgotten" on a 500TB immutable Parquet lakehouse | **PII Vault + Crypto-Shredding** | Isolate PII in `dim_user_pii` encrypted with per-user KMS DEK; destroying DEK renders 7 years of archives unreadable without rewriting files. | 

## 4. Production SQL Idioms (Copy-Ready Mental Templates)

### Idiom A: Safe Drill-Across Across Two Fact Tables (Zero Chasm Trap)

```
WITH spend AS (
    SELECT date_sk, channel_sk, SUM(spend_usd) AS spend
    FROM fct_ad_spend GROUP BY 1, 2
),
orders AS (
    SELECT ordered_date_sk AS date_sk, channel_sk, COUNT(DISTINCT customer_id) AS new_custs
    FROM fct_orders WHERE is_first_order = TRUE GROUP BY 1, 2
)
SELECT s.date_sk, s.channel_sk, s.spend / NULLIF(o.new_custs, 0) AS cac_usd
FROM spend s
LEFT JOIN orders o ON s.date_sk = o.date_sk AND s.channel_sk = o.channel_sk;

```

### Idiom B: 28-Day Bitmask Single-Partition DAU / WAU /  (`BIGINT` Safe)MA-- active_28d_bitmask typed as BIGINT (INT64) to avoid 32-bit sign-bit overflow SELECTECT     activity_date_sk,     COUNT(CASE WHEN (active_28d_bitmask & 1) > 0 THEN 1 END      )   AS dau,     COUNT(CASE WHEN (active_28d_bitmask & 127) > 0 THEN 1 E      ND) AS w -- 2^7 - 1au,     COUNT(CASE W (active_28d_bitmask & 268435455)ask > 0 THEN 1 E    AS   -- 2^28 - 1mau FROM fct_user_daily_activity WHERE activity_date_sk = 20261115 GROUP BY 1; 

### Idiom Idempotent SCD Type 2 `tracked_attr_hash` Guard

```
-- Only trigger SCD2 version split when tracked business columns actually change:
WHERE cur.entity_id IS NULL
   OR XXHASH64(src.tier, src.country, src.commission_pct) <> cur.tracked_attr_hash
   OR cur.is_inferred_member = TRUE

```

### Idiom D:age Point-in-T (PIT)ime ML Train Featureing Join (`ASOF JOIN`-- Zero future data leakage when building ML training labels from SCD2 dimensions: SELECT     lbl.loan_application_id,     lbl.defaulted_within_90d_label,     dim.credit_score_band,     dim.account_tenure_bucket FROM ml_training_labels lbl ASOF JOIN dim_applicant_history dim   MATCH_CONDITION (lbl.submitted_at_utc >= dim.valid_from_utc)   ON lbl.applicant_id = dim.applicant_id; 

## 5. Physical Storage & Lakehouse Sizing Quick-Reference

* **Target Parquet File Size:** `256 MB – 1 GB` compressed (`ZSTD`).

* **Partition Granularity Rule:**

  * `< 500 MB/day` $\rightarrow$ `MONTH(event_ts)` or `YEAR(event_ts)`

  * `500 MB – 50 GB/day` $\rightarrow$ `DAY(event_ts)` (Iceberg Hidden Partitioning: `days(occurred_at_utc)`)

  * `50 GB – 5 TB/day` $\rightarrow$ `HOUR(event_ts)`

  * `> 5 TB/day` $\rightarrow$ `HOUR(event_ts)` + `bucket(16, entity_id)`

* **Clustering /der / Hilbert / Liquid Clustering:** Best for 2–4 orthogonal high-cardinality filter keys (`(merchant_id, customer_id, city_id)`).

  * **Parquet Bloom Filters:** Enable on unsorted high-cardinality lookup strings (`order_id`, `trace_id`, `promo_code`).

* **Lakehouse Maintenance SLAs (Iceberg / Delta):**

  * **Compaction (`rewrite_data_files`):** Hourly/daily bin-pack + sort to ele `LAST_VALUE`.

  * Never store pre-divided ratios (`CTR`, `AOV`, `Take Rate`) in fact/aggregate tables—store constituent numerators & denominators.

  * Store **HyperLogLog (HLL) sketches** in aggregate/MV tables so `COUNT(DISTINCT)` remains algebraically mergeable and incremental-MV compatible.

## 6. Top 1 aggregate tables to makeal Curveball Ans**"Kimball vs. Data Vault**  across rollups.

## 6. Top 10$w$ Layer them: Bronze (raw append-only), Silver (Data Vault / canonical deduplicated entities when integrating multi-ERP sources), Gold Core (Kimball Star Schema for governed semantic truth), Gold Serving (OBT + messy HLL aggregates for sub-second BI/ML).

1. **"How do upstream microservices guarantee zero lost/phantom events into your warehouse?"** $\rightarrow$enny drift across billions of rows and accelerates CPU vectorization.

2. **"How do you prevent `MERGE` write amplification on a 50TB table?"** $\rightarrow$ Partition by immutable creation date, bound target scan to trailing open window (`WHERE created_date >= CURRENT_DATE - 14 AND is_terminal_state = FALSE`), and enable Iceberg/Delta Merge-on-Read deletion vectors.

3. **"Late transaction arrives after Finance closed last month's books?"** $\rightarrow$ Store both `event_date_sk` (cohort truth) and `accounting_date_sk` (`GREATEST(event_date, open_period_start)` for Finance immutability).

4. **"Celebrity / Hot-Key Skew during joins?"** $\rightarrow$ Denormalize top slice attributes (`tier_sk`, `country`) onto facts, isolate/salt `NULL`/`UNKNOWN` sentinel keys, and use two-stage salted pre-aggregation (`MOD(HASH(viewer_id), 64)`).

5. **"Clock skew between microservices causing missed SCD2 joins?"** $\rightarrow$ Set the initial version's `valid_from_utc = '1900-01-01'` and use half-open `[valid_from_utc, valid_to_utc)` intervals.

6. **"Non-sargable predicate trap?"** $\rightarrow$ Never wrap partition columns in functions (`WHERE DATE(CONVERT_TIMEZONE(...)) = ...`); convert local day bounds to a UTC half-open range (`>= utc_start AND < utc_end`) so partition pruning fires.

7. **"How do you backfill 18 months of Gold data safely?"** $\rightarrow$ Write-Audit-Publish (WAP) using Apache Iceberg branches (`audit_branch`), automated reconciliation assertions, and atomic branch fast-forward.

8. **"Why do declarative Materialized Views fall back to full recomputation?"** $\rightarrow$ Non-algebraic Lakehouse from the same Kafka schema: Flink $\rightarrow$ Pinot/ClickHouse (OBT) for 10s Ops, and Kafka $\rightarrow$ Iceberg Silver/Gold Star Schema for Finance.