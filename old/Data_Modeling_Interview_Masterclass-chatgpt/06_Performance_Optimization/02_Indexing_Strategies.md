# Lesson 6.2 — Indexing Strategies for Analytical Data Models

## 1. What an index is—and why the answer depends on the engine

An index is an auxiliary data structure that helps a database locate relevant rows or data blocks without scanning everything. In row-oriented OLTP databases, common examples include B-tree and hash indexes. Analytical warehouses often use columnar storage, metadata statistics, zone maps, min/max indexes, bloom filters, dictionaries, clustering, or search access paths instead of conventional row-store indexes.

**Interview principle:** Do not recommend “add an index to every join key” without first identifying the database engine, storage format, table size, selectivity, write pattern, and query plan.

### Learning objectives
- Explain B-tree, hash, bitmap, covering, composite, and filtered indexes.
- Understand why indexes behave differently in OLTP and columnar OLAP systems.
- Select access structures based on selectivity and query patterns.
- Evaluate write, storage, and maintenance costs.
- Compare index-like features across major cloud warehouses and lakehouses.
- Diagnose slow queries using execution plans and statistics.

## 2. Index fundamentals

### B-tree index
Maintains values in sorted tree structure and supports equality and range lookups. It is common in transactional row stores.

Example:
```sql
CREATE INDEX ix_orders_customer_date
ON orders (customer_id, order_date);
```

This may help:
```sql
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id = 9001
  AND order_date >= DATE '2026-01-01';
```

### Hash index
Uses a hash structure for equality lookups in engines that support it. It generally does not provide ordered range traversal. Support and usefulness are engine-specific.

### Bitmap index
Represents value membership using bitmaps. Historically useful for low-cardinality columns in some analytical systems, but concurrent DML and engine implementation can constrain its use. Do not assume bitmap indexes are universally available or desirable.

### Composite index
An index over multiple columns. Column order matters. For a B-tree index `(customer_id, order_date)`, a query filtering by `customer_id` and then a date range may use the leading prefix efficiently. A query filtering only by `order_date` may not benefit in the same way.

### Covering index
Contains all columns required by a query, potentially avoiding table lookups. This can improve read performance but increases index size and write cost.

### Partial / filtered index
Indexes only rows matching a predicate, where supported. Example: active records only. This can reduce index size when the filtered subset is small and frequently queried.

## 3. Index selectivity and cardinality

- **Cardinality:** number of distinct values.
- **Selectivity:** fraction of rows matching a predicate (terminology can vary; here lower fraction means more selective).
- **High-selectivity predicate:** returns a small subset of rows.
- **Low-selectivity predicate:** returns a large fraction of rows.

An index is often most useful when it can avoid reading a large amount of data. A B-tree index on a highly selective `customer_id` may be useful in an OLTP database. An index on a Boolean column with half the table matching is often less useful unless combined with another key or implemented as a partial index.

Do not use cardinality alone. A high-cardinality column can still be poorly selective for a particular query, and a low-cardinality column may be useful in a composite or bitmap structure.

## 4. Composite index column order

Suppose the common query is:

```sql
SELECT *
FROM fact_order
WHERE customer_id = ?
  AND order_ts >= ?
  AND order_ts < ?;
```

A candidate B-tree index in a row store is `(customer_id, order_ts)` because equality on the leading key is followed by a range on the next key.

But if the dominant query is by date across all customers, `(order_ts, customer_id)` may be more suitable. The right order depends on workload and engine optimizer behaviour.

Interview answer should mention:
- Equality predicates versus range predicates.
- Leading-prefix rules for the engine.
- Sort/order requirements.
- Join patterns.
- Data distribution and skew.
- Write amplification and index maintenance.
- Actual execution plan and benchmark.

## 5. Indexing in columnar analytical systems

Columnar systems often scan only required columns and use metadata to skip blocks/files. The equivalent performance tools may include:
- Min/max statistics and zone maps.
- Bloom filters.
- Dictionary encoding.
- Data skipping statistics.
- Clustering or sorting.
- Search access paths.
- Partition pruning.
- Materialized views or aggregate tables.

A columnar scan over a narrow projection and selective range may be faster than maintaining a traditional row-level index. Conversely, point lookups on huge datasets can need a specialized access path.

