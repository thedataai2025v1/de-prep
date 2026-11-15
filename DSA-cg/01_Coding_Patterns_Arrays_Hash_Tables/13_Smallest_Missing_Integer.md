# Smallest Missing Integer

**Difficulty:** Easy–Medium  
**Topics:** Hash Set, Arrays, Cyclic Placement  
**Problem definition used here:** Find the smallest missing positive integer from an unsorted array.

## 1. Clarify the variant

“Smallest missing integer” can mean smallest missing non-negative integer, smallest missing positive integer, or missing value in a bounded 1..n range. Confirm the exact definition. This lesson solves the smallest missing positive integer.

Example: `[3, 4, -1, 1]` → `2`.

## 2. Brute force

```python
def first_missing_positive_bruteforce(nums):
    candidate = 1
    while candidate in nums:
        candidate += 1
    return candidate
```

This may take O(n²) time due to repeated list membership; O(1) extra space.

## 3. Optimized hash set

```python
def first_missing_positive_set(nums):
    values = set(nums)
    candidate = 1
    while candidate in values:
        candidate += 1
    return candidate
```

Expected O(n) time and O(n) space. At most n distinct positive values can be present among n elements, so the answer is at most n+1.

## 4. O(1) auxiliary-space cyclic placement

```python
def first_missing_positive(nums):
    n = len(nums)
    i = 0

    while i < n:
        value = nums[i]
        target = value - 1
        if 1 <= value <= n and nums[i] != nums[target]:
            nums[i], nums[target] = nums[target], nums[i]
        else:
            i += 1

    for i, value in enumerate(nums):
        if value != i + 1:
            return i + 1

    return n + 1
```

Time O(n), auxiliary space O(1), excluding no extra output. Mutates input.

## 5. Dry run

Input `[3, 4, -1, 1]`:
- 3 belongs at index 2 → swap: `[-1, 4, 3, 1]`.
- -1 is out of range → advance.
- 4 belongs at index 3 → swap: `[-1, 1, 3, 4]`.
- 1 belongs at index 0 → swap: `[1, -1, 3, 4]`.
- Scan: index 0 has 1; index 1 does not have 2.
- Answer = 2.

## 6. Correctness argument

Only values from 1 through n can occupy one of the first n target positions. After placement, if every position i contains i+1, then all positive integers 1..n are present and the answer is n+1. Otherwise, the first incorrect position identifies the smallest missing positive integer.

## 7. Complexity

| Approach | Time | Auxiliary space | Mutates input |
|---|---:|---:|---|
| Repeated membership | O(n²) worst | O(1) | No |
| Hash set | Expected O(n) | O(n) | No |
| Cyclic placement | O(n) | O(1) | Yes |

## 8. Edge cases and mistakes

- Empty input → 1.
- All negatives or zeros → 1.
- Duplicates.
- Missing value is n+1.
- Out-of-range values.
- Failing to check destination equality can cause an infinite loop.
- Using value as index before validating its range.

## 9. Interviewer–candidate discussion

**Interviewer:** Why is the answer bounded by n+1?

**Candidate:** With n array positions, at most n distinct positive integers can be represented. If all values 1 through n are present, the next missing positive is n+1. Otherwise, a gap occurs earlier.

## 10. Follow-ups

- Smallest missing non-negative integer.
- Find all missing values in 1..n.
- Find duplicates in 1..n.
- Solve without mutating input.
- Stream values when the maximum range is known.

## 11. Tests

```python
assert first_missing_positive([1, 2, 0]) == 3
assert first_missing_positive([3, 4, -1, 1]) == 2
assert first_missing_positive([7, 8, 9, 11, 12]) == 1
assert first_missing_positive([]) == 1
assert first_missing_positive([1, 1]) == 2
assert first_missing_positive([2]) == 1
```
