# Minimum Window Substring — Interview Deep Dive

**Difficulty:** Hard  
**Topics:** Sliding window, frequency deficits, multiplicity

> A dedicated advanced companion to the existing Minimum Window Substring lesson. Focus here is on the invariant, correctness argument and production-style edge cases rather than repeating the basic pattern.

## 1. Problem statement

Given strings `s` and `t`, return the shortest substring of `s` containing every character of `t` with the required multiplicity. If no window exists, return `""`.

## 2. Approaches

### Brute force

Enumerate all windows and count their characters.

```python
from collections import Counter

def min_window_brute(s, t):
    if not t:
        return ""
    need = Counter(t)
    best = ""
    for left in range(len(s)):
        counts = Counter()
        for right in range(left, len(s)):
            counts[s[right]] += 1
            if all(counts[ch] >= amount for ch, amount in need.items()):
                candidate = s[left:right + 1]
                if not best or len(candidate) < len(best):
                    best = candidate
                break
    return best
```
Worst-case O(n² * u) time with repeated validation; O(u) counting space plus output.

### Optimized: sliding window with formed/required counts

```python
from collections import Counter

def min_window(s, t):
    if not s or not t or len(t) > len(s):
        return ""
    need = Counter(t)
    have = {}
    required = len(need)
    formed = 0
    left = 0
    best_len = float("inf")
    best_start = 0

    for right, ch in enumerate(s):
        have[ch] = have.get(ch, 0) + 1
        if ch in need and have[ch] == need[ch]:
            formed += 1

        while formed == required:
            window_len = right - left + 1
            if window_len < best_len:
                best_len, best_start = window_len, left

            outgoing = s[left]
            have[outgoing] -= 1
            if outgoing in need and have[outgoing] < need[outgoing]:
                formed -= 1
            left += 1

    return "" if best_len == float("inf") else s[best_start:best_start + best_len]
```
O(n + m) expected time and O(u) space, where n=len(s), m=len(t), u is the number of distinct relevant characters.

## 3. Detailed dry run

`s="ADOBECODEBANC"`, `t="ABC"`.
- Expand until `ADOBEC` contains A, B and C.
- Shrink from the left while valid; removing A makes it invalid.
- Continue expanding. The later valid window `BANC` contains all required characters.
- Shrink to `ANC`? It lacks B, so invalid; `BANC` is the shortest valid window found, length 4.

## 4. Correctness invariant

`formed` counts how many distinct required characters currently meet their required frequency. The window is valid exactly when `formed == required`. While valid, shrink from the left to test whether a shorter valid window exists; when it becomes invalid, resume expansion.

## 5. Interviewer–candidate discussion

**Interviewer:** Why compare equality `have[ch] == need[ch]` when expanding?

**Candidate:** That is the exact moment the character becomes satisfied. Additional copies should not increment `formed` again.

**Interviewer:** Why decrement only when count drops below need?

**Candidate:** The character stops being satisfied only when the outgoing removal makes its count insufficient.

## 6. Edge cases and mistakes

Empty t, t longer than s, repeated target characters (`"AABC"`), no valid window, multiple equal-length windows, case sensitivity, Unicode code points. Do not compare only distinct-character presence; multiplicity matters.

## 7. Tests and variations

```python
assert min_window("ADOBECODEBANC", "ABC") == "BANC"
assert min_window("a", "a") == "a"
assert min_window("a", "aa") == ""
assert min_window("aa", "aa") == "aa"
assert min_window("ab", "b") == "b"
assert min_window("abc", "d") == ""
```

## 8. Follow-ups

1. Return start/end offsets rather than a copied substring.
2. Handle case-insensitive matching with a documented normalization strategy.
3. Stream the source and bound retained data.
4. Explain why the left pointer never moves backward.

**Principal-level connection:** This pattern can match required event types or attributes in a bounded event sequence, but production use must define event identity, duplicate semantics and retention limits.
