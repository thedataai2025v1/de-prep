# Module 2: Gathering Business Requirements
## Lesson 2.5 — Defining Latency Requirements

> *"Latency is the silent architect of your data model."* Before you choose your tables, your grain, or even your database — you must know **how fast the data needs to move**. Latency requirements don't just affect your pipeline technology; they reshape your entire data model.

---

### 1. What Is Latency — And Why It Matters

**Latency** is the elapsed time between when a data event occurs in the source system and when it becomes available for consumption in your data model.

| Term | Definition |
|---|---|
| **Latency** | Time from event occurrence → available for query |
| **Throughput** | Volume of data processed per unit time (rows/sec, events/min) |
| **Freshness** | How "current" the data appears to end users (a function of latency + schedule) |
| **SLA** | The contracted/committed maximum latency the business requires |

**Why latency matters in data modeling:**

Latency is not just an infrastructure concern. It directly dictates:

- **Fact table grain:** Real-time systems insert row-by-row; batch systems load millions at once.
- **Load strategy:** Incremental inserts vs. full rebuilds vs. micro-batch merges.
- **Schema design:** Accumulating snapshots, periodic snapshots, and transaction fact tables all have different latency characteristics.
- **Partitioning and indexing:** Real-time models need more aggressive indexing to serve low-latency queries on growing tables.
- **Tool selection:** Kafka + Flink vs. Airflow + dbt vs. Snowpipe + Snowflake — each maps to a latency tier.

> **Interview insight:** Interviewers use your latency answer to gauge whether you think in **end-to-end systems** or just **schema diagrams**. Seniors connect latency → architecture → model design → trade-offs. Juniors draw tables and hope the pipeline team handles the rest.

---

### 2. Latency Tiers

Every business requirement falls into one of three latency tiers. Knowing which tier you're in is the single most important decision before designing the model.

| Tier | Latency Target | Architecture Pattern | Typical Tools |
|---|---|---|---|
| **Real-time** | < 1 second | Event streaming, change data capture (CDC), incremental inserts | Kafka, Kinesis, Pulsar, Flink, Debezium, ksqlDB |
| **Near-real-time** | < 5 minutes | Micro-batch, frequent materialized view refresh, incremental ELT | Airflow (5-min intervals), dbt + incremental models, Flink micro-batch, Snowpipe Streaming |
| **Batch** | Hourly to daily | Full or incremental ETL/ELT pipelines, overnight batch jobs | Airflow, dbt, dbt Cloud, Spark, AWS Glue, Snowflake Tasks |

#### Visual: Latency Spectrum

```
   Real-Time              Near-Real-Time              Batch
   (sub-second)           (seconds – minutes)         (hours – days)

   │──────────│──────────│──────────│──────────│──────────│──────────│
   0s         1s         5s        30s       5min     1hr      24hr

   Fraud detection   Live dashboards    Executive reports
   Anomaly alerts    Recommendation     Monthly financials
   Stock trading     Supply chain       Year-end audits
```

---

### 3. How Latency Requirements Drive Architecture Decisions

Your latency target dictates the entire pipeline architecture — which in turn constrains your data model design.

#### Real-Time Architecture (< 1 second)

```
Source Systems (DB, App, IoT)
        │
        ▼
   CDC / Event Capture  ←  Debezium, Kafka Connect, Kinesis Data Streams
        │
        ▼
   Stream Broker  ←  Kafka, Kinesis, Pulsar
        │
        ▼
   Stream Processor  ←  Flink, ksqlDB, Spark Structured Streaming
        │
        ▼
   Incremental Load  ←  Row-by-row inserts into target
        │
        ▼
   Real-Time Data Model  ←  Fact table with append-only inserts
```

**Model implications:**
- Fact tables must support **append-only** inserts — no full rebuilds.
- Primary key conflicts are rare; surrogate keys are often auto-incremented at ingestion.
- Dimension changes require **Type 2 SCD in real-time** (open/close effective dates on the fly) — complex and expensive.
- Aggregations are often pushed to **materialized views** or **streaming aggregates** (e.g., KSQL tumbling windows).
- Partitioning by time is critical — but each partition stays "open" and grows continuously.

