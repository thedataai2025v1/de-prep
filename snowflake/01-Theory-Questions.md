# Snowflake Interview Preparation — Theory Questions (Q1–Q30)

> **Target:** Principal Data Engineer (17 YOE)
> **Companies:** Snowflake, Atlassian, Databricks, Uber, Google, Adobe
> **Focus:** Architectural depth, production-scale decisions, failure modes, cost optimization, cross-team governance

---

## Q1. Snowflake Architecture (3-Layer SaaS Design: Cloud Services, Database Storage, Compute)

### Question
Explain Snowflake's three-layer SaaS architecture. How do the Cloud Services Layer, Database Storage Layer, and Compute Layer interact, and what architectural decisions make Snowflake different from traditional data warehouse designs?

### Answer
Snowflake's architecture is purposefully decoupled into three independent layers, each scaling independently. This contrasts sharply with traditional monolithic data warehouses like Teradata or early Redshift, where storage and compute are tightly coupled.

**Cloud Services Layer (the "Brain"):** This layer is entirely managed by Snowflake and handles authentication, security, metadata management, query parsing and optimization, and access control. It is stateless and horizontally scaled. When you submit a query, the cloud services layer parses the SQL, checks metadata (schema, grants, micro-partition statistics), builds an execution plan, and dispatches work to the appropriate warehouse. Critically, this layer does not execute queries — it orchestrates them. It maintains all metadata in a proprietary format, and every table's metadata includes the boundaries and statistics of every micro-partition, enabling aggressive pruning.

**Database Storage Layer:** All data is stored in a columnar format in cloud object storage (S3, Azure Blob, or GCS). Snowflake compresses data using multiple codecs (run-length encoding, dictionary encoding, delta encoding) per column. Data is organized into micro-partitions — immutable segments of 50–500 MB each. The storage layer is completely independent of compute, meaning you can have zero warehouses running and still store petabytes of data. Storage is charged separately from compute, billed at roughly $23/GB/month (standard storage) on AWS. This separation allows Snowflake to scale storage infinitely without touching compute.

**Compute Layer (Virtual Warehouses):** Compute is provisioned as "virtual warehouses" — clusters of servers that execute queries and DML. Each warehouse has its own local memory (SSD cache) that caches frequently accessed data. Warehouses scale from XSMALL (8 servers) to 8XLarge (1024 servers) and beyond. Compute is billed per-second (rounded up to one second) based on warehouse size. Warehouses can be suspended (stopping billing) and resumed on demand.

The decoupling enables Snowflake's key advantages: zero contention between workloads (analytics, ETL, ad-hoc), independent scaling of storage and compute, multi-cluster warehouses for concurrency, and seamless scaling.

### Example
```sql
-- Step 1: Load data into a staging table using Snowpipe
COPY INTO trips_staging FROM @s3_stage/trips/
    FILE_FORMAT = (TYPE = JSON COMPRESSION = GZIP)
    ON_ERROR = 'CONTINUE';

-- Step 2: Transform using a dedicated ETL warehouse (no contention with analytics)
CREATE OR REPLACE TABLE trips_facts AS
SELECT
    trip_id, driver_id, passenger_id,
    pickup_latitude, pickup_longitude,
    dropoff_latitude, dropoff_longitude,
    trip_distance, fare_amount, trip_timestamp,
    DATE_TRUNC('day', trip_timestamp) AS trip_date
FROM trips_staging
WHERE trip_timestamp >= DATEADD('day', -1, CURRENT_TIMESTAMP());

-- Step 3: Analysts query the same table using a separate XL warehouse
SELECT
    trip_date, COUNT(*) AS trip_count,
    SUM(fare_amount) AS total_revenue,
    AVG(trip_distance) AS avg_distance
FROM trips_facts
WHERE trip_date >= '2024-01-01'
GROUP BY trip_date;
```

### Edge Cases & Caveats
- **Cross-layer latency:** Every query incurs network I/O between warehouse servers and cloud storage. Cold queries on large datasets can be slower than on-premise systems with local NVMe storage.
- **Metadata lock contention:** Heavy DML (INSERT/UPDATE/DELETE) on the same table can cause metadata lock contention. Snowflake uses table locks for DML, and concurrent modifications to the same micro-partitions can serialize.
- **Metadata bloat at scale:** Tables with billions of micro-partitions can slow down query planning because the cloud services layer must scan metadata for pruning decisions.
- **Cross-cloud data transfer:** Moving data between cloud providers requires Snowflake's replication feature, not just metadata copy.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Snowflake 3-Layer SaaS** | Fully managed, independent scaling, zero administration, multi-cluster support | Per-second billing adds up, storage/compute separation adds latency, vendor lock-in |
| **Redshift Serverless** | Pay per query, auto-scaling, no warehouse management | Slower cold starts, less predictable latency, limited UDF support |
| **BigQuery** | Truly serverless, excellent for ad-hoc analytics, free tier | Limited ETL capabilities, less control over query execution |
| **Databricks (Unity Catalog)** | Unified analytics, Delta Lake, ML pipeline integration | Steeper learning curve, requires cluster management |
| **On-Premise MPP (Teradata/Netezza)** | No cloud costs, low latency for cached data, full control | Capital expenditure, no elastic scaling, requires DBA team |

### Principal-Level Considerations
- **Cloud lock-in vs. flexibility:** Evaluate whether Snowflake's architectural lock-in (proprietary file formats, micro-partition metadata) creates organizational risk. For multi-cloud companies, Snowflake's multi-cloud support helps, but data cannot be used outside Snowflake without re-exporting.
- **Budget governance across teams:** Design cost attribution strategies using resource monitors at the account level, per-warehouse alerts, and tag-based reporting to attribute costs to business units.
- **Disaster Recovery Architecture:** Use Snowflake's database replication across regions, ensuring the cloud services layer in the DR region has full metadata parity.

### Follow-up Questions
1. **How does Snowflake handle query execution when a warehouse is suspended?**
   - **Answer:** A suspended warehouse releases all compute resources and no billing occurs. When a query is submitted, Snowflake automatically resumes the warehouse (10-30 second startup delay). The cloud services layer detects the suspension, triggers the resume, waits for the warehouse to become active, then dispatches the query. Controlled by `AUTO_RESUME = TRUE` (default). Set `AUTO_RESUME = FALSE` for cost control or security.

2. **What happens when the cloud services layer experiences high load?**
   - **Answer:** The cloud services layer is horizontally scaled. Under heavy load, query planning and metadata operations may add 50-200ms latency per query. For long-running queries this is negligible; for thousands of small concurrent queries (e.g., dashboard tiles) you may see scheduling delays. Snowflake uses internal load balancing to distribute metadata requests.

3. **Can you have multiple virtual warehouses executing the same query?**
   - **Answer:** Yes — this is the core benefit of multi-cluster warehouses. Each cluster within a multi-cluster warehouse can execute the same or different queries. For multi-cluster warehouses, Snowflake distributes sessions across clusters automatically. The storage layer serves data to all clusters simultaneously without contention.

4. **How does Snowflake handle data integrity across the three layers?**
   - **Answer:** The storage layer uses checksums for every micro-partition. The cloud services layer tracks metadata consistency through atomic metadata operations. The compute layer does not modify storage; all writes go through the cloud services layer. Snowflake uses READ COMMITTED (default) and SERIALIZABLE transaction isolation.

5. **How does Snowflake's architecture impact GDPR/HIPAA compliance?**
   - **Answer:** Snowflake is certified for HIPAA, PCI-DSS, SOC 2, and GDPR. Data at rest is encrypted with AES-256, data in transit uses TLS, the cloud services layer supports row-level and column-level access policies, and time travel/fail-safe enable audit trails. The principal engineer must work with legal to ensure data residency and compliance.

---

## Q2. Virtual Warehouses & Concurrency Scaling (multi-cluster, sizing, scaling up vs out)

### Question
How do you size and scale virtual warehouses in Snowflake? Explain scaling up (resizing) vs scaling out (multi-cluster), when to use each, and how to design for concurrency serving hundreds of concurrent users.

### Answer
Warehouses come in sizes from XSMALL (1 credit/hour on AWS) to 9XLarge (1024 credits/hour), each a power-of-two increase. An XSMALL has 8 servers with 8 cores and 16 GB RAM total. A LARGE has 128 servers with 128 cores and 256 GB RAM. Each size doubles parallelism and memory.

**Scaling Up (Sizing):** Increasing warehouse size adds more parallel execution capacity. Resizing from LARGE to XLARGE doubles the threads for query execution. This reduces runtime for compute-bound queries but does not increase concurrency. Large warehouses (4X-8X) can complete complex aggregations over billions of rows in seconds.

**Scaling Out (Multi-Cluster Warehouses):** Multi-cluster warehouses add identical clusters to handle more concurrent sessions without increasing individual query speed. A LARGE multi-cluster warehouse with 3 clusters allows 3 simultaneous queries. This is essential for dashboard-heavy workloads with hundreds of users.

**Concurrency Management:** When all query slots are occupied, additional queries are queued. `STATEMENT_TIMEOUT_IN_SECONDS` prevents runaway queries from holding slots indefinitely. Combine multi-cluster warehouses with warehouse-specific role assignments.

### Example
```sql
-- ETL Warehouse: Large single-cluster for batch processing
CREATE OR REPLACE WAREHOUSE etl_wh
    WAREHOUSE_SIZE = '4XLARGE'
    WAREHOUSE_TYPE = 'STANDARD'
    MAX_CONCURRENCY_LEVEL = 8
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE;

-- Analytics Warehouse: Multi-cluster for dashboard concurrency
CREATE OR REPLACE WAREHOUSE analytics_wh
    WAREHOUSE_SIZE = 'XLARGE'
    MAX_CONCURRENCY_LEVEL = 16
    SCALING_POLICY = 'STANDARD'
    MIN_CLUSTER_COUNT = 2
    MAX_CLUSTER_COUNT = 10;

-- Ad-hoc Warehouse: Small multi-cluster for exploratory queries
CREATE OR REPLACE WAREHOUSE adhoc_wh
    WAREHOUSE_SIZE = 'LARGE'
    SCALING_POLICY = 'STANDARD'
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 5
    AUTO_SUSPEND = 10
    AUTO_RESUME = TRUE;

-- Assign warehouses to roles
GRANT USAGE ON WAREHOUSE etl_wh TO ROLE data_engineering;
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE data_analyst;
GRANT USAGE ON WAREHOUSE adhoc_wh TO ROLE data_scientist;
```

### Edge Cases & Caveats
- **Resizing during active queries:** Running queries complete; new queries wait until resize finishes. Creates a brief pause.
- **Multi-cluster selection is not user-controllable:** Cannot pin a query to a specific cluster. Hot queries may land on loaded clusters.
- **Memory-per-query limits:** Each warehouse size has a max memory allocation per query. Exceeding it spills to SSD, slowing execution.
- **Cold start penalties:** Auto-resumed warehouses add 10-30 seconds latency for first queries.
- **Credit consumption for multi-cluster:** Each cluster consumes credits independently (4-cluster XLarge = 4× credits).

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Large single-cluster** | Simple, predictable cost, best for ETL/batch | Poor concurrency, single point of queue |
| **Multi-cluster (STANDARD)** | Scales with concurrency, auto-managed | Higher cost at peak, unpredictable scaling |
| **Multi-cluster (ECONOMY)** | Up to 70% cost savings | Slightly longer query times |
| **Separate warehouses per workload** | Full isolation, independent sizing | More warehouses to manage, higher minimum cost |

### Principal-Level Considerations
- **Cost architecture at scale:** For a $50K/month budget, warehouse sizing is the primary cost driver. Implement automated `AUTO_SUSPEND` policies (5-10 min for ad-hoc, 30-60 min for ETL). Use resource monitors to alert on budget thresholds.
- **Workload isolation strategy:** Never share a warehouse between ETL and analytics. Define a warehouse-per-workload architecture with clear ownership.
- **Cross-cloud performance parity:** Benchmark specific workloads because real-world performance can differ by 10-20% across cloud providers.

### Follow-up Questions
1. **What is the difference between `MAX_CONCURRENCY_LEVEL` and multi-cluster `MAX_CLUSTER_COUNT`?**
   - **Answer:** `MAX_CONCURRENCY_LEVEL` limits total concurrent queries per warehouse regardless of cluster count. `MAX_CLUSTER_COUNT` determines how many physical clusters Snowflake provisions. Total concurrency is roughly `MAX_CONCURRENCY_LEVEL × MAX_CLUSTER_COUNT`. `MAX_CONCURRENCY_LEVEL` is a cap; `MAX_CLUSTER_COUNT` is scaling capacity.

2. **How do you decide between a single 8X-Large warehouse and 8 X-Large warehouses?**
   - **Answer:** Choose 8X-Large when: (a) massive parallelism for a single query, (b) simple management, (c) moderate concurrency. Choose 8 X-Large when: (a) workload isolation needed, (b) independent auto-suspend policies, (c) different access controls per warehouse. Both cost the same credits.

3. **What happens when a multi-cluster warehouse scales down?**
   - **Answer:** Active queries on decommissioning clusters complete; new queries go to remaining clusters. No data loss — local cache is ephemeral. Takes 1-3 minutes. `SCALING_POLICY` determines timing: `STANDARD` scales more aggressively, `ECONOMY` waits longer.

4. **How does warehouse size affect query execution plan?**
   - **Answer:** Larger warehouses provide more execution threads (one per server). The same logical plan distributes work across more threads. A 4X-Large with 512 threads processes each row faster than an XSMALL with 8 threads due to parallelism, not a smarter plan.

5. **Can you dynamically resize a warehouse mid-query?**
   - **Answer:** No. Running queries complete on the old size before the resize applies. Pre-size warehouses to handle expected peak loads rather than relying on dynamic resizing during active workloads.

6. **How do you monitor warehouse utilization?**
   - **Answer:** Use `SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METRICS` for query slot utilization. <50% utilization indicates over-provisioning. >80% with queued queries indicates under-provisioning. Establish a monthly warehouse review process evaluating 30-day utilization trends.

---

## Q3. Cloud Services Layer (auth, metadata, access control, query planning)

### Question
Describe the Cloud Services Layer. What responsibilities does it hold, and how does its design impact query performance, security, and metadata management at scale?

### Answer
The Cloud Services Layer is Snowflake's central orchestration engine — a fully managed, horizontally scaled service handling authentication, authorization, query parsing, optimization, planning, and metadata management.

**Authentication & Authorization:** All authentication (password, SSO/SAML, OAuth, key-pair, MFA) and authorization (role-based access control) happens here. Every query is authenticated before reaching the warehouse. The layer enforces all grants, row access policies, and dynamic data masking. It maintains a complete privilege graph, resolving inherited roles to determine effective permissions.

**Metadata Management:** Every table's metadata — micro-partition boundaries, min/max statistics per column, row counts, clustering statistics — is stored here. Metadata updates happen atomically with every DML operation. This metadata enables aggressive pruning: before dispatching a query, the layer examines micro-partition metadata to eliminate irrelevant partitions. For a table with 1 billion rows, it might prune 99% of micro-partitions before any compute is engaged.

**Query Planning & Optimization:** The cost-based query optimizer parses SQL, builds a logical plan, and a physical plan considering data distribution, micro-partition statistics, and warehouse capacity. It makes decisions about join algorithms, predicate pushdown, and parallelism using micro-partition statistics.

### Example
```sql
-- The execution plan is generated entirely by the Cloud Services Layer
-- before any warehouse is engaged
EXPLAIN QUERY_ID('latest')
SELECT
    o.order_id, o.order_date, c.customer_name,
    SUM(oi.quantity * oi.unit_price) AS order_total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_date >= '2024-01-01'
    AND o.region = 'US-WEST'
GROUP BY o.order_id, o.order_date, c.customer_name;
```

The Cloud Services Layer does: (1) authenticate user, (2) check table metadata, (3) determine which micro-partitions match filters, (4) estimate join cardinalities, (5) choose join algorithms, (6) generate physical plan, (7) dispatch to warehouse.

### Edge Cases & Caveats
- **Metadata lock contention during heavy DML:** Use bulk `COPY INTO` or transaction batching.
- **Metadata bloat for frequently modified tables:** Use `CLUSTER BY` or manual re-clustering.
- **Cross-database queries:** Must fetch metadata from all referenced databases, adding planning latency.
- **UDF/SP compilation:** Compiled on first execution; subsequent executions use cached plans.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Snowflake (managed metadata)** | Zero administration, automatic stats, aggressive pruning | Cannot tune optimizer, no direct metadata access |
| **Redshift (column stats)** | User can ANALYZE tables | Requires manual stats maintenance |
| **BigQuery (automatic stats)** | No user intervention needed | Less granular control |
| **Databricks (Delta Lake metadata)** | Open format, transaction log | User manages optimization |

### Principal-Level Considerations
- **Metadata as single source of truth:** Unlike systems with separate metastores, Snowflake's metadata is authoritative and atomically updated. No "stale metadata" problems.
- **Security architecture:** The cloud services layer enforces ALL security policies before data reaches warehouses. Security is not dependent on warehouse configuration.
- **Vendor lock-in:** The cloud services layer is entirely proprietary — you cannot export the optimizer or metadata engine.

### Follow-up Questions
1. **How does the cloud services layer handle concurrent DML on the same table?**
   - **Answer:** Uses optimistic locking. INSERT operations are generally non-blocking (new micro-partitions). UPDATE/DELETE on the same micro-partitions is serialized. Design tables so concurrent writers target different date ranges.

2. **Can you view or influence query optimization decisions?**
   - **Answer:** View plans with `EXPLAIN` or `QUERY_HISTORY`. Influence via: (a) clustering keys, (b) materialized views, (c) CTEs to force evaluation order, (d) explicit CAST to avoid type conversion penalties.

3. **How does the cloud services layer handle programmatic authentication?**
   - **Answer:** Supports username/password (with MFA), OAuth 2.0, key pair authentication (RSA), JWT tokens. Key pair is recommended for automated pipelines.

4. **What happens to metadata when you drop a table?**
   - **Answer:** Table is marked "dropped" but data retained in fail-safe period (7 days default). Can UNDROP. After fail-safe expires, both metadata and data are permanently deleted.

5. **How does query queuing and prioritization work?**
   - **Answer:** Queries queue in FIFO when warehouse slots exhausted. No priority levels. Use `STATEMENT_TIMEOUT_IN_SECONDS` to limit run time. Assign high-priority workloads to dedicated warehouses.

6. **How does the cloud services layer contribute to Time Travel?**
   - **Answer:** Tracks every micro-partition's creation timestamp. When querying with `AT(OFFSET => -86400)`, the layer looks up historical metadata and directs the warehouse to read historical micro-partitions. Copy-on-write model means old data remains accessible.

---

## Q4. Micro-partitions (how Snowflake creates/sorts/prunes them, clustering)

### Question
Explain how Snowflake's micro-partitions work. How are they created, sorted, and pruned? What is the relationship between micro-partitions, clustering keys, and query performance?

### Answer
Micro-partitions are the fundamental data organization unit in Snowflake — immutable, columnar data segments, typically 50-500 MB each (compressed), created automatically when data is loaded.

**Creation:** When data is loaded (via `COPY INTO`, Snowpipe, or INSERT), Snowflake writes data in columnar format to cloud storage. Each micro-partition is an independent file. The number created depends on the data volume per operation. Small, frequent inserts create many micro-partitions (anti-pattern); bulk loads create fewer, larger ones.

**Sorting:** Micro-partitions are created in arrival order, typically NOT sorted by any column. Snowflake records min/max statistics for each column. This is where clustering keys become critical — they cause reorganization so rows with similar clustering key values are stored together, enabling efficient pruning.

**Pruning:** When a query includes a filter, the cloud services layer examines min/max statistics of each micro-partition. If a partition's range doesn't overlap with the filter, it's skipped entirely. Well-clustered data with selective filters can prune 99%+ of micro-partitions.

### Example
```sql
-- Anti-pattern: Frequent small inserts create micro-partition bloat
INSERT INTO events SELECT * FROM events_staging WHERE event_date = '2024-01-01';  -- 100 MB
INSERT INTO events SELECT * FROM events_staging WHERE event_date = '2024-01-02';  -- 100 MB
-- ... repeated for 1,000 dates

-- Result: EXPLAIN shows poor pruning
EXPLAIN SELECT * FROM events WHERE event_date = '2024-06-15';
-- Partition total: 100,000
-- Partition pruned: 50,000 (only 50% — poor pruning!)

-- Solution: Use a clustering key
ALTER TABLE events CLUSTER BY (event_date);

-- After clustering: data reorganized so each micro-partition contains
-- a contiguous range of dates
EXPLAIN SELECT * FROM events WHERE event_date = '2024-06-15';
-- Partition total: 100,000
-- Partition pruned: 99,900 (99.9% — excellent pruning!)
-- Partition scanned: 100
```

### Edge Cases & Caveats
- **Micro-partition bloat from small INSERTs:** Avoid `INSERT INTO table VALUES (...)` loops. Use bulk `COPY INTO` or `MERGE` with large batches.
- **Over-clustering:** Clustering on too many columns reduces effectiveness. Stick to 1-3 columns, with highest cardinality first.
- **Re-clustering cost:** Clustering consumes credits. Use `AUTOMATIC_CLUSTERING = TRUE` but monitor costs via `CLUSTERING_HISTORY`.
- **Sorting at load time:** If source data is pre-sorted, Snowflake creates sorted micro-partitions, reducing need for re-clustering.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Clustering key** | Automatic pruning, no manual DBA | Cost of re-clustering, 180-day learning period |
| **Manual partition pruning** | Full control | Requires SQL-level partitioning logic |
| **Materialized views** | Pre-computed results, automatic maintenance | Limited to aggregates, additional storage |
| **No clustering** | Zero cost, simple | Full table scans, very poor performance at scale |

### Principal-Level Considerations
- **Design for pruning:** Every table at PB scale should have a clustering key. Choose columns that appear in WHERE clauses AND are naturally ordered (dates, sequential IDs).
- **Monitor clustering depth:** Use `SYSTEM$CLUSTERING_DEPTH()` to check clustering quality. Depth should be close to the number of micro-partitions for maximum pruning.
- **Partition strategy for multi-cloud:** Different cloud providers have different object store performance characteristics. Benchmark clustering on each cloud.

### Follow-up Questions
1. **How does Snowflake decide when to create a new micro-partition vs. appending to an existing one?**
   - **Answer:** Snowflake creates a new micro-partition for each bulk data loading operation. When data is inserted (even a single row), a new micro-partition is created. There is no appending to existing micro-partitions — they are immutable. This is why frequent small inserts cause partition bloat. For optimal micro-partition count, batch inserts to at least 50-100 MB per operation.

2. **Can you manually control micro-partition boundaries?**
   - **Answer:** No. Micro-partition boundaries are system-managed. However, you can influence them through clustering keys, which reorganize data so that rows with similar key values end up in the same micro-partitions. If you need explicit partition control, consider using separate tables per partition (e.g., a table per day) with a UNION ALL view on top.

3. **What is the relationship between micro-partitions and Time Travel?**
   - **Answer:** Each Time Travel query reads historical micro-partitions that still exist in storage. Because micro-partitions are immutable, dropping a row does not modify existing partitions — it creates new micro-partitions without those rows. The old micro-partitions remain until the Time Travel retention period expires. This is why Time Travel has zero query performance penalty but does consume storage for old versions.

4. **How does Snowflake handle micro-partition metadata refresh for streaming data?**
   - **Answer:** Snowpipe streaming inserts create micro-partitions continuously. Each insert triggers a metadata update in the cloud services layer. For high-frequency streaming (>1000 inserts/second), metadata updates can become a bottleneck. Use larger batch sizes (via Snowpipe Streaming) or use staging tables that are merged hourly into main tables.

5. **How does micro-partition pruning work for JOIN operations?**
   - **Answer:** For JOINs, pruning happens based on the join key in the probe table. If table A is hash-joined with table B on `A.id = B.id`, and table B has a filter (e.g., `B.date = '2024-01-01'`), the pruned B rows determine which A micro-partitions need scanning. Pruning effectiveness depends on the join column being the clustering key of at least one table.

6. **What is the maximum practical number of micro-partitions?**
   - **Answer:** Snowflake recommends keeping micro-partition counts under 1 billion. Beyond that, metadata operations slow down. For a 1 PB table with 100 MB average micro-partition size, that's 10 billion partitions — well past the recommendation. Clustering reduces partition count by merging small partitions during re-clustering. If you approach 1 billion partitions, consider partitioning the table (e.g., by date range) to reduce per-query partition scans.
