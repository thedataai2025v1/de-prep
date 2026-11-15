# Lesson 28 — Date-Based Partitioning

## Learning Objectives
- Understand why and when to partition large tables
- Choose a partition key that matches query patterns
- Compare partitioning strategies (date, monthly, hash) across platforms
- Explain partition pruning and its performance impact
- Handle multi-column partitioning and sub-partitioning
- Discuss columnar storage formats alongside partitioning

---

## What Is Partitioning?

Partitioning divides a large table into smaller, physically separate pieces (partitions) based on a column value. Each partition can be stored, scanned, and managed independently.

### Why Partition?
- **Partition pruning**: queries that filter on the partition column skip entire partitions — only the relevant ones are scanned
- **Faster maintenance**: vacuuming, rebuilding, or dropping old data works partition by partition
- **Parallelism**: cloud platforms process independent partitions in parallel
- **Cost savings**: scan only what you need (especially important in pay-per-GB systems like BigQuery)

### Partitioning vs Clustering
| | Partitioning | Clustering |
|---|---|---|
| Physical separation | Yes — separate files/directories | No — data is sorted within the same file |
| Pruning | Full partitions skipped | Zone maps / min-max indexes guide skips |
| Best for | High-cardinality time columns, large tables | Medium-cardinality columns within a partition |
| Granularity | Coarse (block-level) | Fine (within-block sorting) |

---

## Partitioning Strategies

### 1. Daily Partitioning

```sql
-- Table: rides (100M rows/day, queries filtered by date)
-- Partition key: ride_date

CREATE TABLE rides (
    ride_id         BIGINT,
    ride_date       DATE,
    driver_id       INT,
    passenger_id    INT,
    start_lat       FLOAT,
    start_lng       FLOAT,
    end_lat         FLOAT,
    end_lng         FLOAT,
    duration_secs   INT,
    fare_amt        DECIMAL(10,2)
)
PARTITION BY DATE(ride_date);
```

**Pros:**
- Most granular — skip exact days instantly
- Easy to drop old data: `DROP PARTITION '2024-01-01'`

**Cons:**
- Too many partitions can slow metadata lookups
- Overhead if queries frequently span multiple days

### 2. Monthly Partitioning

```sql
-- Table: sales (10B rows, monthly reporting)
-- Partition key: MONTH(sale_date)

PARTITION BY MONTH(sale_date);
```

**Pros:**
- Fewer partitions — cleaner metadata
- Good match for monthly reporting and roll-ups

**Cons:**
- Less precise pruning — scans entire month even for a single day

### 3. Yearly Partitioning

```sql
-- Table: logs (50B rows over 5 years)
-- Partition key: YEAR(event_timestamp)

PARTITION BY YEAR(event_timestamp);
```

**Pros:**
- Minimal metadata overhead
- Simple to understand and explain

**Cons:**
- Large partitions — limited pruning benefit unless queries span full years

### 4. Weekly Partitioning

```sql
-- Table: ad_impressions (aligned to weekly reporting cycles)
-- Partition key: WEEK(impression_date)

PARTITION BY WEEK(impression_date);
```

**When to use:** When the business naturally thinks in weeks (e.g., weekly ad spend reports).

### 5. Hash Partitioning

```sql
-- Table: users (partitioned evenly for parallel writes)
-- Partition key: HASH(user_id)

PARTITION BY HASH(user_id) PARTITIONS 16;
```

**When to use:** When data needs to be spread evenly (no natural time column), or when queries always filter on the hash key.

---

## Partition Pruning in Action

### Scenario: Daily Partitioned Rides Table
```sql
-- Query: Find all rides on a specific day
SELECT ride_id, driver_id, fare_amt
FROM rides
WHERE ride_date = '2024-06-15';
```
**Execution:** Only the `2024-06-15` partition is scanned. If the table has 365 partitions, 364 are skipped entirely.

### Scenario: Monthly Partitioned Sales Table
```sql
-- Query: Find sales in a single day
SELECT product_id, SUM(amount)
FROM sales
WHERE sale_date = '2024-06-15'
GROUP BY product_id;
```
**Execution:** The entire June 2024 partition is scanned — even though only one day's data is needed. **Fix:** cluster within the partition or switch to daily partitioning if day-level queries are frequent.

### Scenario: Hash Partitioned Users Table
```sql
-- Query: Look up a specific user
SELECT * FROM users WHERE user_id = 12345;
```
**Execution:** Only the hash bucket containing user_id=12345 is scanned (1 of 16 partitions).

---

## Multi-Column Partitioning (Sub-Partitioning)

Some platforms support partitioning by one column and sub-partitioning by another.

```sql
-- Example: Partition by year, sub-partition by month
PARTITION BY RANGE (YEAR(event_date))
SUBPARTITION BY HASH (user_id)
```

**Use case:** Large event tables where queries filter by year but also need user-level lookups within a year.

---

## Columnar Storage Formats (Work With Partitioning)

| Format | Key Feature | Best For |
|---|---|---|
| **Parquet** | Columnar + predicate pushdown | Most analytics workloads |
| **ORC** | Columnar + ACID transactions | Databricks / Hive |
| **Delta Lake** | ACID + time travel + schema evolution | Modern data lakes |
| **Iceberg** | Partition evolution + hidden partitioning | Multi-engine access |

