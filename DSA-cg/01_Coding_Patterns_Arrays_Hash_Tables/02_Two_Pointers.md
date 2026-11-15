# Two Pointers

**Pattern:** Two Pointers  
**Typical difficulty:** Easy–Medium  
**Use cases:** Sorted pair search, in-place compaction, partitioning, palindrome checks.

## 1. Concept and when to use it

Two pointers maintain two positions in a sequence and move them according to a rule. Common forms:
- Opposite direction: one pointer at each end.
- Same direction: read/write or slow/fast.
- Separate sequences: merge two sorted arrays/lists.

Use it when pointer movement can safely discard a portion of the search space or when one pass can compact/partition data.

## 2. Example: pair sum in a sorted array

Given a sorted array, return whether two distinct elements sum to target.

### Brute force

```python
def has_pair_bruteforce(nums, target):
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] + nums[j] == target:
                return True
    return False
```

Time O(n²), auxiliary space O(1).

### Optimized: opposite-direction pointers

```python
def has_pair_sorted(nums, target):
    left, right = 0, len(nums) - 1

    while left < right:
        total = nums[left] + nums[right]
        if total == target:
            return True
        if total < target:
            left += 1
        else:
            right -= 1

    return False
```

Time O(n), auxiliary space O(1), assuming input is already sorted. Sorting first costs O(n log n) and may alter index semantics.

## 3. Dry run

Input `[1, 2, 4, 7, 11]`, target `9`:
- left=1, right=11 → 12, too large; move right.
- left=1, right=7 → 8, too small; move left.
- left=2, right=7 → 9, found.

## 4. Correctness argument

Because the array is sorted, if the smallest remaining value plus the largest is too small, pairing the smallest with any other remaining value cannot reach target. So the left pointer can advance. If the sum is too large, the largest cannot pair with any other remaining value without exceeding target, so the right pointer can retreat.

## 5. Same-direction example: stable compaction

```python
def remove_duplicates_sorted(nums):
    if not nums:
        return 0

    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[write - 1]:
            nums[write] = nums[read]
            write += 1
    return write
```

Returns the length of the unique prefix; the caller can use `nums[:length]`. Time O(n), auxiliary space O(1).

## 6. Edge cases and mistakes

- Empty and one-element arrays.
- Duplicate values.
- Negative numbers.
- Pair must use distinct indices.
- Input not sorted.
- Sorting may destroy original index positions.
- Moving the wrong pointer can skip valid answers.

## 7. Interviewer–candidate discussion

**Interviewer:** Why is this better than a hash map?

**Candidate:** For an already sorted array, two pointers use O(1) extra space and have a simple elimination proof. A hash map is also expected O(n), but uses O(n) space and does not need sorted input. If original indices are required and sorting is not allowed, the hash map may be preferable.

## 8. Follow-ups

- Return all unique pairs without duplicates.
- Find three values summing to target (sort + fixed index + two pointers).
- Container With Most Water.
- Merge two sorted arrays.
- Check whether a string is a palindrome after ignoring punctuation.
- Find the closest pair sum.

## 9. Test cases

```python
assert has_pair_sorted([], 5) is False
assert has_pair_sorted([4], 8) is False
assert has_pair_sorted([1, 2, 4, 7], 9) is True
assert has_pair_sorted([-3, -1, 2, 8], 1) is True
assert has_pair_sorted([1, 2, 3], 10) is False
assert remove_duplicates_sorted([1, 1, 2, 2, 3]) == 3
```

## Data Engineering connection

Two-pointer merging is useful for merging sorted event streams or timestamped records. Define deterministic tie-breaking when timestamps are equal; otherwise results can vary across runs.
