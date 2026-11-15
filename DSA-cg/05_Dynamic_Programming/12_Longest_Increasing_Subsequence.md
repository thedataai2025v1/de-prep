# Longest Increasing Subsequence (LIS)

**Difficulty:** Medium–Hard | **Pattern:** DP, binary search optimization

## Problem

Find the length of the longest strictly increasing subsequence. Elements need not be contiguous.

## Approaches

### O(n²) DP

`dp[i]` is the LIS length ending at i.

```python
def lis_length_dp(nums):
    if not nums:
        return 0
    dp = [1] * len(nums)
    for i in range(len(nums)):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)
```

O(n²) time, O(n) space.

### O(n log n) tails method

```python
from bisect import bisect_left

def lis_length(nums):
    tails = []
    for value in nums:
        i = bisect_left(tails, value)
        if i == len(tails):
            tails.append(value)
        else:
            tails[i] = value
    return len(tails)
```

O(n log n) time, O(n) space. `tails[k]` is the smallest possible tail of an increasing subsequence of length k+1; it is not necessarily a subsequence itself.

## Dry run

For `[10,9,2,5,3,7,101,18]`, tails evolve:
`[10] → [9] → [2] → [2,5] → [2,3] → [2,3,7] → [2,3,7,101] → [2,3,7,18]`.
Length is 4.

## Interviewer–candidate discussion

**Interviewer:** Why replace an existing tail instead of appending?

**Candidate:** A smaller tail gives at least as much opportunity to extend a subsequence later, while preserving the best tail for that length.

**Interviewer:** How do you handle non-decreasing subsequences?

**Candidate:** Use `bisect_right` rather than `bisect_left`.

## Edge cases and mistakes

Empty input; duplicates; strictly decreasing; clarify strictly increasing vs non-decreasing. Do not claim the tails array is the actual LIS.

## Tests

```python
assert lis_length([10,9,2,5,3,7,101,18]) == 4
assert lis_length([]) == 0
assert lis_length([7,7,7]) == 1
assert lis_length([1,2,3,4]) == 4
```

Follow-ups: reconstruct one LIS; count LIS; longest bitonic subsequence; explain patience sorting connection.
