# Greedy Algorithms and Interval Scheduling

**Level:** Medium–Hard | **Patterns:** Greedy choice, sorting, exchange argument

## 1. Concept and when to use it

A greedy algorithm makes the locally best choice at each step and never revisits that decision. It is useful when a locally optimal choice can be shown to preserve a globally optimal solution. Common signals include selecting the maximum number of compatible activities, minimizing lateness, or choosing the next event that frees a resource earliest.

Greedy is not a synonym for “intuitive.” You need a correctness argument—often an **exchange argument**: show that an optimal solution can be transformed to use the greedy choice without becoming worse.

## 2. Problem

Given intervals `[start, end)`, select the maximum number of pairwise non-overlapping intervals. Assume an interval ending at time `t` is compatible with one starting at `t`.

## 3. Approaches

### Brute force

Explore every subset of intervals, check whether it is compatible, and retain the largest. There are `2^n` subsets; checking a subset costs up to `O(n log n)` if sorted or `O(n²)` naively. This is useful only for tiny inputs and as a reference oracle.

### Optimized: sort by finishing time

1. Sort intervals by end time, breaking ties by start.
2. Select the first interval.
3. Repeatedly select the next interval whose start is at least the end of the last selected interval.

Why it works: among all intervals available at a decision point, choosing the one that finishes earliest leaves at least as much room for every future interval. By exchanging the first interval of an optimal solution with the greedy first interval, feasibility and cardinality are preserved.

```python
def max_non_overlapping(intervals):
    if not intervals:
        return 0

    ordered = sorted(intervals, key=lambda item: (item[1], item[0]))
    count = 0
    last_end = float("-inf")

    for start, end in ordered:
        if start >= last_end:
            count += 1
            last_end = end

    return count
```

## 4. Dry run

Input: `[(1, 3), (2, 4), (3, 5), (6, 7)]`.

Sorted by end: `(1,3), (2,4), (3,5), (6,7)`.
- Select `(1,3)`, `last_end = 3`.
- Reject `(2,4)` because `2 < 3`.
- Select `(3,5)` because touching endpoints are allowed.
- Select `(6,7)`.
- Result: `3`.

## 5. Complexity

- Sorting: `O(n log n)`.
- Scan: `O(n)`.
- Total: `O(n log n)` time, `O(n)` for the sorted copy (or `O(1)` extra if sorting in place, excluding sort implementation details).

## 6. Edge cases and common mistakes

- Empty input returns zero.
- A single interval returns one.
- Equal end times: choosing either is safe; deterministic tie-breaking helps reproducibility.
- Clarify whether intervals are closed `[start, end]` or half-open `[start, end)`. The compatibility comparison changes.
- Do not sort by start time for this maximum-cardinality problem; earliest finish is the key.
- Negative timestamps are valid; initialize `last_end` to negative infinity, not zero.

## 7. Interviewer–candidate discussion

**Interviewer:** Why not select the shortest-duration interval?

**Candidate:** Short duration does not necessarily leave the most future capacity. The earliest finishing interval does. For example, a short interval that starts late may block an earlier long sequence.

**Interviewer:** How do you justify the greedy choice?

**Candidate:** Take an optimal schedule whose first interval is not the earliest-finishing one. Replace its first interval with the greedy interval. Since the greedy interval ends no later, every later interval in that optimal schedule remains compatible. The number selected does not decrease.

## 8. Follow-ups and variations

1. Return the selected intervals, not just the count.
2. Weighted interval scheduling: when intervals have values, earliest-finish greedy is no longer sufficient; use DP with predecessor search.
3. Minimum number of rooms: sweep-line or min-heap.
4. If touching endpoints conflict, change `start >= last_end` to `start > last_end`.

## 9. Practical exercises and tests

```python
assert max_non_overlapping([]) == 0
assert max_non_overlapping([(1, 2)]) == 1
assert max_non_overlapping([(1, 3), (2, 4), (3, 5), (6, 7)]) == 3
assert max_non_overlapping([(-5, -2), (-2, 0)]) == 2
assert max_non_overlapping([(1, 10), (2, 3), (3, 4), (4, 5)]) == 3
```

**Data-platform connection:** Selecting non-conflicting maintenance windows or deployment windows is an interval scheduling problem. Real systems may add priorities, dependencies, or resource constraints, which can invalidate the simple greedy model.
