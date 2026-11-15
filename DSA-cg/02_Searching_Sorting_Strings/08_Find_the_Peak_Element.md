# Find the Peak Element

**Difficulty:** Medium  
**Topics:** Binary search, local maxima, slope reasoning

## 1. Problem statement

A peak is an element strictly greater than its neighbors. Treat out-of-bounds neighbors as negative infinity. Return the index of any peak. The common problem contract assumes adjacent values are unequal.

## 2. Approaches

### Brute force

```python
def find_peak_linear(nums):
    for i, value in enumerate(nums):
        left_ok = i == 0 or value > nums[i - 1]
        right_ok = i == len(nums) - 1 or value > nums[i + 1]
        if left_ok and right_ok:
            return i
    return -1
```
O(n) time, O(1) space.

### Optimized binary search

```python
def find_peak(nums):
    if not nums:
        return -1
    left, right = 0, len(nums) - 1
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] > nums[mid + 1]:
            right = mid
        else:
            left = mid + 1
    return left
```
O(log n) time, O(1) space.

## 3. Dry run

For `[1,2,1,3,5,6,4]`, midpoint index 3 has value 3 and next value 5 is larger. Move right. Midpoint of `[4,6]` is index 5 with value 6, greater than next value 4; move left to index 5. Return 5.

## 4. Why it works

If the slope rises from `mid` to `mid+1`, a peak must exist on the right (eventually the sequence ends or turns downward). If it falls, a peak exists at mid or to the left. The search interval always retains at least one peak.

## 5. Interview discussion

**Interviewer:** Does the algorithm find the global maximum?

**Candidate:** No. It finds any local peak. If the requirement is the global maximum, a linear scan is generally needed unless more structure is guaranteed.

**Interviewer:** What if adjacent equal values are allowed?

**Candidate:** The strict comparison proof needs adjustment; plateaus can make the search ambiguous. I would clarify the contract and design a duplicate-aware strategy, which may degrade to O(n).

## 6. Edge cases and mistakes

Empty input, one element, strictly increasing, strictly decreasing, multiple peaks. Do not compare `nums[mid]` with both neighbors and accidentally lose the logarithmic invariant. Do not claim global maximum.

## 7. Tests

```python
assert find_peak([]) == -1
assert find_peak([7]) == 0
assert find_peak([1, 2, 3, 1]) == 2
assert find_peak([1, 2, 1, 3, 5, 6, 4]) in {1, 5}
assert find_peak([5, 4, 3, 2]) == 0
```
Follow-ups: return all peaks, handle plateaus, or find a peak in a 2D matrix.
