# PySpark for Principal Data Engineer Interviews — Part 2: Code Practicals (P1–P30)

**How to use:** Write each solution yourself first (10–15 min), then compare. Every solution notes **why**, **edge cases/scale**, and a **Principal add**. Code assumes Spark 3.4+ unless noted; Delta examples use `delta-spark`. Imports used throughout:

```python
from pyspark.sql import SparkSession, functions as F, Window, types as T
spark = SparkSession.builder.appName("prep").getOrCreate()
```

---

## A. Reading, schemas, and cleaning

### P1. Read CSV and JSON with an explicit schema (no inference) and explain why.
```python
schema = T.StructType([
    T.StructField("order_id", T.StringType(), False),
    T.StructField("customer_id", T.StringType(), True),
    T.StructField("amount", T.DecimalType(12, 2), True),
    T.StructField("order_ts", T.TimestampType(), True),
])
df = (spark.read.schema(schema)
      .option("header", True).option("mode", "FAILFAST")
      .option("timestampFormat", "yyyy-MM-dd'T'HH:mm:ss")
      .csv("s3://bucket/orders/"))
```
`inferSchema` costs an extra full pass and yields unstable types as data changes. Explicit schemas make contracts reviewable and detect drift. Modes: `PERMISSIVE` (default), `DROPMALFORMED`, `FAILFAST`. **Principal add:** store schemas as versioned artifacts (DDL strings or JSON via `schema.json()`) shared across jobs.

### P2. Handle corrupt records without losing them.
```python
schema = schema.add("_corrupt_record", T.StringType(), True)
raw = (spark.read.schema(schema)
       .option("mode", "PERMISSIVE")
       .option("columnNameOfCorruptRecord", "_corrupt_record")
       .json("s3://bucket/events/"))
raw = raw.cache()   # Spark disallows queries referencing only the corrupt column on raw files; cache first
good = raw.filter("_corrupt_record IS NULL").drop("_corrupt_record")
bad  = raw.filter("_corrupt_record IS NOT NULL").select("_corrupt_record")
bad.write.mode("append").json("s3://bucket/dlq/events/")
```
The corrupt-record column must be in the schema. Databricks also offers `badRecordsPath`. **Principal add:** a threshold gate: fail the job if `bad.count()/total > 1%`, and emit counts as metrics.

### P3. Clean and cast columns safely (nulls, trimming, bad dates) with a "quarantine" flag.
```python
clean = (df
  .withColumn("customer_id", F.nullif(F.trim("customer_id"), F.lit("")))
  .withColumn("order_date", F.to_date("order_ts"))
  .withColumn("amount_num", F.col("amount").try_cast("decimal(12,2)"))  # 3.5+: NULL instead of error under ANSI
  .withColumn("is_valid", F.col("order_id").isNotNull() & F.col("amount_num").isNotNull() & (F.col("amount_num") >= 0)))
```
Under non-ANSI mode a bad cast silently yields NULL; under ANSI mode it raises. Be deliberate about which behavior you want and test it. **Principal add:** write invalid rows to a quarantine table with a reason code rather than dropping them.

### P4. Replace a Python UDF with built-ins.
```python
# BAD: row-at-a-time Python
@F.udf("string")
def tier(a): return "high" if a and a > 1000 else "low"

# GOOD: Catalyst-visible expression
df = df.withColumn("tier", F.when(F.col("amount") > 1000, "high").otherwise("low"))
```
Also: `F.expr("CASE WHEN ...")`, `F.regexp_extract`, `F.transform` (higher-order functions on arrays), `F.aggregate`, `F.filter`. **Principal add:** show `df.explain()` for both: the UDF version has a `BatchEvalPython` node (and breaks pushdown/codegen).

## B. Windows and aggregations

### P5. Keep the latest record per key (dedup).
```python
w = Window.partitionBy("id").orderBy(F.col("updated_at").desc(), F.col("lsn").desc())
latest = (df.withColumn("rn", F.row_number().over(w)).filter("rn = 1").drop("rn"))
```
Use a deterministic tie-breaker (`lsn`/sequence), or results vary between runs. For "keep any one" with no ordering, `dropDuplicates(["id"])` is non-deterministic about *which* row survives. **Alternative at scale:** `groupBy("id").agg(F.max_by("payload", "updated_at"))` (3.3+) avoids a window sort.

