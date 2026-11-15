# Kth Largest Element and Top-K Selection

**Difficulty:** Medium  
**Topics:** Heap, quickselect, selection algorithms

## 1. Problem statement

Find the kth largest element, where k is 1-indexed. Clarify whether duplicates count as separate ranked elements; the implementations here do count duplicates.

## 2. Approaches

### Baseline: full sort

```python
def kth_largest_sort(nums, k):
    if not 1 <= k <= len(nums):
        raise ValueError("invalid k")
    return sorted(nums, reverse=True)[k - 1]
```
O(n log n) time, O(n) space.

### Optimized for small k: min-heap of size k

```python
import heapq

def kth_largest_heap(nums, k):
    if not 1 <= k <= len(nums):
        raise ValueError("invalid k")
    heap = []
    for value in nums:
        if len(heap) < k:
            heapq.heappush(heap, value)
        elif value > heap[0]:
            heapq.heapreplace(heap, value)
    return heap[0]
```
O(n log k) time and O(k) space.

### Alternative: Quickselect

Average O(n) time, worst-case O(n²), O(1) or O(n) auxiliary space depending on partition implementation and whether mutation is allowed. Use randomized pivot or median-of-three to reduce adversarial risk.

## 3. Dry run

`nums=[3,2,1,5,6,4]`, k=2. Heap retains the two largest values seen. After processing all elements, heap contains 5 and 6; its minimum is 5, the second largest.

## 4. Interviewer–candidate discussion

**Interviewer:** Which approach would you choose for k close to n?

**Candidate:** Sorting may be simpler and competitive. Heap selection is attractive when k is small. Quickselect is useful for one selection when average linear time matters.

**Interviewer:** What if values arrive continuously?

**Candidate:** A size-k min-heap supports each new item in O(log k), maintaining the top k seen so far.

## 5. Edge cases and mistakes

Invalid k, duplicates, negative numbers, all equal, k=1, k=n. Do not accidentally return the heap's maximum; for a size-k min-heap the root is the kth largest.

## 6. Tests and variations

```python
assert kth_largest_heap([3, 2, 1, 5, 6, 4], 2) == 5
assert kth_largest_heap([3, 2, 3, 1, 2, 4, 5, 5, 6], 4) == 4
assert kth_largest_heap([-1, -2, -3], 1) == -1
assert kth_largest_heap([7], 1) == 7
```
Follow-ups: top-k frequent values; top-k per partition in distributed systems; merge local top-k results; deterministic tie-breaking.
