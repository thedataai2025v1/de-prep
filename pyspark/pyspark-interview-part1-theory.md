# PySpark for Principal Data Engineer Interviews — Part 1: Theory (T1–T30)

**How to use:** Answer aloud first (2 min), then compare. Each entry: core answer, **Principal signal** (what lifts it above a Senior answer), and a likely follow-up. Defaults quoted are for Spark 3.x/4.x open source; Databricks and managed platforms may change them, so verify against the docs of your version.

---

## A. Architecture and execution model

### T1. Explain Spark architecture: driver, executors, cluster manager, application, job, stage, task.
**Answer.** The **driver** runs your `main`, builds the logical plan, the DAG, and schedules tasks. **Executors** are JVM processes (plus Python workers for PySpark) that run tasks and hold cached/shuffle data. A **cluster manager** (YARN, Kubernetes, standalone, or Databricks's own) allocates resources. Hierarchy: **application** → **jobs** (one per action) → **stages** (split at shuffle boundaries) → **tasks** (one per partition per stage). The DAGScheduler cuts stages; the TaskScheduler launches tasks on executors, with locality preferences.

**Principal signal.** Map symptoms to layers: driver OOM (`collect`, huge broadcast, too many partitions' metadata), executor OOM (skew, too-large partitions), scheduler delay (too many tiny tasks), cluster-level (resource starvation).
**Follow-up.** "How many tasks does a stage have?" (number of partitions of its final RDD/data.)

**Interview example (use this to ground your answer):**

Say: "Take our daily orders job: `orders.groupBy('country').count().write.parquet(...)`. The `write` is the one action, so it is one job. The groupBy shuffle splits it into two stages. If it reads 400 files of ~128 MB, stage 1 has ~400 tasks; stage 2 has `spark.sql.shuffle.partitions` = 200 tasks." Then add a debugging story: "Once the *driver* OOM'd. It wasn't executors. A developer had called `collect()` on 50M rows." That shows you map symptoms to the right layer.

### T2. What is lazy evaluation and why does Spark use it?
**Answer.** Transformations (`select`, `filter`, `join`, `groupBy`) only build a plan; **actions** (`count`, `collect`, `write`, `show`) trigger execution. Laziness lets Catalyst see the *whole* plan to prune columns, push filters, reorder joins and fuse operators into one pass, and it gives lineage for fault recovery.

Gotchas: `df.cache()` is lazy too (nothing cached until an action); every action recomputes the lineage unless cached; non-deterministic functions (`rand()`, `current_timestamp()`, UDFs with side effects) can produce different results per recomputation.

**Interview example (use this to ground your answer):**

```python
df = spark.read.parquet(p)                 # nothing runs
f  = df.filter("amt > 100").select("id")   # nothing runs
f.count()                                  # job 1: scan + filter + count
f.write.parquet(out)                       # job 2: scans AGAIN unless f is cached
```
Non-determinism trap: `d = df.withColumn("r", F.rand()); d.count(); d.show()` can show different `r` values each action, because the lineage is recomputed. Say: "I cache or checkpoint when I need a stable value."

### T3. Narrow vs wide transformations, and what is a shuffle?
**Answer.** Narrow: each output partition depends on one input partition (`map`, `filter`, `select`, `coalesce`) and is pipelined in one stage. Wide: output partitions depend on many inputs (`groupBy`, `join` (non-broadcast), `distinct`, `repartition`, `orderBy`), requiring a **shuffle**: map-side write to local disk (sorted, partitioned files), network fetch by reducers. A shuffle is the dominant cost: disk I/O, serialization, network, and a stage barrier.

**Principal signal.** Most tuning is *shuffle avoidance or shuffle reduction*: broadcast joins, pre-aggregation (partial aggregation is automatic for `groupBy` aggregates), bucketing for repeated joins, filtering early, and choosing partition counts well.

**Interview example (use this to ground your answer):**

Narrow: `orders.filter("amt > 0").select("id", "amt")` runs as one pipelined stage, with no data movement. Wide: `orders.groupBy("customer_id").sum("amt")` shows `Exchange hashpartitioning(customer_id, 200)` in `explain()`, i.e. a shuffle and a new stage. Illustrative optimization story: "Filtering `orders` to the last 7 days *before* joining to `customers` cut the shuffle write from terabytes to tens of GB, because the Exchange moved only surviving rows." (Use your own real numbers if you have them.)

### T4. RDD vs DataFrame vs Dataset. Which do you use in PySpark and why?
**Answer.** RDD: low-level, untyped JVM objects (Python objects pickled), no optimizer. DataFrame: schema'd, columnar-ish, optimized by Catalyst/Tungsten. Dataset (typed) exists only in Scala/Java. In PySpark use **DataFrames/Spark SQL** for essentially everything. RDD only for truly unstructured/custom partitioning logic, and RDD operations with Python lambdas pay serialization cost per record.

**Interview example (use this to ground your answer):**

Same word-count-style task, two ways:
```python
# RDD: every row is pickled to a Python worker, the optimizer can't help
rdd.map(lambda r: (r.country, 1)).reduceByKey(lambda a, b: a + b)

# DataFrame: stays in the JVM, Catalyst + Tungsten optimize it
df.groupBy("country").count()
```
Say: "The DataFrame version runs inside the JVM with codegen; the RDD version pays Python serialization per record. I only drop to RDDs for something like custom partitioning of raw binary data.

### T5. Explain the Catalyst optimizer phases.
**Answer.** (1) Parse SQL/DataFrame ops → **unresolved logical plan**. (2) **Analysis**: resolve names/types via the catalog → resolved logical plan. (3) **Logical optimization** (rule-based): predicate pushdown, column pruning, constant folding, join reordering (with CBO if statistics exist), subquery rewrites. (4) **Physical planning**: generate candidate physical plans, pick strategies (e.g. join algorithm) with a cost model. (5) **Code generation** (whole-stage codegen). Inspect with `df.explain(mode="extended"|"formatted"|"cost")`.

**Principal signal.** A Python UDF is opaque to Catalyst: it blocks pushdown through it and prevents codegen fusion. That is a *design-level* reason to prefer built-ins.

**Interview example (use this to ground your answer):**

```python
spark.read.parquet("events").select("user_id", "amount") \
     .filter("dt = '2026-11-16'").explain("formatted")
```
In the plan you can point to: `ReadSchema: struct<user_id,amount>` (column pruning, though the table has ~80 columns), `PartitionFilters: [dt = 2026-11-16]` (only one directory read), and `PushedFilters`. Contrast: a filter written as a Python UDF doesn't appear in `PushedFilters` because Catalyst can't see inside it.

### T6. What is Tungsten and whole-stage code generation?
**Answer.** Tungsten: off-heap/binary memory format (`UnsafeRow`), cache-aware computation, and avoiding JVM object overhead/GC. **Whole-stage codegen** fuses a chain of operators (filter→project→aggregate) into a single generated Java function with tight loops (`WholeStageCodegen` nodes in `explain`, asterisk-marked). Vectorized Parquet/ORC readers deliver columnar batches. Operators that break codegen (some UDFs, complex types, too-wide expressions) fall back to slower paths.

**Interview example (use this to ground your answer):**

```python
df.filter("x > 1").selectExpr("x * 2 AS y").agg(F.sum("y")).explain()
```
In the plan, the operators `Filter`, `Project`, `HashAggregate` appear under one `*(1)` marker, meaning they were fused into a single generated Java function (whole-stage codegen). Insert a Python UDF in the middle and you get a `BatchEvalPython` node that breaks the fusion and forces row transfer to Python. That visual is a good thing to describe in an interview.

### T7. How does PySpark work under the hood (Py4J, Python workers, Arrow)?
**Answer.** The Python driver talks to the JVM driver through **Py4J** (a gateway for method calls). DataFrame API calls just build JVM plans, so pure-DataFrame PySpark performs near Scala. Only when Python code must execute on executors (UDFs, `rdd.map`, `mapInPandas`) does Spark launch **Python worker processes** per executor and ship data JVM→Python and back. Classic UDF: pickled rows. Arrow-enabled paths (`pandas_udf`, `toPandas` with Arrow): columnar Arrow batches.

**Principal signal.** Memory: Python workers live *outside* the JVM heap. Size `spark.executor.memoryOverhead` and `spark.executor.pyspark.memory` accordingly, or containers get OOM-killed by YARN/K8s, with no Java OOM in the logs.

**Interview example (use this to ground your answer):**

Contrast two lines:
```python
df.withColumn("n", F.upper("name"))                      # JVM only, no Python worker started
df.withColumn("n", F.udf(lambda s: s.upper(), "string")("name"))  # rows shipped to Python and back
```
Production story to tell: "Executors were being OOMKilled on Kubernetes (exit code 137) but the Java heap looked fine. The culprit was Python worker memory from a pandas UDF, which sits outside the JVM heap. We raised `spark.executor.memoryOverhead` and `spark.executor.pyspark.memory`.

### T8. Python UDF vs pandas UDF vs built-in function vs SQL expression: rank and justify.
**Answer.** Fastest to slowest generally: **built-in functions / SQL expressions** (JVM, codegen, optimizable) > **Scala/Java UDF** > **pandas (Arrow) UDF** > **plain Python UDF** (row-at-a-time pickling). pandas UDF types: scalar (Series→Series), grouped map (`applyInPandas`), cogrouped, `mapInPandas`, Series→scalar aggregate. 3.5 adds Arrow-optimized Python UDFs (`useArrow=True`) and Python UDTFs.

Pitfalls: pandas UDFs see *batches*, not whole partitions or groups (unless grouped), so stats computed inside are per-batch. UDFs must handle nulls; non-determinism requires `asNondeterministic()`; exceptions inside UDFs produce opaque stack traces.

**Interview example (use this to ground your answer):**

Use case where a built-in can't do the job: validating phone numbers with the Python `phonenumbers` library.
```python
@pandas_udf("boolean")
def valid_phone(s: pd.Series) -> pd.Series:
    return s.map(lambda x: phonenumbers.is_valid_number(phonenumbers.parse(x, "IN")))
```
Say: "I'd first try `regexp_extract`/SQL expressions. Only if I truly need the library do I use a pandas UDF, because it moves data in Arrow batches instead of pickling each row." Per-batch trap: a z-score `(s - s.mean())/s.std()` inside a scalar pandas UDF uses *each batch's* mean, not the global mean.

### T9. How are partitions determined, and what are `repartition`, `coalesce`, and `spark.sql.shuffle.partitions`?
**Answer.** Input partitions come from file splits (`spark.sql.files.maxPartitionBytes`, default 128 MB, plus `openCostInBytes`). Post-shuffle partitions come from `spark.sql.shuffle.partitions` (default 200) unless AQE coalesces them. `repartition(n, cols)` does a full shuffle (hash partitioning or round-robin); `repartitionByRange` samples for ranges; `coalesce(n)` merges partitions *without* a full shuffle (can only reduce, and can cause skew or reduce upstream parallelism if pushed into earlier stages).

Rule of thumb: target roughly **100–200 MB per shuffle partition**, and at least 2–4x as many tasks as total cores. Too few → spills/OOM; too many → scheduler overhead and tiny files.

**Interview example (use this to ground your answer):**

Illustrative arithmetic: 1 TB of input / 128 MB ≈ 8,000 input partitions. After a join, `shuffle.partitions = 200` would mean ~5 GB per partition, which spills or OOMs, so raise it (~8,000 for 1 TB at ~128 MB) or let AQE decide. `coalesce(10)` placed before the final write on a 500-task stage can drag the *whole stage* down to 10 tasks of parallelism (no shuffle in between), so use `repartition(10)` when you need a shuffle boundary there.

### T10. Explain Spark's join strategies and how the planner chooses.
**Answer.**
- **Broadcast hash join (BHJ):** one side broadcast to all executors; no shuffle. Chosen if the side ≤ `spark.sql.autoBroadcastJoinThreshold` (10 MB default), or hinted (`broadcast()`).
- **Sort-merge join (SMJ):** shuffle both sides on the join key, sort, merge. Default for large equi-joins; scalable but expensive.
- **Shuffle hash join (SHJ):** shuffle and build a hash table for the smaller side per partition; used when SMJ is disfavored or hinted.
- **Broadcast nested loop / cartesian:** non-equi joins or no condition. Dangerous.

Join types influence choice (a full outer join cannot use plain BHJ). Hints: `BROADCAST`, `MERGE`, `SHUFFLE_HASH`, `SHUFFLE_REPLICATE_NL`. AQE can convert SMJ→BHJ at runtime using actual stage sizes.

**Interview example (use this to ground your answer):**

Scenario ladder:
- `fact_sales` (2 TB) join `dim_store` (50 MB): broadcast hash join, no shuffle of the 2 TB side.
- `fact_sales` join `fact_returns` (500 GB): sort-merge join, so you'll see two `Exchange` nodes plus `Sort`.
- `ON a.ts BETWEEN b.start AND b.end`: non-equi join, so `BroadcastNestedLoopJoin` or a cartesian. A red flag.
Hint syntax: `SELECT /*+ BROADCAST(d) */ ... FROM f JOIN d ON ...`

### T11. What is Adaptive Query Execution (AQE) and what does it fix?
**Answer.** Enabled by default since 3.2 (`spark.sql.adaptive.enabled`). At shuffle boundaries Spark re-optimizes with *runtime statistics*: (1) **coalesces small shuffle partitions**, (2) **converts SMJ to BHJ** when a side turns out small, (3) **splits skewed partitions** in sort-merge joins (`skewJoin.enabled`, a partition counts as skewed if > `skewedPartitionFactor` (5) × median and > `skewedPartitionThresholdInBytes` (256 MB)), (4) dynamic partition-pruning related improvements. Limits: only helps at shuffle stages, doesn't fix skew in aggregations as well as in joins, and can't rescue a fundamentally bad design.

**Interview example (use this to ground your answer):**

Example to quote: "We set `shuffle.partitions` to 2000 for a big job. After a selective filter only ~15 GB reached the next shuffle, so AQE coalesced 2000 tiny partitions into ~120, which cut task-scheduling overhead. In another job one country key held ~40 GB while the median partition was ~200 MB; AQE split that skewed partition into sub-partitions at runtime." Where to see it: the SQL tab shows `AQEShuffleRead` (coalesced / skewed) and `isFinalPlan=true`.

### T12. Diagnose and fix data skew.
**Symptoms (Spark UI):** a few tasks in a stage take far longer than the median, with large shuffle read, spill, or OOM; max vs median task duration/size. 
**Remedies, in order:** (1) enable AQE skew join; (2) filter or handle the hot key separately (e.g., null or default keys: process them via a separate path); (3) broadcast the small side; (4) **salting**: add a random salt to the skewed side and replicate the other side; for aggregations do two-phase (salted partial agg, then final); (5) increase parallelism; (6) fix upstream (bad default keys, null-key explosions).

**Principal signal.** Quantify the hot key's cardinality and share of rows first (`groupBy(key).count().orderBy(desc)`); don't salt blindly. Also mention NULL-key skew in joins as a very common cause.

**Interview example (use this to ground your answer):**

Concrete cause to cite: joining `events` to `users` on `user_id` where ~30% of events have `user_id IS NULL` or the default `'unknown'`. All those rows hash to one reducer: one task runs ~40 min while the median is ~20 s. Fix sequence: (1) confirm with `events.groupBy("user_id").count().orderBy(F.desc("count")).show(5)`; (2) pull NULL/'unknown' rows out of the join (they can't match anyway), join the rest, then `unionByName` them back; (3) enable AQE skew join; (4) only then salt the remaining hot keys (see practical P14).

### T13. Describe Spark's memory management and the typical OOM causes.
**Answer.** Executor JVM heap (`spark.executor.memory`) splits: ~300 MB reserved; **unified region** = `spark.memory.fraction` (0.6) of the remainder, shared dynamically by execution (shuffles, joins, sorts, aggregations) and storage (cache), with `storageFraction` (0.5) as the *eviction-protected* part of storage; the rest is user memory. Off-heap optional. Outside the heap: `memoryOverhead` (default max(10% of executor memory, 384 MB) on YARN; on Kubernetes the non-JVM factor defaults to 0.4 for PySpark apps), which holds Python workers, native libs, and Netty buffers.

**OOM causes:** skewed or huge partitions; `collect()`/`toPandas()` on the driver; oversized broadcast; Python worker memory (UDF/pandas); too many cores per executor sharing one heap (concurrent task memory pressure); giant single rows/arrays; container kill from overhead.

**Interview example (use this to ground your answer):**

Worked example: executor 8 GB, 4 cores. Reserved 300 MB, so (8192 - 300) × 0.6 ≈ 4.7 GB is the unified execution+storage region, shared by 4 concurrent tasks (~1.2 GB each at most under contention). Container size = 8 GB + overhead (≥ ~800 MB). If YARN reports "Container killed for exceeding memory limits" but there's no Java OOM, the culprit is overhead (Python workers/native), so raise `memoryOverhead` rather than heap.

### T14. How does caching/persistence work and when does it hurt?
**Answer.** `cache()` = `persist(MEMORY_AND_DISK)` for DataFrames (deserialized in-memory columnar format, spilling to disk). Levels: `MEMORY_ONLY`, `MEMORY_AND_DISK`, `DISK_ONLY`, serialized variants, `_2` replicated, `OFF_HEAP`. Use when a DataFrame is reused by **multiple actions** and recomputation is costly. It hurts when used once, when data is bigger than cluster memory (evictions, thrash), or when it just steals execution memory. Always `unpersist()` when done. Note that cached data doesn't survive executor loss reliably, and caching before a filter pushdown can defeat pruning.

**Interview example (use this to ground your answer):**

```python
feats = build_features(raw).cache()
feats.count()                      # materialize once
train(feats); report_a(feats); report_b(feats)   # 3 reuses: cache pays off
feats.unpersist()
```
Counter-example: caching a 3 TB DataFrame that's used once. It evicts blocks, spills to disk and runs *slower* than not caching. Say you cache only when (reuse count > 1) and (recompute cost > memory cost).

### T15. Explain shuffle internals: sort-based shuffle, spill, external shuffle service.
**Answer.** Each map task writes a single sorted-by-partition data file plus an index file (sort-based shuffle, with bypass-merge and unsafe shuffle writer optimizations). Reducers fetch their segments over the network (Netty), merge, then aggregate/sort, possibly **spilling** to disk when execution memory is short. The **external shuffle service** (or Kubernetes shuffle tracking / remote shuffle storage) lets executors be removed under dynamic allocation without losing their shuffle files. `FetchFailedException` means a reducer couldn't fetch map output (executor lost, disk failure, network), and the stage and its parent map stage are retried.

**Interview example (use this to ground your answer):**

Spark UI reading: "Stage 9: Shuffle Write 400 GB; Spill (Disk) 120 GB" means reducers lacked execution memory, so raise partition count or memory. Failure story: after a spot node was reclaimed, reducers hit `FetchFailedException: Failed to connect to executor-12`. The lost map outputs forced the parent stage to be recomputed. External shuffle service / remote shuffle storage avoids losing those files when executors disappear.

### T16. How do you use the Spark UI to find bottlenecks?
**Answer.** *Jobs/Stages:* duration, task count, input/shuffle read/write sizes, spill (memory/disk), GC time, **task-duration distribution** (min/median/max → skew), scheduler delay. *SQL tab:* the physical plan with metrics per operator: rows output, scan time, files read, partitions read vs pruned, join type chosen, AQE final plan. *Executors:* GC %, failed tasks, dead executors, shuffle read/write, memory use. *Storage:* cached data fractions. *Environment:* effective configs.

Method: find the slowest stage → check skew/spill/GC → look at the plan node responsible → hypothesize → change one thing → re-measure.

**Interview example (use this to ground your answer):**

Model answer as a walk-through: "A job took 3 h. Stages tab: stage 14 took 2.5 h. Summary metrics showed median task 30 s but max 2.4 h and max shuffle read ~80 GB vs ~200 MB median, so skew. SQL tab showed a sort-merge join on a key with a hot value. After enabling AQE skew handling and isolating the NULL key, the job dropped to ~25 min." (Illustrative; substitute a real case of yours.) The structure (slowest stage → distribution → plan node → one change → re-measure) is what interviewers want.

### T17. How do file formats and pushdown affect performance (Parquet/ORC vs CSV/JSON)?
**Answer.** Parquet/ORC are columnar and compressed, and they carry min/max statistics per row group/stripe, enabling **column pruning** and **predicate pushdown** (skip row groups) plus vectorized reads. CSV/JSON are row-oriented text: full parse, schema inference cost (extra pass), no pushdown. Partition pruning (directory-based) and dynamic partition pruning (join-driven) skip files entirely. Pitfalls: pushdown doesn't apply through UDFs or casts on the column; nested column pruning is limited without `spark.sql.optimizer.nestedSchemaPruning.enabled`; schema merging (`mergeSchema`) is expensive.

**Interview example (use this to ground your answer):**

Query: `SELECT user_id, amount FROM events WHERE dt = '2026-11-16' AND amount > 1000` on a 200-column Parquet table reads only 2 columns, one partition directory, and skips row groups whose `amount` max ≤ 1000. The same table as CSV would parse every column of every row. Anti-patterns that kill pruning: `WHERE to_date(ts) = '2026-11-16'` (use a range on `ts`), or `WHERE CAST(amount AS STRING) = '5'`.

### T18. What is the small files problem and how do you handle it?
**Answer.** Thousands of tiny files multiply metadata ops, task counts, and S3 list/open latency; the NameNode or object-store listing becomes the bottleneck, and the planner spawns far too many tasks. Causes: over-partitioned writes, high-cardinality partition columns, streaming micro-batches, many shuffle partitions at write time. Fixes: control output partitioning before write (`repartition`/`coalesce`, `maxRecordsPerFile`, `spark.sql.adaptive.coalescePartitions`), choose partition columns with low-to-moderate cardinality, **compaction jobs** (Delta `OPTIMIZE`, Iceberg `rewrite_data_files`, Hudi clustering), and for streaming use larger triggers or auto-compaction. Target file sizes ~128 MB–1 GB.

**Interview example (use this to ground your answer):**

Illustrative: a streaming job with a 30 s trigger writing 200 files per batch produces 2,880 × 200 ≈ 576k small files per day; later reads spend more time listing and opening files than reading. Fix: reduce write parallelism (`repartition(8)` before the sink), enable auto-compaction, and run a nightly `OPTIMIZE` (Delta) / `rewrite_data_files` (Iceberg) on recent partitions, aiming for ~128 MB-1 GB files.

### T19. Partitioning vs bucketing vs clustering (Z-order/liquid) on disk.
**Answer.** **Partitioning** (directories by column) enables partition pruning, so use it for low-cardinality, commonly filtered columns (date). Over-partitioning creates small files. **Bucketing** pre-hashes data into a fixed number of files by key so repeated joins/aggregations on that key can skip shuffles (needs matching bucket counts and a Hive-metastore table). **Data clustering/skipping** (Delta Z-order, liquid clustering, Iceberg sort order and partition evolution) co-locates related values within files for min/max skipping on multiple columns, without directory explosion.

**Interview example (use this to ground your answer):**

Design choices on one `events` table: partition by `event_date` (about 365 directories per year, fine) but never by `user_id` (millions of directories). For repeated joins `orders` ⋈ `customers`, bucket both by `customer_id` into 64 buckets and the `Exchange` disappears from the plan. For queries filtering on `(user_id, country)` use Z-order or liquid clustering so files are co-located without extra directories.

### T20. Explain the Structured Streaming model: micro-batch, triggers, output modes.
**Answer.** A stream is an unbounded table; a query incrementally processes new rows. Default **micro-batch** engine: each trigger plans a batch from source offsets, executes, commits offsets/state to the checkpoint. Triggers: default (ASAP), `processingTime`, `availableNow` (process all available data in multiple batches, then stop; replaces `once`), experimental `continuous`. **Output modes:** `append` (only finalized rows), `update` (changed rows), `complete` (whole result table, for small aggregates). Which modes are legal depends on the query (aggregations without watermark can't use `append`).

**Interview example (use this to ground your answer):**

Same stream, different modes:
```python
# running count per country: changes over time
counts.writeStream.outputMode("update")...     # or complete for a small result table
# filtered raw events: rows never change after emitted
clean.writeStream.outputMode("append")...
# nightly catch-up job instead of an always-on cluster
.trigger(availableNow=True)
```
Say: "`availableNow` gave us streaming semantics (checkpointed, exactly-once) with batch economics.

### T21. Watermarks, late data, and state management.
**Answer.** Watermark = `max(event_time seen) − threshold`; events older than it may be dropped, and it lets Spark **purge state** for windows/keys that can no longer change. Without a watermark, state for aggregations, dedup, and stream-stream joins grows forever. In `append` mode a windowed aggregate emits only after the watermark passes the window end (so latency ≈ window + delay). Watermark advances per batch and is the *minimum* across multiple input streams by default (`spark.sql.streaming.multipleWatermarkPolicy`). For large state use the **RocksDB state store** (instead of heap-based HDFS-backed) to avoid GC pauses. Dedup: `dropDuplicates` / `dropDuplicatesWithinWatermark` (3.5+).

**Interview example (use this to ground your answer):**

```python
events.withWatermark("event_ts", "10 minutes") \
      .groupBy(F.window("event_ts", "5 minutes")).count()
```
Timeline: the 10:00-10:05 window is finalized (emitted in append mode) once the max event time seen reaches 10:15. An event stamped 10:03 that arrives when max event time is already 10:20 is dropped. Say: "I size the watermark from measured lateness percentiles (e.g., p99.9 lag) and monitor dropped-row metrics.

### T22. How do you get exactly-once semantics with Structured Streaming?
**Answer.** Three parts: **replayable source** (Kafka offsets, file listing), **checkpointing** (offsets + state, written *before* processing via write-ahead log, committed after), and **idempotent or transactional sink**. The engine guarantees each batch's offset range is deterministic on restart; the sink must tolerate a replayed batch: file sinks via metadata log, Delta via transaction + `txnAppId/txnVersion` for `foreachBatch`, or upsert/MERGE keyed on a business key. End-to-end exactly-once is "at-least-once + idempotence". Never change query logic that alters state schema or checkpoint-incompatible parts without a migration plan.

**Interview example (use this to ground your answer):**

Failure walk-through: batch covers Kafka offsets 100-199. The job writes to Delta, then crashes before committing the batch to the checkpoint. On restart Spark replays offsets 100-199 (same range, deterministic). If the sink is a plain append you get duplicates. Make the sink idempotent: Delta `txnAppId`/`txnVersion=batch_id` in `foreachBatch`, or `MERGE` on `(topic, partition, offset)` or a business key. Summarize as: replayable source + checkpoint + idempotent sink.

### T23. How does Spark integrate with Kafka? Offsets, partitions, and failure behaviors.
**Answer.** Kafka source: each Kafka topic-partition maps to ≥1 Spark input partition (`minPartitions` can split). Offsets are tracked in the Spark *checkpoint*, not committed to Kafka consumer groups (the `group.id` is not used for progress). Options: `startingOffsets`, `maxOffsetsPerTrigger` (rate control), `failOnDataLoss` (data aged out of retention → fail or continue), `kafka.*` client configs, `includeHeaders`. Values arrive as binary → decode with `cast("string")`, `from_json`, `from_avro` (with Schema Registry handled by a library). Consumer lag is monitored from `StreamingQueryProgress` (`endOffset` vs Kafka log end).

**Interview example (use this to ground your answer):**

Example numbers: topic `orders` has 12 partitions, so you get at least 12 Spark input tasks. `maxOffsetsPerTrigger=600000` caps each micro-batch at 600k records (~50k per partition). If the job is down 10 days and topic retention is 7 days, `failOnDataLoss=true` makes the query fail loudly instead of silently skipping data. Also mention that Kafka consumer-group lag tools won't show Spark's progress, because offsets live in Spark's checkpoint; track lag from `StreamingQueryProgress`.

### T24. How do table formats (Delta, Iceberg, Hudi) change what Spark can do?
**Answer.** They add **ACID transactions** over object storage via a metadata/transaction log, snapshot isolation, **MERGE/UPDATE/DELETE**, schema evolution, **time travel**, file-level statistics for data skipping, and compaction/maintenance (`OPTIMIZE`/`VACUUM`, `rewrite_data_files`/`expire_snapshots`). Concurrency uses optimistic concurrency control: writers conflict if they touch overlapping files, and retries are required. Trade-offs: metadata growth, small-file maintenance, vendor/format lock-in, and MERGE cost (copy-on-write rewrites files; merge-on-read in Iceberg/Hudi defers cost to reads).

**Interview example (use this to ground your answer):**

Concrete uses: GDPR erasure `DELETE FROM customers WHERE id = 42` (ACID on object storage); debug a bad load with `SELECT * FROM t VERSION AS OF 41` and compare to the current version; `MERGE` for upserts. Concurrency story: two jobs `MERGE`-ing into overlapping files, one fails with a concurrent-modification error and retries. Note that `VACUUM` removes old files, so time travel and privacy deletion windows depend on retention settings.

### T25. How does Spark handle failures? Lineage, retries, speculation.
**Answer.** **Task failure:** retried up to `spark.task.maxFailures` (4). **Executor loss:** tasks rescheduled; lost shuffle output triggers upstream stage recomputation via lineage. **Stage failure:** `spark.stage.maxConsecutiveAttempts`. **Driver failure:** app dies unless the cluster manager restarts it (`--supervise`, K8s, Databricks job retries), and streaming recovers from the checkpoint. **Speculation** (`spark.speculation`) re-launches slow tasks, which is risky with non-idempotent side effects. **Output commit:** with the classic committer on S3, rename isn't atomic, so use S3A committers/Magic committer or table formats.

**Interview example (use this to ground your answer):**

Scenario: a spot instance disappears during stage 3. Tasks on it are rerun (`spark.task.maxFailures=4`), and lost shuffle files force the parent stage to be partially recomputed via lineage. Pitfall to mention: `spark.speculation=true` on a job that calls an external API will duplicate calls, so only use speculation when tasks are idempotent. On S3, rename-based committers aren't atomic, so use an S3A committer or a table format.

### T26. Dynamic allocation and cluster sizing: how do you choose executors, cores and memory?
**Answer.** Typical guidance: 4–5 cores per executor (HDFS throughput/GC trade-off; on object storage you can go to ~5–8), leave 1 core and ~1 GB per node for OS/daemons, executor memory sized so `memory × (1 − overhead share)` fits node RAM, and enough overhead for Python. Dynamic allocation (`spark.dynamicAllocation.*`) scales executors with pending tasks, and it requires an external shuffle service or shuffle tracking. Principal view: start from **data volume per stage and target runtime**, not folklore. Profile one run, then size by shuffle bytes and partition counts, and favor spot/preemptible capacity with shuffle-aware fault tolerance for cost.

**Interview example (use this to ground your answer):**

Worked sizing: nodes with 16 cores / 64 GB. Reserve 1 core + 1 GB for OS: 15 cores, 63 GB. With 5 cores per executor you get 3 executors per node, ~21 GB each; subtract ~10% overhead (more for PySpark), so about `--executor-cores 5 --executor-memory 18g --conf spark.executor.memoryOverhead=3g`. Then validate with data: if the heavy stage shuffles 600 GB at ~200 MB per partition, you need ~3,000 partitions, so ~150 cores gives 20 waves of tasks. Say: "I derive sizing from the data and target runtime, then measure.

### T27. Serialization, closures, broadcast variables and accumulators.
**Answer.** JVM-side serialization defaults to Java; **Kryo** is faster/smaller (mostly relevant to RDDs; DataFrames use Tungsten binary). Python closures are pickled (cloudpickle) and shipped to workers, so capturing a big object (model, dict, DataFrame, SparkSession) is expensive or fails. Use **broadcast variables** for read-only lookups to avoid per-task copies. **Accumulators** are write-only for tasks and only reliable in *actions*; in transformations they can double-count on retries and recomputation.

**Interview example (use this to ground your answer):**

```python
mapping = load_big_dict()                  # ~20 MB
b = spark.sparkContext.broadcast(mapping)  # shipped once per executor
@F.udf("string")
def lookup(k): return b.value.get(k)       # vs capturing `mapping` directly in the closure
```
Accumulator trap: incrementing `acc.add(1)` inside a UDF used in `withColumn` can double count if a task retries or the plan is recomputed, so use accumulators only inside actions like `foreach`, or compute counts with DataFrame aggregations.

### T28. Spark SQL semantics that bite: nulls, ANSI mode, type coercion, timestamps.
**Answer.** SQL three-valued logic: `NULL = NULL` → NULL (use `<=>` / `eqNullSafe`); `NOT IN` with a NULL in the list yields no rows; null keys never match in equi-joins. Spark 4.0 turns **ANSI mode on by default** (overflow and invalid casts raise errors rather than returning NULL); in 3.x the default is off, so invalid casts silently produce NULL, a data-quality hazard. Timestamps: `TimestampType` is session-timezone dependent (`spark.sql.session.timeZone`); 3.4+ adds `TimestampNTZType`. Decimal precision overflows, integer overflow wrap in non-ANSI mode, and `inferSchema` surprises are classic traps.

**Interview example (use this to ground your answer):**

```sql
SELECT * FROM a WHERE id NOT IN (SELECT id FROM b);   -- returns 0 rows if b.id contains any NULL
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.id = a.id);  -- safe
```
Casting: `CAST('abc' AS INT)` returns NULL under non-ANSI (Spark 3.x default) but raises an error under ANSI (default in 4.0). Null-safe join: `a.id <=> b.id`. Timestamp: the same `TimestampType` value prints differently if `spark.sql.session.timeZone` changes. Mention that you pin the timezone to UTC in jobs.

### T29. Deployment modes, Spark Connect, and PySpark environment management.
**Answer.** Client mode (driver on the submitting machine; fine for notebooks, bad for production over WAN) vs cluster mode (driver on the cluster). Managers: YARN, Kubernetes, standalone, and managed (Databricks, EMR, Dataproc, Glue). **Spark Connect** (3.4+) is a client–server protocol: a thin client sends unresolved plans over gRPC, which decouples client from server versions, improves multi-tenancy and stability (client crash ≠ driver crash), and has limitations (RDD API not supported, some UDF dependency packaging nuances). Python dependency shipping: `--py-files`, `--archives` venv-pack, conda-pack, Docker images, cluster libraries. Driver and executor Python minor versions must match.

**Interview example (use this to ground your answer):**

```bash
spark-submit --master yarn --deploy-mode cluster \
  --archives pyspark_venv.tar.gz#environment \
  --conf spark.pyspark.python=./environment/bin/python \
  --conf spark.pyspark.driver.python=./environment/bin/python job.py
```
Say: "Notebooks use client mode; production uses cluster mode so the driver isn't tied to a laptop. We pack the venv so driver and executors share the same Python minor version." Spark Connect: `SparkSession.builder.remote("sc://host:15002").getOrCreate()`, so a thin client talks to a shared server over gRPC.

### T30. Principal-level: your methodology for tuning a slow, expensive Spark platform.
**Answer (frame as a program, not a config hunt).**
1. **Measure and attribute:** collect per-job cost, runtime, shuffle bytes, spill, and failure rates (event logs, Spark History Server, platform metrics) and rank by cost × frequency; fix the top 10 jobs, which usually yield the majority of savings.
2. **Classify the bottleneck:** I/O (small files, no pruning), shuffle (join strategy, partition count), skew, memory/spill, UDFs, driver-side collection, or scheduling and cluster configuration.
3. **Apply the cheapest high-leverage fixes first:** predicate/partition pruning, file layout and compaction, broadcast joins, AQE, built-ins over UDFs, columnar formats, fewer wide stages, incremental instead of full reprocessing.
4. **Platform-level levers:** autoscaling policies, spot instances, right-sized instance families, Photon/vectorized engines where available, caching layers, table maintenance automation.
5. **Guardrails:** job templates with sane defaults, linting (anti-patterns: `collect`, cross joins, UDF abuse), cost budgets/alerts per team, regression benchmarks in CI.
6. **Communicate:** report savings in dollars and SLA terms to stakeholders.

**Common mistakes:** tuning configs before reading the plan, blanket `cache()`, increasing `shuffle.partitions` to "fix" everything, `repartition(1)` for output, ignoring data layout, and benchmarking on non-representative samples.

**Interview example (use this to ground your answer):**

Tell it as a STAR story with your own numbers: "On a platform I inherited, I pulled 30 days of event logs and found the top 15 jobs consumed ~70% of spend. Job A: a skewed join stage (AQE + hot-key isolation). Job B: reading ~18k small files (compaction + partition redesign). Job C: a Python UDF (replaced with built-ins; pandas UDF for the rest). Then I added guardrails: job templates, lint rules against `collect()`/cross joins, and per-team cost dashboards." (Replace the figures with real ones. Never state numbers you can't back up.)

---
**Next:** Part 2 has the 30 coding practicals.
