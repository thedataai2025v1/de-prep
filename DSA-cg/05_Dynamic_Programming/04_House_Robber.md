# House Robber

**Difficulty:** Medium | **Pattern:** 1D DP, take-or-skip

## Problem

Each house has non-negative money. Adjacent houses cannot both be robbed. Maximize the total.

## Approaches

Brute force recursively chooses rob or skip at each index: O(2ⁿ).

Define `dp[i]` as maximum money from houses 0..i:
`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`.

```python
def house_robber(nums):
    prev_two = prev_one = 0
    for value in nums:
        prev_two, prev_one = prev_one, max(prev_one, prev_two + value)
    return prev_one
```

O(n) time, O(1) space.

## Dry run

For `[2,7,9,3,1]`:
- after 2: best 2
- after 7: best 7
- after 9: max(7,2+9)=11
- after 3: max(11,7+3)=11
- after 1: max(11,11+1)=12
Answer 12.

## Interviewer–candidate discussion

**Interviewer:** Why do two previous states suffice?

**Candidate:** Taking the current house excludes the immediately previous one, so I need the best result through i−2; skipping uses i−1.

**Interviewer:** How would you return the chosen houses?

**Candidate:** Keep a DP table or reconstruct decisions by comparing take and skip values while walking backward.

## Edge cases and mistakes

Empty list returns 0; one house; zeros; negative values require clarifying whether robbing none is allowed. Circular street changes the problem into max of ranges excluding first or last.

## Tests

```python
assert house_robber([]) == 0
assert house_robber([5]) == 5
assert house_robber([2, 7, 9, 3, 1]) == 12
assert house_robber([2, 1, 1, 2]) == 4
```

Follow-ups: House Robber II (circular); House Robber III (tree DP); return indices; add a cooldown or distance-k exclusion.
