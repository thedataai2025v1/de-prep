# Python for Principal Data Engineer Interviews — Part 2: Code Practicals (Q26–50)

**How to use this:** Write the solution yourself first (timer: 10–15 min each), then compare. Every solution lists complexity and what a Principal candidate adds beyond "it works": edge cases, failure handling, scale and testability.

---

## Section D: Core algorithms and data structures

### Q26. Implement an LRU cache (O(1) get/put).
```python
from collections import OrderedDict
from threading import RLock

class LRUCache:
    def __init__(self, capacity: int):
        if capacity <= 0: raise ValueError("capacity must be > 0")
        self.cap, self._d, self._lock = capacity, OrderedDict(), RLock()

    def get(self, key, default=None):
        with self._lock:
            if key not in self._d: return default
            self._d.move_to_end(key)          # mark most recently used
            return self._d[key]

    def put(self, key, value):
        with self._lock:
            self._d[key] = value
            self._d.move_to_end(key)
            if len(self._d) > self.cap:
                self._d.popitem(last=False)   # evict least recently used
```
**Complexity:** O(1). **Principal adds:** thread safety, TTL, metrics (hit ratio), memory-based vs count-based eviction, and `functools.lru_cache` as the stdlib alternative (note: it caches on arguments, which must be hashable, and can leak memory when decorating methods because it holds `self`). **Follow-up:** implement it from scratch with a doubly linked list plus a dict.

### Q27. Write a retry decorator with exponential backoff and jitter.
```python
import functools, random, time

def retry(max_attempts=5, base=0.5, cap=30.0, retry_on=(Exception,)):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*a, **kw):
            for attempt in range(1, max_attempts + 1):
                try:
                    return fn(*a, **kw)
                except retry_on as e:
                    if attempt == max_attempts: raise
                    delay = random.uniform(0, min(cap, base * 2 ** attempt))  # full jitter
                    time.sleep(delay)
        return wrapper
    return deco

@retry(retry_on=(ConnectionError, TimeoutError))
def fetch(url): ...
```
**Principal adds:** full jitter prevents thundering herds; retry only *idempotent* operations and *transient* errors; respect `Retry-After`; overall deadline budget; circuit breaker; emit metrics; async variant. Libraries: `tenacity`.

### Q28. Read a multi-GB file in chunks and compute per-key aggregates with bounded memory.
```python
import csv
from collections import defaultdict

def aggregate(path, key="customer_id", val="amount"):
    totals = defaultdict(float)
    with open(path, newline="") as f:
        for row in csv.DictReader(f):         # streaming, O(1) per row
            try:
                totals[row[key]] += float(row[val])
            except (KeyError, ValueError):
                continue                       # in production: write to DLQ
    return totals
```
**Memory** is O(distinct keys), not O(rows). If keys are too many for RAM: partition by `hash(key) % N` into N temp files (use a stable hash, not `hash()`) and aggregate each, or use DuckDB/Spark. **Follow-up:** "What if the file is compressed or on S3?" (stream with `gzip.open`, `smart_open`, or range reads.)

### Q29. Merge k sorted streams lazily.
```python
import heapq
def merged(*iters):
    return heapq.merge(*iters)                # O(n log k), lazy
# Manual: push (value, idx, iterator) tuples onto a heap, pop min, push next from same iterator.
```
Use case: external sort, merging sorted shards or Kafka partitions by event time. Use `key=` for records. Note `heapq.merge` assumes each input is already sorted.

### Q30. Keep the latest record per key (dedup by `updated_at`), in pure Python and at scale.
```python
def latest_by_key(records, key="id", ts="updated_at"):
    best = {}
    for r in records:
        k = r[key]
        if k not in best or r[ts] > best[k][ts]:
            best[k] = r
    return list(best.values())
```
Tie-breaker: when timestamps are equal, use a monotonic sequence/LSN, otherwise results are nondeterministic. At scale (SQL/Spark): `ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC, lsn DESC) = 1`. This is the standard CDC "current state" reduction.

### Q31. Implement a token-bucket rate limiter.
```python
import time, threading

class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate, self.cap = rate, capacity
        self.tokens, self.ts = capacity, time.monotonic()
        self.lock = threading.Lock()

    def acquire(self, n=1):
        with self.lock:
            now = time.monotonic()
            self.tokens = min(self.cap, self.tokens + (now - self.ts) * self.rate)
            self.ts = now
            if self.tokens >= n:
                self.tokens -= n; return True
            return False
```
Use `time.monotonic()` (immune to clock changes). Bursts up to `capacity`, long-run average `rate`. For distributed limiting, use Redis (Lua script for atomicity). Compare with sliding window and leaky bucket.

