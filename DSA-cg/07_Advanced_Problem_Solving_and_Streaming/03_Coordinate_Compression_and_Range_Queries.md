# Coordinate Compression and Range Queries

**Level:** Medium–Hard | **Patterns:** Sorting unique keys, rank mapping, Fenwick/segment trees

## 1. Concept and when to use it

Coordinate compression maps large or sparse values to dense integer ranks while preserving order. For example, values `[-10^9, 42, 10^9]` become `[0,1,2]`.

Use it when an algorithm needs array-indexed structures but input coordinates are large, negative, or sparse. It commonly pairs with Fenwick trees, segment trees, inversion counting, and offline range queries.

## 2. Approaches

### Brute force

Store values in a dictionary or repeatedly scan all observed coordinates. This may be fine for small inputs but makes range indexing and memory allocation awkward.

### Optimized: sorted unique coordinate map

```python
def coordinate_compress(values):
    ordered = sorted(set(values))
    rank = {value: index for index, value in enumerate(ordered)}
    return [rank[value] for value in values], ordered
```

For a range query over original coordinates `[low, high]`, use binary search to find the compressed index boundaries:

```python
from bisect import bisect_left, bisect_right

def compressed_range_indices(sorted_unique, low, high):
    left = bisect_left(sorted_unique, low)
    right = bisect_right(sorted_unique, high) - 1
    if left > right:
        return None
    return left, right
```

## 3. Detailed dry run

Input values `[1000000000, -5, 1000000000, 42, -5]`.
- Unique sorted coordinates: `[-5, 42, 1000000000]`.
- Mapping: `-5→0`, `42→1`, `1000000000→2`.
- Compressed input: `[2,0,2,1,0]`.
- Original range `[-5,42]` maps to compressed inclusive indices `(0,1)`.

## 4. Complexity

Compression takes `O(n log n)` time and `O(n)` space. Each later rank lookup is `O(1)` with a dictionary; boundary lookup by binary search is `O(log n)`.

## 5. Edge cases and common mistakes

- Preserve the sorted original coordinates if later converting ranks back.
- Compression preserves order, not distance. The gap between ranks 0 and 1 is not necessarily the same numeric distance as between 1 and 2.
- For inclusive versus half-open range queries, use the appropriate `bisect_left`/`bisect_right` boundary.
- Include every coordinate that future updates may touch; unseen online coordinates require rebuilding or a dynamic ordered structure.
- Duplicate values must map to the same rank.

## 6. Interviewer–candidate discussion

**Interviewer:** Can you use compressed ranks to calculate the physical length of an interval?

**Candidate:** Not by subtracting ranks. Compression preserves order but discards distances. I must use the original coordinate values to calculate lengths.

**Interviewer:** When is compression impossible?

**Candidate:** It is possible for any finite known set. The limitation is online discovery: if new coordinates arrive later, a static mapping may need rebuilding.

## 7. Follow-ups

- Count inversions using coordinate compression plus Fenwick tree.
- Compress interval endpoints while preserving half-open semantics.
- Handle a known batch of updates and queries offline.
- Compare compression with hash maps and balanced ordered maps.

## 8. Practical exercises and tests

```python
compressed, coords = coordinate_compress([10**9, -5, 10**9, 42, -5])
assert coords == [-5, 42, 10**9]
assert compressed == [2, 0, 2, 1, 0]
assert compressed_range_indices(coords, -5, 42) == (0, 1)
assert compressed_range_indices(coords, 0, 0) is None
assert compressed_range_indices(coords, -100, 10**9) == (0, 2)
```

**Data-platform connection:** Compression can make sparse event timestamps, IDs, or price levels usable in indexed structures. Do not mistake compressed rank for elapsed time or physical distance.
