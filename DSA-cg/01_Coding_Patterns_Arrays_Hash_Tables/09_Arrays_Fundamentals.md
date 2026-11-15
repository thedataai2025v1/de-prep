# Arrays: Fundamentals and Interview Reasoning

## 1. Concept

An array stores elements in indexed order. In Python, `list` is a dynamic array of object references, not a fixed-width primitive array.

Typical costs:
- Index read/write: O(1).
- Append: amortized O(1).
- Search by value: O(n).
- Insert/delete at front or middle: O(n).
- Sorting: O(n log n) comparison-sort complexity.

## 2. When arrays are a good fit

Use arrays when order, contiguous ranges, indexed access, sorting or in-place transformations matter. Ask whether the input may be mutated and whether stable order is required.

## 3. Example: maximum subarray sum

### Brute force

```python
def max_subarray_bruteforce(nums):
    if not nums:
        raise ValueError("nums must be non-empty")
    best = nums[0]
    for start in range(len(nums)):
        total = 0
        for end in range(start, len(nums)):
            total += nums[end]
            best = max(best, total)
    return best
```

Time O(n²), auxiliary space O(1).

### Optimized: Kadane's algorithm

```python
def max_subarray_sum(nums):
    if not nums:
        raise ValueError("nums must be non-empty")
    current = best = nums[0]
    for value in nums[1:]:
        current = max(value, current + value)
        best = max(best, current)
    return best
```

Time O(n), auxiliary space O(1).

## 4. Dry run

`[-2, 1, -3, 4, -1, 2, 1, -5, 4]`
- Start current=best=-2.
- At 1: current=1, best=1.
- At -3: current=-2, best=1.
- At 4: current=4, best=4.
- At -1: current=3, best=4.
- At 2: current=5, best=5.
- At 1: current=6, best=6.
- At -5: current=1, best=6.
- At 4: current=5, best=6.
Answer 6.

## 5. Correctness intuition

At each position, the best subarray ending there either starts at the current element or extends the best subarray ending at the previous position. The recurrence is `current = max(value, current + value)`.

## 6. Edge cases and mistakes

- All-negative array: initialize from first value, not zero.
- Empty array: define behavior.
- One element.
- Integer overflow in fixed-width languages.
- Confusing maximum subarray with maximum subsequence.
- Accidentally allowing an empty subarray when the contract forbids it.

## 7. Interviewer–candidate discussion

**Interviewer:** Why is Kadane's O(n)?

**Candidate:** Each element is processed once, and each step performs constant work using only the best sum ending at the previous index and the global best.

## 8. Follow-ups

- Return start and end indices.
- Maximum circular subarray.
- Maximum product subarray.
- Maximum sum with at most one deletion.
- 2D maximum submatrix sum.

## 9. Tests

```python
assert max_subarray_sum([-2, 1, -3, 4, -1, 2, 1, -5, 4]) == 6
assert max_subarray_sum([-5, -2, -8]) == -2
assert max_subarray_sum([7]) == 7
assert max_subarray_sum([1, 2, 3]) == 6
```