## 6. Platform comparison

### PostgreSQL / traditional relational engines
B-tree indexes are common for equality, range, join, and ordering workloads. Other index types (GIN, GiST, BRIN, hash, partial indexes) serve different data types and query patterns. Index selection should be based on `EXPLAIN (ANALYZE, BUFFERS)` and workload tests.

### Amazon Redshift
Do not think of Redshift as a conventional B-tree-index tuning platform. Sort keys, distribution styles, zone-map/block skipping, compression, and automatic table optimization are central. Distribution affects data movement for joins; sort order helps skip blocks for range filters and can assist merge joins in suitable cases.

### Snowflake
Snowflake uses micro-partitions and metadata for pruning. Clustering keys can improve locality on large tables; Search Optimization Service maintains search access paths for selective lookups. These have storage and compute/credit implications. They are not the same as adding a conventional B-tree index.

### BigQuery
BigQuery is a columnar serverless warehouse. Partitioning and clustering are key physical optimizations; search indexes are available for certain lookup patterns and supported data types. Verify current feature support, limits, and cost model for the specific workload.

### Databricks / Delta Lake
Performance commonly uses partition pruning, file statistics/data skipping, liquid clustering, Z-ordering in supported legacy/current configurations, caching, and optimized file layout. Liquid clustering is the current recommended layout approach for many new managed tables in Databricks; it is not a traditional index.

### Apache Iceberg
Iceberg's table metadata, manifest statistics, partition transforms, sort order, and engine-specific data skipping are important. Some engines support additional indexes or file-level structures; support is not identical across Spark, Trino, Flink, and other query engines.

## 7. Join performance: indexing is only one dimension

For large fact-to-dimension joins, investigate:
- Join key type and representation.
- Data distribution and skew.
- Broadcast/hash/sort-merge join strategy.
- Statistics freshness.
- Partition pruning before the join.
- Shuffle bytes and spill.
- Duplicate dimension keys.
- Whether the join is at compatible grain.

In a distributed engine, an index on a join key may not eliminate the need to shuffle data. Data layout, distribution, broadcast, and join strategy may matter more.

Example Spark concepts:
- Broadcast a genuinely small dimension table.
- Avoid broadcasting a dimension that is large after expansion.
- Salt or otherwise mitigate severe skew only after diagnosing it.
- Filter and project early where it reduces data volume.
- Check adaptive query execution and actual runtime plan.

## 8. Index costs and operational considerations

Every access structure has costs:
- Additional storage.
- Insert/update/delete maintenance.
- Background build or refresh compute.
- Increased load latency.
- More complicated schema changes and deployments.
- Potential stale or fragmented structures.
- Operational monitoring.

For write-heavy facts, multiple indexes can slow ingestion substantially. For append-only analytical data, clustering or statistics may offer better cost/performance than row-level indexes.

Do not create an index because a column appears in a query. Confirm that the index changes the plan and total workload cost.

## 9. A disciplined indexing workflow

1. Capture representative queries and workload frequency.
2. Establish baseline latency, scanned bytes, CPU, IO, shuffle, and cost.
3. Inspect execution plan and optimizer estimates.
4. Identify the expensive operation: scan, lookup, join, sort, aggregation, spill, or network exchange.
5. Confirm table statistics and data distribution.
6. Propose one access-path change at a time.
7. Benchmark with representative data volume and concurrency.
8. Measure read improvement and write/maintenance regression.
9. Roll out with monitoring and a rollback plan.
10. Revisit as workload changes.

A useful comparison table:

| Measure | Before | After |
|---|---:|---:|
| p50 query latency | baseline | measured |
| p95 query latency | baseline | measured |
| Bytes/blocks scanned | baseline | measured |
| Rows returned | same result | same result |
| Load/merge duration | baseline | measured |
| Storage overhead | baseline | measured |
| Total workload cost | baseline | measured |

## 10. Worked example: customer support investigation

Requirement: support engineers search a 5-billion-row interaction table by `ticket_id`, sometimes `customer_id`, and analysts run weekly reports by `event_date` and channel.

A poor answer: create indexes on every column.

