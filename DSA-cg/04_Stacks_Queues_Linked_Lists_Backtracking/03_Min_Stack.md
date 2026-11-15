# Min Stack

**Difficulty:** Medium  
**Topics:** Stack augmentation, O(1) minimum query

## 1. Problem statement

Design a stack supporting `push`, `pop`, `top` and `get_min`, each in O(1) time.

## 2. Approaches

### Baseline

Use a normal stack and call `min(stack)` for every minimum query: O(n) per `get_min`.

### Optimized: store value and minimum-so-far

```python
class MinStack:
    def __init__(self):
        self._stack = []

    def push(self, value):
        current_min = value if not self._stack else min(value, self._stack[-1][1])
        self._stack.append((value, current_min))

    def pop(self):
        if not self._stack:
            raise IndexError("pop from empty stack")
        return self._stack.pop()[0]

    def top(self):
        if not self._stack:
            raise IndexError("top from empty stack")
        return self._stack[-1][0]

    def get_min(self):
        if not self._stack:
            raise IndexError("minimum of empty stack")
        return self._stack[-1][1]
```

Each operation is O(1); space O(n). A second stack of minima is another valid design.

## 3. Dry run

Push 5 → `(5,5)`. Push 2 → `(2,2)`. Push 4 → `(4,2)`. Minimum is 2. Pop removes 4; minimum remains 2. Pop removes 2; minimum returns to 5.

## 4. Interviewer–candidate discussion

**Interviewer:** Why store the minimum at every level?

**Candidate:** The top tuple contains the minimum across the entire current prefix of the stack, so the query is constant time and pop naturally restores the previous minimum.

**Interviewer:** Is there a lower-memory alternative?

**Candidate:** A second stack stores only values that are less than or equal to the current minimum. It can use fewer entries when minima are rare, with careful duplicate handling.

## 5. Edge cases and mistakes

Duplicate minimum values, negative values, popping the minimum, empty stack. If using a second min stack, push equal minima too, otherwise popping one duplicate can lose the correct minimum.

## 6. Tests

```python
s = MinStack()
s.push(-2); s.push(0); s.push(-3)
assert s.get_min() == -3
assert s.pop() == -3
assert s.top() == 0
assert s.get_min() == -2
s.push(-2)
assert s.get_min() == -2
```

Follow-ups: max stack; support `get_max`; design a stack with O(1) median (requires a more involved structure); discuss memory overhead.
