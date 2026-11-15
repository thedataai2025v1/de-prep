# Top-K Frequent Elements in a Stream

**Level:** Medium–Hard | **Patterns:** Hash map, bounded heap, streaming aggregation

## 1. Concept and when to use it

Maintain counts while records arrive, then identify the `k` most frequent keys. This is common in log analysis, clickstream analytics, error-code monitoring, and identifying hot keys.

The exact solution needs memory proportional to the number of distinct keys unless the key space is bounded or an approximation is acceptable.

## 2. Approaches

### Brute force

Count all keys, sort every `(key,count)` pair by descending count. Time is `O(n + d log d)` where `d` is distinct-key count.

### Optimized for a completed batch: size-k heap

```python
from collections import Counter
import heapq

def top_k_frequent(items, k):
    if k < 0:
        raise ValueError("k must be non-negative")
    counts = Counter(items)
    if k == 0:
        return []
    # Deterministic tie-break: lexicographically smaller key first.
    return [
        key for key, _ in sorted(
            counts.items(), key=lambda pair: (-pair[1], pair[0])
        )[:k]
    ]

def top_k_frequent_heap(items, k):
    if k < 0:
        raise ValueError("k must be non-negative")
    counts = Counter(items)
    return heapq.nsmallest(
        k, counts.items(), key=lambda pair: (-pair[1], pair[0])
    )
```

For a live stream, keep a `Counter` updated per event; computing the exact top-k repeatedly still requires examining the distinct-key counts or maintaining a more complex indexed structure.

## 3. Detailed dry run

Items: `["api", "db", "api", "cache", "db", "api"]`.
- Counts: `api:3`, `db:2`, `cache:1`.
- For `k=2`, sorted result is `["api","db"]`.
- If two keys tie, the explicit lexical tie-break makes the output deterministic.

## 4. Complexity

Counting is `O(n)` expected time and `O(d)` space. The shown deterministic sorting approach adds `O(d log d)`. A bounded heap can select top-k in `O(d log k)` after counting, using `O(k)` heap space in addition to the count map.

## 5. Edge cases and common mistakes

- `k=0` returns an empty list.
- `k` greater than distinct count returns all distinct keys.
- Negative `k` is invalid.
- Define tie-breaking; otherwise tests may be nondeterministic.
- A count map is not truly bounded-memory for unbounded cardinality.
- Approximate algorithms may be appropriate for huge streams, but must expose error/guarantee trade-offs.

## 6. Interviewer–candidate discussion

**Interviewer:** Can exact top-k use only O(k) memory?

**Candidate:** Not in the general case for an unbounded key universe when exact counts are required. We need enough state to distinguish frequencies of potentially every distinct key, unless the input constraints provide additional structure.

**Interviewer:** What would you do for a high-cardinality stream?

**Candidate:** Consider Count-Min Sketch for approximate frequencies, Space-Saving for heavy hitters, partitioned aggregation, or a bounded key domain. The right choice depends on acceptable error and whether false positives are tolerable.

## 7. Follow-ups

- Top-k per tenant or time window.
- Maintain top-k continuously as counts update.
- Find heavy hitters using Misra–Gries / Space-Saving.
- Compare exact count map, Count-Min Sketch, and distributed aggregation.

## 8. Practical exercises and tests

```python
assert top_k_frequent(["api", "db", "api", "cache", "db", "api"], 2) == ["api", "db"]
assert top_k_frequent([], 3) == []
assert top_k_frequent(["b", "a", "b", "a"], 1) == ["a"]
assert top_k_frequent([1, 1, 2], 5) == [1, 2]
assert top_k_frequent(["x"], 0) == []
try:
    top_k_frequent([1], -1)
    raise AssertionError("Expected negative k to fail")
except ValueError:
    pass
```

**Data-platform connection:** This is a frequent operational analytics problem. Discuss cardinality, windowing, state retention, skew, and whether approximate answers are acceptable.