---

## Q5. Zero-Copy Cloning (how it works, use cases, billing)

### Question
How does Snowflake's Zero-Copy Cloning work? What is the architectural mechanism that makes it "zero-copy," and what are its practical applications for development, testing, and production scenarios?

### Answer
Zero-Copy Cloning creates an independent copy of a database, schema, or table without physically duplicating the underlying data. The clone shares the same storage as the source until modifications diverge.

**Architecture:** When you clone a table, Snowflake creates new metadata entries in the cloud services layer pointing to the same underlying micro-partitions. The metadata includes a timestamp marking when the clone was created. Micro-partitions are immutable, so both source and clone read the same files initially. When you modify data in either the source or the clone, new micro-partitions are created for changed data, while unchanged data continues to be shared. This copy-on-write (CoW) mechanism ensures zero storage overhead at creation time.

**Use Cases:** Creating identical copies for development/testing, performing "what-if" analysis on production data, creating point-in-time snapshots, and staging environments for ETL testing.

### Example
```sql
-- Clone an entire database for a development sandbox
CREATE DATABASE dev_analytics CLONE prod_analytics;

-- Clone a table for ad-hoc analysis
CREATE TABLE refunds_analysis CLONE prod_analytics.facts.refunds
    AT(OFFSET => -7200);  -- Clone as of 2 hours ago

-- Clone a schema for QA testing
CREATE SCHEMA qa.analytics CLONE prod.analytics;

-- Verify that storage is shared (no additional bytes charged initially)
SHOW TABLES IN DATABASE dev_analytics;
SELECT COUNT(*) AS row_count FROM dev_analytics.facts.sales;  -- Same rows as source

-- After this UPDATE, new micro-partitions are created for dev_analytics
UPDATE dev_analytics.facts.sales SET status = 'archived'
WHERE sale_date < '2023-01-01';
```

### Edge Cases & Caveats
- **Cloning a table being actively loaded** may produce a near-consistent snapshot — Snowflake uses a transactional snapshot, but if a load is mid-transaction, you'll see the table as of the snapshot time.
- **Cloning across accounts** requires data sharing or replication, not direct cloning.
- **Storage billing:** While the clone shares micro-partitions initially, any modifications create new ones, and storage costs accrue independently.
- **Time Travel on clones:** Clones have their own Time Travel retention period (default 1 day) — they do not inherit the source's Time Travel.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Zero-Copy Clone** | Instant, no storage overhead initially | Storage grows as modifications diverge; no write isolation (reads only) |
| **CREATE TABLE AS SELECT** | Full data copy, independent | Slow for large tables, storage charged for all data |
| **Snapshot Storage** | Explicit point-in-time | Manual management, requires external tooling |
| **Data Sharing (Reader Accounts)** | No storage at consumer end | Latency for data refresh, no write support |

### Principal-Level Considerations
- **Dev/Test environment strategy:** Use a single clone of production daily for QA; automate refresh with a task. Implement data masking policies on production data before cloning for dev environments.
- **Cost governance:** Monitor storage divergence between clones and production. Set alerts when clone storage exceeds 25% of source storage.
- **Regulatory compliance:** Clones of production contain all data, including PII. Ensure security policies propagate to clones.

### Follow-up Questions
1. **What happens when you clone a database that has external tables?**
   - **Answer:** External table definitions are cloned (metadata), but they still reference the same external location. No external data is duplicated. This is useful for testing external table schemas.

2. **Can you clone across regions or cloud providers?**
   - **Answer:** No — cloning is within-account only. Cross-region requires database replication. Use `ALTER DATABASE ... ENABLE REPLICATION` to set up replication.

3. **How are stored procedures and functions handled when cloning a database?**
   - **Answer:** Stored procedures, UDFs, and their code are cloned as metadata. No recompilation. The cloned objects are fully independent after cloning.

4. **Does cloning trigger any re-clustering or data reorganization?**
   - **Answer:** No — cloning is purely a metadata operation. Clustering state is also cloned (metadata about micro-partition boundaries). No compute cost or time.

5. **How do you ensure that a cloned database is synced daily for QA?**
   - **Answer:** Use a task:
```sql
CREATE OR REPLACE TASK qa.refresh_qa_clone
    WAREHOUSE = etl_wh
    SCHEDULE = 'USING CRON 0 6 * * * America/New_York'
AS
    DROP DATABASE IF EXISTS dev_analytics;
    CREATE DATABASE dev_analytics CLONE prod_analytics;
```

6. **Can you clone a table while another user is querying it?**
   - **Answer:** Yes. Cloning is a metadata operation and does not block reads or writes on the source table. Any concurrent queries continue without interruption.

---

## Q6. Time Travel & Fail-safe (undrop, historical queries, retention periods)

### Question
Explain Snowflake's Time Travel and Fail-safe features. How do they differ in mechanism, retention, and cost? What are their practical uses in production data management?

### Answer
Time Travel and Fail-safe provide data recovery capabilities. Time Travel is user-configurable and user-queryable; Fail-safe is a fixed 7-day recovery period managed by Snowflake.

**Time Travel:** Allows querying and undropping data as it existed at any point within a configurable retention period (1 day Standard, 1-90 days Enterprise/Business Critical). Queries use `AT(OFFSET => seconds)`, `AT(TIMESTAMP => ...)`, or `BEFORE(STATEMENT => ...)`. The `UNDROP` command restores dropped objects within the retention window. The mechanism relies on micro-partition immutability — old micro-partitions are preserved, and metadata tracks which partitions existed at each point in time.

**Fail-safe:** A 7-day non-queryable recovery period after Time Travel expires. Only Snowflake support can initiate recovery (which takes hours to days). Fail-safe exists for Snowflake's internal recovery, not for user access. Data during fail-safe period continues to accrue storage costs.

### Example
```sql
-- Set Time Travel retention for Enterprise/Business Critical
ALTER TABLE sales SET DATA_RETENTION_TIME_IN_DAYS = 30;

-- Query data as of 2 hours ago
SELECT COUNT(*) FROM sales
    AT(OFFSET => -7200);

-- Query data as of a specific timestamp
SELECT COUNT(*), SUM(fare_amount) FROM trips
    AT(TIMESTAMP => '2024-06-15 14:30:00'::TIMESTAMP);

-- Undrop a dropped table
DROP TABLE payments;
SELECT * FROM payments;  -- Error: table does not exist

UNDROP TABLE payments;
SELECT * FROM payments;  -- Data is restored

-- Undrop a dropped database
DROP DATABASE analytics;
UNDROP DATABASE analytics;

-- View historical data with CTE
WITH historical AS (
    SELECT * FROM orders AT(OFFSET => -86400)  -- Yesterday's orders
),
current AS (
    SELECT * FROM orders
)
SELECT 'lost_orders' AS status, COUNT(*) FROM historical h
WHERE NOT EXISTS (SELECT 1 FROM current c WHERE c.order_id = h.order_id)
UNION ALL
SELECT 'new_orders' AS status, COUNT(*) FROM current c
WHERE NOT EXISTS (SELECT 1 FROM historical h WHERE h.order_id = c.order_id);

-- Clone as of a point in time for audit
CREATE TABLE orders_audit CLONE orders AT(OFFSET => -86400);
```

### Edge Cases & Caveats
- **Fail-safe is not user-serviceable:** You cannot query data in fail-safe. Only Snowflake support can recover it.
- **Storage costs for Time Travel:** Old micro-partitions still occupy storage. At high data change rates, Time Travel storage can exceed active storage.
- **`UNDROP` name conflicts:** If a new object with the same name was created after the drop, `UNDROP` fails. Drop the new object first or rename it.
- **Time Travel retention affects cloning:** Cloning a table as of a point within the retention period is allowed; beyond it, you must use fail-safe (support only).

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Time Travel (user)** | Instant queries, self-service, full SQL | Storage cost for old partitions |
| **Fail-safe (Snowflake)** | Automatic safety net, 7 days free | Cannot query, support-request only |
| **Manual snapshots (COPY INTO)** | Full control, independent retention | Extra ETL, storage, management cost |
| **External versioning (S3)** | Cheaper, cross-platform compatible | No SQL access, manual restoration |

### Principal-Level Considerations
- **Set retention policy per table:** High-churn tables (staging, logs) only need 1 day. Core fact tables (accounts, orders) need 30-90 days. This balances cost vs. recovery needs.
- **Budget for Time Travel storage:** At a PB scale with 5% daily data change, Time Travel storage for 90 days can be 4.5× active storage. Build into cost model.
- **Fail-safe recovery is an SLA event:** For Enterprise/Business Critical, fail-safe recovery takes 1-5 hours. Have a playbook for declaring recovery and communicating to stakeholders.

### Follow-up Questions
1. **Can you restore a single row from Time Travel?**
   - **Answer:** Yes — query the historical table with a filter and INSERT the result back into the current table:
```sql
INSERT INTO orders
SELECT * FROM orders AT(OFFSET => -3600)
WHERE order_id = 12345
  AND NOT EXISTS (
    SELECT 1 FROM orders WHERE order_id = 12345
  );
```

2. **How does Time Travel interact with Streams?**
   - **Answer:** Streams only track changes to the current version of a table. If you use Time Travel to query historical data and then INSERT it back, the stream will show insert offsets for the restored rows. If you UNDROP a table, any streams defined on it are not restored automatically.

3. **What is the storage mechanism for Time Travel data?**
   - **Answer:** Time Travel data is just old micro-partitions that haven't been garbage-collected. Snowflake never modifies micro-partitions in place. When rows are updated or deleted, Snowflake creates entirely new micro-partitions for the changed data, and the old ones are retained in storage. Metadata keeps a transaction timeline mapping each version of the table to the set of micro-partitions that were "live" at that point.

4. **How do you estimate Time Travel storage costs?**
   - **Answer:** Use `SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS` which shows ACTIVE_BYTES and TIME_TRAVEL_BYTES per table. The total is `TIME_TRAVEL_BYTES * (storage_rate / 1TB)`. For a 1TB table with 10% daily churn and 90-day retention, expect ~9TB of Time Travel storage (10% × 90 days).

5. **What happens to Time Travel when you drop a table?**
   - **Answer:** The Time Travel clock starts from the drop time. For example, if the table had 30-day retention and was created 10 days ago, you have 30 days from the drop to UNDROP (not 20 days). After that, it enters Fail-safe (7 days), then is permanently deleted.

6. **Can you query Time Travel across tables that were joined in the past?**
   - **Answer:** Yes — both tables must be queried at the same point in time:
```sql
SELECT o.*, c.customer_name
FROM orders AT(OFFSET => -86400) o
JOIN customers AT(OFFSET => -86400) c
  ON o.customer_id = c.customer_id;
```

---

## Q7. Data Loading (COPY INTO, file format detection, staged files)

### Question
Explain Snowflake's data loading mechanisms using `COPY INTO`. How do staged files, file format detection, and error handling work? What patterns do you use at production scale?

### Answer
Data loading in Snowflake centers on the `COPY INTO` command, which loads data from staged files into tables. Staging can be internal (Snowflake-managed) or external (S3, Azure Blob, GCS).

**File Formats:** Snowflake automatically detects file formats (CSV, JSON, Avro, Parquet, ORC, XML) and schema. For CSV, it detects delimiters, escape characters, and header lines. For JSON, it detects newline-delimited JSON vs. structured JSON. File format options are set at stage or COPY level.

**Validation and Error Handling:** The `VALIDATION_MODE = RETURN_ERRORS / RETURN_ALL_ERRORS` option validates files before loading. `ON_ERROR = 'CONTINUE' / 'SKIP_FILE' / 'ABORT_STATEMENT'` controls what happens when bad rows are encountered.

**Production Patterns:**
1. **Bulk Copy (daily/hourly batches):** `COPY INTO` directly from stage
2. **Continuous Loading (Snowpipe):** Event-driven, files loaded as they appear
3. **Streaming (Snowpipe Streaming):** Real-time, row-level inserts via API

### Example
```sql
-- Create external stage
CREATE OR REPLACE STAGE s3_data_stage
    URL = 's3://company-data-prod/incoming/'
    STORAGE_INTEGRATION = s3_int
    FILE_FORMAT = (
        TYPE = PARQUET
        COMPRESSION = SNAPPY
        NULL_IF = ('NULL', 'null', '')
    );

-- Manual COPY INTO with validation
COPY INTO sales_facts FROM @s3_data_stage/sales/2024/
    FILE_FORMAT = (FORMAT_NAME = my_parquet_format)
    VALIDATION_MODE = RETURN_ERRORS;

-- Load with error tolerance, capture errors
COPY INTO sales_facts FROM @s3_data_stage/sales/2024/
    ON_ERROR = 'CONTINUE';

-- Validate rejected records
SELECT * FROM TABLE(
    VALIDATE(sales_facts, JOB_ID => '019e3f2e-0601-7b00-0000-0000c7d20001')
);

-- Transform during load
COPY INTO enriched_sales (
    sale_id, product_id, quantity, sale_date, ingestion_ts
)
FROM (
    SELECT
        $1:sale_id::VARCHAR(50),
        $1:product_id::INT,
        $1:quantity::INT,
        $1:sale_date::DATE,
        CURRENT_TIMESTAMP()
    FROM @s3_data_stage/sales_json/
)
FILE_FORMAT = (TYPE = JSON STRIP_OUTER_ARRAY = TRUE);

-- Load with pattern matching
COPY INTO logs FROM @s3_data_stage/logs/
    PATTERN = '.*log_2024.*\.csv'
    FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1);
```

### Edge Cases & Caveats
- **Schema drift:** If source schema changes, `COPY INTO` fails with column mismatch unless using schema evolution.
- **Large number of small files:** Creates many micro-partitions. Coalesce small files before loading.
- **Concurrent loads into same table:** Allowed but can cause serialization. Use multiple tables then `MERGE` if needed.
- **Semi-structured data with inconsistent keys:** `ON_ERROR = 'CONTINUE'` may create null rows.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Bulk COPY INTO** | Fastest throughput, best for batch | Requires orchestration, no real-time |
| **Snowpipe** | Minimal latency (minutes), serverless | Cost per file, no retry logic, 10MB minimum |
| **Snowpipe Streaming** | Sub-second latency, row-level | Higher per-row cost, API complexity |
| **INSERT INTO SELECT** | Full flexibility, can transform | Slower, no parallel loading |

### Principal-Level Considerations
- **Landing zone architecture:** Design a staging area per source system. Use `ON_ERROR = 'CONTINUE'` for resilient ingestion but pipe errors to a dead-letter table for remediation.
- **Cost model for loading:** Loading into a warehouse costs credits (compute). Snowpipe costs 2 credits/hour per pipe. For high-volume loads (100+ GB/hour), bulk COPY INTO is cheaper.
- **Data freshness SLAs:** Define freshness requirements (real-time vs hourly vs daily) and match loading method to SLA.

### Follow-up Questions
1. **How does `COPY INTO` handle duplicate file loading?**
   - **Answer:** Snowflake records file metadata (name, size, last-modified timestamp). If you re-submit the same file, `COPY INTO` skips it. Use `FORCE = TRUE` to re-load.

2. **What are the recommended file sizes for optimal loading?**
   - **Answer:** 100-250 MB compressed per file. Smaller files create micro-partition bloat; larger files reduce parallelism. Snowpipe has a 10MB minimum for continuous loading.

3. **How do you handle schema evolution in source files?**
   - **Answer:** Use `TYPE = PARQUET` with schema evolution enabled (`MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE`). For CSV, use `SKIP_HEADER = 1` and add new columns with `ALTER TABLE ADD COLUMN`. For JSON, the VARIANT type is schema-flexible.

4. **What is the difference between internal and external stages?**
   - **Answer:** Internal stages are on Snowflake-managed cloud storage; external stages are on customer-managed storage. External stages give you data portability and are cheaper. Internal stages are simpler but lock data in Snowflake.

5. **How do you load data from multiple sources in a single COPY INTO?**
   - **Answer:** Use a union transformation: or use separate COPY INTO commands per source, then `MERGE`.

6. **How do you monitor loading failures in production?**
   - **Answer:** Query `COPY_HISTORY` for errors and bytes loaded. Set up alerts for when error counts exceed thresholds. Use a daily validation that loads a sample from each stage.

---

## Q8. Semi-structured Data (VARIANT, OBJECT, ARRAY, PARSE_JSON, FLATTEN)

### Question
How does Snowflake handle semi-structured data with the VARIANT, OBJECT, and ARRAY types? Explain `PARSE_JSON`, `FLATTEN`, and query patterns for efficiently working with JSON data at scale.

### Answer
Snowflake's native support for semi-structured data via VARIANT column type makes it a leading choice for modern data pipelines that must handle flexible schema data.

**VARIANT Type:** A universal type that can hold any single value (string, number, object, array, null). Internally, each value is stored as a binary format with type markers. Storage is optimized: repeated keys are stored once per micro-partition. A single VARIANT column can store entire JSON documents.

**Querying:** Use dot notation (`column:key`) or bracket notation (`column['key']`) to access fields. Cast strings to appropriate types for joins and comparisons. Use `PARSE_JSON` to convert string JSON to VARIANT. Use `OBJECT_KEYS`, `ARRAY_SIZE`, and `FLATTEN` to explore structure.

**Performance Considerations:** Storing everything as VARIANT (raw JSON) is convenient but has drawbacks: no native indexes, slower comparisons, and larger storage. Extract critical fields into dedicated columns for joins and filtering.

### Example
```sql
-- Create table with VARIANT column
CREATE OR REPLACE TABLE raw_events (
    event_id INTEGER AUTOINCREMENT,
    event_data VARIANT,
    ingested_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Load JSON data directly
COPY INTO raw_events(event_data)
FROM (
    SELECT PARSE_JSON($1) FROM @json_stage/events.json
)
FILE_FORMAT = (TYPE = JSON);

-- Query semi-structured data
SELECT
    event_id,
    event_data:user_id::VARCHAR(50) AS user_id,
    event_data:action::VARCHAR(100) AS action,
    event_data:timestamp::TIMESTAMP_NTZ AS event_time,
    event_data:metadata:device_type::VARCHAR(20) AS device_type,
    event_data:metadata:browser_version::VARCHAR(10) AS browser_version
FROM raw_events
WHERE event_data:action::VARCHAR = 'purchase'
  AND event_data:amount::NUMBER(10,2) > 100;

-- Use FLATTEN to explode arrays
SELECT
    e.event_id,
    f.value:product_id::VARCHAR(20) AS product_id,
    f.value:quantity::INT AS quantity,
    f.value:price::NUMBER(10,2) AS price
FROM raw_events e,
LATERAL FLATTEN(INPUT => e.event_data:items) f
WHERE e.event_data:action::VARCHAR = 'order_completed';

-- Extract commonly queried fields into dedicated columns for performance
CREATE OR REPLACE TABLE events AS
SELECT
    event_id,
    event_data:user_id::VARCHAR(50) AS user_id,
    event_data:action::VARCHAR(100) AS action,
    event_data:timestamp::TIMESTAMP_NTZ AS event_time,
    event_data:amount::NUMBER(10,2) AS amount,
    event_data:metadata:device_type::VARCHAR(20) AS device_type,
    event_data:metadata:region::VARCHAR(10) AS region,
    event_data
FROM raw_events;

-- Parse nested arrays
SELECT
    event_id,
    event_data:checkout:payment:method::VARCHAR(10) AS payment_method,
    event_data:checkout:shipping:address:zip::VARCHAR(10) AS zip_code
FROM raw_events
WHERE event_data:checkout:payment:method::VARCHAR = 'credit_card';
```

### Edge Cases & Caveats
- **Variant vs structured columns:** Storing everything in VARIANT and accessing via dot notation works but has performance cost. Extract filter columns.
- **Deeply nested data:** Use `FLATTEN` with `RECURSIVE => TRUE` for deeply nested arrays. Be careful of exponential explosion.
- **Schema inconsistency:** Semi-structured data can have differing keys per row. Use `OBJECT_KEYS` to discover schema.
- **Null handling:** Missing keys return SQL NULL. Use `IFNULL` or `COALESCE` for defaults.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **VARIANT (raw JSON)** | Flexible schema, easy ingestion | Slower queries, larger storage, no native joins |
| **Extracted columns + VARIANT** | Best performance, preserves raw data | More ETL work, schema must be known |
| **Parquet/Avro** | Native columnar, highly compressed | Requires transformation before loading |
| **External tables (JSON)** | No storage in Snowflake | Slower queries, no clustering |

### Principal-Level Considerations
- **Schema-on-read governance:** With VARIANT, there is no DDL governance. Different teams may read the same `event_data` with different castings. Document the expected schema.
- **Cost of storing raw JSON:** A 1GB JSON file becomes ~200-300 MB as VARIANT (after compression). Still larger than Parquet (80-120 MB). Audit storage costs for raw JSON tables.
- **Data quality in semi-structured data:** Build a validation pipeline that casts all fields and raises alerts for unexpected types.

### Follow-up Questions
1. **How does VARIANT storage work internally?**
   - **Answer:** Each micro-partition stores a shared dictionary of keys. The values are stored as a compact binary format. Repeated keys don't repeat storage per row. Numbers are stored precisely. Strings are stored as UTF-8 with dictionary compression. Date/Time values are validated at query time.

2. **What are the performance implications of querying deeply nested paths?**
   - **Answer:** Each dot-level access (e.g., `data:user:profile:name`) requires traversing the internal tree structure. For 100M rows, a 6-level deep path access adds ~20-50ms per level. Extract nested fields used in filters into dedicated columns.

3. **Can you create a materialized view on a VARIANT column?**
   - **Answer:** Yes, if the SELECT uses a specific cast path. The materialized view requires deterministic expressions. Unpredictable access patterns (different paths per query) won't benefit.

4. **How do you handle arrays of objects within VARIANT?**
   - **Answer:** Use `LATERAL FLATTEN` with the array path:
```sql
SELECT f.value:product_name::VARCHAR(100) AS product_name
FROM orders, LATERAL FLATTEN(INPUT => order_data:items) f;
```

5. **What is the maximum depth/complexity for VARIANT data?**
   - **Answer:** Snowflake supports up to 4,294,967,295 elements per VARIANT. Practical limit is ~16MB per value for query performance. Deeply nested data (>10 levels) should be normalized.

6. **How does query pruning work on VARIANT columns?**
   - **Answer:** Pruning does NOT work on extracted paths from VARIANT columns. The cluster statistics are on the VARIANT column itself, not on specific keys within it. To enable pruning, extract the filter column into a separate typed column and cluster on it.

---

## Q9. Streams & Tasks (CDC, orchestration, DML tracking)

### Question
How do Streams and Tasks work together in Snowflake for Change Data Capture (CDC) and workflow orchestration? Explain the different stream types and how to handle failure/retry scenarios.

### Answer
Streams track DML changes (INSERT, UPDATE, DELETE) on a table. Tasks execute scheduled SQL statements, optionally using streams to process incremental changes.

**Stream Types:**
1. **Standard Stream:** Tracks all changes after creation. Shows before/after values for UPDATEs.
2. **Append-only Stream:** Tracks only INSERT changes (ignores UPDATE/DELETE). Cheaper (no before-image storage).
3. **Timestamp-based Stream:** Tracks changes since a specific time.

**How Streams Work:** When you create a stream, Snowflake creates metadata tracking which micro-partitions are "new" (from DML). When you consume the stream (SELECT from it), you see deltas. After you consume and process the stream (by `INSERT...SELECT` or `MERGE`), the offset advances and processed records disappear. Streams are transactional: a stream's offset advances only when the consuming transaction commits.

**Tasks:** Execute SQL on a schedule (cron) or after a preceding task completes. Tasks can have a warehouse assignment and are managed with `TASK_HISTORY` for monitoring. DAG-based task trees enable multi-step pipelines.

