# Module 2: Gathering Business Requirements
## Lesson 2.6 — Data Volume & Scalability Considerations

> **Why this matters:** The single biggest mistake junior data modelers make is designing schemas for today's data — not tomorrow's. Scalability isn't just "tech stuff"; it starts the moment you sit down to gather requirements.

---

### 1. Why Estimating Data Volume Matters for Schema Design

Schema decisions at the micro level cascade into architecture at the macro level. Consider:

| Design Choice | Small Scale (< 100M rows) | Large Scale (> 1B rows) |
|---|---|---|
| Index strategy | Standard B-tree indexes | Clustered columns / bloom filters |
| Query patterns | Full table scans may work | Must avoid full scans; partition pruning required |
| Storage format | Row-based (e.g., InnoDB) | Columnar (e.g., Parquet, BigQuery) |
| Partitioning | Usually unnecessary | Mandatory for performance |
| Retention policy | "Store everything" | Tiered: hot, warm, cold, archive |

**The core principle:** *Knowing the data volume before you model lets you choose the right storage, partitioning, and query patterns from day one — not after a production meltdown.*

---

### 2. How to Estimate Volume from Business Requirements

The fundamental formula every data modeler must know:

```
Total Rows = Event Rate × Retention Period × Entity Multiplier
```

#### Worked Example: E-Commerce Clickstream

| Parameter | Value |
|---|---|
| Daily unique visitors | 500,000 |
| Average clicks per visitor | 12 |
| **Daily events** | **6,000,000** |
| Retention period | 2 years |
| **Total rows** | **6M × 730 = 4.38 Billion** |

#### Worked Example: IoT Sensor Data

```
Sensors: 10,000
Readings per sensor per hour: 60
Hours per day: 24
Retention: 90 days

Total = 10,000 × 60 × 24 × 90 = 129.6 Billion readings
```

> **Pro tip:** Always ask the business owner *what they do* with the data. If they only aggregate to the daily level, you may not need to store every individual event long-term.

#### Rule of Thumb for Row Size

Estimate the storage footprint:

| Data Type | Approx. Storage per Row |
|---|---|
| Simple metrics (IDs + 5 numeric columns) | 100–300 bytes |
| Rich events (with JSON payload) | 1–5 KB |
| Time-series with tags | 50–200 bytes |

```
Storage = Total Rows × Estimated Row Size × (1 + compression_ratio)

Example: 4.38B rows × 200 bytes × 0.3 (columnar compression)
= ~260 GB of actual storage
```

---

### 3. Volume-Based Design Decisions

Use this decision framework when you estimate the scale:

```
┌──────────────────────────────────────────────────────────┐
│                  VOLUME-BASED DECISION MATRIX            │
├─────────────┬────────────┬──────────────┬────────────────┤
│  Volume     │  Strategy  │  Key Actions │  Tech Choices  │
├─────────────┼────────────┼──────────────┼────────────────┤
│ < 100M rows │  Simple    │  Standard    │  RDBMS,        │
│             │  design    │  indexing,   │  row-based     │
│             │            │  basic PKs   │  storage       │
├─────────────┼────────────┼──────────────┼────────────────┤
│ 100M – 1B   │  Scale     │  Partition   │  Columnar,     │
│             │  up        │  by date,    │  partitioned   │
│             │            │  clustering  │  tables        │
├─────────────┼────────────┼──────────────┼────────────────┤
│ > 1B rows   │  Engineer  │  Tiering,    │  Data lake,    │
│             │  for scale │  aggregates, │  tiered        │
│             │            │  archiving   │  architecture  │
└─────────────┴────────────┴──────────────┴────────────────┘
```

#### Small Scale (< 100M Rows)
- **Schema:** Denormalized tables are often fine.
- **Indexing:** Standard primary keys, foreign keys, and a few targeted indexes.
- **Partitioning:** Rarely needed; the optimizer handles it.
- **Example:** A SaaS app with 50K users, ~1M events/month.

#### Medium Scale (100M – 1B Rows)
- **Schema:** Normalized core tables; denormalized for analytics.
- **Partitioning:** By date or tenant ID — mandatory for write-heavy workloads.
- **Clustering:** Sort the data physically for scan efficiency.
- **Example:** A retail platform with 20M daily transactions across 5 years.

#### Large Scale (> 1B Rows)
- **Schema:** Careful normalization with targeted denormalization for query paths.
- **Partitioning:** Multi-level — by date, then by category/region.
- **Aggregates:** Pre-compute daily/weekly summaries for dashboards.
- **Tiering:** Hot data in fast storage, cold data in archive.
- **Example:** A fintech platform processing 50M+ financial events daily for 10+ years.

