# Best Time to Buy and Sell Stock

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Array, Dynamic Programming |
| **Companies** | Amazon, Google, Meta, Microsoft, Apple, Bloomberg |
| **Interview Frequency** | High |

---

## 1. Problem Statement

You are given an array `prices` where `prices[i]` is the price of a given stock on the `i`th day.

You want to maximize your profit by choosing a single day to buy one stock and choosing a different day in the future to sell that stock. Return the maximum profit you can achieve. If you cannot achieve any profit, return `0`.

**Example:**
```
Input:  prices = [7,1,5,3,6,4]
Output: 5 (buy on day 2 at price 1, sell on day 5 at price 6)

Input:  prices = [7,6,4,3,1]
Output: 0 (no profitable transaction possible)
```

---

## 2. Approaches

### Approach 1: Track Min Price and Max Profit in One Pass (⭐ Optimal / Interview Choice)

**Idea:** Maintain the minimum price seen so far and calculate the potential profit at each step. Update the maximum profit whenever a better one is found.

**Pros:**
- Optimal O(n) time with a single pass through the array
- O(1) space — only two scalar variables needed
- Elegant greedy approach: at each step we make the locally optimal choice
- Easy to extend to variants (e.g., with cooldown or transaction fees)

**Cons:**
- Only works for a single buy-sell transaction
- Greedy logic is non-obvious — requires understanding that tracking running minimum is sufficient
- Doesn't tell you which days to buy/sell (just the profit amount)
- May be hard to derive on the fly without recognizing the pattern

```python
def maxProfit(prices: list[int]) -> int:
    min_price = float('inf')
    max_profit = 0

    for price in prices:
        if price < min_price:
            min_price = price
        elif price - min_price > max_profit:
            max_profit = price - min_price

    return max_profit
```

| Metric | Value |
|---|---|
| **Time** | O(n) — single pass through prices |
| **Space** | O(1) — two scalar variables |

---

### Approach 2: Brute Force All Pairs (Alternative)

**Idea:** Check every possible (buy, sell) pair and find the maximum difference. Simple but inefficient.

**Pros:**
- Trivially correct — exhaustively checks every possibility
- Easy to implement and explain — two nested loops
- Good starting point to discuss before optimizing

**Cons:**
- O(n²) time — far too slow for large inputs (LeetCode will time out)
- No early termination — always checks all pairs even after finding a large profit
- Demonstrates no optimization insight — a red flag in interviews
- Doesn't scale beyond small datasets

```python
def maxProfit(prices: list[int]) -> int:
    max_profit = 0
    n = len(prices)

    for i in range(n):
        for j in range(i + 1, n):
            profit = prices[j] - prices[i]
            if profit > max_profit:
                max_profit = profit

    return max_profit
```

| Metric | Value |
|---|---|
| **Time** | O(n²) — all pairs of days |
| **Space** | O(1) — no extra data structures |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| Decreasing prices `[7,6,4,3,1]` | Return 0 — no profitable buy/sell |
| Single day `[5]` | Return 0 — cannot buy and sell on same day |
| All same prices `[3,3,3,3]` | Return 0 — no profit possible |
| Two days `[1,100]` | Return 99 — buy day 1, sell day 2 |

---

## 4. Interview Talking Points

- "This is a classic dynamic programming optimization: at each step we maintain a running state (minimum price seen) and derive the answer (maximum profit) incrementally."
- "In data engineering, this pattern maps to time-series optimization: detecting peaks in monitoring data, finding optimal batch windows in stream processing, or identifying the best interval for triggering downstream pipelines based on cost/profit signals."

---

## 5. Interview Follow-Up Questions

> **Be prepared for these — they differentiate junior from senior candidates.**

**Q1. What if you can make at most two transactions?**
→ Use a 4-state DP approach: track the max profit after first buy, first sell, second buy, and second sell. Runs in O(n) time with O(1) space.

**Q2. What if there is a transaction fee on each trade?**
→ Subtract the fee when selling: `max_profit = max(max_profit, price - min_price - fee)`. The core algorithm stays the same.

**Q3. How would you find the actual buy and sell days, not just the profit?**
→ Track the `buy_day` index when updating `min_price`, and the `sell_day` index when updating `max_profit`. Return a tuple `(buy_day, sell_day, max_profit)`.

**Q4. Why use `elif` instead of two separate `if` statements?**
→ If `price < min_price`, we update `min_price` and there's no point computing profit (it would be negative or zero). The `elif` avoids unnecessary computation.

**Q5. What if prices can be negative (short selling)?**
→ The problem changes significantly: you'd need to consider both long (buy low, sell high) and short (sell high, buy low) positions. This becomes a more advanced variant.

---

## 6. Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Comparing max and min without ordering | Sells before buying (e.g., max appears before min) | Always track `min_price` first, then compute profit |
| Returning `max(prices) - min(prices)` | May sell before buying (min after max) | Track minimum *as we iterate* to ensure buy comes before sell |
| Not initializing `max_profit = 0` | May return negative value for all-decreasing arrays | Return 0 when no profit is possible |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/) | Unlimited transactions — buy on every upswing |
| [Best Time to Buy and Sell Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/) | At most two transactions — 4-state DP |
| [Contains Duplicate](./18-Contains_Duplicate.md) | Array traversal pattern; same one-pass mindset |

---

*Last updated: 2025 — for data engineering interview preparation.*