# K-Way Merge of Sorted Streams

**Level:** Medium–Hard | **Patterns:** Min-heap, lazy multiway merge, iterators

## 1. Concept and when to use it

Merge `k` already-sorted input sequences into one sorted output. Instead of concatenating and sorting all `N` items, keep only the next unconsumed item from each source in a min-heap.

Use it for merging sorted files, event partitions, database result streams, and external sorting.

## 2. Approaches

### Brute force

Concatenate all items and sort: `O(N log N)` time and `O(N)` storage.

### Optimized: heap of stream heads

```python
import heapq

def merge_sorted_streams(streams):
    heap = []
    iterators = [iter(stream) for stream in streams]

    for stream_id, iterator in enumerate(iterators):
        try:
            value = next(iterator)
            heapq.heappush(heap, (value, stream_id))
        except StopIteration:
            pass

    result = []
    while heap:
        value, stream_id = heapq.heappop(heap)
        result.append(value)

        try:
            next_value = next(iterators[stream_id])
            heapq.heappush(heap, (next_value, stream_id))
        except StopIteration:
            pass

    return result
```

## 3. Detailed dry run

Streams: `[1,4,7]`, `[2,5]`, `[3,6,8]`.
- Heap begins with `(1,stream0)`, `(2,stream1)`, `(3,stream2)`.
- Pop 1; advance stream 0 and push 4.
- Pop 2; advance stream 1 and push 5.
- Pop 3; advance stream 2 and push 6.
- Continue in order until all streams are exhausted.
Result: `[1,2,3,4,5,6,7,8]`.

## 4. Complexity

Each of `N` values is pushed and popped once, with heap size at most `k`: `O(N log k)` time and `O(k)` heap space, excluding the output list. A true streaming generator can avoid storing the full output.

## 5. Edge cases and common mistakes

- Empty list of streams and empty individual streams.
- Equal values: include `stream_id` as a tie-break so Python never tries to compare iterator objects.
- Each input must already be sorted; validate only if required because full validation costs extra.
- If records are ordered by composite keys, use the same key consistently.
- Backpressure and slow sources matter in real distributed streams.

## 6. Interviewer–candidate discussion

**Interviewer:** Why not repeatedly merge pairs of streams?

**Candidate:** Pairwise merging is valid but can repeat work depending on merge order. A heap keeps the smallest current head globally and gives `O(N log k)` time.

**Interviewer:** Can you process data without materializing the output?

**Candidate:** Yes. Replace the result list with `yield value` and preserve the heap/iterators between yields.

## 7. Follow-ups

- Implement a generator version.
- Merge records by `(event_time, event_id)`.
- Handle one source that is temporarily empty but may receive more data.
- Discuss watermarks and late events for distributed event-time ordering.

## 8. Practical exercises and tests

```python
assert merge_sorted_streams([[1, 4, 7], [2, 5], [3, 6, 8]]) == [1, 2, 3, 4, 5, 6, 7, 8]
assert merge_sorted_streams([]) == []
assert merge_sorted_streams([[], [1], []]) == [1]
assert merge_sorted_streams([[1, 1], [1, 2]]) == [1, 1, 1, 2]
assert merge_sorted_streams([[-3, 0], [-2, 4]]) == [-3, -2, 0, 4]
```

**Data-platform connection:** This is the core pattern behind merging sorted partition outputs and external sort runs. In a distributed system, global ordering requires coordination and a clear policy for late or stalled partitions.
