# Find the Duplicate Number

**Difficulty:** Medium  
**Topics:** Array, Binary Search, Linked List, Floyd Cycle Detection  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 03

## 1. Problem Statement

nums has n+1 integers, each in [1,n], with one repeated value (possibly more than twice). Return it without modifying nums and using O(1) extra space.

**Example:**
```text
nums = [1, 3, 4, 2, 2]
Output: 2
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Track seen values in a set and return the first repeated value.

```python
def find_duplicate(nums):
    seen = set()
    for x in nums:
        if x in seen:
            return x
        seen.add(x)
```

- **Complexity:** Average O(n) time; O(n) space (violates constraint)

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Interpret index i as a node and nums[i] as its next pointer. Floyd's cycle algorithm finds the cycle entrance.

```python
def find_duplicate(nums):
    slow = fast = nums[0]
    while True:
        slow = nums[slow]
        fast = nums[nums[fast]]
        if slow == fast:
            break
    slow = nums[0]
    while slow != fast:
        slow = nums[slow]
        fast = nums[fast]
    return slow
```

- **Complexity:** O(n) time; O(1) auxiliary space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Interpret index i as a node and nums[i] as its next pointer. Floyd's cycle algorithm finds the cycle entrance.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Following pointers in [1,3,4,2,2] gives 0→1→3→2→4→2, so a cycle exists. Floyd's first phase finds an intersection; resetting one pointer to index 0 and moving both one step finds the cycle entrance, value 2.

## 4. Interview Conversation

**Interviewer:** A set is easy. Why not use it?

**Candidate:** It uses O(n) extra space, which violates the constraint.

**Interviewer:** How can an array become a linked list?

**Candidate:** Treat each index as a node and nums[index] as its next pointer. The value range guarantees valid indices.

**Interviewer:** Why is there a cycle?

**Candidate:** There are n+1 positions but only n possible next indices, so following the deterministic links must eventually revisit a node.

**Interviewer:** What does the cycle entrance represent?

**Candidate:** The repeated value; Floyd's second phase identifies that entrance.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Binary search on value range is O(n log n) with O(1) space. The Floyd trick depends on the exact [1,n] constraints and must not be generalized to arbitrary integers.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Why the set solution violates the constraint; why the pointer mapping is valid; how repeated values more than twice affect correctness.

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

Use this as an algorithmic reasoning exercise, not as a production duplicate-key check. Real pipelines should use explicit uniqueness and referential-integrity checks.

## 9. Final Interview Summary

**One-sentence explanation:** Interpret index i as a node and nums[i] as its next pointer. Floyd's cycle algorithm finds the cycle entrance.

**Complexity to say aloud:** O(n) time; O(1) auxiliary space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #03 in the supplied study plan.*