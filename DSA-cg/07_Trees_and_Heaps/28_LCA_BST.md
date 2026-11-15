# Lowest Common Ancestor of a BST

**Difficulty:** Medium  
**Topics:** Tree, Binary Search Tree  
**Study-plan section:** Trees and Heaps  
**Problem ID:** 28

## 1. Problem Statement

Given a BST and two nodes p and q, return their lowest common ancestor: the deepest node that has both p and q as descendants (a node can be its own descendant).

**Example:**
```text
BST: [6,2,8,0,4,7,9,null,null,3,5], p=2, q=8
Output: node 6
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Search from the root using the BST property. A node whose value lies between p and q is the split point.

```python
def lowest_common_ancestor(root, p, q):
    low, high = sorted((p.val, q.val))
    cur = root
    while cur:
        if cur.val < low:
            cur = cur.right
        elif cur.val > high:
            cur = cur.left
        else:
            return cur
    return None
```

- **Complexity:** O(h) time; O(1) iterative space, h = tree height

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** The BST property allows discarding one side at every step; no need to traverse the full tree.

```python
def lowest_common_ancestor(root, p, q):
    cur = root
    while cur:
        if p.val < cur.val and q.val < cur.val:
            cur = cur.left
        elif p.val > cur.val and q.val > cur.val:
            cur = cur.right
        else:
            return cur
    return None
```

- **Complexity:** O(h) time; O(1) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** The BST property allows discarding one side at every step; no need to traverse the full tree.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

At root 6, p=2 is smaller and q=8 is larger. They split across left and right subtrees, so 6 is the LCA.

## 4. Interview Conversation

**Interviewer:** What property makes this easier than a general binary tree?

**Candidate:** The BST ordering. If both target values are smaller, move left; if both are larger, move right.

**Interviewer:** When have you found the LCA?

**Candidate:** When the current node lies between the two target values, or equals one of them.

**Interviewer:** What is the complexity?

**Candidate:** O(h), where h is tree height. A balanced BST gives O(log n); a skewed tree can be O(n).

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

General binary tree: use recursive postorder search, O(n). If nodes have parent pointers, compare ancestor paths. Clarify whether p and q are guaranteed to exist.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

BST duplicates require a defined ordering convention. Iterative solution uses O(1) extra space.

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

Tree ancestry is relevant to hierarchical metadata and organizational structures; BST ordering is a special property, not present in arbitrary hierarchies.

## 9. Final Interview Summary

**One-sentence explanation:** The BST property allows discarding one side at every step; no need to traverse the full tree.

**Complexity to say aloud:** O(h) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #28 in the supplied study plan.*