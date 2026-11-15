# Merge Intervals and Non-overlapping Intervals

**Level:** Medium | **Patterns:** Sort + sweep, greedy interval removal

## 1. Concept and when to use it

Intervals represent time windows, numeric ranges, reservations, and validity periods. Two common tasks are:
- Merge all overlapping intervals into a compact union.
- Remove the minimum number of intervals so the remaining intervals do not overlap.

The two tasks share sorting, but their greedy decisions differ.

## 2. Approaches

### Merge intervals — brute force

Repeatedly compare and merge any overlapping pair until no merge remains. A straightforward implementation can take `O(n²)` or worse due to repeated passes and list rebuilding.

### Merge intervals — optimized

Sort by start. Maintain the current merged interval. If the next start is at most the current end, extend the end; otherwise emit the current interval.

```python
def merge_intervals(intervals):
    if not intervals:
        return []

    ordered = sorted(intervals, key=lambda item: item[0])
    merged = [list(ordered[0])]

    for start, end in ordered[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])

    return [tuple(interval) for interval in merged]
```

### Minimum removals — optimized greedy

Sort by end time and keep the maximum number of compatible intervals. The minimum removals are `n - kept`.

```python
def erase_overlap_intervals(intervals):
    if not intervals:
        return 0

    ordered = sorted(intervals, key=lambda item: item[1])
    kept = 0
    last_end = float("-inf")

    for start, end in ordered:
        if start >= last_end:
            kept += 1
            last_end = end

    return len(ordered) - kept
```

## 3. Dry run

Merge `[(1,3), (2,6), (8,10), (9,12)]`:
- Start with `[1,3]`.
- `[2,6]` overlaps; extend to `[1,6]`.
- `[8,10]` is separate; emit `[1,6]`.
- `[9,12]` overlaps; extend to `[8,12]`.
- Result: `[(1,6), (8,12)]`.

For removals `[(1,2), (2,3), (1,3)]`, retain `(1,2)` and `(2,3)`; remove `(1,3)`. Answer `1`.

## 4. Complexity

Both optimized algorithms sort once: `O(n log n)` time and `O(n)` auxiliary/output space for the sorted/merged representation.

## 5. Edge cases and common mistakes

- Empty and singleton inputs.
- Nested intervals: use `max(current_end, end)`; never shrink the merged end.
- Equal endpoints: decide whether touching intervals overlap. The code merges touching intervals (`start <= end`) but permits touching intervals in the removal problem (`start >= last_end`) under half-open semantics.
- Normalize or reject malformed intervals where `start > end`.
- Do not use start-time sorting for the greedy removal count; earliest finish is the correct choice.

## 6. Interviewer–candidate discussion

**Interviewer:** Why do the sort keys differ?

**Candidate:** Merging requires discovering every interval connected to the current union, so start order makes the scan monotonic. For maximum compatible selection, earliest finish leaves the most remaining time.

**Interviewer:** Can I use the merge result to solve minimum removals?

**Candidate:** No. Merging loses the identity and count of original intervals. It does not tell us the minimum number of original intervals to remove.

## 7. Follow-ups and variations

- Return the actual removed intervals.
- Merge only if overlap length exceeds a threshold.
- Add a new interval to an already sorted disjoint list in `O(n)`.
- Weighted removal becomes a weighted interval scheduling DP problem.
- For very large external datasets, discuss sorting, partitioning, and merge phases.

## 8. Tests

```python
assert merge_intervals([]) == []
assert merge_intervals([(1, 3)]) == [(1, 3)]
assert merge_intervals([(1, 3), (2, 6), (8, 10), (9, 12)]) == [(1, 6), (8, 12)]
assert merge_intervals([(1, 10), (2, 3), (4, 5)]) == [(1, 10)]
assert erase_overlap_intervals([]) == 0
assert erase_overlap_intervals([(1, 2), (2, 3), (1, 3)]) == 1
```

**Data-platform connection:** Merging overlapping validity windows can compact maintenance or retention ranges. Removing conflicting jobs is a scheduling problem; if jobs have business value, use weighted optimization rather than unweighted greedy selection.
