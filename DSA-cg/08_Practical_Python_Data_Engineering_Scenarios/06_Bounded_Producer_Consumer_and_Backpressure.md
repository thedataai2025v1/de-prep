# Bounded Producer–Consumer Pipeline and Backpressure

## 1. Interview scenario

One thread reads records from a source while worker threads transform them. The producer can be faster than consumers. Prevent unbounded memory growth and shut down cleanly.

## 2. Core idea

Use a bounded `queue.Queue(maxsize=N)`. When the queue fills, `put()` blocks, applying backpressure to the producer. Use sentinel objects to tell workers to exit after all work is submitted.

```python
from queue import Queue
from threading import Thread

STOP = object()

def run_pipeline(source, transform, worker_count=4, queue_size=100):
    if worker_count <= 0 or queue_size <= 0:
        raise ValueError("worker_count and queue_size must be positive")

    work_queue = Queue(maxsize=queue_size)
    results = []
    errors = []

    def worker():
        while True:
            item = work_queue.get()
            try:
                if item is STOP:
                    return
                results.append(transform(item))
            except Exception as exc:
                errors.append((item, exc))
            finally:
                work_queue.task_done()

    workers = [Thread(target=worker) for _ in range(worker_count)]
    for thread in workers:
        thread.start()

    for item in source:
        work_queue.put(item)  # blocks when full

    for _ in workers:
        work_queue.put(STOP)

    work_queue.join()
    for thread in workers:
        thread.join()

    return results, errors
```

## 3. Dry run

With queue size 2, if workers are slower than the source, the producer can place only two pending records before `put()` blocks. Once a worker consumes an item, the producer can continue. Pending work is bounded.

## 4. Complexity

- Total transformation work: O(N) if each item is processed once.
- Queue memory: O(Q × average item size), where Q is queue capacity.
- Worker memory: depends on number of workers and per-record working set.
- Results list is O(N); for a truly streaming pipeline, replace it with a bounded output sink or write results incrementally.

## 5. Important caveats

- The example assumes `transform` is safe to call concurrently.
- Appending to a list is safe enough for this simple CPython illustration, but output ordering is not guaranteed.
- An exception in the producer/source iteration needs coordinated shutdown; production code should use `try/finally` and cancellation signaling.
- For database writes, batch and commit semantics must be explicit.
- Use bounded queues between every stage, not just at the input.
- Define poison-pill behavior when workers fail.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is a bounded queue important?

**Candidate:** It converts overload into controlled waiting instead of allowing memory to grow until the process is killed. It also exposes the throughput mismatch.

**Interviewer:** Does adding workers always improve throughput?

**Candidate:** No. CPU-bound work may be constrained by the GIL in threads; I/O-bound work can benefit. More workers can increase contention, downstream pressure, and memory use.

## 7. Tests and edge cases

- Empty source.
- More workers than records.
- Transform raises for one record.
- Source iteration raises midway.
- Slow consumer and fast producer.
- Shutdown while queue is full.
- Preserve ordering if the contract requires it (attach sequence numbers and reorder at a bounded stage).

```python
def test_pipeline_basic():
    results, errors = run_pipeline(
        [1, 2, 3],
        lambda value: value * 2,
        worker_count=2,
        queue_size=1,
    )
    assert sorted(results) == [2, 4, 6]
    assert errors == []
```

## 8. Follow-ups

1. Implement ordered output while allowing parallel processing.
2. Add graceful cancellation and worker health monitoring.
3. How would you tune queue capacity from throughput and latency?
4. How do you propagate backpressure across Kafka, Spark, and a database sink?
