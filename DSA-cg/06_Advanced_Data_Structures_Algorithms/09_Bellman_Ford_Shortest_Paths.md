# Bellman–Ford Shortest Paths

**Level:** Hard | **Patterns:** Repeated edge relaxation, negative-cycle detection

## 1. Concept and when to use it

Bellman–Ford computes single-source shortest paths in a weighted directed graph and supports negative edge weights. It can also detect a negative-weight cycle reachable from the source.

Use it when negative weights are meaningful (credits, rebates, exchange-rate transformations) or when the interviewer explicitly asks for negative-cycle detection. For non-negative weights, Dijkstra is usually faster.

## 2. Approaches

### Brute force

Enumerate simple paths from the source and choose the cheapest. The number of paths can be exponential.

### Optimized: relax every edge V−1 times

A shortest simple path contains at most `V-1` edges. Repeated relaxation propagates best-known distances. One additional pass detects a reachable negative cycle.

```python
def bellman_ford(n, edges, source):
    inf = float("inf")
    dist = [inf] * n
    parent = [-1] * n
    dist[source] = 0

    for _ in range(n - 1):
        changed = False
        for u, v, weight in edges:
            if dist[u] != inf and dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                parent[v] = u
                changed = True
        if not changed:
            break

    for u, v, weight in edges:
        if dist[u] != inf and dist[u] + weight < dist[v]:
            raise ValueError("Reachable negative-weight cycle detected")

    return dist, parent
```

## 3. Dry run

Edges: `0→1 (4)`, `0→2 (5)`, `1→2 (-2)`, `2→3 (3)`.
- Initialize `[0, ∞, ∞, ∞]`.
- Relax edges: `dist[1]=4`, `dist[2]=5`, then `dist[2]=2`, then `dist[3]=5`.
- Later passes make no changes.
- Distances: `[0,4,2,5]`.

## 4. Complexity

`O(VE)` time and `O(V)` extra space. Early stopping helps on some inputs but does not improve the worst-case bound.

## 5. Edge cases and mistakes

- Only cycles reachable from the source are detected by this implementation.
- Unreachable vertices remain infinity.
- Guard against relaxing from infinity.
- A negative edge is not itself a negative cycle.
- Parent pointers can be used to reconstruct paths when no negative cycle affects the result.
- For an undirected negative edge represented in both directions, it creates a negative cycle of length two.

## 6. Interviewer–candidate discussion

**Interviewer:** Why exactly V−1 passes?

**Candidate:** Any shortest path without a negative cycle can be chosen simple, and a simple path visits at most V vertices, hence at most V−1 edges. Each pass can propagate paths with one more edge.

**Interviewer:** Does a negative cycle anywhere in the graph invalidate all results?

**Candidate:** No. This source-based version detects only a negative cycle reachable from the source. An unreachable negative cycle cannot affect source distances.

## 7. Follow-ups

- Return a concrete negative-cycle witness.
- Detect a negative cycle anywhere by initializing all distances to zero (equivalent to a super-source).
- Use queue-based SPFA and discuss its poor worst-case behavior.
- Compare with Dijkstra and Floyd–Warshall.

## 8. Tests

```python
dist, parent = bellman_ford(4, [
    (0, 1, 4), (0, 2, 5), (1, 2, -2), (2, 3, 3)
], 0)
assert dist == [0, 4, 2, 5]
assert bellman_ford(3, [(0, 1, 2)], 0)[0] == [0, 2, float("inf")]
try:
    bellman_ford(3, [(0, 1, 1), (1, 2, -2), (2, 1, -2)], 0)
    raise AssertionError("Expected negative cycle to fail")
except ValueError:
    pass
```

**Data-platform connection:** Negative costs may represent credits or adjustments in a transformation graph. For operational dependency graphs, negative cycle detection can expose contradictory constraints; ensure the graph semantics really permit negative edge weights.
