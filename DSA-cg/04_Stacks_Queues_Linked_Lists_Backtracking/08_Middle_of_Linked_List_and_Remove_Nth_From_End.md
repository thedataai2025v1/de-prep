# Middle of Linked List and Remove Nth Node From End

**Difficulty:** Easy–Medium  
**Topics:** Fast/slow pointers, two pointers, dummy node

## 1. Middle node

### Baseline

Count nodes, then traverse to the middle: two O(n) passes, O(1) space.

### Optimized one-pass

```python
def middle_node(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow
```
O(n) time, O(1) space. For even length, this returns the second middle.

## 2. Remove nth node from end

### Baseline

Count length L, then remove position L−n from the start. O(n) time, O(1) space, two passes.

### One-pass two-pointer method

```python
def remove_nth_from_end(head, n):
    if n <= 0:
        raise ValueError("n must be positive")

    dummy = ListNode(0, head)
    fast = slow = dummy

    for _ in range(n):
        fast = fast.next
        if fast is None:
            raise ValueError("n exceeds list length")

    while fast.next:
        fast = fast.next
        slow = slow.next

    slow.next = slow.next.next
    return dummy.next
```
O(n) time and O(1) space. The dummy node handles deletion of the head.

## 3. Dry run

For `1→2→3→4→5`, n=2:
- Move fast two links from dummy: fast at node 2.
- Move fast and slow until fast is node 5.
- Slow is node 3; bypass node 4.
- Result: `1→2→3→5`.

For five nodes, middle-node traversal returns node 3. For four nodes, it returns node 3 (the second middle).

## 4. Interviewer–candidate discussion

**Interviewer:** Why place both pointers at a dummy node?

**Candidate:** It creates a predecessor for the head, so deleting the head uses the same pointer update as any other node.

**Interviewer:** What if n is larger than the list?

**Candidate:** The problem contract may guarantee valid n. This implementation validates and raises `ValueError`.

## 5. Edge cases and mistakes

Empty list, one node, n=1, n equals length, n>length, even/odd list length. Be precise about whether the fast pointer advances n or n+1 links relative to the dummy.

## 6. Tests

```python
assert to_list(middle_node(build_linked_list([1,2,3,4,5]))) == [3,4,5]
assert to_list(middle_node(build_linked_list([1,2,3,4]))) == [3,4]
assert to_list(remove_nth_from_end(build_linked_list([1,2,3,4,5]), 2)) == [1,2,3,5]
assert to_list(remove_nth_from_end(build_linked_list([1]), 1)) == []
```

Follow-ups: remove kth from start; return both middle nodes for even length; delete duplicates; rotate list.
