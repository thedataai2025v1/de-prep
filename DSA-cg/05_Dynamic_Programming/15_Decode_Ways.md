# Decode Ways

**Difficulty:** Medium | **Pattern:** 1D DP with string constraints

## Problem

Digits map 1→A through 26→Z. Count the number of ways to decode a digit string. A leading zero is invalid; `06` cannot be decoded as 6.

## Approaches

Brute force branches into one-digit and two-digit decoding and can be exponential.

Define `dp[i]` as ways to decode the first i characters:
- If current digit is nonzero, add `dp[i-1]`.
- If the last two digits form 10..26, add `dp[i-2]`.

```python
def num_decodings(s):
    if not s or s[0] == "0":
        return 0
    if not s.isdigit():
        raise ValueError("input must contain digits only")

    prev_two, prev_one = 1, 1
    for i in range(1, len(s)):
        current = 0
        if s[i] != "0":
            current += prev_one
        pair = int(s[i - 1:i + 1])
        if 10 <= pair <= 26:
            current += prev_two
        prev_two, prev_one = prev_one, current
    return prev_one
```

O(n) time, O(1) space.

## Dry run

For `"226"`:
- First digit 2: one way.
- At 22: single-digit path plus valid 22 pair → 2 ways.
- At 226: single 6 extends both previous ways; pair 26 adds the earlier count → 3 ways.
Decodings: B B F, B Z, V F.

## Interviewer–candidate discussion

**Interviewer:** Why is zero special?

**Candidate:** Zero cannot stand alone. It must be paired with the preceding 1 or 2, so invalid pairs can make the count zero.

**Interviewer:** What about `"100"`?

**Candidate:** 10 is valid but the final zero cannot stand alone or pair with 0, so the answer is zero.

## Edge cases and mistakes

Empty string convention; leading zero; internal zero; 06; 10; 20; 26; 27. Validate digits before converting pairs.

## Tests

```python
assert num_decodings("12") == 2
assert num_decodings("226") == 3
assert num_decodings("06") == 0
assert num_decodings("10") == 1
assert num_decodings("100") == 0
assert num_decodings("27") == 1
```

Follow-ups: wildcard `*`; return actual decodings; modulo large counts; streaming DP; discuss ambiguity in empty-input convention.
