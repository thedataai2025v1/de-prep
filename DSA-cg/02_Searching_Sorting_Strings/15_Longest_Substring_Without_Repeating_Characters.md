# Longest Substring Without Repeating Characters

**Difficulty:** Medium  
**Topics:** Sliding window, hash map, last-seen index

## 1. Problem statement

Return the length of the longest contiguous substring containing no repeated characters.

## 2. Approaches

### Brute force

```python
def longest_unique_brute(s):
    best = 0
    for start in range(len(s)):
        seen = set()
        for end in range(start, len(s)):
            if s[end] in seen:
                break
            seen.add(s[end])
            best = max(best, end - start + 1)
    return best
```
O(n²) time, O(min(n, alphabet)) space.

### Optimized sliding window

```python
def longest_unique_substring(s):
    last_seen = {}
    left = 0
    best = 0
    best_start = 0
    for right, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= left:
            left = last_seen[ch] + 1
        last_seen[ch] = right
        if right - left + 1 > best:
            best = right - left + 1
            best_start = left
    return best

def longest_unique_value(s):
    length = longest_unique_substring(s)
    return length
```
O(n) expected time, O(u) space for distinct characters retained in the map.

## 3. Dry run

For `"abcabcbb"`:
- a at 0, window `"a"` length 1.
- b at 1, window `"ab"` length 2.
- c at 2, window `"abc"` length 3.
- a at 3 was last seen at 0, so move left to 1; window `"bca"` length 3.
- b at 4 was last seen at 1, so move left to 2; window `"cab"` length 3.
- Continue; maximum remains 3.

## 4. Interviewer–candidate discussion

**Interviewer:** Why check `last_seen[ch] >= left`?

**Candidate:** A previous occurrence before the current window must not move the left pointer backward. The check ensures the duplicate is inside the active window.

**Interviewer:** Can left move backward?

**Candidate:** No. It only moves to `last_seen[ch] + 1` when that position is greater than or equal to the current left.

## 5. Complexity and pitfalls

O(n) expected time, O(u) space. A common bug is assigning `left = last_seen[ch] + 1` unconditionally, which can move left backward. Clarify whether “character” means Unicode code point or grapheme cluster.

## 6. Tests and variations

```python
assert longest_unique_substring("") == 0
assert longest_unique_substring("a") == 1
assert longest_unique_substring("abcabcbb") == 3
assert longest_unique_substring("bbbbb") == 1
assert longest_unique_substring("pwwkew") == 3
assert longest_unique_substring("abba") == 2
```
Follow-ups: return the substring itself; stream input; use a fixed-size array for a constrained ASCII alphabet; discuss memory bounds for unbounded character sets.
