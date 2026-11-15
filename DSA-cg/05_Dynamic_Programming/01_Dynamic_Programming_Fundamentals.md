# Dynamic Programming Fundamentals

**Difficulty:** Foundation–Medium | **Pattern:** Overlapping subproblems + optimal substructure

## Concept and when to use it

Dynamic programming (DP) stores answers to smaller subproblems so they are not recomputed. Look for:
- **Overlapping subproblems:** the same state is reached repeatedly.
- **Optimal substructure:** an optimal answer can be built from optimal answers to smaller states.

A reliable interview workflow:
1. Define the state in plain language.
2. Write the recurrence.
3. Identify base cases.
4. Choose evaluation order.
5. Decide memoization or tabulation.
6. Prove each transition and analyze time/space.

## Brute force vs DP: Fibonacci

Naive recursion branches into repeated calls: exponential time, O(n) call stack.

```python
def fib_naive(n):
    if n < 0:
        raise ValueError("n must be non-negative")
    if n <= 1:
        return n
    return fib_naive(n - 1) + fib_naive(n - 2)
```

Memoization:

```python
from functools import lru_cache

def fib_memo(n):
    if n < 0:
        raise ValueError("n must be non-negative")

    @lru_cache(None)
    def solve(i):
        if i <= 1:
            return i
        return solve(i - 1) + solve(i - 2)

    return solve(n)
```

Tabulation with O(1) space:

```python
def fib_optimized(n):
    if n < 0:
        raise ValueError("n must be non-negative")
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

## Dry run

For n=5, recurrence F(5)=F(4)+F(3). Memoization computes F(3), F(2), F(1), F(0) once, then reuses cached values. Tabulation evolves `(a,b)` as `(0,1) → (1,1) → (1,2) → (2,3) → (3,5) → (5,8)`, returning 5.

## Complexity

| Method | Time | Extra space |
|---|---:|---:|
| Naive recursion | O(φⁿ) | O(n) stack |
| Memoization | O(n) | O(n) cache + stack |
| Tabulation | O(n) | O(n) |
| Rolling variables | O(n) | O(1) |

## Interviewer–candidate discussion

**Interviewer:** When would you choose memoization over tabulation?

**Candidate:** Memoization is convenient when only a subset of states is reachable or the recurrence is naturally recursive. Tabulation avoids recursion overhead and can make memory optimization easier.

**Interviewer:** Does every recursive problem need DP?

**Candidate:** No. If subproblems do not overlap, caching may not help; divide-and-conquer algorithms such as merge sort have distinct subproblems.

## Edge cases and mistakes

Define n=0 explicitly; avoid cache keys that omit state dimensions; do not confuse number of states with transitions per state; ensure the iteration order computes dependencies first.

## Tests and exercises

```python
assert fib_memo(0) == 0
assert fib_memo(1) == 1
assert fib_memo(10) == 55
assert fib_optimized(40) == 102334155
```

Exercises: count ways to reach step n; compare memoization call counts with naive recursion; write the recurrence before coding.
