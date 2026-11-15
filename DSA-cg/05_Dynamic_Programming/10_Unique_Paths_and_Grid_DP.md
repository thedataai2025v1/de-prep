# Unique Paths and Grid Dynamic Programming

**Difficulty:** Medium | **Pattern:** 2D grid DP

## Problem

A robot starts at the top-left of an m×n grid and can move only right or down. Count paths to the bottom-right.

## Approaches

Brute force recursively branches right/down, producing exponential work.

DP state `dp[r][c]` = number of paths to cell `(r,c)`. Each non-origin cell receives paths from above and left.

```python
def unique_paths(m, n):
    if m <= 0 or n <= 0:
        return 0
    dp = [1] * n
    for _ in range(1, m):
        for c in range(1, n):
            dp[c] += dp[c - 1]
    return dp[-1]
```

O(mn) time and O(n) space.

## Dry run

For a 3×3 grid:
- First row is all 1 because only right moves are possible.
- Second row evolves `[1,2,3]`.
- Third row evolves `[1,3,6]`.
Answer 6.

## Interviewer–candidate discussion

**Interviewer:** Why does each cell sum top and left?

**Candidate:** Those are exactly the possible predecessor cells under the allowed moves, and their path sets are disjoint by final move.

**Interviewer:** What changes with obstacles?

**Candidate:** Initialize obstacle cells to zero and only add top/left counts for open cells.

## Edge cases and mistakes

Single row/column; non-positive dimensions; obstacle at start or destination; blocked cells; movement rules changed. Clarify whether dimensions represent rows then columns.

## Tests

```python
assert unique_paths(3, 7) == 28
assert unique_paths(3, 3) == 6
assert unique_paths(1, 5) == 1
assert unique_paths(0, 3) == 0
```

Follow-ups: unique paths with obstacles; minimum path sum; paths with diagonal moves; count paths modulo a number.
