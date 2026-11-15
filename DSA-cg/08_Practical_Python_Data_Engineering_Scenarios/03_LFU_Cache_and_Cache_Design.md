# LFU Cache and Cache Design Trade-offs

## 1. Interview scenario

Implement a cache with O(1) average `get` and `put`, evicting the least frequently used key. When frequencies tie, evict the least recently used key.

This extends the LRU cache problem: the challenge is maintaining multiple linked ordering relationships.

## 2. Data structures

- `values[key] -> value`
- `freq[key] -> frequency`
- `buckets[f] -> OrderedDict of keys`, ordered from least to most recently used within frequency
- `min_freq -> smallest frequency currently present`

```python
from collections import defaultdict, OrderedDict

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.values = {}
        self.freq = {}
        self.buckets = defaultdict(OrderedDict)
        self.min_freq = 0

    def _touch(self, key):
        old_freq = self.freq[key]
        del self.buckets[old_freq][key]
        if not self.buckets[old_freq] and self.min_freq == old_freq:
            self.min_freq += 1

        new_freq = old_freq + 1
        self.freq[key] = new_freq
        self.buckets[new_freq][key] = None

    def get(self, key):
        if key not in self.values:
            return -1
        self._touch(key)
        return self.values[key]

    def put(self, key, value):
        if self.capacity <= 0:
            return
        if key in self.values:
            self.values[key] = value
            self._touch(key)
            return

        if len(self.values) == self.capacity:
            key_to_remove, _ = self.buckets[self.min_freq].popitem(last=False)
            del self.values[key_to_remove]
            del self.freq[key_to_remove]

        self.values[key] = value
        self.freq[key] = 1
        self.buckets[1][key] = None
        self.min_freq = 1
```

## 3. Dry run

Capacity is 2. Insert A and B: both have frequency 1; B is newer in that bucket. Access A: A moves to frequency 2. Insert C: the minimum frequency is 1, so B is evicted. A survives because it is more frequently used.

## 4. Complexity

Average O(1) for get and put using hash maps and ordered dictionaries. Space is O(C), where C is cache capacity. Python's `OrderedDict` operations are average O(1).

## 5. LRU vs LFU

| Dimension | LRU | LFU |
|---|---|---|
| Eviction | Least recently accessed | Lowest access frequency |
| Adapts quickly to phase changes | Usually better | Can retain stale historically hot keys |
| Metadata | Recency order | Frequency buckets + recency |
| Complexity | O(1) common design | O(1) with more bookkeeping |

## 6. Production concerns

- Frequency aging/decay prevents old popularity from dominating forever.
- A cache needs a memory bound and often TTL.
- Define whether a write counts as an access.
- Consider cache stampede protection, negative caching, and stale-while-revalidate.
- Distributed caches need consistency and concurrency semantics beyond this single-threaded implementation.

## 7. Tests

```python
def test_lfu_eviction():
    cache = LFUCache(2)
    cache.put(1, "A")
    cache.put(2, "B")
    assert cache.get(1) == "A"
    cache.put(3, "C")       # key 2 is least frequent
    assert cache.get(2) == -1
    assert cache.get(3) == "C"
    assert cache.get(1) == "A"

def test_zero_capacity():
    cache = LFUCache(0)
    cache.put(1, "A")
    assert cache.get(1) == -1
```

## 8. Interviewer–candidate discussion

**Interviewer:** Why is `min_freq` sufficient?

**Candidate:** Every new key starts at frequency 1, so insertion resets it to 1. On access, the only bucket that can become empty and affect the minimum is the key's old frequency. If that bucket was the minimum, the key moves exactly one frequency higher, so the minimum increases by one.

**Interviewer:** What if frequency values grow indefinitely?

**Candidate:** In a long-running production cache, use aging or periodic frequency normalization. The interview implementation focuses on the core eviction invariant.

## 9. Follow-ups

1. Add TTL expiration.
2. Make the cache thread-safe.
3. Implement LFU with frequency decay.
4. Discuss cache invalidation for slowly changing dimension data.
