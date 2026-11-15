# Permutations and Unique Permutations

**Difficulty:** Medium  
**Topics:** Backtracking, used array, duplicate pruning

## 1. Problem statement

Generate all orderings of the input values. With n distinct elements there are n! permutations.

## 2. Approaches

### Baseline

Generate all permutations and deduplicate using a set. This can waste memory and work when duplicates are frequent.

### Optimized for distinct values

```python
def permutations(nums):
    result = []
    path = []
    used = [False] * len(nums)

    def backtrack():
        if len(path) == len(nums):
            result.append(path.copy())
            return
        for i, value in enumerate(nums):
            if used[i]:
                continue
            used[i] = True
            path.append(value)
            backtrack()
            path.pop()
            used[i] = False

    backtrack()
    return result
```

O(n·n!) time to materialize output; O(n) auxiliary space excluding output.

### Unique permutations with duplicates

```python
def unique_permutations(nums):
    nums = sorted(nums)
    result, path = [], []
    used = [False] * len(nums)

    def backtrack():
        if len(path) == len(nums):
            result.append(path.copy())
            return
        for i in range(len(nums)):
            if used[i]:
                continue
            if i > 0 and nums[i] == nums[i - 1] and not used[i - 1]:
                continue
            used[i] = True
            path.append(nums[i])
            backtrack()
            path.pop()
            used[i] = False

    backtrack()
    return result
```

## 3. Dry run

For `[1,2]`: choose 1, then 2 → `[1,2]`; backtrack; choose 2, then 1 → `[2,1]`.

For `[1,1,2]`, sorting groups equal values. The skip condition prevents choosing the second 1 before the first 1 at the same recursion depth, eliminating duplicate branches.

## 4. Interviewer–candidate discussion

**Interviewer:** Why is permutation complexity factorial?

**Candidate:** There are n choices for the first position, n−1 for the second, and so on: n!.

**Interviewer:** Why does the duplicate check use `not used[i-1]`?

**Candidate:** It prevents equal values from being selected in the wrong order at the current depth, while still allowing both copies in the same permutation.

## 5. Edge cases and mistakes

Empty input convention (one empty permutation), repeated values, input mutation due to sorting, failure to reset `used`, failure to copy path.

## 6. Tests

```python
assert permutations([]) == [[]]
assert {tuple(x) for x in permutations([1, 2])} == {(1, 2), (2, 1)}
unique = unique_permutations([1, 1, 2])
assert len(unique) == 3
assert {tuple(x) for x in unique} == {(1, 1, 2), (1, 2, 1), (2, 1, 1)}
```

Follow-ups: next permutation in-place; kth permutation without generating all; permutations with constraints; compare backtracking and Heap's algorithm.