### Example
```sql
-- Create a stream to track CDC
CREATE OR REPLACE STREAM user_changes ON TABLE users
    SHOW_INITIAL_ROWS = TRUE;  -- Show existing rows as inserts initially

-- Query the stream
SELECT * FROM user_changes;

-- Task that processes the stream every 5 minutes
CREATE OR REPLACE TASK process_user_changes
    WAREHOUSE = etl_wh
    SCHEDULE = '5 MINUTE'
    COMMENT = 'CDC: Sync user changes to analytics layer'
WHEN
    SYSTEM$STREAM_HAS_DATA('user_changes')
AS
    MERGE INTO dim_user AS target
    USING (
        SELECT
            user_id,
            name,
            email,
            status,
            updated_at
        FROM user_changes
        QUALIFY ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY
                CASE METADATA$ACTION
                    WHEN 'DELETE' THEN 0
                    WHEN 'INSERT' THEN 1
                    ELSE 2
                END DESC
        ) = 1
    ) AS source
    ON target.user_id = source.user_id
    WHEN MATCHED AND source.status = 'deleted' THEN DELETE
    WHEN MATCHED THEN UPDATE SET
        target.name = source.name,
        target.email = source.email,
        target.status = source.status,
        target.updated_at = source.updated_at
    WHEN NOT MATCHED THEN INSERT
        (user_id, name, email, status, updated_at)
    VALUES
        (source.user_id, source.name, source.email,
         source.status, source.updated_at);

-- Task DAG: chain tasks after stream processing
CREATE OR REPLACE TASK refresh_aggregates
    WAREHOUSE = etl_wh
    AFTER process_user_changes
AS
    CALL compute_user_aggregates();

ALTER TASK process_user_changes RESUME;
ALTER TASK refresh_aggregates RESUME;

-- Monitor task execution
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP())
));
```

### Edge Cases & Caveats
- **Stream offset management:** If task fails after consuming stream, offset advances but target may be inconsistent. Use `MERGE` for idempotent operations.
- **Stream staleness:** If a stream is not consumed for 14 days, it "stales." Re-create the stream to reset.
- **Stream on views:** Streams cannot be created on views (only on tables). Create on base table and filter in your task.
- **Task scheduling delays:** Tasks may experience scheduling delays of seconds to minutes.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Stream + Task** | Native Snowflake, no external tools, transactional | 14-day staleness limit, no real-time |
| **Snowpipe Streaming** | Sub-second latency, row-level CDC | Higher cost, API-based |
| **Kafka Connect** | External ecosystem, robust offsets | Infrastructure overhead, no DDL tracking |
| **Debezium** | Database-native CDC, real-time | External deployment, schema registry needed |

### Principal-Level Considerations
- **Stream staleness monitoring:** Create a monitoring task that alerts if any stream hasn't been consumed in 7 days. Stream staleness at 14 days requires re-creation.
- **Task failure handling:** Tasks do not auto-retry on failure. Implement a retry pattern: if a task fails, use a secondary task to detect and re-process.
- **TCO comparison:** For under 100K changes/day, Stream+Task is cheapest. For millions/day, consider Snowpipe Streaming or external tools.

### Follow-up Questions
1. **What happens to a stream when a table is truncated?**
   - **Answer:** `TRUNCATE TABLE` is DDL, not DML, so it is NOT captured by streams. After truncating a table, the stream shows no changes (offset unchanged). Use `DELETE FROM` (DML) if you need CDC capture.

2. **How do streams handle schema changes on the source table?**
   - **Answer:** If columns are added, the stream includes them. If columns are dropped, the stream still shows them as NULL for changed rows. After ALTER TABLE DROP COLUMN, the stream continues to work.

3. **Can a task run multiple times concurrently?**
   - **Answer:** No — Snowflake ensures at most one execution of a given task at a time. If the task takes longer than the schedule interval, the next scheduled run is skipped.

4. **How do you ensure exactly-once processing with streams?**
   - **Answer:** Use `MERGE` (not INSERT) when consuming the stream. `MERGE` is idempotent based on the key. If a task fails after a partial insert, the next execution can safely re-read the same stream data.

5. **What is the cost model for streams?**
   - **Answer:** Streams have no direct credit cost. However, they increase storage costs — before-images for UPDATE/DELETE must be preserved. Append-only streams have minimal storage overhead.

6. **How do you test streams before going to production?**
   - **Answer:** Create a development clone, create streams on the clone, run DML operations, query the stream, verify offsets advance correctly after consuming.
---

## Q10. Materialized Views (auto-refresh, clustering, cost)

### Question
Explain how Materialized Views work in Snowflake. How does auto-refresh function, what are the clustering considerations, and when is the cost of maintaining a materialized view justified?

### Answer
Materialized Views (MVs) are pre-computed result sets stored as tables. Snowflake automatically refreshes them as data in the base table changes, unlike standard views which compute results on every query.

**Auto-Refresh Mechanism:** When DML changes the base table, Snowflake maintains a "delta" log of changed micro-partitions. The MV refresh mechanism reads these deltas and incrementally recomputes only affected rows. The refresh is triggered automatically by Snowflake's cloud services layer when the base table's changes reach a threshold (approximately 5-10% of rows modified). Between refreshes, MVs may be slightly stale.

**Clustering:** Materialized views can have their own clustering keys, independent of the base table. This is powerful: you can cluster the MV on a different column than the base table to serve a different query pattern.

**Cost Considerations:**
- Storage cost: MV data is stored as a separate table
- Compute cost: Auto-refresh consumes credits (charged to the account, not a specific warehouse)
- If the base table changes frequently (every few seconds), the MV refresh cost may exceed the query savings

### Example
```sql
-- Base table: billions of rows, clustered on order_date
CREATE OR REPLACE TABLE orders (
    order_id NUMBER(38,0),
    customer_id NUMBER(38,0),
    product_id NUMBER(38,0),
    quantity NUMBER(10,2),
    unit_price NUMBER(10,2),
    order_date DATE,
    region VARCHAR(50)
)
CLUSTER BY (order_date);

-- Materialized View: daily sales by region (pre-aggregated)
CREATE MATERIALIZED VIEW daily_sales_by_region AS
SELECT
    order_date,
    region,
    COUNT(DISTINCT order_id) AS order_count,
    COUNT(customer_id) AS customer_count,
    SUM(quantity * unit_price) AS total_revenue
FROM orders
WHERE order_date >= '2020-01-01'
GROUP BY order_date, region;

-- Cluster the MV on a different column than the base table
ALTER MATERIALIZED VIEW daily_sales_by_region
    CLUSTER BY (region, order_date);

-- Query that benefits from the MV
SELECT
    region,
    SUM(total_revenue) AS revenue
FROM daily_sales_by_region
WHERE region IN ('US-WEST', 'US-EAST')
  AND order_date BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY region;
-- This query would scan ~150 micro-partitions instead of ~500,000
```

### Edge Cases & Caveats
- **Limited SQL support:** MVs cannot contain: self-JOINs, subqueries, DISTINCT, ORDER BY, LIMIT, GROUP BY on expressions, multi-table joins.
- **Staleness:** MVs are not refreshed synchronously with base table updates. The max staleness is typically under 5 minutes.
- **Base table DDL changes:** If you add/remove columns from the base table, the MV definition is re-evaluated. Some changes may require MV recreation.
- **No direct refresh control:** You cannot manually force-refresh an MV. Snowflake manages the refresh schedule.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Materialized View** | Auto-refreshed, query-optimized aggregation | Limited SQL constructs, refresh cost |
| **Standard View** | Always current, no storage cost | Scans base table every query |
| **Manual Aggregation Table** | Full SQL flexibility, manual refresh control | Requires orchestration, stale by design |
| **Dynamic Table** | More flexible SQL, different refresh strategies | Larger storage, newer feature, less mature |
| **Clustering on base table** | No redundancy, benefits all queries | May not optimize the specific query pattern |

### Principal-Level Considerations
- **Cost-benefit analysis:** Build an MV only if: query savings (in credits) > refresh cost. At scale, a well-chosen MV serving 1000+ queries/day can save $5K-$10K/month.
- **Refresh cost monitoring:** Use `SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY` to view refresh credits. If refresh cost > 30% of query savings, reconsider.
- **Design for clustering alignment:** The MV's clustering key should match the most common query filters on the MV.

### Follow-up Questions
1. **How does Snowflake handle materialized view refresh for append-only tables vs frequently updated tables?**
   - **Answer:** For append-only tables, MV refresh is nearly free (just reads new micro-partitions). For tables with UPDATE/DELETE, the refresh must recompute affected partitions. Append-only MVs have ~30% lower refresh cost.

2. **Can you create a materialized view on top of another materialized view?**
   - **Answer:** No — MVs can only reference base tables (or secure views, but not other MVs). Chaining MVs is not supported.

3. **What happens to the MV when the base table is dropped?**
   - **Answer:** The MV is automatically dropped as well. If the base table is restored via UNDROP, the MV is NOT restored automatically — it must be recreated.

4. **How do you estimate MV storage cost before creating it?**
   - **Answer:** Run a `COUNT(*)` and estimate row size. Multiply by per-TB storage rate. For a 10 billion row aggregation table (50 bytes/row), storage is ~500 GB ~ $12/month.

5. **How does query routing work to use an MV?**
   - **Answer:** Snowflake's optimizer automatically rewrites queries to use MVs if they match the MV definition (partial or full). You cannot force a query to use a specific MV. Use `EXPLAIN` to verify.

6. **What is the difference between Enterprise and Business Critical editions for MVs?**
   - **Answer:** MVs are available in Enterprise Edition and higher. Business Critical adds enhanced security (HIPAA, PCI-DSS) but does not change MV functionality.

---

## Q11. Secure Views & Dynamic Data Masking (masking policies, row access policies)

### Question
How do you implement column-level security with Dynamic Data Masking and row-level security with Row Access Policies in Snowflake? What are the architectural patterns for applying these at scale?

### Answer
Snowflake provides two parallel security mechanisms: Dynamic Data Masking (column-level) and Row Access Policies (row-level). Both are applied at the policy level without modifying the underlying data.

**Dynamic Data Masking (DDM):** A masking policy is a SQL expression that conditionally transforms column values based on the role of the querying user. It is attached to a specific column in a table or view. The policy evaluates conditions at query time; the underlying data is never modified.

**Row Access Policies (RAP):** A row access policy is a SQL expression that evaluates a Boolean condition per row, determining which rows are visible to the querying user. RAPs are attached at the table level and apply to all queries on that table.

**Key Architectural Differences:**
- DDM operates on columns; RAP operates on rows.
- DDM can obscure values; RAP can hide entire rows.
- Both are evaluated at query time and cannot be bypassed.
- Both require Enterprise Edition or higher.

### Example
```sql
-- Create a masking policy
CREATE OR REPLACE MASKING POLICY mask_ssn AS
    (val VARCHAR) RETURNS VARCHAR ->
        CASE
            WHEN CURRENT_ROLE() IN ('DATA_ADMIN', 'HR_MANAGER') THEN val
            WHEN CURRENT_ROLE() IN ('HR_ANALYST') THEN
                'XXX-XX-' || RIGHT(val, 4)
            ELSE '*********'
        END;

-- Apply masking policy to a column
ALTER TABLE employees
    MODIFY COLUMN ssn
    SET MASKING POLICY mask_ssn;

-- Row Access Policy: restrict by organization
CREATE OR REPLACE ROW ACCESS POLICY org_isolation AS
    (org_id INT) RETURNS BOOLEAN ->
        EXISTS (
            SELECT 1 FROM user_orgs uo
            WHERE uo.user_email = CURRENT_USER()
              AND uo.org_id = org_id
        );

-- Apply row access policy
ALTER TABLE sales ADD ROW ACCESS POLICY org_isolation ON (org_id);

-- Create a conditional masking policy based on the row's contents
CREATE OR REPLACE MASKING POLICY mask_salary AS
    (val NUMBER, role VARCHAR) RETURNS NUMBER ->
        CASE
            WHEN CURRENT_ROLE() IN ('DATA_ADMIN', 'COMPENSATION') THEN val
            WHEN CURRENT_ROLE() = 'MANAGER' AND role = 'IC' THEN val
            ELSE NULL
        END;

-- Apply with context columns
ALTER TABLE employees
    MODIFY COLUMN salary
    SET MASKING POLICY mask_salary USING (salary, role);

-- Test from different roles
USE ROLE hr_analyst;
SELECT name, ssn, salary FROM employees WHERE dept = 'Engineering';
-- ssn shows 'XXX-XX-1234', salary shows NULL

USE ROLE data_admin;
SELECT name, ssn, salary FROM employees WHERE dept = 'Engineering';
-- ssn shows full value, salary shows full value
```

### Edge Cases & Caveats
- **Performance impact:** Both policies add a WHERE condition or CASE expression to every query. At PB scale, this can add 5-15% query time.
- **Policy stacking:** Cannot have multiple masking policies on the same column. Must merge into one.
- **Materialized views:** Row access policies do NOT apply to materialized views. Apply at the base table, which propagates to the MV.
- **Cloning:** When you clone a table, the policies propagate. Masking policies are shared across clones (same policy object).
- **Cross-database references:** Row access policies that reference a separate access control table must handle schema changes carefully.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Dynamic Data Masking** | No data duplication, real-time, fine-grained | Performance overhead, one policy per column |
| **Row Access Policies** | No data duplication, row-level filtering | Cannot be bypassed even by admin |
| **Separate Views per Role** | Explicit, easy to understand | Maintenance burden, schema proliferation |
| **Secure Views** | Combines row and column filtering | Complex DDL, must manage grants per view |
| **External Security (Apache Ranger)** | Cross-platform, rich ecosystem | Infrastructure overhead, external dependency |

### Principal-Level Considerations
- **Policy organization:** Store all policies in a shared security database. Use descriptive naming (`mask_ssn_production`, `rap_org_isolation`). Document each policy's purpose.
- **Testing policy changes:** Before altering a policy, test in a clone of production to verify behavior for all roles. Blind policy changes can expose sensitive data or break queries.
- **Audit trail:** Use `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` to track which roles accessed which columns. Build a dashboard flagging unusual access patterns.

### Follow-up Questions
1. **What happens when a user with the ACCOUNTADMIN role queries a table with a row access policy?**
   - **Answer:** Row access policies apply to all roles, including ACCOUNTADMIN. Only users explicitly listed in the policy's exception logic can bypass it. This is intentional — RAPs are irrevocable even by admins.

2. **Can you attach both a masking policy and a row access policy to the same table?**
   - **Answer:** Yes — they operate independently. Row access policy filters rows first; masking policy obscures values on the remaining rows. The masking policy sees the post-filtered values.

3. **How do you handle time zones in masking policies?**
   - **Answer:** Use `CURRENT_TIMESTAMP()` in the policy logic. E.g., mask data only during non-business hours:
```sql
CASE
    WHEN EXTRACT(HOUR FROM CURRENT_TIMESTAMP()) BETWEEN 9 AND 17
        THEN val
    ELSE 'MASKED'
END;
```

4. **Do masking policies apply to the results of `COPY UNLOAD`?**
   - **Answer:** Yes — if you unload data from a table with masking policies, the policy is evaluated on export. If data needs to be exported unmasked, use a role exempt from the policy or export from a table without policies.

5. **Can you use a stored procedure to look up the current user in a masking policy?**
   - **Answer:** No — masking policies must contain pure SQL expressions. Stored procedures cannot be invoked inside a masking policy. Use `CURRENT_USER()`, `CURRENT_ROLE()` built-in functions instead.

6. **How do you manage policies across development, staging, and production environments?**
   - **Answer:** Use version-controlled SQL scripts for policy creation. In CI/CD, deploy policies to staging first, verify with automated test queries, then promote to production. Use `DESCRIBE POLICY` to check policy definitions before changing.

---

## Q12. Resource Monitors & Cost Management (credits, thresholds, actions)

### Question
How do Resource Monitors work in Snowflake? Explain how to set up cost management at account, warehouse, and user levels to keep Snowflake costs predictable and prevent budget overruns.

### Answer
Resource Monitors are Snowflake's mechanism for tracking and controlling credit consumption. They can be applied at the account level, warehouse level, or both, with cascading hierarchy.

**Credit Monitoring Hierarchy:**
1. **Account-level monitor:** Tracks total credits consumed by the entire account
2. **Warehouse-level monitors:** Track consumption per virtual warehouse
3. **User-level tracking:** Available through `ACCOUNT_USAGE` views (not separate monitrs)

**Thresholds and Actions:**
- Set multiple thresholds (e.g., 50%, 75%, 90%, 100%) with actions:
  - NOTIFY (send alert)
  - SUSPEND_IMMEDIATELY (suspend warehouse)
  - SUSPEND (allow current queries to finish)
  - ABORT (terminate all running queries and suspend)

**Credit Calculation:** Each warehouse size consumes credits at a specific rate:
- XSMALL: 1 credit/hour
- SMALL: 2 credits/hour
- MEDIUM: 4 credits/hour
- LARGE: 8 credits/hour
- XLARGE: 16 credits/hour
- 2XLARGE: 32 credits/hour
- 4XLARGE: 64 credits/hour
- 8XLARGE: 128 credits/hour

Cloud services layer credits (for metadata operations) are free for the first 10% of compute credits.

### Example
```sql
-- Account-level resource monitor
CREATE OR REPLACE RESOURCE MONITOR account_monthly
    WITH CREDIT_QUOTA = 50000
    FREQUENCY = MONTHLY
    START_TIMESTAMP = '2024-01-01 00:00:00'
    END_TIMESTAMP = '2024-12-31 23:59:59'
    TRIGGERS
        ON 50 PERCENT DO NOTIFY
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND_IMMEDIATE
        ON 110 PERCENT DO ABORT;

ALTER ACCOUNT SET RESOURCE_MONITOR = account_monthly;

-- Warehouse-level monitors with separate quotas
CREATE OR REPLACE RESOURCE MONITOR etl_wh_monthly
    WITH CREDIT_QUOTA = 20000
    FREQUENCY = MONTHLY
    TRIGGERS
        ON 80 PERCENT DO NOTIFY
        ON 95 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND;

ALTER WAREHOUSE etl_wh
    SET RESOURCE_MONITOR = etl_wh_monthly;

-- Per-user consumption monitoring
CREATE OR REPLACE VIEW user_credit_usage AS
SELECT
    user_name,
    role_name,
    warehouse_name,
    SUM(credits_used) AS total_credits,
    COUNT(*) AS query_count,
    SUM(credits_used) / NULLIF(COUNT(*), 0) AS avg_credits_per_query,
    SUM(total_elapsed_time) / 1000 AS total_seconds
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP)
GROUP BY user_name, role_name, warehouse_name
ORDER BY total_credits DESC;

-- Alert on anomalous spikes (compared to 30-day average)
WITH daily_usage AS (
    SELECT
        DATE_TRUNC('day', start_time) AS day,
        SUM(credits_used) AS daily_credits
    FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METRICS
    WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
    GROUP BY day
)
, stats AS (
    SELECT
        AVG(daily_credits) AS avg_daily,
        STDDEV(daily_credits) AS std_daily
    FROM daily_usage
    WHERE day < DATE_TRUNC('day', CURRENT_TIMESTAMP())
)
SELECT
    day, daily_credits,
    (daily_credits - avg_daily) / NULLIF(std_daily, 0) AS z_score
FROM daily_usage, stats
WHERE day = DATE_TRUNC('day', CURRENT_TIMESTAMP())
  AND (daily_credits - avg_daily) > 3 * std_daily;
```

### Edge Cases & Caveats
- **SUSPEND vs SUSPEND_IMMEDIATE:** `SUSPEND` is graceful (queries complete); `SUSPEND_IMMEDIATE` terminates active queries immediately.
- **End_timestamp behavior:** When `END_TIMESTAMP` is reached, the monitor becomes inactive but does not auto-reset.
- **Monitor priority:** Warehouse-level monitors override account-level. A query that violates the warehouse monitor is suspended even if the account monitor has capacity.
- **Cloud services credits:** The first 10% of compute credits for cloud services are free; beyond that, they count toward the quota.
- **Notifying multiple recipients:** Use Snowflake's notification integration (email, webhook, SNS) for alerts.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Resource Monitors** | Native, no external infra, zero configuration | No cost attribution by tag/project |
| **Tag-based Cost Tracking** | Track costs by project/environment | Requires tag governance, no auto-action |
| **External Budgeting (AWS Budgets)** | Multi-service view, centralized | No Snowflake-specific action |
| **Custom Monitoring (Stored Procedure)** | Full flexibility | Requires development, maintenance |

### Principal-Level Considerations
- **Monthly credit budget allocation:** Divide account quota by workload (ETL 40%, Analytics 30%, Ad-hoc 15%, Dev/Test 10%, Cloud Services 5%).
- **Automated remediation:** Set warehouse-level monitors to SUSPEND at 100% for non-production environments. For production, NOTIFY at 90%, SUSPEND at 110%.
- **Cost optimization review cadence:** Monthly review of `WAREHOUSE_METRICS` and `QUERY_HISTORY`. Identify idle warehouses, oversized warehouses, and queries that consume disproportionate credits.

### Follow-up Questions
1. **Can resource monitors be applied retrospectively to running queries?**
   - **Answer:** No — resource monitors only take effect when their threshold is crossed during monitor evaluation. Running queries are not checked retroactively.

2. **How do resource monitors interact with Snowpipe?**
   - **Answer:** Snowpipe is billed separately (serverless compute). Account-level resource monitors include Snowpipe credits in the total. Warehouse-level monitors do not cover Snowpipe.

3. **What happens when a warehouse monitor triggers ABORT?**
   - **Answer:** All active queries are immediately terminated. Users receive an "Operation terminated by resource monitor" error. Completed or queued queries are not affected.

4. **How do you track credit consumption by team without separate warehouses?**
   - **Answer:** Use `TAG` objects to label queries by team or project:
```sql
ALTER SESSION SET TAG team = 'data_science';
ALTER SESSION SET TAG project = 'recommendation_v2';
```
Query `TAG_REFERENCES` in `QUERY_HISTORY` for cost attribution.

5. **What is the best practice for alerting thresholds?**
   - **Answer:** 3-tier: (1) WARNING at 50% (gas pedal was heavy), (2) CRITICAL at 80% (need to evaluate month-end workloads), (3) SUSPEND at 100% (budget exhausted). Notify the team at each level.

6. **How do you handle seasonal peaks (e.g., Black Friday for a retail data warehouse)?**
   - **Answer:** Create temporary resource monitors for peak periods with higher quotas. Remove them after peak. Use `FREQUENCY = WEEKLY` during peak months for tighter control, then revert to monthly.

---

## Q13. Performance Optimization (query acceleration, result cache, pruning, explain)

### Question
What strategies do you use for query performance optimization in Snowflake? Explain Query Acceleration Service, result caching, micro-partition pruning, and how to use EXPLAIN to optimize queries.

### Answer
Snowflake provides multiple levers for tuning query performance, from automatic features to manual optimization.

**Query Acceleration Service (QAS):** A serverless service that accelerates queries with large scans or complex aggregations. When enabled at the warehouse level, Snowflake automatically offloads eligible portions of queries to elastic compute nodes. QAS is billed at 1.5× standard warehouse credits. It helps when: queries scan > 1 TB of data, or queries have high spill-to-ssd (shown in `QUERY_HISTORY`).

**Result Cache:** Snowflake caches the results of every query within 24 hours. Identical queries (matching SQL text, but ignoring whitespace and comments) served from cache in milliseconds — zero credit cost. Cache persists for 24 hours regardless of warehouse state. Only invalidated by DML on the queried table.

**Micro-Partition Pruning:** The most impactful optimization. Ensure clustering keys match WHERE clause filters. Use `EXPLAIN` to verify partition count before and after pruning.

**EXPLAIN Command:** Shows the execution plan with: operations, object names, estimated rows, partition info. Use it to identify full table scans, hash joins (build vs probe side), sorting operations, and spill.

### Example
```sql
-- Enable Query Acceleration Service
ALTER WAREHOUSE analytics_wh
    SET ENABLE_QUERY_ACCELERATION = TRUE
    QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;
-- QAS can use up to 8× the warehouse's compute power

-- Check if QAS was used for a specific query
SELECT
    query_id,
    query_text,
    total_elapsed_time / 1000 AS elapsed_sec,
    query_acceleration_bytes_scanned,
    query_acceleration_partitions_scanned,
    credits_used_query_acceleration
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_acceleration_bytes_scanned > 0
ORDER BY start_time DESC;

-- Use EXPLAIN to find optimization opportunities
EXPLAIN
SELECT
    region,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(amount) AS total_sales
FROM sales
WHERE sale_date >= '2024-01-01'
  AND sale_date < '2024-04-01'
  AND region IN ('US-WEST', 'US-EAST')
GROUP BY region;

-- Look for:
-- 1. "TableScan" with "partitionsPruned" vs "partitionsTotal"
-- 2. "Aggregate" operations (hash vs sort)
-- 3. "Join" operations (hash join build/probe sides)

-- Check cache effectiveness
SELECT
    query_text,
    executed_as_previous_query_result,
    start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE executed_as_previous_query_result = TRUE;

-- Manual techniques to improve pruning
-- 1. Date range in WHERE clause (even if you want all data)
SELECT * FROM huge_table
WHERE date_key > '1900-01-01';  -- Bad: no pruning
WHERE date_key BETWEEN '2024-01-01' AND '2024-12-31';  -- Good: max pruning

-- 2. Use DATE_TRUNC for partition alignment
SELECT COUNT(*), DATE_TRUNC('month', order_date) AS month
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE_TRUNC('month', order_date);
```

