# Contains Duplicate

**Difficulty:** Easy  
**Topics:** Array, Hash Set (not a canonical sliding-window problem)  
**Study-plan section:** Sliding Window  
**Problem ID:** 18

## 1. Problem Statement

Given an integer array nums, return true if any value appears at least twice; otherwise return false.

**Example:**
```text
nums = [1,2,3,1]
Output = True
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Compare every pair of elements.

```python
def contains_duplicate(nums):
    for i in range(len(nums)):
        for j in range(i+1, len(nums)):
            if nums[i] == nums[j]:
                return True
    return False
```

- **Complexity:** O(n²) time; O(1) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Compare the length of the array with the size of a set of its values, or scan while maintaining a seen set.

```python
def contains_duplicate(nums):
    seen = set()
    for x in nums:
        if x in seen:
            return True
        seen.add(x)
    return False
```

- **Complexity:** O(n) average time; O(n) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Compare the length of the array with the size of a set of its values, or scan while maintaining a seen set.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Scan left to right. Add each unseen value to the set. When 1 appears for the second time, it is already present, so return true immediately.

## 4. Interview Conversation

**Interviewer:** What is the straightforward solution?

**Candidate:** Compare all pairs, O(n²).

**Interviewer:** Can you do better?

**Candidate:** A set tracks values seen so far. A repeated value is detected in expected O(1) lookup time.

**Interviewer:** Could you use sorting?

**Candidate:** Yes, sort and compare adjacent values in O(n log n), with sorting space depending on implementation.

**Interviewer:** Is this really a sliding-window problem?

**Candidate:** Not in its standard form. It is a hash-set problem; it may be grouped with nearby array patterns in this study plan.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Return false for empty and single-element inputs. For a bounded recent window, maintain a set plus queue/deque and evict values outside the window.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Set membership is average O(1), not a strict worst-case guarantee. Sorting may mutate the input.

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

Deduplication is a core data-quality check. In distributed data, global duplicate detection requires key partitioning or aggregation, not a process-local set.

## 9. Final Interview Summary

**One-sentence explanation:** Compare the length of the array with the size of a set of its values, or scan while maintaining a seen set.

**Complexity to say aloud:** O(n) average time; O(n) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #18 in the supplied study plan.*