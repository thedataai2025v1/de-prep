# Balanced Binary Tree

**Difficulty:** Easy–Medium  
**Topics:** Height computation, postorder, early termination

## 1. Problem statement

A binary tree is height-balanced if, for every node, the heights of left and right subtrees differ by at most one.

## 2. Approaches

### Brute force

At every node, compute both subtree heights and recursively validate children. O(n²) worst case for a skewed tree.

### Optimized: return height or failure sentinel

```python
def is_balanced(root):
    def check(node):
        if node is None:
            return 0
        left_h = check(node.left)
        if left_h == -1:
            return -1
        right_h = check(node.right)
        if right_h == -1 or abs(left_h - right_h) > 1:
            return -1
        return 1 + max(left_h, right_h)

    return check(root) != -1
```
O(n) time and O(h) recursion space.

## 3. Dry run

For root 1 with left subtree chain 2→3 and right child 4:
- Node 3 returns height 1.
- Node 2 sees heights 1 and 0, returns 2.
- Node 4 returns 1.
- Root sees difference 1 and is balanced.

If the left side were one level deeper, root would detect a difference greater than one and return -1.

## 4. Interviewer–candidate discussion

**Interviewer:** Why return -1?

**Candidate:** It encodes failure and lets the recursion stop doing unnecessary work once an unbalanced subtree is found.

**Interviewer:** Is a complete tree necessarily balanced?

**Candidate:** Yes, but balance is a weaker condition; a balanced tree need not be complete.

## 5. Edge cases and mistakes

Empty tree is balanced; single node; long skew; imbalance deep below root. Avoid recalculating subtree heights at every node.

## 6. Tests

```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val, self.left, self.right = val, left, right

assert is_balanced(None)
assert is_balanced(TreeNode(1, TreeNode(2), TreeNode(3)))
assert not is_balanced(TreeNode(1, TreeNode(2, TreeNode(3)), None))
```
Follow-ups: AVL rotations; red-black tree guarantees; maintain balance under insert/delete.
