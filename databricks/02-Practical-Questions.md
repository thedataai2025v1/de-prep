# Databricks — Practical Interview Questions (30 Q&As)

Principal Data Engineer level. Hands-on scenarios, code reviews, architectural decisions, debugging and production incident response with Databricks, Delta Lake, Spark, DLT and Unity Catalog.

---

## Q1. Delta Table MERGE Fails with Concurrent Writes — Resolve the Conflict

**Scenario:** A production Delta table `events` is updated by two concurrent streaming jobs (micro-batch) using MERGE. One job reports:

```
[2024-03-15 14:32:18] {Executing MERGE INTO events}
org.apache.spark.sql.delta.DeltaConcurrentModificationException:
  Conflicting concurrent writes detected.
  Multiple writers completed with conflicting changes to the table.
```

The second job writes successfully. Both jobs have retries enabled. What happened and how do you fix it?

**Answer:**

### Root Cause

Delta Lake uses **optimistic concurrency control**. Two concurrent MERGE operations both read the same table version, try to write, and one fails because the other already committed.

```
Time:
  T1: Job A reads version 42 (start)
  T2: Job B reads version 42 (start)
  T3: Job A commits version 43 (success)
  T4: Job B tries to commit version 43 → FAILS (version 43 exists)
```

### Immediate Resolution

```python
# Option 1: Retry with a backoff in the application
from delta.tables import DeltaTable
import time

max_retries = 5
retry_count = 0
backoff_base = 1  # seconds

while retry_count < max_retries:
    try:
        delta_table = DeltaTable.forPath(spark, "/path/to/events")
        delta_table.alias("t").merge(
            spark_df.alias("s"),
            "t.event_id = s.event_id"
        ).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()
        break  # success
    except DeltaConcurrentModificationException as e:
        retry_count += 1
        wait = backoff_base * (2 ** retry_count)  # exponential backoff
        print(f"Retry {retry_count}/{max_retries}, waiting {wait}s")
        time.sleep(wait)
```

### Production Fixes

| Fix | Description | When to Use |
|-----|-------------|-------------|
| **Stagger job schedules** | Offset job start times by 30+ seconds | Two scheduled jobs |
| **Single writer pattern** | One job does all MERGEs; others append | High-volume stream + batch |
| **Partition isolation** | Each job writes to a different partition | Partitioned tables (e.g., `event_date`) |
| **Delta Lake idempotency** | Use `foreachBatch` with dedup | Streaming MERGE |
| **Use Append + Table Dedup** | Append new data, deduplicate later | Low-latency requirement |

### Stream-Safe MERGE with foreachBatch

```python
def upsert_events(micro_batch_df, batch_id):
    delta_table = DeltaTable.forPath(spark, "/path/to/events")
    delta_table.alias("t").merge(
        micro_batch_df.alias("s"),
        "t.event_id = s.event_id AND t.event_date = s.event_date"
    ).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()

# Use foreachBatch to enable MERGE in streaming
streaming_df.writeStream \
    .foreachBatch(upsert_events) \
    .outputMode("update") \
    .option("checkpointLocation", "/path/to/checkpoint") \
    .start()
```

### Principal-Level Insight

**Prefer partition-level isolation** over excessive retries. Partition by `event_date` and have each job process a distinct date range. This eliminates concurrency conflicts entirely for most pipelines.

### Follow-up

**Q:** What if both jobs write to the same partition?
**A:** Use an intermediate staging table for the second job, then run a single consolidation MERGE to the main table.

---

## Q2. Debug a Spark Job That Spills to Disk and Slows to a Crawl

**Scenario:** A nightly Spark aggregation job processing 500 GB of event data completes in 2 hours normally. Tonight it has been running for 6 hours and is still going. The Spark UI shows heavy disk spill on the sort stage:

```
Stage 8 (shuffle sort): 45.2 GB read, 89.4 GB spilled
Stage 9 (aggregate): 32.1 GB read, 67.8 GB spilled
Executor: 5 executors, 4 cores, 16 GB memory each
```

What caused the spill and how do you fix it?

**Answer:**

### Root Cause

The executor memory is too small for the data being shuffled and aggregated. When data exceeds the available execution memory, Spark spills to disk, which is 100-1000x slower than in-memory processing.

```
Memory breakdown per executor (16 GB):
  - Spark overhead: 0.4 GB (spark.executor.memoryOverhead)
  - Execution memory: ~6.2 GB (spark.memory.fraction = 0.6)
  - Storage (cache/shuffle): ~9.4 GB

Sort stage needs ~12 GB per executor for 45 GB of data → spills 89 GB
```

### Diagnostic Commands

```python
# Check current configuration
spark.conf.get("spark.executor.memory")         # "16g"
spark.conf.get("spark.sql.shuffle.partitions")  # "200"
spark.conf.get("spark.sql.adaptive.enabled")    # "false"

# Check partition count of input data
spark.read.format("delta").load("/path/to/events") \
    .rdd.getNumPartitions()  # returns number of partitions
```

### Solutions in Order of Preference

**Solution 1: Increase executor memory (easiest)**
```python
# Cluster config: increase executor memory
spark.conf.set("spark.executor.memory", "32g")
spark.conf.set("spark.executor.memoryOverhead", "6g")

# Or in cluster config:
# Spark config: spark.executor.memory 32g
#               spark.executor.memoryOverhead 6g
```

**Solution 2: Increase shuffle partitions (better parallelism)**
```python
# More partitions = less data per task = less memory needed per task
spark.conf.set("spark.sql.shuffle.partitions", "1000")

# With AQE, this auto-adjusts:
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.parallelismFirst", "true")
```

**Solution 3: Optimize the query to reduce data shuffled**
```python
# Before: shuffles all columns
df.groupBy("user_id", "region").agg(
    sum("amount").alias("total_amount"),
    count("*").alias("event_count"),
    sum("tax").alias("total_tax")
)

# After: pre-filter columns to only what's needed
df.select("user_id", "region", "amount", "event_count", "tax") \
  .groupBy("user_id", "region") \
  .agg(
      sum("amount").alias("total_amount"),
      count("*").alias("event_count"),
      sum("tax").alias("total_tax")
  )
```

**Solution 4: Use bucketing for the aggregation key**
```python
# Write with bucketing
df.write \
    .format("delta") \
    .bucketBy(100, "user_id") \
    .sortBy("user_id") \
    .saveAsTable("bucketed_events")

# Future aggregations on user_id will not spill
spark.table("bucketed_events") \
    .groupBy("user_id") \
    .agg(sum("amount"))  # No shuffle needed
```

### Principal-Level Insight

**Never blindly increase memory.** Profile the stage that's spilling. If it's the shuffle sort stage, increase shuffle partitions first. If it's the aggregation stage, increase executor memory. AQE (Adaptive Query Execution) should be on by default for all production jobs.

### Follow-up

**Q:** How do you detect spill without the Spark UI?
**A:** Check the event log or use `spark.listener` to capture stage metrics programmatically:
```python
spark.conf.set("spark.extraListeners", "com.databricks.sql.DatabricksEventListener")
# Or parse event logs: spark.executor.metrics.spill.(memory/disk)*
```
---

## Q3. Production MERGE Job Fails with "Schema Changed" Error Mid-Stream

**Scenario:** Your Delta Live Table (DLT) pipeline with a MERGE operation has been running successfully for weeks. This morning it fails:

```
AnalysisException: A schema mismatch has been detected...
Incompatible change detected in column 'user_agent':
  Expected: StringType
  Found: StructType(user_agent_data: StringType, ...)

The source data for this MERGE has changed schema.
```

Your upstream source team added a new field to `user_agent` in the API response, changing it from a string to a nested struct. How do you handle this without losing data?

**Answer:**

### Immediate Mitigation (Pause the Pipeline)

1. Pause the DLT pipeline to prevent corrupted writes
2. Investigate the schema change from the upstream source
3. Decide on a schema evolution strategy

### Solution 1: Enable mergeSchema (Quickest)

```python
@dlt.view
def source():
    return (
        spark.readStream.format("cloudFiles")
        .option("cloudFiles.schemaEvolutionMode", "rescue")
        .option("cloudFiles.inferColumnTypes", "true")
        .load(f"{source_path}/events")
    )

@dlt.table
def silver_events():
    # Enable mergeSchema for Delta writes
    return (
        dlt.read_stream("source")
        .writeStream
        .option("mergeSchema", "true")
        .trigger(availableNow=True)
        .toTable("silver_events")
    )
```

But this adds a `StructType` column — not what you want. Better to explicitly handle the new schema.

### Solution 2: Normalize the Schema (Recommended)

```python
@dlt.table
def silver_events():
    df = dlt.read_stream("source")

    # Normalize user_agent: if it's a struct, keep only relevant fields
    df = df.withColumn(
        "user_agent",
        when(
            col("user_agent").cast("string").isNotNull(),
            col("user_agent").cast("string")
        ).otherwise(
            col("user_agent.user_agent_data.cast("string"))
        )
    )

    # Or flatten the entire struct
    df = df.withColumn(
        "user_agent_browser",
        col("user_agent.browser")
    ).withColumn(
        "user_agent_version",
        col("user_agent.version")
    ).drop("user_agent")

    return df
```

### Solution 3: Use rescue data column (Catch-all)

```python
# Save the raw data for future reprocessing
@dlt.table
def silver_events_rescue():
    return (
        dlt.read_stream("source")
        .option("rescueDataColumn", "_rescued_data")
        .withColumn("user_agent_normalized", col("_rescued_data.user_agent"))
    )
```

### Principal-Level Insight

**Set up a schema registry or schema validation layer.** Before the data hits DLT, validate against a known schema. When the schema changes unexpectedly, route mismatches to a quarantine table for human review, then update the schema registry.

```python
# Validation using Delta sharing or Unity Catalog schema
schema_registry = spark.table("schema_registry.events")

# Validate source schema before writing
expected_columns = set(schema_registry.columns)
source_columns = set(source_df.columns)

if expected_columns != source_columns:
    # Log the mismatch and route to quarantine
    source_df.write.format("delta").mode("append").save("/quarantine/events")
    raise Exception(f"Schema mismatch: expected {expected_columns}, got {source_columns}")
```

### Follow-up

**Q:** What's the risk of `mergeSchema` in production?
**A:** It silently accepts any column addition or type change, which can break downstream consumers. Only use it when you carefully control both source and sink schema evolution (e.g., via a schema registry).

---

## Q4. Slow `dalteTableOptimize` Job on a 10 TB Delta Table

**Scenario:** You run `OPTIMIZE delta.\`/path/to/10_tb_table\`` on a daily basis. It used to take 30 minutes but now takes over 4 hours. The table has 50,000 small Parquet files (< 16 MB each) in one partition. No one remembers running `VACUUM` in months. What's the problem and what do you do?

**Answer:**

### Root Cause

The `OPTIMIZE` operation must rewrite all 50,000 small files into larger ones. If the table hasn't been vacuumed, old file versions accumulate, making `OPTIMIZE` worse because:

1. **More files to process:** Old versions of files that were already compacted still exist
2. **Increased metadata overhead:** The Delta log tracks all versions, slowing log reads
3. **Disk space:** The table size includes old, unreferenced files

```
Table structure:
  - current version: 500 GB (active data files)
  - historical versions: 9.5 TB (old, unreferenced files)
  - total on disk: ~10 TB
  - file count: 50,000 active files + 200,000 historical files
```

### Immediate Fix: Run VACUUM First

```python
# Step 1: Run VACUUM with a retention period
spark.sql(f"VACUUM delta.`/path/to/10_tb_table` RETAIN 168 HOURS")  # 7 days

# Step 2: Run OPTIMIZE after VACUUM
spark.sql(f"OPTIMIZE delta.`/path/to/10_tb_table`")
```

### Long-Term: Set Up VACUUM as Part of the Maintenance Cycle

```python
# Schedule this after OPTIMIZE:
from delta.tables import DeltaTable

dt = DeltaTable.forPath(spark, "/path/to/table")
# VACUUM retains files from the last 7 days
dt.vacuum(retentionHours=168)
```

### Root Cause Prevention: Set Retention Properties on the Table

```sql
ALTER TABLE 10_tb_table SET TBLPROPERTIES (
    'delta.deletedFileRetentionDuration' = 'interval 14 days',   -- How long VACUUM keeps files
    'delta.logRetentionDuration' = 'interval 30 days'             -- How long to keep transaction log
);
```

### Optimize with Z-ORDER by the Right Column

```sql
OPTIMIZE delta.`/path/to/10_tb_table`
ZORDER BY (event_date, region)  -- High-cardinality column that's often filtered
```

### Principal-Level Insight

**Automate the full maintenance cycle in a scheduled job:**

