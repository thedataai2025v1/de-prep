# Maximum Product Subarray

**Difficulty:** Medium | **Pattern:** DP with maximum and minimum states

## Problem

Find the maximum product of a non-empty contiguous subarray.

## Why ordinary Kadane's algorithm is insufficient

A negative number can turn a very negative product into a large positive product. Track both the maximum and minimum product ending at the current index.

## Approaches

Brute force checks all O(n²) subarrays (or O(n³) if product is recomputed each time).

Optimized:

```python
def max_product_subarray(nums):
    if not nums:
        raise ValueError("nums must be non-empty")

    current_max = current_min = answer = nums[0]
    for value in nums[1:]:
        if value < 0:
            current_max, current_min = current_min, current_max
        current_max = max(value, current_max * value)
        current_min = min(value, current_min * value)
        answer = max(answer, current_max)
    return answer
```

O(n) time, O(1) space.

## Dry run

For `[2,3,-2,4]`:
- Start max=min=answer=2.
- At 3: max=6, min=3, answer=6.
- At -2: swap roles; max=-2, min=-12, answer remains 6.
- At 4: max=4, min=-48, answer remains 6.
Answer 6.

## Interviewer–candidate discussion

**Interviewer:** Why keep the minimum?

**Candidate:** Multiplying a negative minimum by a negative current value can create the new maximum.

**Interviewer:** What does zero do?

**Candidate:** It resets products crossing that position; starting a new subarray at the current value handles the reset naturally.

## Edge cases and mistakes

Single element; all negatives; even/odd number of negatives; zeros; empty input. Do not initialize the answer to zero because all values may be negative.

## Tests

```python
assert max_product_subarray([2, 3, -2, 4]) == 6
assert max_product_subarray([-2, 0, -1]) == 0
assert max_product_subarray([-2, 3, -4]) == 24
assert max_product_subarray([-3]) == -3
```

Follow-ups: return start/end indices; maximum sum subarray; maximum product with at most one deletion; explain state invariant.