### P6. Top-N per group.
```python
w = Window.partitionBy("customer_id").orderBy(F.col("amount").desc())
top3 = df.withColumn("rn", F.row_number().over(w)).filter("rn <= 3")
# rank()/dense_rank() keep ties; row_number() does not
```
Each window partition must fit through a sort on one task: a hot key means skew. **Principal add:** for a huge top-N, pre-reduce per partition with `groupBy(...).agg(F.collect_list(...))` caps or approximate methods.

### P7. Running total, moving average, lag/lead with time ranges.
```python
w_run = Window.partitionBy("acct").orderBy("ts").rowsBetween(Window.unboundedPreceding, Window.currentRow)
w_7d  = (Window.partitionBy("acct").orderBy(F.col("ts").cast("long"))
         .rangeBetween(-7 * 86400, 0))                # time-based range on epoch seconds
out = (df.withColumn("running", F.sum("amt").over(w_run))
         .withColumn("avg_7d",  F.avg("amt").over(w_7d))
         .withColumn("prev_amt", F.lag("amt", 1).over(Window.partitionBy("acct").orderBy("ts"))))
```
`rowsBetween` is positional; `rangeBetween` is value-based (needs a numeric ordering expression). A window with no `partitionBy` pulls everything into one partition: a performance trap.

### P8. Sessionization (new session after 30 minutes of inactivity).
```python
w = Window.partitionBy("user").orderBy("ts")
s = (df
  .withColumn("prev_ts", F.lag("ts").over(w))
  .withColumn("new_sess", (F.col("ts").cast("long") - F.col("prev_ts").cast("long") > 1800)
                          .cast("int"))
  .na.fill({"new_sess": 1})
  .withColumn("session_id", F.sum("new_sess").over(w.rowsBetween(Window.unboundedPreceding, 0))))
```
Flag boundaries, then cumulative-sum to assign IDs. Make the session key globally unique with `concat(user, '-', session_id)`. For streaming, use `session_window` (3.2+).

### P9. Pivot and unpivot.
```python
wide = (df.groupBy("product").pivot("month", ["2026-01", "2026-02", "2026-03"])  # list values explicitly
          .agg(F.sum("sales")))
long = wide.unpivot("product", ["2026-01", "2026-02", "2026-03"], "month", "sales")   # 3.4+
# older: F.expr("stack(3, '2026-01', `2026-01`, '2026-02', `2026-02`, '2026-03', `2026-03`) as (month, sales)")
```
Without an explicit value list, `pivot` triggers an extra job to find distinct values and is capped by `spark.sql.pivotMaxValues` (10,000).

### P10. Distinct counts and percentiles at scale.
```python
agg = df.groupBy("day").agg(
    F.countDistinct("user_id").alias("exact_uu"),
    F.approx_count_distinct("user_id", rsd=0.02).alias("approx_uu"),   # HyperLogLog++
    F.percentile_approx("latency_ms", [0.5, 0.95, 0.99], 10000).alias("pcts"))
```
Exact distinct needs a full shuffle of distinct values; HLL++ uses small sketches with ~2% error and merges. Percentile accuracy parameter trades memory for precision. **Principal add:** persist sketches (`hll_sketch_agg`, 3.5+) to roll up distinct counts across days without rescanning.

## C. Complex types

### P11. Parse JSON strings, explode arrays, and flatten structs.
```python
payload_schema = T.StructType([
  T.StructField("id", T.StringType()),
  T.StructField("items", T.ArrayType(T.StructType([
      T.StructField("sku", T.StringType()), T.StructField("qty", T.IntegerType())]))),
  T.StructField("addr", T.StructType([T.StructField("city", T.StringType())]))])

parsed = df.withColumn("p", F.from_json("raw_json", payload_schema))
flat = (parsed.select("p.id", "p.addr.city", F.explode_outer("p.items").alias("item"))
              .select("id", "city", "item.sku", "item.qty"))
```
`explode` drops rows with empty/NULL arrays; `explode_outer` keeps them. `from_json` returns NULL for unparseable rows (check). **Principal add:** generic recursive struct flattener, and a schema-drift strategy (`VARIANT` type in Spark 4.0 / Delta, or keep the raw column alongside).

