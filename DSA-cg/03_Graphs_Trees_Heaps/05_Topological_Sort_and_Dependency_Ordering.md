# Topological Sort and Dependency Ordering

**Difficulty:** Medium–Hard  
**Topics:** DAG, Kahn's algorithm, DFS postorder

## 1. Concept and when to use

A topological ordering places every vertex before its outgoing neighbors. It exists only for a directed acyclic graph (DAG). Common uses include build pipelines, task scheduling, course prerequisites and data workflow orchestration.

## 2. Approaches

### Baseline: repeatedly scan for zero-indegree nodes

Repeatedly recompute indegrees and remove available nodes; this can cost O(V²+E).

### Optimized: Kahn's algorithm

```python
from collections import deque

def topological_sort(n, edges):
    graph = [[] for _ in range(n)]
    indegree = [0] * n
    for u, v in edges:  # u must happen before v
        graph[u].append(v)
        indegree[v] += 1

    queue = deque(i for i in range(n) if indegree[i] == 0)
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for nei in graph[node]:
            indegree[nei] -= 1
            if indegree[nei] == 0:
                queue.append(nei)

    return order if len(order) == n else []
```
O(V+E) time and O(V+E) space. Empty result indicates a cycle (for n>0).

## 3. Dry run

For `n=4`, edges `0→2, 1→2, 2→3`: indegrees `[0,0,2,1]`. Queue starts `[0,1]`; remove 0 and 1, reducing indegree of 2 to zero; then remove 2 and finally 3. One valid order is `[0,1,2,3]`.

## 4. Interviewer–candidate discussion

**Interviewer:** Is the result unique?

**Candidate:** Not necessarily. If multiple zero-indegree vertices are available, different valid orders exist. A min-heap can produce the lexicographically smallest order at O((V+E) log V).

**Interviewer:** How do you know a cycle exists?

**Candidate:** If fewer than V vertices are emitted, some vertices remain blocked by a cycle.

## 5. Edge cases and mistakes

Disconnected DAG, isolated vertices, duplicate edges, self-loop, cycle. Include all n vertices even if they never appear in edges. Define edge direction clearly: prerequisite→dependent.

## 6. Tests and variations

```python
order = topological_sort(4, [(0, 2), (1, 2), (2, 3)])
assert len(order) == 4
pos = {node: i for i, node in enumerate(order)}
assert pos[0] < pos[2] and pos[1] < pos[2] and pos[2] < pos[3]
assert topological_sort(2, [(0, 1), (1, 0)]) == []
assert topological_sort(3, []) in ([0, 1, 2], [0, 2, 1], [1, 0, 2])
```
Follow-ups: return all possible orders; detect uniqueness; produce lexicographically smallest order; schedule tasks with durations and critical path.
