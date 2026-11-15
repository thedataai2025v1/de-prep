# Longest Common Subsequence (LCS)

**Difficulty:** Medium–Hard | **Pattern:** 2D DP on two strings

## Problem

Find the length of the longest sequence appearing in both strings in the same relative order; characters need not be contiguous.

## Approaches

Brute force enumerates subsequences and checks membership, exponential in string length.

DP state `dp[i][j]` = LCS length for prefixes `a[:i]` and `b[:j]`.
- If characters match: `1 + dp[i-1][j-1]`.
- Otherwise: `max(dp[i-1][j], dp[i][j-1])`.

```python
def lcs_length(a, b):
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i - 1] == b[j - 1]:
                dp[i][j] = 1 + dp[i - 1][j - 1]
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
    return dp[m][n]
```

O(mn) time and O(mn) space.

## Dry run

`a="abcde"`, `b="ace"`:
- Match a/a → length 1.
- Match c/c after skipping b in the first string → length 2.
- Match e/e → length 3.
LCS is `"ace"` and length is 3.

## Interviewer–candidate discussion

**Interviewer:** How is subsequence different from substring?

**Candidate:** A subsequence preserves relative order but may skip characters; a substring must be contiguous.

**Interviewer:** Can space be reduced?

**Candidate:** Each row depends only on the previous row and the current row's left cell, so two rows suffice; one row is possible with a saved diagonal value.

## Edge cases and mistakes

Empty string; repeated characters; off-by-one indexing; using current row instead of previous row on mismatch; returning length when the interviewer asks for the actual sequence.

## Tests

```python
assert lcs_length("abcde", "ace") == 3
assert lcs_length("abc", "abc") == 3
assert lcs_length("abc", "def") == 0
assert lcs_length("", "abc") == 0
```

Follow-ups: reconstruct LCS; shortest common supersequence; diff tools; longest common substring; space-optimized implementation.
