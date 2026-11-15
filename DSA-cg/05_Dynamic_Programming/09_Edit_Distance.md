# Edit Distance (Levenshtein Distance)

**Difficulty:** Hard | **Pattern:** 2D DP, string transformation

## Problem

Find the minimum number of insertions, deletions and substitutions needed to transform one string into another. Each operation costs one.

## Approaches

Brute force explores edit choices recursively and repeats many suffix pairs.

DP state `dp[i][j]` = minimum edits to transform first i characters of source into first j of target.
- Equal final characters: `dp[i-1][j-1]`.
- Otherwise: `1 + min(delete, insert, replace)`.

```python
def edit_distance(source, target):
    m, n = len(source), len(target)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if source[i - 1] == target[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # delete
                    dp[i][j - 1],      # insert
                    dp[i - 1][j - 1]   # replace
                )
    return dp[m][n]
```

O(mn) time and O(mn) space.

## Dry run

`"cat"` → `"cut"`:
- c matches c.
- a differs from u: replace once.
- t matches t.
Minimum edit distance = 1.

For `"horse"` → `"ros"`, one optimal sequence uses deletion/replacement operations; the DP returns 3.

## Interviewer–candidate discussion

**Interviewer:** Why initialize the first row and column?

**Candidate:** Transforming a nonempty prefix into an empty string requires deleting every character; transforming empty into a prefix requires inserting every character.

**Interviewer:** Can we reduce memory?

**Candidate:** Yes, use previous and current rows, giving O(min(m,n)) space after arranging the shorter string as the column dimension.

## Edge cases and mistakes

Empty source/target; identical strings; repeated characters; incorrect interpretation of insert vs delete; forgetting that replacement consumes one character from each string.

## Tests

```python
assert edit_distance("horse", "ros") == 3
assert edit_distance("intention", "execution") == 5
assert edit_distance("", "abc") == 3
assert edit_distance("same", "same") == 0
```

Follow-ups: weighted edits; transposition (Damerau–Levenshtein); return the actual edit script; thresholded/banded DP for approximate matching.
