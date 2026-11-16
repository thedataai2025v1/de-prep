# Python for Principal Data Engineer Interviews — Part 1: Theory and Internals (Q1–25)

**How to use this:** For each question, first answer aloud in 2 minutes, then compare. Each entry has the core answer, a **Principal signal** (what separates a Principal answer from a Senior one), and a **likely follow-up**.

Level key: **S** = Senior expectation, **St** = Staff, **P** = Principal.

---

## Section A: Language model and runtime

### Q1. What happens when you run a Python file? Explain the CPython execution model.
**Answer.** Source is tokenized and parsed into an AST, compiled into bytecode (cached in `__pycache__/*.pyc`), and executed by the CPython eval loop, a stack-based virtual machine. Each function has a code object holding bytecode, constants and names. `import dis; dis.dis(fn)` shows it.

Python 3.11+ adds the *specializing adaptive interpreter*: hot bytecodes are rewritten to type-specialized versions (e.g. `BINARY_OP_ADD_INT`), giving roughly 10–60% speedups. 3.13 adds an experimental JIT and an experimental free-threaded build.

**Principal signal.** Connect it to cost: interpreter overhead per bytecode is why per-row Python in a hot loop is 50–100x slower than vectorized or JVM execution. That fact drives architecture decisions (push logic into Spark SQL, Arrow, Polars or native extensions).
**Follow-up.** "Why is `for` over a list of 10M ints slow, and what are your options?" (Vectorize with NumPy/Arrow, use a compiled extension, move to Spark, or use numba.)

**Interview example (use this to ground your answer):**

```python
>>> import dis
>>> dis.dis(lambda a, b: a + b)
  LOAD_FAST a
  LOAD_FAST b
  BINARY_OP +
  RETURN_VALUE         (3.11+ also prints a RESUME line first)
```
Say: "Each bytecode is dispatched by the eval loop, so a Python `for` loop over 10M numbers does tens of millions of dispatches. `np.sum(arr)` does the loop in C, which is typically orders of magnitude faster. That's why I push per-row logic into Spark SQL, Arrow or NumPy.

### Q2. Explain names, objects, mutability and identity. Why do bugs arise from this?
**Answer.** Variables are *names bound to objects*, not memory boxes. `a = b` copies a reference. Immutable types: `int, str, tuple, frozenset, bytes`. Mutable: `list, dict, set, bytearray`, user classes. `is` compares identity; `==` compares value via `__eq__`. Small ints (-5..256) and interned strings are cached, so `is` can appear to work by accident.

```python
def append(x, acc=[]):      # BUG: default evaluated once at def time
    acc.append(x); return acc
append(1); append(2)         # [1, 2] on the second call: shared state
# Fix: acc=None; if acc is None: acc = []
```
A tuple containing a list is immutable by identity but not by content: `t = ([],); t[0].append(1)` works.

**Principal signal.** Mutable default args and shared-state bugs in long-lived workers (Airflow, Celery, Spark executors reuse Python workers) cause cross-batch contamination that only shows up in production.

**Interview example (use this to ground your answer):**

```python
a = [1, 2]; b = a        # two names, ONE list
b.append(3); print(a)    # [1, 2, 3]
```
Production story: a module-level `cache = {}` used as a default in a task function persisted across runs inside a reused Airflow/Celery worker, so one tenant's data leaked into the next run's output. Fix: create state per call, or inject it explicitly.

### Q3. How does Python manage memory? Reference counting vs the cyclic GC.
**Answer.** Primary mechanism: **reference counting** (`sys.getrefcount`). When count hits 0 the object is freed immediately and deterministically. It cannot collect **reference cycles**, so a secondary **generational cyclic GC** (generations 0/1/2, tuned via `gc.set_threshold`) finds unreachable cycles. Small objects use `pymalloc` arenas, so freed memory is often *not returned to the OS*, which explains RSS staying high.

