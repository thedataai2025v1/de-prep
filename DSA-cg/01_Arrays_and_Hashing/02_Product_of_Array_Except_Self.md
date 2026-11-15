# Product of Array Except Self

**Difficulty:** Medium  
**Topics:** Array, Prefix Product, Suffix Product  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 02

## 1. Problem Statement

Return answer[i] as the product of every element in nums except nums[i]. Do not use division; target O(n) time.

**Example:**
```text
nums = [1, 2, 3, 4]
Output: [24, 12, 8, 6]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** For each index, multiply all other elements directly. This is a useful correctness baseline.

```python
def product_except_self(nums):
    out = []
    for i in range(len(nums)):
        product = 1
        for j, x in enumerate(nums):
            if i != j:
                product *= x
        out.append(product)
    return out
```

- **Complexity:** O(n²) time; O(n) output space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Write prefix products into the output, then multiply by a running suffix product in reverse.

```python
def product_except_self(nums):
    n = len(nums)
    out = [1] * n
    prefix = 1
    for i, x in enumerate(nums):
        out[i] = prefix
        prefix *= x
    suffix = 1
    for i in range(n - 1, -1, -1):
        out[i] *= suffix
        suffix *= nums[i]
    return out
```

- **Complexity:** O(n) time; O(1) auxiliary space excluding output

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Write prefix products into the output, then multiply by a running suffix product in reverse.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

For [1,2,3,4], the prefix pass writes [1,1,2,6]. The reverse pass multiplies by suffixes [24,12,4,1] at the corresponding positions, producing [24,12,8,6].

## 4. Interview Conversation

**Interviewer:** How can you avoid division?

**Candidate:** The product excluding index i is the product to its left times the product to its right.

**Interviewer:** Can you use constant extra space?

**Candidate:** Yes. Store prefixes in the output array, then walk backward with one suffix accumulator.

**Interviewer:** What if there are zeros?

**Candidate:** This approach handles zeros naturally. One zero leaves only that position potentially non-zero; two or more zeros make every result zero.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Why not divide: division fails with zeros and may introduce numeric precision concerns. Output space is normally excluded from auxiliary-space accounting. For a stream, clarify whether an ordered two-pass scan is possible.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Prefix/suffix patterns support cumulative calculations, running aggregates and scan algorithms.

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

Distributed prefix/suffix operations require ordering and partition-boundary coordination; a simple global product is not enough to produce every row's result.

## 9. Final Interview Summary

**One-sentence explanation:** Write prefix products into the output, then multiply by a running suffix product in reverse.

**Complexity to say aloud:** O(n) time; O(1) auxiliary space excluding output

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #02 in the supplied study plan.*