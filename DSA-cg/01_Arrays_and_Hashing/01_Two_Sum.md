# Two Sum

**Difficulty:** Easy  
**Topics:** Array, Hash Table  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 01

## 1. Problem Statement

Given an integer array nums and an integer target, return the indices of two distinct elements whose values add to target. Assume exactly one solution exists.

**Example:**
```text
nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Try every pair of indices i < j and compare their sum with target.

```python
def two_sum(nums, target):
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] + nums[j] == target:
                return [i, j]
    return []
```

- **Complexity:** O(n²) time; O(1) auxiliary space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Store each previously seen value and its index. For current x, look up target - x before inserting x.

```python
def two_sum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        need = target - x
        if need in seen:
            return [seen[need], i]
        seen[x] = i
    return []
```

- **Complexity:** Average O(n) time; O(n) auxiliary space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Store each previously seen value and its index. For current x, look up target - x before inserting x.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

At index 0, x=2 and need=7, which is not present; store 2→0. At index 1, x=7 and need=2, so return [0,1]. Lookup before insertion prevents reusing the same element.

## 4. Interview Conversation

**Interviewer:** Start with a baseline solution.

**Candidate:** I can test every pair, which is O(n²) time and O(1) extra space.

**Interviewer:** Can you improve it?

**Candidate:** A hash map stores values already visited. For each x, I check whether target-x exists, giving expected O(n) time.

**Interviewer:** What about [3,3], target 6?

**Candidate:** The first 3 is stored at index 0. The second finds it and returns [0,1]. I check before inserting the current item.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Sorted input: use two pointers, but preserve original indices if required. Multiple solutions: clarify whether any pair or all pairs are required. Distributed data: global complement matching may require a shuffle or partitioning.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Anagrams require same character frequencies. Grouping anagrams uses a canonical key. Unicode requires normalization rules.

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

Two Sum is a common matching/reconciliation pattern. A global key-based join in Spark may shuffle data; assess whether one side can be broadcast.

## 9. Final Interview Summary

**One-sentence explanation:** Store each previously seen value and its index. For current x, look up target - x before inserting x.

**Complexity to say aloud:** Average O(n) time; O(n) auxiliary space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #01 in the supplied study plan.*