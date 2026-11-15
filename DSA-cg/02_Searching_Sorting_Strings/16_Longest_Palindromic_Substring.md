# Longest Palindromic Substring

**Difficulty:** Medium  
**Topics:** Expand around center, dynamic programming

## 1. Problem statement

Return the longest contiguous substring that is a palindrome. If multiple answers have equal maximum length, returning any one is acceptable unless the contract specifies tie-breaking.

## 2. Approaches

### Brute force

Enumerate every substring and check whether it is a palindrome.

```python
def longest_palindrome_brute(s):
    best = ""
    for left in range(len(s)):
        for right in range(left + 1, len(s) + 1):
            candidate = s[left:right]
            if candidate == candidate[::-1] and len(candidate) > len(best):
                best = candidate
    return best
```
O(n³) time due to O(n²) substrings and O(n) checking/copying; O(n) auxiliary space per candidate.

### Optimized: expand around each center

```python
def longest_palindrome(s):
    if not s:
        return ""
    best_left = best_right = 0

    def expand(left, right):
        while left >= 0 and right < len(s) and s[left] == s[right]:
            left -= 1
            right += 1
        return left + 1, right - 1

    for center in range(len(s)):
        l1, r1 = expand(center, center)       # odd length
        l2, r2 = expand(center, center + 1)   # even length
        if r1 - l1 > best_right - best_left:
            best_left, best_right = l1, r1
        if r2 - l2 > best_right - best_left:
            best_left, best_right = l2, r2
    return s[best_left:best_right + 1]
```
O(n²) time, O(1) auxiliary space excluding returned substring.

## 3. Dry run

For `"babad"`:
- Center at `a` (index 1) expands to `"bab"` (indices 0..2).
- Center at `b` (index 2) expands to `"aba"` (indices 1..3).
- Both have length 3. This implementation keeps the first maximum, returning `"bab"`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why do you test two centers per index?

**Candidate:** Palindromes can have odd length (`aba`) or even length (`abba`), so centers are either a character or the gap between adjacent characters.

**Interviewer:** Can this be linear?

**Candidate:** Manacher's algorithm achieves O(n), but it is more complex. I would use it when input size or repeated queries justify the complexity.

## 5. Complexity and mistakes

O(n²) time, O(1) auxiliary space. Common mistakes: forgetting even-length centers, off-by-one errors after expansion, and confusing substring (contiguous) with subsequence.

## 6. Tests and variations

```python
assert longest_palindrome("") == ""
assert longest_palindrome("a") == "a"
assert longest_palindrome("babad") in {"bab", "aba"}
assert longest_palindrome("cbbd") == "bb"
assert longest_palindrome("aaaa") == "aaaa"
```
Follow-ups: count all palindromic substrings; return all maximum palindromes; implement Manacher's algorithm; handle normalized Unicode text.
