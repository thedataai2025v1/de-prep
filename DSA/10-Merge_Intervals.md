# Merge Intervals

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Sorting, Interval |
| **Companies** | Google, Amazon, Microsoft, Facebook, Bloomberg, Uber |
| **Interview Frequency** | High (Top 100 Must-Know Problems) |

---

## 1. Problem Statement

Given an array of intervals where `intervals[i] = [start_i, end_i]`, **merge all overlapping intervals** and return an array of the non-overlapping intervals that cover all intervals in the input.

**Example:**

```
Input:  intervals = [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]

Explanation: [1,3] and [2,6] overlap, so they are merged into [1,6].
```

**Constraints (LeetCode):**
- `1 <= intervals.length <= 10^4`
- `0 <= start_i <= end_i <= 10^4`

---

## 2. Approaches

### Approach 1: Sort by Start, Then Merge (⭐ Optimal)

**Idea:** Sort intervals by their `start` value. Then iterate through the sorted list:
- If the current interval overlaps with the last merged interval (`curr.start <= last.end`), merge by updating `last.end = max(last.end, curr.end)`.
- Otherwise, add the current interval as a new entry in the result.

```python
def merge(intervals):
    if not intervals:
        return []

    # Sort by start time
    intervals.sort(key=lambda x: x[0])

    merged = [intervals[0]]

    for curr in intervals[1:]:
        last = merged[-1]
        # Overlap detected: curr starts before or at last's end
        if curr[0] <= last[1]:
            last[1] = max(last[1], curr[1])
        else:
            merged.append(curr)

    return merged
```

| Metric | Value |
|---|---|
| **Time** | O(n log n) — dominated by sorting |
| **Space** | O(n) — output storage (or O(1) if sorting in-place) |

**Pros:**
- Simple and efficient
- Handles all overlap patterns
- Easily adaptable to interval intersection problems

**Cons:**
- Sorting step is the bottleneck
- Modifies input (can be avoided by copying)

---

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Sort + Merge** ⭐ | O(n log n) | O(n) | **Interview choice** — only viable solution |

*Note: There is no alternative with better asymptotic complexity because sorting is required to identify overlaps efficiently. Any unsorted approach would be O(n²).*

---

## 3. Edge Cases

> **Always mention these in an interview to show thoroughness.**

| Edge Case | Input | Expected Behavior |
|---|---|---|
| Empty input | `[]` | Return `[]` |
| Singleton | `[[1,5]]` | Return `[[1,5]]` |
| No overlap | `[[1,2],[3,4],[5,6]]` | Return as-is |
| All overlapping | `[[1,4],[2,5],[3,6]]` | Return `[[1,6]]` |
| One interval engulfs others | `[[1,10],[2,3],[4,5]]` | Return `[[1,10]]` |
| Start equals end (point intervals) | `[[1,2],[2,3]]` | Return `[[1,3]]` if touches merge, else depends on definition |
| Unsorted input | `[[5,8],[1,3],[2,6]]` | Sorting ensures correct merging |

---

## 4. Interview Talking Points

- **"The sort + merge approach is optimal because we must at least read all intervals (O(n)) and sorting is O(n log n), which cannot be improved for comparison-based sorting."**
- **Data Engineering Connection:** Merging intervals models combining overlapping time windows in event logs, consolidating batch processing time slots, merging calendar events, and unifying overlapping data partitions in distributed storage.

---

## 5. Interview Follow-Up Questions

**Q1. What if intervals are already sorted by start time?**
→ You can skip the sort step and achieve O(n) time. This is a common optimization when working with pre-sorted data (e.g., time-series logs).

**Q2. How would you insert a new interval into an already merged list?**
→ Use binary search to find the insertion position, then merge with neighboring overlapping intervals. This is LeetCode's **Insert Interval** problem (Medium).

**Q3. What if intervals can have negative start/end values?**
→ The algorithm works unchanged — sorting and comparison handle negative values identically.

**Q4. How would you find the intersection of two interval lists instead of merging?**
→ Use two-pointer technique on sorted lists. Compare overlaps where both intervals contribute, and advance the pointer with the earlier end time.

**Q5. How would you handle this at scale (millions of intervals)?**
→ Use distributed sorting (e.g., Spark `sortBy` or MapReduce), then merge in parallel by processing sorted partitions. Alternatively, use an interval tree for dynamic insertion and query.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Forgetting to sort input | Overlapping intervals may not be adjacent | Always sort by start time first |
| Using `curr[0] < last[1]` instead of `<=` | Intervals that touch at endpoint (e.g., [1,2],[2,3]) are not merged, which may be incorrect depending on requirements | Use `<=` for inclusive merging |
| Not updating `last[1]` with `max` | If current interval ends after last, the merged end is wrong | `last[1] = max(last[1], curr[1])` |
| Modifying original list inadvertently | Sorting changes the input; may cause side effects in production | Work on a copy: `sorted(intervals)` |
| Returning nested list incorrectly | Merging logic works but final list structure is wrong | Verify with edge cases |

---

## 7. Related Problems

| Problem | Key Concept | Difficulty |
|---|---|---|
| [Insert Interval](https://leetcode.com/problems/insert-interval/) | Insert a new interval into merged list | Medium |
| [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/) | Remove minimum intervals to make rest non-overlapping | Medium |
| [Meeting Rooms](https://leetcode.com/problems/meeting-rooms/) | Check if a person can attend all meetings | Easy |
| [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) | Minimum number of conference rooms required | Medium |
| [Interval List Intersections](https://leetcode.com/problems/interval-list-intersections/) | Intersection of two sorted interval lists | Medium |
| [Data Stream as Disjoint Intervals](https://leetcode.com/problems/data-stream-as-disjoint-intervals/) | Dynamic interval merging from stream | Hard |

---

*Last updated: 2025 — for data engineering interview preparation.*