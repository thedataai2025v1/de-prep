# Move Zeros to End of Array

**Difficulty:** Easy  
**Topics:** Arrays, Two Pointers, In-place Mutation  
**Study-plan section:** Overview / Arrays

## 1. Problem statement

Given an integer array, move all zero values to the end while preserving the relative order of non-zero elements. Modify the array in place if the contract requires it.

**Example:** `[0, 1, 0, 3, 12]` → `[1, 3, 12, 0, 0]`

Clarify whether returning a new array is allowed, whether order must be stable, and whether zero means numeric `0` only.

## 2. Approaches

### Baseline: build a new array

```python
def move_zeros_new(nums):
    non_zero = [x for x in nums if x != 0]
    return non_zero + [0] * (len(nums) - len(non_zero))
```

Time O(n); extra space O(n). Simple and stable, but not in-place.

### Optimized: write pointer

```python
def move_zeros(nums):
    write = 0

    for read in range(len(nums)):
        if nums[read] != 0:
            nums[write] = nums[read]
            write += 1

    for i in range(write, len(nums)):
        nums[i] = 0

    return nums
```

Time O(n); auxiliary space O(1). The write pointer marks the next position for a non-zero value.

## 3. Dry run

Input: `[0, 1, 0, 3, 12]`

- Read 0: skip.
- Read 1: write at index 0 → `[1, 1, 0, 3, 12]`.
- Read 0: skip.
- Read 3: write at index 1 → `[1, 3, 0, 3, 12]`.
- Read 12: write at index 2 → `[1, 3, 12, 3, 12]`.
- Fill indices 3 and 4 with zero → `[1, 3, 12, 0, 0]`.

## 4. Correctness argument

The read pointer visits every original position in order. Each non-zero value is copied to the next write position, preserving relative order. The number of non-zero values determines the suffix, which is then filled with zeros.

## 5. Interview conversation

**Interviewer:** Can you avoid swapping every zero?

**Candidate:** Yes. A compacting write pointer writes only non-zero values, then fills the remaining suffix. This is O(n) time and O(1) auxiliary space.

**Interviewer:** Does it preserve order?

**Candidate:** Yes, because the read pointer moves left to right and writes non-zero values in that same order.

## 6. Edge cases

- `[]`
- `[0]`
- `[1, 2, 3]`
- `[0, 0, 0]`
- `[1, 0, 2, 0, 3]`
- Negative numbers and repeated non-zero values.

## 7. Follow-ups

- Move zeros to the beginning.
- Preserve order versus allow arbitrary order.
- Move a configurable sentinel value.
- Process a stream where the final array cannot be revisited.

## Data Engineering connection

This is a stable compaction operation. Similar patterns appear when filtering invalid records into a dense output buffer. For very large datasets, use partitioned transformations rather than trying to mutate a distributed collection in place.
