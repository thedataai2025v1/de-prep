# Gas Station

**Level:** Medium | **Patterns:** Greedy elimination, prefix balance

## 1. Concept and when to use it

There are `n` circular stations. `gas[i]` is fuel gained at station `i`; `cost[i]` is fuel needed to travel to the next station. Return a start index that completes the circuit, or `-1`.

The core insight: if starting at station `start` and failing at station `i`, no station between `start` and `i` can be a valid start.

## 2. Approaches

### Brute force

Try every station as a start and simulate the full circuit. This costs `O(n²)` time and `O(1)` space.

### Optimized greedy

Track the total net fuel and the current segment's tank. When the tank becomes negative at `i`, reset the candidate start to `i + 1`: every candidate in the failed segment is impossible. If total net fuel is non-negative, a solution exists.

```python
def can_complete_circuit(gas, cost):
    if len(gas) != len(cost):
        raise ValueError("gas and cost must have equal lengths")
    if not gas:
        return -1

    total = 0
    tank = 0
    start = 0

    for i, (fuel, required) in enumerate(zip(gas, cost)):
        gain = fuel - required
        total += gain
        tank += gain

        if tank < 0:
            start = i + 1
            tank = 0

    return start if total >= 0 else -1
```

## 3. Dry run

`gas = [1,2,3,4,5]`, `cost = [3,4,5,1,2]`.
Net gains: `[-2,-2,-2,3,3]`.
- At index 0, tank falls below zero; candidate becomes 1.
- At index 1, tank falls below zero; candidate becomes 2.
- At index 2, tank falls below zero; candidate becomes 3.
- Index 3 adds 3; index 4 adds 3.
- Total is 0, so start index 3 works.

## 4. Complexity

`O(n)` time and `O(1)` auxiliary space.

## 5. Edge cases and mistakes

- Unequal array lengths should be rejected.
- Empty arrays: this implementation returns `-1`.
- Total gas below total cost means no solution.
- Multiple valid starts may exist; any valid start is accepted.
- Reset the local tank when moving the candidate start.
- Do not return the candidate without checking total balance.

## 6. Interviewer–candidate discussion

**Interviewer:** Why can you discard all starts between the candidate and the failure point?

**Candidate:** Starting from the candidate, the cumulative net fuel first becomes negative at the failure point. Any later start within that segment has even less accumulated balance available before reaching the same failure boundary, so it cannot complete the segment either.

**Interviewer:** Is total gas at least total cost sufficient by itself?

**Candidate:** It guarantees that some circular start exists, but it does not identify that start. The greedy scan finds a valid candidate.

## 7. Follow-ups

- Return all valid starting stations.
- Return the minimum initial fuel required for each start.
- Add fuel consumption uncertainty: the deterministic greedy proof may no longer hold.
- Explain the prefix-sum interpretation on a circular array.

## 8. Tests

```python
assert can_complete_circuit([1, 2, 3, 4, 5], [3, 4, 5, 1, 2]) == 3
assert can_complete_circuit([2, 3, 4], [3, 4, 3]) == -1
assert can_complete_circuit([5], [4]) == 0
assert can_complete_circuit([1], [1]) == 0
assert can_complete_circuit([], []) == -1
try:
    can_complete_circuit([1, 2], [1])
    raise AssertionError("Expected mismatched arrays to fail")
except ValueError:
    pass
```

**Data-platform connection:** This resembles cumulative resource-budget feasibility around a circular sequence of processing stages. For real workloads, resources may be multidimensional and time-varying, requiring a richer model.
