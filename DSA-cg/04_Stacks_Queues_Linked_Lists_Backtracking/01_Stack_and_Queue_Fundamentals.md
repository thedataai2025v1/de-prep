# Stack and Queue Fundamentals

**Difficulty:** Foundation–Medium  
**Topics:** LIFO, FIFO, deque, operation complexity

## 1. Concepts and when to use them

A **stack** is Last-In, First-Out (LIFO): the most recently inserted item is removed first. Use it for function-call simulation, undo history, parsing, depth-first traversal and monotonic-stack patterns.

A **queue** is First-In, First-Out (FIFO): the earliest inserted item is removed first. Use it for breadth-first traversal, task scheduling and buffering.

Python choices:
- Stack: `list.append()` and `list.pop()` at the end.
- Queue: `collections.deque.append()` and `popleft()`.
- Avoid `list.pop(0)` for queues because shifting elements costs O(n).

## 2. Baseline and optimized implementations

A queue implemented with a list can append in amortized O(1), but removing index zero is O(n).

```python
from collections import deque

class Stack:
    def __init__(self):
        self._items = []

    def push(self, value):
        self._items.append(value)

    def pop(self):
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()

    def peek(self):
        if not self._items:
            raise IndexError("peek from empty stack")
        return self._items[-1]

    def is_empty(self):
        return not self._items


class Queue:
    def __init__(self):
        self._items = deque()

    def enqueue(self, value):
        self._items.append(value)

    def dequeue(self):
        if not self._items:
            raise IndexError("dequeue from empty queue")
        return self._items.popleft()

    def peek(self):
        if not self._items:
            raise IndexError("peek from empty queue")
        return self._items[0]

    def is_empty(self):
        return not self._items
```

## 3. Dry run

Stack: push 4, push 7, push 9 gives `[4,7,9]`; pop returns 9, leaving `[4,7]`.

Queue: enqueue A, B, C gives `[A,B,C]`; dequeue returns A, leaving `[B,C]`.

## 4. Complexity

| Operation | Stack | Queue (`deque`) |
|---|---:|---:|
| Insert | O(1) amortized | O(1) |
| Remove | O(1) amortized | O(1) |
| Peek | O(1) | O(1) |
| Space | O(n) | O(n) |

## 5. Interviewer–candidate discussion

**Interviewer:** Why not use a Python list as a queue?

**Candidate:** `pop(0)` shifts the remaining elements, making each dequeue O(n). A deque provides approximately O(1) operations at both ends.

**Interviewer:** When is a stack better than recursion?

**Candidate:** An explicit stack avoids Python recursion-depth limits and makes traversal state visible.

## 6. Edge cases and common mistakes

Empty pop/dequeue, empty peek, duplicate values, `None` as a legitimate value, and confusing FIFO with LIFO. State the empty-structure contract: return a sentinel, raise an exception, or return an optional result.

## 7. Tests and exercises

```python
s = Stack()
s.push(4); s.push(7)
assert s.peek() == 7
assert s.pop() == 7
assert s.pop() == 4
assert s.is_empty()

q = Queue()
q.enqueue("A"); q.enqueue("B")
assert q.dequeue() == "A"
assert q.peek() == "B"
assert q.dequeue() == "B"
assert q.is_empty()
```

Follow-ups: implement a bounded circular queue; compare deque with a ring buffer; discuss thread-safe producer-consumer queues and backpressure.
