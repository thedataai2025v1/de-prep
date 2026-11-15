# Palindrome Number

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Math, String |
| **Companies** | Amazon, Google, Meta, Microsoft |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an integer `x`, return `true` if `x` is a palindrome, and `false` otherwise.

An integer is a palindrome when it reads the same backward as forward. Negative numbers are not palindromes.

**Example:**
```
Input:  x = 121
Output: true

Input:  x = -121
Output: false

Input:  x = 10
Output: false
```

---

## 2. Approaches

### Approach 1: Reverse Half of Number (⭐ Optimal / Interview Choice)

**Idea:** Avoid integer overflow by only reversing half of the number. Compare the first half with the reversed second half.

**Pros:**
- Optimal O(log n) time — only processes half the digits
- O(1) space — no extra data structures or string allocations
- Prevents integer overflow in languages with fixed-size integers
- Demonstrates mathematical thinking, which interviewers value

**Cons:**
- More complex logic than the string approach — harder to recall under pressure
- Requires careful handling of odd-length palindromes (middle digit check)
- Not immediately obvious why reversing half works — needs clear explanation
- Edge case handling (trailing zeros, negatives) adds code branches

```python
def isPalindrome(x: int) -> bool:
    # Negative numbers and numbers ending in 0 (except 0 itself) are not palindromes
    if x < 0 or (x % 10 == 0 and x != 0):
        return False

    reversed_half = 0
    while x > reversed_half:
        reversed_half = reversed_half * 10 + x % 10
        x //= 10

    # When the number is odd, remove the middle digit from reversed_half
    return x == reversed_half or x == reversed_half // 10
```

| Metric | Value |
|---|---|
| **Time** | O(log n) — number of digits in x |
| **Space** | O(1) — only integer variables |

---

### Approach 2: Convert to String (Alternative)

**Idea:** Convert the integer to a string and check if it reads the same forwards and backwards.

**Pros:**
- Extremely concise — 2 lines of code
- Very easy to understand and explain
- Works for any input type that can be stringified
- Less prone to off-by-one or edge case bugs

**Cons:**
- O(n) extra space for the string representation
- O(n) time to create the string (not strictly optimal)
- Interviewers may ask "can you do it without conversion?"
- Does not demonstrate mathematical problem-solving skills

```python
def isPalindrome(x: int) -> bool:
    s = str(x)
    return s == s[::-1]
```

| Metric | Value |
|---|---|
| **Time** | O(n) — where n is the number of digits |
| **Space** | O(n) — string representation of the number |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| Negative number `-121` | Return `False` — sign breaks symmetry |
| Single digit `7` | Return `True` — trivially a palindrome |
| Number ending in 0 like `10` | Return `False` — reversed would start with 0 |
| Zero `0` | Return `True` — single digit palindrome |
| Large number near max int | Approach 1 avoids overflow by reversing only half |

---

## 4. Interview Talking Points

- "Reversing only half of the number is clever: it prevents overflow in languages with fixed-size integers, and demonstrates mathematical thinking over brute force."
- "In data engineering, palindrome checks map to data quality validation patterns: verifying checksums, validating symmetric identifiers in ETL pipelines, or detecting mirrored corruption in transaction logs."

---

## 5. Interview Follow-Up Questions

> **Be prepared for these — they differentiate junior from senior candidates.**

**Q1. Could you solve it without converting the integer to a string?**
→ Yes — the optimal approach uses modular arithmetic to reverse half the number in-place with O(1) space.

**Q2. Why do we reverse only half the number instead of the whole thing?**
→ Reversing the full number can cause integer overflow in languages with fixed-size integers (e.g., C++, Java). Reversing only half avoids this while still enabling the comparison.

**Q3. How does the algorithm handle odd-length palindromes like 121?**
→ When the loop ends, `x = 1` and `reversed_half = 12`. We check `x == reversed_half // 10` (1 == 1) to account for the middle digit being extra in reversed_half.

**Q4. Why do numbers ending in 0 (except 0 itself) fail early?**
→ If a number ends in 0, its reverse would start with 0 — but leading zeros don't exist in integers. So `10` reversed is `01 = 1`, not `10`. The only exception is `0` itself.

**Q5. Which approach would you choose in an interview?**
→ Start with the string approach for clarity, then offer the mathematical half-reversal to show depth. The interviewer usually expects you to discuss trade-offs between both.

---

## 6. Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Reversing the full number | Causes integer overflow for large inputs | Reverse only half, then compare |
| Not handling trailing zeros | `10` reversed as `01` still gives `False`, but logic must be explicit | Check `x % 10 == 0 and x != 0` early |
| Not checking negatives | `-121` reversed is `121-` which may not be caught | Add `x < 0` guard at the start |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Reverse Integer](https://leetcode.com/problems/reverse-integer/) | Core technique: digit-by-digit reversal |
| [Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list/) | Same concept on a different data structure |
| [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Extended to strings with alphanumeric filtering |

---

*Last updated: 2025 — for data engineering interview preparation.*