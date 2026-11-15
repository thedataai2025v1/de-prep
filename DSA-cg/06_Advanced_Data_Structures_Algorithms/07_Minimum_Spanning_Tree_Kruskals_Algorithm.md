# Minimum Spanning Tree — Kruskal's Algorithm

**Level:** Medium–Hard | **Patterns:** Greedy + DSU, edge sorting

## 1. Concept and when to use it

A minimum spanning tree (MST) connects every vertex in a connected, undirected, weighted graph with minimum total edge weight and no cycles. Kruskal's algorithm considers edges from cheapest to most expensive and accepts an edge only if it joins two different components.

Use it for network design, minimizing total cable/connection cost, and clustering thresholds. MST assumes an undirected graph; negative edge weights are allowed.

## 2. Approaches

### Brute force

Enumerate subsets of `V-1` edges, test whether each is a spanning tree, and find the cheapest. This is exponential and useful only as a tiny-graph correctness oracle.

### Optimized: sort edges + DSU

```python
def kruskal_mst(n, edges):
    # edges are (weight, u, v)
    parent = list(range(n))
    size = [1] * n

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    def union(a, b):
        ra, rb = find(a), find(b)
        if ra == rb:
            return False
        if size[ra] < size[rb]:
            ra, rb = rb, ra
        parent[rb] = ra
        size[ra] += size[rb]
        return True

    chosen = []
    total_weight = 0

    for weight, u, v in sorted(edges):
        if union(u, v):
            chosen.append((u, v, weight))
            total_weight += weight
            if len(chosen) == n - 1:
                break

    if n == 0:
        return 0, []
    if len(chosen) != n - 1:
        raise ValueError("Graph is disconnected; no spanning tree exists")
    return total_weight, chosen
```

## 3. Dry run

Vertices `0..3`, edges `(1,0,1)`, `(4,0,2)`, `(2,1,2)`, `(3,1,3)`, `(5,2,3)`.
- Accept weight 1: connect 0–1.
- Accept weight 2: connect 1–2.
- Weight 3: connect 1–3.
- Three edges connect four vertices; stop.
- Total MST weight = 6.

## 4. Complexity

Sorting edges costs `O(E log E)`. DSU operations over all edges cost `O(E α(V))`. Total `O(E log E)` time; DSU and output use `O(V)` space, with `O(E)` for sorted edge storage depending on implementation.

## 5. Edge cases and mistakes

- Disconnected graph has no spanning tree; it has a minimum spanning forest.
- `n = 1` has an MST with zero edges and weight zero.
- Negative weights are valid.
- Parallel edges are valid; self-loops are always rejected by DSU.
- Do not apply Kruskal to directed graphs.
- Stop after `n-1` accepted edges, not after inspecting `n-1` edges.

## 6. Interviewer–candidate discussion

**Interviewer:** Why does choosing the cheapest safe edge work?

**Candidate:** By the cut property, a minimum-weight edge crossing a cut is safe for some MST. At each step, the current DSU components define cuts; selecting the cheapest edge connecting two components preserves the possibility of an MST.

**Interviewer:** What if multiple MSTs exist?

**Candidate:** The total weight remains minimum, but the chosen edge set may differ. Stable tie-breaking can make output deterministic.

## 7. Follow-ups

- Return a minimum spanning forest instead of raising on disconnected input.
- Find the second-best MST.
- Use MST to form clusters by stopping before accepting edges above a threshold.
- Compare with Prim's algorithm for dense graphs.

## 8. Tests

```python
weight, chosen = kruskal_mst(4, [
    (1, 0, 1), (4, 0, 2), (2, 1, 2), (3, 1, 3), (5, 2, 3)
])
assert weight == 6
assert len(chosen) == 3
assert kruskal_mst(1, []) == (0, [])
try:
    kruskal_mst(3, [(1, 0, 1)])
    raise AssertionError("Expected disconnected graph to fail")
except ValueError:
    pass
```

**Data-platform connection:** A network topology or low-cost connectivity design can be modeled as MST. Real network design may add capacity, resilience, geographic constraints, or redundancy, which changes the optimization problem.
