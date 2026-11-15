# Remove Duplicates from Sorted Array

**Difficulty:** Easy  
**Topics:** Array, Two Pointers  
**Study-plan section:** Two Pointers  
**Problem ID:** 15

## 1. Problem Statement

Given a sorted array nums, remove duplicates in place so each distinct value appears once. Return k, the number of unique elements; the first k positions must contain the unique values in sorted order.

**Example:**
```text
nums = [1,1,2]
Return k = 2; nums[:k] = [1,2]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Build a new list by appending a value only when it differs from the last output value.

```python
def remove_duplicates(nums):
    unique = []
    for x in nums:
        if not unique or unique[-1] != x:
            unique.append(x)
    nums[:len(unique)] = unique
    return len(unique)
```

- **Complexity:** O(n) time; O(n) extra space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use a slow pointer for the next write position and a fast pointer to scan. Sorted order means duplicates are adjacent.

```python
def remove_duplicates(nums):
    if not nums: return 0
    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[write-1]:
            nums[write] = nums[read]
            write += 1
    return write
```

- **Complexity:** O(n) time; O(1) auxiliary space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use a slow pointer for the next write position and a fast pointer to scan. Sorted order means duplicates are adjacent.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

For [1,1,2], read skips the second 1 because it equals the last written value. It writes 2 at index 1 and returns k=2. Values beyond index 1 are irrelevant.

## 4. Interview Conversation

**Interviewer:** Why do you need two pointers?

**Candidate:** The read pointer scans every element; the write pointer marks where the next distinct value belongs.

**Interviewer:** Why compare with nums[write-1]?

**Candidate:** The input is sorted, so duplicates are adjacent and the last unique value is the last value written.

**Interviewer:** Do you need to delete the tail?

**Candidate:** No. The contract only requires the first k positions to contain unique values; the tail can be ignored.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

At most two occurrences: compare with nums[write-2]. Unsorted input: a set can identify uniques, but order and in-place constraints need clarification.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Empty array returns 0. Single item returns 1. Input is mutated.

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

Read/write compaction is useful for filtering sorted or ordered datasets in place; in distributed systems, equivalent filtering is usually a separate transformation.

## 9. Final Interview Summary

**One-sentence explanation:** Use a slow pointer for the next write position and a fast pointer to scan. Sorted order means duplicates are adjacent.

**Complexity to say aloud:** O(n) time; O(1) auxiliary space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #15 in the supplied study plan.*