Tools: `gc.collect()`, `weakref` (caches without owning), `tracemalloc` (allocation diffs), `objgraph`, `memray`.

**Principal signal.** In long-running ingestion workers, RSS growth is usually (a) unbounded caches/dicts, (b) retained exceptions/tracebacks holding frames, (c) fragmentation, or (d) C-extension leaks. Diagnose with `tracemalloc` snapshots first, then `memray`. Mitigate with worker recycling (`max_requests` or `maxtasksperchild`).
**Follow-up.** "Does `del x` free memory?" (It removes the name; memory is freed only if the refcount reaches 0.)

**Interview example (use this to ground your answer):**

```python
class Node: pass
a, b = Node(), Node()
a.other, b.other = b, a     # reference cycle
del a, b                    # refcounts stay at 1, so only the cyclic GC can free them
import gc; gc.collect()
```
Say: "In a long-running worker I'd confirm growth with RSS over time, then diff `tracemalloc` snapshots. The usual culprits are unbounded caches and accumulating futures/results. We also set `maxtasksperchild` as a safety net.

### Q4. What is the GIL? What does it protect, and what does it not?
**Answer.** The Global Interpreter Lock lets only one thread execute Python bytecode at a time per interpreter. It protects CPython's non-thread-safe internals (refcounts). Threads switch roughly every 5 ms (`sys.getswitchinterval`) or when blocking.

- **I/O-bound** work: threads work well, because the GIL is released during blocking syscalls.
- **CPU-bound** pure Python: no parallelism with threads. Use processes or native code.
- C extensions (NumPy, pandas internals, pyarrow, hashlib) release the GIL, so threads *can* scale there.
- The GIL does **not** make your code thread-safe: `counter += 1` is multiple bytecodes, so races are still possible. You need locks or queues.

**3.13+:** PEP 703 free-threaded build (`python3.13t`) removes the GIL (experimental). Many C extensions need updates. PEP 684 per-interpreter GIL enables sub-interpreters.

**Principal signal.** Say clearly that you'd not bet a production platform on free-threading yet: check extension wheels, measure single-thread slowdown, and keep the multiprocessing/Spark design.

**Interview example (use this to ground your answer):**

```python
def work(): sum(i * i for i in range(10**7))
# 4 threads running work(): about the same wall time as running it 4x sequentially (GIL)
# 4 processes running work(): about 1/4 of the time on 4 cores
```
Race example: `counter += 1` from 8 threads can lose updates because it's load → add → store, and the GIL can switch between those bytecodes; fix with `threading.Lock` or per-thread counters merged at the end.

### Q5. Threads vs processes vs asyncio: how do you choose for data workloads?
| Workload | Best tool | Why |
|---|---|---|
| Many HTTP/DB/S3 calls | `asyncio` or `ThreadPoolExecutor` | I/O wait dominates |
| CPU-bound parsing/transform | `ProcessPoolExecutor` / Spark / Ray | Bypass GIL |
| Numeric arrays | NumPy/Arrow (releases GIL) | Native loops |
| Blocking libraries (boto3, psycopg2) | Threads (or `run_in_executor`) | Not async-aware |
| Massive parallel with fault tolerance | Spark, Ray, Dask | Scheduling, retries, shuffle |

Costs: processes need pickling for IPC, `spawn` start is slow on macOS/Windows, and `fork` with threads is unsafe. Threads share memory but need locks. asyncio is single-threaded cooperative, and one blocking call stalls everything.

**Principal signal.** Quantify: S3 `GetObject` latency ~50–100 ms, so 32 threads gives ~300–600 req/s per process. Then talk about backpressure and concurrency limits so you do not overwhelm downstream systems.

**Interview example (use this to ground your answer):**

