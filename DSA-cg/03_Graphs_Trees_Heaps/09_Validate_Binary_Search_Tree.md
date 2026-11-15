# Validate Binary Search Tree

**Difficulty:** Medium  
**Topics:** Tree recursion, bounds, BST invariant

## 1. Problem statement

Determine whether a binary tree satisfies the BST ordering rule. This version requires strict ordering: all keys in the left subtree are less than the node; all keys in the right subtree are greater.

## 2. Approaches

### Local-child check (incorrect baseline to critique)

Checking only `node.left.val < node.val < node.right.val` is insufficient because a deep descendant can violate an ancestor's constraint.

### Optimized: propagate bounds

```python
def is_valid_bst(root):
    def validate(node, low, high):
        if node is None:
            return True
        if not (low < node.val < high):
            return False
        return (validate(node.left, low, node.val) and
                validate(node.right, node.val, high))
    return validate(root, float("-inf"), float("inf"))
```
O(n) time and O(h) recursion space.

## 3. Dry run

Tree root 5, left child 1, right child 7, with 7's left child 4. The node 4 is locally less than 7, but it lies in the root's right subtree and must be greater than 5. Bounds at node 4 are `(5,7)`, so validation fails.

## 4. Interviewer–candidate discussion

**Interviewer:** Why are bounds passed down instead of checking just parent-child?

**Candidate:** Every node must satisfy all ancestor constraints, not just its direct parent.

**Interviewer:** What if duplicates are allowed?

**Candidate:** The contract must specify where duplicates belong. I would adjust one side's strictness consistently, e.g. left < node and right >= node.

## 5. Edge cases and mistakes

Empty tree is valid; single node; extreme integers; duplicate values; deep skew. Avoid fixed numeric sentinel bounds if keys can exceed them; use `None` bounds or a safe representation for arbitrary key domains.

## 6. Tests

```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val, self.left, self.right = val, left, right

assert is_valid_bst(None)
assert is_valid_bst(TreeNode(2, TreeNode(1), TreeNode(3)))
assert not is_valid_bst(TreeNode(5, TreeNode(1), TreeNode(7, TreeNode(4))))
assert not is_valid_bst(TreeNode(2, TreeNode(2), None))
```
Follow-ups: recover swapped BST nodes; kth smallest; predecessor/successor; support duplicates with a clear policy.
