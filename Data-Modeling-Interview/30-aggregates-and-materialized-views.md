# Lesson 30 — Aggregates and Materialized Views

## Learning Objectives
- Understand when and why to pre-compute aggregates
- Design aggregate tables at different granularities
- Compare materialized views vs materialized tables
- Implement refresh strategies for aggregate data
- Evaluate the trade-offs of denormalization for performance

---

## Why Pre-Compute Aggregates?

Raw fact tables can be extremely large (billions of rows). Common dashboard queries that aggregate this data from scratch on every run waste compute and time.

### The Problem
```sql
-- Dashboard query on a 1B-row raw events table:
SELECT
    DATE(event_ts)   AS event_date,
    event_type,
    COUNT(*)         AS event_count,
    AVG(duration_ms) AS avg_duration
FROM events
WHERE event_ts >= '2024-01-01'
GROUP BY DATE(event_ts), event_type;
```

This scans the entire table every time. With partitioning, it scans multiple partitions. Still expensive.

### The Solution: Pre-computed Aggregate Table
```sql
-- Pre-computed once per day, queried instantly:
SELECT event_date, event_type, event_count, avg_duration
FROM events_daily_summary
WHERE event_date >= '2024-01-01';
```

Scans a small aggregate table (365 rows × N event types) instead of 1B rows.

---

## Aggregate Table Granularities

### Level 1: Hourly Aggregates
```sql
-- Granularity: (hour, event_type)
-- Rows per year: 24 × 365 × ~10 event types = ~87,600 rows
CREATE TABLE events_hourly_summary (
    event_hour    TIMESTAMP,
    event_type    VARCHAR(50),
    event_count   BIGINT,
    avg_duration  DECIMAL(10,2),
    p95_duration  DECIMAL(10,2)
);
```

**When to use:** Real-time dashboards, monitoring, alerting

### Level 2: Daily Aggregates
```sql
-- Granularity: (date, event_type)
-- Rows per year: 365 × ~10 event types = ~3,650 rows
CREATE TABLE events_daily_summary (
    event_date    DATE,
    event_type    VARCHAR(50),
    event_count   BIGINT,
    avg_duration  DECIMAL(10,2),
    total_revenue DECIMAL(12,2)
);
```

**When to use:** Daily reporting, standard dashboards

### Level 3: Weekly / Monthly Aggregates
```sql
-- Granularity: (week_start, event_type)
-- Rows per year: 52 × ~10 event types = ~520 rows
CREATE TABLE events_weekly_summary (
    week_start   DATE,
    event_type   VARCHAR(50),
    event_count  BIGINT,
    avg_duration DECIMAL(10,2)
);
```

**When to use:** Executive reports, trend analysis

### Aggregate Hierarchy
```
Raw events (1B rows)
    ↓  (compute hourly)
events_hourly_summary (87K rows)
    ↓  (compute daily)
events_daily_summary (3.6K rows)
    ↓  (compute weekly)
events_weekly_summary (520 rows)
    ↓  (compute monthly)
events_monthly_summary (120 rows)
```

Each level feeds the next. You can roll up from raw → hourly → daily → weekly → monthly, or compute each level independently from the raw data.

---

## Materialized Views vs Materialized Tables

### Materialized Views (Database-Managed)

```sql
-- PostgreSQL example
CREATE MATERIALIZED VIEW events_daily_summary AS
SELECT
    DATE(event_ts) AS event_date,
    event_type,
    COUNT(*) AS event_count,
    AVG(duration_ms) AS avg_duration
FROM events
GROUP BY DATE(event_ts), event_type;

-- Refresh manually
REFRESH MATERIALIZED VIEW events_daily_summary;

-- Or schedule with pg_cron
-- SELECT cron.schedule('0 * * * *', 'REFRESH MATERIALIZED VIEW events_daily_summary');
```

**Pros:**
- Automatic query rewrite (some databases redirect queries to the MV)
- Single source of truth — the MV definition is the schema
- Easy to maintain

**Cons:**
- Refresh can lock the view (depending on the database)
- Not all databases support concurrent refresh
- Limited control over storage/partitioning of the result

### Materialized Tables (Manually Managed)

```sql
-- Two-table pattern: write to staging, swap in
CREATE TABLE events_daily_summary_stg AS
SELECT
    DATE(event_ts) AS event_date,
    event_type,
    COUNT(*) AS event_count,
    AVG(duration_ms) AS avg_duration
FROM events
WHERE event_ts >= '2024-06-01'  -- just this month
GROUP BY DATE(event_ts), event_type;

-- Atomic swap (BigQuery-style)
CREATE OR REPLACE TABLE events_daily_summary AS
SELECT * FROM events_daily_summary_stg;
```

**Pros:**
- Full control over storage, partitioning, clustering
- No locking — use staging table + atomic swap
- Can append incrementally

**Cons:**
- Manual refresh logic
- Must manage schema evolution yourself

---

## Refresh Strategies

### Full Refresh
```sql
-- Recompute the entire aggregate from scratch
-- Simple, correct, but slow for large tables
INSERT OVERWRITE events_daily_summary
SELECT DATE(event_ts), event_type, COUNT(*), AVG(duration_ms)
FROM events
GROUP BY DATE(event_ts), event_type;
```

**When to use:** Small to medium tables, or when correctness is critical

### Incremental Refresh
```sql
-- Only compute new data since last refresh
INSERT INTO events_daily_summary
SELECT DATE(event_ts), event_type, COUNT(*), AVG(duration_ms)
FROM events
WHERE event_ts >= last_refresh_timestamp
GROUP BY DATE(event_ts), event_type;
```

**When to use:** Large tables with continuous incoming data