Two real cases: (1) downloading 5,000 S3 objects: `ThreadPoolExecutor(max_workers=32)` because it's latency-bound (boto3 is blocking). (2) Parsing 200 GB of gzipped JSON: `ProcessPoolExecutor` (or better, Spark), because it's CPU-bound. Hybrid pipeline: threads for fetching, processes for parsing, with a bounded queue between them so a slow stage applies backpressure.

### Q6. Explain how asyncio works internally and its common pitfalls.
**Answer.** An event loop (selectors: epoll/kqueue) schedules *tasks* wrapping coroutines. `await` yields control to the loop until the awaited future completes. A coroutine function returns a coroutine object that does nothing until awaited or scheduled (`asyncio.create_task`, `gather`, `TaskGroup` in 3.11+).

Pitfalls:
1. Calling blocking code (`requests`, `time.sleep`, heavy pandas) inside a coroutine freezes the loop. Use `await asyncio.to_thread(...)`.
2. Forgetting `await` (coroutine never runs; "coroutine was never awaited" warning).
3. Unbounded `gather` over 100k items: use a `Semaphore` or a bounded queue.
4. Swallowed exceptions in fire-and-forget tasks: keep references and use `TaskGroup`.
5. Cancellation: handle `asyncio.CancelledError`, always `finally` to clean up.
6. Mixing event loops across libraries and threads.

**Interview example (use this to ground your answer):**

```python
async def bad():
    time.sleep(1)                         # blocks the whole event loop
async def good():
    await asyncio.sleep(1)                # yields to the loop
    r = await asyncio.to_thread(requests.get, url)   # blocking lib off the loop
```
Say: "I once saw an async service stall because one handler called a blocking DB driver, so every other request waited. We moved it to `to_thread` and capped concurrency with a semaphore.

### Q7. Explain iterators, iterables and generators. Why do they matter in data engineering?
**Answer.** Iterable: has `__iter__`. Iterator: has `__next__` and raises `StopIteration`. A generator function (contains `yield`) returns an iterator that suspends and resumes with local state preserved. Generators are **lazy**, so memory is O(1) rather than O(n). They compose into pipelines:

```python
def read(path):      yield from open(path)
def parse(lines):    return (json.loads(l) for l in lines)
def valid(rows):     return (r for r in rows if r.get("id"))
for r in valid(parse(read("big.jsonl"))): sink(r)
```
Gotchas: generators are single-use; `list(gen)` defeats laziness; `send()`/`throw()`/`close()` give coroutine-like behavior; exceptions inside generators surface at consumption time.

**Principal signal.** This streaming model is the same mental model as Spark partitions' iterators (`mapPartitions`) and Kafka consumer loops: process records without materializing partitions.

**Interview example (use this to ground your answer):**

```python
n = sum(1 for _ in open("50gb.log"))        # constant memory: the file object is an iterator
n = len(open("50gb.log").readlines())      # builds a 50 GB list: OOM
```
Real use: a validation → transform → sink pipeline as chained generators means a record flows through all stages before the next is read, the same mental model as Spark's `mapPartitions` iterators.

### Q8. Decorators and closures: how do they work, and what are the traps?
**Answer.** A closure captures variables from the enclosing scope (cells). A decorator is a callable that takes a function and returns a replacement. `@d` ≡ `f = d(f)`.

```python
import functools, time
def timed(fn):
    @functools.wraps(fn)           # preserves __name__, __doc__, signature info
    def wrapper(*a, **kw):
        t = time.perf_counter()
        try: return fn(*a, **kw)
        finally: print(fn.__name__, time.perf_counter() - t)
    return wrapper
```
Traps: forgetting `functools.wraps`; decorators with arguments need a 3-level nest; **late binding in closures** (`[lambda: i for i in range(3)]` all return 2, fix with `lambda i=i: i`); decorating methods and async functions needs care (`inspect.iscoroutinefunction`); decorators executed at import time can hide side effects.

**Interview example (use this to ground your answer):**

