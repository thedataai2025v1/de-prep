# Optimizing Your Algorithms

## 1. Optimization hierarchy

1. Confirm the problem contract; avoid solving a harder variant unnecessarily.
2. Establish a correct baseline.
3. Identify the actual bottleneck.
4. Reduce repeated work.
5. Select a data structure that supports required operations.
6. Exploit ordering, monotonicity or reusable state.
7. Consider memory, constants and implementation complexity.
8. Benchmark only after correctness is established.

## 2. Common transformations

| Baseline | Optimization | Typical change |
|---|---|---|
| Nested pair scan | Hash map | O(n²) → expected O(n) |
| Repeated range sum | Prefix sums | O(n) per query → O(1) after O(n) build |
| Repeated min extraction | Heap | O(n log n) repeated sort → O(log n) per update |
| Repeated substring scan | Sliding window | Often O(n²) → O(n) |
| Repeated recursive states | Memoization / DP | Exponential → polynomial in state count |
| Repeated linear lookup | Set / dictionary | O(n) → expected O(1) |

These transformations depend on constraints and required output semantics.

## 3. Correctness before speed

An optimization is invalid if it changes:
- Duplicate handling.
- Stable ordering.
- Null / missing-value behavior.
- Time-zone or timestamp semantics.
- Error handling.
- Mutation guarantees.

Always compare the optimized solution against a trusted baseline on small randomized inputs when possible.

## 4. Example: repeated membership

```python
def has_common_slow(a, b):
    return any(x in b for x in a)  # O(n*m) when b is a list

def has_common_fast(a, b):
    lookup = set(b)
    return any(x in lookup for x in a)  # expected O(n + m)
```

The set has a construction cost and memory overhead. It is beneficial when the input sizes justify it or membership is repeated.

## 5. Principal-level optimization questions

- Is the bottleneck CPU, memory, I/O, network or serialization?
- Is the workload batch, streaming or interactive?
- Can work be partitioned without changing semantics?
- Does partitioning create skew or expensive shuffles?
- Is caching beneficial, and what is its invalidation strategy?
- What is the operational cost of the optimization?
- How will performance regressions be detected?

## 6. Interview conversation

**Interviewer:** Would you always choose the asymptotically fastest algorithm?

**Candidate:** No. I would compare constraints, constants, memory, implementation risk and maintainability. For tiny inputs, a simple solution may be preferable. For large inputs, the asymptotic improvement can dominate. I would preserve correctness and validate with representative benchmarks.

## Final checklist

- [ ] Baseline is correct.
- [ ] Bottleneck is identified.
- [ ] Optimization matches constraints.
- [ ] Complexity is re-evaluated.
- [ ] Memory and constant factors are considered.
- [ ] Correctness is tested against edge cases.
- [ ] Operational implications are understood.