### P12. Use higher-order functions on arrays instead of explode+groupBy.
```python
df = (df.withColumn("prices_usd", F.transform("prices", lambda x: x * F.col("fx")))
        .withColumn("n_big", F.size(F.filter("prices", lambda x: x > 100)))
        .withColumn("total", F.aggregate("prices", F.lit(0.0), lambda acc, x: acc + x)))
```
Avoids a shuffle: it's per-row, in-JVM. Good for arrays that are small to moderate per row.

## D. Joins and skew

### P13. Broadcast join and verify it in the plan.
```python
out = facts.join(F.broadcast(dim), "dim_id", "left")
out.explain("formatted")        # look for BroadcastHashJoin and BroadcastExchange
```
Broadcasting a table too large for driver/executor memory causes OOM; the driver collects it first. Raise `spark.sql.autoBroadcastJoinThreshold` carefully, and note that AQE can convert at runtime. **Principal add:** CBO statistics (`ANALYZE TABLE ... COMPUTE STATISTICS`) improve decisions for catalog tables.

### P14. Fix a skewed join with salting.
```python
N = 16
facts_s = facts.withColumn("salt", (F.rand(42) * N).cast("int"))
dim_s = dim.withColumn("salt", F.explode(F.array([F.lit(i) for i in range(N)])))
res = facts_s.join(dim_s, ["key", "salt"]).drop("salt")
```
Only salt *hot* keys to limit replication: `F.when(F.col("key").isin(hot_keys), rand-salt).otherwise(0)` on the big side, and replicate only the matching dim rows. First try AQE skew join:
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

### P15. Incremental change detection with anti/semi joins.
```python
new_rows   = incoming.join(existing, "id", "left_anti")                       # not in existing
changed    = incoming.alias("i").join(existing.alias("e"), "id") \
                .filter("i.row_hash <> e.row_hash").select("i.*")
deleted    = existing.join(incoming, "id", "left_anti")
active_ids = orders.join(customers_active, "customer_id", "left_semi")        # filter without duplicating columns
```
Compute `row_hash` with `F.sha2(F.concat_ws("||", *cols), 256)`. Null-safe: `concat_ws` skips NULLs, so use `coalesce`/sentinels to distinguish `NULL` from `''`.

### P16. Avoid a cartesian-product explosion and detect it.
```python
# Accidentally missing join condition
bad = a.join(b)              # CartesianProduct / BroadcastNestedLoopJoin in the plan
# Spark 3 errors unless spark.sql.crossJoin.enabled / explicit crossJoin
good = a.join(b, a.k == b.k)
```
For range/non-equi joins (`a.ts BETWEEN b.start AND b.end`) use bucketing the time dimension into equi-join keys (explode `b` into time bins) to turn it into an equi-join. Delta/Databricks offer range-join hints.

## E. Writing and layout

### P17. Write partitioned Parquet with controlled file counts and sizes.
```python
(df.repartition("event_date")                       # one task-group per date, avoids many files per partition
   .write.mode("overwrite")
   .option("maxRecordsPerFile", 5_000_000)
   .partitionBy("event_date")
   .parquet("s3://bucket/curated/events/"))

spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")   # overwrite only touched partitions
```
Without `repartition(cols)`, each of N shuffle tasks writes a file into every partition it holds data for: `N × partitions` files. If one date is skewed, use `repartition(n, "event_date", F.rand())` style spreading. **Principal add:** prefer a table format (Delta/Iceberg) over raw directory overwrites for atomicity.

### P18. Compaction/maintenance.
```python
# Delta
spark.sql("OPTIMIZE db.events WHERE event_date >= '2026-11-01' ZORDER BY (user_id)")
spark.sql("VACUUM db.events RETAIN 168 HOURS")
# Iceberg
spark.sql("CALL cat.system.rewrite_data_files(table => 'db.events', strategy => 'binpack')")
spark.sql("CALL cat.system.expire_snapshots(table => 'db.events', older_than => TIMESTAMP '2026-11-01 00:00:00')")
```
`VACUUM` retention must exceed the longest-running reader/stream and time-travel need. Schedule per-partition compaction on recent data only. Liquid clustering (Delta) / sort orders (Iceberg) can replace Z-order.

