# Group Anagrams

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, String, Hash Table, Sorting |
| **Companies** | Google, Facebook, Amazon, Microsoft, Uber |
| **Interview Frequency** | High (Top 100 Must-Know Problems) |

---

## 1. Problem Statement

Given an array of strings, **group all anagrams together**. The output can be returned in any order.

Two strings are **anagrams** if they contain the same characters with the same frequencies.

**Example:**

```
Input:  strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
Output: [["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]
```

**Constraints (LeetCode):**
- `1 <= strs.length <= 10^4`
- `0 <= strs[i].length <= 100`
- `strs[i]` consists of lowercase English letters.

---

## 2. Approaches

### Approach 1: Sorting (Canonical Key)

**Idea:** Sort each string alphabetically. Anagrams will produce the same sorted string. Use the sorted string as a dictionary key and group the original strings.

```python
from collections import defaultdict

def groupAnagrams(strs):
    groups = defaultdict(list)

    for word in strs:
        key = ''.join(sorted(word))
        groups[key].append(word)

    return list(groups.values())
```

| Metric | Value |
|---|---|
| **Time** | O(n × k log k) |
| **Space** | O(n × k) (output + keys) |

- `n` = number of strings
- `k` = maximum string length

**Pros:**
- Simple and easy to implement
- Easy to understand and explain
- Works for arbitrary character sets (Unicode, mixed case, etc.)
- Uses a consistent canonical key

**Cons:**
- Sorting every string adds computational overhead
- Less efficient than character frequency counting for the given constraints

---

### Approach 2: Character Frequency (⭐ Optimal / Interview Choice)

**Idea:** Since strings contain only lowercase English letters, create a frequency array of size 26 for each string. Anagrams will have identical frequency arrays. Use the frequency tuple as the dictionary key.

```python
from collections import defaultdict

def groupAnagrams(strs):
    groups = defaultdict(list)

    for word in strs:
        count = [0] * 26

        for char in word:
            count[ord(char) - ord('a')] += 1

        groups[tuple(count)].append(word)

    return list(groups.values())
```

| Metric | Value |
|---|---|
| **Time** | O(n × k) — **optimal** for fixed alphabet |
| **Space** | O(n × k) (output + dictionary keys) |

**Pros:**
- Optimal asymptotic time complexity for the given alphabet
- Avoids sorting each string
- Frequency tuple uniquely identifies the character composition
- Efficient for large collections of strings

**Cons:**
- Depends on a fixed character set (lowercase English letters only)
- Requires character-to-index conversion (`ord(char) - ord('a')`)
- Slightly more complex than sorting
- Each dictionary key contains 26 frequency values

---

### Approach 3: Using Python `Counter`

**Idea:** Count character frequencies using `Counter` and convert the frequency mapping into a sorted tuple to use as a dictionary key.

```python
from collections import defaultdict, Counter

def groupAnagrams(strs):
    groups = defaultdict(list)

    for word in strs:
        key = tuple(sorted(Counter(word).items()))
        groups[key].append(word)

    return list(groups.values())
```

| Metric | Value |
|---|---|
| **Time** | O(n × k log k) worst case for key construction |
| **Space** | O(n × k) (output + keys) |

**Pros:**
- Concise and readable
- Automatically handles character frequencies
- Supports arbitrary characters
- Avoids manually creating a frequency array

**Cons:**
- Sorting frequency entries adds overhead
- More memory overhead than a fixed-size array
- Not as efficient as Approach 2 for lowercase English letters

---

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Sorting** | O(n × k log k) | O(n × k) | Unicode / mixed character sets |
| **2. Frequency Array** ⭐ | O(n × k) | O(n × k) | **Interview choice** (lowercase only) |
| **3. Counter** | O(n × k log k) | O(n × k) | Quick prototype / readability |

---

## 3. Edge Cases

> **Always mention these in an interview to show thoroughness.**

| Edge Case | Input | Expected Behavior |
|---|---|---|
| Empty list | `[]` | Return `[]` |
| Single element | `["a"]` | Return `[["a"]]` |
| Empty strings | `["", ""]` | Return `[["", ""]]` (both are anagrams of empty string) |
| All unique strings | `["abc", "def"]` | Each in its own group |
| All same anagram | `["ate", "eat", "tea"]` | All in one group |
| Mixed case (not in constraints) | `["Ab", "aB"]` | Depends on case-sensitivity requirement |
| Very long strings | `"a" * 100` | Should handle without timeout |

---

## 4. Interview Follow-Up Questions

> **Be prepared for these — they differentiate junior from senior candidates.**

**Q1. Why use a tuple instead of a list as a dictionary key?**
→ Lists are mutable and cannot be used as dictionary keys. Tuples are immutable and hashable.

**Q2. Why does sorting group anagrams correctly?**
→ Anagrams contain identical characters with identical frequencies. Sorting produces the same character sequence for all anagrams.

**Q3. What if the input contains Unicode characters?**
→ Use the sorting approach or a character-frequency dictionary with an immutable tuple key. Avoid the fixed-size 26-element array.

**Q4. What if there are millions of strings?**
→ Consider partitioning strings by their canonical keys and processing partitions in parallel. For distributed processing, use Spark `groupByKey` or an appropriate aggregation strategy, while considering shuffle and memory costs.

**Q5. Which approach would you choose in an interview?**
→ Choose **Approach 2 (Character Frequency)** because the constraints specify lowercase English letters. It provides **O(n × k)** time complexity.

---

## 5. What to Say in an Interview

> **This is the template to follow during the actual interview.**

### Step-by-step approach:

1. **Clarify constraints:**
   - "Are the strings all lowercase English letters? What about Unicode or mixed case?"
   - "What's the expected range of input size?"

2. **Walk through the example:**
   - `"eat" → sorted → "aet"`, `"tea" → sorted → "aet"` → same group
   - Explain why this works conceptually

3. **Propose Approach 1 (Sorting) first** as a natural starting point:
   - Write the code, analyze time/space
   - This shows you can get to a working solution quickly

4. **Optimize to Approach 2 (Frequency Array):**
   - "Since we know the input is lowercase English only, we can do better in O(n × k) by avoiding sorting."
   - Explain the 26-element frequency array
   - Write the code, compare complexity

5. **Mention edge cases** proactively (empty input, empty strings)

6. **Offer trade-offs:** "If the character set were Unicode, I'd fall back to sorting or a Counter-based approach."

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using a list as dict key | Lists are mutable → unhashable | Use `tuple(count)` |
| Forgetting to handle empty strings | `""` is a valid anagram of `""` | Code naturally handles it, but verify |
| Confusing anagram with substring | `"abc"` is NOT an anagram of `"abcd"` | Same length + same character frequencies |
| Case sensitivity issues | `"Ab"` ≠ `"aB"` by default | Confirm requirements; use `.lower()` if needed |
| Not importing `defaultdict` | `KeyError` on first access | `from collections import defaultdict` |

---

## 7. Related Problems

| Problem | Key Concept | Difficulty |
|---|---|---|
| [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Check if two strings are anagrams | Easy |
| [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Hash table + frequency counting | Medium |
| [Group Shifted Strings](https://leetcode.com/problems/group-shifted-strings/) | Canonical key with relative difference | Medium |
| [Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix/) | String grouping / prefix matching | Easy |
| [Isomorphic Strings](https://leetcode.com/problems/isomorphic-strings/) | Character mapping with hash tables | Easy |

---

*Last updated: 2025-01 — formatted for interview preparation.*
