# Deduplication and Idempotent Event Processing

## 1. Interview scenario

A source system delivers events at least once. Retries and replay can produce duplicates. Design a Python component that avoids applying the same business event twice.

The key distinction: deduplication is a data rule; idempotency is an end-to-end property of the side effect.

## 2. Approaches

**Naive:** keep every event ID in a Python set. Lookup is average O(1), but memory grows as O(N), and a process restart loses the set.

**Improved local illustration:** use a bounded TTL cache for a short replay window. This is useful for a worker-local optimization, but it is not a durable correctness boundary.

```python
from collections import OrderedDict
import time

class RecentEventIds:
    def __init__(self, ttl_seconds: float, max_items: int, clock=time.monotonic):
        if ttl_seconds <= 0 or max_items <= 0:
            raise ValueError("ttl_seconds and max_items must be positive")
        self.ttl = ttl_seconds
        self.max_items = max_items
        self.clock = clock
        self.items = OrderedDict()

    def seen_or_add(self, event_id: str) -> bool:
        now = self.clock()

        # Expire old entries from the oldest end.
        while self.items:
            oldest_id, inserted_at = next(iter(self.items.items()))
            if now - inserted_at <= self.ttl:
                break
            self.items.popitem(last=False)

        if event_id in self.items:
            self.items.move_to_end(event_id)
            return True

        self.items[event_id] = now
        while len(self.items) > self.max_items:
            self.items.popitem(last=False)
        return False
```

## 3. Durable idempotency pattern

For a payment, order, or warehouse sink, use a durable uniqueness boundary:

1. Derive a stable idempotency key, such as `(source_system, event_id)`.
2. In one database transaction, insert the key into an inbox/processed-events table with a unique constraint.
3. Apply the business update in the same transaction.
4. If the unique insert conflicts, treat it as already processed.
5. Commit both changes atomically.

If the side effect is in an external system, pass the same idempotency key to that system where supported. A local cache cannot guarantee exactly-once behavior across process crashes and network failures.

## 4. Complexity

- In-memory cache lookup/insert: average O(1).
- Memory: O(min(unique events in TTL window, max_items)).
- Durable table lookup/insert: depends on indexed storage; typically logarithmic index work, with storage and contention costs.

## 5. Dry run

Events `e1, e2, e1` arrive in order. The first `e1` is accepted, `e2` is accepted, and the second `e1` is identified as a duplicate. If the worker crashes after applying the business update but before recording `e1`, a non-atomic design can apply it twice. That is why the uniqueness record and business update must share a transaction.

## 6. Production considerations

- Choose the event identity carefully: a payload hash may incorrectly collapse two legitimate identical events.
- Include tenant/source namespace in keys to avoid collisions.
- Define retention based on maximum replay horizon and legal/audit requirements.
- Handle out-of-order events separately from duplicate events.
- Monitor duplicate rate, key conflicts, and replay lag.
- For streaming systems, align dedupe state retention with event-time windows and watermark policy.

## 7. Interviewer–candidate discussion

**Interviewer:** Is a Python set enough?

**Candidate:** Only for a single process and bounded lifetime. It is not durable, not shared across workers, and cannot atomically protect a database side effect.

**Interviewer:** Can you promise exactly once?

**Candidate:** I would avoid claiming end-to-end exactly once without carefully defining boundaries. At-least-once delivery plus idempotent processing and transactional state often gives effectively-once business outcomes within a defined system boundary.

## 8. Tests and edge cases

- Duplicate event in same process.
- Event after TTL expiration.
- Capacity eviction before TTL.
- Same event ID from two different source systems.
- Crash between side effect and dedupe write.
- Replayed event after dedupe retention expires.

```python
def test_recent_event_ids():
    now = [100.0]
    cache = RecentEventIds(10, 2, clock=lambda: now[0])
    assert cache.seen_or_add("a") is False
    assert cache.seen_or_add("a") is True
    now[0] = 111.0
    assert cache.seen_or_add("a") is False
```

## 9. Follow-up questions

1. How would you implement idempotency in Kafka consumer + relational database?
2. What happens when a downstream API times out after processing a request?
3. How would you deduplicate events with no natural ID?
4. How do retention and late-arriving data affect correctness?