## F. Table formats and CDC patterns

### P19. SCD Type 2 with Delta MERGE.
```python
from delta.tables import DeltaTable
tgt = DeltaTable.forName(spark, "dim_customer")

# changed records produce two rows: one to close the old version, one to insert the new
updates = src.alias("s")
changed = (updates.join(tgt.toDF().alias("t"),
              (F.col("s.cust_id") == F.col("t.cust_id")) & F.col("t.is_current") &
              (F.col("s.attr_hash") != F.col("t.attr_hash")))
           .select("s.*"))
staged = (changed.withColumn("merge_key", F.lit(None).cast("string"))           # forces insert
          .unionByName(updates.withColumn("merge_key", F.col("cust_id"))))      # forces match/update

(tgt.alias("t").merge(staged.alias("s"), "t.cust_id = s.merge_key AND t.is_current = true")
  .whenMatchedUpdate(condition="t.attr_hash <> s.attr_hash",
      set={"is_current": "false", "end_date": "s.effective_date"})
  .whenNotMatchedInsert(values={
      "cust_id": "s.cust_id", "attr_hash": "s.attr_hash", "name": "s.name",
      "start_date": "s.effective_date", "end_date": "null", "is_current": "true"})
  .execute())
```
The union trick: rows with a NULL `merge_key` can't match so they insert as the new version. Ensure the source has one row per key per run (dedupe first), and handle late-arriving/out-of-order changes by ordering on source sequence.

### P20. Apply Debezium CDC (insert/update/delete) idempotently.
```python
def upsert_cdc(batch_df, batch_id):
    w = Window.partitionBy("id").orderBy(F.col("source_lsn").desc())
    latest = (batch_df.withColumn("rn", F.row_number().over(w)).filter("rn = 1").drop("rn"))
    (DeltaTable.forName(spark, "silver.customers").alias("t")
      .merge(latest.alias("s"), "t.id = s.id")
      .whenMatchedDelete(condition="s.op = 'd'")
      .whenMatchedUpdateAll(condition="s.op IN ('u','c','r') AND s.source_lsn > t.source_lsn")
      .whenNotMatchedInsertAll(condition="s.op <> 'd'")
      .execute())
```
Key points: dedupe *within* the batch to the latest event per key (MERGE fails if multiple source rows match one target row); guard with `source_lsn > t.source_lsn` so replays and out-of-order events are no-ops (idempotent); decide delete policy (hard delete vs soft-delete flag for audit).

### P21. Incremental batch load with a watermark table.
```python
last = spark.sql("SELECT max(wm) FROM ctl.watermarks WHERE tbl='orders'").first()[0] or "1970-01-01"
inc = (spark.read.jdbc(url, "orders", properties=props)
         .filter(F.col("updated_at") > F.lit(last) - F.expr("INTERVAL 5 MINUTES")))   # lookback overlap for late commits
inc.write.format("delta").mode("append").saveAsTable("bronze.orders_raw")
new_wm = inc.agg(F.max("updated_at")).first()[0]
# commit the watermark only after the write succeeds, in the same orchestrator task
```
Idempotence comes from the downstream MERGE on the key (the lookback creates duplicates). For JDBC, parallelize with `partitionColumn/lowerBound/upperBound/numPartitions` and push the predicate into a subquery.

## G. Structured Streaming

### P22. Kafka → Delta bronze with checkpointing and rate control.
```python
raw = (spark.readStream.format("kafka")
       .option("kafka.bootstrap.servers", "b1:9092,b2:9092")
       .option("subscribe", "orders")
       .option("startingOffsets", "earliest")
       .option("maxOffsetsPerTrigger", 500_000)
       .option("failOnDataLoss", "true").load())

bronze = raw.select(F.col("key").cast("string").alias("k"),
                    F.col("value").cast("string").alias("v"),
                    "topic", "partition", "offset", "timestamp")

q = (bronze.writeStream.format("delta")
     .option("checkpointLocation", "s3://bucket/chk/orders_bronze")
     .trigger(processingTime="30 seconds")
     .outputMode("append").toTable("bronze.orders"))
```
Keep `topic/partition/offset` for lineage and dedup. The checkpoint path is part of the query's identity: one per query, never shared, never deleted casually. Use `trigger(availableNow=True)` for scheduled incremental batch runs.

