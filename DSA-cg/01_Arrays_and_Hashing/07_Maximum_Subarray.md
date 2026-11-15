# Maximum Subarray — Kadane's Algorithm

**Difficulty:** Medium  
**Topics:** Array, Dynamic Programming, Greedy  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 07

## 1. Problem Statement

Find the contiguous non-empty subarray with the largest sum and return that sum.

**Example:**
```text
nums = [-2,1,-3,4,-1,2,1,-5,4]
Output = 6 (subarray [4,-1,2,1])
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Enumerate every start/end pair and maintain a running sum.

```python
def max_subarray(nums):
    best = float('-inf')
    for i in range(len(nums)):
        running = 0
        for j in range(i, len(nums)):
            running += nums[j]
            best = max(best, running)
    return best
```

- **Complexity:** O(n²) time; O(1) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** At each position, either extend the best subarray ending at the previous position or start a new subarray here.

```python
def max_subarray(nums):
    current = best = nums[0]
    for x in nums[1:]:
        current = max(x, current + x)
        best = max(best, current)
    return best
```

- **Complexity:** O(n) time; O(1) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** At each position, either extend the best subarray ending at the previous position or start a new subarray here.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

The maximum subarray sum ending at each index is [-2,1,-2,4,3,5,6,1,5]. The global maximum is 6, from [4,-1,2,1].

## 4. Interview Conversation

**Interviewer:** What does current represent?

**Candidate:** It is the maximum sum of a non-empty subarray ending at the current position.

**Interviewer:** Why compare x with current+x?

**Candidate:** If the previous running sum is negative, it hurts the new subarray, so I start at x. Otherwise I extend.

**Interviewer:** What if every value is negative?

**Candidate:** Initialize current and best to nums[0], not zero, so the answer remains a valid non-empty subarray.

**Interviewer:** Can you return the actual range?

**Candidate:** Track a tentative start when restarting and save start/end whenever the global best improves.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Maximum circular subarray: max(normal Kadane, total minus minimum subarray), with an all-negative guard. Maximum product: track both max and min products because negatives swap roles.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Streaming input can be handled with O(1) state if order is defined. Empty input needs a clarified contract.

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

Kadane is an online aggregation pattern for ordered events; window boundaries and event ordering must be explicit.

## 9. Final Interview Summary

**One-sentence explanation:** At each position, either extend the best subarray ending at the previous position or start a new subarray here.

**Complexity to say aloud:** O(n) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #07 in the supplied study plan.*