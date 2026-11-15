# Cyclic Sort and Cyclic Placement

**Pattern:** Place each value at its corresponding index  
**Use cases:** Missing / duplicate values when the domain is tightly constrained.

## 1. Concept and when to use it

If an array has n positions and values are expected in a known range such as 1..n, value `v` belongs at index `v - 1`. Swap values into their target positions until each position is correct or a duplicate blocks progress.

This is not a general-purpose sorting algorithm. Use it only when the value domain and placement rule are explicit.

## 2. Example: find missing numbers in range 1..n

### Baseline

Use a set and scan 1..n: O(n) expected time and O(n) extra space.

### Optimized cyclic placement

```python
def find_missing_numbers(nums):
    n = len(nums)
    i = 0

    while i < n:
        value = nums[i]
        correct_index = value - 1
        if 1 <= value <= n and nums[i] != nums[correct_index]:
            nums[i], nums[correct_index] = nums[correct_index], nums[i]
        else:
            i += 1

    return [i + 1 for i, value in enumerate(nums) if value != i + 1]
```

Time O(n) under the bounded-domain placement argument; auxiliary space O(1) excluding output. The input is mutated.

## 3. Dry run

Input `[4, 3, 2, 7, 8, 2, 3, 1]`:
- Place 4 at index 3.
- Place 7 at index 6.
- Place 3 at index 2.
- Place 2 at index 1.
- Duplicate 2 prevents further swap at its target.
- Continue placing values where possible.
- Final scan finds positions whose expected value is absent: `[5, 6]`.

## 4. Correctness argument

Each successful swap places at least one value into its correct index, and the duplicate check prevents infinite swapping when the target already contains the same value. Once scanning ends, any index i not containing i+1 corresponds to a missing value i+1.

## 5. Complexity

- Expected/standard analysis: O(n) swaps plus O(n) final scan.
- Auxiliary working space: O(1), excluding returned list.
- Input mutation: yes.

## 6. Edge cases and mistakes

- Values outside 1..n.
- Duplicates.
- Empty input.
- Failing to guard the valid range before indexing.
- Swapping without checking whether the destination already holds the same value (can loop forever).
- Assuming the method works for arbitrary integers.

## 7. Interviewer–candidate discussion

**Interviewer:** Why is the repeated while loop still linear?

**Candidate:** Each successful swap puts at least one element into its final position. There can be at most O(n) such placements. The scan index also advances at most n times, so total work is O(n).

## 8. Follow-ups

- Find the one missing number from 1..n.
- Find the duplicate.
- Find the first missing positive integer.
- Return all duplicates.
- Compare with hash-set and sign-marking approaches.

## 9. Tests

```python
assert find_missing_numbers([4, 3, 2, 7, 8, 2, 3, 1]) == [5, 6]
assert find_missing_numbers([1, 1]) == [2]
assert find_missing_numbers([]) == []
assert find_missing_numbers([1, 2, 3]) == []
```

Note: tests mutate their input arrays. Pass a copy if the original data must be retained.
