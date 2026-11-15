# Fast Track: Linked Lists, Trees and Tries

## 1. Linked lists

Know pointer manipulation:
- Reverse a list iteratively.
- Find the midpoint with slow/fast pointers.
- Detect a cycle with Floyd's algorithm.
- Merge two sorted lists.
- Explain dummy/sentinel nodes.

A linked-list traversal is O(n); random access is O(n). Pointer rewiring is often O(1) extra space.

## 2. Trees

A tree problem often reduces to traversal plus a return value from each subtree.

- DFS preorder: node, left, right.
- DFS inorder: left, node, right.
- DFS postorder: left, right, node.
- BFS: level by level using a queue.

For a binary tree with n nodes, traversal is O(n). Recursive call space is O(h), where h is tree height; worst-case skewed tree height is n.

## 3. Binary search trees

BST invariant: values in the left subtree are smaller and values in the right subtree are larger, subject to duplicate policy. Inorder traversal yields sorted order when the invariant holds.

Always clarify duplicate handling and whether the tree is balanced.

## 4. Tries

A trie stores strings by shared prefixes. Search/insert is O(L), where L is key length, independent of the number of stored words in the basic model. Memory can be high because each node may contain many child references.

Use cases: prefix search, autocomplete, dictionary matching and routing tables.

## 5. Interview conversation

**Interviewer:** Why use a trie instead of a hash set?

**Candidate:** A hash set supports exact membership efficiently, but does not naturally support prefix queries. A trie shares prefixes and supports prefix traversal in O(L), at the cost of more structural memory.

## 6. Principal-level considerations

- Recursive depth and stack limits.
- Memory layout and object overhead.
- Skewed input and balancing.
- Serialization format and versioning.
- Whether a distributed tree/trie is actually appropriate.

## Practice

Implement reverse linked list, validate BST, tree diameter, level-order traversal and trie insert/search/prefix-search.