```python
@retry(max_attempts=5)          # decorator factory
@timed                          # applies first (closest to the function)
def load_partition(day): ...
```
Closure trap to mention: `fs = [lambda: i for i in range(3)]; [f() for f in fs]` returns `[2, 2, 2]` because `i` is looked up when called; `lambda i=i: i` binds at definition. Also, missing `functools.wraps` makes every decorated function show up as `wrapper` in logs and Airflow task names.

### Q9. Context managers: protocol and use cases.
**Answer.** `__enter__`/`__exit__(exc_type, exc, tb)` or `@contextlib.contextmanager`. Returning truthy from `__exit__` suppresses the exception. Use for files, locks, DB transactions, temp dirs, timing, Spark session scopes, and `ExitStack` for dynamic numbers of resources. `async with` / `__aenter__` for async resources.

```python
from contextlib import contextmanager
@contextmanager
def txn(conn):
    try:
        yield conn; conn.commit()
    except Exception:
        conn.rollback(); raise
```
**Principal signal.** Guarantees of cleanup under failure are a reliability property: mention leak prevention for connections and file handles in pipelines that run for months.

**Interview example (use this to ground your answer):**

```python
from contextlib import ExitStack
with ExitStack() as stack:
    files = [stack.enter_context(open(p)) for p in paths]   # N files, all closed even if one open fails
```
Production framing: "Every DB connection, temp dir and lock in our ingestion framework is acquired through a context manager, so a crash in step 7 can't leak resources that accumulate over months.

### Q10. Descriptors, properties and `__slots__`.
**Answer.** A descriptor implements `__get__/__set__/__delete__`; it powers `property`, methods (functions are non-data descriptors that bind `self`), `classmethod`, `staticmethod`. Data descriptors (define `__set__`) take precedence over instance `__dict__`. Attribute lookup: type data-descriptor → instance `__dict__` → type non-data descriptor/attr.

`__slots__` removes the per-instance `__dict__`, saving ~40–60% memory per object for millions of small objects and speeding attribute access, but you lose dynamic attributes and weakrefs (unless `__weakref__` is listed) and multiple inheritance gets harder. Python 3.10+ `@dataclass(slots=True)` automates it.

**Interview example (use this to ground your answer):**

```python
@dataclass(slots=True)
class Point:
    x: float
    y: float
```
For 5M such objects, dropping the per-instance `__dict__` roughly halves memory (measure with `tracemalloc` or `sys.getsizeof` plus `__dict__`). Property example: `@property def amount(self)` with validation in a setter is itself a descriptor. Methods are descriptors too, which is why `obj.method` binds `self`.

### Q11. Metaclasses, `__init_subclass__`, and dataclasses: when to use which?
**Answer.** Classes are objects created by `type`. A metaclass customizes class creation (ORMs, registries). Prefer simpler tools in this order: class decorator → `__init_subclass__` → metaclass. Dataclasses generate `__init__`, `__repr__`, `__eq__` (and `__hash__` rules: `eq=True` without `frozen` sets `__hash__ = None`). `frozen=True` gives immutability and hashability. `field(default_factory=list)` avoids the shared-mutable-default bug (dataclasses actually raise an error for bare mutable defaults).

**Principal signal.** Plugin registries for connectors (`__init_subclass__` auto-registering `SourceConnector` subclasses) are a real platform pattern; mention it, then mention keeping magic minimal for maintainability.

**Interview example (use this to ground your answer):**

```python
class Source:
    registry = {}
    def __init_subclass__(cls, name, **kw):
        super().__init_subclass__(**kw)
        Source.registry[name] = cls
class S3Source(Source, name="s3"): ...
Source.registry["s3"]        # S3Source
```
Say: "New connectors self-register just by being defined; no central if/elif. I use this instead of a metaclass because it's easier to read and debug.

