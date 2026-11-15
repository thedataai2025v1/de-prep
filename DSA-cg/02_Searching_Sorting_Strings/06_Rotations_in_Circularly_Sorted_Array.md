# Rotations in a Circularly Sorted Array

**Difficulty:** Medium  
**Topics:** Modified binary search, pivot, duplicates

## 1. Problem statement

A sorted ascending array has been rotated at an unknown pivot. Find the index of its minimum element. Clarify whether duplicates are allowed.

## 2. Approaches

### Brute force

```python
def find_min_linear(nums):
    if not nums:
        raise ValueError("array must not be empty")
    return min(range(len(nums)), key=nums.__getitem__)
```
O(n) time, O(1) auxiliary space.

### Optimized binary search (distinct values)

```python
def find_min_rotated(nums):
    if not nums:
        raise ValueError("array must not be empty")
    left, right = 0, len(nums) - 1
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] > nums[right]:
            left = mid + 1
        else:
            right = mid
    return left
```
O(log n) time, O(1) space when values are distinct.

### Duplicate-aware variant

```python
def find_min_rotated_duplicates(nums):
    if not nums:
        raise ValueError("array must not be empty")
    left, right = 0, len(nums) - 1
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] > nums[right]:
            left = mid + 1
        elif nums[mid] < nums[right]:
            right = mid
        else:
            right -= 1
    return left
```
Worst-case O(n) with many duplicates, average often O(log n).

## 3. Dry run

`[4,5,6,7,0,1,2]`: midpoint is 7 and greater than right value 2, so minimum is to the right. New interval `[4,6]`; midpoint 0 is not greater than 2, so keep left side including 0. Converges at index 4.

## 4. Interview discussion

**Interviewer:** Why compare with `nums[right]`?

**Candidate:** It identifies which side contains the rotation boundary when values are distinct. If midpoint is greater, the minimum must be right of midpoint; otherwise it is at midpoint or left.

**Interviewer:** What changes with duplicates?

**Candidate:** Equality gives no directional information, so I shrink the right boundary by one. This can degrade to O(n).

## 5. Edge cases and mistakes

No rotation, rotation by n-1, singleton, duplicates, empty input. Do not claim O(log n) worst case for the duplicate-aware algorithm. Do not return the minimum value if the problem asks for its index.

## 6. Tests and follow-ups

```python
assert find_min_rotated([3, 4, 5, 1, 2]) == 3
assert find_min_rotated([1, 2, 3]) == 0
assert find_min_rotated([2]) == 0
assert find_min_rotated_duplicates([2, 2, 2, 0, 1]) == 3
```
Follow-up: search for a target in the rotated array; determine the pivot; support repeated queries.
