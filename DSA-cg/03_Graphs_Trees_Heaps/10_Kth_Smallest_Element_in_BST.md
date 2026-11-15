# Kth Smallest Element in a BST

**Difficulty:** Medium  
**Topics:** Inorder traversal, order statistics

## 1. Problem statement

Return the kth smallest value in a BST. Assume 1-indexed k and a valid k unless validation is explicitly required.

## 2. Approaches

### Baseline: collect and sort all values

```python
def kth_smallest_sort(root, k):
    values = []
    def collect(node):
        if node:
            values.append(node.val)
            collect(node.left)
            collect(node.right)
    collect(root)
    values.sort()
    return values[k - 1]
```
O(n log n) time and O(n) space.

### Optimized: inorder traversal with early stop

```python
def kth_smallest(root, k):
    stack = []
    node = root
    while stack or node:
        while node:
            stack.append(node)
            node = node.left
        node = stack.pop()
        k -= 1
        if k == 0:
            return node.val
        node = node.right
    raise ValueError("k is outside the tree size")
```
O(h+k) time in the usual analysis (more precisely O(number of visited nodes)), O(h) space.

## 3. Dry run

BST values in inorder: `[1,2,3,4,5,6]`. For k=3, visit 1 (k=2), 2 (k=1), 3 (k=0), return 3 without traversing the rest.

## 4. Interviewer–candidate discussion

**Interviewer:** What if there are many kth-smallest queries on a static tree?

**Candidate:** Augment each node with subtree size, enabling O(h) selection. Updates must maintain those sizes.

**Interviewer:** What if k is invalid?

**Candidate:** Validate `k>0` and detect exhaustion, as shown.

## 5. Edge cases and mistakes

Empty tree, k=0, k larger than node count, duplicates according to BST policy, skewed tree. Remember the problem's k may be 1-indexed, not zero-indexed.

## 6. Tests and variations

```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val, self.left, self.right = val, left, right

root = TreeNode(4, TreeNode(2, TreeNode(1), TreeNode(3)), TreeNode(6, TreeNode(5)))
assert kth_smallest(root, 1) == 1
assert kth_smallest(root, 3) == 3
assert kth_smallest(root, 6) == 6
```
Follow-ups: kth largest; maintain subtree sizes; handle a mutable BST; return the kth distinct value.
