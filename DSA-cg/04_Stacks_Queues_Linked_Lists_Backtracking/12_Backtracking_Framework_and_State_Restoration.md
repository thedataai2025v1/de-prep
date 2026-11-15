# Backtracking Framework and State Restoration

**Difficulty:** Medium  
**Topics:** Decision tree, choose–explore–unchoose, pruning

## 1. Concept and when to use

Backtracking explores a search space incrementally. At each step:
1. Choose a candidate.
2. Explore the resulting state recursively.
3. Undo the choice before trying another candidate.

Use it for permutations, combinations, subsets, constraint satisfaction and path search. Worst-case runtime is often exponential; pruning avoids exploring branches that cannot lead to a solution.

## 2. Generic framework

```python
def backtrack(candidates, state, start, results):
    # If state is a complete solution:
    #     results.append(state.copy())
    for i in range(start, len(candidates)):
        choice = candidates[i]
        state.append(choice)                 # choose
        backtrack(candidates, state, i + 1, results)  # explore
        state.pop()                          # unchoose
```

This is a template, not a complete solver; the stopping condition and next-index rule depend on the problem.

## 3. Dry run: subsets of [1,2]

- Start `[]`; choose 1 → `[1]`.
- From `[1]`, choose 2 → `[1,2]`; return and remove 2.
- Return and remove 1.
- Choose 2 → `[2]`.
- Final subsets: `[], [1], [1,2], [2]`.

## 4. Complexity

If there are S solutions and each has length up to n, output itself can require O(Sn) space. Runtime depends on the number of search-tree nodes and the cost per node. Do not report a single complexity without explaining the search-space size.

## 5. Interviewer–candidate discussion

**Interviewer:** What is the difference between backtracking and brute force?

**Candidate:** Backtracking is a structured search that can reject partial candidates early and restore state to explore alternatives. Brute force may enumerate every complete candidate without pruning.

**Interviewer:** Why copy state when saving a solution?

**Candidate:** The working list is mutated during backtracking. Appending the same list object would make previously stored solutions change.

## 6. Edge cases and mistakes

No candidates, empty solution, duplicate candidates, incorrect start index, forgetting `pop()`, saving mutable state without copying, pruning valid branches. Decide whether order matters and whether duplicates are distinct.

## 7. Exercises and tests

Write a solver for subsets, permutations, combinations summing to a target and N-Queens. Verify that each branch restores the state before returning.

Follow-ups: branch-and-bound; memoization versus backtracking; iterative simulation with an explicit stack; estimate search-tree growth.
