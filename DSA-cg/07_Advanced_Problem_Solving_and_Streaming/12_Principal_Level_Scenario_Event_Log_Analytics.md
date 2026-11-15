# Principal-Level Scenario: Event Log Analytics Under Memory Constraints

**Level:** Hard | **Patterns:** Parsing, aggregation, top-k, heap merge, bounded memory

## 1. Scenario and requirements

You receive sorted event streams from multiple partitions. Each record contains `event_time`, `service`, and `status`. You need to:
1. Merge the streams by event time.
2. Count events by `(service, status)`.
3. Track the top-k most frequent error services.
4. Process in bounded memory.

Clarify whether the streams are globally sorted, how ties are ordered, whether late events can arrive, and whether counts must be exact.

## 2. Approaches

### Brute force

Load all records into memory, sort by event time, count all keys, and sort service counts. This is straightforward but requires `O(N)` record memory.

### Optimized batch/stream design

- Use a heap to merge sorted partition streams: `O(N log P)` where `P` is partition count.
- Update exact counters by key: `O(1)` expected per record, memory `O(D)` where `D` is distinct `(service,status)` keys.
- For final top-k, select from aggregated service error counts using a size-k heap: `O(Ds log k)` for `Ds` distinct services.
- Use chunked consumption or generators to avoid retaining all events.
- If distinct cardinality is too large, consider approximate sketches and explicitly communicate their error guarantees.

```python
import heapq
from collections import Counter

def merge_event_streams(streams):
    iterators = [iter(stream) for stream in streams]
    heap = []

    for stream_id, iterator in enumerate(iterators):
        try:
            event = next(iterator)
            heapq.heappush(heap, (event["event_time"], stream_id, event))
        except StopIteration:
            pass

    while heap:
        _, stream_id, event = heapq.heappop(heap)
        yield event
        try:
            next_event = next(iterators[stream_id])
            heapq.heappush(
                heap, (next_event["event_time"], stream_id, next_event)
            )
        except StopIteration:
            pass


def analyze_events(streams, k):
    if k < 0:
        raise ValueError("k must be non-negative")

    counts = Counter()
    error_counts = Counter()

    for event in merge_event_streams(streams):
        key = (event["service"], event["status"])
        counts[key] += 1
        if event["status"] == "ERROR":
            error_counts[event["service"]] += 1

    top_errors = sorted(
        error_counts.items(), key=lambda pair: (-pair[1], pair[0])
    )[:k]
    return counts, top_errors
```

## 3. Detailed dry run

Partition A: events at times 1 and 4. Partition B: events at times 2 and 3.
- Heap starts with the first event from each partition: time 1 and time 2.
- Emit time 1, then advance partition A and push time 4.
- Emit time 2, advance partition B and push time 3.
- Emit time 3, then time 4.
- Each event updates one exact `(service,status)` count.
- Error events also increment the service-level error counter.
- At end, sort or heap-select the top-k services.

## 4. Complexity and memory model

Let `N` be total events, `P` partitions, `D` distinct `(service,status)` keys, and `S` distinct services.
- Merge: `O(N log P)` time, `O(P)` heap space.
- Aggregation: expected `O(N)` time, `O(D+S)` counter memory.
- Top-k: `O(S log k)` with a bounded heap, or `O(S log S)` by sorting.
- Total exact memory is dominated by cardinality state, not event buffering.

## 5. Edge cases and common mistakes

- Partition inputs must each be sorted under the same ordering.
- Tie-breaking by partition ID makes merge output deterministic for equal timestamps.
- A timestamp alone may not define a total order; add event ID or sequence number if required.
- Late events break a simple final-order merge unless there is a watermark/buffering policy.
- Exact counters still grow with cardinality; “streaming” does not mean constant memory.
- Define how null service names and unknown statuses are handled.
- Do not call a result exactly-once without a durable checkpoint/commit design.

## 6. Interviewer–candidate discussion

**Interviewer:** Where is the memory bottleneck?

**Candidate:** The merge heap is only `O(P)`, but exact aggregation requires `O(D+S)` state. If key cardinality is very high, that state can dominate. We need to confirm whether exact counts are required and whether aggregation can be partitioned or persisted.

**Interviewer:** What if one partition is delayed?

**Candidate:** A synchronous k-way merge waits for that partition's next head to know the global minimum. For distributed event-time systems, we need watermarks, bounded buffering, timeout behavior, and an explicit late-event policy.

**Interviewer:** How would you make retries safe?

**Candidate:** Use stable event IDs, idempotent writes or deduplication, and checkpoints coordinated with output commits. The precise approach depends on the processing engine and delivery guarantees.

## 7. Follow-ups and variations

1. Add a 5-minute tumbling window and compute top error services per window.
2. Support late-arriving events with a watermark and allowed lateness.
3. Replace exact counters with Count-Min Sketch plus a heavy-hitter candidate strategy.
4. Partition aggregation by service hash and combine partial counts.
5. Design a replay-safe checkpoint and output commit protocol.
6. Explain how skewed services affect partition load and how to mitigate hot keys.

## 8. Practical exercises and tests

```python
a = [
    {"event_time": 1, "service": "api", "status": "ERROR"},
    {"event_time": 4, "service": "db", "status": "OK"},
]
b = [
    {"event_time": 2, "service": "api", "status": "ERROR"},
    {"event_time": 3, "service": "cache", "status": "ERROR"},
]
counts, top = analyze_events([a, b], 2)
assert counts[("api", "ERROR")] == 2
assert counts[("db", "OK")] == 1
assert top == [("api", 2), ("cache", 1)]
assert analyze_events([[], []], 3) == (Counter(), [])
try:
    analyze_events([], -1)
    raise AssertionError("Expected negative k to fail")
except ValueError:
    pass
```

**Principal-level takeaway:** A strong answer separates algorithmic complexity from system constraints: event ordering, cardinality, skew, late data, durability, replay, and acceptable approximation.