### Q12. What is the MRO? How does `super()` work with multiple inheritance?
**Answer.** Method Resolution Order is computed by **C3 linearization**: it preserves local precedence order and monotonicity. Inspect with `Cls.__mro__`. `super()` follows the *MRO of the instance's class*, not "the parent", which enables cooperative multiple inheritance (each `__init__` calls `super().__init__(**kw)`). Diamond inheritance calls each base once.

Mixins should be small, stateless and cooperative. If C3 cannot be satisfied you get `TypeError: Cannot create a consistent MRO`.

**Interview example (use this to ground your answer):**

```python
class A:        pass
class B(A):     pass
class C(A):     pass
class D(B, C):  pass
print([k.__name__ for k in D.__mro__])   # ['D', 'B', 'C', 'A', 'object']
```
`super()` inside `B` can call `C`, not just `A`, when the instance is a `D`. That's why cooperative mixins pass `**kwargs` up the chain.

### Q13. Shallow vs deep copy and the pitfalls with nested data.
**Answer.** `copy.copy`, `list[:]`, `dict.copy()` copy the container but share element references. `copy.deepcopy` recursively clones (handles cycles via memo; slow, and can fail on locks, sockets, generators). `[[0]*3]*3` creates three references to the same row (classic bug); use `[[0]*3 for _ in range(3)]`.

**Principal signal.** In hot paths, deepcopy is a hidden CPU hog; prefer immutable structures or explicit constructors.

**Interview example (use this to ground your answer):**

```python
base = {"retries": 3, "tags": ["etl"]}
cfg = base.copy()               # shallow
cfg["tags"].append("prod")
print(base["tags"])             # ['etl', 'prod']  (shared list!)
cfg = copy.deepcopy(base)       # independent
```
Say: "Per-job configs cloned from a shared template were silently mutating each other. Fix with `deepcopy`, or better, frozen dataclasses so mutation is impossible.

## Section B: Data structures and the data model

### Q14. How do dict and set work internally? What makes an object hashable?
**Answer.** CPython dicts are open-addressing hash tables with a compact insertion-ordered entries array plus a sparse index table (since 3.6, ordering guaranteed from 3.7). Average O(1) lookup/insert; resize at ~2/3 load, amortized O(1). Keys must be hashable: `hash(obj)` stable over life and `a == b ⟹ hash(a) == hash(b)`. Mutable builtins are not hashable. Hash randomization (`PYTHONHASHSEED`) is applied to str/bytes: never use `hash()` for partitioning across processes; use `hashlib` / `mmh3`/`xxhash` for stable hashing.

**Principal signal.** The `hash()` partitioning bug (different workers route the same key differently) is a classic production defect. Likewise Spark's own hash partitioner differs from Python's.

**Interview example (use this to ground your answer):**

```python
# BUG: str hashes differ between processes/runs (PYTHONHASHSEED), so routing is inconsistent
partition = hash("user_42") % 8
# FIX: a stable hash
import zlib; partition = zlib.crc32(b"user_42") % 8
```
Story: two workers disagreed on which shard owned a key, producing duplicate rows. Also note that dict keys must be hashable: `{[1,2]: "x"}` raises `TypeError`, while a tuple key works.

### Q15. Name the key dunder methods and rules for `__eq__` / `__hash__` / ordering.
**Answer.** `__repr__` (unambiguous) vs `__str__` (readable); `__len__`, `__iter__`, `__getitem__`, `__contains__`, `__call__`, `__enter__/__exit__`, `__bool__`, `__lt__` (+ `functools.total_ordering`). Defining `__eq__` without `__hash__` makes the class unhashable. Return `NotImplemented` (not `False`) for unsupported operand types so Python tries the reflected operation.

**Interview example (use this to ground your answer):**

```python
class Money:
    def __init__(self, amt, cur): self.amt, self.cur = amt, cur
    def __eq__(self, o):
        if not isinstance(o, Money): return NotImplemented   # not False
        return (self.amt, self.cur) == (o.amt, o.cur)
    def __hash__(self): return hash((self.amt, self.cur))    # required if you define __eq__ and want it hashable
```
Without `__hash__`, instances can't be used in sets or as dict keys. Mention `functools.total_ordering` for comparisons.

