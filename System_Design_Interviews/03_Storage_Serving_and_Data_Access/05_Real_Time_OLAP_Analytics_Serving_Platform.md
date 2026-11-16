# Scenario 5 — Design a Real-Time OLAP Analytics Serving Platform

**Interview level:** Staff / Principal Data Engineer  
**Primary themes:** ClickHouse, Pinot, Druid, real-time ingestion, columnar storage, indexing, pre-aggregation, replication, freshness  
**Suggested interview time:** 45–60 minutes

## 1. Problem statement

Design a real-time analytical serving platform for product analytics, operational dashboards, customer-facing metrics, and near-real-time business intelligence. The system ingests events continuously and supports high-cardinality filters, group-bys, time-series queries, and dashboard concurrency.

Assume 2 million events/sec peak, 5 KB average uncompressed event payload, 10 TB/day raw logical event volume, 90 days of hot analytical retention, 5,000 dashboard users, and 20,000 query requests/sec peak. A typical dashboard query should return in under one second at p95 for supported query shapes.

The platform must balance ingestion freshness, query latency, high-cardinality dimensions, updates/deletes, replication, and cost. A columnar OLAP engine is not a replacement for a transactional database or a general-purpose lakehouse.

## 2. Clarifying questions

1. What are peak and sustained event rates, event size, and burst duration?
2. What is the freshness target from event occurrence to query visibility?
3. Which dimensions are filtered/grouped, and what is their cardinality?
4. What are the common query shapes and dashboard concurrency?
5. Is data append-only, or are corrections, deduplication, and deletes required?
6. What retention is hot, warm, and cold?
7. What consistency is required across replicas and aggregates?
8. Are queries tenant-scoped and subject to row/column policies?
9. Is approximate aggregation acceptable for some metrics?
10. What is the recovery and replay strategy after ingestion or cluster failure?

## 3. Requirements

### Functional
- High-throughput event ingestion from Kafka or equivalent.
- Near-real-time query visibility.
- Columnar storage and pruning for analytical filters.
- Time-series, group-by, top-N, and dashboard query patterns.
- Pre-aggregation/materialized views for recurring expensive queries.
- Replication, backup, retention, and replay.
- Schema evolution and event validation.
- Tenant isolation, audit, and cost attribution.

### Non-functional
- Defined ingestion-to-query freshness.
- Low p95/p99 query latency for known query cohorts.
- High ingestion durability and bounded lag.
- Horizontal scaling and fault recovery.
- Predictable performance under concurrent queries.
- Cost controls for storage, replication, and high-cardinality indexes.

## 4. Capacity estimation

At 2 million events/sec and 5 KB average raw event size:
- raw ingress ≈ 10 GB/sec,
- raw daily volume ≈ 864 TB/day before compression,
- 90-day raw logical retention ≈ 77.8 PB.

This is an intentionally large illustrative workload. Confirm whether 5 KB is compressed or uncompressed, whether all events are retained, and whether the stated rate is sustained or peak. If events average 500 bytes, the result changes by 10x.

Estimate:
- ingestion bytes/sec and records/sec,
- compressed bytes per column,
- index and primary-key overhead,
- replication factor,
- aggregate/materialized view amplification,
- merge/compaction compute,
- query scan bytes and concurrency.

Do not equate event rate with storage footprint without compression and retention assumptions.

## 5. High-level architecture

```mermaid
flowchart TB
  PROD[Applications / telemetry] --> K[Kafka / durable event log]
  K --> VAL[Schema validation and routing]
  VAL --> ING[OLAP ingestion nodes]
  ING --> HOT[(Replicated columnar OLAP storage)]
  HOT --> Q[Query routers / coordinators]
  Q --> DASH[Dashboards / APIs / analysts]
  K --> LAKE[(Long-term lakehouse archive)]
  HOT --> MV[Materialized aggregates]
  MV --> Q
  IAM[Identity and row/tenant policies] --> Q
  OBS[Ingestion lag / merge / query telemetry] <-- ING
  OBS <-- Q
  OBS <-- HOT
  MAINT[Retention / partition lifecycle / backup] --> HOT
```