### Partitioning + Columnar = Best Performance
1. Partitioning skips whole files
2. Columnar storage reads only the columns you need
3. Together: skip both whole files AND unnecessary columns

```sql
-- Query on a Parquet-partitioned table scans:
-- 1. Only the relevant partition (e.g., 2024-06-15)
-- 2. Only the columns referenced (ride_id, driver_id, fare_amt)
-- 3. Skips start_lat, start_lng, end_lat, end_lng, duration_secs
```

---

## Platform Differences

### BigQuery (Automated Partitioning)
- Uses **clustered tables** instead of manual partitioning (often)
- Partitioned tables: `PARTITION BY DATE(timestamp_column)`
- Automatic partition management; no manual vacuuming
- Cost is proportional to data scanned — partitioning saves $$
- Partition decorators: `table_name[2024-06-15]` to query a single partition

### Databricks (Delta Lake)
- Partition columns are file-based: data stored in `partition_col=value/` directories
- `OPTIMIZE` command compacts small files and repartitions
- `Z-ORDER` replaces clustering — co-locates related data within files
- Auto-optimize and auto-compaction available

### Snowflake (Micro-Partitions)
- **No manual partitioning needed** — Snowflake automatically creates micro-partitions
- Uses metadata (min/max values) for automatic pruning
- Clustering keys are optional — help Snowflake skip irrelevant micro-partitions
- `CLUSTER BY (column_name)` is a directive, not physical partitioning

### Redshift (Explicit Distribution + Sort)
- `DISTKEY` controls data distribution across nodes
- `SORTKEY` controls sort order within each node for zone-map pruning
- `INTERLEAVED SORTKEY` for multiple sort columns
- No true partitioning — uses distribution + sort keys for performance

---

## Common Mistakes

1. **Partitioning by a high-cardinality non-time column** (e.g., user_id creates millions of partitions) → use hash partitioning or clustering instead
2. **Not aligning partition key with query filters** → partitioning won't help if queries don't filter on the partition column
3. **Too-fine granularity** (e.g., hourly partitioning for tables with low daily volume) → metadata overhead exceeds benefits
4. **Ignoring the platform's native approach** (e.g., manually partitioning in Snowflake when micro-partitions handle it)

---

## Interview Dialogue

### Interviewer: "How would you design a table for 100M daily event rows that needs fast date-range queries?"

**Candidate:**
> "I'd partition by date. For a table this large, daily partitioning gives us precise pruning — a query for a single day only scans one partition out of 365.
>
> The partition column would be the event date (extracted from the timestamp). I'd pair this with a columnar format like Parquet for additional read efficiency.
>
> On BigQuery this translates to `PARTITION BY DATE(event_timestamp)`. On Databricks, the data would be stored in `event_date=YYYY-MM-DD/` directories. On Snowflake, I'd rely on micro-partitions and optionally add a clustering key on event_date.
>
> I'd also consider a monthly aggregate table for dashboard queries that span wider time ranges, avoiding direct scans of the raw 100M/day table for high-level reporting."

---

### Interviewer: "What's the difference between partitioning and clustering, and when would you use each?"

**Candidate:**
> "Partitioning physically separates data into different files or directories. Clustering sorts data within those files. Think of partitioning as choosing which chapter to read in a book, and clustering as having an index within that chapter.
>
> I'd partition when I have a high-cardinality column that matches my most common query filters — typically dates. I'd cluster when I need finer-grained skipping within a partition — for example, clustering a daily-partitioned events table by user_id so that user-specific queries within a day are faster.
>
> In Snowflake, the distinction is different — micro-partitions handle the physical separation automatically, and clustering keys guide which micro-partitions to skip."

---

### Interviewer: "Your daily-partitioned table is still slow for queries that filter on a single day. What would you do?"

**Candidate:**
> "First, I'd confirm the query is actually using the partition column in the WHERE clause. If it's computing the date on the fly — like `WHERE DATE(event_timestamp) = '2024-06-15'` — the partition column isn't recognized directly.
>
> I'd add a dedicated date column that matches the partition key exactly. Then I'd check the columnar format settings to ensure predicate pushdown works.
>
> If performance is still not enough, I'd add clustering within the partition — for example, by the most-queried dimension within that day (user_id, event_type).
>
> As a longer-term solution, I'd build aggregate tables at coarser granularities (hourly, daily) for the most common query patterns."

---

## Key Takeaways

| Concept | Rule of Thumb |
|---|---|
| Partition by what you filter on | Always align the partition key with your most common WHERE clause |
| Time-based is most common | Daily, monthly, or yearly depending on volume and query patterns |
| Partition + columnar = best performance | Skip files AND skip columns |
| Know your platform | BigQuery: native partitioning; Snowflake: micro-partitions; Redshift: distkey/sortkey |
| Too many partitions hurt | Balance granularity with metadata overhead |

---

## Related Lessons
- [Lesson 26 — Multi-Fact Query Strategies](./26-multi-fact-query-strategies.md)
- [Lesson 29 — Indexing Strategies](./29-indexing-strategies.md)
- [Lesson 30 — Aggregates and Materialized Views](./30-aggregates-and-materialized-views.md)
- [Lesson 10 — Data Volume and Scalability Considerations](./10-data-volume-and-scalability-considerations.md)
- [Lesson 09 — Defining Latency Requirements](./09-defining-latency-requirements.md)
