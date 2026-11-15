# Choosing the Right Language for Your Technical Interview

**Type:** Interview preparation lesson  
**Recommended language for this study plan:** Python

## 1. Selection criteria

Choose a language you can use fluently under time pressure. Consider:
- Familiarity with core data structures and standard-library APIs.
- Speed of writing correct code.
- Ease of explaining types, mutation and complexity.
- Availability of a local test environment.
- Whether the role explicitly requires a language.

For a Principal Data Engineer interview, Python is a strong default when the role accepts it. It allows concise expression of algorithms and is widely used for data tooling. If a role specifically tests Scala, Java or SQL, prepare that language separately rather than assuming Python will be accepted.

## 2. Python essentials to know without searching

| Need | Python tool | Typical complexity / caveat |
|---|---|---|
| Dynamic array | `list` | Append amortized O(1); insert at front O(n) |
| Hash map | `dict` | Average O(1) lookup / insert |
| Hash set | `set` | Average O(1) membership |
| FIFO queue | `collections.deque` | Append/pop at either end O(1) |
| Priority queue | `heapq` | Push/pop O(log n) |
| Counter | `collections.Counter` | Frequency map |
| Sorting | `sorted`, `.sort()` | O(n log n) comparison sort |
| Binary search | `bisect` | Search O(log n), insertion into list O(n) |
| Stack | `list.append`, `list.pop` | Amortized O(1) at end |

Know the difference between `list.sort()` (mutates and returns `None`) and `sorted()` (returns a new list).

## 3. Language-specific risks

- **Mutable defaults:** Do not use `def f(items=[]):`.
- **Aliasing:** `b = a` does not copy a list.
- **Shallow copy:** `a.copy()` does not recursively copy nested objects.
- **Integer behavior:** Python integers have arbitrary precision, but large integers still consume memory and arithmetic is not literally constant cost for unbounded values.
- **Recursion:** Python has a recursion limit; deep DFS may need an iterative stack.
- **Sorting keys:** Prefer `sorted(records, key=lambda x: x["timestamp"])`.
- **Dictionary iteration:** Avoid changing dictionary size while iterating over it.

## 4. Example: choosing a queue correctly

```python
from collections import deque

def level_order(start):
    queue = deque([start])
    result = []

    while queue:
        node = queue.popleft()
        result.append(node.value)
        for child in node.children:
            queue.append(child)

    return result
```

Using `list.pop(0)` would shift remaining elements and can make queue processing quadratic.

## 5. Interview conversation

**Interviewer:** Why Python rather than Java?

**Candidate:** The role accepts Python, and I can express the algorithm clearly and test it quickly. I understand the runtime trade-offs, including object overhead and interpreter limitations. If the requirement were JVM integration or a production Spark component, I would choose Scala or Java based on the execution environment.

## 6. Principal-level perspective

Separate interview language choice from production language choice. In production, consider ecosystem, runtime, deployment, observability, team expertise, performance profile and operational support. Do not choose a language solely because it is concise.

## Practice

Implement the same BFS using:
1. `list` with `pop(0)`.
2. `collections.deque` with `popleft()`.

Explain why the second is the appropriate queue abstraction.
