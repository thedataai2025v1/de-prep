# Understanding Big O Notation

## 1. Purpose

Big O describes how resource usage grows as input size grows. It is an asymptotic upper-bound notation; in interview discussions it is commonly used to describe growth rate while ignoring constant factors and lower-order terms.

## 2. Common growth rates

| Complexity | Name | Typical example |
|---|---|---|
| O(1) | Constant | Array index access |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | One pass through array |
| O(n log n) | Linearithmic | Comparison sorting |
| O(n²) | Quadratic | All pairs |
| O(2ⁿ) | Exponential | Enumerating subsets |
| O(n!) | Factorial | Enumerating permutations |

## 3. Rules of thumb

- Sequential blocks: add costs, then keep the dominant term.
- Nested loops: multiply iteration counts when independent.
- Halving search space: often O(log n).
- Recursion: derive from the recurrence, not indentation alone.
- Hash operations: usually expected O(1), not guaranteed worst-case O(1).
- Sorting: usually O(n log n) for comparison sorting.

## 4. Examples

```python
def example(nums):
    total = 0                 # O(1)
    for x in nums:            # O(n)
        total += x
    for i in range(len(nums)):
        for j in range(len(nums)):
            total += i + j    # O(n^2)
    return total
```

Total time is O(n²), because it dominates O(n) and O(1).

## 5. Interview conversation

**Interviewer:** Is O(2n) different from O(n)?

**Candidate:** They have the same asymptotic growth class, so O(2n) simplifies to O(n). Constants still matter in real performance, but Big O abstracts them for growth analysis.

## 6. Common misconceptions

- Big O is not an exact runtime.
- O(n) is not always faster than O(n log n) for tiny n.
- Average hash-table behavior is not a worst-case guarantee.
- Auxiliary space and total space are different.
- An algorithm can have different best, average and worst cases.

## Practice

Classify: linear scan, nested pair comparison, binary search, merge sort, subset generation, and factorial permutation generation.