### Edge Cases & Caveats
- **Result cache limitations:** Cached results are NOT returned if: query uses NOW/GETDATE (use CURRENT_TIMESTAMP), query references external tables, session context changes (role, warehouse).
- **QAS eligibility:** Not all queries benefit. Use the `QUERY_ACCELERATION_ELIGIBLE` view to see which queries qualify.
- **Pruning and IN lists:** Large IN lists (>10,000 values) bypass pruning. Use a JOIN with a lookup table instead.
- **Explain plan accuracy:** Estimated rows can be off by 10-100× for skewed data.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Query Acceleration Service** | Zero config, elastic, helps large scans | 1.5× credit cost, not all queries eligible |
| **Result Cache** | Free, instant, no overhead | Only exact-match queries, 24h TTL |
| **Clustering** | Broad benefit, persistent | Credits for re-clustering, learning period |
| **Materialized Views** | Pre-computed aggregates | Limited SQL, refresh cost |
| **Warehouse Sizing** | Simple, reliable | Predictable cost but linear scaling |

### Principal-Level Considerations
- **Identify top 5 slow queries weekly:** Use `QUERY_HISTORY` to find high-duration, high-credit queries. Optimize them before they become monthly cost drivers.
- **Build a query performance baseline:** Run a representative set of queries weekly. Track metrics: avg duration, P95 duration, credits consumed. Detect regressions.
- **Cost vs. performance trade-off:** Not every query needs to run in 1 second. For background ETL, 30 minutes is fine. Focus optimization on user-facing dashboards.

### Follow-up Questions
1. **When is QAS cost-effective?**
   - **Answer:** When queries take >30 seconds and scan >1TB. If a query runs for 10 minutes and QAS reduces it to 2 minutes (5× speedup), but QAS costs 1.5× the base warehouse, the cost per byte scanned is lower but total credits may be similar. Test on a representative workload first.

2. **How does the result cache handle queries with different warehouses?**
   - **Answer:** The result cache is global across the account — different warehouses can share cached results. However, if the query uses session-specific functions (CURRENT_ROLE, CURRENT_USER), cache is invalidated per session.

3. **What is the difference between result cache and data cache (SSD cache)?**
   - **Answer:** Result cache stores final query results in cloud services layer (millisecond access, 24h TTL). Data cache (SSD) stores raw data blocks on warehouse servers (ephemeral, lost when warehouse suspends). Result cache is zero-cost; data cache is included in warehouse credits.

4. **How do you identify queries with poor pruning?**
   - **Answer:** Query `QUERY_HISTORY`:
```sql
SELECT
    query_id, query_text,
    partitions_total, partitions_scanned,
    (partitions_scanned/NULLIF(partitions_total,0))*100 AS scan_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE partitions_total > 10000
  AND (partitions_scanned/NULLIF(partitions_total,0))*100 > 50
ORDER BY partitions_total DESC;
```
A scan ratio >50% on a large table indicates poor pruning.

5. **How do you tune a slow query step-by-step?**
   - **Answer:** (1) Read `EXPLAIN` output, (2) Check partitions scanned vs total, (3) Verify clustering key matches WHERE filters, (4) Check if QAS was used, (5) Look for hash join imbalances (small table on build side), (6) Check for spill-to-disk (high bytes_spilled_to_local_storage), (7) Consider a materialized view or CTE restructure.

6. **What is query compilation time vs execution time?**
   - **Answer:** Query compilation (planning/optimizing) happens in cloud services layer (0.1-2 seconds). Execution happens in warehouse. A query spending 5 seconds compiling and 0.5 seconds executing may indicate metadata complexity. Use `EXPLAIN` to see compile time; `QUERY_HISTORY` shows both.

---

## Q14. Snowpipe vs Bulk Load (continuous vs batch, cost trade-offs)

### Question
Compare Snowpipe and bulk `COPY INTO` for data loading. When would you choose continuous loading vs batch loading, and what are the cost implications at TB/PB scale?

### Answer
Both mechanisms load staged data into Snowflake, but they serve different latency and cost profiles.

**Bulk COPY INTO:** The traditional batch load. It requires a running warehouse and explicitly invoked. Best for: hourly/daily batches, large file processing, transformation during load, retry logic control. Costs are warehouse credits based on size and duration.

**Snowpipe:** A serverless, continuous loading service. It monitors staged files and loads them automatically using Snowflake-managed compute. Billed at 2 credits/hour flat rate per pipe (on Standard Edition; higher on Enterprise). Snowpipe is ideal for: streaming ingestion, near-real-time pipelines (1-5 minute latency), variable-volume data.

**Snowpipe Streaming:** Row-level ingestion via API (not file-based). Sub-second latency. Billed at higher per-row rates. For real-time use cases requiring <1 second latency.

### Example
```sql
-- Bulk load: daily batch
CREATE OR REPLACE WAREHOUSE etl_large
    WAREHOUSE_SIZE = '4XLARGE'
    AUTO_SUSPEND = 120;

COPY INTO orders FROM @s3_stage/orders/2024/10/15/
    FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY)
    ON_ERROR = 'CONTINUE';

-- Snowpipe: continuous loading
CREATE OR REPLACE PIPE orders_pipe
    AUTO_INGEST = TRUE
AS
    COPY INTO orders
    FROM @s3_stage/orders/
    FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY);

-- Monitor Snowpipe load
SELECT *
FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
    PIPE_NAME => 'orders_pipe',
    TARGET_END_TIME => CURRENT_TIMESTAMP()
));

-- Check Snowpipe credits
SELECT
    start_time, end_time,
    credits_used,
    bytes_inserted,
    files_inserted
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE pipe_name = 'ORDERS_PIPE'
ORDER BY start_time DESC;

-- Estimate cost: If Snowpipe runs 24/7 = 2 * 24 * 30 = 1440 credits/month
-- Compare: Daily 30-min bulk load on LARGE warehouse = 4 * 0.5 * 30 = 60 credits/month
```

### Edge Cases & Caveats
- **Snowpipe minimum file size:** Files < 10MB waste Snowpipe overhead. Coalesce small files to 100MB+.
- **File ordering:** Snowpipe loads files in order of event arrival (usually alphabetical). No guarantee of strict ordering.
- **Error handling:** Snowpipe does not auto-retry failed files. Use `ON_ERROR = 'CONTINUE'` and monitor errors via `COPY_HISTORY`.
- **Warehouse requirement:** Bulk load requires running warehouse; Snowpipe does not.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Bulk COPY INTO** | Cheapest for large volumes, full control, predictable cost | Batch latency, need warehouse, manual orchestration |
| **Snowpipe (file-based)** | Serverless, auto-ingest, low latency | Flat 2 credits/hour, min file size |
| **Snowpipe Streaming** | Sub-second latency, row-level | High per-row cost, API complexity |
| **Kafka + Snowpipe** | Real-time ingestion, robust ecosystem | Infrastructure overhead |

### Principal-Level Considerations
- **TCO modeling:** For < 100 GB/day, Snowpipe may be cheaper (no warehouse). For > 1 TB/day, bulk COPY INTO is ~60-70% cheaper. Model both options.
- **Latency SLAs:** Define freshness requirements. If data must be available within 1 minute of landing, use Snowpipe Streaming. If within 1 hour, bulk load is fine.
- **Multi-phase loading:** Use Snowpipe for initial landing, then bulk load for re-processing (merge into fact tables).

### Follow-up Questions
1. **How do you ensure exactly-once semantics with Snowpipe?**
   - **Answer:** Snowpipe has at-least-once semantics by default. Use idempotent MERGE statements in the COPY INTO definition or deduplicate downstream.

2. **Can you load into the same table from Snowpipe and bulk COPY INTO concurrently?**
   - **Answer:** Yes — both can load into the same table simultaneously. However, micro-partition creation will interleave and may increase fragmentation.

3. **How does Snowpipe handle schema changes in source files?**
   - **Answer:** If the COPY INTO definition is fixed column mapping, Snowpipe will fail on schema drift. Use PARSE_JSON for flexible schema, or schema evolution.

4. **What is the maximum throughput of a single Snowpipe?**
   - **Answer:** A single pipe can process about 1-2 GB/minute. For higher throughput, create multiple pipes (e.g., one per source table, or shard by prefix).

5. **How do you monitor Snowpipe latency in production?**
   - **Answer:** Query `PIPE_USAGE_HISTORY` and calculate time between file creation and ingestion. Build a dashboard alerting on latency > 10 minutes.

6. **What triggers a Snowpipe failure and how do you recover?**
   - **Answer:** Failures include: schema mismatch, permission errors, file corruption. Snowpipe does not auto-retry. Monitor `COPY_HISTORY` for errors, fix the issue, and re-ingest manually.
---

## Q11. Materialized Views (auto-refresh, clustering, cost, query acceleration)

### Question
Explain how Snowflake Materialized Views work under the hood. How does auto-refresh work, what are the performance and cost implications, and when should you use one versus other acceleration options?

### Answer
Snowflake Materialized Views store the results of a query as a physical, clustered table, automatically maintained by Snowflake when the underlying data changes. Unlike traditional materialized views in other databases, Snowflake's are auto-refreshing — there is no manual refresh step.

**How Auto-Refresh Works:** When a base table changes (INSERT, UPDATE, DELETE, MERGE), Snowflake detects the change via micro-partition metadata. If the materialized view is defined with `CLUSTER BY` on the same columns as its query's GROUP BY or WHERE clauses, Snowflake can apply the incremental changes directly without a full rebuild. The refresh happens lazily — the next query against the materialized view triggers the refresh if needed. If the materialized view lacks a cluster key, or if the change pattern doesn't match, Snowflake does a full rebuild.

**Performance Impact:** A properly clustered materialized view can accelerate aggregations by 10-100x because Snowflake reads only the pre-computed summary data instead of scanning the base table. The materialized view is transparently used by the query optimizer — you do not need to reference it explicitly. If your query matches the materialized view's definition, Snowflake substitutes it automatically.

**Cost Implications:** Materialized views incur storage cost for the stored results and compute cost when rebuilding. Auto-refresh adds compute credits to your base table's warehouse. However, the overall cost is typically lower because accelerated queries use fewer credits than full scans.

### Example
```sql
-- Create a materialized view for a common aggregation pattern
CREATE OR REPLACE MATERIALIZED VIEW daily_sales_summary
    CLUSTER BY (sale_date)
AS
SELECT
    sale_date,
    product_category,
    region,
    COUNT(*) AS total_orders,
    SUM(sale_amount) AS total_revenue,
    AVG(sale_amount) AS avg_order_value
FROM sales
GROUP BY 1, 2, 3;

-- Snowflake auto-uses the materialized view for this query
-- (no need to reference it explicitly)
SELECT
    sale_date,
    product_category,
    COUNT(*) AS total_orders
FROM sales
WHERE sale_date >= '2024-01-01'
GROUP BY 1, 2;

-- Check if a materialized view is being used (via query plan)
EXPLAIN
SELECT sale_date, COUNT(*) FROM sales GROUP BY 1;

-- Monitor materialized view refresh activity
SELECT
    TABLE_NAME,
    CREATED,
    LAST_ALTERED,
    COMMENT
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'DAILY_SALES_SUMMARY';

-- Drop a materialized view
DROP MATERIALIZED VIEW IF EXISTS daily_sales_summary;
```

### Edge Cases & Caveats
- **Materialized view not used:** If your query doesn't exactly match the materialized view's aggregation, Snowflake won't use it. Even a different aggregate function (SUM vs AVG) means no substitution.
- **Clustering key mismatch:** If your `CLUSTER BY` doesn't match the pattern of changes, incremental refresh degrades to full rebuild, increasing costs.
- **Complex transformations:** Materialized views only support deterministic expressions. Non-deterministic functions (like `CURRENT_TIMESTAMP()`) are not allowed.
- **DML triggers full refresh:** DDL operations on base tables (ALTER, TRUNCATE) cause a full rebuild of all dependent materialized views.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Materialized View (auto-refresh)** | Transparent, incremental refresh, automatic | Storage cost, limited to deterministic expressions |
| **Result Cache (auto, 24h)** | Zero cost, automatic, covers exact queries | Only 24 hours, data must be current |
| **Dynamic Tables (newer)** | Declarative refresh targets, SLA-based | Still evolving, limited SQL constructs |
| **Stream + Task (manual refresh)** | Full control, custom logic, any SQL | No transparency, requires orchestration |

### Principal-Level Considerations
- **Materialized view proliferation risk:** Every materialized view adds storage and maintenance compute cost. At a Principal level, set governance: require justification, measure acceleration benefit, retire unused ones.
- **Cluster key design:** Choose clustering keys that align with the most common filter patterns. A poorly clustered materialized view is worse than no materialized view (adds storage + compute with no benefit).
- **Transparency double-edged sword:** Developers may not know a materialized view is being used, leading to confusion when query results are stale (during rebuild). Document all materialized views in a data catalog.

### Follow-up Questions
1. **How does Snowflake decide whether to use a materialized view?**
   - **Answer:** Snowflake's query optimizer evaluates if your query is equivalent to the materialized view's definition. It checks if the aggregation, filters, and GROUP BY match. If there's a mismatch, Snowflake falls back to the base table. You can verify with `EXPLAIN`.

2. **Can a materialized view depend on another materialized view?**
   - **Answer:** Yes — chained materialized views are supported. Snowflake refreshes them in dependency order. However, chained materialized views are harder to reason about and may have unpredictable refresh latency.

3. **What happens if a materialized view's base table has frequent schema changes?**
   - **Answer:** Schema changes on base tables that affect the materialized view's definition cause the materialized view to be invalidated and rebuilt. Adding new columns is fine; dropping columns used by the materialized view breaks it.

4. **How much storage do materialized views consume?**
   - **Answer:** Materialized views store pre-computed data, so storage depends on the aggregation granularity. A daily aggregation of a 1TB table might be 10-100 GB. Check `ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY` and `TABLE_STORAGE_METRICS`.

5. **What are the performance characteristics of incremental vs. full refresh?**
   - **Answer:** Incremental refresh is proportional to the number of changed rows (fast, low compute). Full rebuild is proportional to the entire base table (slow, high compute). For high-churn tables, incremental refresh saves significant cost.

6. **Can you force a materialized view refresh?**
   - **Answer:** There is no direct `REFRESH MATERIALIZED VIEW` command in Snowflake. The refresh is automatic. To force a refresh, you can query the materialized view, which triggers a lazy refresh if needed.

---

## Q12. Secure Views & Dynamic Data Masking (access control, PII, RLS)

### Question
How do Secure Views, Dynamic Data Masking, and Row-Level Security work in Snowflake? Design an access control strategy that protects PII while maintaining query performance.

### Answer
Snowflake provides three complementary security mechanisms for protecting sensitive data:

**Secure Views:** A Secure View hides the underlying query definition. Users can query the view but cannot see the tables, columns, or transformation logic used. This is ideal for sharing data with external users or restricting access to specific columns. Secure views enforce the `SELECT` privilege — only users with `USAGE` on the schema and `SELECT` on the view can query it.

**Dynamic Data Masking:** Data Masking Policies transform data on-the-fly based on the user's role, context, or criteria. Unlike views, masking applies at the column level and works across all access patterns (views, tables, joins). Masking policies are functions you define and attach to columns. Snowflake evaluates the policy function for each row returned, applying transformations like redaction, hashing, or value substitution.

**Row-Level Security:** Implement row-level security using views with WHERE clauses filtered by `CURRENT_ROLE()`, `CURRENT_USER()`, or application context. Snowflake does not have a built-in RLS feature, so it's pattern-based.

### Example
```sql
-- Create a secure view
CREATE OR REPLACE SECURE VIEW public_sales_summary AS
SELECT
    product_category,
    sale_date,
    COUNT(*) AS order_count,
    SUM(sale_amount) AS revenue
FROM analytics.facts.sales
GROUP BY 1, 2;

-- Users can query the view but cannot see the underlying table
SELECT * FROM public_sales_summary;
DESCRIBE VIEW public_sales_summary;  -- Shows columns, not the query

-- Dynamic Data Masking: Masking policy for email
CREATE OR REPLACE MASKING POLICY mask_email AS
    (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'DATA_ENGINEER', 'ANALYST_ADMIN') THEN val
        WHEN CURRENT_ROLE() = 'ANALYST' THEN CONCAT(SUBSTR(val, 1, 2), '***@***.***')
        WHEN CURRENT_ROLE() = 'REPORT_USER' THEN 'REDACTED'
        ELSE 'REDACTED'
    END;

-- Apply masking to a column
ALTER TABLE users
    ALTER COLUMN email SET MASKING POLICY mask_email;

-- Test masking from different roles
USE ROLE DATA_ENGINEER;
SELECT email FROM users LIMIT 5;  -- Shows real emails

USE ROLE ANALYST;
SELECT email FROM users LIMIT 5;  -- Shows john***@***.***

USE ROLE REPORT_USER;
SELECT email FROM users LIMIT 5;  -- Shows REDACTED

-- Row-level security using a view
CREATE OR REPLACE SECURE VIEW dept_orders AS
SELECT
    order_id,
    customer_id,
    sale_amount,
    region
FROM facts.orders
WHERE region = (
    SELECT region FROM dims.employee
    WHERE user_id = CURRENT_USER()
);

-- Tag-based masking (Enterprise+)
CREATE TAG sensitivity_level ON ACCOUNT;
ALTER TABLE users SET TAG sensitivity_level = 'PII';

-- Query to find all PII-tagged objects
SELECT * FROM DATABASE_NAMESPACES
WHERE CONTAINS(REFERENCED_NAMESPACES, 'PII');
```

### Edge Cases & Caveats
- **Secure views and performance:** Secure views are treated the same as regular views by the optimizer. If the underlying query is complex, performance is the same.
- **Masking policies and joins:** When a masked column is used in a JOIN, the masking is applied before the join. Use unmasked columns for join keys.
- **Row-level security overhead:** Views with subqueries in WHERE clauses can slow queries. Use materialized views or dimension tables for fast lookups.
- **Masking and data exports:** Masking policies apply during exports (COPY INTO, SnowSQL) — data is masked in the output.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Secure Views** | Simple, hides logic, works with any data | View-based, not transparent for existing queries |
| **Dynamic Data Masking** | Column-level, role-aware, transparent across all access | Policy management overhead, performance impact on large result sets |
| **Row-Level Security Views** | Fine-grained access per user | Complex WHERE logic slows queries, hard to maintain |
| **Data Sharing (Reader Accounts)** | Data stays in source account, no copy | Latency, limited to read access |

### Principal-Level Considerations
- **Masking policy governance:** Maintain a centralized registry of masking policies. Audit quarterly. Require approval for masking policy changes.
- **Masking vs. anonymization:** Masking is reversible (data is there, just hidden in query results). For true anonymization, use a separate anonymized table with irreversible transformations (hashing, generalization).
- **Performance budget:** Each masking policy adds function evaluation per row. For large result sets (100M+ rows), the overhead is measurable. Optimize by masking only in the presentation layer, not in raw data warehouses.

### Follow-up Questions
1. **Can you apply multiple masking policies to the same column?**
   - **Answer:** No — only one masking policy per column. If you need multiple transformations, combine them into a single policy function.

2. **How do masking policies interact with views?**
   - **Answer:** Masking policies apply at the column level regardless of how the column is accessed (direct table query, view, join). The policy is evaluated at query time. If a view selects a masked column, the masking is applied in the view result.

3. **What is the performance impact of masking policies?**
   - **Answer:** Masking is a per-row function evaluation. For 100K rows, add ~100-500ms. For 100M rows, add ~10-60 seconds. Keep masking policies simple and use them on columns in the presentation layer, not on raw data.

4. **How do you implement column-level access control?**
   - **Answer:** Use a combination of secure views (for column subset) and masking policies (for value transformation). Grants `SELECT` on the view but not the underlying table.

5. **Can masking policies use session variables or application context?**
   - **Answer:** Yes — use `SESSION_VARIABLE`, `CURRENT_ROLE()`, `CURRENT_USER()`, `CURRENT_ACCOUNT()`, or custom session parameters (`SYSTEM$GET_IGNORED_SESSION_PARAMETER`).

6. **How do you audit who accessed PII data?**
   - **Answer:** Use `ACCOUNT_USAGE.QUERY_HISTORY` joined with `ACCOUNT_USAGE.ACCESS_HISTORY` to track who queried which objects. Set up alerts for unexpected access patterns.

---

## Q13. Resource Monitors & Credit Governance (budget control, alerts, throttling)

### Question
How do Resource Monitors work in Snowflake? How do you implement a cost governance strategy using resource monitors, and what are the different throttle actions?

### Answer
Resource Monitors are Snowflake's primary cost governance mechanism. They track credit consumption against configurable thresholds and take action when limits are reached.

**How Resource Monitors Work:** A resource monitor tracks cumulative credit usage over a time window (daily, weekly, monthly, yearly). When usage reaches a threshold, the monitor triggers a configurable action. You can attach resource monitors to warehouses so that when the organization-wide or warehouse-specific limit is reached, specific warehouses are affected.

**Threshold Levels & Actions:**
1. **THROTTING (default):** Warehouse cannot start new queries; running queries complete normally. Existing sessions are not affected.
2. **SUSPEND:** Suspends all warehouses associated with the resource monitor.
3. **Suspend Individual Warehouses:** You can configure different monitors for different warehouses.

**Multi-Level Strategy:** Create a hierarchical monitoring approach — an organization-wide budget monitor and department-level monitors.

### Example
```sql
-- Create a monthly organization-wide resource monitor
CREATE OR REPLACE RESOURCE MONITOR org_budget_monitor
    WAREHOUSE = ALL
    CREDIT_QUOTA = 10000          -- 10,000 credits per month
    FREQUENCY = MONTHLY
    TRIGGERS =
        ALERT AFTER 80 PERCENT,
        THROTTING AFTER 95 PERCENT,
        SUSPEND AFTER 100 PERCENT;

-- Create a department-level resource monitor
CREATE OR REPLACE RESOURCE MONITOR analytics_budget_monitor
    WAREHOUSE = analytics_wh
    CREDIT_QUOTA = 3000
    FREQUENCY = MONTHLY
    TRIGGERS =
        THROTTING AFTER 90 PERCENT,
        SUSPEND AFTER 100 PERCENT;

-- Attach resource monitor to a specific warehouse
ALTER WAREHOUSE analytics_wh SET RESOURCE_MONITOR = analytics_budget_monitor;

-- Monitor resource usage
SELECT
    resource_monitor_name,
    credits_used,
    credits_allowed,
    next_reset_time,
    triggered_action
FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_METERING_HISTORY())
ORDER BY start_time DESC;

-- Set up an alert via a task
CREATE OR REPLACE TASK check_budget_usage
    WAREHOUSE = utility_wh
    SCHEDULE = 'USING CRON 0 8 * * *'
AS
    INSERT INTO monitoring.budget_alerts
    SELECT
        CURRENT_TIMESTAMP(),
        r.resource_monitor_name,
        r.credits_used,
        r.credits_allowed,
        (r.credits_used / r.credits_allowed * 100)::INT AS pct_used,
        r.triggered_action
    FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())) r;

-- Query resource monitor history
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
WHERE resource_monitor_name = 'ORG_BUDGET_MONITOR'
ORDER BY reset_time_start DESC;
```

### Edge Cases & Caveats
- **Reset behavior:** A monthly resource monitor resets on the 1st of each month. Daily monitors reset at midnight. Plan accordingly.
- **Multiple monitors on a warehouse:** If both org-wide and warehouse-specific monitors exist, the most restrictive applies.
- **Credit quota granularity:** The minimum credit quota is 1 credit. For small warehouses, this may be too coarse.
- **Resource monitors and cloning:** Cloning warehouses inherits the resource monitor.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Resource Monitor (built-in)** | Native, automatic, multiple thresholds | Rigid time windows, no per-user budgeting |
| **Custom billing dashboard** | Flexible, per-project/user, real-time | Requires development, no automatic throttling |
| **Cloud cost tools (FinSpace, etc.)** | Cloud-agnostic, multi-account | Lag, no real-time throttling |
| **Warehouse size + auto-suspend** | Simple, cost-effective | No budget enforcement, reactive only |

