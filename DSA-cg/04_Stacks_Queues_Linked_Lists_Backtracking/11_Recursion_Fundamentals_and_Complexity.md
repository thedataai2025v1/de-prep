# Recursion Fundamentals and Complexity

**Difficulty:** Foundation–Medium  
**Topics:** Base case, recursive case, call stack, recurrence

## 1. Concept and when to use

Recursion solves a problem by reducing it to smaller instances of the same problem. It is natural for trees, divide-and-conquer and backtracking. Every recursive solution needs:
1. A base case.
2. Progress toward the base case.
3. A correct combination of subproblem results.

## 2. Example: factorial

### Iterative baseline

```python
def factorial_iterative(n):
    if n < 0:
        raise ValueError("n must be non-negative")
    result = 1
    for value in range(2, n + 1):
        result *= value
    return result
```

### Recursive version

```python
def factorial_recursive(n):
    if n < 0:
        raise ValueError("n must be non-negative")
    if n <= 1:
        return 1
    return n * factorial_recursive(n - 1)
```

Both take O(n) time. Iterative space is O(1); recursive call-stack space is O(n).

## 3. Dry run

`factorial_recursive(4)` expands to `4 * factorial(3)`, then `3 * factorial(2)`, then `2 * factorial(1)`. Base case returns 1; unwind: 2, 6, 24.

## 4. Interviewer–candidate discussion

**Interviewer:** Is recursion always simpler?

**Candidate:** It can make tree and divide-and-conquer logic clearer, but deep recursion can overflow Python's call stack and adds call overhead. An iterative version may be safer.

**Interviewer:** How do you analyze recursive complexity?

**Candidate:** Write the recurrence. For factorial, T(n)=T(n−1)+O(1)=O(n). For merge sort, T(n)=2T(n/2)+O(n)=O(n log n).

## 5. Common mistakes

Missing base case, no progress, wrong return combination, repeated subproblem work, mutation shared across branches, Python recursion-depth limits. Do not increase the recursion limit casually in production.

## 6. Tests and exercises

```python
assert factorial_recursive(0) == 1
assert factorial_recursive(1) == 1
assert factorial_recursive(4) == 24
assert factorial_iterative(5) == 120
```

Exercises: sum a list recursively; reverse a string; compute tree height; compare recursive and iterative DFS; memoize Fibonacci and analyze its complexity.
