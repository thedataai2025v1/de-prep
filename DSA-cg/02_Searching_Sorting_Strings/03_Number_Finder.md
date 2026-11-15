# Number Finder

**Difficulty:** Easy–Medium  
**Topics:** Search, sorted arrays, boundary reasoning

## 1. Problem statement

Given a sorted list of integers and a target, return its index or `-1` if absent. Clarify whether duplicates exist and whether any, first, or last occurrence is required.

## 2. Approaches

### Brute force

```python
def find_number_linear(nums, target):
    for i, value in enumerate(nums):
        if value == target:
            return i
    return -1
```
O(n) time, O(1) auxiliary space.

### Optimized binary search

```python
def find_number(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
```
O(log n) time, O(1) space.

## 3. Dry run

`nums=[-5, -1, 0, 4, 9, 12]`, target 9:
- `left=0, right=5`, `mid=2`, value 0; discard left half.
- `left=3, right=5`, `mid=4`, value 9; return 4.

## 4. Interview discussion

**Interviewer:** What if the list is not sorted?

**Candidate:** Binary search is invalid. I would use a linear scan, or build a set if there are repeated membership queries and ordering is irrelevant.

**Interviewer:** What if the list contains duplicates?

**Candidate:** The current implementation returns an arbitrary matching index. If the contract requires the first occurrence, I will use lower bound.

## 5. Edge cases and mistakes

Empty list, singleton, target at either end, absent target between values, negative values, duplicates. Do not use binary search on unsorted data or return an arbitrary duplicate index when the contract requires a boundary.

## 6. Follow-ups and tests

```python
assert find_number([], 1) == -1
assert find_number([7], 7) == 0
assert find_number([1, 3, 5], 1) == 0
assert find_number([1, 3, 5], 5) == 2
assert find_number([1, 3, 5], 4) == -1
```

Ask whether the data is static or frequently updated; that changes the index choice.
