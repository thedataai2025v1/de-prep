# Dijkstra's Algorithm — Weighted Shortest Path

**Difficulty:** Medium–Hard  
**Topics:** Greedy algorithm, min-heap, nonnegative edge weights

## 1. Concept and when to use

Dijkstra finds shortest distances from one source when all edge weights are nonnegative. It is widely used in routing and network optimization. It is not correct with negative edge weights.

## 2. Approaches

### Baseline: Bellman–Ford style relaxation

Relax all edges V−1 times: O(VE) time. This supports negative edges (but needs negative-cycle detection).

### Optimized: adjacency list + min-heap

```python
import heapq
from math import inf

def dijkstra(n, edges, source):
    graph = [[] for _ in range(n)]
    for u, v, weight in edges:
        if weight < 0:
            raise ValueError("Dijkstra requires non-negative weights")
        graph[u].append((v, weight))

    dist = [inf] * n
    dist[source] = 0
    heap = [(0, source)]

    while heap:
        current_dist, node = heapq.heappop(heap)
        if current_dist != dist[node]:
            continue  # stale heap entry
        for nei, weight in graph[node]:
            candidate = current_dist + weight
            if candidate < dist[nei]:
                dist[nei] = candidate
                heapq.heappush(heap, (candidate, nei))
    return dist
```
With a binary heap: O((V+E) log V) in common sparse-graph analysis; O(V+E) space.

## 3. Dry run

Edges `0→1 (4), 0→2 (1), 2→1 (2), 1→3 (1), 2→3 (5)`.
- Start dist `[0,∞,∞,∞]`.
- From 0: dist1=4, dist2=1.
- Pop 2: improve dist1=3 and dist3=6.
- Pop 1 at distance 3: improve dist3=4.
- Final distances `[0,3,1,4]`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why can we finalize the smallest popped distance?

**Candidate:** With nonnegative edge weights, no later path through an unprocessed node can reduce that distance below the current minimum.

**Interviewer:** Why stale-entry check?

**Candidate:** Python's heap does not support decrease-key directly. We push improved candidates and skip outdated entries when popped.

## 5. Edge cases and mistakes

Disconnected nodes remain infinity; zero-weight edges are valid; parallel edges; directed versus undirected graph; negative weights must be rejected. Do not mark a node permanently visited when first inserted into the heap.

## 6. Tests and follow-ups

```python
from math import inf
assert dijkstra(4, [(0,1,4),(0,2,1),(2,1,2),(1,3,1),(2,3,5)], 0) == [0,3,1,4]
assert dijkstra(3, [(0,1,2)], 0) == [0,2,inf]
assert dijkstra(2, [(0,1,0)], 0) == [0,0]
```
Follow-ups: reconstruct paths; multi-source Dijkstra; Bellman–Ford; Floyd–Warshall; A*; discuss latency and graph partitioning at scale.
