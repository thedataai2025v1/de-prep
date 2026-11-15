# Analyzing Time Complexity

## 1. A repeatable method

1. Define input-size variables (`n`, `m`, `V`, `E`, key length `L`).
2. Count how many times each important operation executes.
3. Include costs of library operations.
4. Analyze loops, recursion and nested calls.
5. State best/average/worst case where meaningful.
6. Simplify to the dominant asymptotic term.

## 2. Loop examples

- One pass over n items: O(n).
- Two independent passes: O(n) + O(n) = O(n).
- Nested n-by-n loops: O(n²).
- Inner loop doubles a counter: O(n log n) across outer n loop.
- Binary search: O(log n).

## 3. Library operation costs to remember

- Python list membership: O(n).
- Dictionary/set membership: expected O(1).
- Sorting: O(n log n).
- `list.pop(0)`: O(n).
- `deque.popleft()`: O(1).
- Heap push/pop: O(log n).

## 4. Example: nested loop with early exit

```python
def has_pair(nums, target):
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] + nums[j] == target:
                return True
    return False
```

Worst case O(n²), best case O(1) if the first tested pair matches. Average case depends on input distribution; do not invent a precise average without a model.

## 5. Recurrence examples

- Binary search: T(n) = T(n/2) + O(1) → O(log n).
- Merge sort: T(n) = 2T(n/2) + O(n) → O(n log n).
- Naive Fibonacci: T(n) = T(n−1) + T(n−2) + O(1) → exponential.

## 6. Interview phrasing

“The outer loop runs n times. For each i, the inner loop runs at most n−i−1 times. Summing these counts gives n(n−1)/2, so worst-case time is O(n²).”

## Practice

Analyze a two-pointer loop, a heap-based top-k solution, BFS over adjacency lists, and a recursive tree traversal. State all relevant size variables.