### Principal-Level Considerations
- **Budget hierarchy:** At scale, organize budgets by team or project. Create a top-level org budget and sub-monitor per team. Use `WAREHOUSE_LOAD_HISTORY` to attribute costs.
- **Alerting strategy:** Set up alerts at 80%, 95%, and 100%. At 80%, notify team leads. At 95%, notify engineering managers. At 100%, suspend.
- **Budget transparency:** Publish a monthly cost dashboard per team. Include: credits used, query count, top warehouses, storage costs. Drive cost-conscious culture.

### Follow-up Questions
1. **Can a resource monitor throttle only certain warehouses while leaving others running?**
   - **Answer:** Yes — attach a resource monitor to specific warehouses instead of ALL. When the monitor triggers, only the associated warehouses are affected. Other warehouses continue running.

2. **How do you handle unexpected credit spikes from ad-hoc queries?**
   - **Answer:** Use `STATEMENT_TIMEOUT_IN_SECONDS` to limit long-running queries. Set `QUERY_TAG` for cost attribution. Implement query size limits on ad-hoc warehouses.

3. **What is the difference between THROTTING and SUSPEND actions?**
   - **Answer:** THROTTING prevents new queries from starting but allows running queries to complete. SUSPEND stops the warehouse entirely — even running queries are terminated.

4. **How do you attribute costs to specific teams or projects?**
   - **Answer:** Use `QUERY_TAG` to tag every session or task with a team name. Query `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` to aggregate costs by tag.

5. **Can you create per-user resource limits?**
   - **Answer:** Not directly — Snowflake doesn't have per-user credit budgets. Workaround: create dedicated warehouses per user/team, each with its own resource monitor.

6. **What happens to running queries when a resource monitor suspends a warehouse?**
   - **Answer:** The warehouse is suspended immediately. Running queries are terminated and return an error. Use THROTTING if you want running queries to complete.

---

## Q14. Performance Optimization (query tuning, pruning, clustering, session parameters)

### Question
What is your systematic approach to diagnosing and fixing slow Snowflake queries? Cover pruning, clustering keys, warehouse sizing, and query optimization techniques.

### Answer
Performance optimization in Snowflake requires understanding the interaction between data layout, query patterns, and compute resources. My approach is: diagnose → isolate → fix → validate.

**Diagnostic Phase:**
1. **Query Profile:** Run `EXPLAIN` and check the query profile (in the UI). Identify the bottleneck: data scanning, join processing, or aggregation.
2. **Micro-partition pruning:** Check if Snowflake is scanning fewer partitions than expected. Poor pruning is the #1 cause of slow queries.
3. **Warehouse sizing:** Check if the warehouse is the right size. A too-small warehouse serializes queries; too-large wastes credits.

**Common Optimization Techniques:**
- **Clustering Keys:** For tables queried with range filters on dates or IDs, clustering can dramatically reduce scanning. But clustering must match query patterns.
- **Session Parameters:** Use `STATEMENT_TIMEOUT_IN_SECONDS`, `QUERY_TAG`, and `QUERY_ACCELERATION` strategically.
- **Query Rewriting:** Break complex queries into CTEs, use temporary tables for intermediate results, avoid unnecessary functions.

### Example
```sql
-- Check query profile for pruning effectiveness
SELECT
    query_id,
    query_text,
    execution_status,
    total_elapsed_time,
    bytes_scanned,
    bytes_spilled_to_local_disk,
    bytes_spilled_to_remote_storage,
    partitions_total,
    partitions_scanned
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Check pruning ratio (low = good, high = bad)
SELECT
    query_id,
    partitions_scanned / NULLIF(partitions_total, 0) AS prune_ratio
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE warehouse_name = 'ANALYTICS_WH'
  AND execution_status = 'SUCCESS'
ORDER BY partitions_scanned / NULLIF(partitions_total, 0) DESC
LIMIT 10;

-- Identify the best clustering key for a table
-- Check which columns are most frequently used in filters
SELECT
    table_name,
    clustering_key,
    cluster_by_columns
FROM INFORMATION_SCHEMA.TABLES
WHERE table_name = 'SALES';

-- Use a clustered materialized view for common filter patterns
CREATE MATERIALIZED VIEW sales_by_date_product
    CLUSTER BY (sale_date, product_id)
AS
SELECT
    sale_date,
    product_id,
    SUM(sale_amount) AS revenue,
    COUNT(*) AS order_count
FROM sales
GROUP BY 1, 2;

-- Optimize with query acceleration (Enterprise+)
ALTER MATERIALIZED VIEW sales_by_date_product SET SCALING_FACTOR = 32;

-- Use session parameters for query control
ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 300;
ALTER SESSION SET QUERY_TAG = 'finance_monthly_report';

-- Rewrite a slow query using a temp table
CREATE TEMP TABLE temp_sales_agg AS
SELECT
    sale_date,
    product_category,
    SUM(sale_amount) AS revenue
FROM sales
WHERE sale_date >= '2024-01-01'
GROUP BY 1, 2;

-- Then use the temp table in the final query
SELECT
    t.sale_date,
    t.product_category,
    t.revenue,
    d.category_name
FROM temp_sales_agg t
JOIN dim_products d ON t.product_category = d.category_code;
```

### Edge Cases & Caveats
- **Over-clustering:** If a table has too many clustering keys, the clustering overhead exceeds the benefit. Monitor `CLUSTERING_DEPTH` — values below 100 indicate good clustering.
- **Query acceleration cost:** Query acceleration can cost more than using a larger warehouse for the same query. Measure before enabling.
- **Materialized view refresh overhead:** A materialized view with frequent refresh can cost more than the base table scans it's meant to avoid.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Clustering Key (automatic/manual)** | Transparent, reduces scanning | Clustering compute cost, must match query patterns |
| **Query Acceleration** | Up to 50x faster for some queries | Extra cost, limited to specific queries |
| **Larger Warehouse** | More parallelism, faster | Proportional credit cost |
| **Materialized Views** | Pre-computed results, transparent | Storage + refresh cost, limited SQL |

### Principal-Level Considerations
- **Performance SLAs:** Define performance targets per query class (ad-hoc < 30s, reports < 60s, ETL < 10min). Use `QUERY_HISTORY` to measure compliance.
- **Query governance:** Implement query review for new complex queries. Require `EXPLAIN` output. Ban anti-patterns: SELECT *, unbounded scans.
- **Cost-performance tradeoff:** The cheapest query is one that doesn't run. Optimize query frequency, not just query speed. Cache results where possible.

### Follow-up Questions
1. **How do you know if a clustering key is effective?**
   - **Answer:** Check the pruning ratio (partitions_scanned/partitions_total). A good ratio is < 5% for filtered queries. Use `CLUSTERING_DEPTH` — lower is better (< 100). Monitor `ACCOUNT_USAGE.CLUSTERING_INFORMATION`.

2. **What is the difference between automatic and manual clustering?**
   - **Answer:** Automatic clustering (Snowflake Enterprise+) is set with `ALTER TABLE ... CLUSTER BY` and Snowflake automatically maintains it. Manual clustering requires the `CLUSTER` command. Automatic is recommended; manual is deprecated.

3. **How does Snowflake optimize joins?**
   - **Answer:** Snowflake uses a distributed join algorithm. For hash joins, data is partitioned by the join key across virtual warehouses. For broadcast joins, the smaller table is broadcast to all nodes. The optimizer chooses based on table sizes and selectivity.

4. **What are the top 5 anti-patterns in Snowflake SQL?**
   - **Answer:** (1) SELECT * on large tables, (2) unbounded scans (no WHERE filter), (3) unnecessary subqueries, (4) using functions on join columns, (5) excessive semi-structured data extraction in filters.

5. **How do you handle slow ETL pipelines?**
   - **Answer:** Profile each step. Break large transformations into smaller stages. Use multiple warehouses for parallel processing. Consider Snowpark for complex transformations.

6. **When should you use a temporary table vs a CTE?**
   - **Answer:** Use CTEs for simple, read-only transformations. Use temporary tables for complex intermediate results that are referenced multiple times, or when you want to materialize an intermediate step for debugging.
---

## Q15. Snowpipe vs Bulk Loading (continuous vs. batch, cost, latency, patterns)

### Question
Compare Snowpipe (automated continuous loading) with bulk COPY INTO for data loading. When would you choose each, and what are the cost implications at scale?

### Answer
Both Snowpipe and bulk COPY INTO serve distinct loading patterns. The choice depends on latency requirements, cost tolerance, and data volume.

**Bulk COPY INTO (Batch Loading):** Loading is explicitly triggered (manual, scheduled task, or external orchestrator). Best for high-volume batch loads (hundreds of GB to TB) where latency of minutes to hours is acceptable. Loading costs are bound to the warehouse. You control parallelism via warehouse sizing and can retry failed loads easily.

**Snowpipe (Continuous Loading):** Automatically loads files as they appear in a stage. Best for incremental, near-real-time loading (5-minute latency). Snowpipe is serverless: Snowflake allocates credits automatically (2 credits/hour per pipe). It is designed for continuous, smaller file loads.

**Cost Comparison:**
- Bulk COPY INTO on a medium warehouse (16 credits/hour): $6-8/hour. Loading 100 GB takes ~15 minutes = ~$1.50-2.00.
- Snowpipe (2 credits/hour, billed per second): Loading 100 GB continuously over 24 hours = ~$16-24. But if the data arrives in bursts, Snowpipe only runs when files land.

**Latency Comparison:**
- Bulk COPY INTO: Latency = schedule interval + load time (e.g., 1 hour + 15 min).
- Snowpipe: < 5 minutes after file arrives in stage.
- Snowpipe Streaming: < 1 second (row-level API).

### Example
```sql
-- Bulk COPY INTO (scheduled daily)
CREATE OR REPLACE TASK daily_load_sales
    WAREHOUSE = load_wh
    SCHEDULE = 'USING CRON 0 3 * * * America/New_York'
AS
    COPY INTO sales_fact
    FROM @s3_data_stage/sales/
    PATTERN = '.*\.parquet'
    FILE_FORMAT = (FORMAT_NAME = parquet_format);

-- Snowpipe setup (continuous)
-- 1. Create pipe
CREATE OR REPLACE PIPE sales_pipe
    AUTO_INGEST = TRUE
    AWS_SNS_TOPIC = 'arn:aws:sns:us-east-1:123456789012:s3_event_topic'
AS
    COPY INTO sales_fact
    FROM @s3_data_stage/sales/
    FILE_FORMAT = (FORMAT_NAME = parquet_format);

-- 2. Monitor pipe status
SELECT
    pipe_name,
    pipe_status,
    notification_channel_name,
    pending_file_count
FROM INFORMATION_SCHEMA.PIPES
WHERE pipe_name = 'SALES_PIPE';

-- 3. Check pipe history
SELECT
    pipe_name,
    file_name,
    row_count,
    error_count,
    first_error_message
    status,
    last_scheduled_time
FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
    DATE_RANGE_START => DATEADD('day', -7, CURRENT_TIMESTAMP()),
    PIPE_NAME => 'sales_pipe'
));

-- 4. Bulk validation before switching to Snowpipe
SELECT FILE_FORMAT = parquet_format;
SELECT
    validation_result
FROM TABLE(
    VALIDATE_PIPE_LOAD(
        PIPE_NAME => 'sales_pipe',
        START_TIME => DATEADD('hour', -24, CURRENT_TIMESTAMP())
    )
);
```

### Edge Cases & Caveats
- **Snowpipe file size minimum:** 10 MB compressed per file. Smaller files waste credits and increase pipe scheduling overhead.
- **Retry logic:** Snowpipe does NOT retry failed loads automatically. Implement a dead-letter queue for failed files.
- **Concurrent loads:** Bulk COPY INTO to the same table is serialized (one at a time). Snowpipe supports concurrent loads but with eventual consistency.
- **Cost at low volume:** Loading 100 MB/day with Snowpipe costs ~$2/day (2 credits/hour × 0.01 hours × $3/credit). Bulk COPY INTO on a small warehouse for 5 minutes costs ~$0.30.

### Trade-offs & Alternatives

| Aspect | Bulk COPY INTO | Snowpipe | Snowpipe Streaming |
|--------|---------------|----------|--------------------|
| **Latency** | Minutes-hours | ~5 min | < 1 sec |
| **Cost** | Warehouse credits | 2 credits/hr/pipe | Per-row cost |
| **File size** | 100-250 MB optimal | 10+ MB | Row-level |
| **Retry** | Manual retry | No auto-retry | Depends on client |
| **Best for** | Batch ETL, high volume | Near-real-time, incremental | Real-time, streaming events |

### Principal-Level Considerations
- **Lakehouse pattern:** Use Snowpipe for raw ingestion from sources (low latency), then bulk COPY INTO for high-volume transformation outputs.
- **Cost allocation at scale:** At 10+ Snowpipes, cost becomes significant. Evaluate if the latency SLA can be met with hourly bulk loads.
- **Error monitoring:** Build a centralized error log for Snowpipe failures. Use `PIPE_USAGE_HISTORY` to track errors daily.

### Follow-up Questions
1. **How does Snowpipe handle duplicate files?**
   - **Answer:** Snowpipe tracks metadata (file name, size, last-modified timestamp). Duplicate files are skipped. To force re-load, use `ALTER PIPE ... SET PIPE_EXECUTION_PAUSED = FALSE` and re-add the file.

2. **What is the maximum number of Snowpipes per account?**
   - **Answer:** The default limit is 1,000 pipes per account. This can be increased by contacting Snowflake support. Beyond 100, consider consolidating or using bulk loading.

3. **How do you monitor Snowpipe costs?**
   - **Answer:** Use `SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY` to see credits per pipe. Join with `METERING_HISTORY` for cost attribution by pipe.

4. **Can you use Snowpipe with multiple file formats?**
   - **Answer:** Yes — each pipe can handle one file format. Create separate pipes for CSV, JSON, Parquet. Each pipe's COPY statement defines the format.

5. **How does Snowpipe handle files that arrive during a warehouse suspension?**
   - **Answer:** Snowpipe does not use a warehouse. If the pipe is paused or deleted, files accumulate in the stage. When the pipe is resumed, they are loaded in order.

6. **What happens if Snowpipe encounters a corrupt file?**
   - **Answer:** The file is skipped (ON_ERROR defaults to SKIP_FILE). The error is logged in `PIPE_USAGE_HISTORY`. The file remains in the stage; you must handle it manually or via a dead-letter process.

---

## Q16. Result Caching & Query Acceleration (automatic, 24-hour, transparent)

### Question
Explain Snowflake's Result Cache: how it works, what invalidates it, and when it automatically kicks in. How does it differ from Query Acceleration and Warehouse Caching?

### Answer
Snowflake's Result Cache stores the result set of every query that completes successfully. For 24 hours, if an identical query is executed and the underlying data hasn't changed, the result is returned from cache instantly — zero compute cost.

**How It Works:** When a query finishes, Snowflake stores:
- The result rows (in compressed form)
- The query text (canonicalized, including whitespace and capitalization)
- The timestamp of the version of each table used
- The role and warehouse context

On the next execution, Snowflake canonicalizes the new query, compares it to cached queries, checks if no DML has altered the referenced tables, and if the role/warehouse context matches. If all match, the result is served from cache.

**Invalidation Triggers:**
1. DML on any referenced table (INSERT, UPDATE, DELETE, MERGE, TRUNCATE)
2. DDL on any referenced table (ALTER, DROP, CREATE)
3. Role or warehouse context change
4. Non-deterministic functions (CURRENT_TIMESTAMP, RANDOM, UUID_STRING)
5. Session-level parameters that could alter results (e.g. TIMEZONE change)

**Cost Savings:** Result Cache is free. No warehouse credits are consumed. This is particularly impactful for repeated dashboards, monitoring queries, and parameterized reports where many users see the same result set.

### Example
```sql
-- Run a query once
SELECT
    sale_date,
    product_category,
    SUM(sale_amount) AS revenue
FROM sales
WHERE sale_date = '2024-06-15'
GROUP BY 1, 2;

-- Run the exact same query again within 24 hours
-- Result is served from cache (0 credits, instant)

-- Check if result cache was used
SELECT
    query_id,
    query_text,
    total_elapsed_time,
    credits_used_cloud_services
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Force cache bypass by adding a unique session variable
ALTER SESSION SET QUERY_TAG = 'no_cache_' || UUID_STRING();
-- (Not recommended — this defeats the cache purpose)

-- Invalidate cache by running DML on the table
INSERT INTO sales (
    sale_id, sale_date, product_category, sale_amount, region
) VALUES (99999, '2024-06-15', 'Electronics', 299.99, 'West');

-- Now the same SELECT query against sales will NOT use cache
-- (because the table state changed)

-- Check if a cached result exists for a query
SELECT
    query_id,
    execution_status,
    cached_result
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE total_elapsed_time < 100  -- Very fast queries are likely cached
  AND query_type = 'SELECT'
ORDER BY start_time DESC
LIMIT 10;

-- Using RESULT_SCAN to re-read cached results
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

### Edge Cases & Caveats
- **Cache and UNION queries:** If part of a UNION references a non-deterministic function, the entire UNION result is not cached.
- **Cache and role context:** Two users in different roles may not share a cache, even if the query is identical, because role context matters.
- **Cache and time-based queries:** Queries with `CURRENT_TIMESTAMP` are never cached (non-deterministic).
- **Cache and large results:** Snowflake may not cache results larger than ~10 GB (estimated). Very large result sets are recomputed.

### Trade-offs & Alternatives

| Aspect | Result Cache | Query Acceleration | Warehouse Cache |
|--------|-------------|-------------------|-----------------|
| **Duration** | 24 hours or until DML | Per query | Until warehouse is suspended |
| **Cost** | Free | Extra per query | Free (included in warehouse) |
| **Scope** | Exact query match | Similar queries | Same table queries |
| **Transparency** | Automatic | Automatic (must be enabled) | Automatic |
| **Best for** | Repeated dashboards, reports | Long-running scans, joins | ETL, transformations |

### Principal-Level Considerations
- **Cache-aware dashboard design:** Design BI dashboards to reuse identical query parameters. Avoid appending random timestamps or usernames to queries — this breaks cache.
- **Cache invalidation strategy:** For near-real-time dashboards, schedule cache refresh by running a lightweight "ping" query. For batch reporting, rely on 24-hour cache.
- **Cache monitoring:** Track cache hit ratio via `QUERY_HISTORY`. A low hit ratio (< 30%) suggests query diversity is high — consider if you need caching.

### Follow-up Questions
1. **How do you disable the Result Cache?**
   - **Answer:** Set `USE_CACHED_RESULT = FALSE` at session or account level.
```sql
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
```

2. **Does the Result Cache persist across warehouse suspend/resume?**
   - **Answer:** Yes — the Result Cache is in the Cloud Services Layer, not in the warehouse. It persists across warehouse operations.

3. **How does the Result Cache interact with Secure Views?**
   - **Answer:** Caching works the same for secure views. However, if the view is defined with a masking policy, the cache is evaluated per role — different roles get different cached results (or no cache at all).

4. **Can you share a cached result across accounts?**
   - **Answer:** No — the Result Cache is per-account. Cross-account data sharing does not share cached results.

5. **What happens to the Result Cache after a Time Travel operation?**
   - **Answer:** If you query a table using `AT(OFFSET => ...)`, the result is cached. If the same time-travel query is run again, cache is used. But if data changes, the cache for the time-travel query is invalidated independently of the current table's cache.

6. **How do you measure the cost savings from the Result Cache?**
   - **Answer:** Query `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` and sum credits_used_cloud_services for queries that returned from cache (where `total_elapsed_time < 100ms` and `execution_status = 'SUCCESS'`).

---

## Q17. Snowpark (DataFrame API, stored procedures, UDFs, ML)

### Question
What is Snowpark and how does it extend Snowflake beyond SQL? Use cases for Snowpark DataFrame API, stored procedures, and ML integration.

### Answer
Snowpark is Snowflake's developer framework that allows non-SQL languages (Python, Java, Scala) to process data inside Snowflake's elastic compute engine — without moving data out.

**Key Components:**
1. **DataFrame API:** A lazy-evaluation, chainable API similar to PySpark DataFrames. You build transformations declaratively; Snowpark compiles them to SQL and executes on Snowflake's virtual warehouses.
2. **Stored Procedures (Python/Java/Scala):** Server-side programs that run inside Snowflake. They can execute multiple SQL commands, orchestrate pipelines, and handle complex logic.
3. **User-Defined Functions (UDFs):** Custom functions written in Python (with Anaconda packages), Java, or Scala that run on each row.
4. **ML:** Snowpark ML provides model training and inference inside Snowflake (using XGBoost, LightGBM, scikit-learn). Models are stored in a Snowflake model registry.

**DataFrame vs SQL:** Snowpark DataFrames give you programmatic control (loops, conditionals, imports of Python libraries) while running as optimized SQL underneath.

### Example
```sql
-- Python UDF for sentiment analysis
CREATE OR REPLACE FUNCTION sentiment_score(text STRING)
    RETURNS FLOAT
    LANGUAGE PYTHON
    RUNTIME_VERSION = '3.10'
    PACKAGES = ('textblob')
    HANDLER = 'compute_sentiment'
AS
$$
from textblob import TextBlob
def compute_sentiment(text):
    if not text:
        return 0.0
    return TextBlob(text).sentiment.polarity
$$;

-- Use the UDF in a query
SELECT
    review_id,
    review_text,
    sentiment_score(review_text) AS polarity
FROM product_reviews
WHERE sentiment_score(review_text) < -0.5;

-- Java UDF for custom transformation
CREATE OR REPLACE FUNCTION mask_phone(phone VARCHAR)
    RETURNS VARCHAR
    LANGUAGE JAVA
    HANDLER = 'PhoneMask.mask'
AS
$$
class PhoneMask {
    public static String mask(String phone) {
        if (phone == null || phone.length() < 4) return phone;
        return "XXX-XXX-" + phone.substring(phone.length() - 4);
    }
}
$$;

-- Python stored procedure for complex orchestration
CREATE OR REPLACE PROCEDURE monthly_aggregation_proc()
    RETURNS STRING
    LANGUAGE PYTHON
    RUNTIME_VERSION = '3.10'
    PACKAGES = ('snowflake-snowpark-python')
    HANDLER = 'run'
AS
$$
from snowflake.snowpark import Session

def run(session: Session):
    # Create monthly aggregation
    session.sql("""
        CREATE OR REPLACE TABLE monthly_sales_stats AS
        SELECT
            DATE_TRUNC('month', sale_date) AS sale_month,
            product_category,
            COUNT(*) AS orders,
            SUM(sale_amount) AS revenue,
            AVG(sale_amount) AS avg_order_value
        FROM sales
        GROUP BY 1, 2
    """).collect()
    return "Monthly aggregation complete"
$$;