---

### 4. Scaling Strategies

#### Strategy A: Partitioning by Date

```sql
-- BigQuery: Partition by date column
CREATE TABLE event_log (
  event_id   INT64,
  user_id    INT64,
  event_type STRING,
  event_date DATE
)
PARTITION BY event_date;

-- Query naturally prunes partitions:
SELECT * FROM event_log
WHERE event_date >= '2024-01-01'  -- Only scans 2024 partitions
  AND event_type = 'click';
```

> **Interview tip:** Emphasize that partitioning must align with your most common *filter* queries. Partitioning by `user_id` helps user-specific queries; partitioning by `date` helps time-range queries. You can't optimize for both equally.

#### Strategy B: Data Tiering (Hot, Warm, Cold)

```
┌──────────────────────────────────────────────┐
│           DATA TIERING ARCHITECTURE           │
├──────────────┬───────────┬──────────────────┤
│  Tier        │  Retention│  Storage/Access   │
├──────────────┼───────────┼──────────────────┤
│  🔥 HOT      │  30 days  │  SSD, in-memory, │
│              │           │  sub-second SLA  │
├──────────────┼───────────┼──────────────────┤
│  🌤️ WARM     │  1 year   │  Standard cloud  │
│              │           │  storage, sec SLA│
├──────────────┼───────────┼──────────────────┤
│  ❄️ COLD      │  1–7 years│  Archive (Glacier│
│              │           │  /Coldline),     │
│              │           │  hours to retreive│
├──────────────┼───────────┼──────────────────┤
│  🗑️ EXPIRED   │  0 days   │  Deleted /       │
│              │           │  anonymized      │
└──────────────┴───────────┴──────────────────┘
```

**Implementation approaches:**
- **Automated TTL:** BigQuery table expiration, Snowflake time travel limits.
- **ETL-based migration:** Nightly jobs moving data from hot to warm tables.
- **Storage policies:** AWS S3 lifecycle rules, Azure storage tiering.

#### Strategy C: Pre-Aggregated Tables

When raw data hits billions of rows, dashboards that aggregate on-the-fly become unusable:

```sql
-- Instead of querying 1B+ raw rows every time:
-- SELECT date, region, SUM(revenue), COUNT(DISTINCT user_id)
-- FROM transactions GROUP BY date, region;

-- Build a materialized summary daily:
CREATE OR REPLACE TABLE daily_revenue_summary AS
SELECT
  DATE(event_timestamp) AS report_date,
  region,
  SUM(amount)          AS total_revenue,
  COUNT(DISTINCT user_id) AS unique_users,
  COUNT(*)              AS total_events
FROM transactions
WHERE event_timestamp >= CURRENT_DATE - INTERVAL '1' DAY
GROUP BY 1, 2;
```

| Approach | Refresh Frequency | Storage Cost | Query Latency |
|---|---|---|---|
| Raw on-the-fly aggregation | Real-time | Low (one table) | Minutes on large tables |
| Materialized daily view | Once/day | Medium (raw + summary) | Sub-second |
| Materialized hourly view | Once/hour | Higher (more summary tables) | Sub-second |

#### Strategy D: Columnar Storage Selection

| Engine | Best For | Compression | Query Pattern |
|---|---|---|---|
| **Parquet** | Analytics, data lakes | 5–10× | Column subset scans |
| **ORC** | Hadoop ecosystem | 5–10× | Column subset scans |
| **Delta Lake** | ACID + analytics | 5–10× | Mixed read/write |
| **BigQuery** | Serverless analytics | Auto | Serverless SQL |
| **Snowflake** | Enterprise analytics | Auto | Serverless SQL |
| **InnoDB (row)** | OLTP, point lookups | Low | Primary key lookups |

---

### 5. Growth Rate Considerations

Your growth rate fundamentally changes the architecture:

| Growth Rate | Implication | Design Response |
|---|---|---|
| **Flat** (< 20% YoY) | Design for current + 30% buffer | Simple design; revisit in 2 years |
| **Moderate** (2× YoY) | Today's small table = tomorrow's large table | Design for medium scale from day one |
| **Aggressive** (10× YoY) | Every design decision must be future-proof | Start with large-scale architecture |
| **Unknown** | Business may not have visibility | Build modular design; add tiering later |

> **Real example:** A startup expects 10K daily events. At 10× YoY growth, year 3 sees 1M/day — your 3-year retention jumps from 11M to 1.1 billion rows. The schema designed in year 1 must survive or be replaceable.

