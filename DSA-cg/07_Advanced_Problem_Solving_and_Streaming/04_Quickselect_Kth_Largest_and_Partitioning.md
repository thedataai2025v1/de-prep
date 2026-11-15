# Quickselect: Kth Largest and Partitioning

**Level:** Medium–Hard | **Patterns:** Partition, randomized selection

## 1. Concept and when to use it

Quickselect finds the element at a given sorted position without fully sorting the array. It repeatedly partitions around a pivot and continues only in the partition containing the target rank.

Use it for medians, percentiles, kth-largest elements, and selection tasks when a complete ordering is unnecessary.

## 2. Approaches

### Brute force

Sort the array and index the desired element: `O(n log n)` time.

### Optimized expected-linear Quickselect

This implementation uses randomized pivots and in-place Lomuto partitioning. It returns the kth largest value.

```python
import random

def find_kth_largest(nums, k, rng=None):
    if not 1 <= k <= len(nums):
        raise ValueError("k must be between 1 and len(nums)")

    values = list(nums)
    target = len(values) - k
    rand = rng if rng is not None else random

    left, right = 0, len(values) - 1
    while left <= right:
        pivot_index = rand.randint(left, right)
        values[pivot_index], values[right] = values[right], values[pivot_index]
        pivot = values[right]

        store = left
        for i in range(left, right):
            if values[i] < pivot:
                values[store], values[i] = values[i], values[store]
                store += 1

        values[store], values[right] = values[right], values[store]

        if store == target:
            return values[store]
        if store < target:
            left = store + 1
        else:
            right = store - 1

    raise RuntimeError("Selection failed unexpectedly")
```

## 3. Detailed dry run

Input `[3,2,1,5,6,4]`, `k=2`; target ascending index is `6-2=4`.
- Choose a pivot and partition values into less-than and greater-or-equal regions.
- Suppose pivot 4 lands at index 3. Target index 4 is to the right, so discard indices 0..3.
- Partition the right side. Pivot 6 may land at index 5; target is left of it.
- Next pivot 5 lands at index 4. Return 5.
The exact pivot sequence varies because selection is randomized; the invariant is that the target remains inside the active range.

## 4. Complexity

Expected time is `O(n)` with randomized pivots. Worst-case time is `O(n²)` (for example, repeatedly choosing extreme pivots). The copied array uses `O(n)` space; partitioning itself is in-place.

## 5. Edge cases and common mistakes

- Validate `k` is one-based and in range.
- Duplicates are valid; the partition condition still yields a correct order statistic.
- Be explicit about kth-largest versus kth-smallest.
- The algorithm mutates its working copy, not the caller's list.
- Randomization improves expected behavior, not the worst-case guarantee.
- For deterministic worst-case linear selection, discuss median-of-medians, noting its implementation complexity.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is Quickselect faster on average than sorting?

**Candidate:** Sorting orders every element. Quickselect only partitions enough to isolate the target rank and ignores the irrelevant side after each partition.

**Interviewer:** What is the worst case?

**Candidate:** Repeatedly poor pivots can reduce the active range by only one item, giving quadratic work.

## 7. Follow-ups

- Return the kth smallest.
- Return the top-k elements in arbitrary order.
- Find median of two unsorted arrays.
- Compare heap-based top-k (`O(n log k)`) with Quickselect.
- Use three-way partitioning for many duplicate values.

## 8. Practical exercises and tests

```python
import random

assert find_kth_largest([3, 2, 1, 5, 6, 4], 2, random.Random(7)) == 5
assert find_kth_largest([3, 2, 3, 1, 2, 4, 5, 5, 6], 4, random.Random(3)) == 4
assert find_kth_largest([9], 1, random.Random(1)) == 9
try:
    find_kth_largest([1, 2], 0)
    raise AssertionError("Expected invalid k")
except ValueError:
    pass
```

**Data-platform connection:** Selecting a percentile or threshold from a batch can avoid sorting all records. For distributed datasets, local selection plus a global merge or approximate quantile sketches may be more appropriate.
