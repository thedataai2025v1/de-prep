# Sweep-Line Algorithm and Event Ordering

**Level:** Medium–Hard | **Patterns:** Sorting events, active set, interval geometry

## 1. Concept and when to use it

A sweep-line algorithm sorts important coordinates or times, then processes them in order while maintaining the active objects. It converts many pairwise comparisons into a sorted sequence of local updates.

Use it for maximum overlap, calendar conflicts, event concurrency, geometric intersections, and time-based resource usage.

## 2. Problem: maximum number of overlapping intervals

Assume half-open intervals `[start, end)`. At equal timestamps, end events must be processed before start events so an interval ending at `t` does not overlap one starting at `t`.

### Brute force

For every interval start, count how many intervals contain that time. This is `O(n²)`.

### Optimized: sorted events

```python
def max_concurrent_intervals(intervals):
    events = []
    for start, end in intervals:
        if start > end:
            raise ValueError("start must not exceed end")
        if start == end:
            continue
        events.append((start, 1))
        events.append((end, -1))

    # -1 end events sort before +1 start events at equal time.
    events.sort()
    active = maximum = 0

    for _, delta in events:
        active += delta
        maximum = max(maximum, active)

    return maximum
```

## 3. Detailed dry run

Intervals: `[(1,5), (2,6), (5,8)]`.
Events in order: `(1,+1), (2,+1), (5,-1), (5,+1), (6,-1), (8,-1)`.
- At 1, active becomes 1.
- At 2, active becomes 2; maximum is 2.
- At 5, the first interval ends: active 1.
- At 5, the third interval starts: active 2.
- At 6, active 1; at 8, active 0.
Maximum concurrency is 2.

## 4. Complexity

There are `2n` events. Sorting costs `O(n log n)` and scanning costs `O(n)`. Space is `O(n)`.

## 5. Edge cases and common mistakes

- Decide whether intervals are half-open or closed before choosing event tie order.
- Empty and zero-length intervals.
- Negative and duplicate timestamps.
- Do not update the maximum before applying the event if the event ordering semantics differ.
- For closed intervals `[start,end]`, starts at a timestamp may need to be processed before ends.
- Validate malformed intervals.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is event tie ordering important?

**Candidate:** It encodes the interval semantics. With half-open intervals, a resource becomes free at its end time and can be reused immediately. Processing the end first avoids counting a false overlap.

**Interviewer:** How does this scale better than pairwise comparison?

**Candidate:** Sorting establishes a global order. We update only the active count as we cross each event rather than comparing every pair.

## 7. Follow-ups and variations

- Return the timestamp where maximum concurrency first occurs.
- Return the actual active intervals at a point.
- Find whether any overlap exists.
- Extend to 2D rectangle overlap; active-set management becomes more complex.
- Support event streams arriving out of order: discuss buffering and watermarks.

## 8. Practical exercises and tests

```python
assert max_concurrent_intervals([]) == 0
assert max_concurrent_intervals([(1, 5), (2, 6), (5, 8)]) == 2
assert max_concurrent_intervals([(1, 2), (2, 3)]) == 1
assert max_concurrent_intervals([(1, 5), (1, 5), (1, 5)]) == 3
assert max_concurrent_intervals([(-4, -1), (-3, 0)]) == 2
```

**Data-platform connection:** A sweep line can compute concurrent jobs, active subscriptions, or overlapping maintenance windows. A production event-time pipeline must also define late-event handling and whether results can be revised.