-- Call the procedure
CALL monthly_aggregation_proc();
```

### Edge Cases & Caveats
- **UDF performance:** Python UDFs are ~10-100x slower than native SQL operators. Use for non-SQL logic only.
- **Snowpark DataFrame compilation:** Large DataFrames can produce overly complex SQL. Monitor execution plan.
- **Package restrictions:** Only Anaconda-approved packages are available. Custom packages require workarounds.
- **Stored procedure concurrency:** Procedures run on the warehouse. Long-running procedures block other warehouse queries.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Snowpark Python** | Rich ecosystem, ML integration | Slower than SQL for simple operations |
| **Snowpark Java/Scala** | Type-safe, JVM ecosystem | Less community content |
| **UDFs** | Custom logic, code reuse | Performance overhead per row |
| **Stored Procedures** | Server-side, orchestration, no data movement | Debugging complexity, concurrency issues |

### Principal-Level Considerations
- **Snowpark adoption strategy:** Start with Python UDFs for custom transformations. Move to Snowpark DataFrames for pipeline orchestration. Avoid using Snowpark for trivial SQL operations.
- **ML governance:** Snowpark ML models are stored as Snowflake objects. Implement model versioning, drift monitoring, and approval workflow.
- **Team enablement:** Train data engineers on Snowpark DataFrame API. Establish coding standards: prefer SQL for simple transformations, Snowpark for complex logic.

### Follow-up Questions
1. **How does Snowpark handle large-scale Python transformations?**
   - **Answer:** UDFs run row-level (each row is a function call) — this can be slow. For transformations on batches, use stored procedures with Snowpark DataFrames, which compile to SQL and run on the warehouse engine.

2. **What are the limitations of Python UDFs in Snowflake?**
   - **Answer:** (1) Performance: ~1M rows/second vs SQL's ~100M rows/second. (2) Package restrictions (Anaconda). (3) No GPU access. (4) Memory limit of ~40 MB per UDF execution.

3. **Can Snowpark replace dbt or Airflow?**
   - **Answer:** Partially. Snowpark is good for programmatic transformations and orchestration (replace portions of Airflow DAGs). dbt excels at SQL-based transformations with lineage tracking. Use both: dbt for SQL, Snowpark for Python logic.

4. **How do you debug Snowpark code?**
   - **Answer:** Use `session.sql("EXPLAIN SELECT ...").show()` to inspect generated SQL. Use local testing with Snowpark local mode. Add print statements in stored procedures (they show in query logs).

5. **What is the best practice for training ML models in Snowflake?**
   - **Answer:** Use Snowpark ML's `snowflake.ml.modeling` module for distributed training. Split data into training/test within Snowflake. Log hyperparameters and metrics in a model registry table.

6. **How does Snowpark handle vectorized data processing?**
   - **Answer:** Snowpark DataFrames compile to columnar SQL operations. Python UDFs can use `vectorized=True` decorator to process batches instead of rows, improving performance by 10-100x.

---

## Q18. Data Sharing & Reader Accounts (collaboration data clean room)

### Question
How does Snowflake's Data Sharing work? Explain the difference between reader accounts and standard data sharing, and how you would design a data clean room for cross-org collaboration.

### Answer
Snowflake's Data Sharing allows you to share data with other Snowflake accounts without copying or transferring data. The data remains in your account; consumers only see what you grant.

**Standard Data Sharing:** Both parties have Snowflake accounts. The provider creates a share (database + schema/objects) and grants access to the consumer's account. The consumer imports the share as a read-only database. No data movement, no storage cost for the consumer, zero ETL.

**Reader Accounts:** For non-Snowflake consumers (or accounts in different regions). The provider creates a reader account (a lightweight Snowflake account). The consumer accesses data via SQL (using a username/password). Reader accounts have no compute — the provider pays for compute.

**Data Clean Room Pattern:** Combine data sharing + secure views + row-level security to create a controlled collaboration environment. Both teams share data via shares, query it using secure views with masking policies, and audit all queries.

### Example
```sql
-- Provider side: Create a share and grant objects
CREATE OR REPLACE SHARE sales_share
    COMMENT = 'Daily sales data for partner analytics';

GRANT USAGE ON DATABASE analytics TO SHARE sales_share;
GRANT USAGE ON SCHEMA analytics.facts TO SHARE sales_share;
GRANT SELECT ON TABLE analytics.facts.sales TO SHARE sales_share;

-- Restrict access to specific rows (secure view)
CREATE OR REPLACE SECURE VIEW analytics.facts.sales_partner AS
SELECT
    sale_date,
    product_id,
    sale_amount AS revenue_share
FROM analytics.facts.sales
WHERE region = 'Partner_A'
  AND sale_date >= CURRENT_DATE - 30;

GRANT SELECT ON VIEW analytics.facts.sales_partner TO SHARE sales_share;

-- Add consumer accounts
ALTER SHARE sales_share
    SET ACCOUNTS = partner_account_locator;

-- Check share status
SHOW SHARES;

-- Consumer side: Import the share
CREATE OR REPLACE DATABASE partner_sales
    FROM SHARE sales_share;

SELECT * FROM partner_sales.analytics.facts.sales_partner;

-- Provider side: Create a reader account for non-Snowflake partners
CREATE READER ACCOUNT reader_partner_acme
    ADMIN_NAME = 'partner_admin'
    ADMIN_PASSWORD = 'TemporaryP@ss123'
    COMMENT = 'Acme Corp data access';

-- Grant share to reader account
ALTER SHARE sales_share
    SET ACCOUNTS = reader_partner_acme;
```

### Edge Cases & Caveats
- **Reader account limitations:** No compute, limited SQL, no stored procedures, no UDFs. Data can only be exported (COPY INTO) from reader accounts.
- **Share dependencies:** If you share a view, the underlying tables must also be in the share.
- **Cloning and shares:** Cloning a shared database is NOT allowed. Consumer must use it as-is or create views on top.
- **Cross-cloud sharing:** Not available — both accounts must be on the same cloud provider.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Standard Share** | Zero-copy, real-time, no consumer storage | Both parties need Snowflake |
| **Reader Account** | Non-Snowflake partners, self-service | No compute, limited functionality |
| **Data Clean Room** | Controlled collaboration, masking, auditing | Complex setup, requires trust |
| **Data Export (CSV/Parquet)** | Universal access, simple | Stale data, data movement |

### Principal-Level Considerations
- **Data sharing governance:** Document all active shares, consumers, and expiration dates. Quarterly audit: is the share still needed? Revoke stale shares.
- **Cost attribution:** Provider pays for storage; consumer pays for compute (if standard share). For reader accounts, provider pays all compute.
- **Security model:** Use secure views + masking policies for consumer shares. Never share tables directly — always through views.

### Follow-up Questions
1. **Can you update data shared via a reader account?**
   - **Answer:** No — reader accounts are read-only. If you need to update shared data, use standard data sharing or refresh by recreating the share.

2. **How do you share data across regions?**
   - **Answer:** Use database replication. First, enable replication on the source database. Then, replicate to the target region. Then, share the replicated database in the target region.

3. **What are the security implications of data sharing?**
   - **Answer:** Shares grant `SELECT` only. No DDL/DML. Use secure views + masking to limit what the consumer sees. Audit shares regularly.

4. **Can you share data across cloud providers?**
   - **Answer:** No — Snowflake's data sharing only works within the same cloud provider (AWS ↔ AWS, Azure ↔ Azure, GCP ↔ GCP). Cross-cloud requires third-party solutions.

5. **How do you handle data sharing at scale (100+ consumers)?**
   - **Answer:** Use reader accounts per consumer. Automate creation via Snowflake APIs. Implement a self-service portal where consumers request data access.

6. **What happens if you drop a shared object?**
   - **Answer:** The consumer loses access immediately. Queries against the shared object fail. Use `ALTER SHARE ... REVOKE` instead of dropping, to notify consumers.
---

## Q19. Dynamic Tables (declarative data pipelines, incremental refresh, use cases)

### Question
What are Dynamic Tables in Snowflake? How do they enable declarative data pipelines, and when should you use them vs. streams/tasks/dbt?

### Answer
Dynamic Tables (DTs) are Snowflake's declarative (automatic) transformation objects. You define a SELECT query that transforms source data; Snowflake automatically and incrementally maintains the result table.

**How They Work:** A Dynamic Table is defined by:
- A **target table** (the DT itself)
- A **SELECT query** against one or more source tables/views
- A **refresh mode**: `LAG` (incremental, near-real-time) or `SCHEDULE` (batch)

Snowflake's engine analyzes your SELECT query, determines if incremental refresh is possible, and if so, only processes changed rows from sources. This is a game changer — no need to write merge logic, change data capture, or scheduling.

**Use Cases:**
- Data mart aggregation (summarize raw data)
- Denormalization (join multiple tables)
- Staging → Curated → Mart pipeline (chain DTs)
- Real-time dashboards with 1-minute freshness

### Example
```sql
-- Incremental Dynamic Table (lag = 1 minute)
CREATE OR REPLACE DYNAMIC TABLE daily_sales_dt
    TARGET_LAG = '1 minute'
    WAREHOUSE = analytics_wh
AS
    SELECT
        DATE_TRUNC('day', o.order_date) AS order_day,
        o.product_id,
        p.category,
        COUNT(DISTINCT o.order_id) AS order_count,
        SUM(o.quantity * o.unit_price) AS revenue
    FROM orders o
    JOIN products p
        ON o.product_id = p.product_id
    GROUP BY 1, 2, 3;

-- Schedule-based Dynamic Table (batch, 1 hour)
CREATE OR REPLACE DYNAMIC TABLE customer_360_dt
    TARGET_LAG = '1 hour'
    WAREHOUSE = analytics_wh
    REFRESH_MODE = SCHEDULED
AS
    SELECT
        c.customer_id,
        c.name,
        c.segment,
        COUNT(o.order_id) AS lifetime_orders,
        SUM(o.total_amount) AS lifetime_value,
        MAX(o.order_date) AS last_order_date
    FROM customers c
    LEFT JOIN orders o
        ON c.customer_id = o.customer_id
    GROUP BY 1, 2, 3;

-- Check refresh history
SELECT
    dynamic_table_name,
    refresh_id,
    data_timestamp,
    bytes_written,
    rows_inserted,
    rows_updated,
    rows_deleted,
    status
FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY
WHERE dynamic_table_name = 'DAILY_SALES_DT'
ORDER BY data_timestamp DESC
LIMIT 10;

-- Check DT graph (dependencies)
SHOW DYNAMIC TABLES IN DATABASE analytics;
```

### Edge Cases & Caveats
- **Incompatible SELECTs:** Not all queries support incremental refresh. SUM, COUNT, MIN, MAX work incrementally. ROW_NUMBER, LATERAL FLATTEN, UNION ALL with non-incremental sources do NOT.
- **Source changes:** If a source is truncated (TRUNCATE), DT performs full refresh.
- **Storage:** Each DT stores a complete copy of the result. Storage costs are proportional to the DT size.
- **Refresh churn:** DTs at the start of a chain refresh most frequently. Plan warehouse sizing accordingly.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Dynamic Tables** | Declarative, auto-incremental, no merge code | Limited query types, storage cost |
| **Streams + Tasks** | Full control, any SQL logic | Manual merge, scheduling, maintenance |
| **dbt** | Version-controlled transformations, testing, docs | Not incremental by default, separate infrastructure |

### Principal-Level Considerations
- **DT-first for simple pipelines:** Use DTs for aggregations, joins, and denormalization. Use streams+ tasks for complex CDC or custom logic.
- **DT chain design:** Break complex transformations into 2-3 DTs (source → intermediate → mart). Avoid deep chains (refresh latency multiplies).
- **Cost monitoring:** Track DT refresh costs separately. If a DT refreshes every 1 minute on 500 GB, the cost is significant.

### Follow-up Questions
1. **What happens when a Dynamic Table's source table is dropped?**
   - **Answer:** The DT becomes invalid (error state). You must recreate it. Before dropping a source, clone the DT or create a new DT from a different source.

2. **Can you create a Dynamic Table on top of another Dynamic Table?**
   - **Answer:** Yes — this is a "chained" DT. Snowflake automatically propagates refreshes. However, each DT adds storage and refresh overhead.

3. **How do you monitor Dynamic Table freshness?**
   - **Answer:** Use `DYNAMIC_TABLE_REFRESH_HISTORY` and `DYNAMIC_TABLE_GRAPH_HISTORY` to check lag and refresh times. Set alerts for DTs where `TARGET_LAG` > defined threshold.

4. **What is the difference between TARGET_LAG and refresh frequency?**
   - **Answer:** `TARGET_LAG` is the maximum acceptable data staleness. Snowflake decides actual refresh frequency based on cost: it may refresh every 30 seconds for a 1-minute lag, or every 10 minutes for a 1-hour lag.

5. **Can you use Dynamic Tables for streaming data?**
   - **Answer:** Yes — with `TARGET_LAG` as low as 1 minute. For sub-second latency, use Snowpipe Streaming + streams/tasks instead.

6. **How do you handle schema changes in Dynamic Table sources?**
   - **Answer:** Adding columns to a source table is NOT automatically propagated to the DT. You must recreate the DT (CREATE OR REPLACE).

---

## Q20. Data Retention & Time Travel (cloning, fail-safe, storage costs)

### Question
Explain Snowflake's Data Retention model: Time Travel retention window, Fail-safe period, and how storage costs accumulate across both. How would you estimate and optimize data retention costs?

### Answer
Snowflake provides continuous data protection with two mechanisms:

**Time Travel:** Retains historical data for a configurable period (1-90 days depending on edition). Within this window, you can:
- Query data as of a point in time (`AT(TIMESTAMP => ...)`, `AT(OFFSET => ...)`)
- Clone a table as of a point in time
- Undrop dropped objects

**Fail-safe:** An additional period (7 days for Snowflake Standard/Business Critical) where Snowflake guarantees data recovery (but you cannot query it). You must contact Snowflake to restore data from fail-safe.

**Storage Costs:**
- **Logical bytes:** The uncompressed data stored (what you see in INFORMATION_SCHEMA.TABLE_STORAGE_METRICS).
- **Time Travel bytes:** The logical bytes of changed/deleted data within the Time Travel window.
- **Fail-safe bytes:** The logical bytes of changed/deleted data in the fail-safe period.

All three are billed at the same storage rate (approx. $23/TB/month for Standard Edition). Time Travel and Fail-safe bytes can be 2-10x the active data size for frequently updated tables.

### Example
```sql
-- Configure Time Travel retention (max 90 days for Business Critical)
CREATE OR REPLACE TABLE sales
    DATA_RETENTION_TIME_IN_DAYS = 30
    CLUSTER BY (sale_date)
AS
    SELECT * FROM STAGING;

-- Query historical data
SELECT * FROM sales
    BEFORE(STATEMENT => '019e3f2e-0601-7b00-0000-0000c7d20001');

SELECT * FROM sales
    AT(TIMESTAMP => '2024-06-01 00:00:00'::TIMESTAMP_NTZ);

-- Clone as of a point in time
CREATE OR REPLACE TABLE sales_restore
    CLONE sales
    AT(OFFSET => -3600);  -- 1 hour ago

-- Calculate Time Travel storage cost
SELECT
    s.table_name,
    s.database_name,
    s.schema_name,
    s.active_bytes / (1024*1024*1024) AS active_gb,
    s.time_travel_bytes / (1024*1024*1024) AS tt_gb,
    s.failsafe_bytes / (1024*1024*1024) AS fs_gb,
    ROUND(
        (s.active_bytes + s.time_travel_bytes + s.failsafe_bytes)
        * 23.0 / (1024*1024*1024*1024) * 12,
        2
    ) AS annual_cost_usd
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS s
WHERE s.database_name NOT IN ('SNOWFLAKE')
ORDER BY annual_cost_usd DESC;

-- Check Time Travel retention settings per table
SELECT
    table_catalog,
    table_schema,
    table_name,
    retention_time AS retention_days
FROM INFORMATION_SCHEMA.TABLES
WHERE retention_time < 90
ORDER BY retention_time DESC;
```

### Edge Cases & Caveats
- **Fail-safe is NOT queryable:** You cannot SELECT from fail-safe data. You must contact Snowflake and pay restoration fees.
- **Time Travel on transient tables:** Default retention is 1 day (can be 0). Fail-safe is 0 days.
- **Time Travel on temporary tables:** 0 days (no Time Travel, no fail-safe).
- **Cloning with Time Travel:** Cloning as of a point within the retention window does NOT consume additional Time Travel bytes (the clone shares underlying micro-partitions).

### Trade-offs & Alternatives

| Retention Setting | Use Case | Storage Cost Multiplier |
|------------------|----------|------------------------|
| 0-1 day | Transient staging, temp tables | 1x (active only) |
| 7 days (Standard) | General production | 1.5-2x (active + TT) |
| 30-90 days | Regulatory compliance, auditing | 3-10x |
| Clone (zero-copy) | Point-in-time backups without additional TT cost | 0 extra (clones share storage) |

### Principal-Level Considerations
- **Cost optimization:** Analyze `TABLE_STORAGE_METRICS` quarterly. Reduce retention on tables that are rebuilt frequently (e.g., staging tables).
- **Clone for backup:** Instead of extending Time Travel to 90 days, take daily clones. This gives you restore points without doubling storage.
- **Compliance vs. cost:** Regulatory requirements (e.g., GDPR "right to be forgotten") may require short retention. Balance retention period with audit needs.

### Follow-up Questions
1. **How does Time Travel affect storage costs for tables with high DML activity?**
   - **Answer:** High insert/update/delete activity increases Time Travel bytes significantly each day. For tables changed 100% daily, Time Travel bytes can be 7x active bytes (for 7-day retention).

2. **Can you extend Time Travel beyond 90 days?**
   - **Answer:** No — 90 days is the maximum for Business Critical Edition. For longer retention, use clones or export data to external storage (e.g., S3 as Parquet).

3. **What happens to a cloned table's Time Travel?**
   - **Answer:** The clone has its own independent Time Travel (starting from the clone creation time). Cloning does NOT inherit the source's Time Travel.

4. **How do you restore a table from fail-safe?**
   - **Answer:** Contact Snowflake Support. Provide the table name and desired restore time. Snowflake restores it using fail-safe data. There is a fee ($200-500 per TB restored).

5. **How do you set different retention periods for different schemas?**
   - **Answer:** Use `ALTER DATABASE ... SET DATA_RETENTION_TIME_IN_DAYS = N` to set the database default. Individual tables can override with `ALTER TABLE ... SET DATA_RETENTION_TIME_IN_DAYS = N`.

6. **What is the relationship between Time Travel and Zero-Copy Cloning?**
   - **Answer:** Both share micro-partitions. A cloned table references the same micro-partitions as the source until changes are made. Time Travel references the old micro-partitions that were replaced by DML.

---

## Q21. Pruning & Search Optimization (micro-partition pruning, clustering, search access path)

### Question
How does Snowflake's automatic pruning work? Explain the role of micro-partitions, clustering, and the Search Optimization Service in query performance.

### Answer
Snowflake optimizes query performance through three layers of pruning:

**1. Micro-partition Metadata Pruning (Automatic):**
Every table is divided into micro-partitions (~16-512 MB each). Snowflake stores metadata per partition: min/max values for each column, distinct value count, null count. When a query includes a WHERE clause, Snowflake reads this metadata and skips micro-partitions whose min/max range does not overlap the query predicate. This is automatic and requires no configuration.

**2. Clustering (Optional but Recommended for Large Tables):**
For tables > 1 TB, natural inserts may scatter similar values across micro-partitions, reducing pruning effectiveness. Clustering reorganizes data so that rows with similar values (e.g., same date) are in the same micro-partition. This maximizes min/max pruning.

**3. Search Optimization Service (SOS) (Column-Level, Paid):**
For selective predicates on non-clustered columns (e.g., point lookups like `WHERE email = 'user@example.com'`), SOS builds a persistent access path (a "search index"). This accelerates lookups by up to 10x for selective queries. Cost: ~$12-25/TB/month depending on data type.

### Example
```sql
-- Check clustering depth (lower = better organized)
SELECT
    table_name,
    clustering_depth,
    average_depth,
    average_overlap
FROM SNOWFLAKE.ACCOUNT_USAGE.CLUSTERING_HISTORY
WHERE database_name = 'ANALYTICS'
ORDER BY clustering_depth DESC;

-- Enable clustering on a large table
ALTER TABLE sales_fact
    CLUSTER BY (sale_date, region_id);

-- Recluster (automatic or manual)
ALTER TABLE sales_fact
    RECLUSTER;

-- Check pruning effectiveness
SELECT
    query_id,
    query_text,
    partitions_scanned,
    partitions_total,
    (partitions_scanned * 100 / NULLIF(partitions_total, 0)) AS scan_efficiency_pct,
    bytes_scanned,
    bytes_spilled_to_local_storage,
    bytes_spilled_to_remote_storage
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_type = 'SELECT'
  AND partitions_total > 100
  AND scan_efficiency_pct < 10
ORDER BY scan_efficiency_pct ASC
LIMIT 10;

-- Enable Search Optimization Service
ALTER TABLE customers
    ADD SEARCH OPTIMIZATION;

-- Add specific columns to SOS (for targeted acceleration)
ALTER TABLE customers
    ADD SEARCH OPTIMIZATION ON
        EQ(email),
        SUBSTRING(name, 1, 3);

-- Check SOS costs
SELECT
    query_id,
    function_name,
    credits_used,
    bytes_processed
FROM SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
ORDER BY start_time DESC
LIMIT 20;
```

### Edge Cases & Caveats
- **Clustering on high-cardinality columns:** Columns with millions of unique values (e.g., `sale_id`) do NOT benefit from clustering (each micro-partition covers a large range).
- **SOS on frequently updated tables:** Each DML on a table with SOS requires SOS index refresh, which adds cost.
- **Large VARCHAR predicates:** SOS is most effective for short strings. For long JSON paths, pruning relies on micro-partition metadata.
- **Clustering cost:** Automatic reclustering consumes credits (about 1-3 credits per TB). Monitor with `CLUSTERING_HISTORY`.

### Trade-offs & Alternatives

| Optimization | Cost | Best For | Works With |
|-------------|------|----------|------------|
| Micro-partition pruning (auto) | Free | All queries | All columns |
| Clustering | Credits | Large tables, range queries | Time-series, categorical |
| Search Optimization | Per-column cost | Selective lookups, point queries | High-cardinality columns |

### Principal-Level Considerations
- **Pruning-first approach:** Before adding clustering or SOS, analyze `partitions_scanned` vs `partitions_total`. If >50% scan efficiency, clustering helps.
- **Clustering strategy:** Cluster by the most common filter column (usually a date). Avoid clustering on multiple columns — it increases maintenance cost with diminishing returns.
- **SOS for specific use cases:** Enable SOS only for columns used in selective WHERE clauses in production queries. Do NOT enable it on all columns.

### Follow-up Questions
1. **How do you measure clustering effectiveness?**
   - **Answer:** Use `CLUSTERING_DEPTH` (lower = better depth) and `AVERAGE_OVERLAP` (lower = less overlap). Values < 5 are good; > 20 indicates poor clustering.

2. **What types of queries benefit most from Search Optimization Service?**
   - **Answer:** Queries with highly selective predicates (returning < 1% of rows) such as `WHERE email = '...'`, `WHERE phone = '...'`, or `WHERE order_id = '...'`.

3. **Can you cluster a table by an expression?**
   - **Answer:** Yes — use `CLUSTER BY (DATE_TRUNC('day', timestamp_column), region_id)`. The expression is applied during reclustering.

4. **What is the relationship between clustering and materialized views?**
   - **Answer:** Materialized views can also be clustered. A mat view's clustering key is independent of the base table's. Cluster the mat view by the columns most commonly filtered in queries against it.

5. **Does Search Optimization Service work on views?**
   - **Answer:** No — SOS can only be created on tables. For views, create SOS on the underlying tables.

6. **How do you estimate SOS cost before enabling it?**
   - **Answer:** Use `SYSTEM$ESTIMATE_SEARCH_OPTIMIZATION_COSTS('table_name')` to get a monthly credit estimate. Start with a trial on a non-production table.

---

## Q22. Network Policies & Security (IP allow/block, private endpoints, VPC peering)

### Question
How do you implement network-level security in Snowflake? Explain Network Policies (IP-based), Private Connectivity (AWS PrivateLink / Azure Private Link / GCP Private Service Connect), and VPC Peering.

### Answer
Snowflake provides three layers of network security:

**1. Network Policies (IP Allow/Block):**
Account-level or user-level rules that restrict which IP addresses can connect. You define a list of allowed IP ranges and optionally block specific ranges. Works with standard SQL JDBC/ODBC connections.

**2. Private Connectivity:**
- **AWS PrivateLink:** Creates a private endpoint in your VPC. Traffic from your VPC to Snowflake stays on the AWS backbone — never traverses the public internet.
- **Azure Private Link:** Same concept for Azure.
- **GCP Private Service Connect:** Same for GCP.

**3. External Network Access (Egress Control):**
Snowflake supports network rules that control outbound connections from stored procedures, UDFs, and external functions.

### Example
```sql
-- Create a network policy
CREATE OR REPLACE NETWORK POLICY corporate_network
    ALLOWED_IP_LIST = (
        '10.0.0.0/8',          -- Corporate VPN
        '192.168.0.0/16',       -- Office network
        '203.0.113.0/24'        -- Data center
    )
    BLOCKED_IP_LIST = (
        '10.0.100.0/24'         -- Restricted zone
    )
    COMMENT = 'Corporate network access policy';

-- Apply at account level
ALTER ACCOUNT
    SET NETWORK_POLICY = corporate_network;

-- Apply at user level (overrides account for specific users)
ALTER USER data_engineer
    SET NETWORK_POLICY = corporate_network;

-- Check current network policy
SHOW NETWORK POLICIES;
SELECT SYSTEM$GET_ALLOWED_IP_LIST('corporate_network');
SELECT SYSTEM$GET_BLOCKED_IP_LIST('corporate_network');

