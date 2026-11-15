# Koko Eating Bananas — Binary Search on the Answer

**Difficulty:** Medium  
**Topics:** Monotonic predicate, binary search on answer

## 1. Problem statement

Given pile sizes and `h` hours, find the minimum integer eating speed `k` such that all bananas can be eaten within `h` hours. Each hour, Koko chooses one pile and eats up to k bananas.

## 2. Approaches

### Brute force

Try every speed from 1 to the largest pile and return the first feasible speed.

```python
def min_eating_speed_brute(piles, h):
    if not piles:
        return 0
    for speed in range(1, max(piles) + 1):
        hours = sum((p + speed - 1) // speed for p in piles)
        if hours <= h:
            return speed
    return -1
```
O(n * max(piles)) time, O(1) auxiliary space.

### Optimized: binary search feasible speed

```python
def min_eating_speed(piles, h):
    if not piles:
        return 0
    if h < len(piles):
        return -1  # one non-empty pile requires at least one hour
    left, right = 1, max(piles)
    while left < right:
        speed = left + (right - left) // 2
        hours_needed = sum((p + speed - 1) // speed for p in piles)
        if hours_needed <= h:
            right = speed
        else:
            left = speed + 1
    return left
```
O(n log M) time, O(1) auxiliary space, where M is the largest pile.

## 3. Dry run

`piles=[3,6,7,11]`, `h=8`. Search speed between 1 and 11.
- At speed 6: hours = 1+1+2+2 = 6, feasible; search lower.
- At speed 3: hours = 1+2+3+4 = 10, too slow; search higher.
- At speed 4: hours = 1+2+2+3 = 8, feasible.
- The minimum feasible speed is 4.

## 4. Correctness intuition

If speed `k` is feasible, any larger speed is also feasible. Feasibility is monotonic (`False...False, True...True`), so binary search can find the first true value.

## 5. Interviewer–candidate discussion

**Interviewer:** Why use ceiling division?

**Candidate:** A pile of p bananas at speed k needs `ceil(p/k)` hours. Integer arithmetic `(p+k-1)//k` avoids floating-point precision issues.

**Interviewer:** What is the search range?

**Candidate:** Minimum 1, maximum `max(piles)`, which finishes every pile in one hour per pile.

## 6. Edge cases and mistakes

- Empty piles: define behavior (here returns 0).
- `h < number_of_piles`: impossible because each non-empty pile takes at least one hour.
- Very large values: use integer arithmetic.
- Zero/negative pile sizes: validate if input constraints do not guarantee positive piles.
- Searching for any feasible speed rather than the minimum feasible speed.

## 7. Tests and variations

```python
assert min_eating_speed([3, 6, 7, 11], 8) == 4
assert min_eating_speed([30, 11, 23, 4, 20], 5) == 30
assert min_eating_speed([30, 11, 23, 4, 20], 6) == 23
assert min_eating_speed([1, 2], 1) == -1
```
Variation: minimum capacity to ship packages within D days; minimum rate to complete jobs by a deadline; minimum machine count under a throughput constraint.

**Principal-level signal:** Identify the monotonic feasibility predicate and separate the cost of one predicate evaluation from the number of binary-search iterations.
