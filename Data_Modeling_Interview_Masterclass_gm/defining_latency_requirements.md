# Lesson 9: Defining Latency Requirements (Batch vs. Micro-Batch vs. Streaming)

> **Principal Mentor Note:** Every order-of-magnitude reduction in data latency (Daily Batch $\rightarrow$ Hourly Micro-Batch $\rightarrow$ Sub-Minute Streaming $\rightarrow$ Sub-Second Real-Time) increases infrastructure cost, operational complexity, and schema rigidity by roughly **3x to 5x**. At the Principal level, your job is never to blindly propose Kafka + Flink for everything—it is to align **data freshness SLAs** with actual **business decision latency**, and design hybrid **Lambda/Kappa/Lakehouse** models where historical accuracy and low-latency visibility coexist without duplicating business logic.

---

## 1. The Latency-to-Architecture Spectrum

Before drawing tables, map the stakeholder's freshness requirement onto one of four concrete architectural tiers:

| Latency Tier | Freshness SLA | Typical Use Cases | Modeling & Storage Pattern | Primary Failure Modes & Traps |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 1: Daily / Multi-Daily Batch** | `6h – 24h` | Executive financial reporting, cohort retention, board KPIs, billing reconciliation. | Full Kimball Star Schema (SCD2, surrogate keys, cross-fact joins) in Snowflake / BigQuery / Iceberg. | Timezone cutoff drift (UTC vs. local midnight), late-arriving facts missing the daily window. |
| **Tier 2: Near-Real-Time Micro-Batch** | `5m – 60m` | Operational control towers (driver supply/demand), intraday marketing spend pacing, inventory alerts. | Incremental `MERGE` / append-only micro-partitions in Lakehouse (Iceberg/Delta) + Dynamic Tables / Materialized Views. | **Small file proliferation** (hundreds of tiny Parquet files per hour) and `MERGE` write amplification. |
| **Tier 3: Streaming Analytics (OLAP)** | `5s – 60s` | Live marketplace surge dashboards, live sporting event telemetry, QoS buffering monitors. | Pre-joined / flat **One Big Table (OBT)** or star-tree indexed rollups in **Apache Pinot, Apache Druid, or ClickHouse**. | Limited join support at query time (requires upstream stream enrichment in Flink) and weak SCD2 history support. |
| **Tier 4: Sub-Second Real-Time Serving** | `< 500ms` | Fraud scoring at checkout, dynamic pricing, real-time ML feature lookup. | Key-value / low-latency feature store (**Redis, DynamoDB, Cassandra, Feast**) fed by stateful Flink jobs. | Eventual consistency vs. historical training/serving skew if offline warehouse logic diverges from Flink. |

---

## 2. How Latency Requirements Dictate Your Data Model

Latency is not just an infrastructure choice—it **directly forces structural changes to your tables**:

### Structural Impact 1: Star Schema Joins vs. Upstream Stream Enrichment
* **In Daily Batch:** You can safely load `dim_customer` first, assign integer surrogate keys (`customer_sk`), and then load `fct_orders` by looking up `customer_sk`.
* **In Sub-Minute Streaming:** Waiting for slow dimension ETL jobs to finish before publishing `fct_orders` violates the latency SLA. You must:
  1. Either use **natural/durable business keys** (`customer_id`) or **deterministic hash keys** (`MD5/SHA256(customer_id)`) so `fct_orders` can be written immediately without blocking on dimension lookups, OR
  2. Perform **temporal stream enrichment** in Apache Flink (joining the `orders` Kafka topic against a compacted changelog topic of `customers` stored in RocksDB state) to stamp current dimension attributes directly onto the event (**Streaming OBT**).

### Structural Impact 2: SCD Type 2 vs. Append-Only State Logs
* Running `UPDATE` / `MERGE` to expire `valid_to` and set `is_current = FALSE` in an SCD Type 2 table every 30 seconds on an object-store lakehouse creates severe lock contention and metadata commit conflicts.
* **Principal Pattern:** In low-latency pipelines, write an **append-only versioned dimension log** (`dim_merchant_versions` with `version_ts`), and let a **View** or **Dynamic Table / Materialized View** compute `LEAD(version_ts) OVER (PARTITION BY merchant_id ORDER BY version_ts) AS valid_to` lazily at read/refresh time!

### Structural Impact 3: Event Time vs. Processing Time (Watermarks & Late Data)
Whenever latency drops below 24 hours, **out-of-order arrival** becomes your biggest correctness risk:
* `event_time` (`occurred_at_utc`): When the user actually tapped "Book Ride" on their phone (could be inside a subway tunnel with no signal).
* `ingestion_time` (`ingested_at_utc`): When Kafka/Bronze received the payload (45 minutes later).
* **Modeling Rule:** Always store **both** `occurred_at_utc` and `ingested_at_utc` on every fact table. Partition physical files by `DATE(occurred_at_utc)` (or `ingested_at_utc` for ultra-high volume with a secondary `occurred_at_utc` cluster key) and define an explicit **re-statement lookback window** (e.g., rolling 72-hour upsert window) so late events seamlessly backfill into their true historical date bucket.

