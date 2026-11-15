# Searching and Sorting Fundamentals

**Difficulty:** Foundation  
**Topics:** Searching, sorting, algorithm selection

## 1. Concept and when to use

Searching locates an item or boundary; sorting establishes order so later operations become simpler. Before choosing an algorithm, ask whether the input is already sorted, whether the original order must be preserved, whether mutation is allowed, and whether the data fits in memory.

For `n` records, comparison sorting has an Ω(n log n) lower bound in the general case. Python's `list.sort()` and `sorted()` use Timsort: stable, adaptive, and O(n log n) worst-case.

## 2. Baseline and optimized approaches

### Baseline: repeated linear search

```python
def contains_all_linear(source, queries):
    return [any(x == q for x in source) for q in queries]
```
For `n` source items and `m` queries: O(mn) time, O(m) output space.

### Optimized: sort once, binary-search each query

```python
from bisect import bisect_left

def contains_all_sorted(source, queries):
    ordered = sorted(source)
    answer = []
    for q in queries:
        i = bisect_left(ordered, q)
        answer.append(i < len(ordered) and ordered[i] == q)
    return answer
```
O(n log n + m log n) time; O(n + m) space including sorted copy and output. If source is reused for many query batches, the one-time sort can pay off. If source is already sorted, omit the sort.

### When a hash set is better

```python
def contains_all_hash(source, queries):
    lookup = set(source)
    return [q in lookup for q in queries]
```
Expected O(n + m) time and O(n + m) space. Use this when order and range queries do not matter. Explain expected-case hash behavior rather than promising strict worst-case O(1).

## 3. Dry run

`source=[8, 2, 5, 2]`, `queries=[2, 7]`. Sorting produces `[2, 2, 5, 8]`. `bisect_left(..., 2)` returns index 0 and confirms a match. For 7 it returns index 3, where the value is 8, so the answer is false.

## 4. Complexity and trade-offs

| Method | Preprocessing | Each query | Extra space |
|---|---:|---:|---:|
| Linear scan | O(1) | O(n) | O(1) |
| Sort + binary search | O(n log n) | O(log n) | O(n) if copied |
| Hash set | O(n) expected | O(1) expected | O(n) |

## 5. Interviewer–candidate discussion

**Interviewer:** Would you always sort before searching?

**Candidate:** No. If there is one query, linear scan may be cheaper. If there are many membership queries, a set may be best. If range queries or ordered traversal matter, sorting plus binary search is useful.

**Interviewer:** What if records are large objects?

**Candidate:** Sort or index compact keys/IDs rather than repeatedly comparing large payloads. I would also account for index memory and the cost of maintaining it as records change.

## 6. Edge cases and common mistakes

- Empty source and empty query list.
- Duplicate keys and whether the first, last, or any match is required.
- Sorting records with equal keys: use a stable sort if original relative order matters.
- Sorting mixed incomparable types raises `TypeError` in Python 3.
- Forgetting preprocessing cost when reporting total complexity.
- Mutating caller-owned data with `list.sort()` when a copy was expected.

## 7. Follow-ups and variations

1. How would you support frequent inserts and queries? Consider a balanced tree or database index.
2. How do you find all records in a key range? Use lower and upper bounds.
3. How would you search data too large for RAM? Discuss external sorting, partitioning and storage-engine indexes.

## 8. Practical exercises and tests

```python
assert contains_all_linear([], [1]) == [False]
assert contains_all_sorted([3, 1, 2], [2, 4]) == [True, False]
assert contains_all_hash([2, 2, 2], [2, 3]) == [True, False]
```

**Principal-level connection:** A data platform index is a deliberate trade-off between read latency, write amplification, storage and operational cost.