## 6. Storage engine and data model

### 6.1 Columnar layout

Columnar engines store values by column and compress similar values efficiently. Queries that read a small subset of columns avoid scanning every field. Sorting keys, sparse indexes, min/max statistics, partition pruning, and data skipping reduce the amount of data read.

Choose the sort/order key based on common filter patterns and cardinality. For a product analytics table, a candidate order might be `(tenant_id, event_date, event_type, user_id)`—but this is only valid if it matches query patterns and ingestion distribution.

Partitioning is usually coarse, often time-based. Do not create one partition per user or event ID. The sorting key and partition key serve different purposes: partitioning manages coarse lifecycle/pruning; ordering supports data locality and sparse-index skipping.

### 6.2 Event schema and evolution

Require:
- event ID for deduplication,
- tenant ID,
- event type,
- event timestamp,
- ingestion timestamp,
- schema version,
- dimensions and measures,
- source/producer identity.

Validate schema at ingestion and route invalid events to a quarantine/dead-letter path. Additive nullable fields are generally easier to evolve than renaming or changing types. Define how old producers and new readers coexist.

### 6.3 Ingestion and batching

Many OLAP engines are optimized for inserting batches/parts rather than tiny row-by-row writes. Buffer events and batch inserts to balance freshness with part/segment creation. Too many tiny inserts create merge pressure and metadata overhead.

Use Kafka as a durable replay buffer:
1. Producers publish keyed events.
2. Schema validation checks compatibility.
3. Consumer groups ingest batches.
4. OLAP engine acknowledges successful writes according to the chosen delivery design.
5. Consumer offsets advance only after durable sink acknowledgement.
6. Failed records route to retry/DLQ with bounded retry and observability.

Exactly-once is not automatic across Kafka and an OLAP sink. Design deduplication using event IDs, deterministic batch IDs, sink idempotency, or replay-safe semantics. Explain the actual guarantee.

### 6.4 MergeTree-style parts and background merges

In ClickHouse MergeTree-family engines, inserts create data parts and background merges combine parts. The ORDER BY expression determines physical sort order and sparse-index behaviour. This architecture enables high insert throughput but makes part count and merge backlog operationally important.

Monitor:
- active parts per partition,
- merge queue and duration,
- insert batch size,
- replication queue,
- disk utilization,
- mutation backlog,
- query read amplification.

Do not assume a merge immediately produces one fully consolidated part. Background merges are asynchronous and resource-consuming.

### 6.5 Pre-aggregation and materialized views

Dashboards often repeat similar group-bys. Pre-aggregate common metrics by time bucket, tenant, and key dimensions. Use materialized views or aggregate tables when query patterns are stable.

Trade-offs:
- lower query latency and compute,
- increased ingest/write amplification,
- storage duplication,
- correction/retraction complexity,
- potential semantic drift if definitions are inconsistent.

Keep metric definitions versioned and test aggregate reconciliation against raw data.

### 6.6 High cardinality

High-cardinality dimensions (user ID, URL, device ID) can stress indexes, memory, aggregation state, and group-by cardinality. Avoid indiscriminately indexing every field. Use projections, skip indexes, bloom filters, dictionary encoding, or specialized structures only after measurement and engine-specific validation.

For approximate analytics, sketches such as HyperLogLog or quantile sketches can reduce memory and compute, but must be documented with error characteristics. Exactness requirements vary by metric.

## 7. Query serving and workload isolation

Separate:
- dashboard/interactive queries,
- customer-facing API queries,
- ad hoc exploration,
- ingestion and background merges,
- maintenance and mutations.

Use query limits, concurrency caps, timeouts, memory limits, and workload routing. A dashboard can generate many near-identical queries; cache or pre-aggregate carefully. Avoid letting unbounded ad hoc group-bys consume memory needed for ingestion.

For customer-facing metrics, enforce tenant scope in a trusted query layer and validate that every query path—including materialized views and cache—preserves tenant isolation.

## 8. Replication, sharding, and recovery

