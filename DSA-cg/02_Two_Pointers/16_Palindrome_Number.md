# Palindrome Number

**Difficulty:** Easy  
**Topics:** Math, Two Pointers (digit symmetry)  
**Study-plan section:** Two Pointers  
**Problem ID:** 16

## 1. Problem Statement

Given an integer x, return true if x reads the same forward and backward. Negative numbers are not palindromes.

**Example:**
```text
x = 121 → True
x = -121 → False
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Convert to a string and compare it with its reverse.

```python
def is_palindrome(x):
    s = str(x)
    return s == s[::-1]
```

- **Complexity:** O(d) time; O(d) space, d = number of digits

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Reverse only the second half numerically and compare it with the first half; this avoids string conversion and overflow concerns.

```python
def is_palindrome(x):
    if x < 0 or (x % 10 == 0 and x != 0):
        return False
    reversed_half = 0
    while x > reversed_half:
        reversed_half = reversed_half * 10 + x % 10
        x //= 10
    return x == reversed_half or x == reversed_half // 10
```

- **Complexity:** O(d) time; O(1) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Reverse only the second half numerically and compare it with the first half; this avoids string conversion and overflow concerns.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

For 1221, move digits from the end into reversed_half until it reaches 12; the remaining x is 12, so they match. For 12321, the reversed half becomes 123 and x becomes 12; dropping the middle digit gives 12.

## 4. Interview Conversation

**Interviewer:** What quick cases can you reject?

**Candidate:** Negative numbers are false. A non-zero number ending in zero is also false because it would need a leading zero.

**Interviewer:** Can you avoid converting to a string?

**Candidate:** Reverse only half the digits, then compare the two halves. For odd digit counts, remove the middle digit from the reversed half.

**Interviewer:** Why only half?

**Candidate:** Once the reversed half has at least as many digits as the remaining half, we have enough information to compare.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

String reversal is simplest and often acceptable. Numeric half reversal uses O(1) space. Clarify integer overflow in fixed-width languages.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Zero is a palindrome. 10 is not. 1001 is. The problem is arithmetic rather than a literal two-pointer array problem.

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

Useful as a symmetry reasoning exercise. In interviews, challenge the given topic label when a problem is better categorized under math.

## 9. Final Interview Summary

**One-sentence explanation:** Reverse only the second half numerically and compare it with the first half; this avoids string conversion and overflow concerns.

**Complexity to say aloud:** O(d) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #16 in the supplied study plan.*