# Container With Water

**Difficulty:** Medium  
**Topics:** Array, Two Pointers, Greedy  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 06

## 1. Problem Statement

Given non-negative heights, choose two lines that with the x-axis form a container holding maximum water. Return the maximum area.

**Example:**
```text
height = [1,8,6,2,5,4,8,3,7]
Output = 49
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Check every pair and calculate (right-left) × min(height[left], height[right]).

```python
def max_area(height):
    best = 0
    for i in range(len(height)):
        for j in range(i + 1, len(height)):
            best = max(best, (j-i) * min(height[i], height[j]))
    return best
```

- **Complexity:** O(n²) time; O(1) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Start at both ends; move the pointer at the shorter line because it limits the current area.

```python
def max_area(height):
    left, right, best = 0, len(height)-1, 0
    while left < right:
        best = max(best, (right-left) * min(height[left], height[right]))
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
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

**Recommended:** Start at both ends; move the pointer at the shorter line because it limits the current area.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

At indices 0 and 8, area is 8×1=8. Move the shorter left wall. At indices 1 and 8, area is 7×7=49. Continue moving the shorter boundary while retaining the best.

## 4. Interview Conversation

**Interviewer:** Why is brute force not ideal?

**Candidate:** It checks O(n²) pairs. The two-pointer method reduces this to O(n).

**Interviewer:** Why move the shorter line?

**Candidate:** The shorter line limits water height. Moving the taller line only reduces width while the limiting height cannot improve, so it cannot produce a better area for that boundary.

**Interviewer:** What if both heights are equal?

**Candidate:** Either pointer can move; I use the else branch to move the right pointer.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Contrast with Trapping Rain Water: this maximizes one container area; trapping rain water sums water at every index. Explain the greedy proof, not just the pointer rule.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Empty/single-element input returns zero; heights are non-negative per problem constraints.

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

Greedy pointer elimination is useful in sorted scans; state the invariant and prove why discarded candidates cannot improve the answer.

## 9. Final Interview Summary

**One-sentence explanation:** Start at both ends; move the pointer at the shorter line because it limits the current area.

**Complexity to say aloud:** O(n) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #06 in the supplied study plan.*