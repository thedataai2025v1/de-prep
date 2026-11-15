# Fenwick Tree / Binary Indexed Tree

**Level:** Medium–Hard | **Patterns:** Prefix aggregates, binary indexed tree

## 1. Concept and when to use it

A Fenwick tree maintains prefix sums under point increments and answers prefix or range-sum queries in `O(log n)`. It is simpler and usually more memory-efficient than a segment tree when the operation is additive.

The key bit trick is `i & -i`, which isolates the least significant set bit and determines how far an index covers.

## 2. Approaches

### Brute force

Store the raw array. Point increment is `O(1)`; prefix sum is `O(n)`.

### Prefix-sum array

Prefix queries are `O(1)`, but changing one value requires rebuilding `O(n)` entries.

### Optimized Fenwick tree

This implementation uses one-based internal indexing while exposing zero-based external indices.

```python
class FenwickTree:
    def __init__(self, n):
        if n < 0:
            raise ValueError("n must be non-negative")
        self.n = n
        self.bit = [0] * (n + 1)

    def add(self, index, delta):
        if not 0 <= index < self.n:
            raise IndexError(index)
        i = index + 1
        while i <= self.n:
            self.bit[i] += delta
            i += i & -i

    def prefix_sum(self, index):
        if index < 0:
            return 0
        if index >= self.n:
            raise IndexError(index)
        total = 0
        i = index + 1
        while i > 0:
            total += self.bit[i]
            i -= i & -i
        return total

    def range_sum(self, left, right):
        if left < 0 or right >= self.n or left > right:
            raise ValueError("Invalid range")
        return self.prefix_sum(right) - self.prefix_sum(left - 1)
```

## 3. Dry run

Create `FenwickTree(5)` and add values 2, 1, 5, 3, 4 at indices 0..4.
- `prefix_sum(2)` returns `2+1+5 = 8`.
- `range_sum(1,3)` returns `1+5+3 = 9`.
- Add `2` to index 2.
- `range_sum(1,3)` now returns `1+7+3 = 11`.

## 4. Complexity

- Point increment: `O(log n)`.
- Prefix sum: `O(log n)`.
- Range sum: `O(log n)`.
- Space: `O(n)`.

## 5. Edge cases and mistakes

- External indices are zero-based; internal Fenwick indices are one-based.
- `index & -index` is meaningful only for positive internal indices.
- Range sum is `prefix(right) - prefix(left-1)`.
- The `add` operation is an increment, not assignment. To assign a new value, track the old value and add the difference.
- Standard Fenwick trees are naturally suited to invertible prefix operations such as sum; arbitrary range min with updates is not a drop-in replacement.

## 6. Interviewer–candidate discussion

**Interviewer:** Why does each update touch only logarithmically many cells?

**Candidate:** Each step adds the least significant set bit, moving to a larger covered block. The index grows geometrically in terms of covered ranges, so at most logarithmically many cells are visited.

**Interviewer:** How would you support assignment?

**Candidate:** Keep the current array value, compute `delta = new_value - old_value`, and call `add(index, delta)`.

## 7. Follow-ups

- Build from an initial array in `O(n log n)` or optimize construction.
- Count inversions using coordinate compression plus Fenwick tree.
- Find the k-th smallest frequency position using binary lifting.
- Support range-add / point-query using a difference representation.

## 8. Tests

```python
bit = FenwickTree(5)
for i, value in enumerate([2, 1, 5, 3, 4]):
    bit.add(i, value)
assert bit.prefix_sum(2) == 8
assert bit.range_sum(1, 3) == 9
bit.add(2, 2)
assert bit.range_sum(1, 3) == 11
assert bit.prefix_sum(-1) == 0
```

**Data-platform connection:** Fenwick trees can support online counters, cumulative event counts, and rank queries in a single process. At distributed scale, approximate sketches or partitioned aggregations may be more appropriate.