```python
def optimize_vacuum(table_path, zorder_cols=["event_date"]):
    dt = DeltaTable.forPath(spark, table_path)

    # Step 1: Optimize with ZORDER
    dt.optimize().executeZOrderBy(*zorder_cols)

    # Step 2: Vacuum with retention (after optimize, old files are safe to delete)
    dt.vacuum(retentionHours=168)

    # Step 3: Analyze table stats for cost-based optimizer
    dt.generate("symlink_format_manifest")
```

### Follow-up

**Q:** How do you detect that a table needs VACUUM before it becomes slow?
**A:** Monitor table size vs. data size:

```python
# Check Delta table size on disk vs. actual data size
describe = spark.sql(f"DESCRIBE EXTENDED delta.`{table_path}`")
size_on_disk = describe.filter("col_name = 'Size'").select("data_type").collect()[0][0]
data_size = spark.read.format("delta").load(table_path).count() * avg_row_size_bytes

if size_on_disk > data_size * 1.5:  # 50% more
    print("Table has significant bloat — schedule VACUUM")
```

---

## Q5. Streaming Job Lags and Consumer Offset Lag Grows Unbounded

**Scenario:** A Structured Streaming job consuming from a Kafka topic with 20 partitions processes 50K events/second normally. Recently, consumer lag has grown from 10 seconds to over 30 minutes. The Spark UI shows the batch processing time is 2 minutes for a 5-second trigger interval, meaning the processing can't keep up with the incoming rate.

```python
# Current configuration
df = spark.readStream.format("kafka") \
    .option("subscribe", "user_events") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .load()

# Aggregation job
aggregated = df.groupBy(window("timestamp", "5 minutes"), "user_id").agg(...)

query = aggregated.writeStream \
    .outputMode("update") \
    .trigger(processingTime="5 seconds") \
    .option("checkpointLocation", "/path/to/checkpoint") \
    .start()
```

What's wrong and how do you fix it?

**Answer:**

### Root Cause

The trigger interval (5 seconds) is less than the batch processing time (2 minutes). The streaming job spends more time processing each batch than the interval between triggers, so lag grows unbounded.

```
Processing timeline:
  Batch 1: 0:00 - 0:02 (triggers at 0:00, processes data from 0:00-0:05)
  Batch 2: 0:05 - 0:07 (triggers at 0:05, still processing...)
  Batch 3: 0:10 - waiting...
```

### Immediate Fix: Increase Trigger Interval

```python
# Option 1: Align trigger to processing time
query = aggregated.writeStream \
    .trigger(processingTime="2 minutes") \
    .option("checkpointLocation", "/path/to/checkpoint") \
    .start()

# Option 2: Use processingTime="Once" for batch-like processing
query = aggregated.writeStream \
    .trigger(availableNow=True) \
    .option("checkpointLocation", "/path/to/checkpoint") \
    .start()
```

### Root Fix: Optimize the Processing

**Fix 1: Reduce shuffles in the aggregation**
```python
# Before: 200 shuffle partitions (default) — too many for this data
# After: set shuffle partitions based on data size
spark.conf.set("spark.sql.shuffle.partitions", "20")  # Match Kafka partitions
```

**Fix 2: Pre-aggregate before the windowed aggregation**
```python
# If the aggregation is expensive, pre-aggregate per-micro-batch first
from pyspark.sql.functions import approx_count_distinct

# Pre-aggregate on user_id before windowing
per_user = df.groupBy("user_id").agg(sum("amount").alias("amount"))

# Then do the windowed aggregation on pre-aggregated data
aggregated = per_user.groupBy(window("timestamp", "5 minutes"), "user_id") \
    .agg(sum("amount"))
```

**Fix 3: Increase parallelism**
```python
# Re-partition input to match cluster processing capacity
df = df.repartition(spark.sparkContext.defaultParallelism * 2)
```

### Principal-Level Insight

**Always monitor batch processing time vs. trigger interval.** Set up a CloudWatch / Azure Monitor alert on the `processingTime/triggerDuration` ratio. If it consistently exceeds 0.8, you need to either:
- Increase the trigger interval
- Increase cluster size
- Optimize the processing logic

### Follow-up

**Q:** What happens if you set the trigger interval to 0?
**A:** Spark processes as fast as possible (no trigger delay). This is good for throughput but can overwhelm downstream sinks (e.g., Delta write throughput limits). Use with caution.

---

## Q6. DLT Pipeline Expectation Failure: Orchestration vs. Remediation

**Scenario:** Your DLT pipeline has an expectation that checks for null primary keys:

```python
@dlt.expect_or_drop("valid_pk", "pk IS NOT NULL")
```

Today the upstream source sent 15% of records with null PKs. The pipeline dropped these records silently. The business team discovered the data loss during reconciliation. What went wrong and how do you build a robust data quality system?

**Answer:**

### The Problem

`expect_or_drop` silently drops bad records. There is no alert, no quarantine, and no way to replay the dropped records later. For production pipelines, this hides data quality issues.

### Better Approach: Use Multiple Expectation Levels

```python
@dlt.expect("record_complete", "pk IS NOT NULL AND event_type IS NOT NULL AND timestamp IS NOT NULL")

@dlt.expect_or_drop("valid_pk", "pk IS NOT NULL")

@dlt.expect_or_fail("no_duplicate_pk", "pk IS UNIQUE")

@dlt.table
def silver_events():
    df = dlt.read_stream("bronze_events")

    # Track dropped records for alerting
    dlt.create_streaming_live_table("silver_events_dropped")
    dlt.apply_changes(
        target="silver_events_dropped",  # quarantine table
        source="bronze_events",           # source view
        keys=["pk"],
        sequence_by="timestamp",
        apply_as_deletes=False,
        expect_all_valid_records=False,
        expectation={
            "valid_pk": "pk IS NOT NULL"
        }
    )

    return df
```

### Production Data Quality Framework

1. **Quarantine bad records** instead of dropping them
2. **Alert on expectation failures** via webhook or Databricks alerts
3. **Set up a data quality dashboard** with the `dlt.expectations` metadata table
4. **Create a replay pipeline** that reads from quarantine and fixes data

```python
# Query DLT expectation metrics
spark.sql("""
    SELECT
        pipeline_id,
        dataset_name,
        expectation,
        passing_records,
        failing_records
    FROM
        pipeline_metrics.events_silver_expectations
    ORDER BY
        timestamp DESC
""").show()
```

### Principal-Level Insight

**Build expectation remediation as part of the pipeline orchestration:**

```python
# Use dlt.expect_or_fail for critical quality gates
@dlt.expect_or_fail("no_nulls_for_payments", "amount IS NOT NULL AND currency IS NOT NULL")

# Use dlt.expect_or_warn for alert-only
@dlt.expect_or_warn("high_accuracy", "accuracy > 0.95")

# Use custom metrics to drive auto-scaling of data quality scrutiny
@dlt.table
def quality_metrics():
    return dlt.read_stream("bronze_events").agg(
        count(when(col("pk").isNull(), 1)).alias("null_pk_count"),
        count(when(col("amount") < 0, 1)).alias("negative_amount_count")
    )
```

### Follow-up

**Q:** How do you replay records that were dropped?
**A:** Re-process the bronze table with a broader expectation:
```python
# Re-process with expect_or_warn instead of expect_or_drop
@dlt.expect_or_warn("valid_pk", "pk IS NOT NULL")
# Upsert to silver with the corrected data
```

---

## Q7. Unity Catalog Query Fails with "NoSuchTableException" After Table Rename

**Scenario:** A downstream BI dashboard that queries `dashboard.v_kpi_summary` (a view in Unity Catalog) has worked for months. The data team renamed `bronze.events` to `bronze.events_v2` because of a schema migration. Now the BI dashboard fails with:

```
AnalysisException [NO_SUCH_TABLE]:
  Table or view not found: bronze.events
```

How do you fix this and prevent it from happening again?

**Answer:**

### Root Cause

The view `dashboard.v_kpi_summary` was defined with a direct reference to `bronze.events`. When the table was renamed, the view's underlying reference became invalid.

### Immediate Fix

```sql
-- 1. Identify all views that reference the old table
SELECT * FROM information_schema.views
WHERE view_definition LIKE '%bronze.events%';

-- 2. Recreate the view with the new table name
CREATE OR REPLACE VIEW dashboard.v_kpi_summary AS
SELECT
    event_type,
    count(*) AS event_count,
    current_timestamp() AS last_refreshed
FROM bronze.events_v2
GROUP BY event_type;
```

### Long-Term Fixes

**Fix 1: Use Data Contracts in Unity Catalog**

Set up a data contract between the bronze and dashboard teams. Unity Catalog's lineage tracking helps identify all downstream dependencies:

```sql
-- Check lineage before renaming
DESCRIBE TABLE EXTENDED bronze.events;
-- Look for: created in catalog/dashboard.v_kpi_summary

-- Use system tables for lineage
SELECT * FROM system.lineage.tables
WHERE target_table = 'bronze.events';
```

**Fix 2: Use Aliases / Views as Interfaces**

```sql
-- Create a "latest" alias for the main table
CREATE OR REPLACE VIEW bronze.events_latest AS
SELECT * FROM bronze.events;

-- Dashboard queries: bronze.events_latest
-- When renaming: just point the view to the new table
CREATE OR REPLACE VIEW bronze.events_latest AS
SELECT * FROM bronze.events_v2;
```

**Fix 3: Use Table Properties for Migration**

```sql
-- On the old table, set a redirect property
ALTER TABLE bronze.events SET TBLPROPERTIES (
    'deprecated' = 'true',
    'replaced_by' = 'bronze.events_v2'
);

-- Automate checks: pre-production validation
-- (Run as part of your CI/CD pipeline)
```

### Principal-Level Insight

**Adopt semantic naming and data contracts.** Renaming tables should never break downstream consumers. Use:
- **Logical data contracts** (documentation + automated lineage checks)
- **Views as API layers** (bronze/ becomes an interface, not a physical table)
- **Automated dependency checks** in your CI/CD pipeline before renaming any table

### Follow-up

**Q:** How do you find all downstream dependencies of a table?
**A:** Query Unity Catalog system tables:
```sql
SELECT * FROM system.lineage.tables
WHERE source_table = 'bronze.events'
   OR target_table = 'bronze.events';
```
---

## Q8. Auto Loader Job Reads Duplicate Files After Pipeline Restart

**Scenario:** Your Auto Loader job reads from an S3 bucket with `cloudFiles.allowOverwrites = false`. After a planned pipeline restart, several file batches are re-ingested, causing duplicate records in the target table. Your dedup logic in the MERGE query doesn't catch them all. What happened?

**Answer:**

### Root Cause

Auto Loader tracks files using **directory listing** by default. If the checkpoint location was lost or corrupted during the restart, Auto Loader doesn't remember which files were already ingested and re-ingests everything.

```
Before restart:
  Auto Loader checkpoint: {file1.json, file2.json, file3.json}
  Ingested: file1, file2, file3

After restart (checkpoint lost):
  Auto Loader checkpoint: {}  (empty!)
  Ingested again: file1, file2, file3 → DUPLICATES
```

### Prevention: Use File Notification Mode

```python
# Instead of directory listing, use file notification mode
df = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.useNotifications", "true") \
    .option("cloudFiles.resourceManagerQueue", "arn:aws:sqs:us-east-1:...") \
    .option("cloudFiles.region", "us-east-1") \
    .load("/path/to/source")
```

File notification mode (SQS / Event Grid) only processes new files, not re-scanned ones. Even with a checkpoint loss, it won't re-ingest files that were already notified.

### Fix for Existing Data: Deduplicate After Ingestion

```python
# Add a watermark-like dedup layer
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number

deduplicated = df.withColumn(
    "rn",
    row_number().over(
        Window.partitionBy("event_id").orderBy(col("file_path").desc())
    )
).filter("rn = 1").drop("rn")
```

### Better: Use Delta Change Data Feed for Error Handling

```python
# Enable change data feed on the target table
spark.sql("ALTER TABLE events SET TBLPROPERTIES (delta.enableChangeDataFeed = true)")

# Read changes since last processed version
changes = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", last_processed_version) \
    .table("events")

# Dedup based on event_id and file_path
deduped = changes.dropDuplicates(["event_id"])
```

### Principal-Level Insight

**Never rely on directory listing for production Auto Loader.** Use file notification mode with SQS or Event Grid, and store checkpoints in durable storage with proper backup:

```python
# Durable checkpoint location
checkpoint_path = "abfss://checkpoints@storage.dfs.core.windows.net/events_loader"

df.writeStream \
    .format("delta") \
    .option("checkpointLocation", checkpoint_path) \
    .table("events")
```

### Follow-up