---

## 3. The "Speed + Batch" Hybrid Architecture (Modern Kappa / Lakehouse Pattern)

When an interviewer says: *"Operations needs live trip counts within 30 seconds, AND Finance needs audited monthly revenue reconciliation with SCD Type 2 history,"* present this clean two-speed architecture:

```
                    +---> [Flink Enrichment] ---> [ClickHouse / Pinot (OBT)] (30s SLA: Live Ops Dashboards)
                    |
[Kafka Event Bus] --+
                    |
                    +---> [Bronze Iceberg] ---> [Silver Deduped] ---> [Gold Kimball Star Schema] (1h/Daily SLA: Finance & BI)
```

* **Why Interviewers Love This:** You avoid forcing a single database engine to do two incompatible jobs (sub-second streaming aggregations vs. multi-year SCD2 historical restatements), while guaranteeing **100% schema parity** because both paths consume the exact same canonical Avro/Protobuf event contract from Kafka.

---

## 4. Real Interview Questions & Model Answers

### Q1: "The VP of Operations asks for a real-time dashboard showing active orders, driver wait times, and SLA breaches updated every 10 seconds. Right now, our warehouse is a nightly dbt + Snowflake star schema. How do you evolve the data model?"

**Principal-Level Answer:**
> *"First, I would verify whether '10 seconds' is truly required for human operational intervention—if dispatchers actively re-route drivers within 60 seconds, sub-minute latency is justified; if managers only review shift performance hourly, a 15-minute micro-batch saves 70% of engineering cost.*
>
> *Assuming true 10-second operational dispatch is required, I would **not** run 10-second `MERGE` loops against our Snowflake Kimball star schema—that would cause massive warehouse credit burn and table lock contention. Instead, I would introduce a **Streaming Serving Mart** alongside our core warehouse:*
> 1. ***Canonical Event Stream:** Capture order state transitions (`CREATED`, `ASSIGNED`, `PICKED_UP`, `DELIVERED`) via CDC/Kafka.*
> 2. ***Stateful Stream Enrichment (Flink):** Enrich events in-flight with lightweight dimension context (`city_id`, `restaurant_tier`, `courier_vehicle_type`) from compacted dimension topics, emitting a flat, denormalized event (`obt_order_live_status`).*
> 3. ***Real-Time OLAP Store (Apache Pinot / ClickHouse):** Ingest `obt_order_live_status` into an upsert-enabled real-time table keyed by `order_id` with a 7-day TTL, serving live operational slice-and-dice queries in <100ms.*
> 4. ***Core Warehouse Reconciliation:** The same Kafka stream lands in our Lakehouse/Snowflake Bronze layer, feeding our nightly/hourly **Kimball Star Schema** (`fct_orders`, `fct_order_fulfillment`) for multi-year retention, SCD Type 2 history, and financial auditing."*

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "If you use Apache Iceberg or Delta Lake for 5-minute micro-batch ingestion, how do you prevent the Small File Problem from degrading query performance?"**
   * **Expected Insight:** Decouple write frequency from file layout optimization. Let the 5-minute streaming writer append small Parquet files with low commit overhead, and run an **asynchronous background compaction job** (`rewrite_data_files` / `OPTIMIZE`) every hour or day to bin-pack small files into target 256MB–512MB files and rewrite manifests, without blocking concurrent readers or writers.
2. **Top-Up 2: "How do you handle a metric like 'Active Sessions Right Now' in a streaming data model when users close the app without sending a `session_end` event?"**
   * **Expected Insight:** Use **Session Windows with Inactivity Gaps** (e.g., 30-minute inactivity timeout) in Flink/Spark Structured Streaming, emitting intermediate heartbeat updates every 1 minute for live dashboards and a finalized `fct_user_sessions` record once the watermark passes `last_event_ts + 30m`.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Challenges unrealistic latency demands** by tying freshness SLAs to actual human/system decision loops. | Agrees blindly to "sub-second real-time" for an annual executive financial report. |
| **Explains how low latency changes table structure** (flat OBT + natural/hash keys in streaming vs. normalized Star + integer SKs in batch). | Designs a 12-table 3NF/Star schema with multi-step SCD2 lookups and claims it will run at 1-second latency. |
| **Explicitly separates `occurred_at_utc` from `ingested_at_utc`** and defines watermark/lookback rules for late data. | Uses `CURRENT_TIMESTAMP()` at ingestion time as the business transaction timestamp. |