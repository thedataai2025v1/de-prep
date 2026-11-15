# Container With Water — Two-Pointer Focus

**Difficulty:** Medium  
**Topics:** Array, Two Pointers, Greedy  
**Study-plan section:** Two Pointers  
**Problem ID:** 13

## 1. Problem Statement

Given non-negative heights, choose two lines that form a container with maximum area. This is the same canonical problem as entry 06, revisited here to emphasize the two-pointer proof.

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

**Idea:** Check all pairs; this baseline is O(n²).

```python
def max_area(height):
    best = 0
    for i in range(len(height)):
        for j in range(i+1, len(height)):
            best = max(best, (j-i)*min(height[i],height[j]))
    return best
```

- **Complexity:** O(n²) time; O(1) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use two pointers and discard the shorter boundary after evaluating the current area.

```python
def max_area(height):
    left, right, best = 0, len(height)-1, 0
    while left < right:
        best = max(best, (right-left)*min(height[left],height[right]))
        if height[left] <= height[right]:
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

**Recommended:** Use two pointers and discard the shorter boundary after evaluating the current area.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

The current area is width × shorter height. If the left wall is shorter, any pair retaining it has smaller width and no greater limiting height, so it cannot improve the current area. Discard it.

## 4. Interview Conversation

**Interviewer:** State the invariant behind the pointer movement.

**Candidate:** At every step, all pairs involving a discarded shorter boundary and a remaining point have been ruled out as better than the area already considered for that boundary.

**Interviewer:** Could you move both pointers when heights match?

**Candidate:** Yes. When equal, neither side is taller; moving both is safe after evaluating the current pair, though moving one is simpler.

**Interviewer:** How is this different from rain water trapping?

**Candidate:** This maximizes one pair's area. Rain Water computes the water level at every index and sums the trapped amount.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Explain the greedy proof. Empty/single-item input returns zero. For negative heights, the original physical interpretation and constraints no longer hold.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Two pointers are most powerful when a monotonic property lets you eliminate a range of candidates.

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

Use the same proof discipline when optimizing sorted joins and range scans; do not rely on a memorized pointer rule.

## 9. Final Interview Summary

**One-sentence explanation:** Use two pointers and discard the shorter boundary after evaluating the current area.

**Complexity to say aloud:** O(n) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #13 in the supplied study plan.*