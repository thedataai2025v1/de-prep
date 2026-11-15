# Clone Graph

**Difficulty:** Medium  
**Topics:** Graph, DFS, BFS, Hash Table  
**Study-plan section:** Graphs BFS DFS  
**Problem ID:** 26

## 1. Problem Statement

Given a reference to a node in a connected undirected graph, return a deep copy of the graph.

**Example:**
```text
Example graph adjacency: 1:[2,4], 2:[1,3], 3:[2,4], 4:[1,3]
Output: a distinct graph with the same structure.
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Traverse the graph and create copies, but without a map cycles can cause infinite recursion or duplicate nodes.

```python
def clone_graph(node):
    if node is None: return None
    copies = {}
    def dfs(cur):
        if cur in copies: return copies[cur]
        clone = Node(cur.val)
        copies[cur] = clone
        for nei in cur.neighbors:
            clone.neighbors.append(dfs(nei))
        return clone
    return dfs(node)
```

- **Complexity:** O(V+E) time; O(V) map/recursion

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use a map from original node identity to cloned node; BFS or DFS ensures each original is cloned once.

```python
from collections import deque
def clone_graph(node):
    if node is None: return None
    copies = {node: Node(node.val)}
    q = deque([node])
    while q:
        cur = q.popleft()
        for nei in cur.neighbors:
            if nei not in copies:
                copies[nei] = Node(nei.val)
                q.append(nei)
            copies[cur].neighbors.append(copies[nei])
    return copies[node]
```

- **Complexity:** O(V+E) time; O(V) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use a map from original node identity to cloned node; BFS or DFS ensures each original is cloned once.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Clone the starting node and put it in the map. For each neighbor, create a clone only if absent, then connect the current clone to the neighbor clone. The map prevents cycles from causing repeated work.

## 4. Interview Conversation

**Interviewer:** Why is a visited map necessary?

**Candidate:** Graphs can contain cycles and multiple paths to the same node. The map ensures each original node has exactly one clone.

**Interviewer:** Should the key be node value?

**Candidate:** No. Values may not be unique. I should key by original node identity/reference.

**Interviewer:** How do you prove it is a deep copy?

**Candidate:** Every cloned node is a new object, and every cloned edge points only to cloned nodes, not originals.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Disconnected graph: the prompt provides one node and normally guarantees a connected graph; otherwise accept all components as input. Self-loops and parallel edges should be preserved.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Test null node, cycle, self-loop, repeated neighbor references and duplicate node values.

Additional checks:
- Minimum valid input size.
- Maximum input size and performance.
- Duplicate values / repeated characters / repeated edges where applicable.
- Boundary values and absent target/result.

## 7. Common Interview Mistakes

- Coding before confirming constraints and output semantics.
- Explaining complexity without stating what `n`, `m`, `k` or other variables represent.
- Ignoring empty inputs, duplicates, boundary conditions or mutation requirements.
- Claiming a hash operation is worst-case O(1); it is generally expected/average O(1).
- Giving an optimized algorithm without explaining its invariant or correctness argument.

## 8. Data Engineering Connection

Graph cloning resembles copying dependency graphs and metadata lineage. Identity must be stable; labels alone may not uniquely identify vertices.

## 9. Final Interview Summary

**One-sentence explanation:** Use a map from original node identity to cloned node; BFS or DFS ensures each original is cloned once.

**Complexity to say aloud:** O(V+E) time; O(V) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #26 in the supplied study plan.*