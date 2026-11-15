# Coin Change II — Number of Combinations

**Difficulty:** Medium | **Pattern:** Unbounded knapsack, loop-order semantics

## Problem

Count combinations that make amount using unlimited coins. Different orders of the same coin multiset count as one combination.

## Approaches

Brute force recursively explores include/exclude choices and repeats many states.

Optimized 1D DP:

```python
def coin_change_combinations(amount, coins):
    if amount < 0 or any(c <= 0 for c in coins):
        raise ValueError("amount must be non-negative and coins positive")
    dp = [0] * (amount + 1)
    dp[0] = 1

    for coin in coins:
        for value in range(coin, amount + 1):
            dp[value] += dp[value - coin]
    return dp[amount]
```

O(amount × number_of_coins) time, O(amount) space.

## Dry run

Coins `[1,2,5]`, amount 5:
- Start dp `[1,0,0,0,0,0]`.
- Process coin 1: every amount becomes 1.
- Process coin 2: add combinations ending with another 2; dp[5] becomes 3.
- Process coin 5: dp[5] becomes 4.
Combinations: 1+1+1+1+1, 1+1+1+2, 1+2+2, 5.

## Why loop order matters

Coin outer / amount ascending counts combinations without counting permutations separately. Amount outer / coin inner generally counts ordered sequences, a different problem.

## Interviewer–candidate discussion

**Interviewer:** Why initialize dp[0] to 1 rather than 0?

**Candidate:** There is one way to make amount zero: choose no coins. This seed lets each denomination extend valid combinations.

**Interviewer:** Why iterate amounts upward?

**Candidate:** Unlimited reuse is allowed, so the current coin can contribute multiple times within the same pass.

## Edge cases and mistakes

Amount zero gives one combination; no coins gives zero for positive amount; ensure positive denominations; do not change loop order without understanding whether order matters.

## Tests

```python
assert coin_change_combinations(5, [1, 2, 5]) == 4
assert coin_change_combinations(3, [2]) == 0
assert coin_change_combinations(0, [1, 2]) == 1
assert coin_change_combinations(10, []) == 0
```

Follow-ups: ordered combinations; bounded denominations; reconstruct combinations; compare with minimum-coins DP.
