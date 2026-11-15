# Linked List Cycle Detection and Cycle Entry

**Difficulty:** Medium  
**Topics:** Floyd's tortoise and hare, fast/slow pointers

## 1. Problem statement

Determine whether a linked list contains a cycle. A follow-up asks for the node where the cycle begins.

## 2. Approaches

### Brute force: visited-node set

```python
def has_cycle_set(head):
    seen = set()
    while head:
        if head in seen:
            return True
        seen.add(head)
        head = head.next
    return False
```
O(n) time and O(n) space.

### Optimized: Floyd's algorithm

```python
def detect_cycle_entry(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            seeker = head
            while seeker is not slow:
                seeker = seeker.next
                slow = slow.next
            return seeker
    return None


def has_cycle(head):
    return detect_cycle_entry(head) is not None
```
O(n) time and O(1) auxiliary space.

## 3. Dry run and intuition

Suppose the list is `A → B → C → D → E → C`. Slow advances one node; fast advances two. Once both enter the cycle, fast gains one node per iteration and eventually meets slow.

After the meeting, reset one pointer to head. Advance both one step at a time; they meet at the cycle entry. This follows from the distance relationship between the non-cycle prefix and cycle length.

## 4. Interviewer–candidate discussion

**Interviewer:** Why does fast eventually meet slow inside a cycle?

**Candidate:** Once both are in the cycle, their relative speed is one node per iteration. On a finite cycle, that relative movement must eventually wrap to the same position.

**Interviewer:** Why compare node identity rather than node values?

**Candidate:** Distinct nodes may have equal values. A cycle is about revisiting the same object/node.

## 5. Edge cases and mistakes

Empty list, one node without a cycle, self-loop, cycle at head, long non-cyclic prefix. Use `is`, not value equality. Guard both `fast` and `fast.next`.

## 6. Tests

```python
a, b, c = ListNode(1), ListNode(2), ListNode(3)
a.next = b; b.next = c; c.next = b
assert has_cycle(a)
assert detect_cycle_entry(a) is b

x = ListNode(9)
assert not has_cycle(x)
assert detect_cycle_entry(x) is None

y = ListNode(5); y.next = y
assert detect_cycle_entry(y) is y
```

Follow-ups: find cycle length; remove a cycle safely; determine whether two lists intersect; discuss corrupted pointer structures.
