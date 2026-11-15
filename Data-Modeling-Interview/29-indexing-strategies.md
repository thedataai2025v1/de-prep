# Lesson 29 — Indexing Strategies

## Learning Objectives
- Understand when indexes help vs hurt analytical workloads
- Compare B-tree, bitmap, columnar, and zone-map indexes
- Choose indexing strategies per platform (BigQuery, Databricks, Snowflake, Redshift)
- Explain the trade-off between read performance and write overhead

---

## When Indexes Help

Indexes speed up **lookups** and **filtering**. They work best when:
- Queries repeatedly filter on the same column(s)
- The table is large enough that a full scan is too slow
- The indexed column has enough selectivity (not all values are the same)

### When Indexes Hurt
- **Write overhead**: every INSERT/UPDATE/DELETE must also update the index
- **Storage cost**: indexes consume extra disk space
- **Maintenance cost**: indexes can fragment and need periodic rebuilding
- **Over-indexing**: too many indexes slow down every write without improving reads

---

## Index Types

### 1. B-Tree Index (Traditional RDBMS)

```sql
-- PostgreSQL, MySQL, etc.
CREATE INDEX idx_rides_date ON rides(ride_date);
CREATE INDEX idx_rides_driver ON rides(driver_id);
```

**Pros:**
- Fast point lookups and range scans
- Well-understood, mature technology

**Cons:**
- Overhead on every write
- Less effective for analytical scans (which read large portions of data)
- Not commonly used in modern columnar data warehouses

### 2. Bitmap Index

```sql
-- Used by some analytical databases internally
-- Not typically created manually
```

**Pros:**
- Excellent for low-cardinality columns (e.g., status, category)
- Very fast for complex boolean combinations (AND/OR of multiple filters)

**Cons:**
- Update overhead on write-heavy tables
- Best for read-optimized, mostly-static data

### 3. Columnar Index (Implicit in Columnar Storage)

Modern columnar formats (Parquet, ORC) have built-in statistics:
- **Min/max values** per row group (acts like a zone map)
- **Null count** per column chunk
- **Bloom filters** (some formats) for existence checks

```
-- Parquet file structure:
file.parquet
├── row_group_1/
│   ├── column_a/    (min=10, max=50, nulls=2)
│   ├── column_b/    (min=100, max=200, nulls=0)
│   └── column_c/    (min=1, max=1, nulls=0)   -- constant column
└── row_group_2/
    ├── column_a/    (min=60, max=90, nulls=0)
    ├── column_b/    (min=300, max=400, nulls=5)
    └── column_c/    (min=1, max=1, nulls=0)
```

**How it works:** When querying `WHERE column_a > 45`, the engine reads the min/max stats and skips row_group_1 entirely (max=50, but the query needs >45 — it still scans this group). It scans row_group_2 (min=60, all values >45).

### 4. Zone Maps (Oracle, Redshift)

Zone maps store min/max statistics for each data zone (similar to columnar row-group stats). They're used automatically for pruning.

---

## Platform Indexing Approaches

### BigQuery
- **No manual indexes**: BigQuery uses columnar storage with automatic statistics
- **Clustering**: `CLUSTER BY (column)` sorts data within partitions
- **Partitioning**: The primary performance mechanism
- **OmniSci / Vector Search**: Available for specific use cases (geospatial, similarity search)

```sql
-- BigQuery: Partition + Cluster
CREATE TABLE events (
    event_id    INT64,
    event_ts    TIMESTAMP,
    user_id     INT64,
    event_type  STRING
)
PARTITION BY DATE(event_ts)
CLUSTER BY user_id, event_type;
```

### Databricks (Delta Lake)
- **No traditional indexes**: Uses file-based partitioning + Z-ORDER
- **Z-ORDER**: Co-locates related data within files (multi-column clustering)
- **OPTIMIZE**: Compacts small files and applies Z-ORDER
- **Auto-optimize**: Background task for continuous optimization

```python
# Databricks: Z-ORDER on multiple columns
spark.sql("OPTIMIZE events ZORDER BY (user_id, event_type)")
```

### Snowflake
- **Micro-partitions**: Automatic, no manual indexing needed
- **Clustering keys**: Guide Snowflake's pruning (optional, not always beneficial)
- **Search optimization**: Available for low-latency point lookups on large tables

```sql
-- Snowflake: Clustering key (often unnecessary)
ALTER TABLE events CLUSTER BY (event_date, user_id);

-- Search optimization for point lookups
ALTER TABLE events ADD SEARCH OPTIMIZATION;
```