### P23. Windowed aggregation with watermark (and what gets emitted when).
```python
agg = (events.withWatermark("event_ts", "10 minutes")
       .groupBy(F.window("event_ts", "5 minutes"), "user_id")
       .agg(F.count("*").alias("n"), F.sum("amt").alias("amt")))
agg.writeStream.outputMode("append").format("delta") \
   .option("checkpointLocation", chk).toTable("silver.user_5m")
```
In `append` mode each window is emitted once, after the watermark passes the window end, so a 5-min window with a 10-min watermark appears at least ~10 min late. `update` mode emits earlier with revisions (the sink must upsert). Events later than the watermark are dropped (they aren't errors), so monitor `numRowsDroppedByWatermark`.

### P24. `foreachBatch` for idempotent upserts and multi-sink writes.
```python
def process(df, batch_id):
    df = df.dropDuplicates(["event_id"])
    (df.write.format("delta").mode("append")
       .option("txnAppId", "orders_enrich").option("txnVersion", batch_id)   # Delta idempotent write per batch
       .saveAsTable("silver.orders"))
    df.write.mode("append").parquet("s3://bucket/audit/")                    # second sink

(stream.writeStream.foreachBatch(process)
 .option("checkpointLocation", chk).start())
```
`foreachBatch` can run *twice* for the same `batch_id` after a failure, so every side effect must be idempotent. Persist `df` inside the function if writing to multiple sinks to avoid recompute, and `unpersist` after.

### P25. Stream-stream join with time bounds and watermarks.
```python
imp = impressions.withWatermark("imp_ts", "2 hours")
clk = clicks.withWatermark("clk_ts", "3 hours")
joined = imp.join(clk,
    F.expr("""imp.ad_id = clk.ad_id AND
              clk_ts BETWEEN imp_ts AND imp_ts + INTERVAL 1 HOUR"""), "leftOuter")
```
The time-range condition and watermarks bound state; without them state grows forever. Outer joins emit unmatched rows only after the watermark passes. Stateful ops → use the RocksDB state store for large state: `spark.sql.streaming.stateStore.providerClass=...RocksDBStateStoreProvider`.

### P26. Streaming deduplication.
```python
dedup = (events.withWatermark("event_ts", "1 hour")
         .dropDuplicatesWithinWatermark(["event_id"]))      # 3.5+; state is bounded by the watermark
# older: .dropDuplicates(["event_id", "event_ts"]) with watermark (event_ts must be in the key)
```
Duplicates arriving later than the watermark window pass through. If the exact-once requirement is strict, add a durable idempotent MERGE at the sink.

## H. Quality, testing, and observability

### P27. A reusable data-quality check layer.
```python
def dq_report(df, checks: dict[str, "Column"]):
    aggs = [F.sum(F.when(~cond, 1).otherwise(0)).alias(name) for name, cond in checks.items()]
    return df.agg(F.count("*").alias("rows"), *aggs).first().asDict()

checks = {
  "id_not_null":   F.col("id").isNotNull(),
  "amount_nonneg": F.col("amount") >= 0,
  "status_valid":  F.col("status").isin("NEW", "PAID", "CANCELLED"),
}
res = dq_report(df, checks)
assert res["id_not_null"] == 0, res
```
One scan computes all checks. Add uniqueness (`count` vs `countDistinct`), referential integrity (left anti join), freshness (max timestamp lag), and volume anomaly (compare against trailing average). **Principal add:** severity levels (block vs warn), results written to a metrics table, and owners notified. Tools: Great Expectations, Deequ/PyDeequ, Soda, dbt tests.

### P28. Unit-test PySpark transformations with pytest.
```python
# conftest.py
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope="session")
def spark():
    s = (SparkSession.builder.master("local[2]")
         .config("spark.sql.shuffle.partitions", "2")        # small for speed
         .config("spark.ui.enabled", "false").getOrCreate())
    yield s
    s.stop()

# test_transform.py
from pyspark.testing import assertDataFrameEqual              # 3.5+
def test_dedup_latest(spark):
    src = spark.createDataFrame([(1, "a", 1), (1, "b", 2)], "id int, v string, ts int")
    out = latest_by_key(src, "id", "ts")
    assertDataFrameEqual(out, spark.createDataFrame([(1, "b", 2)], "id int, v string, ts int"))
```
Design for testability: pure functions `DataFrame → DataFrame`, I/O at the edges, session injected. Test edge cases (NULLs, ties, empty input, schema drift). Keep a handful of end-to-end tests on small real samples.

### P29. Read an execution plan and find the problem.
```python
df.explain("formatted")
```
What to look for: `Exchange` (shuffle) nodes and their count; `BroadcastHashJoin` vs `SortMergeJoin`; `PushedFilters` / `PartitionFilters` in `FileScan` (are your filters pushed? is `PartitionFilters: []` when it should prune?); `BatchEvalPython` / `ArrowEvalPython` (UDFs); `CartesianProduct`; `Sort` on huge data; `Generate` (explode) blowups; `AdaptiveSparkPlan isFinalPlan=true` for the post-AQE plan (view in the SQL tab). Example fix: a filter wrapped in a function, `WHERE to_date(ts) = '2026-11-16'`, defeats pruning; write it as a range: `ts >= '2026-11-16' AND ts < '2026-11-17'`.

### P30. End-to-end medallion pipeline skeleton: parameterized, idempotent, observable.
```python
import argparse, logging, time
log = logging.getLogger("pipeline")

def bronze(spark, run_date):
    return (spark.read.schema(SCHEMA).json(f"s3://lake/raw/orders/dt={run_date}/")
            .withColumn("_ingest_ts", F.current_timestamp())
            .withColumn("_src_file", F.input_file_name()))

def silver(df):
    df = df.filter("order_id IS NOT NULL").withColumn("order_date", F.to_date("order_ts"))
    w = Window.partitionBy("order_id").orderBy(F.col("updated_at").desc())
    return df.withColumn("rn", F.row_number().over(w)).filter("rn = 1").drop("rn")

def gold(df):
    return df.groupBy("order_date", "country").agg(F.sum("amount").alias("revenue"),
                                                    F.countDistinct("customer_id").alias("buyers"))

def main():
    p = argparse.ArgumentParser(); p.add_argument("--run-date", required=True); a = p.parse_args()
    spark = SparkSession.builder.appName(f"orders-{a.run_date}").getOrCreate()
    spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")
    t0 = time.time()
    b = bronze(spark, a.run_date).cache()
    metrics = {"bronze_rows": b.count()}
    s = silver(b)
    # idempotent: replace only this run's partition (or MERGE for table formats)
    (s.write.mode("overwrite").partitionBy("order_date").format("delta").saveAsTable("silver.orders"))
    gold(spark.table("silver.orders").filter(F.col("order_date") == a.run_date)) \
        .write.mode("overwrite").format("delta") \
        .option("replaceWhere", f"order_date = '{a.run_date}'").saveAsTable("gold.daily_revenue")
    metrics["duration_s"] = round(time.time() - t0, 1)
    log.info("run_metrics=%s", metrics)        # also push to your metrics system
    b.unpersist()

if __name__ == "__main__":
    main()
```
Principal checklist this demonstrates: **rerun safety** (same `run_date` gives same result via dynamic overwrite/`replaceWhere`), **lineage columns**, **separation of layer logic into testable functions**, **parameterization by logical date**, and **metrics emission**. In an interview, add: how it's scheduled (Airflow/Workflows with retries, SLAs, backfill via date ranges), data contracts at the bronze→silver boundary, DQ gates before gold publish, and cost controls (right-sized cluster, autoscaling, compaction schedule).

---

## Scorecard: self-evaluate per practical
| Dimension | Senior | Staff | Principal |
|---|---|---|---|
| Correct, runnable solution | ✔ | ✔ | ✔ |
| Explains plan impact (shuffle, pushdown, skew) | some | ✔ | ✔ |
| Idempotency, replay, late/out-of-order data | – | ✔ | ✔ |
| Testing and observability | – | some | ✔ |
| Cost, scale path, alternatives | – | some | ✔ |
| Platform standardization and team enablement | – | – | ✔ |
