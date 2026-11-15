# Contiguous Subarray Sum

**Difficulty:** Hard  
**Topics:** Arrays, Prefix Sum, Hash Map

## 1. Problem statement

Given an integer array and a target value, determine whether a non-empty contiguous subarray sums to the target. Some variants ask for the start/end indices or count of matching subarrays; clarify the required output.

**Example:** `nums = [1, 2, 3, -2, 5]`, `target = 3` → `True` because `[1, 2]` or `[3]` sums to 3.

## 2. Approaches

### Baseline: enumerate subarrays

```python
def has_subarray_sum_bruteforce(nums, target):
    for start in range(len(nums)):
        total = 0
        for end in range(start, len(nums)):
            total += nums[end]
            if total == target:
                return True
    return False
```

Time O(n²), auxiliary space O(1). Incremental sums avoid recomputing each subarray from scratch.

### Optimized: prefix sum + set

Let prefix sum P[j] be the sum through index j. A subarray ending at j sums to target when an earlier prefix equals `P[j] - target`.

```python
def has_subarray_sum(nums, target):
    seen_prefixes = {0}
    prefix = 0

    for value in nums:
        prefix += value
        if prefix - target in seen_prefixes:
            return True
        seen_prefixes.add(prefix)

    return False
```

Expected O(n) time and O(n) space. Initializing with zero handles a matching prefix beginning at index 0.

## 3. Dry run

`nums = [1, 2, -1, 4]`, target 3:
- Start with prefix 0, seen `{0}`.
- Read 1: prefix 1; need -2; add 1.
- Read 2: prefix 3; need 0, found → `True` for `[1,2]`.

## 4. Correctness argument

For current prefix P, any earlier prefix Q satisfying P − Q = target defines a contiguous segment whose sum is target. The set stores all earlier prefixes, so the lookup is sufficient.

## 5. Important variant: count matching subarrays

```python
from collections import defaultdict

def count_subarrays_sum(nums, target):
    frequency = defaultdict(int)
    frequency[0] = 1
    prefix = 0
    count = 0

    for value in nums:
        prefix += value
        count += frequency[prefix - target]
        frequency[prefix] += 1

    return count
```

Expected O(n) time and O(n) space. A frequency map is needed because the same prefix sum can occur multiple times.

## 6. Interview conversation

**Interviewer:** Why not use a sliding window?

**Candidate:** A conventional sum-based sliding window relies on monotonic behavior, usually non-negative values. Negative values can make the sum increase or decrease unpredictably when the window moves. Prefix sums work with positive, zero and negative integers.

## 7. Edge cases

- Empty input.
- Single element equal to target.
- Negative numbers.
- Target zero.
- Repeated prefix sums.
- Multiple matching subarrays.

## 8. Follow-ups

- Return indices, not just a boolean.
- Count all matching subarrays.
- Longest subarray with sum K.
- Binary array with sum K.
- Streaming input: prefix-frequency state can be maintained incrementally, but memory may grow with distinct prefix sums.

## Data Engineering connection

Prefix-sum state appears in running balances, cumulative counters and reconciliation. For distributed partitions, partial prefix totals must be offset by preceding partition totals; independent partition sums are not sufficient to recover global contiguous segments.