### Q32. Group-by and top-K without pandas.
```python
from collections import Counter, defaultdict
import heapq, itertools

# top-K frequent: O(n log k) with heap, Counter.most_common does the same
def top_k(items, k): return Counter(items).most_common(k)

# group by then aggregate
def group_sum(rows):
    g = defaultdict(int)
    for k, v in rows: g[k] += v
    return dict(g)

# itertools.groupby REQUIRES input sorted by key, a classic gotcha
rows.sort(key=lambda r: r["k"])
for k, grp in itertools.groupby(rows, key=lambda r: r["k"]): ...
```
For streaming top-K on unbounded data use Count-Min Sketch + heap (approximate), or Space-Saving.

## Section E: Concurrency and parallelism

### Q33. Producer–consumer with threads and a bounded queue (backpressure).
```python
import queue, threading
q = queue.Queue(maxsize=1000)                 # bounded, so producer blocks when full
SENTINEL = object()

def producer(src):
    for item in src: q.put(item)              # blocks = backpressure
    for _ in range(N_WORKERS): q.put(SENTINEL)

def consumer():
    while (item := q.get()) is not SENTINEL:
        try: process(item)
        finally: q.task_done()

N_WORKERS = 4
ts = [threading.Thread(target=consumer) for _ in range(N_WORKERS)]
[t.start() for t in ts]; producer(source()); [t.join() for t in ts]
```
**Principal adds:** exception propagation from workers (a failed thread silently dies unless handled), graceful shutdown (signals), poison pills, bounded memory, and metrics on queue depth.

### Q34. Fetch 10,000 URLs concurrently with asyncio and a concurrency limit.
```python
import asyncio, aiohttp

async def fetch(session, sem, url):
    async with sem:
        try:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as r:
                r.raise_for_status()
                return url, await r.read()
        except Exception as e:
            return url, e

async def main(urls, limit=50):
    sem = asyncio.Semaphore(limit)
    async with aiohttp.ClientSession() as s:
        return await asyncio.gather(*(fetch(s, sem, u) for u in urls))
```
Weakness: this creates 10k task objects up front. For millions, use a worker-pool pattern with an `asyncio.Queue`. Always set timeouts, reuse the session (connection pooling), and add retries and per-host limits.

### Q35. CPU-bound parsing with multiprocessing.
```python
from concurrent.futures import ProcessPoolExecutor
import os

def parse_file(path): ...                     # top-level function (picklable)

if __name__ == "__main__":                    # REQUIRED for spawn start method
    paths = list_files()
    with ProcessPoolExecutor(max_workers=os.cpu_count()) as ex:
        for res in ex.map(parse_file, paths, chunksize=4):
            handle(res)
```
Pitfalls: lambdas/closures are not picklable; large args/results are serialized (pass file paths, not data); `fork` vs `spawn` vs `forkserver`; BrokenProcessPool if a worker is OOM-killed; use shared memory (`multiprocessing.shared_memory`) or Arrow for big arrays. Size `chunksize` to amortize IPC.

### Q36. Write a thread-safe counter and explain why `+=` is not atomic.
```python
import threading
class Counter:
    def __init__(self): self._n, self._l = 0, threading.Lock()
    def inc(self):
        with self._l: self._n += 1
```
`self._n += 1` compiles to load → add → store; a thread switch between them loses updates. Fix with a lock, `queue`, or per-thread counters merged at the end (lower contention). `dis.dis` demonstrates the multiple bytecodes.

## Section F: Data manipulation

### Q37. Flatten nested JSON into a record with dotted keys; handle lists.
```python
def flatten(obj, parent="", sep="."):
    out = {}
    if isinstance(obj, dict):
        for k, v in obj.items():
            out.update(flatten(v, f"{parent}{sep}{k}" if parent else k, sep))
    elif isinstance(obj, list):
        for i, v in enumerate(obj):
            out.update(flatten(v, f"{parent}[{i}]", sep))
    else:
        out[parent] = obj
    return out
```
Discuss: recursion limit on deep structures (iterate with an explicit stack), key collisions, arrays (explode into child tables vs indexed columns vs keep as JSON/array type), schema drift. Principal point: in a lakehouse you'd prefer schema-on-write with a struct/array type and explicit evolution policy over blind flattening.

