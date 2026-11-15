# Trapping Rain Water

**Difficulty:** Hard  
**Topics:** Array, Two Pointers, Dynamic Programming, Stack  
**Study-plan section:** Two Pointers  
**Problem ID:** 14

## 1. Problem Statement

Given non-negative bar heights, compute how much rain water can be trapped after raining.

**Example:**
```text
height = [0,1,0,2,1,0,1,3,2,1,2,1]
Output = 6
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** For each index, scan left and right to find the tallest boundary on each side. Water at i is max(0, min(leftMax,rightMax)-height[i]).

```python
def trap(height):
    total = 0
    for i, h in enumerate(height):
        left_max = max(height[:i+1])
        right_max = max(height[i:])
        total += max(0, min(left_max, right_max)-h)
    return total
```

- **Complexity:** O(n²) time; O(n) temporary slices

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Two pointers maintain left_max and right_max. Process the side with the smaller current maximum, because that side determines the trapped water.

```python
def trap(height):
    left, right = 0, len(height)-1
    left_max = right_max = 0
    water = 0
    while left < right:
        if height[left] <= height[right]:
            left_max = max(left_max, height[left])
            water += left_max - height[left]
            left += 1
        else:
            right_max = max(right_max, height[right])
            water += right_max - height[right]
            right -= 1
    return water
```

- **Complexity:** O(n) time; O(1) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Two pointers maintain left_max and right_max. Process the side with the smaller current maximum, because that side determines the trapped water.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

The left side is processed when height[left] <= height[right]. The right boundary is at least as tall as the current left bar, so left_max is sufficient to determine trapped water at left. Symmetrically for the right side. The total is 6.

## 4. Interview Conversation

**Interviewer:** What determines water at index i?

**Candidate:** The shorter of the tallest bars to its left and right, minus the bar's own height, floored at zero.

**Interviewer:** Can you avoid precomputing two arrays?

**Candidate:** Yes. Two pointers maintain the maximum boundary seen from each side and process the side with the smaller current boundary.

**Interviewer:** Why is that safe?

**Candidate:** If the left boundary is smaller, the right side is guaranteed to have a boundary at least that high, so left_max determines the water level at the left pointer.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Alternative: prefix/suffix maxima arrays use O(n) space. Monotonic stack is another O(n) approach. Discuss empty and monotonic arrays.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Do not confuse local height with maximum boundary. A bar cannot trap water unless taller boundaries exist on both sides.

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

Similar to computing bounded capacity from neighboring maxima. In streaming, right-side information is not known until later, so a one-pass online answer needs buffering or a different contract.

## 9. Final Interview Summary

**One-sentence explanation:** Two pointers maintain left_max and right_max. Process the side with the smaller current maximum, because that side determines the trapped water.

**Complexity to say aloud:** O(n) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #14 in the supplied study plan.*