**Q:** What's the risk of file notification mode?
**A:** If SQS/Event Grid loses messages, you lose events. Mitigate by:
- Enabling dead-letter queues for failed notifications
- Running periodic "catch-up" cycles with directory listing as a safety net

---

## Q9. Slow `deltaTableUpdate` on Large Fact Table

**Scenario:** A `MERGE` statement that updates 10,000 rows in a 50-billion-row fact table takes 45 minutes. The table is partitioned by `event_date` (daily partitions). The MERGE matches on `transaction_id` (not a partition column). How do you speed it up?

**Answer:**

### Root Cause

The MERGE performs a full-table scan because `transaction_id` is not a partition column. Even though you're updating 10,000 rows across 50 billion, Spark must scan all partitions.

```sql
-- Current (slow): scans all partitions
MERGE INTO fact_table AS f
USING updates AS u
ON f.transaction_id = u.transaction_id   -- Not a partition column
WHEN MATCHED THEN UPDATE SET f.amount = u.amount;
```

### Solution 1: Partition Pruning with Event Date

```sql
-- If the updates include event_date, prune to relevant partitions
MERGE INTO fact_table AS f
USING updates AS u
ON f.transaction_id = u.transaction_id
   AND f.event_date = u.event_date        -- Added partition filter
WHEN MATCHED THEN UPDATE SET f.amount = u.amount;
```

### Solution 2: Z-ORDER by transaction_id

```sql
-- Improve data skipping on transaction_id
OPTIMIZE delta.`/path/to/fact_table`
ZORDER BY (transaction_id);
```

After Z-ORDER, use data skipping:
```sql
-- Expect significant improvement even without event_date filter
MERGE INTO fact_table AS f
USING updates AS u
ON f.transaction_id = u.transaction_id;
```

### Solution 3: Partition by transaction_id Hash

If transaction_id lookups are frequent, repartition:
```sql
-- Redesign partitioning strategy
ALTER TABLE fact_table SET TBLPROPERTIES (
    'delta.partitionBy' = 'transaction_id_hash'
);

-- Plus partition by date for range scans
ALTER TABLE fact_table ADD PARTITION(event_date = '2024-01-15');
```

### Solution 4: Use Delta Lake Indexes (Databricks SQL)

```sql
-- For Databricks SQL warehouses, create an index
CREATE INDEX idx_fact_transaction ON fact_table(transaction_id);
```

### Principal-Level Insight

**The most effective optimization is making the MERGE condition match the table layout.** Before optimizing complex join logic, always check the partition columns and Z-ORDER columns of the table. The fastest MERGE is one that reads only the data it needs.

### Follow-up

**Q:** How do you estimate the cost of a full scan vs. partition/Z-ORDER optimizations?
**A:** Use `EXPLAIN` on the MERGE query and look for:
- `PartitionFilters: []` (no partition pruning)
- Data size read: `"files": "500"` vs. `"files": "3"` (with partition)
- Time spent in Scan Delta: Spark UI > SQL > Scan Duration

---

## Q10. Apache Spark Shuffle Out-of-Memory in a Databricks Job

**Scenario:** A Databricks job joins two large tables (200 GB each) and then aggregates. The job fails with:

```
ERROR Executor: Exception in task 42.0 in stage 8.0 (TID 1234)
java.lang.OutOfMemoryError: Java heap space
  at java.util.HashMap.resize(Unknown Source)
  at org.apache.spark.util.collection.ExternalAppendOnlyMap.insertAll(ExternalAppendOnlyMap.java:162)
```

The cluster has 20 executors with 8 cores and 32 GB each. What causes the shuffle OOM and how do you fix it?

**Answer:**

### Root Cause

The shuffle operation needs to hold intermediate data in memory. When the data for a single task (partition) exceeds available memory, Spark spills to disk, but if the spill is also too large, OOM occurs.

```
For a 200 GB join:
  - Number of shuffle partitions: 200 (default)
  - Data per task: ~1 GB + overhead × 20 executors = 20 GB / executor
  - Available execution memory per executor: ~12 GB (32 × 0.6 × 0.6)
  - Shuffle data per executor: ~20 GB → 8 GB over memory → OOM
```

### Fixes in Order of Preference

**Fix 1: Increase shuffle partitions (most effective)**
```python
spark.conf.set("spark.sql.shuffle.partitions", "800")
# 200 GB / 800 = ~250 MB per task → fits in memory
```

**Fix 2: Increase executor memory**
```python
spark.conf.set("spark.executor.memory", "48g")
spark.conf.set("spark.executor.memoryOverhead", "8g")
```

**Fix 3: Use broadcast hint for smaller table**
```python
from pyspark.sql.functions import broadcast

# If one table is < 2 GB, broadcast it
joined = df_large.join(broadcast(df_small), "key")
```

**Fix 4: Enable shuffle service for better memory management**
```python
spark.conf.set("spark.shuffle.service.enabled", "true")
spark.conf.set("spark.shuffle.service.port", "7337")
```

**Fix 5: Use serialized shuffles**
```python
spark.conf.set("spark.sql.shuffle.partitions", "800")
spark.conf.set("spark.shuffle.compress", "true")
spark.conf.set("spark.shuffle.spill.compress", "true")
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

### Debugging Checklist

```python
# 1. How many shuffle partitions?
print(spark.conf.get("spark.sql.shuffle.partitions"))  # Should be 200+

# 2. How large is each shuffle block?
# In Spark UI: Stages > Shuffle Read Size / Records
# Target: < 200 MB per task

# 3. What's the join order?
# Check EXPLAIN output
explain_string = joined._jdf.queryExecution().logical().toString()
print(explain_string)  # Look for BroadcastHashJoin vs SortMergeJoin
```

### Principal-Level Insight

**Shuffle OOM is almost always solved by increasing shuffle partitions, not increasing memory.** More partitions = less data per task = less memory pressure. Set `spark.sql.adaptive.enabled=true` so Spark can dynamically coalesce partitions after the shuffle.

### Follow-up

**Q:** What's the difference between shuffle OOM and executor OOM?
**A:** Shuffle OOM occurs during the hash/sort phase of a join/aggregate; executor OOM occurs during general processing (loading data, RDD operations). The fix for shuffle OOM is more partitions; the fix for executor OOM is more memory or fewer tasks per executor.

---

## Q11. Unity Catalog Permissions: Deny Access to a Specific Column

**Scenario:** A `finance.orders` table contains a `credit_card_number` column. The marketing team has `SELECT` access on the table but must not see the credit card data. The data governance team demands column-level security. How do you implement this in Unity Catalog?

**Answer:**

### Solution 1: Column-Level Access Control in Unity Catalog

```sql
-- Grant access to all columns except credit_card_number
GRANT SELECT ON finance.orders TO marketing_team_role;

-- Deny access to the sensitive column
DENY SELECT (credit_card_number) ON finance.orders TO marketing_team_role;
```

Users in `marketing_team_role` can query all columns except `credit_card_number`. Tries to `SELECT credit_card_number FROM finance.orders` will fail with a permission error.

### Solution 2: Create a Masked View

```sql
CREATE VIEW finance.orders_masked AS
SELECT
    order_id,
    customer_id,
    order_total,
    order_date,
    CONCAT('****-****-****-', SUBSTRING(credit_card_number, -4)) AS credit_card_number
FROM finance.orders;

GRANT SELECT ON finance.orders_masked TO marketing_team_role;
```

### Solution 3: Dynamic Data Masking (Unity Catalog)

```sql
-- Create a masking policy (Databricks SQL)
CREATE MASKING POLICY mask_credit_card AS (val STRING) RETURNS STRING ->
  CASE
    WHEN is_member('finance_team') THEN val
    ELSE CONCAT('****-****-****-', SUBSTRING(val, -4))
  END;

-- Apply the policy to a column
ALTER TABLE finance.orders
ALTER COLUMN credit_card_number SET MASKING POLICY mask_credit_card;
```

### Principal-Level Insight

**Use column masks for dynamic policies**, not static views. They:
- Scale to thousands of tables
- Work across all warehouses
- Are audited through Unity Catalog lineage
- Can be revoked per user group without view changes

### Follow-up

**Q:** How do you audit who accessed the credit card column?
**A:** Query Unity Catalog system tables:
```sql
SELECT *
FROM system.access.audit
WHERE action = 'accessColumn'
  AND table_name = 'finance.orders'
  AND column_name = 'credit_card_number';
```
---

## Q12. Spark DataFrame Join Produces Wrong Results (Data Skew)

**Scenario:** You join a 1 TB orders table with a 10 GB customers table on `customer_id`. The result is wrong — some customers show orders belonging to other customers. The join key `customer_id` has a skewed distribution: the top 10 customers have 80% of the orders. What happened and how do you fix it?

**Answer:**

### Root Cause

Data skew causes Spark to hash partitions unevenly. For `customer_id`:
- One partition (the "hot" key) contains 80% of the data
- During the shuffle join, Spark puts all "hot" keys into one partition
- Within that partition, keys are mixed incorrectly because the sort/merge step can't handle the massive partition size

```
Without salt (before):
  partition 0: customer_1, customer_1, customer_1, customer_2
  partition 1: customer_3, customer_3, customer_4, ...
  partition 42: customer_1, customer_1, customer_1, ... (80% of data)
```

### Solution 1: Salting the Join Key

```python
from pyspark.sql.functions import col, rand, concat, lit, floor

# Add salt to the skewed table
salted_orders = orders_df.withColumn(
    "salt", floor(rand() * 10)  # 10 salt buckets
).withColumn(
    "salted_key", concat(col("customer_id"), lit("_"), col("salt"))
)

# Add salt and duplicate rows for the small table (broadcast)
salted_customers = customers_df.withColumn(
    "salt_exploded", explode(array([lit(i) for i in range(10)]))
).withColumn(
    "salted_key", concat(col("customer_id"), lit("_"), col("salt_exploded"))
)

# Join on the salted key
result = salted_orders.join(salted_customers, "salted_key")
```

### Solution 2: Use Skew Join Hint (Databricks 12.2+)

```python
# Databricks SQL has built-in skew join handling
from pyspark.sql.functions import skew

# Option A: Use hint
orders_df.alias("o").join(
    customers_df.alias("c"),
    col("o.customer_id") === col("c.customer_id"),
    "skew"
)

# Option B: Use the skew optimization config
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

### Solution 3: Two-Phase Join for Skewed Keys

```python
# Separate skewed vs non-skewed keys
# Assuming you know the top 10 customers
skewed_keys = ["customer_1", "customer_2", "customer_3"]

skewed_orders = orders_df.filter(col("customer_id").isin(skewed_keys))
non_skewed_orders = orders_df.filter(~col("customer_id").isin(skewed_keys))

# Join non-skewed normally
non_skewed_result = non_skewed_orders.join(customers_df, "customer_id")

# Join skewed with salting
salted_skewed = skewed_orders.withColumn("salt", floor(rand() * 10))
skewed_result = salted_skewed.join(
    customers_df.withColumnRenamed("customer_id", "original_customer_id"),
    salted_skewed["customer_id"] === customers_df["customer_id"],
    "left"
)

# Union results
result = non_skewed_result.union(skewed_result)
```

### Principal-Level Insight

**Enable AQE skew join by default on all production clusters.** It handles most skew cases automatically without code changes. Only manually salt when AQE can't detect the skew (e.g., very subtle skew or when you need deterministic results).

### Follow-up

**Q:** How do you detect data skew proactively?
**A:** During query execution:
```python
# Check partition sizes in the Spark UI
# Or programmatically:
df.groupBy("customer_id").count().select(
    max("count").alias("max_count"),
    min("count").alias("min_count"),
    max("count") / min("count").alias("skew_ratio")
).show()
```
A skew ratio > 200 is problematic.

---

## Q13. Databricks Workflow Fails with "Failed to start DLT pipeline" Intermittently

**Scenario:** A Databricks Workflow that triggers a DLT pipeline fails ~10% of the time with:

```
Failed to start DLT pipeline. 
No healthy cluster available to run the pipeline.
```

The DLT pipeline has one cluster definition with `min_workers: 1, max_workers: 5`, auto-scaling enabled. Other jobs on the same cluster run fine. What's wrong and how do you fix it?

**Answer:**

### Root Cause

The DLT pipeline's cluster is configured with auto-scaling starting at 1 worker. When multiple DLT pipelines with overlapping schedules compete for cluster resources, a pipeline may fail to start because the cluster can't spin up a healthy worker fast enough.

```
Scenario:
  10:00 AM: DLT pipeline A starts (1 worker)
  10:01 AM: DLT pipeline B starts (needs 1 worker)
  10:02 AM: Both are running → cluster scaling triggers
  10:03 AM: Cluster has 2 workers, but spinner time + health check
            sometimes fails before the pipeline timeout
```

### Immediate Fix

