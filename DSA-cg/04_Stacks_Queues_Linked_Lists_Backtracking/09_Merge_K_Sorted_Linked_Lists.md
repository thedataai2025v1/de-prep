# Merge K Sorted Linked Lists

**Difficulty:** Hard  
**Topics:** Min-heap, divide and conquer, linked-list pointer rewiring

## 1. Concept and when to use

Merge k individually sorted linked lists into one sorted linked list. This is a canonical k-way merge problem.

## 2. Approaches

### Baseline: collect values and sort

Traverse every list, collect values, sort them and build a new list: O(N log N) time and O(N) extra space, where N is total nodes.

### Optimized: min-heap of current heads

```python
import heapq
from itertools import count

def merge_k_linked_lists(lists):
    heap = []
    sequence = count()

    for node in lists:
        if node:
            heapq.heappush(heap, (node.val, next(sequence), node))

    dummy = ListNode()
    tail = dummy

    while heap:
        _, _, node = heapq.heappop(heap)
        next_node = node.next
        tail.next = node
        tail = node
        if next_node:
            heapq.heappush(heap, (next_node.val, next(sequence), next_node))

    tail.next = None
    return dummy.next
```

O(N log k) time, O(k) heap space. Nodes are reused rather than copied. The sequence counter breaks ties without comparing node objects.

### Alternative: divide and conquer

Pairwise merge lists in rounds: O(N log k) time, O(1) extra space aside from recursion/iteration and output node reuse.

## 3. Dry run

Lists: `1→4→7`, `2→5`, `3→6`.
- Heap contains heads 1,2,3.
- Pop 1; append it; insert 4.
- Pop 2; append it; insert 5.
- Pop 3; append it; insert 6.
- Continue in sorted order until heap is empty.

## 4. Interviewer–candidate discussion

**Interviewer:** Why is the heap size O(k)?

**Candidate:** There is at most one active candidate node per input list.

**Interviewer:** Why preserve a next pointer before rewiring?

**Candidate:** Once `tail.next` is changed, the original traversal link could be lost. Saving `node.next` first keeps the remainder of that list reachable.

## 5. Edge cases and mistakes

No lists, empty lists, duplicate values, one list, negative values, shared nodes (usually excluded by the problem). Avoid comparing node objects on equal values; ensure the final tail points to `None`.

## 6. Tests

```python
a = build_linked_list([1, 4, 7])
b = build_linked_list([2, 5])
c = build_linked_list([3, 6])
assert to_list(merge_k_linked_lists([a, b, c])) == [1, 2, 3, 4, 5, 6]
assert merge_k_linked_lists([]) is None
assert to_list(merge_k_linked_lists([None, build_linked_list([0])])) == [0]
```

Follow-ups: merge two lists iteratively; divide-and-conquer analysis; merge sorted disk files; distributed merge with bounded buffers.
