# Graph Fundamentals and Representations

**Difficulty:** Foundation–Medium  
**Topics:** Vertices, edges, directed/undirected graphs, adjacency list/matrix

## 1. Concept and when to use

A graph models entities and relationships: services and dependencies, users and friendships, cities and roads, jobs and prerequisites. A graph is \(G=(V,E)\), where V is the vertex set and E is the edge set.

Clarify:
- Directed or undirected?
- Weighted or unweighted?
- Cyclic or guaranteed acyclic?
- Sparse or dense?
- Are vertices identified by integers, strings, or objects?
- Can the graph be disconnected?

## 2. Representations

### Edge-list baseline

```python
edges = [(0, 1), (0, 2), (1, 3)]
```
Simple to store and iterate over all edges: O(V+E) space. Finding all neighbors of a vertex costs O(E).

### Adjacency list

```python
def build_graph(n, edges, directed=False):
    graph = [[] for _ in range(n)]
    for u, v in edges:
        graph[u].append(v)
        if not directed:
            graph[v].append(u)
    return graph
```
O(V+E) space. Iterating neighbors is O(degree(v)); ideal for sparse graphs.

### Adjacency matrix

```python
def build_matrix(n, edges, directed=False):
    matrix = [[0] * n for _ in range(n)]
    for u, v in edges:
        matrix[u][v] = 1
        if not directed:
            matrix[v][u] = 1
    return matrix
```
O(V²) space, O(1) edge lookup. Useful for dense graphs or constant-time edge queries.

## 3. Dry run

For undirected edges `(0,1), (0,2), (1,3)`, adjacency list becomes:
- 0: [1,2]
- 1: [0,3]
- 2: [0]
- 3: [1]

Each undirected edge appears twice in the adjacency list.

## 4. Complexity and trade-offs

| Operation | Adjacency list | Adjacency matrix |
|---|---:|---:|
| Space | O(V+E) | O(V²) |
| Check edge | O(degree) | O(1) |
| Iterate neighbors | O(degree) | O(V) |
| Add edge | O(1) amortized | O(1) |

## 5. Interviewer–candidate discussion

**Interviewer:** Which representation would you use for a dependency graph with millions of services but relatively few edges?

**Candidate:** An adjacency list. A matrix would waste O(V²) memory.

**Interviewer:** What if edges are weighted?

**Candidate:** Store `(neighbor, weight)` pairs in the adjacency list or weights in the matrix.

## 6. Edge cases and common mistakes

- Self-loops and parallel edges.
- Duplicate edges.
- Invalid vertex IDs.
- Accidentally adding reverse edges to a directed graph.
- Forgetting isolated vertices when building from an edge list.
- Treating edge count in an undirected adjacency list as E rather than 2E entries.

## 7. Exercises and tests

```python
g = build_graph(4, [(0, 1), (0, 2), (1, 3)])
assert sorted(g[0]) == [1, 2]
assert 0 in g[1] and 3 in g[1]
assert g[2] == [0]
assert build_graph(2, [(0, 1)], directed=True) == [[1], []]
```

**Principal-level connection:** Dependency graphs underpin orchestration systems, build systems, workflow engines and service topology analysis.
