# Connected Components and Reachability

**Difficulty:** Medium  
**Topics:** Graph traversal, component counting, Union-Find

## 1. Problem framing

In an undirected graph, a connected component is a maximal set of vertices where every pair is connected by a path. Use components to group related entities, find isolated clusters, or count disconnected regions.

## 2. Approaches

### Baseline: run a search for every vertex

Without a shared visited set, repeated traversals redo work and can cost O(V(V+E)).

### Optimized: one traversal per unvisited component

```python
def count_components(n, edges):
    graph = [[] for _ in range(n)]
    for u, v in edges:
        graph[u].append(v)
        graph[v].append(u)

    visited = set()
    components = 0
    for start in range(n):
        if start in visited:
            continue
        components += 1
        stack = [start]
        visited.add(start)
        while stack:
            node = stack.pop()
            for neighbor in graph[node]:
                if neighbor not in visited:
                    visited.add(neighbor)
                    stack.append(neighbor)
    return components
```
O(V+E) time and O(V+E) space.

### Alternative: Disjoint Set Union (Union-Find)

```python
def count_components_dsu(n, edges):
    parent = list(range(n))
    size = [1] * n

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    components = n
    for a, b in edges:
        ra, rb = find(a), find(b)
        if ra != rb:
            if size[ra] < size[rb]:
                ra, rb = rb, ra
            parent[rb] = ra
            size[ra] += size[rb]
            components -= 1
    return components
```
Near O((V+E) α(V)) time with path compression and union by size; O(V) extra space.

## 3. Dry run

`n=5`, edges `[(0,1),(1,2),(3,4)]`: first traversal discovers `{0,1,2}`; second discovers `{3,4}`. Count is 2. DSU begins with 5 components and performs 3 successful unions, leaving 2.

## 4. Interviewer–candidate discussion

**Interviewer:** When would you choose DSU over BFS?

**Candidate:** DSU is convenient for incremental undirected connectivity queries and edge processing. BFS/DFS is better when I need actual component members or paths.

**Interviewer:** Does DSU handle directed reachability?

**Candidate:** Not general directed reachability. Its equivalence relation is symmetric.

## 5. Edge cases and mistakes

Zero vertices, no edges, self-loops, duplicate edges, all vertices connected, isolated vertices. Do not omit isolated vertices by deriving V only from edge endpoints if the problem supplies n separately.

## 6. Tests and follow-ups

```python
assert count_components(5, [(0, 1), (1, 2), (3, 4)]) == 2
assert count_components(3, []) == 3
assert count_components(0, []) == 0
assert count_components(4, [(0, 1), (1, 2), (2, 3)]) == 1
```
Follow-ups: return component sizes; process edges as a stream; detect whether a new edge merges two existing components.