### Q16. List comprehension vs generator expression vs `map`/`filter`: performance and semantics?
**Answer.** List comps are faster than equivalent `for`+`append` loops (specialized bytecode, no method lookup) and build the full list. Generator expressions are lazy and use constant memory but have slightly higher per-item overhead. Since 3.12 (PEP 709) comprehensions are inlined (no separate function frame). Use `sum(x for x in it)` rather than `sum([x for x in it])` for large inputs. Walrus `:=` avoids double computation: `[y for x in xs if (y := f(x)) is not None]`.

**Interview example (use this to ground your answer):**

```python
total = sum(x * x for x in range(10**7))      # generator: O(1) memory
total = sum([x * x for x in range(10**7)])    # list: ~tens of MB to hundreds of MB first
rows = [y for x in data if (y := parse(x)) is not None]   # walrus avoids calling parse twice
```
Say: "Comprehension for a result I need in memory; generator expression when feeding a reducer or writing a stream.

### Q17. Exceptions: hierarchy, chaining, custom errors, and pipeline error strategy.
**Answer.** Catch specific exceptions; never bare `except:` (it catches `KeyboardInterrupt`/`SystemExit`). `raise X from e` preserves the cause (`__cause__`); implicit chaining is `__context__`. `else` runs if no exception; `finally` always runs. 3.11 adds `ExceptionGroup` and `except*` for concurrent errors (TaskGroup).

Pipeline taxonomy: **transient** (retry with backoff+jitter), **permanent per-record** (dead-letter queue with the error and the offending payload), **systemic** (fail fast, alert, circuit-break). Define a small custom hierarchy (`PipelineError → RetryableError / DataQualityError`).

**Principal signal.** Never let one bad record kill a TB-scale job, and never silently drop it either: route to DLQ with metrics and an SLO on the DLQ rate.

**Interview example (use this to ground your answer):**

```python
try:
    row = parse(line)
except json.JSONDecodeError as e:
    dlq.write(line)                              # permanent, per-record: dead-letter it
    metrics.inc("bad_records")
except (ConnectionError, TimeoutError) as e:
    raise RetryableError("sink unavailable") from e   # transient: retry upstream; `from e` keeps the cause
```
Say: "One bad record must never kill a TB-scale job, but it also must never vanish silently, so DLQ plus metrics plus an SLO on the DLQ rate.

### Q18. Python typing: what does it give you, and what doesn't it do?
**Answer.** Type hints are not enforced at runtime. Static checkers (mypy, pyright) catch interface bugs. Know `Optional`/`X | None`, `Protocol` (structural typing, good for pluggable sources/sinks), `TypedDict`, `Generic`/`TypeVar`, `Literal`, `Final`, `ParamSpec` for decorators, `Self`. For runtime validation at boundaries use **pydantic v2** or `msgspec`; type hints on internal functions plus validation at ingestion boundaries is the right split.

**Interview example (use this to ground your answer):**

```python
class Sink(Protocol):
    def write(self, batch: list[dict]) -> None: ...

class Order(BaseModel):          # pydantic v2: validate at the boundary
    id: str
    amount: float
```
Say: "Type hints are for mypy in CI; pydantic validates data entering the system. Teams can add a new `Sink` without inheriting from my base class, and mypy checks conformance structurally.

### Q19. Packaging and environments: how do you make Python builds reproducible?
**Answer.** `pyproject.toml` (PEP 517/621) defines metadata. Use a lockfile (`uv lock`, `pip-tools`, `poetry.lock`) with hashes. Build wheels, not sdists, for deployment. Pin the Python version; use Docker multi-stage builds with a wheelhouse. Separate runtime vs dev dependencies. Avoid dependency conflicts via constraints files. For Spark/Databricks: ship zipped venvs/wheels (`--py-files`, `venv-pack`, cluster libraries) and align the driver and executor Python minor versions or PySpark fails.

