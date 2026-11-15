# Binary Search: Boundaries and Variants

**Difficulty:** Medium  
**Topics:** Lower bound, upper bound, first/last occurrence, insertion position

## 1. Concept and when to use

Classic binary search answers “is the target present?” Boundary search answers “where does a monotonic condition change?” Use it for first occurrence, insertion points, range counts, and threshold decisions.

Maintain a precise invariant. The implementations below use a half-open interval `[left, right)`.

## 2. Approaches

### Baseline: scan for first and last occurrence

```python
def equal_range_linear(nums, target):
    hits = [i for i, x in enumerate(nums) if x == target]
    return (hits[0], hits[-1]) if hits else (-1, -1)
```
O(n) time and O(k) auxiliary space for `k` matches.

### Optimized: lower bound and upper bound

```python
def lower_bound(nums, target):
    left, right = 0, len(nums)  # [left, right)
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid
    return left

def upper_bound(nums, target):
    left, right = 0, len(nums)
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] <= target:
            left = mid + 1
        else:
            right = mid
    return left

def equal_range(nums, target):
    first = lower_bound(nums, target)
    after_last = upper_bound(nums, target)
    if first == len(nums) or nums[first] != target:
        return (-1, -1)
    return (first, after_last - 1)
```
O(log n) time and O(1) auxiliary space.

## 3. Detailed dry run

For `nums=[1, 2, 2, 2, 4]`, target 2:
- `lower_bound`: midpoint value 2 is not less than target, so move `right` left. Continue until index 1; result 1.
- `upper_bound`: midpoint values equal to 2 satisfy `<= target`, so move `left` right. First value greater than 2 is at index 4; result 4.
- The inclusive matching range is `(1, 3)` and count is `4 - 1 = 3`.

## 4. Complexity

Both boundaries: O(log n) time, O(1) auxiliary space. The input must be sorted according to the same ordering used by the comparison.

## 5. Interviewer–candidate discussion

**Interviewer:** Why is `right=len(nums)` valid?

**Candidate:** The interval is half-open, so `len(nums)` is a valid exclusive boundary and allows the insertion point to be at the end.

**Interviewer:** How do you count duplicates?

**Candidate:** `upper_bound - lower_bound`.

## 6. Edge cases and common mistakes

- Target smaller than all values or larger than all values.
- Empty array.
- All values equal.
- Confusing upper bound (first value `> target`) with last occurrence.
- Mixing inclusive and half-open interval conventions.
- Accessing `nums[left]` without checking `left < len(nums)`.

## 7. Follow-ups

1. Find the first index whose value is at least `x`.
2. Count values in `[low, high]`: `upper_bound(high) - lower_bound(low)`.
3. Find the first `True` in a monotonic Boolean array.

## 8. Practical tests

```python
assert equal_range([1, 2, 2, 2, 4], 2) == (1, 3)
assert equal_range([], 3) == (-1, -1)
assert equal_range([1, 3, 5], 0) == (-1, -1)
assert lower_bound([1, 3, 5], 6) == 3
assert upper_bound([1, 3, 5], 3) == 2
```

**Principal-level connection:** Boundary search is common in time-series partitioning, pagination cursors and finding the first timestamp meeting an SLA threshold.
