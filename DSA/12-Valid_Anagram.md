# Valid Anagram

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Hash Table, String, Sorting |
| **Companies** | Google, Amazon, Facebook, Microsoft, Apple |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given two strings `s` and `t`, return `true` if `t` is an **anagram** of `s`, and `false` otherwise.

An **anagram** is a word or phrase formed by rearranging the letters of a different word or phrase, using all the original letters exactly once.

**Example:**

```
Input:  s = "anagram", t = "nagaram"
Output: true

Input:  s = "rat", t = "car"
Output: false
```

**Constraints (LeetCode):**
- `1 <= s.length, t.length <= 5 * 10^4`
- `s` and `t` consist of lowercase English letters.

---

## 2. Approaches

### Approach 1: Character Count Hash Map (⭐ Optimal)

**Idea:** Use a fixed-size array (26 for lowercase letters) to count character frequencies. Increment counts for `s`, decrement for `t`. If all counts are zero at the end, the strings are anagrams.

```python
def isAnagram(s, t):
    if len(s) != len(t):
        return False

    count = [0] * 26

    for i in range(len(s)):
        count[ord(s[i]) - ord('a')] += 1
        count[ord(t[i]) - ord('a')] -= 1

    return all(c == 0 for c in count)
```

| Metric | Value |
|---|---|
| **Time** | O(s + t) — linear over both strings |
| **Space** | O(1) — constant 26-element array |

**Pros:**
- Optimal time complexity
- Constant space (fixed alphabet)
- Fast for large strings

**Cons:**
- Assumes fixed lowercase English alphabet
- Requires character-to-index conversion

---

### Approach 2: Sort Both Strings and Compare

**Idea:** Sort both strings alphabetically. If they are anagrams, the sorted versions will be identical.

```python
def isAnagram(s, t):
    return sorted(s) == sorted(t)
```

| Metric | Value |
|---|---|
| **Time** | O(n log n) — dominated by sorting |
| **Space** | O(n) — for sorted arrays (or O(1) depending on sort implementation) |

**Pros:**
- Extremely concise (one line)
- Works with any character set (Unicode, etc.)
- Readable and easy to verify

**Cons:**
- Slower than the counting approach for large strings
- Extra space for the sorted arrays

---

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Character Count** ⭐ | O(s + t) | O(1) | **Interview choice** — optimal for lowercase letters |
| **2. Sort + Compare** | O(n log n) | O(n) | Quick prototype / Unicode / readability |

---

## 3. Edge Cases

> **Always mention these in an interview to show thoroughness.**

| Edge Case | Input | Expected Behavior |
|---|---|---|
| Different lengths | `s = "ab"`, `t = "abc"` | Return `false` immediately (length check) |
| Empty strings | `s = ""`, `t = ""` | Return `true` |
| Single character | `s = "a"`, `t = "a"` | Return `true` |
| Same string | `s = "abc"`, `t = "abc"` | Return `true` |
| Completely different | `s = "abc"`, `t = "xyz"` | Return `false` |
| Case sensitivity | `s = "Ab"`, `t = "aB"` | Return `false` unless case is normalized |
| Unicode / special chars | `s = "café"`, `t = "facé"` | Use sorting approach or Counter |

---

## 4. Interview Talking Points

- **"The character count approach is optimal: O(n) time and O(1) space because the alphabet is limited to 26 lowercase letters. This cannot be improved."**
- **Data Engineering Connection:** Validating anagrams models data deduplication via canonical hashing — transforming data into a normalized key (sorted or frequency-tuple) to identify duplicates, used in ETL pipelines for cleaning and merging records.

---

## 5. Interview Follow-Up Questions

**Q1. What if the input contains Unicode characters?**
→ Use the sorting approach (Approach 2) or `collections.Counter` to build a frequency dictionary. A fixed 26-element array only works for lowercase English letters.

**Q2. How would you handle very large strings (millions of characters)?**
→ Use the character count approach with a hash map (to support arbitrary characters) or process in chunks with a streaming frequency counter. The counting approach scales linearly.

**Q3. What is the time complexity of the sorting approach?**
→ O(n log n) for comparison-based sorting. Python's `sorted()` uses Timsort, which is O(n log n) worst-case.

**Q4. Can you solve it without extra space?**
→ Sorting is the best in-place approach (O(log n) space for recursion stack). The counting approach uses a small array (O(1)). There's no O(1) time approach without extra space because we must examine all characters.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Not checking length first | Sorting both strings of different lengths wastes time | Early return `false` if lengths differ |
| Using `Counter(s) == Counter(t)` without import | NameError: `Counter` not imported | `from collections import Counter` |
| Forgetting lowercase constraint | Using fixed array for Unicode strings | Fall back to sorting or `Counter` |
| Modifying original strings | Strings are immutable in Python; not a bug but affects approach | Use `sorted()` which returns a new list |
| Off-by-one in character index | `ord('a')` is 97; forgetting to subtract gives index 97+ | Always subtract `ord('a')` |

---

## 7. Related Problems

| Problem | Key Concept | Difficulty |
|---|---|---|
| [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Group all anagrams together | Medium |
| [Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/) | Sliding window + frequency count | Medium |
| [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | String compare with filtering | Easy |
| [Ransom Note](https://leetcode.com/problems/ransom-note/) | Frequency count with subtraction | Easy |
| [Sort Characters By Frequency](https://leetcode.com/problems/sort-characters-by-frequency/) | Frequency counting + sorting | Medium |

---

*Last updated: 2025 — for data engineering interview preparation.*