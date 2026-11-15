# Longest Repeating Character Replacement

**Difficulty:** Medium  
**Topics:** String, Hash Table, Sliding Window  
**Study-plan section:** Sliding Window  
**Problem ID:** 20

## 1. Problem Statement

Given an uppercase string s and integer k, return the length of the longest substring that can be made of one repeated character by replacing at most k characters.

**Example:**
```text
s = "AABABBA", k = 1
Output = 4
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Enumerate all substrings and count the most frequent character in each.

```python
from collections import Counter
def character_replacement(s, k):
    best = 0
    for left in range(len(s)):
        counts = Counter()
        for right in range(left, len(s)):
            counts[s[right]] += 1
            if right-left+1-max(counts.values()) <= k:
                best = max(best, right-left+1)
    return best
```

- **Complexity:** O(n²·alphabet) time; O(alphabet) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Maintain a sliding window and the highest character frequency seen in the window. A window is feasible when length - max_frequency <= k.

```python
def character_replacement(s, k):
    counts = {}
    left = best = max_freq = 0
    for right, ch in enumerate(s):
        counts[ch] = counts.get(ch, 0) + 1
        max_freq = max(max_freq, counts[ch])
        while (right-left+1) - max_freq > k:
            counts[s[left]] -= 1
            left += 1
        best = max(best, right-left+1)
    return best
```

- **Complexity:** O(n) time; O(alphabet) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Maintain a sliding window and the highest character frequency seen in the window. A window is feasible when length - max_frequency <= k.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

For AABABBA with k=1, window AABA has length 4 and max frequency 3; only one replacement is needed. A longer window may require more than one replacement, so shrink it.

## 4. Interview Conversation

**Interviewer:** How do you know how many replacements a window needs?

**Candidate:** Keep the most frequent character and replace every other character. Required replacements are window_length - max_frequency.

**Interviewer:** Why can max_freq remain stale when shrinking?

**Candidate:** A stale maximum may make the window appear more feasible than it is, but this standard algorithm still returns the correct maximum length because the window is only contracted when the condition fails and the maximum length is tracked. If exact current-window validity is needed, recompute it.

**Interviewer:** What if the alphabet is large?

**Candidate:** The dictionary grows with distinct characters; complexity is O(n) expected with hash operations.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Return the length, not necessarily the substring. A version requiring an exact valid window can recompute max frequency or maintain a more complex frequency structure. Test k=0 and k>=len(s).

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Be ready to explain the stale max-frequency optimization carefully; do not claim every intermediate window is exactly valid.

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

Sliding-window counts are useful for bounded event sequences and log pattern detection when ordering and window semantics are explicit.

## 9. Final Interview Summary

**One-sentence explanation:** Maintain a sliding window and the highest character frequency seen in the window. A window is feasible when length - max_frequency <= k.

**Complexity to say aloud:** O(n) time; O(alphabet) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #20 in the supplied study plan.*