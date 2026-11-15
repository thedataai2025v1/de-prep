# Implement a Queue Using Two Stacks

**Difficulty:** Easy–Medium  
**Topics:** Amortized analysis, stack simulation

## 1. Problem statement

Implement FIFO queue operations using only stack operations. Use one stack for incoming values and one for outgoing values.

## 2. Approaches

### Costly dequeue baseline

Move all elements between stacks on every operation. This can make each dequeue O(n).

### Optimized lazy transfer

```python
class MyQueue:
    def __init__(self):
        self.in_stack = []
        self.out_stack = []

    def push(self, value):
        self.in_stack.append(value)

    def _prepare_out(self):
        if not self.out_stack:
            while self.in_stack:
                self.out_stack.append(self.in_stack.pop())

    def pop(self):
        self._prepare_out()
        if not self.out_stack:
            raise IndexError("pop from empty queue")
        return self.out_stack.pop()

    def peek(self):
        self._prepare_out()
        if not self.out_stack:
            raise IndexError("peek from empty queue")
        return self.out_stack[-1]

    def empty(self):
        return not self.in_stack and not self.out_stack
```

Push O(1); pop/peek O(1) amortized, O(n) worst case for one transfer; space O(n).

## 3. Dry run

Push 1,2,3 → `in=[1,2,3]`, `out=[]`. First pop transfers all: `out=[3,2,1]`; pop returns 1. Next pop returns 2 without transfer. Push 4 goes to `in`; pop returns 3, then next pop transfers 4 and returns it.

## 4. Interviewer–candidate discussion

**Interviewer:** Why is dequeue amortized O(1)?

**Candidate:** Every element moves from input stack to output stack at most once and is popped once. Across m operations the total transfer work is O(m), although an individual dequeue may cost O(n).

**Interviewer:** Can we make every individual operation worst-case O(1)?

**Candidate:** Not with this simple two-stack transfer design; real-time queue designs use more complex scheduling.

## 5. Edge cases and mistakes

Empty queue, alternating pushes/pops, pushes after a transfer, repeated peek, and transferring only when output stack is empty. Do not transfer on every pop.

## 6. Tests

```python
q = MyQueue()
q.push(1); q.push(2)
assert q.peek() == 1
assert q.pop() == 1
assert not q.empty()
q.push(3)
assert q.pop() == 2
assert q.pop() == 3
assert q.empty()
```

Follow-ups: stack using two queues; implement a deque; compare amortized and worst-case guarantees.
