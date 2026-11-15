# Union-Find / Disjoint Set Union (DSU)

**Level:** Medium | **Patterns:** Connectivity, path compression, union by rank/size

## 1. Concept and when to use it

Disjoint Set Union maintains a partition of elements into non-overlapping connected components. It supports:
- `find(x)`: identify the representative of x's component.
- `union(a, b)`: merge the components containing a and b.
- `connected(a, b)`: check whether two elements share a component.

Use DSU for undirected connectivity, incremental edge insertion, cycle detection, Kruskal's MST, and grouping equivalent entities. It is not designed for arbitrary deletions or fully dynamic connectivity.

## 2. Approaches

### Brute force

Maintain explicit component labels. To merge two components, scan and relabel all elements in one component. A sequence of operations can cost `O(n)` per merge.

### Optimized DSU

Path compression flattens trees during `find`. Union by size attaches the smaller tree beneath the larger one.

```python
class DSU:
    def __init__(self, n):
        self.parent = list(range(n))
        self.size = [1] * n
        self.components = n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, a, b):
        root_a = self.find(a)
        root_b = self.find(b)

        if root_a == root_b:
            return False

        if self.size[root_a] < self.size[root_b]:
            root_a, root_b = root_b, root_a

        self.parent[root_b] = root_a
        self.size[root_a] += self.size[root_b]
        self.components -= 1
        return True

    def connected(self, a, b):
        return self.find(a) == self.find(b)
```

## 3. Dry run

Create `DSU(5)`: components `{0},{1},{2},{3},{4}`.
- `union(0,1)` merges to `{0,1}`; count 4.
- `union(1,2)` finds root 0 and merges 2; count 3.
- `union(3,4)` merges `{3,4}`; count 2.
- `connected(0,2)` is true; `connected(0,4)` is false.
- `union(0,2)` returns false because they are already connected.

## 4. Complexity

With path compression and union by size/rank, a sequence of operations takes amortized `O(α(n))` per operation, where inverse Ackermann `α(n)` grows so slowly it is effectively constant for practical input sizes. Space is `O(n)`.

## 5. Edge cases and common mistakes

- Validate indices if inputs are untrusted.
- `union` should report whether a merge actually occurred.
- Decrement component count only when roots differ.
- Do not compare raw parent values to test connectivity; call `find`.
- DSU handles undirected connectivity, not directed reachability.
- Recursive `find` is safe with balancing; iterative variants are possible for strict recursion constraints.

## 6. Interviewer–candidate discussion

**Interviewer:** Why do we need both optimizations?

**Candidate:** Union by size limits tree height when merging. Path compression flattens paths encountered by queries. Together they provide the inverse-Ackermann amortized bound.

**Interviewer:** Can DSU handle edge deletions?

**Candidate:** Not efficiently in the standard form. Deleting an edge can split a component, and DSU has no simple inverse-union operation.

## 7. Follow-ups

- Track component sizes and number of components.
- Detect whether adding an edge creates a cycle.
- Count connected components in a graph.
- Extend to “accounts merge” or equivalent-entity grouping.
- Discuss rollback DSU for offline queries with time intervals.

## 8. Tests

```python
dsu = DSU(5)
assert dsu.components == 5
assert dsu.union(0, 1) is True
assert dsu.union(1, 2) is True
assert dsu.connected(0, 2) is True
assert dsu.connected(0, 4) is False
assert dsu.union(0, 2) is False
assert dsu.components == 3
assert dsu.size[dsu.find(1)] == 3
```

**Data-platform connection:** DSU can group records linked by identity edges, connected service dependencies, or entities sharing identifiers. Be careful: production entity resolution often needs confidence scores, provenance, and reversible decisions beyond plain connectivity.