#### Near-Real-Time Architecture (< 5 minutes)

```
Source Systems
        │
        ▼
   Staging Layer  ←  Raw landing zone (S3, ADLS, GCS)
        │
        ▼
   Micro-Batch Pipeline  ←  Airflow DAG runs every 5 minutes
        │
        ▼
   Incremental Transform  ←  dbt incremental model (merge/insert)
        │
        ▼
   Curated Layer  ←  Materialized views refreshed every 10–15 minutes
        │
        ▼
   Serving Layer  ←  Query engine / data warehouse
```

**Model implications:**
- **Incremental models** are natural: `WHERE event_time > LAST_RUN_TIME`.
- **MERGE statements** (UPSERT) work well — batch is small enough to be fast.
- Periodic snapshot fact tables can refresh every 15–30 minutes.
- Materialized views can be refreshed on a schedule without unacceptable staleness.

#### Batch Architecture (Hourly/Daily)

```
Source Systems
        │
        ▼
   Staging / Raw Zone  ←  Full daily extract or CDC log
        │
        ▼
   ELT Pipeline  ←  dbt, Airflow, nightly job
        │
        ▼
   Full Rebuild or Incremental  ←  Depends on volume
        │
        ▼
   Aggregated / Summary Layer  ←  Pre-computed cubes, rollups
        │
        ▼
   Data Warehouse / Data Mart  ←  Star schema, serving BI tools
```

**Model implications:**
- **Full rebuilds** are feasible and often simpler than incremental logic.
- **Accumulating snapshot fact tables** work naturally — you can close out entire events daily.
- **Periodic snapshot fact tables** are refreshed daily at a point-in-time.
- Pre-aggregated summary tables can be rebuilt from scratch each night.
- Dimension tables (SCD Type 2) are processed in bulk — effective date ranges are clean.

---

### 4. How Latency Affects Data Model Design

Latency doesn't just change your pipeline — it changes your **tables**. Here's how each Kimball fact table type behaves under different latency tiers.

#### Fact Table Types and Latency Compatibility

| Fact Table Type | Real-Time | Near-Real-Time | Batch |
|---|---|---|---|
| **Transaction Fact** | ✅ Natural fit — each event is an insert | ✅ Works well with incremental MERGE | ✅ Standard nightly load |
| **Periodic Snapshot** | ⚠️ Hard — must poll frequently | ✅ Good — refresh every 15–30 min | ✅ Natural — daily snapshot |
| **Accumulating Snapshot** | ❌ Difficult — closing events arrive late | ⚠️ Feasible — update within minutes | ✅ Natural — close at end of day |

#### Real-Time Fact Tables: Incremental Inserts

```sql
-- Real-time: append-only, no backfills
INSERT INTO fact_clickstream (
    event_id, user_key, session_key, page_key, event_time, event_type
)
SELECT
    gen_random_uuid(),  -- surrogate key generated at ingestion
    user_key,
    session_key,
    page_key,
    event_time,
    event_type
FROM staging_clickstream_events
WHERE event_time > LAST_CONSUMED_TIMESTAMP;  -- exactly-once semantics via offsets
```

- **No MERGE, no UPDATE** — just INSERT. Simpler, faster, more reliable.
- **Surrogate keys** are generated at ingestion time (UUID, sequence, or hash).
- **Slowly Changing Dimensions** are handled differently: often a "current-only" dimension table updated in real-time, or deferred to a nightly batch reconciliation job.

#### Batch Fact Tables: Full Rebuilds

```sql
-- Batch: full rebuild of the fact table nightly
-- Step 1: Truncate and reload
TRUNCATE TABLE fact_daily_sales;

INSERT INTO fact_daily_sales
SELECT
    d.date_key,
    p.product_key,
    c.customer_key,
    r.region_key,
    COUNT(*) AS order_count,
    SUM(o.quantity * o.unit_price) AS revenue,
    AVG(o.quantity * o.unit_price) AS avg_order_value
FROM staging_orders o
JOIN dim_date d ON o.order_date = d.date
JOIN dim_product p ON o.product_id = p.product_id
JOIN dim_customer c ON o.customer_id = c.customer_id
JOIN dim_region r ON c.region_id = r.region_id
WHERE o.order_date = CURRENT_DATE - INTERVAL '1 day'
GROUP BY d.date_key, p.product_key, c.customer_key, r.region_key;
```

