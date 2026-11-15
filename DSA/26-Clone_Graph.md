# Clone Graph

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Graph, BFS/DFS, Hash Table |
| **Companies** | Meta, Google, Amazon, Netflix |
| **Interview Frequency** | High |

---

## 1. Problem Statement
Given a reference (pointer) to a node in a connected undirected graph, return a **deep copy** (clone) of the graph. Each node contains a `val` (integer) and a list of its `neighbors` (list of Node references).

```python
class Node:
    def __init__(self, val = 0, neighbors = None):
        self.val = val
        self.neighbors = neighbors if neighbors is not None else []
```

**Example:**
```
Input: adjList = [[2,4],[1,3],[2,4],[1,3]]
Output: A graph where each node is a clone of the original.
Explanation:
- Node 1 connected to nodes 2 and 4
- Node 2 connected to nodes 1 and 3
- Node 3 connected to nodes 2 and 4
- Node 4 connected to nodes 1 and 3
```

**Example 2:**
```
Input: adjList = [[]]
Output: A single isolated node [[[]]]
```

---

## 2. Approaches

### Approach 1: BFS with Hash Map (Optimal)
Use a hash map to track the mapping from original nodes to their clones. Start BFS from the given node: create a clone of the current node, then for each neighbor, clone it if not already cloned, and add the clone to the current clone's neighbor list.

**Pros:**
- Iterative — no risk of stack overflow on large or deep graphs
- Natural left-to-right traversal order mirrors how graphs are typically visualized

**Cons:**
- Queue can grow large for very wide graphs (many neighbors per node)

```python
class Node:
    def __init__(self, val = 0, neighbors = None):
        self.val = val
        self.neighbors = neighbors if neighbors is not None else []

from collections import deque

def cloneGraph(node: 'Node') -> 'Node':
    if not node:
        return None

    old_to_new = {}  # Original node -> Cloned node
    old_to_new[node] = Node(node.val)
    queue = deque([node])

    while queue:
        curr = queue.popleft()
        for neighbor in curr.neighbors:
            if neighbor not in old_to_new:
                old_to_new[neighbor] = Node(neighbor.val)
                queue.append(neighbor)
            old_to_new[curr].neighbors.append(old_to_new[neighbor])

    return old_to_new[node]
```

| Metric | Value |
|---|---|
| **Time** | O(V + E) — each node and edge visited once |
| **Space** | O(V) for the hash map + O(V) for the queue |

### Approach 2: DFS Recursive (Alternative)
Same concept using depth-first traversal. Use a hash map to track cloned nodes. For each node, create a clone, then recursively clone each neighbor, adding clones to the neighbor list.

**Pros:**
- Clean recursive structure — very compact code
- Processes each node and wires up all its neighbors immediately in one call

**Cons:**
- Risk of stack overflow on deep or narrow graphs (Python recursion limit ~1000)

```python
def cloneGraph(node: 'Node') -> 'Node':
    if not node:
        return None

    old_to_new = {}

    def dfs(curr: 'Node') -> 'Node':
        if curr in old_to_new:
            return old_to_new[curr]

        clone = Node(curr.val)
        old_to_new[curr] = clone
        for neighbor in curr.neighbors:
            clone.neighbors.append(dfs(neighbor))

        return clone

    return dfs(node)
```

| Metric | Value |
|---|---|
| **Time** | O(V + E) |
| **Space** | O(V) for the hash map + O(V) for recursion stack |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| `None` input | Return `None` |
| Single node with no neighbors (`[[]]`) | Return a single isolated clone |
| Graph with cycles | Hash map prevents infinite recursion |
| Self-loop (node points to itself) | Hash map returns existing clone |
| Disconnected components (via different entry points) | Only the connected component from the given node is cloned |

---

## 4. Interview Talking Points
- "The key insight is using a **hash map to track old→new node references**. Without it, we'd either clone nodes multiple times or follow cycles infinitely."
- "This is the classic **graph cloning pattern**: visit each node once, create its clone, and wire up cloned neighbors. Works for any graph traversal order."
- "DE connection: This pattern applies to **cloning data models with references** — e.g., copying graph-structured data in social networks, deep-copying dependency graphs in build systems, or cloning object graphs in serialization."

---

## 5. Interview Follow-Up Questions
**Q1. What if the graph is a directed graph?**
-> The algorithm works the same way — we just follow outgoing edges (neighbors) and clone in traversal order.

**Q2. How would you handle a very large graph that might cause recursion depth issues?**
-> Use BFS (iterative) instead of DFS to avoid stack overflow. The hash map approach is identical.

**Q3. What if nodes have additional properties (edges have weights, nodes have metadata)?**
-> Copy all properties when creating the clone: `clone = Node(val=curr.val, weight=curr.weight, metadata=copy.deepcopy(curr.metadata))`.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Not checking if node is already cloned | Creates duplicate nodes, breaks graph structure | Check `if neighbor in old_to_new` before cloning |
| Cloning the root twice | Hash map lookup prevents it, but forgetting to add root to map first causes issues | Add root to `old_to_new` before BFS/DFS starts |
| Not handling `None` input | Causes AttributeError | Return `None` immediately if input is `None` |
| Modifying original graph | Side effects on caller's data | Only create new `Node` objects; never mutate the original |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| [Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer/) | Same clone-with-references pattern on a linked list |
| [Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/) | Similar traversal + reconstruction pattern |
| [Find the Town Judge](https://leetcode.com/problems/find-the-town-judge/) | Graph traversal to find specific node properties |

---

*Last updated: 2025*