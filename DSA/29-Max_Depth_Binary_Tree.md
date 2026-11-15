# Maximum Depth of Binary Tree

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Tree, BFS/DFS, Recursion |
| **Companies** | Google, Meta, Amazon, Microsoft |
| **Interview Frequency** | Medium |

---

## 1. Problem Statement
Given the `root` of a binary tree, return its **maximum depth**. The maximum depth is the number of nodes along the longest path from the root node down to the farthest leaf node.

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

**Example:**
```
Input: root = [3,9,20,null,null,15,7]
Output: 3
Explanation: The longest path is 3 → 20 → 15 (or 3 → 20 → 7), which has 3 nodes.
```

**Example 2:**
```
Input: root = [1,null,2]
Output: 2
```

---

## 2. Approaches

### Approach 1: DFS Recursion (Optimal)
The maximum depth of a tree is 1 (the root) plus the maximum of the depths of its left and right subtrees. Base case: an empty tree has depth 0. This elegant recursive formulation handles all cases naturally.

**Pros:**
- Extremely concise (3 lines of logic) — the simplest possible solution
- Intuitive divide-and-conquer formulation that maps directly to the tree definition

**Cons:**
- Stack overflow risk on very deep or skewed trees (Python recursion limit ~1000)

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def maxDepth(root: 'TreeNode') -> int:
    if not root:
        return 0
    return 1 + max(maxDepth(root.left), maxDepth(root.right))
```

| Metric | Value |
|---|---|
| **Time** | O(n) — visits every node once |
| **Space** | O(h) — recursion stack, where h is tree height |

### Approach 2: BFS Level-by-Level (Alternative)
Use a queue to traverse the tree level by level. Each complete level increments the depth counter. After processing all nodes at the current level, move to the next.

**Pros:**
- No recursion depth issues — safe for arbitrarily deep trees
- Naturally tracks level count; easily extensible to level-order problems

**Cons:**
- More verbose than the recursive one-liner
- Queue can hold many nodes at the widest level — higher peak memory on wide trees

```python
from collections import deque

def maxDepth(root: 'TreeNode') -> int:
    if not root:
        return 0

    depth = 0
    queue = deque([root])

    while queue:
        depth += 1
        for _ in range(len(queue)):  # Process all nodes at current level
            node = queue.popleft()
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return depth
```

| Metric | Value |
|---|---|
| **Time** | O(n) — visits every node once |
| **Space** | O(w) — queue holds nodes at the widest level |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| Empty tree (`root = None`) | Return 0 |
| Single node (no children) | Return 1 |
| Skewed tree (all left or all right) | Return n (number of nodes) — behaves like a linked list |
| Perfectly balanced tree | Return log₂(n) + 1 |

---

## 4. Interview Talking Points
- "This is the canonical **tree recursion template**: base case at `None`, recursive step combining results from children."
- "DFS recursion is cleaner to write, but BFS is better when you need level information or want to avoid recursion depth issues on very deep trees."
- "DE connection: Tree depth maps directly to **pipeline depth / layer counting in DAG-based ETL**. Knowing the depth helps parallelize stages — each level can run concurrently since nodes at the same depth have no dependencies on each other."

---

## 5. Interview Follow-Up Questions
**Q1. How would you find the minimum depth?**
-> For DFS: return `1 + min(maxDepth(left), maxDepth(right))` but be careful — a `None` child is not a leaf. The minimum depth is the shortest path to an actual leaf (a node with no children).

**Q2. What if the tree is extremely deep (millions of levels)?**
-> Use BFS (iterative) to avoid recursion stack overflow. Or convert DFS to iterative with an explicit stack.

**Q3. How would you count the number of nodes at each level?**
-> Use the BFS approach and track `len(queue)` at the start of each level iteration.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Returning `max(left, right)` without `+ 1` | Misses the current node — undercounts by 1 | Always add 1 for the current level |
| Not checking `if not root` | Causes `None` attribute access | Return 0 for `None` root |
| BFS: not processing all nodes at the current level | Counts individual nodes as levels | Use `for _ in range(len(queue))` to process a full level |
| Confusing depth (nodes) with height (edges) | Depth counts nodes; height = depth - 1 | Problem asks for node count along the path |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| [Minimum Depth of Binary Tree](https://leetcode.com/problems/minimum-depth-of-binary-tree/) | Same structure, find shortest path to leaf |
| [Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree/) | Uses depth calculation at every subtree |
| [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | Same BFS level-by-level pattern |
| [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) | Uses similar recursion: max path through a node |

---

*Last updated: 2025*