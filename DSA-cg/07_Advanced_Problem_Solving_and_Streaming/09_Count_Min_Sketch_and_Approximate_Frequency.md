# Count-Min Sketch: Approximate Frequency Estimation

**Level:** Hard | **Patterns:** Streaming summaries, hash tables, probabilistic bounds

## 1. Concept and when to use it

Count-Min Sketch estimates item frequencies using a small 2D array of counters and several independent hash functions. Each update increments one counter per row. A query returns the minimum of the corresponding counters.

It is useful for high-cardinality streams where exact per-key counters are too expensive. It overestimates frequency (for non-negative updates) because hash collisions add noise; it does not underestimate under the standard model.

## 2. Approaches

### Brute force

Use a dictionary of exact counts. Query and update are expected `O(1)`, but memory is `O(d)` for `d` distinct keys.

### Optimized: fixed-width sketch

The example below uses deterministic SHA-256-based row hashing for repeatable behavior.

```python
import hashlib

class CountMinSketch:
    def __init__(self, width, depth):
        if width <= 0 or depth <= 0:
            raise ValueError("width and depth must be positive")
        self.width = width
        self.depth = depth
        self.table = [[0] * width for _ in range(depth)]

    def _index(self, item, row):
        payload = f"{row}:{item}".encode("utf-8")
        digest = hashlib.sha256(payload).digest()
        return int.from_bytes(digest[:8], "big") % self.width

    def add(self, item, count=1):
        if count < 0:
            raise ValueError("This implementation supports non-negative updates only")
        for row in range(self.depth):
            self.table[row][self._index(item, row)] += count

    def estimate(self, item):
        return min(
            self.table[row][self._index(item, row)]
            for row in range(self.depth)
        )
```

## 3. Detailed dry run

Create a sketch and add `"timeout"` 5 times and `"not_found"` 2 times.
- Each `"timeout"` update increments one cell in every row.
- Query `"timeout"` returns the minimum of its row counters; absent collisions, it returns 5.
- A different key may collide with some of those cells and appear to have a positive estimated count.
- Taking the minimum across rows reduces the chance that all queried cells are heavily contaminated.

## 4. Complexity

With width `w` and depth `d`:
- Update: `O(d)`.
- Query: `O(d)`.
- Space: `O(wd)`, independent of distinct-key count.

A common parameterization uses width proportional to `e/ε` and depth proportional to `ln(1/δ)` to obtain additive error around `εN` with probability at least `1-δ`, under the standard assumptions.

## 5. Edge cases and common mistakes

- This implementation rejects negative updates; deletions require a different design and can break the one-sided error guarantee.
- Counts can grow large; fixed-width integer implementations must consider overflow.
- A sketch estimates frequency, not membership identity.
- Hash quality and independence assumptions matter.
- Small width increases collision error.
- Do not claim an exact top-k from a sketch without a candidate-generation strategy.

## 6. Interviewer–candidate discussion

**Interviewer:** Why take the minimum across rows?

**Candidate:** Each row gives an upper-biased count due to collisions. The minimum is the least contaminated estimate and preserves the one-sided upper-bound behavior.

**Interviewer:** How is this different from a Bloom filter?

**Candidate:** A Bloom filter answers approximate membership. Count-Min Sketch estimates frequencies. Both trade memory for probabilistic behavior but answer different questions.

## 7. Follow-ups

- Use a heavy-hitter algorithm to generate candidate keys, then estimate them with the sketch.
- Compare Count-Min Sketch with Misra–Gries and Space-Saving.
- Explain error parameters `ε` and `δ`.
- Design per-tenant or per-time-window sketches and discuss state expiry.

## 8. Practical exercises and tests

```python
cms = CountMinSketch(width=200, depth=5)
for _ in range(5):
    cms.add("timeout")
for _ in range(2):
    cms.add("not_found")
assert cms.estimate("timeout") >= 5
assert cms.estimate("not_found") >= 2
assert cms.estimate("timeout") >= cms.estimate("not_found")
try:
    cms.add("bad", -1)
    raise AssertionError("Expected negative update to fail")
except ValueError:
    pass
```

**Data-platform connection:** Approximate frequency sketches are useful for monitoring high-cardinality events and identifying hot keys without retaining every distinct key. Explain the error budget to stakeholders before using approximate results for decisions.