**Questions to ask the business:**
- What's your projected growth rate for user base?
- How long do regulatory/compliance rules require us to keep data?
- Are there seasonal spikes (e.g., holiday shopping, tax season)?
- Is there a maximum "contracted" volume the system must handle?

---

### 6. Cost Implications of Volume

Data isn't free. Volume decisions are *budget* decisions.

#### Storage Costs (Approximate, 2024)

| Storage Tier | Cost per GB/Month | Example: 1 TB |
|---|---|---|
| Cloud hot storage (S3/BigQuery) | $0.020–$0.023 | $20–$23 |
| Cloud warm storage (S3 IA) | $0.010–$0.013 | $10–$13 |
| Cloud cold storage (Glacier) | $0.001–$0.004 | $1–$4 |
| On-premise SSD | $0.10+ (capex) | $100+ |

#### Compute Costs

```
BigQuery: ~$6.25 per TB scanned
→ 10 queries/day scanning 1 TB of raw data
→ 10 × $6.25 = $62.50/day
→ ~$22,812/year

→ 10 queries/day scanning 100 GB pre-aggregated summary
→ 10 × $0.625 = $6.25/day
→ ~$2,281/year

Pre-aggregation saves ~$20,000/year on this workload alone.
```

#### Cost-Optimization Checklist

| Optimization | Typical Savings | Effort |
|---|---|---|
| Columnar storage over row-based | 5–10× storage reduction | Medium |
| Partition pruning | 80–99% query cost reduction | Low |
| Pre-aggregated tables | 70–90% compute cost reduction | Medium |
| Data tiering (warm/cold) | 50–90% storage cost reduction | High |
| Compression (Snappy, Zstd) | 2–5× storage reduction | Low |
| TTL-based expiration | Eliminates unnecessary storage | Low |

---

### Interviewer Questions You'll Face

**Q1: "We're building an event tracking system that will ingest 50 million events per day. Walk me through how you'd design the schema considering scale."**

*Key points to hit:* Estimate total volume over retention, choose partitioning strategy, recommend columnar storage, discuss aggregation layers, mention tiering and cost.

**Q2: "How do you decide between normalizing and denormalizing when you're dealing with billions of rows?"**

*Key points:* Normalize for write efficiency and data integrity; denormalize for read performance. At scale, the answer is often *both* — normalized source + denormalized analytics tables built via pipeline.

**Q3: "The business says 'we'll figure out storage later.' How do you respond as a data modeler?"**

*Key points:* Push back respectfully. Explain that storage decisions *are* schema decisions. Partitioning, indexing, and data modeling choices made today cannot be easily undone. Offer a phased approach: design for the expected 3-year volume, with clear migration paths.

**Q4: "How do you estimate the data volume for a system you've never built before?"**

*Key points:* Break it down: entity count × events per entity per time unit × retention period. Validate with the business. Use order-of-magnitude estimates early (1M vs 1B matters more than 4.3M vs 4.7M). Build a volume spreadsheet.

---

### Questions YOU Should Ask the Interviewer

| Question | Why It Matters |
|---|---|
| "What's the expected daily event volume in year 1, year 2, and year 3?" | Reveals if they've thought about growth. |
| "What's the regulatory retention requirement for this data?" | Compliance often dictates minimum retention. |
| "What are the top 3 query patterns this data will support?" | Partitioning and indexing must align with queries. |
| "Is there a budget constraint on cloud storage or compute?" | Shows business awareness and drives tradeoff decisions. |
| "Have you built anything similar before? What volumes did you hit?" | May reveal past pain points you should avoid. |

---

### Top-Up Questions (Senior-Level Probes)

These are the follow-ups senior interviewers use to separate strong candidates from exceptional ones:

**S1: "You recommended partitioning by date, but some queries filter by user_id. How do you handle the trade-off between partition pruning and query flexibility?"**

*Strong answer:* Acknowledge the tension. Propose clustering within partitions, or secondary indexing strategies. Mention that the *dominant* query pattern should drive partitioning, and less frequent queries accept some penalty. In modern warehouses, consider materialized views for the alternative query pattern.

**S2: "How would your design change if we need to support real-time dashboards vs. only batch analytics?"**

*Strong answer:* Real-time needs a different tier: a low-latency store (e.g., ClickHouse, Druid, or a materialized view) fed by streaming ingestion. Batch can tolerate columnar storage with hourly/daily refresh. The schema may need to diverge — a unified raw layer feeds both a real-time layer and a batch analytics layer (a "lambda architecture" or "Kappa architecture" pattern).

