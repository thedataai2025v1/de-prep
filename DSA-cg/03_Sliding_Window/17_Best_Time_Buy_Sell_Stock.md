# Best Time to Buy and Sell Stock

**Difficulty:** Easy  
**Topics:** Array, Greedy, One-pass Minimum (often taught with sliding window)  
**Study-plan section:** Sliding Window  
**Problem ID:** 17

## 1. Problem Statement

Given daily stock prices, choose one day to buy and a later day to sell to maximize profit. If no positive profit is possible, return 0.

**Example:**
```text
prices = [7,1,5,3,6,4]
Output = 5 (buy at 1, sell at 6)
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Try every buy day and every later sell day, retaining the maximum difference.

```python
def max_profit(prices):
    best = 0
    for i in range(len(prices)):
        for j in range(i+1, len(prices)):
            best = max(best, prices[j]-prices[i])
    return best
```

- **Complexity:** O(n²) time; O(1) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Maintain the minimum price seen so far and calculate the profit from selling today.

```python
def max_profit(prices):
    min_price = float('inf')
    best = 0
    for price in prices:
        min_price = min(min_price, price)
        best = max(best, price-min_price)
    return best
```

- **Complexity:** O(n) time; O(1) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Maintain the minimum price seen so far and calculate the profit from selling today.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Scan [7,1,5,3,6,4]. The minimum becomes 1. Selling at 5 gives profit 4; selling at 6 gives profit 5, the maximum. The buy day always precedes the sell day.

## 4. Interview Conversation

**Interviewer:** How do you enforce buying before selling?

**Candidate:** I only calculate profit against the minimum price seen earlier in the scan.

**Interviewer:** Why not use a classic fixed-size sliding window?

**Candidate:** This is more accurately a one-pass prefix-minimum greedy pattern. It is often grouped near sliding-window questions, but there is no fixed window.

**Interviewer:** What if prices always decrease?

**Candidate:** The best remains zero because the problem allows choosing not to trade.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Return buy/sell indices by storing the day when min_price was updated and saving the best pair. Multiple transactions is a different problem requiring a different strategy.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Empty/single-day arrays return 0. Profit must be non-negative. Same-day buy/sell is not useful.

Additional checks:
- Minimum valid input size.
- Maximum input size and performance.
- Duplicate values / repeated characters / repeated edges where applicable.
- Boundary values and absent target/result.

## 7. Common Interview Mistakes

- Coding before confirming constraints and output semantics.
- Explaining complexity without stating what `n`, `m`, `k` or other variables represent.
- Ignoring empty inputs, duplicates, boundary conditions or mutation requirements.
- Claiming a hash operation is worst-case O(1); it is generally expected/average O(1).
- Giving an optimized algorithm without explaining its invariant or correctness argument.

## 8. Data Engineering Connection

An online scan can compute best margin from a running minimum; analogous to monitoring minimum cost and current value in time-ordered metrics.

## 9. Final Interview Summary

**One-sentence explanation:** Maintain the minimum price seen so far and calculate the profit from selling today.

**Complexity to say aloud:** O(n) time; O(1) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #17 in the supplied study plan.*