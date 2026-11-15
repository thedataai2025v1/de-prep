# Diameter of a Binary Tree

**Difficulty:** Medium  
**Topics:** Postorder DFS, subtree height, global optimum

## 1. Problem statement

The diameter is the length of the longest path between any two nodes. This lesson returns the number of edges. The path may or may not pass through the root.

## 2. Approaches

### Brute force

For every node, compute left and right subtree heights independently. Repeated height computation can cost O(n²).

### Optimized: compute height once per node

```python
def diameter_of_binary_tree(root):
    best = 0

    def height(node):
        nonlocal best
        if node is None:
            return 0
        left_h = height(node.left)
        right_h = height(node.right)
        best = max(best, left_h + right_h)
        return 1 + max(left_h, right_h)

    height(root)
    return best
```
O(n) time and O(h) recursion space.

## 3. Dry run

For root 1 with left chain 2→4 and right child 3:
- At node 2, left height 1 and right 0; candidate diameter 1.
- At root 1, left height 2 and right height 1; candidate 3.
- Return diameter 3 edges.

## 4. Interviewer–candidate discussion

**Interviewer:** Why does the longest path through a node equal left height plus right height?

**Candidate:** The path descends from that node into its left subtree and into its right subtree; heights count edges differently depending on convention. Here height returns node count, so the sum gives edge count across both branches.

**Interviewer:** Why use a global best?

**Candidate:** A subtree's height is returned to its parent, while the best diameter can occur entirely inside a child subtree.

## 5. Edge cases and mistakes

Empty tree and single node have diameter 0 edges. Do not confuse diameter with height. Clarify whether answer counts nodes or edges.

## 6. Tests

```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val, self.left, self.right = val, left, right

assert diameter_of_binary_tree(None) == 0
assert diameter_of_binary_tree(TreeNode(1)) == 0
root = TreeNode(1, TreeNode(2, TreeNode(4)), TreeNode(3))
assert diameter_of_binary_tree(root) == 3
```
Follow-ups: return the actual path; compute diameter in a generic N-ary tree; avoid recursion depth issues.
