# Segment Tree: Range Queries and Point Updates

**Level:** Hard | **Patterns:** Tree decomposition, associative aggregation

## 1. Concept and when to use it

A segment tree stores aggregates over intervals of an array. It supports point updates and range queries in `O(log n)` for operations such as sum, minimum, maximum, gcd, or any associative combine operation with a suitable identity.

Use it when values change and queries cover arbitrary ranges. A prefix-sum array is simpler for static data but point updates can make many prefix values stale.

## 2. Approaches

### Brute force

Answer each range sum by scanning the requested range: `O(n)` per query; point update is `O(1)`.

### Prefix sums

Static range sum is `O(1)` after `O(n)` preprocessing, but a point update costs `O(n)` to rebuild.

### Optimized: recursive segment tree

```python
class SegmentTree:
    def __init__(self, values):
        self.n = len(values)
        self.tree = [0] * (4 * max(1, self.n))
        if self.n:
            self._build(1, 0, self.n - 1, values)

    def _build(self, node, left, right, values):
        if left == right:
            self.tree[node] = values[left]
            return
        mid = (left + right) // 2
        self._build(node * 2, left, mid, values)
        self._build(node * 2 + 1, mid + 1, right, values)
        self.tree[node] = self.tree[node * 2] + self.tree[node * 2 + 1]

    def update(self, index, value):
        if not 0 <= index < self.n:
            raise IndexError(index)
        self._update(1, 0, self.n - 1, index, value)

    def _update(self, node, left, right, index, value):
        if left == right:
            self.tree[node] = value
            return
        mid = (left + right) // 2
        if index <= mid:
            self._update(node * 2, left, mid, index, value)
        else:
            self._update(node * 2 + 1, mid + 1, right, index, value)
        self.tree[node] = self.tree[node * 2] + self.tree[node * 2 + 1]

    def query(self, ql, qr):
        if not self.n or ql < 0 or qr >= self.n or ql > qr:
            raise ValueError("Invalid query range")
        return self._query(1, 0, self.n - 1, ql, qr)

    def _query(self, node, left, right, ql, qr):
        if ql <= left and right <= qr:
            return self.tree[node]
        mid = (left + right) // 2
        result = 0
        if ql <= mid:
            result += self._query(node * 2, left, mid, ql, qr)
        if qr > mid:
            result += self._query(node * 2 + 1, mid + 1, right, ql, qr)
        return result
```

## 3. Dry run

Values `[2, 1, 5, 3]`.
- Build root sum 11; children represent `[2,1]` sum 3 and `[5,3]` sum 8.
- Query `[1,3]`: combine index 1 (1) and right child (8), result 9.
- Update index 2 to 7: affected leaf changes 5→7; ancestors recompute.
- Query `[1,3]` now returns 11.

## 4. Complexity

- Build: `O(n)`.
- Point update: `O(log n)`.
- Range query: `O(log n)` for this sum tree.
- Space: `O(n)` (allocated as roughly `4n` slots).

## 5. Edge cases and common mistakes

- Empty arrays: queries/updates are invalid.
- Define whether query ranges are inclusive; this implementation uses inclusive `[ql, qr]`.
- Do not recurse into a child that does not intersect the query.
- Use the correct identity: zero for sum, positive infinity for minimum, negative infinity for maximum.
- For non-commutative operations, preserve left-to-right combine order.
- Lazy propagation is required for efficient range updates; this implementation supports point updates only.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is a query logarithmic if it may visit multiple nodes?

**Candidate:** A contiguous query decomposes into a logarithmic number of canonical tree segments. Only boundary paths are partially explored; fully covered subtrees return immediately.

**Interviewer:** Could a Fenwick tree solve this?

**Candidate:** For prefix sums and point increments, yes, with less code and memory. Segment trees are more general, especially for min/max and richer associative operations.

## 7. Follow-ups

- Implement range minimum.
- Add lazy propagation for range-add/range-sum.
- Support point increments rather than assignment.
- Use an iterative segment tree.
- Discuss persistent segment trees for historical snapshots.

## 8. Tests

```python
tree = SegmentTree([2, 1, 5, 3])
assert tree.query(0, 3) == 11
assert tree.query(1, 3) == 9
tree.update(2, 7)
assert tree.query(1, 3) == 11
assert tree.query(2, 2) == 7
try:
    tree.query(0, 4)
    raise AssertionError("Expected invalid range to fail")
except ValueError:
    pass
```

**Data-platform connection:** A segment tree is useful for mutable range aggregates in an in-memory service. For distributed analytics, partitioning, shuffles, and storage layout often dominate; do not assume an in-process segment tree directly solves distributed range aggregation.
