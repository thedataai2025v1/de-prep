# Valid Anagram

**Difficulty:** Easy  
**Topics:** String, Hash Table, Sorting  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 12

## 1. Problem Statement

Given strings s and t, return true if t is an anagram of s; otherwise return false.

**Example:**
```text
s = "anagram", t = "nagaram"
Output = True
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Sort both strings and compare the sorted sequences.

```python
def is_anagram(s, t):
    return sorted(s) == sorted(t)
```

- **Complexity:** O(n log n) time; O(n) space for sorted copies

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Count character frequencies. For lowercase English letters, use a fixed array of 26 counters.

```python
def is_anagram(s, t):
    if len(s) != len(t): return False
    counts = [0] * 26
    for a, b in zip(s, t):
        counts[ord(a)-97] += 1
        counts[ord(b)-97] -= 1
    return all(x == 0 for x in counts)
```

- **Complexity:** O(n+26) time; O(1) auxiliary space for a-z

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Count character frequencies. For lowercase English letters, use a fixed array of 26 counters.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Each character in s increments a counter and the corresponding character in t decrements it. For anagram pairs, all 26 counters finish at zero.

## 4. Interview Conversation

**Interviewer:** What is the simplest solution?

**Candidate:** Sort both strings and compare; O(n log n).

**Interviewer:** Can you do better?

**Candidate:** Yes, count character frequencies in O(n). For lowercase letters, a 26-slot array uses constant space.

**Interviewer:** Why check lengths?

**Candidate:** Anagrams must have equal lengths, so it is a fast rejection.

**Interviewer:** What about Unicode?

**Candidate:** We need normalization and case-sensitivity rules. A dictionary counter or sorting supports a wider character set.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Ignore punctuation: normalize/filter first. Empty strings are anagrams. Group Anagrams uses a canonical key for many strings; this problem compares two strings.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Clarify whether whitespace, punctuation and case count. Test repeated characters, not only distinct letters.

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

Frequency comparison is a basic reconciliation pattern: compare distributions rather than ordering, but it is not a cryptographic integrity check.

## 9. Final Interview Summary

**One-sentence explanation:** Count character frequencies. For lowercase English letters, use a fixed array of 26 counters.

**Complexity to say aloud:** O(n+26) time; O(1) auxiliary space for a-z

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #12 in the supplied study plan.*