**Principal signal.** Dependency supply chain: private index, hash-checking, Dependabot/Renovate, SBOM, vulnerability scanning.

**Interview example (use this to ground your answer):**

```bash
uv lock                                          # or pip-compile --generate-hashes
pip install --require-hashes -r requirements.txt # reproducible, tamper-evident
venv-pack -o env.tar.gz                          # ship a venv to Spark: --archives env.tar.gz#environment
```
Story: a transitive dependency auto-upgraded and broke a nightly job; lockfiles with hashes plus a private index and Renovate PRs made upgrades deliberate and reviewable.

### Q20. How does the import system work? Circular imports?
**Answer.** `import x` checks `sys.modules`, then runs finders (`sys.meta_path`) and loaders, executes module top-level once, and caches the module. Circular imports fail when a module reads a name from a partially initialized module. Fixes: restructure (extract shared code), import inside the function, or `from __future__ import annotations` plus `TYPE_CHECKING` for type-only imports. Heavy top-level imports slow CLI and serverless cold-starts (use lazy imports; `python -X importtime`).

**Interview example (use this to ground your answer):**

```python
# a.py: from b import B    b.py: from a import A   -> ImportError (partially initialized module)
# fix 1: move shared code to c.py
# fix 2: import inside the function
# fix 3: type-only import
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from a import A
```
Cold-start example: a CLI took 3 s to start because it imported pandas and boto3 at the top; `python -X importtime` showed it, and lazy imports fixed it.

## Section C: Performance, data and ecosystem

### Q21. How do you profile and speed up slow Python?
**Process:** measure first. `cProfile`/`snakeviz` (function-level), `py-spy` (sampling a live process with no code change, flame graphs), `line_profiler`, `memray`/`tracemalloc` (memory), `timeit`.

**Ladder of fixes (cheapest first):** better algorithm/data structure → remove repeated work (`functools.lru_cache`, hoisting) → built-ins and C-backed stdlib (`itertools`, `collections`, `bisect`, `heapq`) → vectorize (NumPy/Arrow/Polars) → batch I/O → parallelize (processes/Spark) → compile (numba, Cython, Rust via PyO3, `mypyc`).

**Interview example (use this to ground your answer):**

```bash
python -m cProfile -o out.prof job.py && snakeviz out.prof    # where is time spent?
py-spy dump --pid 12345                                       # what is a live process doing right now?
```
Story: "Profiling showed ~70% of runtime in `json.loads`. Switching to `orjson` and batching writes cut runtime enough that we didn't need to scale out." (Use your real numbers.) Walk the ladder: algorithm → stdlib → vectorize → parallelize → compile.

### Q22. Serialization: pickle, JSON, Avro, Parquet, Arrow: when is each right?
- **pickle:** Python-only, can execute arbitrary code on load (never unpickle untrusted data), version-fragile. OK for ephemeral IPC.
- **JSON/JSONL:** universal, text, slow, no schema or decimals/datetime types. `orjson` is 3–10x faster than stdlib.
- **Avro/Protobuf:** row-oriented with schema evolution, good for Kafka.
- **Parquet:** columnar at-rest storage with predicate pushdown.
- **Arrow:** in-memory columnar, zero-copy between pandas/Polars/Spark/DuckDB; Arrow IPC/Flight for transport.

**Interview example (use this to ground your answer):**

Decision table in one sentence each: Kafka events → Avro/Protobuf with a schema registry; lake storage → Parquet; passing DataFrames between Python/Spark/DuckDB → Arrow; JSON only at the edges. Security example: `pickle.loads(untrusted_bytes)` can execute arbitrary code via `__reduce__`, so never unpickle data you didn't create. `orjson.dumps(obj)` is a drop-in speedup over stdlib `json`.

