# Floyd–Warshall: All-Pairs Shortest Paths

**Level:** Hard | **Patterns:** Dynamic programming over intermediate vertices

## 1. Concept and when to use it

Floyd–Warshall computes shortest-path distances between every pair of vertices. It works with negative edge weights but not with negative cycles. It is most appropriate for relatively small or dense graphs when all-pairs answers are required.

The DP invariant: after processing intermediate vertices `0..k`, `dist[i][j]` is the shortest path from `i` to `j` whose internal vertices come only from that processed set.

## 2. Approaches

### Brute force

Run a single-source shortest-path algorithm independently from every vertex. With Bellman–Ford this is `O(V²E)`; with Dijkstra it depends on edge weights and graph density.

### Optimized: Floyd–Warshall DP

```python
def floyd_warshall(n, edges):
    inf = float("inf")
    dist = [[inf] * n for _ in range(n)]

    for i in range(n):
        dist[i][i] = 0

    for u, v, weight in edges:
        dist[u][v] = min(dist[u][v], weight)

    for k in range(n):
        for i in range(n):
            if dist[i][k] == inf:
                continue
            for j in range(n):
                if dist[k][j] == inf:
                    continue
                candidate = dist[i][k] + dist[k][j]
                if candidate < dist[i][j]:
                    dist[i][j] = candidate

    if any(dist[i][i] < 0 for i in range(n)):
        raise ValueError("Negative-weight cycle detected")
    return dist
```

## 3. Dry run

Edges `0→1 (3)`, `1→2 (2)`, `0→2 (10)`.
- Initially `dist[0][2]=10`.
- When `k=1`, route `0→1→2` costs `3+2=5`.
- Update `dist[0][2]` to 5.
- All other unreachable pairs remain infinity.

## 4. Complexity

`O(V³)` time and `O(V²)` space. This is generally unsuitable for very large sparse graphs.

## 5. Edge cases and mistakes

- Initialize diagonal entries to zero.
- For multiple edges, retain the minimum weight.
- Skip infinity arithmetic for clarity and safety.
- A negative cycle is detected by a negative diagonal after the DP.
- Directed edges must not be mirrored unless the graph is undirected.
- Ensure loop order is `k`, then `i`, then `j`; the intermediate-vertex invariant depends on it.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is the intermediate vertex loop outermost?

**Candidate:** Each `k` stage extends the allowed set of intermediate vertices. The recurrence compares paths that do not use `k` with paths that do use `k`, namely `dist[i][k] + dist[k][j]`.

**Interviewer:** When would you not choose Floyd–Warshall?

**Candidate:** For a large sparse graph or only a few source queries. Repeated Dijkstra or another single-source approach is generally more scalable.

## 7. Follow-ups

- Reconstruct actual paths using a `next` matrix.
- Count shortest paths when distances tie.
- Identify pairs whose shortest paths are undefined due to a negative cycle.
- Compare all-pairs algorithms by graph density and constraints.

## 8. Tests

```python
d = floyd_warshall(3, [(0, 1, 3), (1, 2, 2), (0, 2, 10)])
assert d[0][2] == 5
assert d[2][0] == float("inf")
assert d[1][1] == 0
try:
    floyd_warshall(2, [(0, 1, -2), (1, 0, 1)])
    raise AssertionError("Expected negative cycle to fail")
except ValueError:
    pass
```

**Data-platform connection:** All-pairs reachability or cost matrices can support small service topology analysis. At scale, materializing a dense `V × V` matrix is often prohibitive; consider query-specific paths, sparse indexes, or precomputation only for bounded subsets.
