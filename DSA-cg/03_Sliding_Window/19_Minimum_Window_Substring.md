# Minimum Window Substring

**Difficulty:** Hard  
**Topics:** String, Hash Table, Sliding Window  
**Study-plan section:** Sliding Window  
**Problem ID:** 19

## 1. Problem Statement

Given strings s and t, return the minimum-length substring of s containing every character in t with the required multiplicity. Return an empty string if no such window exists.

**Example:**
```text
s = "ADOBECODEBANC", t = "ABC"
Output = "BANC"
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Enumerate all substrings and test whether each contains the required character counts.

```python
from collections import Counter
def min_window(s, t):
    need = Counter(t)
    best = ""
    for i in range(len(s)):
        have = Counter()
        for j in range(i, len(s)):
            have[s[j]] += 1
            if all(have[ch] >= count for ch, count in need.items()):
                if not best or j-i+1 < len(best):
                    best = s[i:j+1]
                break
    return best
```

- **Complexity:** O(n²·u) or worse depending on counter checks; O(u) temporary state

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Expand the right boundary until all required counts are satisfied, then shrink from the left while the window remains valid.

```python
from collections import Counter
def min_window(s, t):
    if not s or not t: return ""
    need = Counter(t)
    missing = len(t)
    left = 0
    best_start, best_len = 0, float('inf')
    for right, ch in enumerate(s):
        if need[ch] > 0:
            missing -= 1
        need[ch] -= 1
        while missing == 0:
            if right-left+1 < best_len:
                best_start, best_len = left, right-left+1
            need[s[left]] += 1
            if need[s[left]] > 0:
                missing += 1
            left += 1
    return "" if best_len == float('inf') else s[best_start:best_start+best_len]
```

- **Complexity:** O(|s|+|t|) time; O(u) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Expand the right boundary until all required counts are satisfied, then shrink from the left while the window remains valid.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Expand through the string until the window covers A, B and C. When the window reaches `BANC`, it is valid. Shrink from the left, recording shorter valid windows; removing the leading B would make it invalid, so BANC is minimal.

## 4. Interview Conversation

**Interviewer:** What makes this harder than longest substring?

**Candidate:** The window must satisfy multiplicity requirements from t, not merely a property of distinct characters.

**Interviewer:** What does missing track?

**Candidate:** It tracks how many required character instances are still missing. It starts at len(t), decreases when a needed occurrence enters, and increases when a needed occurrence leaves.

**Interviewer:** Why can you shrink while missing is zero?

**Candidate:** Every current window is valid, so shrinking may find a shorter valid answer. I stop once removing a character makes it invalid.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Duplicate characters in t must be counted, not treated as a set. If t is empty, return empty according to the usual contract. Unicode normalization is a separate requirement.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Explain why each pointer only moves forward, yielding linear time. A common bug is decrementing missing for surplus characters.

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

Windowed event correlation can use the same pattern when a bounded sequence must contain required event types, but event time and late arrivals complicate it.

## 9. Final Interview Summary

**One-sentence explanation:** Expand the right boundary until all required counts are satisfied, then shrink from the left while the window remains valid.

**Complexity to say aloud:** O(|s|+|t|) time; O(u) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #19 in the supplied study plan.*