```python
# Option 1: Increase min workers to guarantee capacity
@dlt.pipeline(
    cluster=[{
        "min_workers": 2,
        "max_workers": 10,
        "autoscale": True
    }]
)
def pipeline_A():
    pass

# Option 2: Enable pipeline-level cluster (isolated per pipeline)
@dlt.pipeline(
    cluster=[{
        "min_workers": 1,
        "max_workers": 5,
        "autoscale": True,
        "policy_id": "pipeline_cluster_policy"  # Dedicated policy
    }]
)
```

### Root Fix: Cluster Pool Configuration

```python
# In the Databricks cluster config:
# Cluster mode: Multi-node
# Min workers: 1, Max workers: 10
# Policy: DLT Pipeline Cluster Policy
# Use cluster pool for faster startup:
# Pool: "dlt_pipeline_pool" (pre-warmed instances)
```

### Clean Separation: Pipeline vs. Job Clusters

```python
# Use separate clusters for DLT pipelines vs. Notebook jobs
dlt_cluster = {
    "num_workers": 2,
    "spark_version": "13.3.x-scala2.12",
    "node_type_id": "i3.xlarge",
    "autoscale": {"min_workers": 2, "max_workers": 10},
    "policy_id": "dlt_pipeline_policy"
}

job_cluster = {
    "num_workers": 5,
    "spark_version": "13.3.x-scala2.12",
    "node_type_id": "r5.2xlarge",
    "autoscale": {"min_workers": 2, "max_workers": 20}
}
```

### Principal-Level Insight

**Never share clusters between DLT and non-DLT workloads.** DLT has specific cluster configuration requirements (column mapping support, constraints on spark config). Keep DLT clusters isolated and pre-warmed with a pool.

### Follow-up

**Q:** How do you monitor DLT pipeline health?
**A:** Use the DLT pipeline system tables:
```sql
SELECT
    pipeline_id,
    pipeline_name,
    state,
    start_time,
    duration,
    expected_records,
    actual_records
FROM system.dlt.pipeline_runs
WHERE
    state IN ('FAILED', 'CANCELLED')
    AND start_time > current_timestamp() - INTERVAL 7 DAYS;
```

---

## Q14. Delta Table Write Performance Degrades Over Time

**Scenario:** A high-frequency streaming job writes to a Delta table at 1000 records/second. For the first month, write latency is < 1 second. After 3 months, write latency is > 30 seconds. The table has 300 partitions and ~500 GB of data. What's the bottleneck and how do you fix it?

**Answer:**

### Root Cause

As a Delta table grows, the **transaction log** grows linearly. Each write scans the log for conflicts, determines the next version, and writes a new log entry. With 300 partitions and millions of small files, log operations dominate.

```
Month 1:
  - 1000 files, 10 GB
  - Log size: ~5 MB
  - Write: scan log in 10 ms, write in 20 ms

Month 3:
  - 50,000 files, 500 GB
  - Log size: ~250 MB
  - Write: scan log in 5 seconds, write in 25 seconds
```

### Fix 1: Optimize the Table (Reduce File Count)

```python
# Step 1: Run OPTIMIZE to compact files
spark.sql(f"OPTIMIZE delta.`/path/to/events` ZORDER BY (event_timestamp)")

# Step 2: Set bin-packing target file size
spark.conf.set("spark.databricks.delta.optimizeWrite.numShuffleBlocks", "5000000")
spark.conf.set("spark.databricks.delta.optimizeWrite.binSize", "512")
```

### Fix 2: Enable Delta Log Compaction

```python
# Delta automatically compacts the log when it exceeds the threshold
spark.sql("""
    ALTER TABLE events SET TBLPROPERTIES (
        'delta.checkpoint.writeStatsAsJson' = 'false',
        'delta.checkpoint.writeStatsAsStruct' = 'true'
    )
""")
```

### Fix 3: Reduce Partition Count

```python
# If you have 300 partitions, you likely have very small partitions
# Consider a coarser partition scheme:
# Before: PARTITIONED BY (event_date, event_hour) → 300 partitions/year
# After: PARTITIONED BY (event_date) → 365 partitions/year

spark.sql("ALTER TABLE events SET TBLPROPERTIES (delta.partitionBy = 'event_date')")
```

### Fix 4: Use Low-Shuffle Merge

```python
# Enable low-shuffle merge for faster updates
spark.sql("SET spark.databricks.delta.merge.enableLowShuffle = true")
```

### Principal-Level Insight

**Implement a table compaction and log cleanup schedule.** Run OPTIMIZE + VACUUM on high-write tables weekly. Monitor the Delta log size and file count metrics:

```python
# Monitor log size
log_path = "/path/to/events/_delta_log"
log_files = dbutils.fs.ls(log_path)
log_size = sum(f.size for f in log_files) / (1024 * 1024)  # MB
print(f"Log size: {log_size:.2f} MB")

# Monitor file count
table_data = spark.sql(f"DESCRIBE DETAIL delta.`/path/to/events`")
print(f"File count: {table_data.select('numFiles').collect()[0][0]}")
```

### Follow-up

**Q:** How do you detect that the log is the bottleneck?
**A:** In the Spark UI, look at the "Delta log" stage — if it takes > 10% of the total write time, the log is a bottleneck. Also check: `DESCRIBE DETAIL` output for `numFiles`; if > 10,000, you need compaction.

---

## Q15. External Hive Metastore Not Syncing with Unity Catalog

**Scenario:** You have an existing Hive Metastore (HMS) with 500 tables used by legacy ETL pipelines. You've set up Databricks Unity Catalog with a new Metastore. Some tables created via HMS are not visible in Unity Catalog. Queries using the HMS tables work fine, but queries in Unity Catalog fail with `TABLE_OR_VIEW_NOT_FOUND`. What's the configuration issue?

**Answer:**

### Root Cause

Unity Catalog does **not automatically sync** with an external Hive Metastore. They are independent metastores. A table created in HMS is not visible in Unity Catalog unless explicitly registered or a catalog is configured to mirror the HMS.

```
Hive Metastore:
  - gold.daily_summary (created 2023-01-15)
  - silver.events (created 2023-06-01)

Unity Catalog:
  - catalog_name: main (empty)
  - catalog_name: hms_metastore (configured to mirror HMS → now visible)
```

### Solution: Create a Unity Catalog Mirror of the Hive Metastore

```python
# Step 1: Create a Unity Catalog that points to the Hive Metastore
# In the Databricks UI or Terraform:

resource "databricks_catalog" "hms_metastore" {
  name           = "hms_metastore"
  metastore_id   = databricks_metastore.this.id
  comment        = "Mirror of the Hive Metastore for legacy tables"
  connection_name = "hms_connection"
  options = {
    "sync" = "true"
  }
}

# Step 2: Register the connection
resource "databricks_connection" "hms" {
  name            = "hms_connection"
  connection_type = "HIVE_METASTORE"
  options = {
    "metastoreUri" = "thrift://hive-metastore:9083"
    "principal"    = "hive/databricks@DOMAIN"
    "keytab"       = "/path/to/keytab"
  }
}
```

### Alternative: Use a Foreign Catalog

```python
# Create a foreign catalog pointing to the HMS
CREATE FOREIGN CATALOG hms_metastore
USING CONNECTION hms_connection
OPTIONS (sync = 'true');
```

### For Immediate Visibility: Manually Register Tables

```python
# Register a specific table from HMS in Unity Catalog
CREATE TABLE uc_catalog.schema.events
USING DELTA
LOCATION 's3://bucket/events'
AS
SELECT * FROM hms_metastore.default.events;
```

### Principal-Level Insight

**Plan for a phased migration from HMS to Unity Catalog:**
1. Create a mirror catalog (no downtime)
2. Gradually update pipelines to use Unity Catalog references
3. Drop HMS references after all pipelines are migrated
4. Run a validation pass: `SELECT COUNT(*) FROM ...` across both metastores

### Follow-up

**Q:** What happens if you have both HMS and UC references to the same Delta table?
**A:** Both will work correctly because metadata is stored in the Delta log. However, table properties and permissions are managed independently. Keep only one metastore active for each table to avoid confusion.
**A:** Both work correctly since Delta metadata lives in `_delta_log`. However, permissions, grants and table properties are managed independently per metastore. Keep only one active metastore reference per table to avoid confusion.

---

## Q16. Spark SQL Warehouse Query Returns Stale Data

**Scenario:** A business user runs `SELECT * FROM gold.daily_metrics` in a Databricks SQL warehouse and gets data from yesterday. You know the DLT pipeline refreshed the table at 06:00 AM with today's data. A Spark notebook query on the same table returns the correct data. What's happening?

**Answer:**

### Root Cause

Databricks SQL warehouses do **not** use the Delta transaction log for read consistency by default. Instead, they use a **materialized view cache** (snapshot isolation) that may not have refreshed yet.

```
Spark Notebook (direct Delta read):
  - Reads latest Delta log version
  - Sees today's data ✓

Databricks SQL Warehouse:
  - Uses warehouse query result cache (default 3 hours)
  - Sees yesterday's cached snapshot ✗
```

### Fix 1: Invalidate Warehouse Cache

```sql
-- In the Databricks SQL warehouse:
REFRESH MATERIALIZED VIEW IF EXISTS gold_daily_metrics_cache;

-- Or clear the entire query result cache:
ALTER SYSTEM SET spark.databricks.sql.resultCache.enabled = false;
```

### Fix 2: Disable Result Cache for Time-Sensitive Queries

```sql
-- Disable result cache at the session level
SET spark.databricks.sql.resultCache.enabled = false;

-- Or use a volatile table hint
SELECT /*+ NOCACHE */ * FROM gold.daily_metrics;
```

### Fix 3: Use Materialized Views for Controlled Caching

```sql
-- Create a materialized view with explicit refresh logic
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_metrics_mv
AS
SELECT * FROM gold.daily_metrics
WHERE date >= CURRENT_DATE - 7;

-- Schedule a refresh after the DLT pipeline completes
-- Use Databricks Workflows to chain DLT → SQL Warehouse refresh
```

### Fix 4: Ensure DLT Pipeline Finishes Before Warehouse Refreshes

```python
# In your Databricks Workflow (DAG):
# Task 1: DLT Pipeline (run mode: trigger_once)
# Task 2: Run After: Task 1
#          SQL: REFRESH MATERIALIZED VIEW gold_daily_metrics_mv;
# Task 3: Run After: Task 2
#          SQL: SET spark.databricks.sql.resultCache.invalidateAll = true;
```

### Principal-Level Insight

**Databricks SQL warehouses are not real-time.** If business users need real-time data:
1. Use **Photon** engine warehouses (lower latency)
2. Disable result cache for time-critical queries
3. Materialize views with explicit refresh triggers after DLT pipelines complete
4. Use **Streaming Tables** in DLT for incremental materialization (near real-time)

### Follow-up

**Q:** How do you verify that the warehouse is reading fresh data?
**A:** Compare record counts between the Spark notebook and the warehouse:
```sql
-- In the SQL warehouse
SELECT COUNT(*), MAX(updated_at) FROM gold.daily_metrics;
```
Compare the result with a Spark notebook query:
```python
spark.sql("SELECT COUNT(*), MAX(updated_at) FROM gold.daily_metrics").show()
```

---

## Q17. Spark ML Pipeline Fails with "Feature Mismatch" After Model Deployment

**Scenario:** You trained a Spark ML pipeline model on historical data using a pipeline with feature transformations. The model performs well in training. You deployed it to a production streaming pipeline that serves predictions in real-time. After a week, you notice the predictions are wildly off — the RMSE increased from 0.05 to 0.5. What went wrong?

**Answer:**

### Root Cause

Feature mismatch between training and serving — the transformations applied during training are not exactly the same as the transformations applied during inference.

```
Training:
  - Data: Historical batch (static schema)
  - Pipeline: FeatureAssembler → StringIndexer → OneHotEncoder → StandardScaler → LogisticRegression
  - Input features: raw columns with known schema
  - Output features: normalized, encoded vectors

Serving (Production):
  - Data: Streaming (dynamic schema, missing columns, different types)
  - Pipeline: Same pipeline but with streaming data
  - Problem: A column that existed during training is now missing or has a different type
```

### Fix 1: Use a Feature Store for Consistent Transformations

```python
from spark_ml import FeatureStore

# Register the feature pipeline
feature_store = FeatureStore()
feature_store.register_pipeline(
    "fraud_detection_features",
    feature_pipeline=pipeline,
    feature_columns=["user_id", "amount", "merchant_id", "hour_of_day"],
    label_column="is_fraud"
)

# Get consistent features for both training and serving
training_features = feature_store.get_training_data(
    pipeline_name="fraud_detection_features",
    data="historical_events"
)

# In the streaming pipeline:
serving_features = feature_store.get_features(
    pipeline_name="fraud_detection_features",
    data=streaming_events
)
```

