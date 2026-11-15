# Python Concurrency: Threads vs Processes vs asyncio

## 1. Interview scenario

A Python service must fetch data from hundreds of APIs, decompress files, and perform CPU-heavy transformations. Choose a concurrency model and explain the trade-offs.

## 2. Decision framework

| Workload | Typical fit | Why |
|---|---|---|
| Blocking I/O (HTTP, database) | Threads or asyncio | Overlap waiting |
| CPU-heavy pure Python | Multiprocessing | Separate interpreters/processes |
| Many concurrent network sockets | asyncio | Cooperative, low per-task overhead |
| Native libraries releasing GIL | Threads may scale | Work executes outside Python bytecode lock |

The GIL limits parallel execution of Python bytecode in standard CPython builds. It does not mean threads are useless: they are valuable for I/O overlap. Verify runtime/version and library behavior rather than repeating a blanket rule.

## 3. Thread-pool example for blocking I/O

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch_all(urls, fetch, max_workers=16):
    results = {}
    failures = {}

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {executor.submit(fetch, url): url for url in urls}
        for future in as_completed(futures):
            url = futures[future]
            try:
                results[url] = future.result()
            except Exception as exc:
                failures[url] = repr(exc)

    return results, failures
```

## 4. Asyncio example

```python
import asyncio

async def fetch_all_async(urls, fetch_async, concurrency=20):
    semaphore = asyncio.Semaphore(concurrency)

    async def bounded_fetch(url):
        async with semaphore:
            return url, await fetch_async(url)

    tasks = [asyncio.create_task(bounded_fetch(url)) for url in urls]
    results = {}
    failures = {}

    for task in asyncio.as_completed(tasks):
        try:
            url, value = await task
            results[url] = value
        except Exception as exc:
            failures[str(exc)] = repr(exc)

    return results, failures
```

## 5. Multiprocessing example

```python
from concurrent.futures import ProcessPoolExecutor

def cpu_transform(value):
    # Keep this function top-level and picklable.
    return sum(i * i for i in range(value))

def transform_many(values, workers=None):
    with ProcessPoolExecutor(max_workers=workers) as executor:
        return list(executor.map(cpu_transform, values))
```

On platforms that use process spawning, protect process-pool startup with the usual `if __name__ == "__main__":` guard in executable scripts. Large objects sent to workers incur serialization and copy costs.

## 6. Complexity and trade-offs

Concurrency does not change the algorithmic complexity of the underlying operation. It changes wall-clock behavior and resource consumption. More concurrency increases sockets, memory, context switching, and pressure on dependencies. Bound in-flight tasks.

## 7. Interviewer–candidate discussion

**Interviewer:** You have 500 API calls and a CPU-heavy JSON transformation. What do you choose?

**Candidate:** Separate the stages. Use bounded async or thread-based I/O concurrency for API calls, then a process pool only if profiling shows CPU-bound Python work. Avoid moving huge payloads between processes; consider vectorized/native libraries or distributed processing.

**Interviewer:** Why not create 500 tasks immediately?

**Candidate:** Task count itself can consume memory. A semaphore bounds active work, but a producer with a bounded queue can also bound pending tasks.

## 8. Edge cases and tests

- Timeouts and cancellation.
- Partial failures and retry policy.
- Thread safety of shared clients and mutable state.
- Process pickling and startup overhead.
- Async function accidentally called without `await`.
- Connection pool limits lower than worker count.
- Graceful shutdown and task cancellation.

## 9. Follow-ups

1. Explain the GIL and its practical effect.
2. How do you pass cancellation through nested async functions?
3. How would you rate-limit concurrent API calls?
4. How do you profile whether a workload is CPU- or I/O-bound?
