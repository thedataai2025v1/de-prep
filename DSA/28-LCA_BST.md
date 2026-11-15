# Lowest Common Ancestor of a Binary Search Tree

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Tree, Binary Search Tree |
| **Companies** | Google, Meta, Amazon, Microsoft |
| **Interview Frequency** | High |

---

## 1. Problem Statement
Given a binary search tree (BST), find the **lowest common ancestor (LCA)** of two given nodes `p` and `q`. The LCA is the deepest node that is an ancestor of both `p` and `q`. A node can be a descendant of itself.

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

**Example:**
```
Input: root = [6,2,8,0,4,7,9,null,null,3,5], p = 2, q = 8
Output: 6
Explanation: The LCA of nodes 2 and 8 is node 6.
```

**Example 2:**
```
Input: root = [6,2,8,0,4,7,9,null,null,3,5], p = 2, q = 4
Output: 2
Explanation: The LCA of nodes 2 and 4 is node 2, since a node can be a descendant of itself.
```

---

## 2. Approaches

### Approach 1: BST Property (Optimal — Iterative)
Use the BST ordering property: all values in the left subtree are less than the root, and all in the right subtree are greater. Starting from the root:
- If both `p.val` and `q.val` are less than `root.val`, the LCA must be in the left subtree.
- If both are greater, the LCA must be in the right subtree.
- Otherwise, the current node is the LCA (one value is on each side, or one *is* the current node).

**Pros:**
- O(1) space — iterative, no recursion stack or extra data structures
- O(log n) on balanced trees — dramatically faster than a full tree traversal

**Cons:**
- Only works for BSTs (not general binary trees) — requires the ordering invariant

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def lowestCommonAncestor(root: 'TreeNode', p: 'TreeNode', q: 'TreeNode') -> 'TreeNode':
    curr = root
    while curr:
        if p.val < curr.val and q.val < curr.val:
            curr = curr.left
        elif p.val > curr.val and q.val > curr.val:
            curr = curr.right
        else:
            return curr
    return None  # Should never reach here if p, q exist in BST
```

| Metric | Value |
|---|---|
| **Time** | O(h) where h is tree height — O(log n) balanced, O(n) skewed |
| **Space** | O(1) — iterative, no extra space |

### Approach 2: General LCA for Any Binary Tree (Alternative)
When BST properties aren't guaranteed, search both subtrees. Each recursive call returns the node if either `p` or `q` is found. If both subtrees return non-None, the current node is the LCA.

**Pros:**
- Works on any binary tree, not just BSTs — no ordering requirements

**Cons:**
- O(n) worst-case time — visits every node instead of following a single path down
- Recursion stack can overflow on deep or skewed trees

```python
def lowestCommonAncestor(root: 'TreeNode', p: 'TreeNode', q: 'TreeNode') -> 'TreeNode':
    if not root or root == p or root == q:
        return root

    left = lowestCommonAncestor(root.left, p, q)
    right = lowestCommonAncestor(root.right, p, q)

    if left and right:
        return root    # p and q found in different subtrees
    return left if left else right  # Return whichever non-None result
```

| Metric | Value |
|---|---|
| **Time** | O(n) — visits every node in the worst case |
| **Space** | O(h) — recursion stack |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| `p` or `q` is the root | Return root — it's the LCA by definition |
| One node is ancestor of the other (e.g., `p=2, q=4`) | Return the ancestor node |
| Both nodes in the left subtree | Recurse/search left only |
| Both nodes in the right subtree | Recurse/search right only |
| `p == q` | Return `p` (a node is its own ancestor) |

---

## 4. Interview Talking Points
- "The BST approach leverages the **ordering invariant** to do a guided search instead of exploring the full tree — it's essentially a binary search for the split point."
- "The iterative BST solution uses O(1) space, which matters for deeply nested trees where recursion might overflow the stack."
- "DE connection: Finding the LCA maps to **identifying the merge point in branching data pipelines** — e.g., two downstream ETL jobs that converge from different sources share a common upstream transformation step."

---

## 5. Interview Follow-Up Questions
**Q1. What if `p` and `q` are not guaranteed to exist in the tree?**
-> First verify both exist (O(h) each in BST). Only return LCA if both are present.

**Q2. How would you find the LCA of *multiple* nodes?**
-> Sort the nodes by value. The LCA of a set in a BST is the LCA of the min and max value nodes — all others lie between them.

**Q3. What if the tree is not a BST (just a general binary tree)?**
-> Use the general approach (Approach 2). Cannot use ordering to prune the search.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Using `val == p.val or val == q.val` instead of `root == p` | Compares values, not identities — fails if duplicate values exist | Use identity check: `root == p or root == q` |
| Not handling `p == q` case | Works naturally in both approaches, but good to confirm | Both approaches handle it correctly |
| Returning `None` when `p` or `q` not found | If guaranteed to exist, `None` is unreachable | Problem statement guarantees both nodes exist in the tree |
| Confusing LCA with LCA of values | Nodes are references, not values | Compare `p.val` and `q.val` against `curr.val` in BST approach |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| [Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/) | Same problem without BST property — uses general approach |
| [Lowest Common Ancestor of a Binary Tree III](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree-iii/) | LCA with parent pointers — O(1) space possible |
| [Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) | Uses BST ordering property for guided traversal |
| [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/) | Verifies the BST property used by LCA-BST |

---

*Last updated: 2025*