# Partition Equal Subset Sum

**Difficulty:** Medium | **Pattern:** 0/1 knapsack feasibility, subset sum

## Problem

Determine whether a list of positive integers can be partitioned into two subsets with equal sums.

## Approaches

Brute force explores include/exclude assignments: O(2ⁿ).

If total sum is odd, answer is false. Otherwise ask whether a subset sums to `total//2`.

```python
def can_partition(nums):
    if any(value < 0 for value in nums):
        raise ValueError("this implementation expects non-negative values")
    total = sum(nums)
    if total % 2:
        return False

    target = total // 2
    possible = [False] * (target + 1)
    possible[0] = True

    for value in nums:
        for current in range(target, value - 1, -1):
            possible[current] = possible[current] or possible[current - value]
    return possible[target]
```

O(n × target) time, O(target) space. Descending iteration ensures each item is used at most once.

## Dry run

`[1,5,11,5]`: total 22, target 11. Processing values eventually marks sum 11 reachable (the single 11, or 1+5+5), so return true.

## Interviewer–candidate discussion

**Interviewer:** Why is the DP array boolean?

**Candidate:** We only need reachability, not the number of ways or maximum value.

**Interviewer:** Why descend through sums?

**Candidate:** To prevent reusing the same number multiple times in the same iteration.

## Edge cases and mistakes

Odd total; empty input; zeros; duplicates; negative values outside this implementation's contract; target zero. Avoid ascending iteration.

## Tests

```python
assert can_partition([1, 5, 11, 5])
assert not can_partition([1, 2, 3, 5])
assert can_partition([])
assert can_partition([0, 0])
```

Follow-ups: return the actual partition; minimum subset-sum difference; count partitions; target sum with plus/minus signs; bitset optimization.