### Fix 2: Validate Schema at Pipeline Entry

```python
from pyspark.sql.types import (
    StructType, StructField, IntegerType, DoubleType, StringType
)

expected_schema = StructType([
    StructField("user_id", StringType(), False),
    StructField("amount", DoubleType(), False),
    StructField("merchant_id", StringType(), False),
    StructField("hour_of_day", IntegerType(), False),
    StructField("is_fraud", IntegerType(), False)
])

# Validate the streaming schema matches training
schema_diff = streaming_df.schema.except(expected_schema)
if schema_diff:
    raise ValueError(f"Schema mismatch: {schema_diff}")
```

### Fix 3: Save and Load the Complete Pipeline

```python
# Save the entire pipeline (not just the model)
pipelineModel.write().overwrite().save("s3://models/fraud_detection_pipeline")

# Load and use the complete pipeline
from pyspark.ml import PipelineModel
loaded_pipeline = PipelineModel.load("s3://models/fraud_detection_pipeline")

# Use the loaded pipeline for serving
predictions = loaded_pipeline.transform(streaming_df)
```

### Fix 4: Add Monitoring for Feature Drift

```python
# Monitor feature statistics periodically
from pyspark.sql.functions import col, stddev, mean

# Compute feature statistics
feature_stats = predictions.select(
    "amount", "hour_of_day"
).describe()

# Check for drift
current_mean = feature_stats.filter("summary = 'mean'").head()
if current_mean[1] > 100.0:
    raise ValueError("Feature drift detected in 'amount'")
```

### Principal-Level Insight

**Train/serving skew is the #1 cause of model degradation in production.** Always use:
1. **Pipeline-based feature engineering** — encode the entire feature pipeline (not just the model)
2. **Schema validation** — validate streaming schema matches training schema at pipeline entry
3. **Feature store** — manage features separately from models
4. **Monitoring** — track feature statistics, prediction distribution, and model performance in production
5. **Re-training** — schedule periodic model re-training with recent data

### Follow-up

**Q:** How do you version and rollback ML models in production?
**A:** Use MLflow with Unity Catalog:
```python
import mlflow
from mlflow import MlflowClient

# Register model
model_uri = f"runs:/{run_id}/model"
mlflow.register_model(model_uri, "fraud_detection")

# Set model version for production
client = MlflowClient()
client.set_model_version_tag("fraud_detection", "v2", "environment", "production")

# Rollback to previous version
client.set_model_version_tag("fraud_detection", "v1", "environment", "production")
```

---

## Q18. Delta Sharing Fails with "ACCESS_DENIED" for External Recipients

**Scenario:** You're using Delta Sharing to share a dataset with a partner company. You've created the sharing configuration and sent the `.dsc` file. The partner gets "ACCESS_DENIED" when they try to query the shared data. The sharing is set up in Unity Catalog with a public sharing token. What's wrong?

**Answer:**

### Root Cause

Delta Sharing access is controlled at the **catalog/schema/table** level in Unity Catalog. The sharing token is valid but the recipient hasn't been granted access to the specific tables.

```
Your setup:
  - Sharing token: ✓ (created)
  - Recipient: ✗ (not granted access)
  - Table: ✗ (not granted access)
```

### Fix 1: Grant Access to the Specific Tables

```sql
-- In Unity Catalog:

-- Grant the recipient access to the sharing token
GRANT SHARE ON CATALOG prod_data TO RECIPIENT partner_company;

-- Grant access to the specific tables
GRANT SELECT ON CATALOG prod_data.gold_events TO RECIPIENT partner_company;
GRANT SELECT ON CATALOG prod_data.gold_orders TO RECIPIENT partner_company;
```

### Fix 2: Create the Recipient in Unity Catalog

```sql
-- Create the recipient in Unity Catalog
CREATE RECIPIENT partner_company
COMMENT = 'External partner for data sharing';

-- Add the recipient to the share
GRANT SHARE ON CATALOG prod_data TO RECIPIENT partner_company;

-- Grant table access
GRANT SELECT ON CATALOG prod_data.gold_events TO RECIPIENT partner_company;
GRANT SELECT ON CATALOG prod_data.gold_orders TO RECIPIENT partner_company;
```

### Fix 3: Verify the `.dsc` File Configuration

```yaml
# The .dsc file should contain:
# endpoint: https://databricks.com/sharing/your-account-id
# protocol: delta-sharing
# version: 1
# token: valid-token
# recipient: partner_company
```

### Fix 4: Check the Sharing Token Expiry

```sql
-- Check if the sharing token has expired
SELECT * FROM system.sharing.sharing_tokens
WHERE token_id = 'your-token-id';

-- Re-issue if expired
REVOKE SHARE ON CATALOG prod_data FROM RECIPIENT partner_company;
GRANT SHARE ON CATALOG prod_data TO RECIPIENT partner_company;
```

### Principal-Level Insight

**Use Delta Sharing selectively.** Only share the minimum data needed:
1. Share specific tables, not entire catalogs
2. Use column-level masking if needed:
   ```sql
   ALTER TABLE gold_events
   SET TBLPROPERTIES (delta.columnMask.defaultMask = 'md5');
   ```
3. Monitor access:
   ```sql
   SELECT * FROM system.sharing.access_audit
   WHERE table_name = 'gold_events';
   ```
4. Consider **external tables** for simpler sharing:
   ```sql
   CREATE EXTERNAL TABLE shared_gold_events
   USING DELTA
   LOCATION 's3://shared-bucket/gold_events';
   ```

### Follow-up

**Q:** Can you share data from an external Delta table (not Unity Catalog)?
**A:** Yes, but you need to register the external table in Unity Catalog first:
```sql
CREATE TABLE uc_catalog.gold_events
USING DELTA
LOCATION 's3://bucket/gold_events'
TBLPROPERTIES (delta.logRetentionDuration = '30 days');
```
Then share it as you would any Unity Catalog table.

---

## Q19. Streaming Job Checkpoint Recovery Fails After Schema Change

**Scenario:** A Structured Streaming job reads from Kafka, applies transformations, and writes to a Delta table. After 6 months of stable operation, you add a new column to the Kafka schema (optional, nullable). The next time the streaming job restarts (e.g., after a cluster resize), it fails to recover from checkpoint with:

```
java.io.InvalidClassException: 
com.databricks.spark.streaming.KafkaEvent: 
local class incompatible: stream classdesc serialVersionUID
```

The checkpoint location is on cloud storage (S3/ADLS). What's the right approach?

**Answer:**

### Root Cause

Spark's checkpoint stores serialized RDDs, shuffle data and state. A schema change to the input (Kafka) causes serialization incompatibility between the old checkpoint and the new code/schema.

```
Old checkpoint (serialized with old schema):
  - EventRecord { id, timestamp, event_type, amount }
  
New code (new schema):
  - EventRecord { id, timestamp, event_type, amount, metadata }
```

### Fix 1: Drop and Restart the Checkpoint

```python
# In your streaming code:
checkpoint_path = "s3://bucket/checkpoints/event_processing"

# Option 1: Drop the checkpoint
dbutils.fs.rm(checkpoint_path, recurse=True)

# Restart the streaming job
streaming_df.writeStream \
    .outputMode("append") \
    .option("checkpointLocation", checkpoint_path) \
    .trigger(processingTime="60 seconds") \
    .start()
```

### Fix 2: Handle Schema Evolution with a Tolerant Parser

```python
# Use a schema-agnostic reader
from pyspark.sql.types import StringType

# Read as JSON (tolerates schema changes)
streaming_df = spark \
    .readStream \
    .format("cloudFiles") \
    .option("cloudFiles.format", "json") \
    .option("multiLine", "true") \
    .load("s3://bucket/events")

# Parse the schema dynamically
parsed_df = streaming_df.select(
    col("id").cast("string"),
    col("timestamp").cast("timestamp"),
    col("event_type").cast("string"),
    col("amount").cast("double"),
    coalesce(col("metadata").cast("string"), lit("")).alias("metadata")
)
```

### Fix 3: Use Schema Evolution in Delta

```python
# Enable schema evolution for Delta writes
spark.conf.set("spark.databricks.delta.schema.autoMerge.enabled", "true")

streaming_df.writeStream \
    .format("delta") \
    .option("mergeSchema", "true") \
    .option("checkpointLocation", checkpoint_path) \
    .trigger(processingTime="60 seconds") \
    .start()
```

### Fix 4: Version the Checkpoint Path

```python
# Append a version suffix to the checkpoint path
import datetime
checkpoint_path = f"s3://bucket/checkpoints/event_processing/v2/{datetime.datetime.now().isoformat()}"

# Use a new checkpoint path for the new schema
streaming_df.writeStream \
    .option("checkpointLocation", checkpoint_path) \
    .start()
```

### Principal-Level Insight

**Never assume streaming checkpoints are portable across code/schema changes.** Plan for checkpoint incompatibilities:
1. Use **schema-agnostic reading** (JSON, Avro with schema registry)
2. **Version checkpoint paths** for schema changes
3. Use **stateful streaming with explicit state management** (Watermarking, StateStore)
4. Monitor checkpoint health:
   ```sql
   SELECT * FROM system.tables WHERE name LIKE '%checkpoint%';
   ```
5. Design for **graceful degradation** — if a schema change breaks the checkpoint, drop it and restart (with a small data gap).

### Follow-up

**Q:** How do you handle checkpoint cleanup to prevent storage costs?
**A:** Enable auto-retention and periodically clean old checkpoints:
```python
# Set checkpoint retention (e.g., 7 days)
spark.conf.set("spark.databricks.delta.checkpointRetentionHours", "168")

# Or clean manually:
dbutils.fs.rm("s3://bucket/checkpoints/event_processing", recurse=True)
```
Note: **Never clean an active checkpoint** — only clean when the streaming job is stopped.

---

## Q20. Cross-Workspace DLT Pipeline Deployment Fails After Promotion

**Scenario:** You have a DLT pipeline that runs in the `dev` workspace. It works correctly. You promote it to `staging` using the same notebook code. In `staging`, the pipeline fails with `TABLE_OR_VIEW_NOT_FOUND` for tables that exist in `dev` but not in `staging`. The DLT pipeline references tables from other schemas in the same workspace. What's the right deployment strategy?

**Answer:**

### Root Cause

The DLT pipeline references tables from other schemas (e.g., `dev.events`, `dev.orders`) using fully qualified names. In the `staging` workspace, those schemas exist but with different table names or the schemas don't exist at all.

```
Dev workspace:
  - Schema: dev_events (exists)
  - Table: dev_events.raw_events (exists)
  
Staging workspace:
  - Schema: staging_events (different name)
  - Table: staging_events.raw_events (not created yet)
```

### Fix 1: Use Environment Variables for Table References

```python
# Use environment variables for schema names
import os

schema_name = os.environ.get("SCHEMA_NAME", "dev_events")
events_table = f"{schema_name}.raw_events"
orders_table = f"{schema_name}.raw_orders"

# In DLT:
@dlt.table(name="silver_events")
def silver_events():
    return spark.sql(f"SELECT * FROM {events_table}")
```

### Fix 2: Use Unity Catalog for Cross-Workspace References

```python
# Use Unity Catalog to reference tables across workspaces
# In dev workspace:
# CREATE TABLE uc_catalog.dev.raw_events ...

# In staging workspace (same metastore):
# CREATE TABLE uc_catalog.staging.raw_events ...

@dlt.table(name="silver_events")
def silver_events():
    return spark.sql("SELECT * FROM uc_catalog.staging.raw_events")
```

### Fix 3: Create a Workspace Configuration File

```json
// .databricks/workspaces/dev.json
{
  "workspace": "dev",
  "schema": "dev_events",
  "tables": {
    "raw_events": "dev_events.raw_events",
    "raw_orders": "dev_events.raw_orders"
  }
}

// .databricks/workspaces/staging.json
{
  "workspace": "staging",
  "schema": "staging_events",
  "tables": {
    "raw_events": "staging_events.raw_events",
    "raw_orders": "staging_events.raw_orders"
  }
}
```

### Fix 4: Use Terraform for Infrastructure as Code

```hcl
resource "databricks_notebook" "dlt_pipeline" {
  path     = "/Users/dlt_pipelines/${var.workspace_name}"
  language = "PYTHON"
  source   = "${path.module}/dlt_pipeline.py"
  tags     = ["workspace=${var.workspace_name}"]
}

resource "databricks_job" "dlt_job" {
  name = "${var.workspace_name}_dlt_pipeline"
  
  new_cluster {
    num_workers    = 5
    spark_version  = databricks_current_spark_version.latest
    node_type_id   = "i3.xlarge"
  }
  
  spark_jar_task {
    jar_params = [var.schema_name]
  }
}

variable "schema_name" {
  default = "dev_events"
}
```

