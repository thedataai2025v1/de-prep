# Merge K Sorted Lists Using a Heap

**Difficulty:** Hard  
**Topics:** K-way merge, min-heap, linked lists / sorted streams

## 1. Concept and when to use

Merge k individually sorted sequences into one sorted sequence. This is common in external sorting, merging sorted files, log streams and distributed query execution.

## 2. Approaches

### Baseline: concatenate and sort

```python
def merge_k_sort(lists):
    values = [x for seq in lists for x in seq]
    return sorted(values)
```
If N is the total number of elements: O(N log N) time and O(N) space.

### Optimized: k-way heap merge

```python
import heapq

def merge_k_sorted(lists):
    heap = []
    for list_id, seq in enumerate(lists):
        if seq:
            heapq.heappush(heap, (seq[0], list_id, 0))

    result = []
    while heap:
        value, list_id, index = heapq.heappop(heap)
        result.append(value)
        next_index = index + 1
        if next_index < len(lists[list_id]):
            heapq.heappush(
                heap,
                (lists[list_id][next_index], list_id, next_index)
            )
    return result
```
O(N log k) time and O(k+N) space including output; heap itself is O(k).

## 3. Dry run

Lists `[[1,4,7],[2,5],[3,6]]`: heap begins with 1,2,3. Pop 1 and add 4; pop 2 and add 5; pop 3 and add 6; continue until all streams are exhausted. Result `[1,2,3,4,5,6,7]`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why include list_id and index in the heap tuple?

**Candidate:** They provide deterministic tie-breaking and prevent Python from trying to compare arbitrary payload objects when values tie.

**Interviewer:** What if the data is too large for memory?

**Candidate:** Stream one current item per input source and write output incrementally. This keeps heap memory O(k), excluding buffers.

## 5. Edge cases and mistakes

No lists, empty inner lists, duplicate values, one list, unequal lengths. Do not push the entire input into the heap. Ensure the source sequences are sorted under the same comparator.

## 6. Tests and variations

```python
assert merge_k_sorted([]) == []
assert merge_k_sorted([[], []]) == []
assert merge_k_sorted([[1, 4], [2, 5], [3, 6]]) == [1, 2, 3, 4, 5, 6]
assert merge_k_sorted([[1, 1], [1], [0, 2]]) == [0, 1, 1, 1, 2]
```
Follow-ups: merge k linked lists; external merge sort; distributed merge with network failures and backpressure; stable ordering for equal keys.
