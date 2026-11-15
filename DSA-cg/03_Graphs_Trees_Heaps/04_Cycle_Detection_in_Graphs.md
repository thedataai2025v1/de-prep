# Cycle Detection in Directed and Undirected Graphs

**Difficulty:** Medium–Hard  
**Topics:** DFS states, parent tracking, topological reasoning

## 1. Concept and when to use

A cycle means a path can return to an earlier vertex. The correct detection rule differs for directed and undirected graphs.

## 2. Undirected graph

### Baseline

For every edge, remove it and test whether its endpoints remain connected. This is expensive: O(E(V+E)).

### Optimized DFS with parent

```python
def has_undirected_cycle(n, edges):
    graph = [[] for _ in range(n)]
    for u, v in edges:
        graph[u].append(v)
        graph[v].append(u)

    visited = set()

    def dfs(node, parent):
        visited.add(node)
        for nei in graph[node]:
            if nei not in visited:
                if dfs(nei, node):
                    return True
            elif nei != parent:
                return True
        return False

    for node in range(n):
        if node not in visited and dfs(node, -1):
            return True
    return False
```
O(V+E) time and O(V) visited/recursion space. For deep graphs, use iterative DFS or DSU.

## 3. Directed graph

Use three states: 0=unvisited, 1=currently on recursion path, 2=fully processed.

```python
def has_directed_cycle(n, edges):
    graph = [[] for _ in range(n)]
    for u, v in edges:
        graph[u].append(v)

    state = [0] * n

    def dfs(node):
        state[node] = 1
        for nei in graph[node]:
            if state[nei] == 1:
                return True
            if state[nei] == 0 and dfs(nei):
                return True
        state[node] = 2
        return False

    return any(state[node] == 0 and dfs(node) for node in range(n))
```
O(V+E) time and O(V) space.

## 4. Dry run

Directed edges `0→1, 1→2, 2→1`: DFS marks 0, then 1, then 2. From 2 it sees 1 in state 1 (active recursion path), so a cycle exists. An edge to state 2 is not necessarily a cycle.

## 5. Interviewer–candidate discussion

**Interviewer:** Why is `neighbor != parent` sufficient for undirected DFS?

**Candidate:** The reverse edge to the immediate parent is expected. A visited neighbor other than the parent indicates another route back, hence a cycle. This simple implementation assumes a simple graph; parallel edges need careful handling.

**Interviewer:** Why not use a single visited Boolean for directed graphs?

**Candidate:** It cannot distinguish a back edge to the active recursion stack from an edge to a vertex fully processed earlier.

## 6. Edge cases and mistakes

Disconnected graph, self-loop, parallel edges, long path, empty graph. Recursive depth may exceed Python's limit. The undirected parent approach needs adaptation for multigraphs because two parallel edges form a cycle.

## 7. Tests

```python
assert not has_undirected_cycle(3, [(0, 1), (1, 2)])
assert has_undirected_cycle(3, [(0, 1), (1, 2), (2, 0)])
assert has_directed_cycle(3, [(0, 1), (1, 2), (2, 1)])
assert not has_directed_cycle(3, [(0, 1), (1, 2)])
```
Follow-ups: use DSU for undirected cycle detection; return an actual cycle path; detect strongly connected components.