- **Full rebuilds are cheap** when data is bounded (e.g., one day's worth).
- **No incremental logic** — no tracking `LAST_CONSUMED_TIMESTAMP`.
- **Accumulating snapshots work naturally:** update all open events daily, close completed ones.

#### Accumulating Snapshots: A Latency-Sensitive Design

Accumulating snapshot fact tables track events with a defined lifecycle (e.g., order → shipped → delivered).

```
fact_order_lifecycle
├── order_key            (PK)
├── order_date_key       (FK → dim_date)
├── customer_key         (FK → dim_customer)
├── order_time           (timestamp)
├── shipped_date_key     (FK → dim_date, NULL until shipped)
├── shipped_time         (timestamp, NULL until shipped)
├── delivered_date_key   (FK → dim_date, NULL until delivered)
├── delivered_time       (timestamp, NULL until delivered)
├── canceled_date_key    (FK → dim_date, NULL until canceled)
└── status               ('open', 'shipped', 'delivered', 'canceled')
```

| Latency Tier | How You Handle It |
|---|---|
| **Real-time** | ⚠️ Every status change triggers a row UPDATE. At high volume, this creates contention. Better to use a transaction fact + real-time materialized view. |
| **Near-real-time** | ✅ MERGE every 5–15 minutes: update rows where status changed. Manageable. |
| **Batch** | ✅ Natural fit. Daily job: identify all orders that advanced a stage, UPDATE those rows. Close out completed orders. |

> **Key Insight:** Accumulating snapshots are **the most latency-sensitive** fact table pattern. They require UPDATE semantics — which real-time streaming engines (Kafka → Kappa architectures) struggle with.

---

### 5. Trade-offs: Latency vs. Cost vs. Complexity

You can't optimize for latency alone. Every improvement in freshness costs money and complexity.

| Dimension | Real-Time | Near-Real-Time | Batch |
|---|---|---|---|
| **Infrastructure Cost** | Highest (stream brokers + processors + always-on) | Moderate (frequent micro-batch runs) | Lowest (scheduled, burst compute) |
| **Engineering Complexity** | High (exactly-once, schema evolution, late data) | Medium (incremental logic, idempotency) | Low (simple ETL/ELT, well-understood) |
| **Data Consistency** | Risk of partial updates, stale snapshots | Good with proper idempotency | Best — full rebuilds are clean |
| **Debuggability** | Difficult (distributed, stateful processing) | Moderate (checkpointing helps) | Easy (deterministic, reproducible) |
| **Time to Market** | Slowest (new tech, new team skills) | Moderate | Fastest (familiar tools, patterns) |

#### The Latency-Cost Curve

```
Cost & Complexity

    High  │        Real-Time
          │       ╱
          │      ╱
    Med   │     ╱   Near-Real-Time
          │    ╱
          │   ╱
    Low   │  ╱   Batch
          │ ╱
          └─────────────────────→ Freshness
               Poor      Good     Excellent

    The "Sweet Spot" for most businesses is near-real-time.
```

**Decision Framework:**

```
Is sub-second latency REQUIRED for the business decision?
    └── YES → Real-time architecture
    └── NO → Is sub-5-minute freshness needed for user trust?
              └── YES → Near-real-time
              └── NO → Batch is sufficient (and much cheaper)
```

> **Interview pro tip:** When an interviewer describes a system, ask: *"What's the cost of stale data?"* If stale data costs millions (fraud), go real-time. If stale data is annoying (dashboard off by 3 hours), batch is fine. The best engineers optimize for the **right** latency, not the fastest.

---

### 6. Common Latency Requirements by Domain

Here are real-world examples of how different business domains approach latency.

| Domain | Typical Requirement | Why | Example Scenario |
|---|---|---|---|
| **Fraud Detection** | Real-time (< 1 sec) | Must block transactions BEFORE they complete | A credit-card transaction is scored against known fraud patterns as the authorization request arrives |
| **Cybersecurity** | Real-time (< 1 sec) | Threats must be detected and blocked instantly | DDoS detection, anomalous login patterns, API abuse |
| **Stock Trading** | Real-time (< 100 ms) | Every millisecond affects profit | Price feeds, order matching, arbitrage detection |
| **Recommendation Systems** | Near-real-time (< 5 min) | User preferences change; model must adapt quickly | "Users who watched this also watched..." — refreshed as viewing data accumulates |
| **Live Dashboards** | Near-real-time (< 5 min) | Executives need current data during meetings | Sales pipeline, supply chain status, campaign performance |
| **Customer Service** | Near-real-time (< 5 min) | Agents need current customer context | Support agent sees latest order status, return history |
| **Supply Chain** | Near-real-time to batch | Tracking shipments, inventory; some delay acceptable | Warehouse inventory updated every 15 minutes; daily replenishment orders |
| **Executive Dashboards** | Batch (daily) | Strategic metrics don't need to be live | Monthly revenue, quarterly growth — daily is more than enough |
| **Financial Reporting** | Batch (daily/monthly) | Regulated, audited, reconciliation-heavy | GAAP-compliant revenue recognition, monthly close |
| **Marketing Attribution** | Batch (daily) | Complex joins, multi-touch; freshness is low priority | "This week's campaigns generated $X in revenue" — computed overnight |
| **Product Analytics** | Near-real-time to batch | Feature usage, funnel analysis | Daily funnel is fine; A/B test results can be near-real-time |

#### Domain-Specific Nuances

**Fraud Detection Model Design:**
```
fact_transaction_events          ← Real-time append-only
├── transaction_id (PK)
├── customer_key (FK)
├── merchant_key (FK)
├── amount
├── timestamp
├── risk_score         ← Computed by ML model at ingestion
└── fraud_flag         ← 'approved', 'flagged', 'blocked'

dim_fraud_rules              ← Updated occasionally (not real-time)
├── rule_id
├── rule_type
├── threshold
└── effective_date
```

**Recommendation System Model Design:**
```
fact_user_interactions         ← Near-real-time incremental
├── interaction_id (PK)
├── user_key (FK)
├── content_key (FK)
├── interaction_type   ('view', 'like', 'share', 'watch')
├── duration_seconds
└── event_time

agg_user_content_scores        ← Materialized view, refreshed every 5 min
├── user_key
├── content_key
├── weighted_score     ← Computed from interactions + content features
└── last_refreshed
```

**Executive Dashboard Model Design:**
```
fact_daily_summary             ← Full rebuild nightly
├── date_key (FK)
├── region_key (FK)
├── channel_key (FK)
├── total_revenue
├── total_orders
├── total_customers
├── avg_order_value
└── conversion_rate
```

---

### Interviewer Questions You'll Face

| # | Question | What They're Testing |
|---|----------|---------------------|
| 1 | *"We have a fraud detection use case. How does that change your data model compared to a standard analytics model?"* | Understanding that real-time demands append-only facts, ML-ready features, and minimal joins at query time |
| 2 | *"Our stakeholders want their dashboard to be 'always up to date.' How do you interpret and handle that requirement?"* | Pushing back: "always" is impossible — you define acceptable SLAs (e.g., < 5 min freshness) and justify the architecture choice |
| 3 | *"We're using Kafka for event ingestion. How does that affect how you design the fact tables in our data warehouse?"* | Connecting streaming to modeling: append-only facts, surrogate key generation at ingestion, handling late-arriving events |
| 4 | *"We need to track the full lifecycle of an order (placed → shipped → delivered) and query it in near-real-time. What's your approach?"* | Recognizing accumulating snapshots are hard in near-real-time; suggesting a hybrid: transaction fact for each event + a materialized view for lifecycle state |

---

### Questions YOU Should Ask the Interviewer

| # | Question | What It Reveals |
|---|----------|-----------------|
| 1 | *"What's the maximum acceptable delay between an event occurring and it being queryable?"* | You're thinking about SLAs — not just schema |
| 2 | *"What's the business cost if the data is stale for an hour? A day?"* | You connect technical choices to business impact |
| 3 | *"Are there any downstream systems that depend on this data being fresh in real-time?"* | You're mapping the full data consumer landscape |
| 4 | *"Do any of the consumers need to make decisions within seconds of an event, or is analysis always retrospective?"* | You distinguish real-time decisioning from analytics |
| 5 | *"How much engineering effort is the team willing to invest on the pipeline side — do we have streaming expertise, or should we keep it batch?"* | You're pragmatic about team skills and capacity |
| 6 | *"Is there a phased approach — can we start with daily batch and incrementally improve freshness?"* | You're thinking iteratively, not boil-the-ocean |

---

### Top-Up Questions (Senior-Level Probes)

These demonstrate deep operational experience. Use 1–2 to elevate your interview.

1. *"How do you handle late-arriving data in a near-real-time pipeline? Do we have a reconciliation batch job that backfills gaps?"*
2. *"What's our strategy for exactly-once vs. at-least-once processing? Does the data model need to support idempotent merges?"*
3. *"How do we reconcile streaming and batch dimensions? If our dimension tables are updated nightly, how do real-time fact inserts reference current dimension state?"*
4. *"Do we have a data quality SLA tied to latency — for example, 'data must be fresh within 5 minutes AND complete to 99.9%'? What happens when freshness and completeness conflict?"*
5. *"What's the rollback strategy? If a real-time pipeline produces bad data, can we recover quickly, or do we need the batch pipeline as a safety net?"*

---

### Practice Exercise

**Scenario:** You're interviewing for a Data Engineer role at *PayShield*, a fintech startup. The interviewer says:

> *"We process online payment transactions. We need to detect fraudulent transactions in real-time, but we also need to produce daily revenue reports for our finance team. Currently, all our data goes into a nightly batch warehouse and we're losing money to fraud because we can't stop transactions fast enough."*

**Your task:** Design a data architecture that satisfies both requirements. Walk through:

1. **Latency classification:** What parts need real-time vs. batch?
2. **Architecture diagram:** How do the two pipelines coexist?
3. **Data model decisions:** What does each fact table look like and how is it loaded?
4. **Trade-off discussion:** What complexity are you introducing, and why is it worth it?

<details>
<summary>💡 Click to reveal a model answer</summary>

#### 1. Latency Classification

| Component | Latency Tier | Justification |
|---|---|---|
| Fraud scoring on incoming transactions | **Real-time** (< 1 sec) | Must block before authorization completes |
| Fraud alert dashboard for investigators | **Near-real-time** (< 5 min) | Needs current view of flagged transactions |
| Daily revenue reports for finance | **Batch** (daily) | Financial accounting is inherently period-based |
| Monthly fraud statistics / trend analysis | **Batch** (daily/monthly) | Analytical, not operational |

#### 2. Architecture Diagram

```
                    ┌─────────────────────┐
   Payment Gateway  │    Kafka / Kinesis   │
                    │  (transaction events)│
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐  ┌────────────┐  ┌──────────────┐
        │ Fraud ML  │  │ Flink /    │  │ S3 / Landing  │
        │ Scorer    │  │ ksqlDB     │  │ Zone (raw)   │
        │ (sub-sec) │  │ (aggre-    │  │              │
        └────┬─────┘  │  gates)    │  │              │
             │        └────┬───────┘  │              │
             │             │          │              │
             ▼             ▼          ▼              │
   ┌─────────────────┐ ┌──────────────────┐         │
   │ fact_transactions│ │fact_fraud_alerts │         │
   │ (real-time       │ │(near-real-time   │         │
   │  append-only)    │ │ micro-batch)     │         │
   └────────┬────────┘ └────────┬─────────┘         │
            │                    │                    │
            │      ┌─────────────┼────────────────────┤
            │      │             ▼                    │
            │      │     Airflow Nightly Job          │
            │      │   (pulls from raw + real-time)   │
            │      │             ▼                    │
            │      │  ┌──────────────────────┐        │
            │      │  │fact_daily_revenue     │        │
            │      │  │(full rebuild nightly) │        │
            │      │  └──────────────────────┘        │
            │      └──────────────────────────────────┘
            │
            ▼
   Real-time API / Dashboard
   (query fact_transactions)
```

#### 3. Data Model Decisions

**fact_transactions (Real-Time, Append-Only):**
```sql
CREATE TABLE fact_transactions (
    transaction_id     VARCHAR(36) PRIMARY KEY,
    customer_key       INT,
    merchant_key       INT,
    amount             DECIMAL(12,2),
    currency           CHAR(3),
    transaction_time   TIMESTAMP,
    risk_score         DECIMAL(5,4),
    fraud_decision     VARCHAR(20),  -- 'approved', 'flagged', 'blocked'
    event_source       VARCHAR(50)   -- for tracking
);
-- Loaded: Kafka → Flink → INSERT INTO (one row per event)
```

**fact_fraud_alerts (Near-Real-Time, Incremental Merge):**
```sql
CREATE TABLE fact_fraud_alerts (
    alert_id           BIGINT GENERATED ALWAYS AS IDENTITY,
    transaction_id     VARCHAR(36),
    customer_key       INT,
    risk_score         DECIMAL(5,4),
    alert_time         TIMESTAMP,
    investigation_status VARCHAR(20) DEFAULT 'open',
    investigator_key   INT,
    resolution_notes   TEXT
);
-- Loaded: Micro-batch every 5 minutes (Airflow + MERGE)
```

**fact_daily_revenue (Batch, Full Rebuild):**
```sql
CREATE TABLE fact_daily_revenue (
    date_key           INT,
    merchant_key       INT,
    currency           CHAR(3),
    transaction_count  BIGINT,
    gross_revenue      DECIMAL(18,2),
    fraud_blocked      BIGINT,
    fraud_loss         DECIMAL(18,2),
    net_revenue        DECIMAL(18,2)
);
-- Loaded: Nightly full rebuild from fact_transactions
```

#### 4. Trade-off Discussion

| Concern | Response |
|---|---|
| **Complexity of two pipelines** | Acceptable: fraud loss is expensive; the real-time pipeline pays for itself by blocking fraudulent transactions. The batch pipeline remains simple and familiar. |
| **Data duplication** | fact_transactions is ingested in real-time AND used by the nightly batch job. We accept this duplication because the two pipelines serve fundamentally different consumers with different latency needs. |
| **Dimension table consistency** | Real-time inserts use current dimension state (lookup at ingestion time). Nightly batch reconciles any dimension changes. A common anti-pattern to discuss in interviews. |
| **Cost** | Kafka + Flink adds ~$3K/month vs. batch-only at ~$500/month. If we're losing $50K/month to fraud, the ROI is clear. |

> **Key takeaway for the interview:** You demonstrated that you can **split a single business problem into distinct latency tiers**, design each with the appropriate architecture and model, and justify the trade-offs in business terms. That's exactly what senior data engineers do.

</details>

---

### Quick Reference: Latency Decision Matrix

| If the business says... | You ask... | Then you model... |
|---|---|---|
| "We need to stop fraud as it happens" | "What's the max time between transaction and decision?" | Real-time append-only facts + ML features |
| "The dashboard should always be up to date" | "How quickly does a stakeholder notice if it's stale?" | Near-real-time materialized views + micro-batch |
| "We need monthly financials" | "Is this for compliance or exploration?" | Batch full-rebuild summary tables |
| "We want to recommend products in real-time" | "Can recommendations be slightly stale, or must they be live?" | Near-real-time: user interaction facts + scored materialized views |
| "We need to track order status at all times" | "Is this for customer-facing (self-serve) or internal ops?" | Internal: accumulating snapshot (batch update). Customer-facing: append-only + live query |

---

> **Final Thought:** In every data modeling interview, latency is your **first architectural constraint** — not the last afterthought. Frame it early, justify it clearly, and let it drive every design decision that follows.

---

### Related Lessons

- ⬅️ [Lesson 8: Analyzing Query Patterns](./08-analyzing-query-patterns.md) — query patterns influence refresh cadence
- ➡️ [Lesson 10: Data Volume and Scalability Considerations](./10-data-volume-and-scalability-considerations.md) — volume and latency trade-offs
- ➡️ [Lesson 11: Data Retention and Historical Data Management](./11-data-retention-policies-and-historical-data-management.md) — retention windows along with latency tiers
