# Lesson 6.1 — Date-Based Partitioning for Analytical Data Models

## 1. Why partitioning exists

Partitioning divides a large logical table into smaller physical or metadata-managed segments according to a partitioning scheme. The goal is not to make a table conceptually smaller; it is to let the query engine avoid reading data that cannot contribute to the result, simplify lifecycle operations, and sometimes improve parallelism and maintenance.

For data warehouses and lakehouses, date-based partitioning is common because event facts, transaction facts, and snapshots are naturally queried by time windows. A query for the last seven days should not need to scan several years of data if the physical layout and query predicates allow pruning.

**Interview principle:** Partitioning is a physical data-layout decision informed by workload, data volume, and platform behaviour. It is not a substitute for a correct dimensional model, selective predicates, or a well-designed query.

### Learning objectives
- Explain partitioning, pruning, and the difference between logical and physical partitioning.
- Select a date column and partition granularity using query patterns and data volume.
- Avoid over-partitioning, small files, skew, and hot partitions.
- Handle late-arriving data, backfills, retention, and time zones.
- Compare partitioning across Spark/Delta, Iceberg, BigQuery, Snowflake, and Redshift.
- Defend a partitioning strategy with evidence rather than rules of thumb.

## 2. Core terminology

- **Partition key:** the column or expression used to divide data.
- **Partition:** a subset of table data associated with one key value or range.
- **Partition pruning:** eliminating partitions that cannot satisfy a query predicate.
- **Partition granularity:** day, month, year, or another unit.
- **Partition skew:** uneven data distribution across partitions.
- **Small-file problem:** too many tiny files, often caused by excessive partition cardinality or frequent small writes.
- **Clustering / data organization:** arranging rows or files so related values are colocated; it is related to but not identical to partitioning.
- **Partition evolution:** changing the partition specification as workloads or data characteristics evolve, where the table format supports it.

Do not confuse table partitioning with Spark execution partitions. Spark partitions are units of parallel processing in a job; table partitions are a persistent storage layout concept. They can interact, but they are not the same thing.

## 3. Choosing the date column

Common candidates:
- `event_date`: date the business event occurred.
- `created_date`: date the source record was created.
- `ingestion_date`: date the platform received the record.
- `updated_date`: date the source record was last changed.
- `snapshot_date`: date represented by a periodic snapshot.

Choose the column that aligns with the dominant query and lifecycle pattern. For a fact table used for event-time reporting, `event_date` is often a sensible choice. For ingestion monitoring or immutable arrival logs, `ingestion_date` may be better.

A frequent mistake is partitioning by ingestion date while users filter by event date. If late data is common, a query for an old event period may need to scan many ingestion partitions. Conversely, event-date partitioning means a late record writes into an old partition, which can increase file rewrites or maintenance depending on the engine and table format.

A principal engineer should explicitly ask:
1. What date do users filter on most often?
2. Which date controls retention and legal deletion?
3. How late can records arrive?
4. How often are historical partitions rewritten?
5. Is the data append-only, mutable, or CDC-updated?
6. Are there multiple business time zones?

## 4. Choosing partition granularity

Typical options:

| Granularity | Often suitable when | Main risk |
|---|---|---|
| Year | Very large, long-lived data with broad time scans | Weak pruning for short windows |
| Month | Moderate volume per month and monthly reporting | Large partitions for high-volume tables |
| Day | High-volume event facts and daily filtering/retention | Too many small partitions on low-volume tables |
| Hour | Very high-volume streaming/operational event data | Excessive metadata and small files |
| No explicit date partition | Platform manages layout or table is not large | Depends on engine's automatic data skipping |

There is no universal “always partition by day” rule. A table with only a few gigabytes may become slower and harder to manage if divided into thousands of tiny partitions. A multi-terabyte event table may benefit from date-oriented pruning, but the best layout still depends on file size, query selectivity, and platform.

A useful first estimate is average bytes per partition:

```text
average partition bytes =
    total table bytes / expected number of partitions
```

This average hides skew, so also inspect min/median/p95/max partition sizes. A single high-volume tenant or campaign can make one day much larger than others.

## 5. Example: a high-volume clickstream fact

Assume:
- 4 TB compressed data per year.
- 3 years retained.
- Most queries use the last 1, 7, or 30 days.
- Late events usually arrive within 48 hours, but a small number arrive later.
- Data is written in micro-batches.

A candidate design is date partitioning by `event_date` at daily granularity, subject to the target engine's recommendations and actual file sizing. Keep `event_ts` as a timestamp for precise ordering and `ingestion_ts` for operational latency measurement.