### Redshift
- **DISTKEY**: Controls data distribution across nodes
- **SORTKEY**: Controls sort order for zone-map pruning
- **INTERLEAVED SORTKEY**: Multiple sort columns

```sql
-- Redshift: Distribution + Sort keys
CREATE TABLE events (
    event_id    BIGINT DISTKEY,
    event_date  DATE SORTKEY,
    user_id     INT,
    event_type  VARCHAR(50)
);
```

---

## Choosing an Indexing Strategy

### Decision Framework

| Query Pattern | Best Approach |
|---|---|
| Point lookup (single row by ID) | Primary key / B-tree / Search Optimization |
| Date range scan | Partition by date |
| Filter on 1-2 specific columns | Cluster / Z-ORDER on those columns |
| Complex multi-column filter | Z-ORDER / INTERLEAVED SORTKEY |
| Aggregation over entire table | No index needed — full scan is fine |
| Low-cardinality filter (status, type) | Bitmap / columnar statistics |

### The "Read vs Write" Trade-off

```
Write-heavy table:          Read-heavy table:
┌──────────────┐           ┌──────────────┐
│ Minimal      │           │ More indexes │
│ indexing     │           │ = faster     │
│ (write speed)│           │ reads        │
└──────────────┘           └──────────────┘

Rule of thumb:
- Data warehouse / analytics: lean on partitioning + clustering, avoid manual indexes
- OLTP / transactional: B-tree indexes on lookup columns
- Lakehouse: Z-ORDER + OPTIMIZE, let the engine handle the rest
```

---

## Common Mistakes

1. **Creating B-tree indexes on columnar data warehouses** — they add overhead without benefit; use clustering instead
2. **Over-clustering** — too many clustering columns can hurt more than help; pick 1-3 that match your top query patterns
3. **Not optimizing after loading** — Delta Lake tables loaded in small batches need `OPTIMIZE` to compact files
4. **Indexing every column** — indexes cost storage and slow writes; only index what you actually query

---

## Interview Dialogue

### Interviewer: "How would you speed up queries on a 1B-row events table?"

**Candidate:**
> "First, I'd look at the query patterns. Most analytics queries filter by date, so I'd start with date-based partitioning. That alone might reduce a full-scan of 1B rows to scanning just the relevant partition.
>
> Next, I'd check the secondary filter columns. If queries often combine date with user_id, I'd add clustering on user_id within each date partition. In Databricks that's Z-ORDER; in BigQuery it's CLUSTER BY.
>
> I wouldn't create traditional B-tree indexes — in a columnar warehouse, the built-in statistics (min/max per row group) plus clustering are more effective for analytical workloads.
>
> If point lookups are also needed (e.g., 'find event by ID'), I'd consider a dedicated lookup table or Snowflake's search optimization for low-latency single-row access."

### Interviewer: "When would you NOT use indexes?"

**Candidate:**
> "When the workload is write-heavy — every write must update every index, which adds latency. In streaming data pipelines with high throughput, indexes can become a bottleneck.
>
> Also, when queries scan large portions of the table (e.g., full aggregations or joins), indexes don't help — the engine needs to read most of the data anyway.
>
> And in Snowflake, manual indexing is rarely needed because micro-partitions and automatic statistics handle most pruning scenarios. Adding clustering keys without measuring the benefit first can actually hurt performance."

---

## Key Takeaways

| Concept | Rule of Thumb |
|---|---|
| Partitioning > Indexing | In columnar warehouses, partitioning and clustering are your primary tools |
| Match indexes to queries | Only index columns that appear in WHERE clauses |
| Know your platform | BigQuery: partition + cluster; Databricks: Z-ORDER; Snowflake: micro-partitions; Redshift: distkey/sortkey |
| Less is more | 1-3 well-chosen clustering columns beat 10 random indexes |
| Optimize after loading | Delta Lake tables need OPTIMIZE after bulk loads |

---

## Related Lessons
- [Lesson 28 — Date-Based Partitioning](./28-date-based-partitioning.md)
- [Lesson 30 — Aggregates and Materialized Views](./30-aggregates-and-materialized-views.md)
- [Lesson 08 — Analyzing Query Patterns](./08-analyzing-query-patterns.md)
- [Lesson 10 — Data Volume and Scalability Considerations](./10-data-volume-and-scalability-considerations.md)
