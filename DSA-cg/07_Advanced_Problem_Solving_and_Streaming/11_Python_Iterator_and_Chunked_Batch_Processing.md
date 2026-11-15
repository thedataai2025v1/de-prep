# Python Iterators and Chunked Batch Processing

**Level:** Medium | **Patterns:** Lazy iteration, bounded memory, streaming transforms

## 1. Concept and when to use it

A Python iterator produces values one at a time. Chunking lets a pipeline process large inputs in bounded batches rather than loading all records into memory.

Use this for large files, database cursors, API pagination, and ETL stages where memory must remain predictable.

## 2. Approaches

### Brute force

Read all records into a list, then split into batches. This is simple but memory is `O(n)`.

### Optimized: generator-based chunking

```python
from itertools import islice

def chunks(iterable, size):
    if size <= 0:
        raise ValueError("size must be positive")
    iterator = iter(iterable)

    while True:
        batch = list(islice(iterator, size))
        if not batch:
            break
        yield batch
```

Example aggregation without materializing the whole input:

```python
def count_by_status(records, batch_size):
    totals = {}
    for batch in chunks(records, batch_size):
        for record in batch:
            status = record["status"]
            totals[status] = totals.get(status, 0) + 1
    return totals
```

## 3. Detailed dry run

Input is a generator producing records 1 through 7; chunk size is 3.
- First `next()` from `chunks` consumes 1,2,3 and yields `[1,2,3]`.
- Second consumes 4,5,6 and yields `[4,5,6]`.
- Third consumes 7 and yields `[7]`.
- Next call sees exhaustion and stops.
Only one chunk is held by the consumer at a time, assuming it does not retain old batches.

## 4. Complexity

Chunking performs `O(n)` total work and holds `O(size)` items per batch. The downstream aggregation uses additional memory for its result state (for example, number of distinct statuses).

## 5. Edge cases and common mistakes

- `size<=0` must fail to avoid an infinite loop or invalid slicing.
- The input may be a one-shot iterator; do not attempt to iterate it twice.
- A generator is lazy, but a downstream `list(...)` can still materialize everything.
- Handle exceptions and partial batch failures explicitly in production.
- Idempotency and checkpointing are separate concerns from chunking.
- If processing batches asynchronously, bound the in-flight queue to preserve backpressure.

## 6. Interviewer–candidate discussion

**Interviewer:** Does using a generator guarantee low memory?

**Candidate:** No. It makes production lazy. If the consumer accumulates every yielded batch, memory still grows with input size. The entire pipeline must preserve bounded retention.

**Interviewer:** What happens if processing batch 4 fails after batches 1–3 were committed?

**Candidate:** We need a checkpoint or durable offset, idempotent writes, and a retry policy. Iterator mechanics alone do not provide exactly-once processing.

## 7. Follow-ups

- Add retry with bounded exponential backoff.
- Process chunks concurrently with a bounded worker pool.
- Support async iterators.
- Explain at-least-once versus exactly-once semantics.
- Add metrics for throughput, lag, and batch failures.

## 8. Practical exercises and tests

```python
assert list(chunks(range(7), 3)) == [[0, 1, 2], [3, 4, 5], [6]]
assert list(chunks([], 4)) == []
assert list(chunks([1, 2], 2)) == [[1, 2]]
try:
    list(chunks([1], 0))
    raise AssertionError("Expected invalid chunk size")
except ValueError:
    pass

records = [{"status": "ok"}, {"status": "error"}, {"status": "ok"}]
assert count_by_status(records, 2) == {"ok": 2, "error": 1}
```

**Data-platform connection:** Chunking is a basic building block for memory-safe ETL. Principal-level discussion should include backpressure, retries, idempotency, checkpointing, and observability—not just generator syntax.
