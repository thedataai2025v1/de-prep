# Meeting Rooms I and II

**Level:** Easy–Medium | **Patterns:** Interval sorting, sweep line, min-heap

## 1. Concept and when to use it

Meeting Rooms I asks whether any intervals overlap. Meeting Rooms II asks for the minimum number of rooms required so all meetings can run. These model capacity planning, worker concurrency, batch slots, and resource allocation.

Assume intervals are half-open `[start, end)`: a meeting ending at 10:00 does not conflict with one starting at 10:00.

## 2. Approaches

### Brute force

For Meeting Rooms I, compare every pair: `O(n²)` time. For Meeting Rooms II, test candidate room assignments by backtracking; this grows exponentially and is not practical.

### Optimized I: sort and compare neighbors

```python
def can_attend_all_meetings(intervals):
    ordered = sorted(intervals)
    for i in range(1, len(ordered)):
        if ordered[i][0] < ordered[i - 1][1]:
            return False
    return True
```

### Optimized II-A: sweep line

Sort all start times and end times separately. At each start, release every room whose meeting has ended. Track active rooms and the maximum active count.

```python
def min_meeting_rooms_sweep(intervals):
    if not intervals:
        return 0

    starts = sorted(start for start, _ in intervals)
    ends = sorted(end for _, end in intervals)
    start_i = end_i = active = maximum = 0

    while start_i < len(starts):
        if starts[start_i] < ends[end_i]:
            active += 1
            maximum = max(maximum, active)
            start_i += 1
        else:
            active -= 1
            end_i += 1

    return maximum
```

### Optimized II-B: min-heap

Keep the end time of each occupied room in a min-heap. If the earliest room is free by the next start, reuse it; otherwise allocate another room.

```python
import heapq

def min_meeting_rooms_heap(intervals):
    if not intervals:
        return 0

    ordered = sorted(intervals)
    endings = []

    for start, end in ordered:
        if endings and endings[0] <= start:
            heapq.heapreplace(endings, end)
        else:
            heapq.heappush(endings, end)

    return len(endings)
```

## 3. Dry run

For `[(0, 30), (5, 10), (15, 20)]`:
- Start 0: allocate room ending at 30.
- Start 5: 30 is not free; allocate second room ending at 10.
- Start 15: earliest end is 10, so reuse that room with end 20.
- Two rooms are sufficient.

## 4. Complexity

- Meeting Rooms I: `O(n log n)` time for sorting, `O(n)` scan.
- Sweep line: `O(n log n)` time and `O(n)` arrays.
- Heap: `O(n log n)` time and up to `O(n)` heap space.

## 5. Edge cases and mistakes

- Empty input needs zero rooms.
- Equal endpoints: use `<=` to release a room for half-open intervals.
- Duplicate intervals require separate rooms.
- Validate or define zero-length intervals.
- In sweep-line code, ensure `end_i` is always valid under the interval assumptions.
- Do not confuse “number of meetings” with “maximum simultaneous meetings.”

## 6. Interviewer–candidate discussion

**Interviewer:** Why does the heap size equal the room count?

**Candidate:** Each heap entry represents a currently assigned room's latest ending meeting. We reuse the room with the earliest end whenever possible. If no room is free, a new room is necessary. The heap therefore represents the active room allocations.

**Interviewer:** Which solution would you choose?

**Candidate:** The heap is easy to extend when I need to return room assignments. The sweep-line version is compact when only the peak concurrency is needed.

## 7. Follow-ups

- Return the room number assigned to every meeting.
- Process streaming arrivals: what state must be retained?
- Support cancellations and arbitrary updates: a static sort is insufficient; discuss balanced ordered structures or event-driven scheduling.
- What changes if endpoints are closed intervals?

## 8. Tests

```python
assert can_attend_all_meetings([]) is True
assert can_attend_all_meetings([(0, 5), (5, 9)]) is True
assert can_attend_all_meetings([(0, 5), (4, 9)]) is False
assert min_meeting_rooms_heap([]) == 0
assert min_meeting_rooms_heap([(0, 30), (5, 10), (15, 20)]) == 2
assert min_meeting_rooms_sweep([(7, 10), (2, 4)]) == 1
assert min_meeting_rooms_heap([(1, 5), (1, 5), (1, 5)]) == 3
```

**Data-platform connection:** Peak concurrent jobs is a capacity metric. A production scheduler additionally considers CPU/memory vectors, priority, affinity, and dependencies—not just one-dimensional time overlap.