### Principal-Level Insight

**Use Unity Catalog for all cross-workspace references.** It provides a single, consistent table reference regardless of the workspace:
1. All workspaces connect to the same Unity Catalog metastore
2. Table names are consistent across workspaces (use schema names to differentiate)
3. Permissions are managed centrally in Unity Catalog
4. Use Terraform to provision workspace-specific resources
5. Implement **workspace promotion pipelines** — automated promotion from dev → staging → production with validation at each step.

### Follow-up

**Q:** How do you handle data differences between workspaces (e.g., dev has 1K rows, prod has 1M rows)?
**A:** Use environment-specific data sources:
```python
# In the DLT pipeline:
workspace = os.environ.get("WORKSPACE", "dev")

if workspace == "dev":
    spark.conf.set("databricks.cluster.mode", "dev")
    spark.conf.set("databricks.data.source", "dev_data")
else:
    spark.conf.set("databricks.cluster.mode", "production")
    spark.conf.set("databricks.data.source", "prod_data")
```
Use the appropriate data source for each workspace, and validate the data counts after promotion.
---

## Q21. Spark Cluster OOM on Shuffled Join (Memory Allocation)

**Scenario:** You run a join between a 500 GB Delta table and a 10 GB Delta table on the same Spark cluster (10 workers, each with 32 GB RAM). The job fails with:

```
java.lang.OutOfMemoryError: Java heap space
```

The join key is a 32-byte string column with high cardinality (millions of unique values). What's the right fix?

**Answer:**

### Root Cause

A 500 GB table joined with a 10 GB table on a 10-node cluster with 32 GB RAM per node means each node tries to hold ~50 GB of data in memory — far exceeding the available RAM.

```
Join key: event_id (high cardinality, 32 bytes)
Join type: Inner join
```

### Fix 1: Broadcast the Small Table

```python
from pyspark.sql.functions import broadcast

# The 10 GB table is small enough to broadcast (fits in each node's memory)
# Each node gets ~1 GB of the broadcast hash table
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "10737418240")  # 10 GB

# Explicitly broadcast
spark.sql("""
    SELECT *
    FROM delta.`/data/events` e
    INNER JOIN broadcast(delta.`/data/events_metadata`) m
    ON e.event_id = m.event_id
""")
```

### Fix 2: Increase Executor Memory

```python
# In your cluster configuration:
spark.conf.set("spark.executor.memory", "28GB")
spark.conf.set("spark.memory.fraction", "0.8")
spark.conf.set("spark.memory.storageFraction", "0.3")
spark.conf.set("spark.sql.shuffle.partitions", "2000")
```

### Fix 3: Filter Before the Join

```python
# Pre-filter to reduce the data size before the join
from pyspark.sql.functions import col, date_add

events_filtered = spark.table("delta.events").where(
    col("event_date") > date_add(current_date(), -30)
)

# Now the join is on a much smaller dataset
events_filtered.join(spark.table("delta.events_metadata"), "event_id")
```

### Fix 4: Use Adaptive Query Execution (AQE)

```python
# Enable AQE to optimize the join strategy
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.autoBroadcastJoinThreshold", "10737418240")  # 10 GB
```

### Fix 5: Repartition Before the Join

```python
# Repartition to reduce the shuffle data size
spark.conf.set("spark.sql.shuffle.partitions", "5000")

# Or use a coarser partitioning scheme
spark.conf.set("spark.sql.adaptive.coalescePartitions.minPartitionSize", "500MB")
```

### Principal-Level Insight

**Always prefer broadcast joins for small tables.** The threshold is configurable:
```python
# Increase the auto-broadcast threshold for small tables
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "10GB")
```
For larger tables, use **AQE skew join** to handle data skew, and **repartition** to control shuffle size. Never increase cluster memory without first optimizing the query logic.

### Follow-up

**Q:** How do you estimate the memory needed for a broadcast join?
**A:** The broadcast table must fit in memory on *every* executor. With 10 executors and 10 GB each = 100 GB total. The broadcast hash table takes roughly 2x the data size. If the small table is > 2x the executor memory, broadcast is not viable.

---

## Q22. Delta Table Compaction Causes Concurrent Write Failures

**Scenario:** You have a high-throughput Delta table that receives ~10,000 writes/hour from multiple streaming jobs. You run `OPTIMIZE` regularly to compact files. The `OPTIMIZE` job fails with:

```
DeltaConcurrentModificationException: 
  Version 105232302 is already committed.
```

The streaming jobs are using `append` mode. What's the correct approach to handle compaction with concurrent writes?

**Answer:**

### Root Cause

The `OPTIMIZE` operation is a **major compaction** that rewrites data files and updates the transaction log. Concurrent writers update the log simultaneously, causing version conflicts.

```
Stream Job 1: Appends 100 rows → Version 100
Stream Job 2: Appends 100 rows → Version 101
OPTIMIZE: Reads Version 100 → tries to write Version 102
Stream Job 3: Appends 100 rows → Version 102 → Conflict!
```

### Fix 1: Optimize Before Writes (Not During)

```python
# Schedule OPTIMIZE to run during low-traffic periods
# In your Databricks workflow:
# Task 1: Run OPTIMIZE at 2 AM (low traffic)
# Task 2: Run streaming jobs at 3 AM (after OPTIMIZE completes)
```

### Fix 2: Use `VACUUM` with a Short Retention Window

```python
# Use VACUUM with a short retention window (7 days by default)
spark.sql("VACUUM delta.`/data/events` RETAIN 7 DAYS")
```

### Fix 3: Separate Read/Write Partitions

```python
# Use different partitions for reads vs. writes
# Write to a "staging" partition, then merge to the main table
spark.sql(f"""
    MERGE INTO delta.`/data/events` e
    USING delta.`/data/events_staging` s
    ON e.event_id = s.event_id
    WHEN NOT MATCHED THEN INSERT *
""")
```

### Fix 4: Use `OPTIMIZE` with `VACUUM` in a Single Transaction

```python
# Run OPTIMIZE and VACUUM in a single transaction
# This prevents concurrent modifications
spark.sql("OPTIMIZE delta.`/data/events` VACUUM 7 DAYS")
```

### Fix 5: Enable Delta Auto Optimize

```python
# Enable auto-optimization in Databricks
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.databricks.delta.autoOptimize.optimizeWrite", "true")
spark.conf.set("spark.databricks.delta.autoOptimize.autoCompact", "true")
```

### Principal-Level Insight

**Never run OPTIMIZE on a table that has active concurrent writers.** Instead:
1. Use **auto-optimize** on streaming writes (Delta Auto Compact)
2. Run `OPTIMIZE` + `VACUUM` during low-traffic windows (nightly)
3. Use **partition pruning** to limit the scope of OPTIMIZE
4. Monitor concurrent writes:
   ```sql
   SELECT * FROM system.tables WHERE name = 'events'
   ```

### Follow-up

**Q:** What's the difference between `OPTIMIZE` and `VACUUM`?
**A:** 
- `OPTIMIZE` rewrites data files to improve query performance (reduces file count)
- `VACUUM` removes old, unreferenced files from cloud storage (frees disk space)
- Use `OPTIMIZE` for performance, `VACUUM` for storage. They should be used together but not concurrently with active writers.

---

## Q23. Photon Acceleration Causes Incorrect Results in a Complex Query

**Scenario:** You enabled Photon engine on your Databricks SQL warehouse. A complex query with window functions, CTEs and sub-queries that works correctly with the Apache Spark engine now returns wrong results with Photon. The query has 5 CTEs, a window function with `PARTITION BY ... ORDER BY ...`, and a nested sub-query with `LATERAL VIEW`. What's happening and how do you fix it?

**Answer:**

### Root Cause

Photon is a new execution engine that reorders operations for performance. Some complex query patterns (especially window functions with lateral joins) are not yet fully supported or have subtle differences in execution order vs. Spark.

```
Photon execution order (optimized):
  1. Scan tables
  2. Apply filters early (pushed down)
  3. Evaluate window functions (different order than Spark)
  4. Join with lateral view

Spark execution order (logical):
  1. Scan tables
  2. Join with lateral view
  3. Apply filters
  4. Evaluate window functions
```

### Fix 1: Force Spark Engine for Problematic Queries

```sql
-- Disable Photon for specific queries
SET spark.databricks.photon.usePhotonExecution = false;

-- Or use a different execution engine for the query
SET spark.databricks.sql.optimizer.forcePhoton = false;
```

### Fix 2: Rewrite the Query to Use Photon-Friendly Patterns

```sql
-- Instead of complex CTEs with lateral joins, use a simpler structure
WITH cte1 AS (
    SELECT * FROM events
    WHERE event_date >= '2023-01-01'
),
cte2 AS (
    SELECT *, 
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_timestamp) as rn
    FROM cte1
)
SELECT * FROM cte2 WHERE rn = 1;
```

### Fix 3: Use the Photon Function Support Checker

```sql
-- Check which functions are supported by Photon
SELECT * FROM system.photon_supported_functions
WHERE function_name LIKE '%window%';
```

### Fix 4: Validate Results with Both Engines

```sql
-- Run the query with both engines and compare results
-- With Spark:
SET spark.databricks.photon.usePhotonExecution = false;
SELECT * FROM complex_query;

-- With Photon:
SET spark.databricks.photon.usePhotonExecution = true;
SELECT * FROM complex_query;
```

### Principal-Level Insight

**Photon is not a drop-in replacement for all queries.** Validate critical queries with both engines before enabling Photon in production. Keep a **query compatibility matrix** and run validation tests after every major Photon update.

### Follow-up

**Q:** How do you know which queries benefit from Photon?
**A:** Compare execution time and resource usage:
```sql
-- In the query profile:
-- Photon: 15 seconds, 10 GB memory
-- Spark: 45 seconds, 20 GB memory
```
Photon typically gives 5-10x speedups for analytical queries (aggregations, window functions). Use the Spark UI to compare execution times.

---

## Q24. Airflow DAG That Triggers Databricks Jobs Fails Intermittently

**Scenario:** You have an Airflow DAG that triggers a Databricks Job using the `DataPipelineJobOperator`. The job runs fine when triggered manually from the Databricks UI, but fails ~20% of the time when triggered by Airflow with:

```
airflow.exceptions.AirflowException: 
Task failed: databricks_job_task_1
Error: HTTP 502: Bad Gateway
```

The Databricks job takes 2 hours to complete. What's the issue and how do you fix it?

**Answer:**

### Root Cause

The `DataPipelineJobOperator` uses the Databricks Jobs API. A 502 Bad Gateway error typically means the Databricks API endpoint timed out while waiting for the job to complete. The operator is configured to **poll for job completion** but the timeout is too short.

```python
# Current configuration:
task = DataPipelineJobOperator(
    task_id='databricks_job_task_1',
    databricks_conn_id='databricks_prod',
    pipeline_id='123456789',
    params={'source': 'production', 'batch_id': '2023-10-15'},
    retries=3,
    retry_delay=timedelta(minutes=5),
    # No timeout configuration → defaults to 1 hour
)
```

### Fix 1: Increase the API Timeout

```python
from airflow.providers.databricks.operators.databricks import DataPipelineJobOperator
from datetime import timedelta

task = DataPipelineJobOperator(
    task_id='databricks_job_task_1',
    databricks_conn_id='databricks_prod',
    pipeline_id='123456789',
    params={'source': 'production', 'batch_id': '2023-10-15'},
    retries=3,
    retry_delay=timedelta(minutes=5),
    # Increase the timeout for long-running jobs
    timeout=timedelta(hours=4)
)
```

### Fix 2: Use the Databricks Jobs API with Run ID Polling

```python
from airflow.providers.databricks.triggers.databricks import DatabricksJobsTrigger
from airflow.triggers import BaseTrigger

# Use the Databricks run ID to poll for completion
class DatabricksJobTrigger(BaseTrigger):
    def __init__(self, run_id: str):
        super().__init__()
        self.run_id = run_id
    
    async def run(self):
        # Poll the Databricks API for job status
        while True:
            status = await self.get_job_status(self.run_id)
            if status == 'SUCCESS':
                yield TriggerEvent({'status': 'success'})
            elif status == 'FAILED':
                yield TriggerEvent({'status': 'failed'})
            await asyncio.sleep(30)  # Poll every 30 seconds
```

### Fix 3: Use a Retry Policy with Exponential Backoff

```python
from airflow.utils.retry import RetryPolicy

task = DataPipelineJobOperator(
    task_id='databricks_job_task_1',
    databricks_conn_id='databricks_prod',
    pipeline_id='123456789',
    params={'source': 'production', 'batch_id': '2023-10-15'},
    retries=5,
    retry_delay=timedelta(minutes=2),
    retry_exponential_backoff=True,
    retry_max_delay=timedelta(hours=1),
    timeout=timedelta(hours=4)
)
```

