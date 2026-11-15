# 0/1 Knapsack

**Difficulty:** Medium–Hard | **Pattern:** 2D/1D DP, each item used at most once

## Problem

Each item has a weight and value. Choose items without exceeding capacity, maximizing total value. Each item can be selected zero or one time.

## Approaches

Brute force considers include/exclude for each item: O(2ⁿ).

2D recurrence:
`dp[i][c] = max(dp[i-1][c], value[i-1] + dp[i-1][c-weight[i-1]])` when the item fits.

```python
def knapsack_01(weights, values, capacity):
    if len(weights) != len(values) or capacity < 0:
        raise ValueError("invalid input")
    if any(w < 0 for w in weights):
        raise ValueError("weights must be non-negative")

    dp = [0] * (capacity + 1)
    for weight, value in zip(weights, values):
        for cap in range(capacity, weight - 1, -1):
            dp[cap] = max(dp[cap], value + dp[cap - weight])
    return dp[capacity]
```

O(n × capacity) time, O(capacity) space. Descending capacity is essential for 0/1 semantics.

## Dry run

Weights `[2,3,4]`, values `[4,5,6]`, capacity 5:
- Item (2,4): best at capacity 2..5 becomes 4.
- Item (3,5): capacity 5 can become 9 (2+3).
- Item (4,6): cannot improve capacity 5 beyond 9.
Answer 9.

## Interviewer–candidate discussion

**Interviewer:** Why iterate capacity downward?

**Candidate:** It prevents the current item from being reused in the same iteration. Ascending order would turn this into unbounded knapsack.

**Interviewer:** How do you recover selected items?

**Candidate:** Keep the full 2D table and backtrack from `dp[n][capacity]`, comparing whether each item changed the optimum.

## Edge cases and mistakes

Capacity zero; empty items; zero-weight items; negative values; mismatched arrays. If zero-weight positive-value items are allowed, descending iteration still adds each once in this implementation, but validate the intended contract.

## Tests

```python
assert knapsack_01([2, 3, 4], [4, 5, 6], 5) == 9
assert knapsack_01([], [], 10) == 0
assert knapsack_01([1], [10], 0) == 0
assert knapsack_01([1, 2], [2, 4], 2) == 4
```

Follow-ups: unbounded knapsack; bounded knapsack; return chosen items; multi-dimensional capacity; explain pseudo-polynomial complexity.
