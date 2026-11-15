# Climbing Stairs

**Difficulty:** Easy–Medium | **Pattern:** 1D DP

## Problem

You can climb 1 or 2 steps at a time. Count distinct ways to reach step n.

## Approaches

Brute force recursion explores both choices and takes O(2ⁿ) time in the worst case.

Optimized recurrence: `ways(n) = ways(n-1) + ways(n-2)`, with `ways(0)=1`, `ways(1)=1`.

```python
def climb_stairs(n):
    if n < 0:
        return 0
    one_back = two_back = 1
    for _ in range(2, n + 1):
        one_back, two_back = one_back + two_back, one_back
    return one_back
```

Time O(n), space O(1). The values are Fibonacci-shifted.

## Dry run

n=4: start `(ways(1), ways(0))=(1,1)`. Compute ways(2)=2, ways(3)=3, ways(4)=5. The five sequences are 1+1+1+1, 1+1+2, 1+2+1, 2+1+1, 2+2.

## Interviewer–candidate discussion

**Interviewer:** Why is the recurrence additive?

**Candidate:** Every valid sequence ends with either a one-step move from n−1 or a two-step move from n−2; these cases are disjoint.

**Interviewer:** What if you can climb up to k steps?

**Candidate:** `dp[n]` becomes the sum of the previous k valid states, with a sliding sum or window to improve repeated summation.

## Edge cases and mistakes

n=0 has one empty sequence; negative n returns zero here; clarify whether the start position counts as a way. Avoid returning 0 for n=0 if using the standard combinatorial convention.

## Tests

```python
assert climb_stairs(0) == 1
assert climb_stairs(1) == 1
assert climb_stairs(2) == 2
assert climb_stairs(4) == 5
assert climb_stairs(5) == 8
```

Follow-ups: 1/2/3 steps; forbidden steps; minimum number of moves with variable jumps; large n with modular arithmetic.