### Fix 4: Use a Webhook for Job Completion Notification

```python
# In your Databricks job, add a webhook for completion notification
resource "databricks_job" "dlt_pipeline" {
  name = "dlt_pipeline"
  
  new_cluster {
    num_workers = 5
    spark_version = "13.3.x-scala2.12"
    node_type_id = "i3.xlarge"
  }
  
  webhook {
    id = "airflow_notification"
    success_url = "https://airflow-server/api/v1/webhooks/databricks_job_complete"
    failure_url = "https://airflow-server/api/v1/webhooks/databricks_job_failed"
  }
}
```

### Principal-Level Insight

**Never rely on synchronous polling for long-running jobs.** Use:
1. **Webhooks** for job completion notifications (Airflow has built-in support)
2. **Retry policies** with exponential backoff
3. **Timeouts** that exceed the maximum job duration
4. **Monitoring** of Databricks job status via the Airflow UI

### Follow-up

**Q:** How do you handle Airflow DAG failures when Databricks jobs fail?
**A:** Implement proper error handling in your DAG:
```python
from airflow.utils.email import send_email

# In the on_failure_callback
def on_failure(context):
    send_email(
        to='team@company.com',
        subject='Databricks Job Failed',
        html=f"""
        <p>Databricks job {context['task_id']} failed.</p>
        <p>Run ID: {context['run_id']}</p>
        <p>Error: {context['exception']}</p>
        """
    )
```
Use **SLA misses** for downstream tasks and implement alerting.

---

## Q25. Delta Live Table Pipeline Fails with "No Schema Found" on Auto Loader

**Scenario:** You have a DLT pipeline that uses Auto Loader to ingest JSON files from a cloud storage landing zone. The pipeline has been running for months. A new data source starts publishing files to the same landing zone with a different schema (additional optional fields). Auto Loader fails with:

```
java.lang.IllegalArgumentException: 
  No schema found for file: s3://landing/events/2023/10/15/part-00000.json
```

The `cloudFiles.schema.inferLimit` is set to the default (1000 files). What's the right fix?

**Answer:**

### Root Cause

Auto Loader infers the schema from the first `cloudFiles.schema.inferLimit` files it reads. A new data source with a different schema causes Auto Loader to fail because the inferred schema doesn't match the new files.

```
Old schema:
  {
    "id": "string",
    "timestamp": "timestamp",
    "event_type": "string"
  }

New schema:
  {
    "id": "string",
    "timestamp": "timestamp",
    "event_type": "string",
    "metadata": {
      "source": "string",
      "version": "integer"
    }
  }
```

### Fix 1: Increase the Schema Inference Limit

```python
# In your DLT pipeline:
@dlt.table(name="bronze_events")
@dlt.expect_or_drop("valid_id", "id IS NOT NULL")
def bronze_events():
    return (spark
        .readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.schema.inferLimit", "10000")  # Increase limit
        .option("cloudFiles.schema.evolve", "true")       # Evolve schema
        .load("s3://landing/events/")
    )
```

### Fix 2: Enable Schema Evolution

```python
# Enable schema evolution for Auto Loader
spark.conf.set("cloudFiles.schema.evolve", "true")

# Or in the DLT table definition:
@dlt.table(name="bronze_events", schema_evolution=true)
def bronze_events():
    return (spark
        .readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.schema.evolve", "true")
        .load("s3://landing/events/")
    )
```

### Fix 3: Provide a Fixed Schema

```python
from pyspark.sql.types import (
    StructType, StructField, StringType, TimestampType, IntegerType, MapType
)

events_schema = StructType([
    StructField("id", StringType(), False),
    StructField("timestamp", TimestampType(), False),
    StructField("event_type", StringType(), False),
    StructField("metadata", MapType(StringType(), StringType()), True)
])

# Use the fixed schema for reading
streaming_df = spark \
    .readStream \
    .format("cloudFiles") \
    .option("cloudFiles.format", "json") \
    .schema(events_schema) \
    .load("s3://landing/events/")
```

### Fix 4: Use a Schema Registry

```python
# Store the schema in a central location
schema_registry = {
    "events": {
        "schema": events_schema,
        "version": "1.2",
        "updated": "2023-10-15"
    }
}

# Load the schema from the registry
def get_schema(event_type):
    return schema_registry[event_type]["schema"]

# In the DLT pipeline:
@dlt.table(name="bronze_events")
def bronze_events():
    return (spark
        .readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .schema(get_schema("events"))
        .load("s3://landing/events/")
    )
```

### Principal-Level Insight

**Always define schemas explicitly for production data.** Schema inference is fragile and will break when the data changes. Use a schema registry to centralize schema management and versioning.

### Follow-up

**Q:** How do you handle schema changes in a DLT pipeline without downtime?
**A:** Use schema evolution with a migration step:
```python
@dlt.table(name="bronze_events", schema_evolution=True)
def bronze_events():
    return (spark
        .readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.schema.evolve", "true")
        .load("s3://landing/events/")
    )

# Add a validation step to detect schema changes
@dlt.expect_or_warn("valid_schema", "id IS NOT NULL AND event_type IS NOT NULL")
@dlt.table(name="silver_events")
def silver_events():
    return spark.table("bronze_events")
```
Monitor the expectation failures to detect schema drift.
---

## Q26. Photon Query Performance Regresses After Schema Evolution

**Scenario:** You enabled Photon on a SQL warehouse and noticed that a query that ran in 15 seconds now takes 45 seconds after schema evolution added 5 new columns to a Delta table. The query uses `SELECT *` and scans a 1 TB table with 100 partitions. What's causing the regression and how do you fix it?

**Answer:**

### Root Cause

Photon's code generation and column projection are sensitive to schema changes. When new columns are added, Photon may regenerate its execution plan and lose optimized column pruning, especially with `SELECT *`.

```
Old schema: 10 columns
New schema: 15 columns
Query: SELECT * FROM events WHERE event_date >= '2023-01-01'
```

### Fix 1: Replace `SELECT *` with Explicit Columns

```sql
-- Instead of SELECT *
SELECT
    id,
    event_type,
    event_timestamp,
    user_id,
    session_id
FROM events
WHERE event_date >= '2023-01-01'
```

### Fix 2: Force Photon to Use Column Pruning

```sql
-- Enable column pruning in Photon
SET spark.databricks.photon.columnPruning.enabled = true;

-- Force Photon to re-compile the query plan
SET spark.databricks.photon.queryPlanCache.enabled = false;
```

### Fix 3: Use Delta Table Optimizations

```sql
-- Run OPTIMIZE to compact files after schema evolution
OPTIMIZE delta.`/data/events` ZORDER BY (event_type, user_id);

-- Update statistics for the query optimizer
ANALYZE TABLE events COMPUTE STATISTICS;
```

### Fix 4: Create a Materialized View

```sql
-- Create a materialized view with the columns you need
CREATE MATERIALIZED VIEW mv_events_summary AS
SELECT
    id,
    event_type,
    event_timestamp,
    user_id,
    session_id
FROM events
WHERE event_date >= '2023-01-01';
```

### Fix 5: Use a Different Storage Format for Hot Data

```python
# Convert frequently queried partitions to Photon-friendly format
# Photon works best with columnar formats (Parquet, Delta)
# Avoid JSON/CSV for analytical queries
spark.conf.set("spark.sql.parquet.columnarReader.caseSensitive", "true")
```

### Follow-up

**Q:** How do you prevent schema evolution from impacting performance?
**A:** 
1. **Pin the schema** for production tables — add a schema validation step in your ingestion pipeline
2. **Use explicit column lists** in all production queries (no `SELECT *`)
3. **Monitor query performance** after schema changes with the Query History API
4. **Run OPTIMIZE** after every schema evolution to compact files

---

## Q27. Delta Sharing Cross-Organization Data Access Fails with Permission Denied

**Scenario:** Your company uses Delta Sharing to share a Delta table with a partner organization. The partner receives the error:

```
Error: PERMISSION_DENIED
Reason: User does not have permission to access the shared table
```

You've confirmed the partner has the correct receiver token and the table is listed in the share. What's the issue and how do you fix it?

**Answer:**

### Root Cause

Delta Sharing uses a two-layer permission model: the **share** grants access to the partner organization, and the **table** grants access to specific tables within the share. Both must be properly configured.

```
Share: company_events_share
  └── Tables:
      └── events (granted to partner)

Receiver: partner_org
  └── Profile: partner_profile.json
      └── Token: abc123...
```

### Fix 1: Verify Share and Table Permissions

```sql
-- In Databricks SQL:
-- Check the share exists
SHOW SHARES;

-- Check the table is in the share
SHOW TABLES IN SHARE company_events_share;

-- Grant the partner access to the table
GRANT SELECT ON TABLE events TO SHARE company_events_share;

-- Grant the share to the partner organization
GRANT READ ON SHARE company_events_share TO RECIPIENT partner_org;
```

### Fix 2: Verify the Profile Configuration

```json
// partner_profile.json
{
  "name": "company_events_share",
  "share": "company_events_share",
  "url": "https://databricks.net/sharing/receiver/abc123",
  "token": "abc123..."
}
```

### Fix 3: Check the Receiver Token

```sql
-- In Databricks SQL:
-- Verify the receiver token is valid
SHOW RECEIVERS;

-- Check the token expiration
SELECT * FROM system.receivers WHERE name = 'partner_org';
```

### Fix 4: Use the Delta Sharing CLI

```bash
# Verify the share is accessible
delta sharing list-shares --profile partner_profile

# List the tables in the share
delta sharing list-tables --share company_events_share --profile partner_profile

# Query the shared table
delta sharing query --table company_events_share.events --profile partner_profile
```

### Fix 5: Monitor Access Logs

```sql
-- In Databricks SQL:
-- Check the access logs for the shared table
SELECT * FROM system.access_logs
WHERE share_name = 'company_events_share'
  AND table_name = 'events'
ORDER BY timestamp DESC;
```

### Follow-up

**Q:** How do you revoke access to a shared table?
**A:**
```sql
-- Revoke access to the table
REVOKE SELECT ON TABLE events FROM SHARE company_events_share;

-- Revoke access to the share
REVOKE READ ON SHARE company_events_share FROM RECIPIENT partner_org;

-- Drop the share (optional)
DROP SHARE company_events_share;
```
Monitor the access logs to verify the revocation.

---

## Q28. MLflow Model Registry: Production Model Drifts After Registry Update

**Scenario:** You use MLflow Model Registry to manage production ML models. A new model version is promoted to "Production" stage. After deployment, the model's predictions drift significantly from the previous version. The model was trained on the same data pipeline but with a different feature set. What's the issue and how do you fix it?

**Answer:**

### Root Cause

The model was promoted to production without proper validation. MLflow Model Registry allows you to tag, log and track models, but it does not automatically validate model quality or drift.

```
Model v1: Trained on features A, B, C (baseline)
Model v2: Trained on features A, B, C, D, E (drifted)
```

### Fix 1: Implement Model Validation in the Promotion Pipeline

```python
from mlflow.models import validate

# Validate the model before promotion
def validate_model(model_uri, validation_data):
    """Validate the model before promotion"""
    # Load the model
    model = mlflow.pyfunc.load_model(model_uri)
    
    # Predict on validation data
    predictions = model.predict(validation_data)
    
    # Calculate metrics
    metrics = {
        'accuracy': accuracy_score(validation_data['target'], predictions),
        'precision': precision_score(validation_data['target'], predictions),
        'recall': recall_score(validation_data['target'], predictions)
    }
    
    # Validate against thresholds
    if metrics['accuracy'] < 0.95:
        raise ValueError(f"Model accuracy too low: {metrics['accuracy']}")
    
    return metrics
```

### Fix 2: Use MLflow Model Registry with Automated Testing

```python
# In your CI/CD pipeline:
def promote_model(model_uri, run_id):
    """Promote a model to production after validation"""
    client = MlflowClient()
    
    # Create the model version
    version = client.create_model_version(
        name="production_model",
        source=model_uri,
        run_id=run_id
    )
    
    # Run automated tests
    validation_data = load_validation_data()
    metrics = validate_model(model_uri, validation_data)
    
    # Log the metrics
    client.log_metric(version.run_id, "accuracy", metrics['accuracy'])
    client.log_metric(version.run_id, "precision", metrics['precision'])
    
    # Transition to production
    client.transition_model_version_stage(
        name="production_model",
        version=version.version,
        stage="Production"
    )
```

### Fix 3: Implement Model Drift Monitoring

