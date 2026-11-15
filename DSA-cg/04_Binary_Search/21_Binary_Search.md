# Binary Search

**Difficulty:** Easy  
**Topics:** Array, Binary Search  
**Study-plan section:** Binary Search  
**Problem ID:** 21

## 1. Problem Statement

Given a sorted array nums and target, return its index if present, otherwise -1. Use O(log n) time.

**Example:**
```text
nums = [-1,0,3,5,9,12], target = 9
Output = 4
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Scan every element until target is found.

```python
def search(nums, target):
    for i, x in enumerate(nums):
        if x == target:
            return i
    return -1
```

- **Complexity:** O(n) time; O(1) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Maintain an inclusive search interval. Compare the middle value and discard the half that cannot contain the target.

```python
def search(nums, target):
    left, right = 0, len(nums)-1
    while left <= right:
        mid = left + (right-left)//2
        if nums[mid] == target:
            return mid
        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
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

**Recommended:** Maintain an inclusive search interval. Compare the middle value and discard the half that cannot contain the target.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Middle index is 2 with value 3, less than 9, so search right half. Next middle is 4 with value 9, so return 4. Each iteration halves the remaining range.

## 4. Interview Conversation

**Interviewer:** What property makes binary search possible?

**Candidate:** The array is sorted, so comparing the middle tells me which half can be discarded.

**Interviewer:** Why use left <= right?

**Candidate:** The interval is inclusive. When left equals right, one candidate remains and must still be checked.

**Interviewer:** How do you avoid overflow in other languages?

**Candidate:** Use left + (right-left)//2 rather than (left+right)//2.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

First/last occurrence: use lower/upper bound variants. Rotated arrays require identifying which half is sorted. Duplicates can make the decision ambiguous.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Empty array returns -1. Clarify duplicate target semantics if duplicates are allowed.

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

Binary search applies to monotonic predicates, not only sorted arrays; useful for threshold and capacity planning problems.

## 9. Final Interview Summary

**One-sentence explanation:** Maintain an inclusive search interval. Compare the middle value and discard the half that cannot contain the target.

**Complexity to say aloud:** O(log n) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #21 in the supplied study plan.*