# Bitmask Dynamic Programming

**Level:** Hard | **Patterns:** Subset state, bit operations, DP over masks

## 1. Concept and when to use it

Bitmask DP represents a subset of up to roughly 15–20 items (depending on constraints) as an integer. Bit `i` indicates whether item `i` is included. It is useful when the state depends on which items have been selected, not the order in which they were selected.

Typical problems: assignment, traveling-salesperson variants, minimum-cost matching for small `n`, and subset partitioning. The exponential state count is the key constraint.

## 2. Problem: minimum-cost assignment

Given an `n×n` cost matrix, assign each worker exactly one unique job, minimizing total cost.

### Brute force

Try all `n!` permutations of jobs. Time is `O(n!·n)` if costs are summed per permutation.

### Optimized: DP over assigned-job mask

Let `dp[mask]` be the minimum cost after assigning the first `popcount(mask)` workers to the jobs selected by `mask`. The next worker index is `mask.bit_count()`.

```python
def min_assignment_cost(cost):
    n = len(cost)
    if any(len(row) != n for row in cost):
        raise ValueError("cost matrix must be square")
    if n == 0:
        return 0

    size = 1 << n
    inf = float("inf")
    dp = [inf] * size
    dp[0] = 0

    for mask in range(size):
        worker = mask.bit_count()
        if worker == n or dp[mask] == inf:
            continue

        for job in range(n):
            if not (mask & (1 << job)):
                next_mask = mask | (1 << job)
                candidate = dp[mask] + cost[worker][job]
                dp[next_mask] = min(dp[next_mask], candidate)

    return dp[-1]
```

## 3. Dry run

Cost matrix `[[9,2,7], [6,4,3], [5,8,1]]`.
- `mask=000`: worker 0 can take jobs 0, 1, or 2.
- Assign worker 0 to job 1 (cost 2): state `010`.
- Worker 1 then chooses job 0 (cost 6): state `011`, total 8.
- Worker 2 takes job 2 (cost 1): final total 9.
- The best assignment is worker0→job1, worker1→job0, worker2→job2; cost 9.

## 4. Complexity

There are `2^n` masks and up to `n` transitions per mask: `O(n·2^n)` time and `O(2^n)` space. This is exponentially better than `n!` for small `n`, but still does not scale to large `n`.

## 5. Edge cases and common mistakes

- Empty matrix returns zero.
- Validate square dimensions.
- Negative costs are fine for this acyclic state transition.
- `mask.bit_count()` requires a sufficiently recent Python version; use `bin(mask).count("1")` if needed.
- Ensure the worker index matches the number of assigned jobs.
- Do not use bitmask DP when `n` is too large; choose a polynomial algorithm such as the Hungarian algorithm for general assignment.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is the next worker determined by the number of set bits?

**Candidate:** We assign workers in fixed order. Every selected job corresponds to exactly one already-assigned worker, so the subset cardinality equals the next worker index.

**Interviewer:** Why not store the worker index separately?

**Candidate:** It is derivable from the mask, so storing it would duplicate state. A separate index may still be useful if the assignment order is not fixed.

## 7. Follow-ups

- Reconstruct the actual assignment by storing predecessor choices.
- Add forbidden worker-job pairs.
- Solve minimum Hamiltonian path / TSP with a fixed start.
- Partition items into equal-sum groups using bitmask state.
- Explain when Hungarian algorithm is preferable.

## 8. Tests

```python
assert min_assignment_cost([[9, 2, 7], [6, 4, 3], [5, 8, 1]]) == 9
assert min_assignment_cost([]) == 0
assert min_assignment_cost([[4]]) == 4
assert min_assignment_cost([[1, 100], [100, 1]]) == 2
try:
    min_assignment_cost([[1, 2], [3]])
    raise AssertionError("Expected non-square matrix to fail")
except ValueError:
    pass
```

**Data-platform connection:** Bitmask DP can optimize small bounded assignment problems such as assigning a handful of workers to jobs or matching a small set of resources. It is not a replacement for large-scale scheduling or matching algorithms.