### Incremental with Deduplication
```sql
-- For idempotent refresh (safe to re-run):
INSERT INTO events_daily_summary (event_date, event_type, event_count, avg_duration)
SELECT
    DATE(event_ts) AS event_date,
    event_type,
    COUNT(*) AS event_count,
    AVG(duration_ms) AS avg_duration
FROM events
WHERE DATE(event_ts) >= '2024-06-01'  -- today only
GROUP BY DATE(event_ts), event_type
ON CONFLICT (event_date, event_type)
DO UPDATE SET
    event_count = EXCLUDED.event_count,
    avg_duration = EXCLUDED.avg_duration;
```

**When to use:** Pipelines that may re-run (retry-safe)

---

## Denormalization for Performance

### The Denormalized Aggregate Table
```sql
-- Instead of joining multiple aggregate tables:
-- dim_events_daily joins dim_date joins dim_event_type

-- Pre-join everything:
CREATE TABLE events_daily_denorm (
    event_date    DATE,
    day_of_week   VARCHAR(10),
    month_name    VARCHAR(10),
    event_type    VARCHAR(50),
    category      VARCHAR(50),
    event_count   BIGINT,
    avg_duration  DECIMAL(10,2),
    total_revenue DECIMAL(12,2)
);
```

**Trade-off:**
- ✅ Faster reads (no joins needed)
- ❌ Wider rows, more storage
- ❌ Harder to maintain (schema changes cascade)

**When to use:** Dashboard tables where read performance matters more than storage cost

---

## Platform Implementation

### BigQuery
```sql
-- Materialized view with incremental refresh
CREATE OR REPLACE MATERIALIZED VIEW events_daily_summary
OPTIONS (
    enable_refresh = TRUE,
    refresh_interval_minutes = 60
) AS
SELECT
    DATE(event_ts) AS event_date,
    event_type,
    COUNT(*) AS event_count,
    AVG(duration_ms) AS avg_duration
FROM events
GROUP BY DATE(event_ts), event_type;
```

### Snowflake
```sql
-- Materialized view with automatic maintenance
CREATE MATERIALIZED VIEW events_daily_summary AS
SELECT
    DATE(event_ts) AS event_date,
    event_type,
    COUNT(*) AS event_count
FROM events
GROUP BY DATE(event_ts), event_type;

-- Snowflake automatically maintains this as the base table changes
```

### Databricks
```python
# Delta Live Tables or simple notebook job
spark.sql("""
    CREATE OR REPLACE TABLE events_daily_summary AS
    SELECT
        DATE(event_ts) AS event_date,
        event_type,
        COUNT(*) AS event_count
    FROM events
    GROUP BY DATE(event_ts), event_type
""")

# Schedule with Delta Live Tables or Airflow
```

---

## Design Decisions

### When NOT to Pre-Compute
- When queries are rare (compute on-demand is fine)
- When the raw table is small enough that aggregating on-the-fly is fast
- When the aggregation logic changes frequently (maintaining stale aggregates is worse than not having them)

### What to Pre-Compute
Ask these questions:
1. **How often is this query run?** — Frequently → pre-compute
2. **How many rows must it scan?** — Billions → pre-compute
3. **What's the latency requirement?** — Sub-second → pre-compute
4. **How complex is the aggregation?** — Simple COUNT/SUM → pre-compute easily; complex window functions → harder

---

## Interview Dialogue

### Interviewer: "How do you handle a dashboard that needs to load in under 2 seconds from a 5B-row table?"

**Candidate:**
> "I'd pre-compute the aggregates at the granularity the dashboard needs. If it shows daily metrics, I'd build a daily summary table — 365 rows per metric, not 5B rows.
>
> I'd set up a scheduled job that refreshes the aggregate table (e.g., every hour or daily, depending on how fresh the data needs to be). The dashboard queries the small summary table, which returns results instantly.
>
> For platforms that support it (BigQuery, Snowflake), I'd use materialized views with automatic refresh. For others (Databricks), I'd use a staging table + atomic swap pattern to avoid locking.
>
> I'd also consider a multi-level aggregation: raw → hourly (for real-time dashboards) → daily (for standard reports) → monthly (for executive views). Each level serves a different consumer with appropriate latency."

### Interviewer: "What's the trade-off between materialized views and materialized tables?"

**Candidate:**
> "Materialized views are easier to maintain — the database manages the refresh, and the view definition is the source of truth. But they can lock during refresh, and you have less control over how the underlying data is stored.
>
> Materialized tables give you full control — you decide the partitioning, clustering, and refresh logic. The staging + swap pattern avoids locks. But you write and maintain the refresh logic yourself.
>
> My rule of thumb: use materialized views when the database supports concurrent refresh and the aggregation is straightforward. Use materialized tables when you need fine-grained control over storage or the refresh logic is complex (e.g., incremental with deduplication)."

---

## Key Takeaways

| Concept | Rule of Thumb |
|---|---|
| Pre-compute frequently queried aggregates | If a query runs often on large data, pre-compute it |
| Match granularity to consumer needs | Dashboards → daily; monitoring → hourly; executives → monthly |
| MV vs Materialized Table | MV for simplicity; table for control |
| Incremental refresh for large tables | Only compute new data, not the whole thing |
| Denormalize for read speed | When join overhead exceeds storage cost |

---

## Related Lessons
- [Lesson 28 — Date-Based Partitioning](./28-date-based-partitioning.md)
- [Lesson 29 — Indexing Strategies](./29-indexing-strategies.md)
- [Lesson 26 — Multi-Fact Query Strategies](./26-multi-fact-query-strategies.md)
- [Lesson 09 — Defining Latency Requirements](./09-defining-latency-requirements.md)
- [Lesson 10 — Data Volume and Scalability Considerations](./10-data-volume-and-scalability-considerations.md)
