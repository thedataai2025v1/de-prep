# Jump Game I and II

**Level:** Medium | **Patterns:** Greedy reachability, frontier expansion

## 1. Concept and when to use it

Given `nums[i]`, the maximum forward jump from index `i`, Jump Game I asks whether the last index is reachable. Jump Game II asks for the minimum number of jumps, assuming the end is reachable (or asks you to detect impossibility in a generalized version).

This is a useful example of how a greedy invariant can replace an explicit graph search.

## 2. Approaches

### Brute force / graph search

Treat each index as a node with edges to every reachable next index. BFS finds the minimum jumps in `O(n²)` worst-case time because there can be quadratically many edges. For reachability alone, DFS/BFS also costs `O(n²)`.

### Optimized Jump Game I

Track the farthest index reachable so far. If the current index exceeds that frontier, it is unreachable.

```python
def can_jump(nums):
    farthest = 0
    for i, jump in enumerate(nums):
        if i > farthest:
            return False
        farthest = max(farthest, i + jump)
        if farthest >= len(nums) - 1:
            return True
    return len(nums) <= 1
```

### Optimized Jump Game II

Process the current reachable layer. `current_end` is the end of the range reachable with the current number of jumps; `farthest` is the next layer's boundary.

```python
def min_jumps(nums):
    n = len(nums)
    if n <= 1:
        return 0

    jumps = 0
    current_end = 0
    farthest = 0

    for i in range(n - 1):
        farthest = max(farthest, i + nums[i])
        if i == current_end:
            if farthest == current_end:
                raise ValueError("Last index is unreachable")
            jumps += 1
            current_end = farthest
            if current_end >= n - 1:
                break

    return jumps
```

## 3. Dry run

`[2, 3, 1, 1, 4]`:
- Initial frontier `0`.
- At index 0, reach 2; finish current layer, increment jumps to 1, next frontier 2.
- At index 1, farthest becomes 4.
- At index 2, current layer ends; increment jumps to 2, frontier reaches final index.
- Answer: 2.

## 4. Complexity

Both greedy algorithms are `O(n)` time and `O(1)` auxiliary space. BFS may require `O(n²)` edge work and `O(n)` queue/visited space.

## 5. Edge cases and mistakes

- Empty list: define behavior explicitly; `can_jump([])` above returns `True` by convention.
- One element requires zero jumps.
- Zeros can create an unreachable frontier.
- Jump lengths are non-negative by problem definition.
- Jump Game II commonly guarantees reachability; if not, detect it rather than returning a misleading count.
- Do not increment the jump count at every index—increment only when the current frontier is exhausted.

## 6. Interviewer–candidate discussion

**Interviewer:** Why does the layer-boundary approach give the minimum number of jumps?

**Candidate:** All indices in a layer are reachable using the same number of jumps. Scanning the whole layer computes the farthest next-layer boundary. This is equivalent to BFS by levels, but avoids materializing edges.

**Interviewer:** What is the invariant?

**Candidate:** Before crossing `current_end`, `farthest` is the maximum index reachable with one additional jump from any index in the current layer.

## 7. Follow-ups

- Return one actual shortest path of indices.
- Count the number of distinct shortest jump sequences.
- If jumps can go both forward and backward, this greedy invariant no longer applies; use graph search.
- If each position has a cost, formulate shortest path / DP.

## 8. Tests

```python
assert can_jump([2, 3, 1, 1, 4]) is True
assert can_jump([3, 2, 1, 0, 4]) is False
assert can_jump([0]) is True
assert can_jump([1, 0]) is True
assert min_jumps([2, 3, 1, 1, 4]) == 2
assert min_jumps([0]) == 0
try:
    min_jumps([1, 0, 1])
    raise AssertionError("Expected unreachable input to fail")
except ValueError:
    pass
```

**Data-platform connection:** A sequence of deployment stages or retry transitions can be modeled as reachability. If transitions have arbitrary costs or can move backward, use shortest-path algorithms rather than this specialized greedy solution.
