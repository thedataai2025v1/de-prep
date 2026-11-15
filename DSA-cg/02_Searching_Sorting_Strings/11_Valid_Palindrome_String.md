# Valid Palindrome — String

**Difficulty:** Easy  
**Topics:** Two pointers, normalization

## 1. Problem statement

Determine whether a string reads the same forward and backward after ignoring non-alphanumeric characters and case. This contract uses Unicode-aware `isalnum()` and `lower()`.

## 2. Approaches

### Brute force: normalize and reverse

```python
def is_palindrome_brute(s):
    cleaned = "".join(ch.lower() for ch in s if ch.isalnum())
    return cleaned == cleaned[::-1]
```
O(n) time and O(n) space.

### Optimized: two pointers

```python
def is_palindrome(s):
    left, right = 0, len(s) - 1
    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        if s[left].lower() != s[right].lower():
            return False
        left += 1
        right -= 1
    return True
```
O(n) time and O(1) auxiliary space.

## 3. Dry run

`"A man, a plan, a canal: Panama"`: skip spaces and punctuation; compare A/a, m/m, a/a, n/n, and so on. Pointers meet without mismatch, so return True.

## 4. Interviewer–candidate discussion

**Interviewer:** Why not clean the whole string first?

**Candidate:** That is simpler but allocates O(n) additional memory. Two pointers skip ignored characters in place and use O(1) auxiliary space.

**Interviewer:** What does alphanumeric mean for Unicode?

**Candidate:** Python's `isalnum()` is Unicode-aware. If the specification is ASCII-only or requires Unicode normalization/case folding, I would make that explicit.

## 5. Edge cases and mistakes

Empty string and punctuation-only strings are palindromes under this definition. Handle one-character strings. Avoid moving pointers beyond bounds. Do not assume ASCII if the contract says Unicode.

## 6. Tests

```python
assert is_palindrome("A man, a plan, a canal: Panama")
assert not is_palindrome("race a car")
assert is_palindrome(" ")
assert is_palindrome(".,,")
assert is_palindrome("a")
```
Follow-ups: longest palindromic substring, palindrome permutation, streaming validation.
