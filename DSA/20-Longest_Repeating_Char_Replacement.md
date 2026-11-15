# Longest Repeating Character Replacement

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | String, Sliding Window, Hash Table |
| **Companies** | Google, Meta, Microsoft, Amazon, Bloomberg |
| **Interview Frequency** | High |

---

## 1. Problem Statement

You are given a string `s` and an integer `k`. You can choose any character of the string and change it to any other uppercase English character. You can perform this operation at most `k` times.

Return the length of the longest substring containing the same letter you can get after performing the above operations.

**Example:**
```
Input:  s = "AABABBA", k = 1
Output: 4
Explanation: Replace the one 'B' in "AABA" to get "AAAA".

Input:  s = "AABABBA", k = 2
Output: 6
Explanation: Replace the two 'A's in "ABABBA" to get "BBBBBB".

Input:  s = "AAABABBCCAA", k = 2
Output: 5
```

---

## 2. Approaches

### Approach 1: Sliding Window Tracking Max Freq Char (⭐ Optimal)  

Expand the window to the right. Track the maximum frequency of any character in the current window. If `window_size - max_freq > k`, shrink from the left. The window never shrinks in size — it only grows, keeping track of the maximum valid window seen.

**Pros:**
- Optimal O(n) time — single pass, each character visited at most twice
- O(1) extra space — bounded by 26 uppercase English letters
- Elegant insight: `window_size - max_freq` directly equals replacements needed
- The window only grows (never shrinks), simplifying the tracking logic

**Cons:**
- The "running max_freq" trick can be confusing — stale values stretch the window but still produce correct max length
- Does not identify which character becomes the uniform target
- Doesn't return the actual substring, just its length
- Requires careful explanation to justify why a stale `max_freq` is acceptable

```python
from collections import Counter

def characterReplacement(s: str, k: int) -> int:
    count = Counter()
    max_freq = 0  # max occurrences of any single char in current window
    left = 0
    result = 0

    for right in range(len(s)):
        # Add current character to the window
        count[s[right]] += 1
        # Update the max frequency seen so far
        max_freq = max(max_freq, count[s[right]])

        # If replacements needed exceed k, shrink window from left
        window_size = right - left + 1
        if window_size - max_freq > k:
            count[s[left]] -= 1
            left += 1

        # Window size = right - left + 1 (never shrinks in net size)
        result = max(result, right - left + 1)

    return result
```

| Metric | Value |
|---|---|
| **Time** | O(n) where n = len(s) — each character visited at most twice |
| **Space** | O(1) — at most 26 entries in the counter (uppercase English letters) |

### Approach 2: Expand and Shrink Fully (Alternative)

A more intuitive but slightly different variant: shrink the window every time the condition is violated, maintaining a strictly valid window at all times.

**Pros:**
- More intuitive — the window is always valid after shrinking
- No stale `max_freq` concerns — the window is strictly correct at all times
- Easier to explain step-by-step during an interview
- Naturally handles edge cases (k = 0, empty string)

**Cons:**
- Uses a `while` loop instead of `if` — more pointer movement on each violation
- Still doesn't return the actual substring
- The amortized O(n) analysis requires explanation (each element removed at most once)
- Slightly more variable overhead per iteration compared to the "if" variant

```python
from collections import Counter

def characterReplacement(s: str, k: int) -> int:
    count = Counter()
    max_freq = 0
    left = 0
    max_length = 0

    for right in range(len(s)):
        count[s[right]] += 1
        max_freq = max(max_freq, count[s[right]])

        # Shrink until window is valid again
        while (right - left + 1) - max_freq > k:
            count[s[left]] -= 1
            left += 1

        max_length = max(max_length, right - left + 1)

    return max_length
```

| Metric | Value |
|---|---|
| **Time** | O(n) — amortized: each element added once, removed at most once |
| **Space** | O(1) — bounded by character set (26 uppercase letters) |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| `k = 0` | Find the longest substring of identical characters |
| `k >= len(s)` | Return `len(s)` — can replace all characters |
| Single character `"A"` | Return 1 |
| All same characters `"AAAA"` | Return `len(s)` — no replacements needed |
| Empty string `""` | Return 0 |

---

## 4. Interview Talking Points

- "The key insight is that `window_size - max_freq` tells us how many replacements we need. When this exceeds `k`, we shrink. We track `max_freq` as a running maximum — it never decreases, which means the window only grows in size overall."
- "In data engineering, this pattern maps to tolerance-based grouping in streaming data: allowing a configurable error rate (k replacements) while finding the longest valid segment. Useful for data normalization in noisy streams, outlier tolerance in sensor data, or fuzzily matching patterns in log parsing."

---

## 5. Interview Follow-Up Questions

> **Be prepared for these — they differentiate junior from senior candidates.**

**Q1. What if we needed to return the actual substring, not just its length?**
→ Track `best_left` and `best_right` whenever `result` is updated. After the loop, return `s[best_left:best_right + 1]`.

**Q2. Why is it acceptable for `max_freq` to be stale after shrinking?**
→ A stale (higher) `max_freq` only makes the condition `window_size - max_freq > k` easier to pass, which means the window may grow slightly beyond where it strictly should. However, this never reduces the maximum window size we've already found — it only helps us potentially find a larger one.

**Q3. How would this change if characters could be any ASCII value (not just uppercase)?**
→ Use a `Counter` or `defaultdict(int)` instead of a fixed-size array of 26. The algorithm stays the same; only the space bound changes to O(|alphabet|).

**Q4. What if `k = 0`?**
→ The problem reduces to finding the longest consecutive run of identical characters. The algorithm still works: `window_size - max_freq > 0` fires whenever a different character is encountered, forcing the window to shrink.

**Q5. How does this relate to finding the longest substring with at most k distinct characters?**
→ They are complementary sliding window problems. Here we allow at most `k` *changes* (cost to make the window uniform). The other problem allows at most `k` *distinct characters* in the window. The sliding window pattern is the same but the condition differs.

---

## 6. Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Recomputing `max_freq` on shrink | `max_freq` might temporarily be wrong, but the window size cap prevents incorrect results | Keep `max_freq` as a running maximum — it's OK to be slightly stale |
| Shrinking until window is exactly valid | Misses the optimization that we only need to find the max window size | Use `if` instead of `while` to only shrink by one step when needed |
| Not handling `k >= len(s)` | Could overcomplicate with unnecessary shrinking | Early return `len(s)` or let the algorithm handle it naturally |
| Using `max(count.values())` each time | O(26) per step adds up; also slow in practice | Track `max_freq` as a separate variable updated on each add |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Minimum Window Substring](./19-Minimum_Window_Substring.md) | Sliding window with character counts and a tolerance threshold |
| [Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii/) | Same pattern: flip at most k values to maximize consecutive 1s |
| [Replace the Substring for Balanced String](https://leetcode.com/problems/replace-the-substring-for-balanced-string/) | Sliding window with character count balancing |

---

*Last updated: 2025 — for data engineering interview preparation.*