-- Remove network policy for maintenance access
ALTER ACCOUNT
    UNSET NETWORK_POLICY;

-- Set up PrivateLink (AWS)
-- 1. Get the Snowflake PrivateLink endpoint
SELECT SYSTEM$GET_PRIVATELINK_CONFIG();

-- 2. Create VPC endpoint in AWS Console using the Snowflake endpoint service name
-- 3. Enable PrivateLink in Snowflake
ALTER ACCOUNT SET
    ENABLE_PRIVATE_ENDPOINTS = TRUE;

-- 4. Verify connectivity
SELECT SYSTEM$ALLOWED_NETWORK_POLICY_IPS();
```

### Edge Cases & Caveats
- **Network policy latency:** Changes to network policy take effect immediately for new connections. Existing connections remain until re-established.
- **PrivateLink and data sharing:** PrivateLink access to shared databases is transparent (the consumer's network policy applies).
- **IP list limit:** Maximum 256 CIDR ranges per network policy.
- **Blocked list priority:** BLOCKED_IP_LIST takes precedence over ALLOWED_IP_LIST.

### Trade-offs & Alternatives

| Security Layer | Type | Setup Complexity | Cost |
|---------------|------|-----------------|------|
| Network Policy | IP-based | Low (SQL commands) | Free |
| PrivateLink | Network-level | Medium (AWS/Azure/GCP config) | $0.01-0.05/hr per endpoint |
| VPC Peering | Network-level | High (requires Snowflake enterprise) | $0.01-0.02/hr + data transfer |

### Principal-Level Considerations
- **Defense in depth:** Combine network policies (IP allowlist) with PrivateLink (private network) and MFA (user-level). Do not rely on a single layer.
- **PrivateLink adoption:** For any production Snowflake deployment, use PrivateLink. It eliminates internet exposure and reduces latency by 10-30%.
- **Network policy auditing:** Monitor `LOGIN_HISTORY` for failed login attempts from blocked IPs. Investigate repeated attempts.

### Follow-up Questions
1. **How do you troubleshoot network policy issues?**
   - **Answer:** Check `SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY` for `IS_BLOCKED_BY_NETWORK_POLICY = TRUE`. Use `SYSTEM$GET_ALLOWED_IP_LIST()` to verify the current policy.

2. **Can you have multiple network policies for different groups?**
   - **Answer:** Yes — assign different policies to different users. One user can have only one policy. User-level policies override account-level policies.

3. **How does PrivateLink affect data transfer costs?**
   - **Answer:** AWS/Azure/GCP charge for VPC endpoint hours and data transfer. Usually $0.01-0.03/hr per endpoint. Snowflake does not charge extra for PrivateLink.

4. **What is the difference between PrivateLink and VPC Peering?**
   - **Answer:** PrivateLink connects your VPC to a Snowflake-managed VPC service without peering. VPC Peering is a direct connection between two VPCs (requires Snowflake's approval and is rarely used).

5. **Can you use network policies with Snowsight?**
   - **Answer:** Yes — Snowsight respects account-level and user-level network policies. Users connecting via browser must have their IP in the allowlist.

6. **How do you handle contractors who need temporary access?**
   - **Answer:** Create a temporary network policy for the contractor's IP range and assign it to their user. Set a task to remove the policy after the engagement period.
---

## Q23. MFA & RBAC (role-based access control, authentication, SSO, SCIM)

### Question
How does Snowflake handle authentication and authorization? Explain the RBAC model, MFA enforcement, SSO integration, and SCIM provisioning at enterprise scale.

### Answer
Snowflake uses a **Role-Based Access Control (RBAC)** model where permissions are granted to roles, and roles are granted to users. This is more scalable than user-level permissions.

**Authentication Options:**
1. **Username/Password:** Basic. Must enforce MFA.
2. **MFA (Multi-Factor Authentication):** SMS, DUO, or TOTP. Can be enforced at account level or per user via network policies.
3. **SSO (SAML/OAuth/OpenID Connect):** Integrate with Okta, Azure AD, ADFS. Users authenticate via their corporate IdP.
4. **Key-Pair Authentication:** For service accounts and automation.

**RBAC Model:**
- **Users:** Human or service accounts
- **Roles:** Hierarchical (parent → child via GRANT ROLE TO ROLE)
- **Privileges:** SELECT, INSERT, CREATE, USAGE, OWNERSHIP
- **Securable Objects:** Databases, schemas, tables, views, warehouses

**SCIM (System for Cross-domain Identity Management):**
Automates user and role provisioning. When a user joins the company via your IdP, SCIM automatically creates the Snowflake user and assigns roles. When they leave, SCIM disables the user.

### Example
```sql
-- Create custom roles
CREATE OR REPLACE ROLE analyst_role;
CREATE OR REPLACE ROLE data_engineer_role;
CREATE OR REPLACE ROLE admin_role;

-- Grant privileges to roles
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE analyst_role;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.facts TO ROLE analyst_role;
GRANT CREATE TABLE ON SCHEMA analytics.facts TO ROLE data_engineer_role;

-- Role hierarchy (data_engineer inherits analyst privileges)
GRANT ROLE analyst_role TO ROLE data_engineer_role;

-- Assign role to user
CREATE USER analyst_john
    PASSWORD = 'TemporaryP@ss123'
    DEFAULT_ROLE = analyst_role
    DEFAULT_WAREHOUSE = analytics_wh
    MUST_CHANGE_PASSWORD = TRUE;

GRANT ROLE analyst_role TO USER analyst_john;

-- Enforce MFA (create a network policy that requires MFA)
CREATE OR REPLACE NETWORK POLICY secure_access
    ALLOWED_IP_LIST = ('0.0.0.0/0')
    BLOCKED_IP_LIST = ()
    COMMENT = 'Enforces MFA for all users';

ALTER ACCOUNT SET
    MFA_ENFORCEMENT = TRUE;

-- Enable SSO via SAML (Okta example)
CREATE OR REPLACE SAML_IDENTITY_PROVIDER okta_sso
    TYPE = OKTA
    ISSUER = 'http://www.okta.com/abc123'
    SSO_URL = 'https://company.okta.com/app/snowflake/abc123/sso/saml'
    CERTIFICATE = 'MIID...';

ALTER ACCOUNT SET
    SAML_IDENTITY_PROVIDER = okta_sso;

-- Check role assignments
SELECT
    grantee_name,
    role_name,
    granted_by_name
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE granted_to = 'ROLE'
ORDER BY grantee_name;

-- Check privilege assignments
SELECT
    grantee_name,
    object_name,
    object_type,
    privilege_type
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE granted_by_name IS NOT NULL;
```

### Edge Cases & Caveats
- **Role hierarchy depth:** Maximum 100 levels deep. Keep it flat (parent → child, not deeper).
- **MFA recovery:** If a user loses their MFA device, the account admin can reset it with `ALTER USER ... SET MFA_RESET = TRUE`. After login, user re-enrolls MFA.
- **SCIM sync frequency:** Usually 5-15 minutes. For urgent revocations, manually disable the user in Snowflake.
- **Object ownership:** Only one role can OWN a securable object. Use MANAGED ACCESS schemas to separate ownership from data access decisions.

### Trade-offs & Alternatives

| Auth Method | Pros | Cons |
|------------|------|------|
| Username/Password + MFA | Simple, low overhead | MFA fatigue, no SSO |
| SAML SSO + MFA | Centralized, enterprise-ready | Single IdP dependency |
| OAuth | Token-based, ideal for apps | Token management complexity |
| Key-Pair | Service accounts, automation | Key rotation overhead |

### Principal-Level Considerations
- **Least privilege principle:** Start with ANALYST (read-only) and DATA_ENGINEER (read/create). Only grant ADMIN to Tier-1 engineers.
- **Separation of duties:** Different roles for data producers (staging), transformers (curated), and consumers (analytics). Use MANAGED ACCESS schemas to let schema owners control access.
- **Automate provisioning:** Use SCIM for employee lifecycle. For service accounts, use Terraform or Snowflake's SQL API.

### Follow-up Questions
1. **What is a MANAGED ACCESS schema and when should you use it?**
   - **Answer:** A schema where only the schema owner (or role with OWNERSHIP) can grant access to objects. Useful in regulated environments where a central data governance team controls permissions.

2. **How do you rotate service account keys?**
   - **Answer:** Create a second key-pair, add it to the user (`ALTER USER ... SET RSA_PUBLIC_KEY_2 = '...'`), rotate to it, then remove the old key. This ensures no downtime.

3. **What happens when a role is dropped?**
   - **Answer:** All privileges granted through that role are revoked. Users lose access immediately. Future GRANT statements referencing the dropped role fail.

4. **How do you audit role usage?**
   - **Answer:** Query `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` with `session_role_name`. Check which roles run which queries. Revoke unused roles.

5. **Can you enforce MFA for specific roles only?**
   - **Answer:** Not directly via MFA enforcement. Use network policies with conditional access (e.g., block non-MFA IPs for admin_role users) or SSO conditional access policies.

6. **How does Snowflake handle federated authentication with OAuth?**
   - **Answer:** Snowflake acts as the resource server. Your app gets an OAuth token from the IdP, then uses it to connect to Snowflake. Tokens have configurable expiry (default 10 hours).

---

## Q24. Multi-cluster Warehouses (concurrency, elasticity, scaling policy)

### Question
Explain Multi-cluster Warehouses in Snowflake. How do they handle concurrency spikes, what scaling policies are available, and how do you optimize cost vs. performance?

### Answer
A Multi-cluster Warehouse (MCW) is a warehouse that can automatically add or remove clusters based on workload demand. Each cluster is an identical compute resource with its own local cache. MCWs are essential for handling concurrent query surges without queueing or manual intervention.

**Key Parameters:**
- `MIN_CLUSTER_COUNT`: Always-on clusters (minimum 1).
- `MAX_CLUSTER_COUNT`: Maximum clusters allowed (up to 10).
- `MAX_CONCURRENCY_LEVEL`: Maximum queries per cluster (default 8, max 20+).
- `SCALING_POLICY`: `STANDARD` (cost-optimized, slow scaling) or `ECONOMY` (cost-minimized, slower scaling).

**How Scaling Works:**
1. A query arrives at the warehouse.
2. If all clusters are at `MAX_CONCURRENCY_LEVEL`, the query enters the queue.
3. After the cooldown period (~5 min for STANDARD, ~15 min for ECONOMY), a new cluster spins up.
4. Adding a cluster takes 1-2 minutes (warm-up).
5. Clusters are removed when idle for the cooldown period.
6. Caches are NOT shared across clusters (each cluster has its own local disk cache).

### Example
```sql
-- Create a multi-cluster warehouse
CREATE OR REPLACE WAREHOUSE analytics_wh
    WAREHOUSE_SIZE = LARGE
    MIN_CLUSTER_COUNT = 2
    MAX_CLUSTER_COUNT = 5
    MAX_CONCURRENCY_LEVEL = 8
    SCALING_POLICY = 'STANDARD'
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE
    COMMENT = 'Multi-cluster warehouse for BI dashboards';

-- Change scaling policy for cost optimization
ALTER WAREHOUSE analytics_wh
    SET SCALING_POLICY = 'ECONOMY';

-- Monitor cluster usage
SELECT
    start_time,
    end_time,
    warehouse_name,
    warehouse_size,
    cluster_number,
    credits_used,
    credits_used_compute,
    credits_used_cloud_services
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE warehouse_name = 'ANALYTICS_WH'
ORDER BY start_time DESC
LIMIT 20;

-- Check warehouse load history
SELECT
    warehouse_name,
    average_running,
    average_queued,
    average_blocked,
    session_count,
    query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
ORDER BY start_time DESC
LIMIT 10;

-- Check if queries are queuing (need more clusters)
SELECT
    query_id,
    query_text,
    warehouse_name,
    warehouse_size,
    warehouse_type,
    cluster_number,
    queued_provisioning_time,
    queued_overload_time,
    transaction_blocked_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE (queued_provisioning_time > 0 OR queued_overload_time > 0)
  AND start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY queued_overload_time DESC;
```

### Edge Cases & Caveats
- **Cache fragmentation:** Each cluster has its own cache. If 5 clusters are active, queries may hit 20% cache hit rate instead of 80%. Consider Query Acceleration Service for such cases.
- **Scaling lag:** Adding a cluster takes 1-2 minutes. For sudden spikes, queries queue during that window.
- **Min_cluster_count cost:** Setting `MIN_CLUSTER_COUNT = 2` ensures 2 clusters are always running (24/7 cost). For non-production, set min=1 and rely on auto-scaling.
- **Scaling policy impact:** `ECONOMY` adds clusters more slowly (15 min cooldown vs 5 min). Use for predictable workloads.

### Trade-offs & Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| Single warehouse, large size | Simple, single cache | No elasticity, queuing under load |
| Multi-cluster (STANDARD) | Fast scaling, good for unpredictable load | Cache fragmentation, cost |
| Multi-cluster (ECONOMY) | Cost-efficient scaling | Slow scaling, cache fragmentation |
| Query Acceleration Service | No warehouse change, helps specific queries | Extra cost per query |

### Principal-Level Considerations
- **Right-size min_cluster_count:** For production BI, start with 2 clusters (redundancy). For development, use 1.
- **Monitor queuing:** If `queued_overload_time` > 0 for more than 5% of queries, increase `MAX_CLUSTER_COUNT` or `MAX_CONCURRENCY_LEVEL`.
- **Cost vs. performance:** Scaling from 2 to 10 clusters increases cost 5x but performance increase is sub-linear (cache fragmentation reduces benefit).

### Follow-up Questions
1. **What is the difference between STANDARD and ECONOMY scaling?**
   - **Answer:** STANDARD adds clusters after ~5 min of queuing; ECONOMY after ~15 min. STANDARD removes clusters after 5 min idle; ECONOMY after 15 min idle.

2. **Can you set MIN_CLUSTER_COUNT to 0?**
   - **Answer:** No — MIN_CLUSTER_COUNT must be at least 1. Use AUTO_SUSPEND (suspend after idle period) to save cost.

3. **How does multi-clustering affect ETL workloads?**
   - **Answer:** ETL typically runs sequentially (one COPY INTO, one MERGE). Multi-clustering is unnecessary. Use a single warehouse with appropriate size.

4. **What happens to running queries when a cluster is removed?**
   - **Answer:** Running queries finish on their cluster. New queries are routed to remaining clusters or wait for a new cluster to spin up.

5. **How do you size MAX_CONCURRENCY_LEVEL?**
   - **Answer:** Default 8 works for most workloads. For heavy queries (scanning TB+), lower it to 4. For light queries (aggregated dashboards), increase to 16-20.

6. **Can you limit multi-clustering to specific users?**
   - **Answer:** No — multi-clustering is per-warehouse. Create separate warehouses for different user groups (e.g., BI_WH for dashboards, ETL_WH for batch processing).

---

## Q25. Query Acceleration Service (QAS) (elastic, partitioned scans, cost)

### Question
What is Query Acceleration Service? When does it make sense, how does it work under the hood, and how do you monitor its cost/benefit?

### Answer
Query Acceleration Service (QAS) is a serverless, elastic compute layer that accelerates large, scan-heavy queries (aggregations, JOINs, GROUP BYs over TB+ of data). It complements warehouse compute by offloading parts of query execution to a separate, on-demand compute pool.

**How It Works:**
1. Snowflake's optimizer identifies sub-operations that benefit from parallelization.
2. These operations are offloaded to QAS (serverless compute) while the warehouse handles the rest.
3. QAS processes data in parallel across many nodes.
4. Results are merged back into the warehouse.

**When to Use QAS:**
- Queries scanning > 100 GB regularly
- Queries with high spill-to-remote-storage (indicating warehouse is undersized)
- Multi-cluster warehouse with cache fragmentation (QAS processes data independently of cluster cache)

**Cost Model:**
- QAS is billed per query (credits are consumed only when QAS is used)
- Cost varies: typically $0.50-2.00 per TB scanned by QAS
- Can be enabled at warehouse level or as a parameter

### Example
```sql
-- Enable QAS on a warehouse
ALTER WAREHOUSE analytics_wh
    SET ENABLE_QUERY_ACCELERATION = TRUE;

-- Enable QAS with a max cost limit
ALTER WAREHOUSE analytics_wh
    SET ENABLE_QUERY_ACCELERATION = TRUE
    QUERY_ACCELERATION_MAX_SCALE_FACTOR = 10;

-- Check QAS effectiveness
SELECT
    query_id,
    query_text,
    warehouse_size,
    partitions_scanned,
    bytes_scanned,
    bytes_spilled_to_remote_storage,
    EXECUTION_TIME,
    query_acceleration_bytes_scanned,
    query_acceleration_elapsed_time,
    query_acceleration_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_acceleration_credits > 0
  AND start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY query_acceleration_credits DESC;

-- Simulate QAS benefit (use EXPLAIN with QAS enabled)
EXPLAIN USING QUERY_ACCELERATION
SELECT
    DATE_TRUNC('month', sale_date) AS sale_month,
    region_id,
    SUM(sale_amount) AS revenue
FROM sales_fact
WHERE sale_date >= '2024-01-01'
GROUP BY 1, 2;

-- Disable QAS for specific queries
ALTER SESSION SET ENABLE_QUERY_ACCELERATION = FALSE;
```

### Edge Cases & Caveats
- **Not all queries qualify:** QAS helps only large scans and aggregations. Point lookups do not benefit.
- **Small queries may be slower:** The overhead of offloading to QAS can exceed benefit for queries scanning < 10 GB. Enable per-warehouse, not globally.
- **Cost can surprise:** In rare cases, QAS cost exceeds warehouse cost (especially for suboptimal queries). Set `QUERY_ACCELERATION_MAX_SCALE_FACTOR` to limit spend.

### Trade-offs & Alternatives

| Scenario | Without QAS | With QAS |
|----------|-------------|----------|
| TB-sized GROUP BY | Slow or spill | 2-5x faster |
| Multi-cluster, cache misses | 60% cache hit, slow scans | Faster scans independently of cache |
| Spill-to-remote-storage | High wait, high cost for SSD/Network spill | Reduced spill |
| Small queries (< 10 GB) | Fast | Slower (overhead) |

### Principal-Level Considerations
- **QAS + multi-cluster:** Use QAS when multi-clustering is not enough (e.g., cache fragmentation hurts). QAS bypasses cluster cache, providing fresh scan performance.
- **Max scale factor:** Start with 10. Monitor `query_acceleration_credits`. If > 5% of total warehouse spend, reduce scale factor.
- **QAS for ad-hoc queries:** Enable QAS for analyst warehouses but disable for ETL warehouses (ETL queries are predictable; optimize warehouse size instead).

### Follow-up Questions
1. **Can QAS be used with cached results?**
   - **Answer:** No — if the result is from cache, QAS is not invoked (the query is instant). QAS only applies when the query must scan data.

2. **How do you estimate QAS cost before enabling?**
   - **Answer:** Use `SYSTEM$ESTIMATE_QUERY_ACCELERATION('SELECT ...')` to get an estimated cost. Compare with the query's current cost.

3. **Does QAS work with Materialized Views?**
   - **Answer:** Yes — if a query against a materialized view still scans many micro-partitions, QAS can accelerate the scan.

4. **What is the QUERY_ACCELERATION_MAX_SCALE_FACTOR?**
   - **Answer:** The maximum ratio of QAS cost to warehouse cost. Max scale factor = 10 means QAS can use up to 10x the warehouse cost for a single query.

5. **How does QAS affect concurrency?**
   - **Answer:** QAS runs in a separate, elastic compute pool. It does NOT consume warehouse slots. It improves overall concurrency by offloading heavy scans.

6. **Can you see which parts of a query were accelerated?**
   - **Answer:** Use `EXPLAIN USING QUERY_ACCELERATION`. The query plan shows which operations are marked as "accelerated." Also check `query_acceleration_bytes_scanned` in QUERY_HISTORY.

---

## Q26. Data Governance & Classification (tag-based, classification, access history, lineage)

### Question
How does Snowflake support data governance? Explain Object Tagging, Automatic Classification, Access History, and Object Lineage.

### Answer
Snowflake provides a comprehensive data governance framework built into the platform:

**1. Object Tagging:**
Attach key-value metadata to Snowflake objects (databases, schemas, tables, columns). Tags are used for governance, cost allocation, and access control (via tag-based masking policies).

**Example tags:** `sensitivity = 'PII'`, `data_class = 'PHI'`, `retention = '7_years'`, `cost_center = 'finance'`.

**2. Automatic Classification:**
Snowflake automatically scans column data and suggests classification tags (PII, PHI, financial data, etc.) based on content patterns. Results are stored in the `TAG_REFERENCES` view.

**3. Access History:**
Records every query that accesses objects. Shows which users accessed which columns/data. Essential for compliance (GDPR, SOX, HIPAA) and auditing.

**4. Object Lineage:**
Shows how data flows between tables and views. Trace data from source → staging → curated → report. Critical for impact analysis (which reports break if a source column changes).

### Example
```sql
-- Create and apply governance tags
CREATE OR REPLACE TAG sensitivity
    ALLOWED_VALUES = 'PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'PII', 'PHI';

CREATE OR REPLACE TAG data_class
    ALLOWED_VALUES = 'HR', 'FINANCE', 'CUSTOMER', 'PRODUCT', 'SALES';

CREATE OR REPLACE TAG retention_policy
    ALLOWED_VALUES = '1_YEAR', '3_YEARS', '7_YEARS', 'INDEFINITE';

-- Apply tags to columns
ALTER TABLE customers
    MODIFY COLUMN email
        SET TAG sensitivity = 'PII', data_class = 'CUSTOMER';
ALTER TABLE customers
    MODIFY COLUMN ssn
        SET TAG sensitivity = 'PII', data_class = 'CUSTOMER', retention_policy = '7_YEARS';

-- View all tags and assignments
SELECT
    tag_name,
    tag_value,
    object_name,
    column_name,
    domain
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE tag_name IN ('SENSITIVITY', 'DATA_CLASS');

-- Automatic classification
CALL SYSTEM$CLASSIFY('analytics.public.customers');
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Access History (who accessed which columns)
SELECT
    user_name,
    query_id,
    query_text,
    objects_accessed,
    records_accessed,
    first_access_time
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE user_name = 'ANALYST_JOHN'
  AND first_access_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY first_access_time DESC;

