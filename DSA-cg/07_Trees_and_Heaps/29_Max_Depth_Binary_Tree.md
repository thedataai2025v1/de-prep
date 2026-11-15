# Maximum Depth of Binary Tree

**Difficulty:** Easy  
**Topics:** Tree, DFS, BFS, Recursion  
**Study-plan section:** Trees and Heaps  
**Problem ID:** 29

## 1. Problem Statement

Return the maximum number of nodes along a path from the root down to the farthest leaf.

**Example:**
```text
root = [3,9,20,null,null,15,7]
Output = 3
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Recursively compute the depth of each subtree and return 1 plus the larger depth.

```python
def max_depth(root):
    if root is None: return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))
```

- **Complexity:** O(n) time; O(h) recursion space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use BFS level-order traversal and count levels.

```python
from collections import deque
def max_depth(root):
    if root is None: return 0
    q = deque([root])
    depth = 0
    while q:
        for _ in range(len(q)):
            node = q.popleft()
            if node.left: q.append(node.left)
            if node.right: q.append(node.right)
        depth += 1
    return depth
```

- **Complexity:** O(n) time; O(w) queue space, w = maximum width

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use BFS level-order traversal and count levels.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

The root is level 1. Its children form level 2; nodes 15 and 7 form level 3. The maximum depth is 3.

## 4. Interview Conversation

**Interviewer:** What is the recursive recurrence?

**Candidate:** Depth(node)=0 for null; otherwise 1+max(depth(left), depth(right)).

**Interviewer:** When would BFS be preferable?

**Candidate:** BFS naturally counts levels and avoids recursion-stack limits on a very deep tree, though it may hold a wide level in memory.

**Interviewer:** What is the space complexity?

**Candidate:** DFS recursion uses O(h); BFS uses O(w), where w is maximum tree width.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Minimum depth is different: the nearest leaf, so do not return min(left,right) when one child is null. Test empty tree, single node and skewed tree.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Recursion depth can exceed Python's recursion limit on a skewed tree. Iterative DFS/BFS is safer for unbounded depth.

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

Hierarchical depth calculations appear in dependency and organizational trees; distinguish depth from height conventions.

## 9. Final Interview Summary

**One-sentence explanation:** Use BFS level-order traversal and count levels.

**Complexity to say aloud:** O(n) time; O(w) queue space, w = maximum width

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #29 in the supplied study plan.*