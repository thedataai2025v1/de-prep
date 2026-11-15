# Dynamic Programming Optimization and Interview Problem Solving

**Difficulty:** Medium–Hard | **Pattern:** State reduction, transition optimization

## A repeatable interview method

1. Start with a recursive definition and small examples.
2. Identify repeated states; memoize to validate the recurrence.
3. Convert to tabulation when dependency order is clear.
4. Reduce memory only after identifying exactly which previous states are needed.
5. Look for faster transitions: prefix sums, monotonic queues, binary search, bitsets, or data structures.
6. Explain whether the complexity is polynomial, pseudo-polynomial or exponential.

## Example: maximum sum with no adjacent elements

The classic take/skip recurrence is O(n) time and O(1) space:

```python
def max_non_adjacent_sum(nums):
    two_back = one_back = 0
    for value in nums:
        two_back, one_back = one_back, max(one_back, two_back + value)
    return one_back
```

A table implementation stores every state in O(n) space; rolling variables are sufficient because each state depends only on the prior two.

## Dry run

For `[3,2,7,10]`: best values progress 3,3,10,13. The optimal selection is 3+10.

## Optimization patterns to recognize

- **Rolling array:** recurrence depends on a fixed number of previous rows/states.
- **Prefix sums:** replace repeated range summation.
- **Monotonic queue/deque:** optimize sliding-window min/max transitions.
- **Binary search on answer:** feasibility is monotonic in a candidate answer.
- **Bitset:** accelerate subset-sum reachability using machine-word parallelism.
- **Coordinate compression/Fenwick tree:** optimize DP transitions over ordered values.
- **Divide-and-conquer or convex-hull optimization:** apply only when recurrence structure and proof conditions support it.

## Interviewer–candidate discussion

**Interviewer:** When is O(n × target) not polynomial in input size?

**Candidate:** If target is represented in binary, its numeric value may be exponential in the number of bits used to encode it. Such an algorithm is pseudo-polynomial.

**Interviewer:** Should you optimize space immediately?

**Candidate:** No. First establish a correct recurrence and table. Then prove that discarded states are never referenced again.

## Common mistakes

Unclear state meaning; missing base cases; wrong loop direction for 0/1 vs unbounded choice; using a greedy rule without proof; overlooking output reconstruction; claiming O(n) when each transition scans O(n).

## Practical exercises

1. Convert Coin Change memoization into tabulation.
2. Reduce LCS from O(mn) space to O(min(m,n)).
3. Explain why 0/1 Knapsack iterates capacity downward.
4. Implement LIS in O(n log n) and reconstruct one sequence.
5. Compare Word Break DP with trie-based search.
6. Identify a pseudo-polynomial algorithm and explain the encoding-size issue.

## Test

```python
assert max_non_adjacent_sum([3, 2, 7, 10]) == 13
assert max_non_adjacent_sum([]) == 0
assert max_non_adjacent_sum([5]) == 5
assert max_non_adjacent_sum([2, 1, 1, 2]) == 4
```