### Q38. Stream JSONL, validate against a schema, and route bad records to a dead-letter file.
```python
import json
from pydantic import BaseModel, ValidationError

class Event(BaseModel):
    id: str
    ts: int
    amount: float

def process(src, good, dlq):
    for line_no, line in enumerate(src, 1):
        try:
            ev = Event.model_validate_json(line)
            good.write(ev.model_dump_json() + "\n")
        except (ValidationError, ValueError) as e:
            dlq.write(json.dumps({"line": line_no, "error": str(e), "raw": line}) + "\n")
```
Add counters (good/bad), threshold-based abort (e.g., fail the job if more than 1% bad), and replay tooling for the DLQ.

### Q39. Pandas: latest row per group, rolling metrics, and as-of join.
```python
df = df.sort_values(["id", "ts"])
latest = df.drop_duplicates("id", keep="last")             # or groupby().tail(1)
df["roll7"] = df.groupby("id")["val"].transform(lambda s: s.rolling(7, min_periods=1).mean())
# faster: df.groupby("id")["val"].rolling(7, min_periods=1).mean().reset_index(level=0, drop=True)
enriched = pd.merge_asof(trades.sort_values("ts"), quotes.sort_values("ts"),
                         on="ts", by="symbol", direction="backward", tolerance=pd.Timedelta("1s"))
```
`merge_asof` needs sorted keys. Know `groupby().agg` named aggregation, `pd.NamedAgg`, `merge(validate="one_to_one")`, and `indicator=True` for join diagnostics.

### Q40. Pandas performance: why is `df.apply(axis=1)` slow, and how do you fix it?
`apply(axis=1)` runs a Python function per row (builds a Series per row). Fix: vectorize (`np.where`, `Series.str`, `.dt`), use `np.select` for multi-branch logic, `merge`/`map` for lookups, categorical dtypes, `eval/query` for expressions, or switch to Polars/DuckDB.

```python
# slow
df["tier"] = df.apply(lambda r: "high" if r.amount > 1000 else "low", axis=1)
# fast
df["tier"] = np.where(df["amount"] > 1000, "high", "low")
```
Also: avoid chained assignment (`SettingWithCopyWarning`), avoid growing a DataFrame in a loop (collect then `concat` once), and use `pd.options.mode.copy_on_write = True` (default in pandas 3.0).

### Q41. PySpark: top-N per group, broadcast join, and avoiding a skew disaster.
```python
from pyspark.sql import functions as F, Window

w = Window.partitionBy("customer_id").orderBy(F.col("amount").desc())
top3 = df.withColumn("rn", F.row_number().over(w)).filter("rn <= 3")

# small dimension table: broadcast to avoid a shuffle
out = facts.join(F.broadcast(dim), "dim_id", "left")

# skew: salt the hot key
SALTS = 16
facts_s = facts.withColumn("salt", (F.rand() * SALTS).cast("int"))
dim_s = dim.crossJoin(spark.range(SALTS).withColumnRenamed("id", "salt"))
out = facts_s.join(dim_s, ["dim_id", "salt"])
```
On Spark 3+, enable AQE (`spark.sql.adaptive.enabled`, `skewJoin.enabled`) first; salting is the manual fallback. Diagnose with the Spark UI: one task far longer than the median, and spilled bytes.

### Q42. Write a pandas UDF and explain why it beats a plain UDF.
```python
import pandas as pd
from pyspark.sql.functions import pandas_udf

@pandas_udf("double")
def zscore(s: pd.Series) -> pd.Series:
    return (s - s.mean()) / s.std()
```
Data crosses JVM↔Python in Arrow batches (columnar, vectorized) rather than pickled row by row, with typically 3–100x speedup. Caveat: the mean here is *per batch*, not global, which is a correctness trap. For group-wise semantics use `groupBy().applyInPandas(...)`. Control batch size with `spark.sql.execution.arrow.maxRecordsPerBatch`.

## Section G: Reliability and design in Python

### Q43. Implement an idempotent incremental loader with a watermark and checkpoint.
```python
import json, os, tempfile

def load_checkpoint(p):
    return json.load(open(p)) if os.path.exists(p) else {"watermark": "1970-01-01"}

def save_checkpoint(p, state):                # atomic write: temp file + rename
    fd, tmp = tempfile.mkstemp(dir=os.path.dirname(p))
    with os.fdopen(fd, "w") as f: json.dump(state, f)
    os.replace(tmp, p)

def run(src, sink, ckpt):
    st = load_checkpoint(ckpt)
    batch = src.read_since(st["watermark"])   # at-least-once read
    sink.upsert(batch, key="id")              # idempotent: MERGE on a primary key
    save_checkpoint(ckpt, {"watermark": max(r["updated_at"] for r in batch)})
```
**Key reasoning:** exactly-once effect = at-least-once delivery + idempotent write. Order matters: write the sink first, checkpoint second, so a crash replays rather than loses. Handle late data with a lookback overlap (re-read the last N minutes) and watermark by a *source-assigned* monotonic value, not wall-clock time.

