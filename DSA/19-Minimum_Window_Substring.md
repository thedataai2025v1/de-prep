# Minimum Window Substring

| Category | Details |
|---|---|
| **Difficulty** | Hard |
| **Topics** | String, Sliding Window, Hash Map |
| **Companies** | Amazon, Google, Meta, Microsoft |
| **Interview Frequency** | Medium |

---

## 1. Problem Statement

Given two strings `s` and `t`, return the minimum window in `s` which contains all the characters in `t`. If no such window exists, return an empty string.

If there is exactly one minimum window, you are guaranteed that it will be unique.

**Example:**
```
Input:  s = "ADOBECODEBANC", t = "ABC"
Output: "BANC" (shortest window containing A, B, and C)

Input:  s = "a", t = "a"
Output: "a"

Input:  s = "a", t = "aa"
Output: "" (t requires two 'a's but s has only one)
```

---

## 2. Approaches

### Approach 1: Sliding Window with Character Counts (⭐ Optimal / Interview Choice)

**Idea:** Use a sliding window with two pointers. Expand the right pointer to include all required characters, then shrink from the left to find the minimum valid window.

**Pros:**
- Optimal O(|s| + |t|) time — each character visited at most twice
- O(1) extra space — character frequency maps bounded by alphabet size (at most 128 ASCII)
- Classic sliding window pattern — demonstrates mastery of an important technique
- Easily adaptable to variations (e.g., with character limits, multiple targets)

**Cons:**
- Complex logic with two pointers and a "required" counter — harder to get right in one attempt
- Requires careful tracking of what counts as a "valid" window
- Edge cases with unicode/non-ASCII characters need special handling
- Not immediately intuitive — requires understanding of the sliding window pattern

```python
def minWindow(s: str, t: str) -> str:
    if not s or not t:
        return ""

    from collections import Counter

    target_count = Counter(t)
    window_count = Counter()
    required = len(target_count)  # number of unique chars needed
    formed = 0  # number of unique chars that meet their count requirement

    # [left, right] sliding window
    left = 0
    min_len = float('inf')
    result = ""

    for right, char in enumerate(s):
        window_count[char] += 1

        if char in target_count and window_count[char] == target_count[char]:
            formed += 1

        # Try to shrink from the left while the window is still valid
        while formed == required:
            if right - left + 1 < min_len:
                min_len = right - left + 1
                result = s[left:right + 1]

            left_char = s[left]
            window_count[left_char] -= 1
            if left_char in target_count and window_count[left_char] < target_count[left_char]:
                formed -= 1
            left += 1

    return result
```

| Metric | Value |
|---|---|
| **Time** | O(|s| + |t|) — each character in s visited at most twice, t once for counting |
| **Space** | O(1) — bounded by alphabet size (e.g., 128 for ASCII) |

---

### Approach 2: Brute Force All Substrings (Alternative)

**Idea:** Check every substring of `s` to see if it contains all characters from `t`. Track the shortest valid one.

**Pros:**
- Simple and straightforward logic — easy to understand
- Correct for all edge cases by exhaustive search
- Good baseline to discuss before optimizing

**Cons:**
- O(|s|² × |t|) or O(|s|³) time — far too slow for large inputs
- Creates substring copies at each step — extra memory overhead
- No pruning — checks invalid windows just as thoroughly as valid ones
- Will time out on any real-world or interview-scale input

```python
def minWindow(s: str, t: str) -> str:
    from collections import Counter

    target_count = Counter(t)
    result = ""
    min_len = float('inf')

    for i in range(len(s)):
        for j in range(i + len(t), len(s) + 1):
            window = s[i:j]
            if all(c in window and window.count(c) >= target_count[c]
                   for c in target_count):
                if j - i < min_len:
                    min_len = j - i
                    result = window

    return result
```

| Metric | Value |
|---|---|
| **Time** | O(|s|³) — substring creation and character checking |
| **Space** | O(|t|) — target count map |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| `t` longer than `s` | Return `""` — impossible to contain all characters |
| `t` has more of a char than `s` | Return `""` — insufficient supply in `s` |
| `s == t` | Return `s` — the entire string is the minimum window |
| `t` has repeating characters | Must match the exact count (e.g., `t="AA"` needs two A's in window) |
| `s` has multiple valid windows | Return the first one found (smallest leftmost) |

---

## 4. Interview Talking Points

- "The sliding window approach is the gold standard for substring containment problems. The key insight is that once the window is valid, we only shrink it, never expand it again until we find a new candidate."
- "In data engineering, this pattern is critical for stream processing: finding the shortest window of events that satisfies a set of conditions. This maps directly to session detection, anomaly detection windows, or trigger conditions in real-time dashboards."

---

## 5. Interview Follow-Up Questions

> **Be prepared for these — they differentiate junior from senior candidates.**

**Q1. What if `t` contains duplicate characters?**
→ The `Counter` approach naturally handles this: each character's count in the window must match its count in `t`. The `formed` counter only increments when `window_count[char] == target_count[char]`.

**Q2. Can you solve this in a single pass?**
→ The algorithm is effectively single-pass: the right pointer moves forward once, and the left pointer only moves forward. Each character is processed at most twice (once by right, once by left).

**Q3. How would you handle case sensitivity?**
→ Convert both strings to lowercase (or uppercase) before processing. Alternatively, treat each case as distinct — the Counter approach handles both scenarios naturally.

**Q4. What if you need the window indices instead of the substring?**
→ Track `best_left` and `best_right` indices instead of the substring. Return `(best_left, best_right)` or the substring using those indices.

**Q5. What if there are multiple windows of the same minimum length?**
→ The problem guarantees uniqueness. If not guaranteed, track all minimum windows or return the leftmost one. The current implementation returns the first found (leftmost).

---

## 6. Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Using `set(t)` instead of `Counter(t)` | Misses duplicate character counts (e.g., `t="AA"` needs 2 A's) | Use `Counter` to track exact character frequencies |
| Shrinking too aggressively | Window becomes invalid before checking if it's the minimum | Check minimum length *before* shrinking, then shrink |
| Not resetting `formed` correctly | Window appears valid when it's not | Decrement `formed` only when `window_count[left_char] < target_count[left_char]` |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Longest Repeating Character Replacement](./20-Longest_Repeating_Char_Replacement.md) | Same sliding window pattern, different validity condition |
| [Permutation in String](https://leetcode.com/problems/permutation-in-string/) | Sliding window with fixed-size constraint |
| [Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/) | Fixed-size sliding window with character frequency matching |

---

*Last updated: 2025 — for data engineering interview preparation.*