# Subsets and Power Set

**Difficulty:** Medium  
**Topics:** Backtracking, include/exclude, bitmask enumeration

## 1. Problem statement

Return every subset of a list of distinct values. For n values there are 2^n subsets.

## 2. Approaches

### Iterative include/exclude expansion

```python
def subsets_iterative(nums):
    result = [[]]
    for value in nums:
        result += [subset + [value] for subset in result]
    return result
```
O(n·2^n) time including output construction; output space O(n·2^n).

### Backtracking

```python
def subsets(nums):
    result = []
    path = []

    def backtrack(start):
        result.append(path.copy())
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1)
            path.pop()

    backtrack(0)
    return result
```
O(n·2^n) output-sensitive time, O(n) auxiliary path/recursion space, excluding output.

### Bitmask alternative

For each mask from 0 to 2^n−1, include item i if bit i is set. Also O(n·2^n) time to materialize all subsets.

## 3. Dry run

For `[1,2]`: begin `[[]]`; process 1 → `[[],[1]]`; process 2 → `[[],[1],[2],[1,2]]`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why does the result contain 2^n subsets?

**Candidate:** Each element has two independent choices: include or exclude.

**Interviewer:** What if duplicate input values exist?

**Candidate:** The distinct-subset version needs sorting and skipping equal values at the same recursion depth, or a frequency-based approach.

## 5. Edge cases and mistakes

Empty list must return `[[]]`; do not append the mutable path without copying; clarify whether duplicates are allowed and whether output order matters.

## 6. Tests

```python
assert subsets([]) == [[]]
result = subsets([1, 2])
assert len(result) == 4
assert {tuple(sorted(x)) for x in result} == {(), (1,), (2,), (1, 2)}
assert len(subsets([1, 2, 3])) == 8
```

Follow-ups: subsets with duplicates; subsets of size k; bitmask approach; stream results lazily to reduce peak memory.
