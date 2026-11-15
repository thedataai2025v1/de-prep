# Fast Track: Binary Search, Heaps and Intervals

## 1. Binary search

Use binary search when the search space is ordered or a monotonic predicate exists.

```python
def binary_search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
```

Time O(log n), space O(1). For “binary search on answer”, define a monotonic feasibility function and find the first/last feasible value.

## 2. Heaps

A heap is useful when repeatedly retrieving the smallest or largest item.
- Push/pop: O(log n).
- Peek: O(1).
- Heapify an array: O(n).
- Top-k: often O(n log k) with a size-k heap.

Python `heapq` is a min-heap. A max-heap can be simulated with negated numeric keys, with care for tuple tie-breakers and non-numeric objects.

## 3. Intervals

Common tasks:
- Merge overlapping intervals.
- Insert a new interval.
- Find maximum concurrent events.
- Allocate minimum meeting rooms.

For merge intervals, sort by start then scan once: O(n log n) total due to sorting, O(n) output space.

```python
def merge_intervals(intervals):
    intervals = sorted(intervals)
    merged = []
    for start, end in intervals:
        if not merged or start > merged[-1][1]:
            merged.append([start, end])
        else:
            merged[-1][1] = max(merged[-1][1], end)
    return merged
```

Clarify whether touching endpoints overlap. The comparison may be `>` or `>=` depending on the interval convention.

## 4. Pattern selection

- Ordered lookup → binary search.
- Repeated min/max extraction → heap.
- Top-k with small k → bounded heap.
- Overlap / scheduling → intervals and possibly sweep line.
- First feasible capacity / rate → binary search on answer.

## 5. Follow-ups

Discuss duplicates, empty input, stable tie-breaking, online versus offline input, and whether the entire dataset fits in memory.