### Q23. How do you process data larger than memory in Python?
**Answer.** (1) Stream with generators or chunked reads (`pd.read_csv(chunksize=)`, `pyarrow.dataset` scanners, row-group iteration). (2) Choose efficient dtypes (`category`, `int32`, Arrow-backed strings). (3) Columnar projection and predicate pushdown. (4) Polars lazy/streaming or DuckDB for out-of-core. (5) `mmap` for random access. (6) External sort/merge via `heapq.merge`. (7) Distribute (Spark/Dask/Ray). A pandas DataFrame commonly needs 2–5x the file size in RAM, so know the rule of thumb.

**Interview example (use this to ground your answer):**

```python
for chunk in pd.read_csv("big.csv", chunksize=1_000_000, dtype={"id": "int32", "cat": "category"}):
    partial.append(chunk.groupby("cat")["amt"].sum())
import duckdb
duckdb.sql("SELECT cat, sum(amt) FROM 'data/*.parquet' GROUP BY cat")   # out-of-core, columnar
```
Say: "A 5 GB CSV can need 10-25 GB RAM in pandas (string objects). I first try dtypes and column projection, then chunking, then DuckDB/Polars, then Spark.

### Q24. PySpark: Python UDF cost and how to avoid it.
**Answer.** A plain Python UDF serializes rows JVM→Python worker (pickle) and back, row by row, and blocks Catalyst optimization (a black box: no predicate pushdown through it). Prefer built-in `pyspark.sql.functions` (run in JVM, Tungsten codegen). If custom logic is unavoidable use **pandas UDF / Arrow-optimized UDF** (vectorized over Arrow batches), `mapInPandas`/`applyInPandas`, or write a Scala/Java UDF. Watch out for Python worker memory (`spark.executor.pyspark.memory`), broadcasting large Python objects, and driver/executor Python version mismatch.

**Interview example (use this to ground your answer):**

```python
# slow: one Python call per row, rows pickled both ways
upper = F.udf(lambda s: s.upper(), "string")
# fast: built-in in the JVM
F.upper("name")
# when you truly need Python: vectorized over Arrow batches
@pandas_udf("string")
def up(s: pd.Series) -> pd.Series: return s.str.upper()
```
Say: "I rank them built-in > pandas UDF > Python UDF, and I confirm with `explain()` that no `BatchEvalPython` sits in a hot path.

### Q25. How do you test Python data pipelines at a Principal level?
**Answer.** Pyramid: fast **unit tests** on pure transformation functions (separate I/O from logic), **contract tests** for schemas, **integration tests** with ephemeral infra (testcontainers for Kafka/Postgres, local Spark), **data-quality tests** (Great Expectations, dbt tests, Soda) in production. `pytest`: fixtures, parametrize, `tmp_path`, `monkeypatch`; mock at boundaries (`unittest.mock.patch` where the name is *used*, not defined). **Property-based testing** (`hypothesis`) is excellent for idempotency, ordering and round-trip properties. Deterministic time: inject clocks. Track coverage but gate on mutation/behavior tests for critical code.

**Interview example (use this to ground your answer):**

```python
@pytest.mark.parametrize("rows,expected", [([], []), ([(1, "a", 1), (1, "b", 2)], [(1, "b", 2)])])
def test_latest_by_key(rows, expected): assert latest_by_key(rows) == expected

from hypothesis import given, strategies as st
@given(st.lists(st.integers()))
def test_dedup_is_idempotent(xs): assert dedup(dedup(xs)) == dedup(xs)
```
Say: "Transform logic is pure functions so it's unit-testable without Spark or a DB; contract and integration tests use testcontainers; production has data-quality gates (dbt tests / Great Expectations).

---
**Next:** Part 2 contains the 25 coding practicals (Q26–50).