### Q44. Build a tiny DAG executor with topological sort and cycle detection (Airflow in miniature).
```python
from collections import deque, defaultdict

def run_dag(tasks: dict, deps: dict):          # deps: task -> set of upstream tasks
    indeg = {t: len(deps.get(t, ())) for t in tasks}
    down = defaultdict(list)
    for t, ups in deps.items():
        for u in ups: down[u].append(t)
    ready = deque(t for t, d in indeg.items() if d == 0)
    done = 0
    while ready:
        t = ready.popleft()
        tasks[t]()                              # execute (parallelize with a pool in real life)
        done += 1
        for d in down[t]:
            indeg[d] -= 1
            if indeg[d] == 0: ready.append(d)
    if done != len(tasks): raise ValueError("cycle detected")
```
Kahn's algorithm, O(V+E). Extensions: parallel execution of the ready set, retries and failure propagation (skip downstream), partial reruns from a failed task, and idempotent tasks keyed by logical date. `graphlib.TopologicalSorter` is in the stdlib.

### Q45. Design a pluggable source/sink connector interface.
```python
from typing import Protocol, Iterator, Mapping, Any

class Source(Protocol):
    def read(self, since: str | None) -> Iterator[Mapping[str, Any]]: ...

class Sink(Protocol):
    def write(self, batch: list[Mapping[str, Any]]) -> None: ...

REGISTRY: dict[str, type] = {}
def register(name):
    def deco(cls): REGISTRY[name] = cls; return cls
    return deco
```
Structural typing (`Protocol`) lets teams add connectors without inheriting from a base class. Config-driven instantiation (YAML + pydantic), versioned contracts, uniform error and metrics handling in the framework, and contract tests every connector must pass.

### Q46. Implement a simple Bloom filter. When would you use one?
```python
import hashlib

class Bloom:
    def __init__(self, m=1_000_003, k=5):
        self.m, self.k, self.bits = m, k, bytearray((m + 7) // 8)

    def _idx(self, item):
        h = hashlib.blake2b(str(item).encode(), digest_size=16).digest()
        a, b = int.from_bytes(h[:8], "big"), int.from_bytes(h[8:], "big")
        return ((a + i * b) % self.m for i in range(self.k))      # double hashing

    def add(self, item):
        for i in self._idx(item): self.bits[i >> 3] |= 1 << (i & 7)

    def __contains__(self, item):
        return all(self.bits[i >> 3] & (1 << (i & 7)) for i in self._idx(item))
```
No false negatives; tunable false-positive rate (≈ (1-e^(-kn/m))^k). Uses: dedup pre-check in streaming, skipping Parquet row groups/files (Parquet and Delta/Iceberg use bloom filters), join pruning. Cannot delete (use counting Bloom or cuckoo filters).

### Q47. A Python worker's memory grows until OOM. Walk through your diagnosis.
```python
import tracemalloc
tracemalloc.start(25)
s1 = tracemalloc.take_snapshot()
# ... run workload ...
s2 = tracemalloc.take_snapshot()
for stat in s2.compare_to(s1, "lineno")[:10]: print(stat)
```
**Process:** (1) confirm it is growth, not a high watermark (plot RSS over time). (2) `tracemalloc` diff for Python-level allocations; `memray` for native allocations. (3) Typical causes: unbounded dict/list/`lru_cache(maxsize=None)`, accumulating results from `executor.submit` futures, logging handlers holding records, exception tracebacks stored in lists, closures capturing big objects, C-extension or Arrow buffer leaks, glibc fragmentation (try `MALLOC_ARENA_MAX=2` or jemalloc). (4) Fixes: bound the structures, stream instead of accumulate, recycle workers, set container memory limits with alerting.

