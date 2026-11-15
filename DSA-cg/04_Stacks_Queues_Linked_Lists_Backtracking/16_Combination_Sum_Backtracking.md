# Combination Sum

**Difficulty:** Medium  
**Topics:** Backtracking, unbounded reuse, pruning with sorted candidates

## 1. Problem statement

Given distinct positive candidate values and a target, return unique combinations whose sum equals target. A candidate may be reused any number of times. Combinations are order-insensitive.

## 2. Approaches

### Baseline

Enumerate sequences up to a safe length and deduplicate after sorting each sequence. This explores permutations of the same combination and is inefficient.

### Optimized: nondecreasing index search

```python
def combination_sum(candidates, target):
    if target < 0 or any(value <= 0 for value in candidates):
        raise ValueError("target must be non-negative and candidates positive")

    candidates = sorted(set(candidates))
    result, path = [], []

    def backtrack(start, remaining):
        if remaining == 0:
            result.append(path.copy())
            return
        for i in range(start, len(candidates)):
            value = candidates[i]
            if value > remaining:
                break
            path.append(value)
            backtrack(i, remaining - value)  # i allows reuse
            path.pop()

    backtrack(0, target)
    return result
```

Runtime is output/search-tree dependent and can be exponential. Auxiliary space is O(target/min_candidate) for recursion/path; output is additional.

## 3. Dry run

Candidates `[2,3,6,7]`, target 7:
- Choose 2; remaining 5. Choose 2; remaining 3. Choose 2 would exceed; choose 3 → `[2,2,3]`.
- Backtrack to root; choose 3 and continue, but no exact completion.
- Choose 6; cannot complete.
- Choose 7 → `[7]`.
Results: `[[2,2,3],[7]]`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why recurse with index i rather than i+1?

**Candidate:** Reuse is allowed, so the same candidate can be selected again. Using i+1 would enforce each candidate at most once.

**Interviewer:** Why sort candidates?

**Candidate:** Once a candidate exceeds the remaining target, all later candidates are also too large, so we can break and prune.

## 5. Edge cases and mistakes

Target zero returns `[[]]`; negative target; zero or negative candidates would cause nontermination or invalid assumptions; duplicates in input; confusion between reusable and single-use variants.

## 6. Tests and variations

```python
assert {tuple(x) for x in combination_sum([2, 3, 6, 7], 7)} == {(2, 2, 3), (7,)}
assert combination_sum([2], 1) == []
assert combination_sum([2, 3], 0) == [[]]
```

Follow-ups: Combination Sum II (each input used once, duplicates skipped); count combinations using DP; return minimum number of candidates; impose a maximum combination length.