A query should express a direct, sargable range predicate:

```sql
SELECT event_date, event_type, COUNT(*) AS events
FROM fact_clickstream
WHERE event_date >= DATE '2026-10-01'
  AND event_date <  DATE '2026-11-01'
GROUP BY event_date, event_type;
```

The half-open interval `[start, end)` avoids ambiguity at midnight and works well for adjacent windows.

Potentially less effective pattern:

```sql
WHERE DATE(event_ts) = DATE '2026-10-15'
```

Applying a function to a timestamp column may inhibit pruning in some systems, depending on optimizer rewrites and partition expressions. Prefer predicates that align with the physical partition column and verify the actual plan.

## 6. Late-arriving events and backfills

Late-arriving data is not an edge case in mature pipelines. A platform should define:
- Expected lateness distribution.
- Normal watermark or allowed-lateness window.
- Historical correction policy.
- Maximum backfill window.
- How downstream aggregates and materialized views are refreshed.
- How consumers are informed that a published period was restated.

If the table is partitioned by event date, a late event can land in an older partition. In Delta/Iceberg-style tables, updates or merges may rewrite affected files; the details depend on file layout, deletion vectors, row-level operations, and runtime version. For append-only inserts, old partitions may simply receive new files, which can create file fragmentation.

A practical design:
1. Ingest raw events with both event and ingestion timestamps.
2. Deduplicate using a stable source event identity.
3. Route events to event-date partitions.
4. Monitor lateness by event date versus ingestion date.
5. Recompute affected aggregates for the impacted dates.
6. Compact or optimize files where justified.
7. Track backfill completion and publish data freshness/completeness metadata.

Do not blindly overwrite an entire date partition if the incoming batch is only a subset of that date. Use a safe merge, dynamic partition overwrite with a verified complete partition payload, or another platform-appropriate atomic strategy.

## 7. Time zones and business dates

A timestamp is an instant; a business date is a calendar interpretation. For a global platform, `2026-11-15 00:30 UTC` may belong to the previous business date in some regions.

Store the canonical event instant (often UTC) and derive the business date using an explicitly defined timezone or tenant/location rule. If users report by local business date, consider storing `business_date` and `business_timezone` or the relevant location key.

Do not derive a single global date key from UTC and assume it matches every business calendar. Daylight-saving transitions also mean a local day can have 23 or 25 hours.

## 8. Partitioning versus clustering, sorting, and bucketing

| Technique | Primary purpose | Typical behaviour |
|---|---|---|
| Partitioning | Coarse elimination of data segments | Separate folders/metadata partitions or partition transforms |
| Clustering | Co-locate similar values | Improves data skipping for selected predicates |
| Sorting | Order rows within a table/file/block | Helps range scans, compression, merge joins on some engines |
| Bucketing | Hash-distribute rows into a fixed number of buckets | Can help joins when compatible and supported |
| Indexing | Maintain an auxiliary access structure | Engine-specific; not universal in columnar warehouses |

A common design is date partitioning plus clustering on a frequently filtered high-cardinality key such as `customer_id` or `tenant_id`, but this should only be done if the platform supports it and measured results justify it.

## 9. Platform-specific differences you must know

### Apache Spark / Delta Lake / Databricks
Traditional directory partitioning is available, but high-cardinality partition columns and small partitions can be harmful. Current Databricks guidance increasingly favours liquid clustering for many new managed tables and advises evaluating whether explicit partitioning is needed. Liquid clustering is not the same as directory partitioning; it organizes data for skipping and can evolve clustering keys. Confirm runtime and table-format support before proposing it.

### Apache Iceberg
Iceberg uses partition transforms and hidden partitioning rather than requiring users to query a physical folder layout. A table can partition by transforms such as day or month of a timestamp, and partition evolution is supported. Avoid coupling application SQL to partition directory names. File sizing, manifest management, sort order, and engine support matter.

### BigQuery
Partitioned tables can prune partitions when predicates filter the partitioning column. BigQuery also supports clustering within partitions. Partition filters and `require_partition_filter` can help control accidental full scans. The exact supported partition types and limits should be checked against current product documentation.

### Snowflake
Snowflake manages micro-partitions automatically. Users generally do not create Hive-style date partitions. Clustering keys can improve pruning for very large tables and selective recurring workloads, but reclustering consumes credits and is not appropriate for every table. Search Optimization is a different feature suited to highly selective point lookups.

### Amazon Redshift
Redshift is not typically designed around user-managed Hive-style date partitions for native tables. Sort keys and distribution styles are central physical design choices. A leading timestamp sort key can help recent-range scans, while distribution choices influence data movement for joins. Automatic table optimization may be appropriate depending on workload.

