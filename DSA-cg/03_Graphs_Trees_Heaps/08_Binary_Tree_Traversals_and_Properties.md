# Binary Tree Traversals and Core Properties

**Difficulty:** Foundation–Medium  
**Topics:** Preorder, inorder, postorder, level order, recursion

## 1. Concept and when to use

A binary tree node has up to two children. Traversal order determines when a node is processed:
- Preorder: node, left, right — copy/serialize structure.
- Inorder: left, node, right — yields sorted order for a BST.
- Postorder: left, right, node — child results before parent, useful for height and deletion.
- Level order: breadth-first by depth.

## 2. Approaches

### Recursive DFS

```python
def preorder(root):
    if root is None:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

def inorder(root):
    if root is None:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

def postorder(root):
    if root is None:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]
```
O(n) time; recursion space O(h), but list concatenation can add avoidable copying. Prefer append-based accumulator for production-scale code.

### Iterative level order

```python
from collections import deque

def level_order(root):
    if root is None:
        return []
    result, queue = [], deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result
```
O(n) time and O(w) queue space, where w is maximum tree width.

## 3. Dry run

For root 2 with left child 1 and right child 3:
- Preorder: `[2,1,3]`
- Inorder: `[1,2,3]`
- Postorder: `[1,3,2]`
- Level order: `[[2],[1,3]]`

## 4. Interviewer–candidate discussion

**Interviewer:** Why is inorder sorted for a BST?

**Candidate:** BST invariant: every left-subtree key is less than the node and every right-subtree key is greater (or follows a defined duplicate policy). Inorder visits these regions in ascending order.

**Interviewer:** What is tree height?

**Candidate:** Clarify convention. This lesson can define height as number of nodes on the longest root-to-leaf path; some problems count edges.

## 5. Edge cases and mistakes

Empty tree, one node, skewed tree, duplicate values, recursion depth. Avoid repeated list concatenation for large trees. State whether height counts nodes or edges.

## 6. Tests and variations

Use a minimal `TreeNode` class in the exercise harness:
```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val, self.left, self.right = val, left, right

root = TreeNode(2, TreeNode(1), TreeNode(3))
assert preorder(root) == [2, 1, 3]
assert inorder(root) == [1, 2, 3]
assert postorder(root) == [1, 3, 2]
assert level_order(root) == [[2], [1, 3]]
```
Follow-ups: zigzag traversal, right-side view, boundary traversal, iterative inorder.
