# Lesson 6: Data Volume and Scalability Considerations

## Learning objectives

-   Estimate data volume using transparent assumptions.
-   Identify growth, skew, concurrency, and retention effects.
-   Connect scale estimates to modeling and physical design.
-   Discuss horizontal scalability and cost trade-offs.

## 1. Ask for more than total row count

Understand: - Events or transactions per second/day. - Average and
maximum record size. - Peak-to-average traffic ratio. - Number of
entities and distinct keys. - Historical retention period. -
Update/delete rate. - Late-arrival and replay volume. - Number of
concurrent users and queries. - Data growth and seasonal patterns. -
Geographic distribution and source count.

"Ten billion rows" alone is not enough to size a system or choose a
model.

## 2. Back-of-the-envelope estimation

Example assumptions: - 20 million order-line records per day. - Average
logical row size: 300 bytes. - Retention: 3 years.

Raw logical volume per day:

\[ 20{,}000{,}000 `\times 300`{=tex} =
6{,}000{,}000{,}000 `\text{bytes}`{=tex} \]

Approximately 6 GB/day in decimal units, before compression, indexes,
metadata, replication, and derived datasets.

Three-year raw logical volume:

\[ 6 `\text{GB}`{=tex} `\times 365`{=tex} `\times 3`{=tex}
`\approx 6.57`{=tex} `\text{TB}`{=tex} \]

This is a rough estimate. Actual storage depends on file format,
compression, column cardinality, partitioning, indexes, replicas, and
retained intermediate data.

## 3. Peak traffic and skew

Average daily volume hides peaks. For ingestion, estimate: - Average
events per second. - Peak events per second. - Burst duration. -
Retry/replay multiplier. - Largest tenant or customer share.

Skew matters. One very large customer, product, or date partition can
create uneven work even when total volume is manageable.

## 4. How scale influences modeling

Scale may influence: - Partitioning and clustering strategy. -
Incremental processing and change capture. - Aggregate tables and data
products. - Fact retention and archival. - Key distribution and join
strategy. - File sizing and compaction. - Handling hot keys and skew. -
Backfill windows and recovery design.

It should not be used as an excuse to create an unclear or incorrect
grain.

## 5. Partitioning and pruning

Partitioning can reduce scanned data when queries filter on the
partition key. But too many small partitions can increase metadata and
file-management overhead. A partition key should be selected based on
data volume, query predicates, platform behavior, and operational
maintenance.

For time-series facts, date-based partitioning is often a candidate, not
a universal rule. Some platforms favor clustering or automatic data
organization instead.

## 6. Concurrency and workload isolation

A model may serve: - A small number of heavy analysts. - Hundreds of BI
dashboard users. - Automated machine-learning jobs. - Operational
applications. - Scheduled batch transformations.

Concurrency can matter as much as data size. Consider workload
isolation, caching, materialized aggregates, resource governance, and
separate serving layers where justified.

## 7. Example: ride-sharing platform

Suppose a platform records trip lifecycle events, GPS pings, driver
availability, and payment events.

Do not necessarily place every GPS ping in the trip fact. A trip fact
may have one row per trip, while location pings are a high-volume event
stream with a separate grain and retention policy. Aggregated route or
utilization products can be derived for analytical use.

This keeps trip-level reporting manageable while preserving detailed
telemetry only for use cases that require it.

## 8. Interviewer--candidate conversation

**Interviewer:** Design a model for a platform processing 100 million
events per day. What do you consider?

**Candidate:** I'd clarify event size, peak rate, event types,
retention, update patterns, query concurrency, and freshness. I'd
estimate raw and retained volume, then separate business processes with
different grains. For example, trip events and GPS telemetry should not
automatically be stored in one fact table.

**Interviewer:** Would you partition by customer?

**Candidate:** I'd inspect query patterns and key distribution. Customer
partitioning can create many small partitions or hot partitions for
large tenants. Time partitioning, clustering, or platform-managed
organization may be more suitable. I'd benchmark with representative
workloads.

**Interviewer:** What about backfills?

**Candidate:** I'd estimate replay volume and recovery time, make
transformations idempotent, isolate backfill compute where appropriate,
and ensure aggregates can be rebuilt or corrected without
double-counting.

## 9. Common mistakes

-   Sizing only from average traffic.
-   Ignoring retention and derived copies.
-   Assuming compression ratio without measurement.
-   Partitioning on high-cardinality keys without evaluating overhead.
-   Ignoring skew and concurrency.
-   Treating all event types as one grain.
-   Forgetting backfill and replay capacity.

## 10. Practice questions

1.  Estimate three years of data for 20 million 300-byte records per
    day.
2.  Why is peak throughput important?
3.  What is data skew, and how can it affect processing?
4.  When can partitioning hurt performance?
5.  How would you model trip records separately from GPS pings?
6.  How do backfills affect capacity planning?

## 11. Principal-level follow-ups

-   How do you set capacity and cost budgets across domains?
-   How do you design for a 10x growth scenario?
-   How do you decide whether to retain raw events or derived
    aggregates?
-   How do you isolate backfills from production workloads?
-   How do you evaluate cost per query, per data product, or per
    business outcome?

## Revision checklist

-   [ ] I estimate volume with explicit assumptions.
-   [ ] I consider peaks, skew, concurrency, and retention.
-   [ ] I connect scale to physical and operational design.
-   [ ] I understand partitioning trade-offs.
-   [ ] I include replay and backfill in capacity planning.

## Key takeaway

**Scale is a workload profile---volume, velocity, variety, skew,
retention, and concurrency---not a single row-count number.**
