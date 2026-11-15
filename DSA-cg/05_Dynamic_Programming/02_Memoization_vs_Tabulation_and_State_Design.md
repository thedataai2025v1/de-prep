# Memoization vs Tabulation and DP State Design

**Difficulty:** Medium | **Pattern:** State modeling and recurrence design

## Concept

- **Memoization (top-down):** recursive function plus cache. Compute only requested states.
- **Tabulation (bottom-up):** fill a table in dependency order. Usually no recursion.
- **State:** the smallest information needed to determine the answer to a subproblem.

A state that omits relevant information produces incorrect caching. For example, a path problem may need `(row, col)`; a constrained path may also need remaining budget or previous choice.

## Example: minimum cost to reach the end

Given `cost[i]`, pay the cost of the step you land on; start at step 0 or 1, and reach beyond the last index.

```python
def min_cost_climbing_stairs(cost):
    n = len(cost)
    if n <= 1:
        return 0
    dp = [0] * (n + 1)
    for i in range(2, n + 1):
        dp[i] = min(dp[i - 1] + cost[i - 1],
                    dp[i - 2] + cost[i - 2])
    return dp[n]
```

Recurrence: `dp[i] = min(dp[i-1]+cost[i-1], dp[i-2]+cost[i-2])`, with `dp[0]=dp[1]=0`.

Rolling-space version:

```python
def min_cost_climbing_stairs_o1(cost):
    one = two = 0
    for i in range(2, len(cost) + 1):
        one, two = min(two + cost[i - 1], one + cost[i - 2]), one
    return one
```

## Dry run

For `[10,15,20]`: `dp[0]=dp[1]=0`; `dp[2]=min(15,10)=10`; `dp[3]=min(10+20,0+15)=15`. Answer 15.

## Complexity

Both versions O(n) time. Table version O(n) space; rolling version O(1).

## Interviewer–candidate discussion

**Interviewer:** How do you know what the state should contain?

**Candidate:** It must distinguish subproblems with different future possibilities, but should not retain irrelevant history. I test the proposed state by asking whether two paths with the same state always have identical remaining decisions and costs.

**Interviewer:** Why can rolling arrays be valid?

**Candidate:** If the recurrence only depends on a fixed number of previous states, older table entries are never needed again.

## Mistakes and edge cases

Write down what `dp[i]` means; distinguish “cost to reach i” from “cost starting at i”; test short arrays; do not compress memory before confirming dependencies.

## Tests

```python
assert min_cost_climbing_stairs([10, 15, 20]) == 15
assert min_cost_climbing_stairs([1, 100, 1, 1, 1, 100, 1, 1, 100, 1]) == 6
assert min_cost_climbing_stairs([]) == 0
```

Exercises: convert a memoized recurrence to tabulation; draw a dependency graph; identify when state compression is safe.