```python
from mlflow.models import validate

# Monitor model drift in production
def monitor_model_drift(model_uri, production_data):
    """Monitor model drift in production"""
    # Load the model
    model = mlflow.pyfunc.load_model(model_uri)
    
    # Predict on production data
    predictions = model.predict(production_data)
    
    # Calculate drift metrics
    drift_metrics = {
        'data_drift': calculate_data_drift(production_data),
        'concept_drift': calculate_concept_drift(predictions, production_data['target'])
    }
    
    # Alert if drift is detected
    if drift_metrics['data_drift'] > 0.1:
        send_alert("Data drift detected", drift_metrics)
    
    return drift_metrics
```

### Fix 4: Use Feature Store for Consistent Features

```python
# Use a feature store to ensure consistent features
from feast import FeatureStore

# Load features from the feature store
feature_store = FeatureStore(repo_path=".")

# Get the feature set
feature_set = feature_store.get_feature_set("events_features")

# Get the features for prediction
features = feature_store.get_online_features(
    entities=[{"event_id": "12345"}],
    feature_names=["feature_a", "feature_b", "feature_c"]
).to_df()
```

### Follow-up

**Q:** How do you rollback a model that was promoted to production?
**A:**
```python
# Rollback to the previous model version
client = MlflowClient()

# Get the current production model
current = client.get_model_version_by_stage("production_model", "Production")

# Transition the previous version back to production
client.transition_model_version_stage(
    name="production_model",
    version=current.version - 1,  # Previous version
    stage="Production"
)

# Log the rollback
client.log_metric(current.run_id, "rollback_reason", "model_drift")
```
Always maintain a rollback plan and test it regularly.

---

## Q29. Cluster Cost Optimization: Reduce Databricks Cluster Costs by 50%

**Scenario:** Your Databricks cluster costs $25,000/month. The cluster runs 24/7 with 20 workers, each with 16 GB RAM. The cluster is used for both batch processing (nightly) and interactive analytics (during business hours). What's the right approach to reduce costs by 50% without impacting performance?

**Answer:**

### Root Cause

A 24/7 cluster with 20 workers is expensive because it's idle for ~16 hours/day (batch only runs nightly, analytics only during business hours).

```
Current: 20 workers × 24 hours × 30 days = 14,400 hours/month
Cost: $25,000/month
```

### Fix 1: Use Auto-Scaling

```python
# Configure auto-scaling for the cluster
{
  "num_workers": 5,
  "min_num_workers": 2,
  "max_num_workers": 20,
  "enable_elastic_disk": true,
  "autotermination_minutes": 30  # Auto-terminate after 30 mins idle
}
```

### Fix 2: Use Cluster Pools

```python
# Use cluster pools to share idle resources
{
  "pool_id": "prod-pool-1",
  "num_workers": 10,
  "min_num_workers": 2,
  "max_num_workers": 20
}

# Assign jobs to the pool
{
  "existing_cluster_id": null,
  "new_cluster": {
    "cluster_source": "JOB",
    "spark_version": "13.3.x-scala2.12",
    "node_type_id": "i3.xlarge",
    "num_workers": 10
  }
}
```

### Fix 3: Use Job Clusters Instead of All-Purpose Clusters

```python
# Use job clusters for batch processing
# Job clusters are cheaper than all-purpose clusters
# They are terminated after the job completes
{
  "new_cluster": {
    "spark_version": "13.3.x-scala2.12",
    "node_type_id": "i3.xlarge",
    "num_workers": 10,
    "autotermination_minutes": 15
  }
}
```

### Fix 4: Use Spot Instances for Non-Critical Workloads

```python
# Use spot instances for non-critical workloads
{
  "node_type_id": "i3.xlarge",
  "availability": "SPOT",
  "spot_bid_price_per_hour": 0.5
}

# Use on-demand instances for critical workloads
{
  "node_type_id": "i3.xlarge",
  "availability": "ON_DEMAND"
}
```

### Fix 5: Implement a Cost Budget and Alerts

```python
# Set up a cost budget
{
  "budget_amount": 12500,  # $12,500/month (50% reduction)
  "alert_threshold": 0.8  # Alert at 80% of budget
}

# Monitor costs
SELECT
    date,
    cluster_name,
    cost,
    hours_used
FROM system.billing_costs
WHERE date >= '2023-10-01'
ORDER BY cost DESC;
```

### Follow-up

**Q:** How do you measure the ROI of cost optimization?
**A:**
1. **Baseline cost**: $25,000/month
2. **Optimized cost**: $12,500/month (50% reduction)
3. **Savings**: $12,500/month = $150,000/year
4. **Implementation cost**: 2 weeks of engineering time
5. **ROI**: (150,000 - 10,000) / 10,000 = 14x

Track the savings in your cost dashboard and report them quarterly.

---

## Q30. Streaming Exactly-Once with External Kafka System

**Scenario:** You need to build a streaming pipeline that reads from Kafka, processes the data and writes to a Delta table. The pipeline must guarantee exactly-once processing even if the streaming job restarts. Kafka has 50 partitions. The Delta table is partitioned by `date`. What's the correct architecture?

**Answer:**

### Root Cause

Exactly-once semantics require:
1. **Idempotent writes** to the sink (Delta table)
2. **Checkpointing** to track offsets
3. **Transaction management** for atomic commits

```
Kafka → Spark Streaming → Delta Table
```

### Fix 1: Use Kafka Reader with Offset Management

```python
from pyspark.sql.functions import col

# Read from Kafka with offset tracking
streaming_df = (spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka1:9092,kafka2:9092,kafka3:9092")
    .option("subscribe", "events")
    .option("startingOffsets", "latest")
    .option("maxOffsetsPerTrigger", "10000")
    .option("failOnDataLoss", "false")  # Handle data loss gracefully
    .load()
)
```

### Fix 2: Use Checkpointing for Exactly-Once Semantics

```python
# Enable checkpointing
checkpoint_path = "s3://datalake/checkpoints/events/"

# Write to Delta with checkpointing
query = (streaming_df
    .writeStream
    .format("delta")
    .option("checkpointLocation", checkpoint_path)
    .option("path", "s3://datalake/events/")
    .option("trigger", "processingTime", "30 seconds")
    .option("maxFilesPerTrigger", "10")
    .start()
)
```

### Fix 3: Use Delta's Transaction Management

```python
# Delta's transaction log ensures exactly-once semantics
# The transaction log tracks all changes to the table
# Use MERGE for idempotent updates

from pyspark.sql.functions import col

# Process the data
processed_df = streaming_df.select(
    col("key").cast("string").alias("event_id"),
    col("value").cast("string").alias("event_data"),
    col("timestamp").cast("timestamp").alias("event_timestamp"),
    col("partition").cast("string").alias("event_date")
)

# Write to Delta with idempotent MERGE
query = (processed_df
    .writeStream
    .format("delta")
    .option("checkpointLocation", checkpoint_path)
    .option("path", "s3://datalake/events/")
    .trigger(processingTime="30 seconds")
    .start()
)
```

### Fix 4: Handle Kafka Offset Management

```python
# Manually manage Kafka offsets for exactly-once processing
# Use the offset commit feature in Spark Streaming

from pyspark.sql.functions import col

# Read the current offsets
offsets = streaming_df.select("offset").rdd.map(lambda r: r[0]).collect()

# Save the offsets to a checkpoint
# On restart, resume from the last committed offset
```

### Fix 5: Monitor the Streaming Pipeline

```python
# Monitor the streaming pipeline
# Use the Spark UI to track the progress
# Use the Delta transaction log to track changes

# In the Spark UI:
# - Checkpoint location: s3://datalake/checkpoints/events/
# - Trigger: 30 seconds
# - Max files per trigger: 10
# - Batch size: 10,000 records
```

### Follow-up

**Q:** How do you handle late-arriving data in exactly-once streaming?
**A:** Use **watermarks** and **windowed aggregations**:
```python
# Set a watermark for late-arriving data
streaming_df = streaming_df.withWatermark("event_timestamp", "1 hour")

# Use windowed aggregations
windowed_df = streaming_df.groupBy(
    window(col("event_timestamp"), "1 hour")
).count()

# Write to Delta with watermark
query = (windowed_df
    .writeStream
    .format("delta")
    .option("checkpointLocation", checkpoint_path)
    .option("path", "s3://datalake/events_windowed/")
    .trigger(processingTime="30 seconds")
    .start()
)
```
The watermark ensures that late-arriving data is processed up to 1 hour after the event time.

---

## Summary

| # | Topic | Key Takeaways |
|---|-------|---------------|
| 1 | Delta Live Tables vs. Spark Streaming | DLT provides simplified pipelines with auto-scaling, schema evolution, and built-in data quality checks; use for ETL pipelines, Spark Streaming for custom processing |
| 2 | Delta Merge Performance | Use Z-ordering, batch size tuning, and partition pruning; monitor file sizes and partition count |
| 3 | Delta Time Travel for Data Recovery | Use `TIMESTAMP AS OF` or `VERSION AS OF`; set retention period; test recovery procedures regularly |
| 4 | Delta Streaming with Schema Evolution | Use `mergeSchema`, `cloudFiles.schema.evolve`; monitor schema changes and validate data quality |
| 5 | Spark Structured Streaming with Kafka | Use `maxOffsetsPerTrigger`, `failOnDataLoss`, checkpointing; handle late data with watermarks |
| 6 | Delta Table ACID Transactions | Delta provides ACID guarantees; use optimistic concurrency control; handle write conflicts with retries |
| 7 | Databricks SQL Warehouse vs. Compute | SQL Warehouse for interactive analytics with auto-scaling; Compute for ad-hoc queries; use cluster pools for shared resources |
| 8 | Unity Catalog for Data Governance | Centralized metadata, fine-grained access control, audit logging; implement row-level security and data masking |
| 9 | Delta Lake vs. Apache Iceberg | Delta has better integration with Databricks; Iceberg has broader ecosystem support; both provide ACID, time travel and schema evolution |
| 10 | Spark Structured Streaming Watermarks | Use `withWatermark()` for late data handling; set watermark interval based on data freshness requirements |
| 11 | Delta Concurrency and Performance | Use optimistic concurrency control; monitor write conflicts; tune partition size and file count |
| 12 | DLT Pipeline Failure and Recovery | Use `expect_or_drop`, `expect_or_fail`, `expect_or_warn`; implement retry logic and monitoring |
| 13 | Delta Write Performance Degradation | Run `OPTIMIZE` and `VACUUM`; tune `spark.databricks.delta.autoCompact.enabled`; monitor file sizes |
| 14 | Hive Metastore to Unity Catalog Migration | Use `SYNC` command; resolve object name conflicts; test permissions after migration |
| 15 | Cross-Workspace Data Sharing | Use Delta Sharing or Unity Catalog; manage access tokens; monitor cross-workspace data access |
| 16 | Stale Data in Real-Time Dashboard | Use incremental processing with `autoOptimize`; partition by time; monitor data freshness SLAs |
| 17 | Feature Store Mismatch | Use Unity Catalog for consistent features; implement feature versioning; validate model inputs |
| 18 | Delta Sharing Cross-Organization | Use receiver tokens; set retention period; monitor access logs; revoke access when needed |
| 19 | Streaming Checkpoint Recovery | Use `checkpointLocation` with durable storage; handle corrupted checkpoints by deleting and restarting |
| 20 | Cross-Workspace Job Orchestration | Use Databricks Jobs API or Airflow with proper timeout/retry; implement idempotent writes |
| 21 | Spark Cluster OOM on Shuffled Join | Broadcast small tables; enable AQE skew join; increase shuffle partitions; pre-filter before join |
| 22 | Delta Compaction with Concurrent Writes | Run `OPTIMIZE` during low-traffic; use `autoOptimize`; separate read/write partitions |
| 23 | Photon Query Incorrect Results | Force Spark engine for problematic queries; validate with both engines; check function compatibility |
| 24 | Airflow DAG Databricks Job Failure | Increase API timeout; use webhooks for job completion; implement retry with exponential backoff |
| 25 | DLT Auto Loader Schema Issues | Provide explicit schema; enable schema evolution; use schema registry; monitor schema drift |
| 26 | Photon Performance Regression | Replace `SELECT *`; force column pruning; run `OPTIMIZE` after schema changes; use materialized views |
| 27 | Delta Sharing Permission Denied | Verify share/table permissions; check receiver token; use Delta Sharing CLI; monitor access logs |
| 28 | MLflow Model Registry Drift | Validate model before promotion; use automated testing; implement drift monitoring; use feature store |
| 29 | Cluster Cost Optimization | Use auto-scaling, cluster pools, job clusters, spot instances; implement cost budgets and alerts |
| 30 | Streaming Exactly-Once with Kafka | Use checkpointing, offset management, transaction management; handle late data with watermarks |