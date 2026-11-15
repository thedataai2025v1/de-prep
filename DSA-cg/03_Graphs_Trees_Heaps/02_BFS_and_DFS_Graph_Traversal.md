# BFS and DFS — Graph Traversal

**Difficulty:** Medium  
**Topics:** Breadth-first search, depth-first search, visited set

## 1. Concept and when to use

BFS explores in layers and finds shortest path lengths in an unweighted graph. DFS explores one path deeply before backtracking; it is useful for reachability, components, cycle detection and structural analysis.

## 2. Approaches

### Repeated reachability checks (baseline)

For each query, scan the graph from scratch. This may cost O(V+E) per query and repeats work.

### Optimized traversal

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    order = []
    queue = deque([start])
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)       # mark when enqueued
                queue.append(neighbor)
    return order

def dfs_iterative(graph, start):
    visited = set()
    order = []
    stack = [start]
    while stack:
        node = stack.pop()
        if node in visited:
            continue
        visited.add(node)
        order.append(node)
        for neighbor in reversed(graph[node]):
            if neighbor not in visited:
                stack.append(neighbor)
    return order
```
Both are O(V+E) time and O(V) auxiliary space for the reachable subgraph.

## 3. Dry run

Graph: `0:[1,2], 1:[0,3], 2:[0,3], 3:[1,2]`.
- BFS from 0: visit 0; enqueue 1,2; visit 1 and enqueue 3; visit 2; visit 3. Order `[0,1,2,3]`.
- DFS order depends on neighbor ordering. With the implementation above it visits one branch before returning.

## 4. Interviewer–candidate discussion

**Interviewer:** Why mark visited when enqueuing in BFS?

**Candidate:** It prevents the same node entering the queue multiple times through different parents, keeping queue growth bounded by O(V).

**Interviewer:** Recursive DFS or iterative?

**Candidate:** Recursive code is concise, but Python's recursion limit can fail on deep graphs. Iterative DFS is safer for large or adversarial depth.

## 5. Edge cases and mistakes

Disconnected graph, isolated vertex, cycle, self-loop, duplicate edges, invalid start. For a full-graph traversal, loop over all vertices and start a traversal from each unvisited vertex. Use `deque`, not `list.pop(0)`.

## 6. Tests and variations

```python
g = [[1, 2], [0, 3], [0, 3], [1, 2]]
assert bfs(g, 0) == [0, 1, 2, 3]
assert set(dfs_iterative(g, 0)) == {0, 1, 2, 3}
assert bfs([[1], [0], []], 2) == [2]
```
Follow-ups: shortest path reconstruction, multi-source BFS, level-order processing, DFS timestamps.
