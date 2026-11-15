# Unweighted Shortest Path and Path Reconstruction

**Difficulty:** Medium  
**Topics:** BFS, predecessor map, shortest path

## 1. Concept and when to use

In an unweighted graph, BFS discovers vertices in nondecreasing distance from the source. The first discovery of a vertex is therefore along a shortest path. Use BFS for minimum number of edges, not for minimum total weight when edges have different costs.

## 2. Approaches

### Brute force

Enumerate simple paths and choose the shortest. The number of paths can be exponential.

### Optimized BFS with parent pointers

```python
from collections import deque

def shortest_path_unweighted(graph, start, target):
    queue = deque([start])
    parent = {start: None}

    while queue:
        node = queue.popleft()
        if node == target:
            path = []
            cur = target
            while cur is not None:
                path.append(cur)
                cur = parent[cur]
            return path[::-1]

        for nei in graph[node]:
            if nei not in parent:
                parent[nei] = node
                queue.append(nei)
    return None
```
O(V+E) time and O(V) space.

## 3. Dry run

Graph `0:[1,2], 1:[0,3], 2:[0,3], 3:[1,2]`, source 0, target 3. BFS discovers 1 and 2 with parent 0. It next discovers 3 from 1, so parent[3]=1. Reconstruct backward: 3→1→0; reverse to `[0,1,3]`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why stop when target is dequeued?

**Candidate:** BFS has already discovered it at minimum edge distance. Stopping at discovery is also safe for an unweighted graph, but dequeuing makes the level-order argument straightforward.

**Interviewer:** What if edges have weights?

**Candidate:** Use Dijkstra for nonnegative weights, or Bellman–Ford when negative edges are allowed and constraints justify it.

## 5. Edge cases and mistakes

Source equals target, unreachable target, cycles, disconnected graph, directed edges. Store parent at discovery time to avoid revisiting. Do not use BFS for arbitrary weighted shortest paths.

## 6. Tests and variations

```python
g = [[1, 2], [0, 3], [0, 3], [1, 2]]
assert shortest_path_unweighted(g, 0, 3) in ([0, 1, 3], [0, 2, 3])
assert shortest_path_unweighted(g, 2, 2) == [2]
assert shortest_path_unweighted([[1], [0], []], 0, 2) is None
```
Follow-ups: multi-source BFS, shortest path in a grid, count shortest paths, reconstruct all shortest paths.
