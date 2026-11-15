# Coin Change — Minimum Number of Coins

**Difficulty:** Medium | **Pattern:** Unbounded knapsack, 1D DP

## Problem

Given positive coin denominations and an amount, return the minimum number of coins needed, or -1 if impossible. Each denomination can be used unlimited times.

## Approaches

Brute force tries every coin recursively, causing repeated subproblems and exponential work.

DP state: `dp[a]` = minimum coins to make amount a. Initialize `dp[0]=0`; all other states are infinity.

```python
def coin_change(coins, amount):
    if amount < 0 or any(c <= 0 for c in coins):
        raise ValueError("amount must be non-negative and coins positive")
    inf = amount + 1
    dp = [inf] * (amount + 1)
    dp[0] = 0

    for value in range(1, amount + 1):
        for coin in coins:
            if coin <= value:
                dp[value] = min(dp[value], dp[value - coin] + 1)

    return dp[amount] if dp[amount] != inf else -1
```

O(amount × number_of_coins) time, O(amount) space.

## Dry run

Coins `[1,2,5]`, amount 6:
- dp[0]=0.
- dp[1]=1, dp[2]=1 (coin 2), dp[3]=2, dp[4]=2, dp[5]=1, dp[6]=2 (5+1).
Answer 2.

## Interviewer–candidate discussion

**Interviewer:** Why does greedy fail?

**Candidate:** For coins `[1,3,4]` and amount 6, greedy picks 4+1+1 (3 coins), while optimal is 3+3 (2 coins).

**Interviewer:** Why is this unbounded knapsack?

**Candidate:** A denomination can be reused, so each amount state may transition from `a-coin` without excluding that coin for future use.

## Edge cases and mistakes

Amount zero returns zero; unreachable amount; duplicate denominations; zero/negative coin values; confusing minimum coin count with number of combinations.

## Tests

```python
assert coin_change([1, 2, 5], 11) == 3
assert coin_change([2], 3) == -1
assert coin_change([1], 0) == 0
assert coin_change([1, 3, 4], 6) == 2
```

Follow-ups: count combinations; count ordered sequences; reconstruct the coins used; bounded coin supply; prove when greedy is valid.
