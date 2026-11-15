# Monotonic Queue: Sliding Window Maximum

**Level:** Medium–Hard | **Patterns:** Deque, monotonic structure, amortized analysis

## 1. Concept and when to use it

A monotonic deque stores candidate indices in an order where their values are decreasing. For each sliding window, the front is the maximum. Smaller values at the back are removed because a newer, larger value will remain useful for at least as long.

Use this when a window moves one position at a time and you need its maximum or minimum in near-constant amortized time. Typical signals: rolling maximum, peak load over the last `k` events, and max/min over fixed-size windows.

## 2. Approaches

### Brute force

For every window, scan its `k` values to find the maximum. There are `n-k+1` windows, so time is `O(nk)` and extra space is `O(1)`.

### Optimized: decreasing deque of indices

```python
from collections import deque

def sliding_window_max(nums, k):
    if k <= 0 or k > len(nums):
        raise ValueError("k must be between 1 and len(nums)")

    candidates = deque()
    result = []

    for i, value in enumerate(nums):
        # Remove indices that have fallen outside the window.
        while candidates and candidates[0] <= i - k:
            candidates.popleft()

        # Remove dominated values from the back.
        while candidates and nums[candidates[-1]] <= value:
            candidates.pop()

        candidates.append(i)

        if i >= k - 1:
            result.append(nums[candidates[0]])

    return result
```

## 3. Detailed dry run

Input: `nums = [1, 3, -1, -3, 5, 3, 6, 7]`, `k = 3`.

- `i=0`, value 1: deque `[0]`.
- `i=1`, value 3: 1 is dominated; deque `[1]`.
- `i=2`, value -1: deque `[1,2]`; first window max is 3.
- `i=3`, value -3: deque `[1,2,3]`; max is still 3.
- `i=4`, value 5: index 1 expires; -3 and -1 are dominated; deque `[4]`; max is 5.
- `i=5`, value 3: deque `[4,5]`; max is 5.
- `i=6`, value 6: old candidates are dominated; deque `[6]`; max is 6.
- `i=7`, value 7: 6 is dominated; deque `[7]`; max is 7.

Result: `[3, 3, 5, 5, 6, 7]`.

## 4. Complexity

Each index is appended once and removed at most once. Total time is `O(n)` amortized; deque space is `O(k)`.

## 5. Edge cases and common mistakes

- `k=1` returns the original values.
- `k=len(nums)` returns one value.
- Reject `k<=0` and `k>n`.
- Store indices, not just values, so expired entries can be identified.
- Remove expired indices before reading the front.
- Use `<=` to discard equal older values; using `<` is also correct but may retain duplicates.
- Explain amortized complexity: a single iteration may pop many items, but every item is popped only once.

## 6. Interviewer–candidate discussion

**Interviewer:** Why can you discard a smaller value when a larger value arrives?

**Candidate:** The newer value is larger and expires later. Any future window containing the older candidate and the new value will prefer the new value. Once the older one is behind the new one, it can never become the maximum again.

**Interviewer:** Is each loop iteration O(1)?

**Candidate:** Not necessarily. One iteration can pop many candidates. Across the full input, each index enters and leaves the deque at most once, so the total work is linear.

## 7. Follow-ups and variations

1. Sliding-window minimum: maintain an increasing deque.
2. Return the index of each maximum rather than the value.
3. Support variable window sizes.
4. Combine max and min deques to find the longest subarray whose max-min is at most a threshold.
5. Explain why a heap solution is usually `O(n log k)` and requires careful stale-entry handling.

## 8. Practical exercises and tests

```python
assert sliding_window_max([1, 3, -1, -3, 5, 3, 6, 7], 3) == [3, 3, 5, 5, 6, 7]
assert sliding_window_max([4], 1) == [4]
assert sliding_window_max([2, 2, 2], 2) == [2, 2]
assert sliding_window_max([-5, -2, -7], 2) == [-2, -2]
try:
    sliding_window_max([1, 2], 0)
    raise AssertionError("Expected invalid k")
except ValueError:
    pass
```

**Data-platform connection:** Rolling peak CPU, request rate, or queue depth is a sliding-window maximum problem. In distributed systems, event ordering, late arrivals, and window boundaries must also be specified.
