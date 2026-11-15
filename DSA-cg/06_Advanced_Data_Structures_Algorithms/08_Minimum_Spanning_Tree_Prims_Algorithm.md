# Minimum Spanning Tree — Prim's Algorithm

**Level:** Medium–Hard | **Patterns:** Greedy frontier, min-heap

## 1. Concept and when to use it

Prim's algorithm grows one tree: start at any vertex and repeatedly add the minimum-weight edge connecting the current tree to an outside vertex. It solves the same undirected MST problem as Kruskal.

Prim is often natural with adjacency lists and a priority queue; Kruskal is often convenient when the edge list is already available.

## 2. Approaches

### Brute force

Repeatedly scan every edge to find the cheapest edge crossing from the current tree to an unvisited vertex. This can cost `O(VE)`.

### Optimized: adjacency list + heap

```python
import heapq

def prim_mst(n, edges):
    if n == 0:
        return 0, []

    graph = [[] for _ in range(n)]
    for weight, u, v in edges:
        graph[u].append((weight, v))
        graph[v].append((weight, u))

    visited = [False] * n
    heap = [(0, 0, -1)]  # weight, vertex, parent
    total = 0
    chosen = []

    while heap and len(chosen) < n - 1:
        weight, node, parent = heapq.heappop(heap)
        if visited[node]:
            continue

        visited[node] = True
        total += weight
        if parent != -1:
            chosen.append((parent, node, weight))

        for next_weight, neighbor in graph[node]:
            if not visited[neighbor]:
                heapq.heappush(heap, (next_weight, neighbor, node))

    if len(chosen) != n - 1:
        raise ValueError("Graph is disconnected; no spanning tree exists")
    return total, chosen
```

## 3. Dry run

Using edges `(1,0,1)`, `(4,0,2)`, `(2,1,2)`, `(3,1,3)`, `(5,2,3)`:
- Start at 0; frontier has 0–1 (1) and 0–2 (4).
- Choose 0–1 (1); add 1–2 (2), 1–3 (3).
- Choose 1–2 (2).
- Choose 1–3 (3).
- Total = 6.

## 4. Complexity

With an adjacency list and binary heap: typically `O(E log E)` (often expressed as `O(E log V)` for simple graphs) and `O(V + E)` space. The lazy heap may contain multiple candidate edges.

## 5. Edge cases and mistakes

- Disconnected graphs: this implementation reports no spanning tree.
- Single vertex: total weight zero.
- Negative weights are fine.
- Ignore stale heap entries whose vertex is already visited.
- Add each undirected edge in both adjacency directions.
- Do not confuse Prim with Dijkstra: Prim minimizes total tree weight, not shortest-path distance from the source.

## 6. Interviewer–candidate discussion

**Interviewer:** Why can the heap contain duplicate vertices?

**Candidate:** Several tree vertices may offer different edges to the same outside vertex. The heap stores candidate crossing edges. Once the vertex is added, later entries are stale and skipped.

**Interviewer:** How is this different from Dijkstra?

**Candidate:** Dijkstra's priority is the best known path distance from the source. Prim's priority is the weight of a single edge crossing the current tree boundary.

## 7. Follow-ups

- Return a minimum spanning forest for disconnected graphs.
- Implement an indexed heap to support decrease-key.
- Compare Prim and Kruskal on sparse and dense graphs.
- Explain tie handling and deterministic output.

## 8. Tests

```python
edges = [(1, 0, 1), (4, 0, 2), (2, 1, 2), (3, 1, 3), (5, 2, 3)]
weight, chosen = prim_mst(4, edges)
assert weight == 6
assert len(chosen) == 3
assert prim_mst(1, []) == (0, [])
try:
    prim_mst(3, [(1, 0, 1)])
    raise AssertionError("Expected disconnected graph to fail")
except ValueError:
    pass
```

**Data-platform connection:** Prim is useful when a graph is naturally explored from a seed node, such as designing a low-cost connected infrastructure topology. For service architecture, resilience and failure domains may matter more than pure cost.
