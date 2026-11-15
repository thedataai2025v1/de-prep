# String Manipulation Fundamentals

**Difficulty:** Foundation  
**Topics:** Immutability, counting, normalization, two pointers

## 1. Concept and when to use

Strings are immutable in Python. Repeated concatenation in a loop can create avoidable copying; collect fragments in a list and use `"".join(parts)`. Choose a representation based on the task: set for membership, dictionary/Counter for frequencies, two pointers for symmetric comparisons, stack for nested structure.

Clarify whether comparison is case-sensitive, whether whitespace/punctuation matter, and whether Unicode normalization is required.

## 2. Baseline and optimized examples

### Reverse a string

```python
def reverse_baseline(s):
    result = ""
    for ch in s:
        result = ch + result
    return result
```
Repeated concatenation may take O(n²) time due to copying.

```python
def reverse_string(s):
    return s[::-1]
```
O(n) time and O(n) output space.

### Character frequencies

```python
def frequencies(s):
    counts = {}
    for ch in s:
        counts[ch] = counts.get(ch, 0) + 1
    return counts
```
O(n) expected time and O(u) space for u distinct characters.

## 3. Dry run

For `"data"` the frequency map evolves: `d:1`, `a:1`, `t:1`, `a:2`, producing `{'d':1,'a':2,'t':1}`.

## 4. Interviewer–candidate discussion

**Interviewer:** Is a Python string an array of characters?

**Candidate:** It is an immutable sequence of Unicode code points. Indexing returns a one-character string, but a user-perceived grapheme can consist of multiple code points.

**Interviewer:** What about case-insensitive comparison?

**Candidate:** `casefold()` is more suitable than `lower()` for caseless matching. If canonical equivalence matters, normalize Unicode with `unicodedata.normalize`.

## 5. Complexity and pitfalls

A single pass is O(n); a frequency map uses O(u) space. Common mistakes: assuming ASCII, treating bytes and text interchangeably, mutating strings in place, ignoring normalization, and repeated immutable concatenation.

## 6. Edge cases and tests

```python
assert reverse_string("") == ""
assert reverse_string("a") == "a"
assert reverse_string("data") == "atad"
assert frequencies("aabb") == {"a": 2, "b": 2}
```

## 7. Follow-ups

1. Find the first non-repeating character.
2. Group anagrams using normalized or raw Unicode rules.
3. Stream text in chunks while maintaining frequency state.
4. Compare strings while ignoring punctuation and case.

**Data Engineering connection:** Text normalization and encoding contracts must be consistent across ingestion, indexing, deduplication and downstream joins.