Choose shard key to distribute writes and queries while avoiding hot partitions. Tenant ID may be appropriate if tenant sizes are balanced; otherwise a composite/hash key may distribute load more evenly. Shard strategy affects query fan-out and rebalancing complexity.

Replicas improve availability and read capacity but increase storage/network cost. Understand whether replication is synchronous or asynchronous, what acknowledgement means, and whether quorum settings are required for the durability target.

Kafka replay is useful for rebuilding or repairing an OLAP table, but replay duration must fit recovery objectives. Keep long-term raw archive in a lakehouse/object store rather than assuming the serving cluster is the only durable record.

## 9. Corrections, deduplication, and deletes

Analytical engines may implement updates/deletes through mutations, versioned rows, replacing engines, or background merges. These are not necessarily equivalent to transactional row updates.

For event corrections:
- preserve original event ID and version,
- define latest-wins or correction semantics,
- ensure aggregates account for retractions,
- test duplicate replay,
- monitor mutation/merge backlog.

For privacy deletion, propagate through raw log retention, OLAP tables, aggregates, caches, exports, and archive. A logical delete may not immediately remove all physical copies.

## 10. Security and governance

- Tenant-aware access and policy enforcement.
- TLS, encryption at rest, and managed secrets.
- Role-based permissions for query, insert, mutation, and administration.
- Data classification and retention labels.
- Audit query access and administrative operations.
- Masking or excluding sensitive attributes from broad analytical schemas.
- Restrict dangerous query functions and resource-intensive operations for untrusted clients.
- Separate ingestion credentials from user query credentials.

## 11. Observability and SLOs

Ingestion:
- Kafka lag and oldest unconsumed event age,
- event-to-queryable freshness,
- rejected/quarantined events,
- insert throughput and batch size,
- duplicate rate and schema errors.

Storage:
- active parts/segments,
- merge backlog and duration,
- replication lag,
- disk and memory pressure,
- mutation backlog,
- bytes per row and compression ratio.

Query:
- p50/p95/p99 latency by query fingerprint,
- rows/bytes read,
- memory, CPU, spill, and concurrency,
- cache hit rate,
- queue wait and timeout rate,
- tenant-level resource consumption.

Example SLOs:
- 99% of valid events queryable within 60 seconds.
- Dashboard cohort p95 below 1 second for defined query templates.
- 99.9% availability for read endpoints.
- No sustained merge backlog beyond the agreed operational window.

## 12. Failure modes

| Failure | Mitigation |
|---|---|
| Kafka consumer falls behind | Scale consumers, batch tuning, partition capacity, replay planning |
| Tiny inserts create part explosion | Batch ingestion, buffer, tune flush interval |
| Merge backlog grows | Reserve merge resources, reduce ingest fragmentation, add capacity |
| Hot shard/tenant | Revisit shard key, isolate large tenants, rebalance strategy |
| Replica lag/outage | Read routing, quorum policy, repair queue |
| Query memory exhaustion | Query limits, workload isolation, pre-aggregation |
| Bad schema rollout | Compatibility gates, quarantine, rollback |
| Duplicate replay | Event IDs, dedupe strategy, reconciliation |
| Delete not physically complete | End-to-end retention/deletion ledger |

## 13. Cost model

Include:
- raw Kafka retention,
- OLAP hot storage and replication,
- long-term lakehouse archive,
- compute for ingestion, merges, and queries,
- materialized view amplification,
- backup and cross-region replication,
- network transfer,
- operational overhead and capacity headroom.

Use hot OLAP storage for workloads that need low latency; move historical data to lakehouse/object storage or tiered storage where query patterns allow. Avoid keeping all history in expensive hot serving nodes by default.

## 14. Trade-offs and alternatives

| Option | Strength | Limitation |
|---|---|---|
| ClickHouse | Fast columnar analytics, SQL, MergeTree family | Requires careful part/merge/shard operations |
| Apache Pinot | Real-time user-facing analytics and indexing | Operational and modelling complexity |
| Apache Druid | Time-oriented ingestion and interactive analytics | Query/data modelling constraints |
| Warehouse | Managed SQL and broad ecosystem | Cost/latency may not fit high-QPS serving |
| Lakehouse + Trino | Open storage and flexible compute | Not always suitable for sub-second high-QPS dashboards |
| Pre-aggregates | Predictable latency | Write amplification and semantic maintenance |

