# K-Messed Array Sort

**Difficulty:** Medium  
**Topics:** Heap, bounded disorder, sorting

## 1. Problem statement

Each element in an array is at most `k` positions away from its correct position in the fully sorted array. Sort the array efficiently.

## 2. Approaches

### Brute force

```python
def sort_bruteforce(nums):
    return sorted(nums)
```
O(n log n) time, O(n) space for a new result.

### Optimized: min-heap of size at most k+1

```python
import heapq

def sort_k_messed(nums, k):
    if k < 0:
        raise ValueError("k must be non-negative")
    heap = nums[:k + 1]
    heapq.heapify(heap)
    result = []
    for i in range(k + 1, len(nums)):
        result.append(heapq.heappushpop(heap, nums[i]))
    while heap:
        result.append(heapq.heappop(heap))
    return result
```
O(n log(k+1)) time, O(k+1) auxiliary space. For `k=0`, the input is already sorted and the method still works.

## 3. Dry run

Input `[3, 1, 2, 5, 4]`, `k=2`. Heapify `[3,1,2]`; pop smallest 1 when inserting 5. Insert 4 and emit 2, then drain 3,4,5. Result `[1,2,3,4,5]`.

## 4. Why k+1?

The next correct element cannot be farther than `k` positions ahead of its current position. Therefore, among the next `k+1` candidates, the smallest must be the next output.

## 5. Interviewer–candidate discussion

**Interviewer:** What if k is close to n?

**Candidate:** Heap complexity approaches O(n log n), so Python's Timsort is likely simpler and competitive. I would benchmark rather than assume the heap wins.

**Interviewer:** Can you sort in place?

**Candidate:** The presented version returns a new list. We can write output back into the input, but must be careful not to overwrite unread elements.

## 6. Edge cases and mistakes

Empty input, `k=0`, `k>=n`, negative k, duplicates, already sorted input. Do not initialize only k heap items; the window is k+1. State whether mutation is permitted.

## 7. Tests and variations

```python
assert sort_k_messed([], 2) == []
assert sort_k_messed([1, 2, 3], 0) == [1, 2, 3]
assert sort_k_messed([3, 1, 2, 5, 4], 2) == [1, 2, 3, 4, 5]
assert sort_k_messed([2, 1, 1], 2) == [1, 1, 2]
```
Variation: merge multiple nearly sorted streams with a heap; discuss k-way merge and external sorting.
