# Search in Rotated Sorted Array

**Difficulty:** Medium  
**Topics:** Array, Binary Search  
**Study-plan section:** Binary Search  
**Problem ID:** 22

## 1. Problem Statement

Search target in a strictly increasing array that has been rotated at an unknown pivot. Return its index or -1. Aim for O(log n).

**Example:**
```text
nums = [4,5,6,7,0,1,2], target = 0
Output = 4
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Scan every value until the target is found.

```python
def search_rotated(nums, target):
    for i, x in enumerate(nums):
        if x == target: return i
    return -1
```

- **Complexity:** O(n) time; O(1) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** At each midpoint, at least one half is sorted. Determine whether target lies within that sorted half and discard the other half.

```python
def search_rotated(nums, target):
    left, right = 0, len(nums)-1
    while left <= right:
        mid = (left+right)//2
        if nums[mid] == target: return mid
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid-1
            else:
                left = mid+1
        else:
            if nums[mid] < target <= nums[right]:
                left = mid+1
            else:
                right = mid-1
    return -1
```

- **Complexity:** O(log n) time; O(1) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** At each midpoint, at least one half is sorted. Determine whether target lies within that sorted half and discard the other half.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

For [4,5,6,7,0,1,2], mid points to 7. The left half [4,5,6,7] is sorted, but 0 is not in it, so discard it. The next midpoint finds 0 at index 4.

## 4. Interview Conversation

**Interviewer:** How do you adapt binary search after rotation?

**Candidate:** At least one side of the midpoint is sorted. I identify that side, check whether the target lies in its range, and discard the other side.

**Interviewer:** What if nums[left] equals nums[mid]?

**Candidate:** With duplicates, sorted-half identification can become ambiguous. The usual problem's distinct values avoid that case; a duplicate variant may degrade to O(n).

**Interviewer:** Why are the range checks asymmetric?

**Candidate:** They avoid checking the midpoint twice: the left range uses target < nums[mid], and the right range uses target > nums[mid].

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Duplicates: handle ambiguous equal endpoints by shrinking a boundary, potentially O(n). Find minimum in rotated sorted array is a related binary-search problem.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Empty array and target absent. Use strict inequalities consistently to avoid infinite loops.

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

Binary search over rotated sorted data illustrates how invariants survive a transformed ordering.

## 9. Final Interview Summary

**One-sentence explanation:** At each midpoint, at least one half is sorted. Determine whether target lies within that sorted half and discard the other half.

**Complexity to say aloud:** O(log n) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #22 in the supplied study plan.*