**S3: "What happens when growth outpaces your initial estimate? Walk me through a data migration strategy."**

*Strong answer:* Design with an exit strategy. Keep the raw layer immutable — append only. Build transformations as separate layers (bronze → silver → gold). When scale demands change, modify the transformation layer, not the raw data. Use schema evolution tools (Avro, Protobuf, Delta Lake schema evolution). Run dual pipelines during transition.

**S4: "How do you quantify the ROI of pre-aggregation vs. just buying more compute?"**

*Strong answer:* Build a cost model: compare (storage cost of aggregates + refresh compute) vs. (on-demand compute for raw queries). Factor in latency requirements — pre-aggregation isn't just cheaper, it's often the *only* way to meet sub-second SLAs on large datasets. At scale, the savings are dramatic: 10× compute reduction typically outweighs 2–3× storage increase.

---

### Practice Exercise

**Scenario:** You're interviewing for a Data Engineer role at a health-tech startup.

> The company is building a patient monitoring platform. Each connected wearable device sends vitals (heart rate, blood oxygen, temperature) every 5 minutes. There are 50,000 active patients today, each wearing 2 devices. The company expects to double its patient base every year for the next 3 years. By law, patient data must be retained for 7 years after the last record. The clinical team wants to query individual patient histories (by patient_id) and also run population-level analytics (e.g., "average heart rate by age group, monthly"). The CTO wants to know if the data layer can scale without a complete rebuild.

**Your task (prepare a 2-minute answer):**

1. **Estimate the data volume** for year 1, year 3, and total over 7 years.
2. **Propose a schema design** that handles both patient-specific and population queries.
3. **Recommend a tiering strategy** considering the legal retention requirement.
4. **Identify one risk** in the growth plan and how you'd mitigate it.

---

<details>
<summary><strong>Suggested Answer (click to expand)</strong></summary>

**1. Volume Estimate:**
```
Year 1: 50,000 patients × 2 devices × 12 readings/hr × 24 hr × 365 days
       = 10,512,000,000 records (~10.5 billion)

Year 3: 200,000 patients × 2 devices × 12 × 24 × 365
       = 42,048,000,000 records (~42 billion)

7-year cumulative: ~100+ billion records (accounting for exponential growth)
At ~100 bytes/row with columnar compression (30%): ~300+ GB raw, ~90 GB compressed per year at year-3 rate
```

**2. Schema Design:**
- **Raw facts table:** Partitioned by `record_date`, clustered by `patient_id` (serves the dominant patient-history query).
- **Pre-aggregated monthly table:** Grouped by age_group, metric, month — refreshed monthly for population analytics.
- Both stored in columnar format (e.g., BigQuery, Snowflake, or Parquet on S3).

**3. Tiering Strategy:**
- **Hot (0–2 years):** Full-resolution data in primary warehouse for patient queries.
- **Warm (2–5 years):** Down-sampled data (e.g., hourly averages) in cheaper tier.
- **Cold (5–7 years):** Daily aggregates only, in archive storage — satisfies legal retention at minimal cost.
- **Expired (>7 years):** De-identified aggregates retained; individual records deleted per compliance.

**4. Risk & Mitigation:**
- **Risk:** The exponential growth means year 3 volume is 4× year 1. A schema designed for year 1 will break.
- **Mitigation:** Design the transformation layer (not just the raw layer) for year-3 scale from day one. Use a layered architecture (bronze/silver/gold) so the raw ingestion layer is append-only and the transformation logic can be modified without data migration.
</details>

---

### Key Takeaways

1. **Always estimate volume early** — it's the foundation of every design decision.
2. **Match your architecture to the volume tier** — simple tables for small data, engineered tiers for big data.
3. **Design for the 3rd year, not the 1st** — growth is exponential, not linear.
4. **Partition, aggregate, tier, and compress** — these are your four pillars of scale.
5. **Talk about costs** — data modelers who understand budget get hired over those who don't.

---

### Related Lessons

- ⬅️ [Lesson 9: Defining Latency Requirements](./09-defining-latency-requirements.md) — latency drives architecture scale
- ➡️ [Lesson 11: Data Retention and Historical Data Management](./11-data-retention-policies-and-historical-data-management.md) — how retention affects storage planning
- ➡️ [Lesson 14: Evolving Models Based on Changing Requirements](./14-evolving-models-based-on-changing-requirements.md) — adapting when scale changes