-- Object Lineage (data flow tracking)
SELECT
    source_object_name,
    target_object_name,
    target_column_name,
    source_column_name,
    created_time,
    last_altered_time
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE source_object_name LIKE '%analytics.facts.sales%';
```

### Edge Cases & Caveats
- **Tag propagation:** Tags on a table are NOT automatically applied to views or downstream objects. You must manually tag derived objects.
- **Classification accuracy:** Automatic classification is heuristic-based. It may miss some PII or over-tag (false positives). Always review and manually adjust.
- **Access History retention:** Data is retained for 365 days in ACCOUNT_USAGE.
- **Lineage for complex transformations:** Object dependencies may not capture all intermediate steps in stored procedures. Supplement with manual lineage documentation.

### Trade-offs & Alternatives

| Governance Feature | Use Case | Limitations |
|-------------------|----------|-------------|
| Object Tagging | Metadata, cost allocation, access control | Manual application |
| Auto Classification | PII/PHI discovery | Heuristic, needs review |
| Access History | Compliance, audit | 365-day retention |
| Object Lineage | Impact analysis | Does not capture procedure internals |

### Principal-Level Considerations
- **Tag-based masking:** Combine tags with masking policies: `CREATE MASKING POLICY pii_mask AS (val STRING) RETURNS STRING -> CASE WHEN CURRENT_ROLE() IN ('ADMIN') THEN val ELSE '***' END`. Apply the mask to all columns tagged `sensitivity = 'PII'`.
- **Automated governance pipeline:** Use Snowflake tasks + stored procedures to scan for untagged columns weekly and apply missing tags.
- **Access History monitoring:** Set alerts for users accessing columns tagged `PII` without a valid business reason (audit trail).

### Follow-up Questions
1. **How do you enforce tag-based masking policies?**
   - **Answer:** Create a masking policy and use `ALTER TAG sensitivity SET MASKING POLICY pii_mask`. The mask applies to all columns with that tag (new columns tagged 'PII' get the mask automatically).

2. **What is the difference between Object Tagging and Dynamic Data Masking?**
   - **Answer:** Tags are metadata (key-value pairs). Dynamic Data Masking is a policy that transforms data at query time. Tags can be used to trigger masking policies.

3. **How do you track changes to tags?**
   - **Answer:** Query `SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES` with `last_altered_time` to see when tags were changed. For audit trails, query `QUERY_HISTORY` for ALTER TABLE ... SET TAG statements.

4. **Can you export Access History to external systems?**
   - **Answer:** Yes — query ACCESS_HISTORY and export to external storage (S3). Snowflake also supports streaming Access History to external monitoring tools via Event Tables (preview).

5. **What are the costs of storing Access History data?**
   - **Answer:** Access History is stored in the SNOWFLAKE database (free, no storage cost). However, querying it incurs warehouse credits.

6. **How do you use Object Dependencies (lineage) for impact analysis?**
   - **Answer:** Before changing a source table, query OBJECT_DEPENDENCIES to find all downstream objects. Then update or notify consumers before the change.
---

## Q27. Stored Procedures & UDFs (JavaScript, Python, SQL, Snowpark, best practices)

### Question
Compare the different languages for Stored Procedures and UDFs in Snowflake. When would you choose JavaScript vs Python vs SQL? What are best practices for writing robust procedures?

### Answer
Snowflake supports stored procedures and UDFs in multiple languages, each with different strengths:

**SQL Stored Procedures:** Best for simple DML orchestration (INSERT, UPDATE, DELETE, CREATE). No external dependencies. Execute multi-statement SQL transactions. Fastest to write and deploy.

**JavaScript Stored Procedures:** Traditional Snowflake choice. Good for procedural logic (loops, conditionals) when you need to manipulate SQL results within a procedure. Uses Spidermonkey engine (ES5.1 — no modern JS features).

**Python Stored Procedures (Snowpark):** Modern choice for complex logic. Access to Anaconda packages (pandas, numpy, scikit-learn). Best for ML, advanced transformations, and orchestration.

**Java/Scala Stored Procedures:** For JVM-native workloads. Less common.

**UDFs (all languages):** Scalar functions (one row in, one out). Use for column-level transformations. Python UDFs support Anaconda packages.

### Example
```sql
-- SQL stored procedure (simple DML orchestration)
CREATE OR REPLACE PROCEDURE refresh_daily_aggregation()
    RETURNS STRING
    LANGUAGE SQL
AS
$$
BEGIN
    CREATE OR REPLACE TEMPORARY TABLE daily_agg_temp AS
    SELECT
        DATE_TRUNC('day', order_date) AS order_day,
        product_id,
        SUM(quantity) AS total_qty,
        SUM(amount) AS total_amount
    FROM orders
    WHERE order_date >= CURRENT_DATE - 3
    GROUP BY 1, 2;

    MERGE INTO daily_sales_summary t
    USING daily_agg_temp s
    ON t.order_day = s.order_day AND t.product_id = s.product_id
    WHEN MATCHED THEN UPDATE SET
        total_qty = s.total_qty,
        total_amount = s.total_amount
    WHEN NOT MATCHED THEN INSERT
        (order_day, product_id, total_qty, total_amount)
    VALUES (s.order_day, s.product_id, s.total_qty, s.total_amount);

    RETURN 'Aggregation complete for: ' || CURRENT_DATE;
END;
$$;

-- JavaScript stored procedure (looping through results)
CREATE OR REPLACE PROCEDURE archive_old_records(cutoff_date DATE)
    RETURNS STRING
    LANGUAGE JAVASCRIPT
AS
$$
    var stmt = snowflake.createStatement({
        sqlText: `SELECT table_name FROM INFORMATION_SCHEMA.TABLES
                   WHERE table_schema = 'CURATED' AND table_type = 'BASE TABLE'`
    });
    var result = stmt.execute();
    var archivedCount = 0;

    while (result.next()) {
        var tableName = result.getColumnValue(1);
        var archiveStmt = snowflake.createStatement({
            sqlText: `INSERT INTO archive.${tableName}
                      SELECT * FROM curated.${tableName}
                      WHERE created_date < :1`,
            binds: [cutoff_date]
        });
        archiveStmt.execute();
        archivedCount++;
    }
    return 'Archived ' + archivedCount + ' tables';
$$;

-- Python stored procedure (Snowpark, with pandas)
CREATE OR REPLACE PROCEDURE calculate_regression_coefficients()
    RETURNS STRING
    LANGUAGE PYTHON
    RUNTIME_VERSION = '3.10'
    PACKAGES = ('snowflake-snowpark-python', 'pandas', 'scikit-learn')
    HANDLER = 'run'
AS
$$
import pandas as pd
from sklearn.linear_model import LinearRegression
from snowflake.snowpark import Session

def run(session: Session):
    df = session.sql("""
        SELECT feature_1, feature_2, target
        FROM features
    """).to_pandas()

    X = df[['feature_1', 'feature_2']]
    y = df['target']
    model = LinearRegression().fit(X, y)

    session.sql(f"""
        INSERT INTO model_coefficients
        VALUES (CURRENT_TIMESTAMP,
                {model.coef_[0]},
                {model.coef_[1]},
                {model.intercept_})
    """).collect()

    return f"Model trained. Coef: {model.coef_}, Intercept: {model.intercept_}"
$$;
```

### Edge Cases & Caveats
- **JS stored procedure limitations:** No ES6+ features (no let, arrow functions, promises). Use `var` for variables. No async/await.
- **Python UDF memory:** Limited to ~40 MB per UDF execution. Use vectorized UDFs for larger data.
- **SQL procedure overhead:** Simple SQL in a procedure is slower than running raw SQL (due to compilation overhead).
- **Error handling:** Always wrap critical logic in `BEGIN ... EXCEPTION ... END` for SQL procedures, or try/catch for JS/Python.

### Principal-Level Considerations
- **Language choice matrix:**
  - Simple DML → SQL procedure
  - Loop through tables → JavaScript or Python
  - Complex transformation → Python (Snowpark)
  - Column-level transform → SQL UDF (fastest)
  - ML inference → Python UDF (with Anaconda packages)

- **Testing:** Unit test UDFs by selecting sample data. Test procedures by calling with mock parameters.
- **Version control:** Store procedure/UDF code in git (not just Snowflake). Automate deployment via CI/CD.

### Follow-up Questions
1. **How do you pass parameters to stored procedures?**
   - **Answer:** Define parameters in the CREATE PROCEDURE signature (e.g., `CREATE PROCEDURE my_proc(param1 VARCHAR, param2 INT)`). Call with `CALL my_proc('value', 42)`.

2. **What is the maximum size of a UDF?**
   - **Answer:** The function body is limited to 2 MB. For larger code, use stored procedures instead.

3. **Can a stored procedure call another stored procedure?**
   - **Answer:** Yes — SQL and Snowpark procedures can call other procedures. Use `CALL proc_name()` within the procedure body.

4. **How do you handle errors in stored procedures?**
   - **Answer:** SQL: `BEGIN ... EXCEPTION WHEN OTHER THEN ... END`. JavaScript: `try { ... } catch(err) { ... }`. Python: `try: ... except: ...`.

5. **Can you schedule a stored procedure?**
   - **Answer:** Yes — create a task with `CALL proc_name()`. Schedule it with CRON or interval.

6. **How do you debug a stored procedure?**
   - **Answer:** Add `RETURN` statements at checkpoints. Read output by calling the procedure. For Python procedures, use `print()` and read the logs from the query history.

---

## Q28. Database Replication & Failover (cross-region, cross-cloud, business continuity)

### Question
Explain Snowflake's Database Replication and Failover/Failback capabilities. How do you design a multi-region disaster recovery strategy?

### Answer
Snowflake offers two disaster recovery (DR) features:

**1. Database Replication:**
Asynchronously copy a database from one Snowflake account (primary, source) to another account (secondary, target) across regions or clouds. Replication is automated and near-real-time (typically < 60 second lag for continuous replication).

**2. Failover/Failback:**
- **Failover:** Promote the secondary database to primary (switch roles).
- **Failback:** Replicate the promoted primary back to the original region.

**Supported Scenarios:**
- Same cloud, different region (e.g., AWS us-east-1 → us-west-2)
- Different clouds (e.g., AWS us-east-1 → Azure eastus)
- Same region (e.g., for account-level isolation)

**Requirements:**
- Business Critical Edition (or higher)
- Both accounts must be in the same Snowflake organization
- Replication must be enabled on the primary database

### Example
```sql
-- PRIMARY REGION (us-east-1): Enable replication
ALTER DATABASE analytics_db
    ENABLE REPLICATION TO ACCOUNTS
        org.myorg.analytics_secondary;

-- PRIMARY: Check replication status
SELECT
    database_name,
    account_locator,
    region_group,
    region,
    scheduled_time,
    actual_replication_time,
    bytes_transferred,
    status
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_REPLICATION_USAGE_HISTORY
WHERE database_name = 'ANALYTICS_DB'
ORDER BY scheduled_time DESC;

-- SECONDARY REGION (eu-west-1): Create the secondary database
CREATE DATABASE analytics_db_secondary
    AS REPLICA OF org.myorg.analytics_db;

-- SECONDARY: Verify replication lag
SELECT
    database_name,
    replication_group_name,
    primary_account_locator,
    primary_region,
    scheduled_time,
    actual_replication_time,
    bytes_transferred
FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS
WHERE database_name = 'ANALYTICS_DB_SECONDARY';

-- FAILOVER: Promote secondary to primary
ALTER DATABASE analytics_db_secondary
    PRIMARY;

-- Now secondary becomes primary, and the old primary becomes secondary
-- Replication direction changes automatically

-- FAILBACK: After the original region is restored
ALTER DATABASE analytics_db_secondary
    ENABLE REPLICATION TO ACCOUNTS
        org.myorg.analytics_primary;

-- RECREATE primary in the original region
CREATE DATABASE analytics_db_primary
    AS REPLICA OF org.myorg.analytics_db_secondary;

-- Promote back
ALTER DATABASE analytics_db_primary
    PRIMARY;
```

### Edge Cases & Caveats
- **Replication lag:** Typically < 60 seconds, but can spike during high DML activity. Monitor with `DATABASE_REPLICATION_USAGE_HISTORY`.
- **Replication objects:** Not all objects are replicated automatically (e.g., stages, pipes, tasks, stored procedures → must be recreated in the secondary).
- **Failover downtime:** Converting a secondary to primary takes 5-10 seconds. Connections must be re-established.
- **Storage costs:** Secondary database incurs full storage cost (data is replicated, not zero-copy).

### Trade-offs & Alternatives

| DR Strategy | RPO (Data Loss) | RTO (Recovery Time) | Cost |
|------------|-----------------|---------------------|------|
| Database Replication | < 60 seconds | 5-10 minutes | 2x storage + replication credits |
| Client-side Multi-casting | Minutes | Minutes | Application-level |
| Snowpipe + External Table | Minutes-Hours | Hours | Low (external storage) |

### Principal-Level Considerations
- **Replication group design:** Group related databases into a replication group for atomic failover (all databases fail over together).
- **Automated failover:** Use a health-check script + Snowflake API. Do NOT rely on manual failover for critical systems.
- **Testing:** Conduct a failover drill quarterly. Verify that secondary account has the same roles, warehouses, and network policies.

### Follow-up Questions
1. **What objects are NOT replicated?**
   - **Answer:** Stages (external and internal), pipes, tasks (must be recreated), data sharing shares, network policies (account-level), and stored procedures referencing non-replicated objects.

2. **How do you minimize replication lag?**
   - **Answer:** Use Business Critical Edition. Avoid long-running DMLs that block replication. Schedule large batch DMLs during low-load periods.

3. **Can you replicate to multiple secondary regions?**
   - **Answer:** Yes — you can create up to 10 replicas per database. Each secondary incurs separate replication credits and storage costs.

4. **What happens to queries on the secondary during failover?**
   - **Answer:** Queries continue on the secondary database (it is readable during replication). During failover, any in-flight queries are interrupted.

5. **How is replication billed?**
   - **Answer:** Storage costs for the secondary database (full copy). Replication credits for data transfer (varies by region).

6. **Can you have a secondary database in a different cloud?**
   - **Answer:** Yes — Snowflake supports cross-cloud replication (e.g., AWS us-east-1 → Azure eastus). Requires Business Critical Edition on both sides.

---

## Q29. Snowflake Editions & Pricing (Standard, Enterprise, Business Critical, Virtual Private Snowflake)

### Question
Compare Snowflake Editions: Standard, Enterprise, Business Critical, and Virtual Private Snowflake (VPS). What features justify upgrading from one edition to the next?

### Answer
Snowflake offers four editions, each adding features on top of the previous:

**Standard Edition:**
- Base features: cloud services, storage, compute, Time Travel (1 day)
- Data sharing (both ways)
- Materialized views (limited)
- $2/credit

**Enterprise Edition (most common):**
- All Standard + Multi-cluster warehouses, Materialized views (full), Clustering, Resource monitors, Database replication (to Enterprise†)
- Time Travel up to 90 days
- $3/credit

**Business Critical Edition:**
- All Enterprise + Data encryption at rest with customer-managed keys (CMK/DEK), Database replication (to Business Critical only), HIPAA compliance, Fail-safe*, PrivateLink
- $4/credit

**Virtual Private Snowflake (VPS) — now part of Business Critical+ customization:**
- Isolated compute and storage
- Dedicated metadata store
- Additional compliance (FedRAMP, ITAR)
- Custom per-contract pricing (no public credit rate)

### Example
```sql
-- Check current account edition
SELECT CURRENT_ACCOUNT() AS account_locator,
       CURRENT_REGION() AS region,
       SYSTEM$WHITELIST() AS edition_details;

-- Check which features are available
SHOW PARAMETERS LIKE '%edition%' IN ACCOUNT;

-- Resource Monitor (Enterprise+)
CREATE OR REPLACE RESOURCE MONITOR monthly_budget
    CREDIT_QUOTA = 1000
    FREQUENCY = MONTHLY
    START_TIMESTAMP = '2024-01-01 00:00:00'
    TRIGGERS ON 80 PERCENT NOTIFY
             ON 100 PERCENT NOTIFY
             ON 120 PERCENT NOTIFY
    WAREHOUSE = analytics_wh;

-- Multi-cluster warehouse (Enterprise+)
CREATE WAREHOUSE enterprise_wh
    WAREHOUSE_SIZE = LARGE
    MAX_CLUSTER_COUNT = 5;

-- Materialized view (Enterprise+)
CREATE MATERIALIZED VIEW customer_summary_mv AS
    SELECT customer_id, COUNT(*), SUM(amount)
    FROM transactions
    GROUP BY customer_id;

-- Time Travel 90 days (Enterprise+)
ALTER TABLE sensitive_data
    SET DATA_RETENTION_TIME_IN_DAYS = 90;

-- Customer-managed keys (Business Critical+)
-- (Configured via AWS KMS / Azure Key Vault / GCP Cloud KMS in Snowflake UI)

-- Check feature usage
SELECT
    feature_name,
    feature_value,
    description
FROM SNOWFLAKE.ACCOUNT_USAGE.FEATURE_USAGE
WHERE feature_name IN (
    'MULTI_CLUSTER_WAREHOUSE',
    'MATERIALIZED_VIEWS',
    'SEARCH_OPTIMIZATION',
    'DATABASE_REPLICATION'
);
```

### Edge Cases & Caveats
- **Edition upgrades:** Upgrading from Standard to Enterprise is seamless (no downtime). Downgrading is NOT supported (you must recreate the account).
- **Feature parity across clouds:** Not all features are available simultaneously on AWS/Azure/GCP. Check the Snowflake release notes.
- **VPS is not a separate edition anymore:** It is a customization on top of Business Critical. Requires dedicated infrastructure.

### Pricing Summary

| Edition | Credit Price | Time Travel Max | Key Features |
|---------|-------------|----------------|--------------|
| Standard | $2 | 1 day | Basic Snowflake |
| Enterprise | $3 | 90 days | Multi-cluster, MV, clustering |
| Business Critical | $4 | 90 days | CMK, HIPAA, PrivateLink |
| VPS | Custom | 90+ days | Dedicated, compliance |

### Principal-Level Considerations
- **Edition ROI analysis:** For most enterprises, the jump from Standard to Enterprise pays for itself through multi-clustering (reducing queue time) and 90-day Time Travel (reducing backup costs).
- **Business Critical for regulated industries:** HIPAA, SOC 2, and PCI-DSS require Business Critical. If your data includes PHI/PII, start with Business Critical.
- **VPS for massive scale:** Deploy over 1 PB of storage and 10,000+ queries/hour. The dedicated metadata store improves concurrency.

### Follow-up Questions
1. **What features are in Enterprise but not Standard?**
   - **Answer:** Multi-cluster warehouses, materialized views, clustering (automatic), resource monitors, 90-day Time Travel, database replication (to Enterprise accounts).

2. **What additional security does Business Critical provide?**
   - **Answer:** Customer-managed encryption keys (CMK/DEK), HIPAA compliance, SOC 2 Type II, PrivateLink, and fail-safe* (for an additional 7 days beyond Time Travel).

3. **Can you have a mix of editions in the same account?**
   - **Answer:** No — one account = one edition. But you can have multiple accounts in different editions under the same organization.

4. **How do you estimate costs when upgrading from Standard to Enterprise?**
   - **Answer:** Credit price increases ~50% ($2 → $3). However, compute may decrease (more efficient queries via MV/clustering). Model both scenarios: current spend × 1.5 vs. new spend × improved efficiency.

5. **What is included in the "Cloud Services" cost?**
   - **Answer:** Metadata management, query optimization, security, authentication. Generally 0-10% of total Snowflake spend (free up to 10% of warehouse credits).

6. **Can you switch from Enterprise to Business Critical without downtime?**
   - **Answer:** Yes — contact Snowflake Support. The upgrade takes 1-2 hours with zero downtime. Once upgraded, you cannot downgrade.

---

## Q30. Incident Response & Troubleshooting (performance, cost, data issues)

### Question
Describe your approach to handling a production incident in Snowflake — performance degradation, cost spike, or data corruption. What tools and queries do you use?

### Answer
A structured incident response approach:

**1. Detection & Triage:**
- Is the issue **performance** (queries slow, queuing)?
- **Cost** (credit spike, unexpected warehouse usage)?
- **Data** (missing rows, incorrect values, schema drift)?

**2. Investigation Tools:**
- `QUERY_HISTORY`: Identify problematic queries, execution plans, bytes scanned.
- `WAREHOUSE_LOAD_HISTORY`: Check warehouse queuing and scaling.
- `WAREHOUSE_METERING_HISTORY`: Credit consumption breakdown.
- `TABLE_STORAGE_METRICS`: Check storage anomalies (Time Travel spikes).
- `ACCESS_HISTORY`: Who/what accessed the affected data.

**3. Containment:**
- Suspend offending warehouses (`ALTER WAREHOUSE ... SUSPEND`)
- Pause resource-intensive queries (`SYSTEM$CANCEL_QUERY(query_id)`)
- Restrict access to affected data (alter masking policy to limit access)
- For data corruption: use Time Travel to restore (CREATE OR REPLACE TABLE ... CLONE ... AT(...))

**4. Root Cause Analysis:**
- Review query patterns, data volume changes, parameter changes
- Check if a release or schema change triggered the issue
- Review monitoring dashboards for trends

**5. Resolution & Post-mortem:**
- Apply fix (query optimization, warehouse sizing, data correction)
- Document findings and prevention strategy

### Example
```sql
-- STEP 1: Find the top 5 slowest queries in the last hour
SELECT
    query_id,
    query_text,
    warehouse_name,
    warehouse_size,
    total_elapsed_time / 1000 AS elapsed_sec,
    bytes_scanned / (1024*1024*1024) AS gb_scanned,
    partitions_scanned,
    partitions_total,
    bytes_spilled_to_remote_storage / (1024*1024) AS mb_spilled,
    EXECUTION_STATUS,
    error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('hour', -1, CURRENT_TIMESTAMP())
  AND query_type IN ('SELECT', 'INSERT', 'MERGE')
ORDER BY total_elapsed_time DESC
LIMIT 5;

-- STEP 2: Check warehouse queueing
SELECT
    warehouse_name,
    average_running,
    average_queued,
    average_blocked,
    session_count,
    query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time >= DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY average_queued DESC;

-- STEP 3: Check credit spike
SELECT
    DATE_TRUNC('hour', start_time) AS usage_hour,
    warehouse_name,
    SUM(credits_used) AS total_credits,
    COUNT(*) AS query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY usage_hour DESC;

-- STEP 4: Check data corruption (compare row counts)
SELECT 'sales' as table_name, COUNT(*) as row_count
FROM sales
UNION ALL
SELECT 'sales_yesterday', COUNT(*)
FROM sales AT(OFFSET => -86400);

-- STEP 5: Restore from Time Travel (if data issue)
CREATE OR REPLACE TABLE sales
    CLONE sales
    AT(TIMESTAMP => '2024-06-20 14:30:00'::TIMESTAMP_NTZ);

-- STEP 6: Kill a runaway query
SELECT SYSTEM$CANCEL_QUERY('019e3f2e-0601-7b00-0000-0000c7d20001');

-- STEP 7: Check if query acceleration could help
EXPLAIN USING QUERY_ACCELERATION
SELECT * FROM large_table
WHERE created_date >= '2024-01-01'
  AND region = 'EMEA';
```

### Edge Cases & Caveats
- **Time Travel restore timing:** Restoring from Time Travel takes time proportional to table size (minutes to hours for TB+).
- **Query cancellation impact:** Cancelling a query that is part of a MERGE may leave the table in an intermediate state. Verify consistency.
- **Warehouse suspension:** Suspending a warehouse kills all running queries. Notify affected users first.

### Incident Response Playbook Summary

| Symptom | Immediate Action | Investigation |
|---------|-----------------|---------------|
| Queries slow, high queue | Add clusters or increase warehouse size | Check `QUERY_HISTORY`, `WAREHOUSE_LOAD_HISTORY` |
| Credit spike | Identify warehouse and user | Check `WAREHOUSE_METERING_HISTORY`, join with `QUERY_HISTORY` |
| Data missing/corrupt | Restore from Time Travel | Compare row counts, check `ACCESS_HISTORY` for unauthorized changes |
| Query failing | Check error message, check table existence | Review `QUERY_HISTORY.error_message`, check schema versions |
| Network timeout | Check network policy, VPC endpoint | Test connectivity, check `LOGIN_HISTORY` for failures |
| Storage cost spike | Check Time Travel bytes | Run `TABLE_STORAGE_METRICS`, reduce retention if appropriate |

### Principal-Level Considerations
- **Proactive monitoring:** Set up alerts for: (1) Queries queuing > 10 minutes, (2) Credit usage > 120% of daily budget, (3) Time Travel bytes > 500 GB per table.
- **Runbook automation:** Create stored procedures for common incident responses (e.g., `restore_table(target_table, timestamp)`).
- **Post-mortem culture:** After every incident, write a post-mortem including: timeline, root cause, fix, and prevention. Share with the team.

### Follow-up Questions
1. **How do you set up proactive monitoring for cost spikes?**
   - **Answer:** Create a resource monitor with notifications at 80%, 100%, and 120% of monthly budget. Use `SYSTEM$SEND_EMAIL` or Snowflake alerts to notify the team.

2. **What is the most common cause of performance degradation?**
   - **Answer:** Inefficient queries (full table scans, missing filters) combined with undersized warehouses. Also: cache fragmentation in multi-cluster warehouses.

3. **How do you handle a data corruption incident?**
   - **Answer:** (1) Stop writes to the affected table. (2) Query Time Travel to confirm the last known good state. (3) Use `CREATE OR REPLACE TABLE ... CLONE ... AT(TIMESTAMP => ...)` to restore. (4) Replay any lost data from source.

4. **How do you prevent accidental data deletion?**
   - **Answer:** Use Time Travel (retention >= 7 days). Implement role-based access control (only DATA_ENGINEER role can DROP). Use `ALTER TABLE ... SET DATA_RETENTION_TIME_IN_DAYS = 30`.

5. **What tools does Snowflake provide for query optimization?**
   - **Answer:** EXPLAIN plans, `CLUSTERING_HISTORY`, `QUERY_HISTORY`, `SYSTEM$ESTIMATE_QUERY_ACCELERATION`, and the Query Profile UI (visual execution plan).

6. **How do you handle a failover incident?**
   - **Answer:** (1) Verify the primary region is down (don't failover prematurely). (2) Run `ALTER DATABASE ... PRIMARY` on the secondary. (3) Update application connection strings. (4) Notify users. (5) After primary recovers, failback.