A better answer:
- Separate point-lookup and analytical workloads.
- On a row-store operational replica, evaluate composite B-tree indexes such as `(ticket_id, event_ts)` and `(customer_id, event_ts)` based on actual access patterns.
- On a columnar warehouse, evaluate date partitioning/clustering or data skipping for reporting, and a search-optimized access path if supported and justified for ticket lookups.
- Avoid duplicating all interaction data into a transactional database solely to support a few lookups unless latency/operational requirements justify it.
- Benchmark both concurrency and ingestion impact.

## 11. Interview conversation

**Interviewer:** You have a 5-billion-row fact table. Which indexes do you create?

**Candidate:** First I need to know the engine. In a row-oriented database I would evaluate B-tree or other supported indexes against the actual predicates. In a columnar warehouse I would usually start with partition pruning, clustering/sorting, statistics, and the engine's search or data-skipping features. Five billion rows alone does not determine the index strategy.

**Interviewer:** The query filters by customer and date. What composite index?

**Candidate:** In a B-tree row store, if the common predicate is equality on customer followed by a date range, `(customer_id, event_date)` is a candidate. I would verify the leading-prefix behaviour, selectivity, ordering needs, and actual plan. If date-only scans dominate, another order or separate physical strategy may be required.

**Interviewer:** Why not index every foreign key?

**Candidate:** Indexes consume storage and slow writes or merges. In distributed columnar systems, they may not solve the dominant cost, which could be shuffle, scan volume, skew, or poor file layout. I would optimize measured bottlenecks, not column names.

**Interviewer:** How do you prove the index helps?

**Candidate:** Compare actual execution plans and representative workloads before and after, including p50/p95 latency, IO or bytes scanned, concurrency, load duration, storage, and total cost. The query must return the same result.

## 12. Common interview questions

1. What is an index?
2. Explain B-tree versus hash index.
3. What is a composite index?
4. Why does column order matter?
5. What is a covering index?
6. What is selectivity?
7. Why can indexing a low-cardinality column be ineffective?
8. Why do indexes slow down writes?
9. Do Snowflake and Redshift use traditional indexes?
10. What is the difference between clustering and indexing?
11. How do you optimize a large fact-to-dimension join?
12. How do you validate index effectiveness?

### Follow-up / top-up questions
- Why might an index exist but the optimizer ignore it?
- What is a bitmap index and when can it be useful?
- When would a partial index help?
- What is the leftmost-prefix principle?
- How does a covering index reduce table lookups?
- Why might a columnar scan beat an index lookup?
- How do statistics influence index selection?
- How does skew affect distributed joins?
- What is the difference between Redshift sort keys and indexes?
- When is Snowflake Search Optimization preferable to clustering?
- How do you avoid index bloat or excessive maintenance?
- What would you do if a query plan regresses after a data growth event?

### Principal-level probes
- Define an indexing/access-path policy across OLTP, warehouse, and lakehouse platforms.
- How would you separate point-lookup and analytical workloads?
- How do you evaluate performance at high concurrency rather than single-query benchmarks?
- How do you quantify the total cost of an index-like feature?
- How would you migrate a 10 TB table to a new access strategy with minimal downtime?
- What observability should be built into a platform's query performance programme?

## 13. Practice exercise

A company uses PostgreSQL for operational ticket lookup and Snowflake for analytics. The interaction table has 2 billion rows. Workloads:
- Search one ticket in under one second.
- Show customer interaction history for 90 days.
- Produce monthly channel/agent metrics.
- Load 100 million new events per day.

Recommend access structures for both platforms. Explain why the answer differs, propose indexes or clustering/search features, and define the benchmark and rollback plan.

**Evaluation rubric (10 points):**
- Engine-specific reasoning: 2
- Selectivity and column-order reasoning: 2
- Read/write cost trade-offs: 2
- Query-plan validation: 2
- Workload and operational design: 2

## 14. Key takeaways

- Indexing is engine-specific; not every analytical platform has conventional row indexes.
- Selectivity, query shape, and write rate determine whether an access structure helps.
- Composite index column order matters in row stores.
- Distributed joins often need data-layout and shuffle optimization, not just indexes.
- Benchmark total workload cost, not only one query's latency.
