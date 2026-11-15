# Tortoise and Hare (Floyd's Cycle Detection)

**Pattern:** Slow and fast pointers  
**Use cases:** Linked-list cycle detection, cycle entry, middle node, functional-graph cycles.

## 1. Concept

Move one pointer by one step and another by two steps. If a cycle exists in a deterministic linked structure, the fast pointer eventually laps the slow pointer and they meet.

Use only when each node has a well-defined next transition. Clarify whether the task asks for cycle existence, entry node, cycle length or list midpoint.

## 2. Brute force cycle detection

Track visited node identities:
```python
def has_cycle_set(head):
    seen = set()
    current = head
    while current:
        if current in seen:
            return True
        seen.add(current)
        current = current.next
    return False
```

Time O(n), space O(n).

## 3. Optimized Floyd detection

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

Time O(n), auxiliary space O(1). Compare node identity (`is`), not node values.

## 4. Dry run

For `A → B → C → D → B`:
- Start slow=A, fast=A.
- Move: slow=B, fast=C.
- Move: slow=C, fast=B.
- Move: slow=D, fast=D → meet, so a cycle exists.

Exact meeting iteration depends on cycle shape.

## 5. Find cycle entry

After slow and fast meet, reset one pointer to head. Move both one step at a time; their next meeting is the cycle entry.

```python
def cycle_entry(head):
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
```

## 6. Find middle node

```python
def middle_node(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow
```

For even-length lists this returns the second middle. Clarify if the first middle is expected.

## 7. Complexity

- Set approach: O(n) time, O(n) space.
- Floyd: O(n) time, O(1) auxiliary space.
- Middle node: O(n) time, O(1) space.

## 8. Edge cases and mistakes

- Empty list.
- Single node with no cycle.
- Single node pointing to itself.
- Cycle begins at head.
- Cycle begins after several nodes.
- Comparing values instead of object identity.
- Dereferencing `fast.next.next` without checking `fast` and `fast.next`.

## 9. Interviewer–candidate discussion

**Interviewer:** Why do they have to meet?

**Candidate:** Once both pointers are in the cycle, their relative speed is one node per iteration. The distance around the cycle is finite, so the relative position eventually becomes zero modulo the cycle length.

## 10. Follow-ups

- Return cycle entry.
- Return cycle length.
- Find list midpoint.
- Detect repeated state in a deterministic transformation.
- Explain why this does not directly solve arbitrary graph cycle detection with branching.

## 11. Tests

Use a local `Node` class and test: `None`, one node, self-loop, acyclic chain, and cycle beginning at a middle node.