Select using workload benchmarks, not vendor feature checklists. Benchmark ingestion, query concurrency, cardinality, updates, replication, and operational effort with representative data.

## 15. Principal-level follow-up questions

**Q1. How do you sustain high ingestion while keeping dashboard p95 under one second?**  
Separate ingestion and query resources where possible, batch writes to avoid tiny parts, reserve merge capacity, pre-aggregate common dashboard shapes, and enforce query concurrency. Measure freshness and query latency together because optimizing one can hurt the other.

**Q2. Why is the sort key more important than adding many indexes?**  
Columnar OLAP engines often use sparse indexes and data skipping. A sort key that clusters common predicates reduces blocks read. Extra indexes add storage and ingestion overhead and may not help if query patterns do not match.

**Q3. How do you handle duplicate Kafka delivery?**  
Use stable event IDs and sink-specific deduplication or idempotent batch semantics. Commit offsets only after sink acknowledgement, but acknowledge that replay after ambiguous failure can duplicate data unless the sink write is idempotent or deduplicated. Reconcile counts and keys.

**Q4. What if one tenant produces 40% of all events?**  
A tenant-only shard key can create a hotspot. Consider hash distribution, tenant-specific routing for large tenants, or separate clusters. Preserve tenant-level query isolation and understand query fan-out trade-offs.

**Q5. How do you correct a metric after late events arrive?**  
Define event-time windows and correction horizon. Update raw facts and recompute or retract affected aggregates. Track aggregate version/watermark and reconcile pre-aggregated values against source facts.

**Q6. When should data move from OLAP to lakehouse?**  
Keep hot recent data in the serving engine for latency-sensitive queries; retain long history in a cheaper lakehouse. Route historical queries through a federated or precomputed path and make freshness/latency expectations explicit.

**Q7. How do you ensure tenant isolation in a shared OLAP cluster?**  
Use trusted identity propagation, mandatory tenant predicates or row policies, resource quotas, audit, and negative security tests. Ensure materialized views and caches cannot bypass policy. Dedicated shards/clusters may be required for strict isolation.

**Q8. What is the first signal that the platform is unhealthy?**  
No single metric. A combination of ingestion lag, merge backlog, part count, replication lag, query tail latency, memory pressure, and freshness indicates whether the bottleneck is ingest, storage maintenance, or serving.

## 16. Common mistakes

- Confusing peak event rate with sustained rate.
- Ignoring part/segment creation and merge backlog.
- Selecting ORDER BY based on intuition rather than query patterns.
- Creating too many partitions or indexes.
- Assuming Kafka + sink means exactly-once.
- Serving every query from raw events rather than pre-aggregating.
- Ignoring high-cardinality group-by memory.
- Keeping all history in expensive hot storage.
- Treating mutations as instantaneous transactional updates.

## 17. Interview drills

**Whiteboard:** Draw event log, validation, OLAP ingestion, replicated columnar store, materialized views, query routers, policy, and lakehouse archive. Mark replay and merge paths.

**Verbal:** Explain why low-latency OLAP requires coordinated decisions across ingestion batch size, sorting, merges, pre-aggregation, and query concurrency.

**Assumption change:** Event rate remains constant, but query concurrency grows 5x and tenants demand stricter isolation. Revisit workload pools, replicas, materialized views, shard key, and per-tenant quotas.

## 18. References

- [ClickHouse MergeTree documentation](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree)
- [ClickHouse architecture documentation](https://clickhouse.com/docs/development/architecture)
- [ClickHouse create a MergeTree table](https://clickhouse.com/docs/getting-started/quick-start)
- [Apache Pinot documentation](https://docs.pinot.apache.org/)
- [Apache Druid documentation](https://druid.apache.org/docs/latest/)

Product behaviour differs by engine and release. Validate insert, replication, mutation, deduplication, and query semantics against the selected version.