### Q48. Predict the output (gotchas the interviewer loves).
```python
# 1. Late binding
fs = [lambda: i for i in range(3)]
print([f() for f in fs])                  # [2, 2, 2]

# 2. Mutable default
def f(a, b=[]): b.append(a); return b
print(f(1), f(2))                         # [1, 2] [1, 2]  (same list)

# 3. Identity vs equality
a = 256; b = 256; c = 257; d = 257
print(a is b, c is d)                     # True, usually False (implementation detail)

# 4. Chained comparison and truthiness
print(1 < 2 < 3, 3 > 2 > 2)               # True False
print(bool([]), bool([0]), bool(""), bool(" "))   # False True False True

# 5. Tuple augmented assignment
t = ([1],)
try: t[0] += [2]
except TypeError: pass
print(t)                                  # ([1, 2],): mutated, then raised

# 6. Dict modification during iteration
d = {1: 1}
# for k in d: d[k+1] = 1                  # RuntimeError: dictionary changed size

# 7. Generator is exhausted after one pass
g = (x for x in range(3)); print(list(g), list(g))   # [0,1,2] []

# 8. Class vs instance attribute
class A: x = []
a1, a2 = A(), A(); a1.x.append(1); print(a2.x)       # [1]
```
Explain the *why* for each (cells and late binding, definition-time defaults, small-int cache, `__iadd__` then item assignment, dict resize invariants, single-pass iterators, class attribute sharing).

### Q49. Write a timing + logging context manager and decorator that works for sync and async functions.
```python
import asyncio, functools, logging, time
log = logging.getLogger(__name__)

def instrumented(fn):
    if asyncio.iscoroutinefunction(fn):
        @functools.wraps(fn)
        async def aw(*a, **kw):
            t = time.perf_counter()
            try: return await fn(*a, **kw)
            finally: log.info("%s took %.3fs", fn.__name__, time.perf_counter() - t)
        return aw
    @functools.wraps(fn)
    def sw(*a, **kw):
        t = time.perf_counter()
        try: return fn(*a, **kw)
        finally: log.info("%s took %.3fs", fn.__name__, time.perf_counter() - t)
    return sw
```
A sync wrapper around an `async def` would time only coroutine *creation*, not execution; that is the trap this question tests. In production emit structured metrics (Prometheus/OpenTelemetry spans), not just log lines. Use lazy `%s` logging args rather than f-strings.

### Q50. Principal design question: "Design a reusable Python ingestion framework used by 20 teams."
**How to structure a 45-minute answer:**

1. **Clarify:** sources (DB CDC, APIs, files, Kafka), volume (rows/s, TB/day), latency (batch vs streaming), who the users are (data engineers vs analysts), deployment (Airflow, k8s, Databricks jobs), compliance (PII).
2. **Core abstractions:** `Source` / `Transform` / `Sink` protocols, a `Pipeline` config (YAML validated by pydantic), a connector registry, a `Context` (run id, logical date, checkpoint store, metrics, secrets).
3. **Guarantees:** at-least-once + idempotent sinks (MERGE/upsert keyed on business key plus run id), atomic checkpoints, deterministic reruns by logical date, schema contract validation at the boundary, DLQ with replay.
4. **Reliability:** retry policy objects (transient vs permanent), circuit breakers, bounded concurrency/backpressure, timeouts everywhere, graceful shutdown, resource limits.
5. **Observability:** structured logs with run/trace IDs, standard metrics (rows in/out/rejected, lag, duration, DLQ rate), lineage emission (OpenLineage), data-quality gates.
6. **Developer experience:** a project template (`cookiecutter`), local runner with fixtures, contract-test kit for connectors, typed API, semantic versioning with deprecation policy, docs, and an `uv`-locked reproducible environment.
7. **Scale path:** the same pipeline definition runs in-process (small), on Spark/Ray (large) through an execution-engine adapter. Don't force everything through Python rows.
8. **Governance and adoption (the Principal part):** a platform is a product. Define paved-road vs escape-hatch, an RFC process for breaking changes, SLOs for the framework, adoption metrics, cost attribution per team, and a migration plan from existing ad-hoc scripts. Weigh *build vs adopt* (Airbyte, Meltano/Singer, dlt, Spark Structured Streaming, Flink) honestly: a Principal justifies custom code only where the existing tools fail a stated requirement.

**Common mistakes:** over-abstracting before three real use cases exist, ignoring versioning and backward compatibility, no story for failure and replay, treating Python as the data plane for TB-scale transforms, and no ownership/on-call model.

---

## Scorecard: self-evaluate after each question
| Dimension | S | St | P |
|---|---|---|---|
| Correct solution | ✔ | ✔ | ✔ |
| Complexity and edge cases | some | ✔ | ✔ |
| Failure handling and idempotency | – | ✔ | ✔ |
| Scale path and alternatives | – | some | ✔ |
| Operability (metrics, testing, rollout) | – | some | ✔ |
| Org impact and trade-off narrative | – | – | ✔ |