**Interview warning:** Never say “we partition every table by date” or “Snowflake uses daily partitions.” Explain the platform's actual storage abstraction.

## 10. Query plan and evidence

A performance claim should be demonstrated. Inspect:
- Partitions or files scanned versus total.
- Bytes scanned.
- Rows read versus rows returned.
- Scan time and shuffle volume.
- File count and average file size.
- Skew by partition.
- Query plan filters and pushed predicates.
- Runtime and cost before/after.

A useful metric:

```text
pruning effectiveness =
    1 - (bytes_scanned / total_table_bytes)
```

This is only an approximate diagnostic. Some systems report compressed bytes, logical bytes, or scan metrics differently. Compare like with like.

## 11. Interview conversation

**Interviewer:** You have a 20 TB event fact. How would you partition it?

**Candidate:** I would not decide from size alone. I would inspect the dominant query windows, data arrival pattern, retention, file sizes, and target platform. If most queries filter by event date and the platform benefits from explicit date partitioning, I would test daily or monthly granularity based on bytes per partition and skew. I would preserve ingestion time for latency and late-arrival monitoring.

**Interviewer:** Why not partition by customer ID?

**Candidate:** Customer ID is usually high cardinality and can create a huge number of tiny partitions. I would consider clustering, sorting, bucketing, or an engine-specific index-like feature if customer point lookups are important. The right choice depends on the engine.

**Interviewer:** What if events arrive seven days late?

**Candidate:** Event-date partitioning still places them in the correct business period. I would define an allowed-lateness window, deduplicate by event identity, update affected partitions safely, refresh dependent aggregates, and monitor the long-tail lateness. I would avoid replacing a full date partition with an incomplete batch.

**Interviewer:** The query scans all files despite a date filter. What do you investigate?

**Candidate:** First verify the predicate matches the partition transform or clustering column and isn't hidden behind a non-sargable expression. Then inspect the query plan, table metadata, partition/file statistics, type casts, timezone conversions, and whether the optimizer can infer the filter. I would also check whether the table layout actually reflects the chosen key.

## 12. Common interview questions

1. What is partition pruning?
2. Why is date a common partition key?
3. How do you choose day versus month partitioning?
4. What is over-partitioning?
5. What is the small-file problem?
6. How do late-arriving events affect date partitions?
7. What is the difference between event date and ingestion date?
8. What is the difference between partitioning and clustering?
9. How does partitioning differ across Snowflake, Redshift, BigQuery, and Databricks?
10. How do you prove that partitioning improved performance?

### Follow-up / top-up questions
- What if 90% of data belongs to one tenant?
- What if queries mostly filter by customer ID and only sometimes by date?
- How do you handle a correction to a date partition that is six months old?
- Why can a function around a date column hurt pruning?
- What happens when a query has no partition predicate?
- How do partition transforms work in Iceberg?
- How do you manage partition evolution?
- How do you avoid small files from streaming micro-batches?
- How do you choose a retention policy without breaking audit requirements?
- What is the impact of timezone conversion on partition filters?

### Principal-level probes
- Design a storage layout for a multi-tenant event platform with 10 TB/day ingestion.
- How would you handle hot tenants and skewed daily partitions?
- Propose a migration from Hive-style partitions to liquid clustering or Iceberg partition evolution.
- How do you define a platform-wide partitioning standard without forcing every table into the same layout?
- How do you measure performance, cost, and operational complexity together?
- How do you keep backfills, compaction, and streaming ingestion from competing for resources?

## 13. Practice exercise

A SaaS platform stores 12 TB of events per month. Queries include:
- Last 24 hours by tenant.
- Monthly trends by event type.
- A small number of investigations by `event_id`.
- Historical reprocessing for a subset of tenants.
- Seven-day typical lateness, with rare 90-day corrections.

Design a physical layout for Databricks Delta or Iceberg and explain how it would differ on Snowflake and Redshift. Include partition/clustering choices, file sizing, late-data strategy, retention, and query-plan validation.

**Evaluation rubric (10 points):**
- Workload and platform discovery: 2
- Partition key and granularity: 2
- Skew/small-file management: 2
- Late data and backfill strategy: 2
- Evidence and cost reasoning: 2

## 14. Key takeaways

- Partitioning is a workload-driven physical design choice.
- Choose the date column based on query, lateness, and lifecycle semantics.
- Over-partitioning and small files can make performance worse.
- Event time and ingestion time serve different purposes.
- Platform-specific storage abstractions matter: do not transfer one engine's rules blindly.
