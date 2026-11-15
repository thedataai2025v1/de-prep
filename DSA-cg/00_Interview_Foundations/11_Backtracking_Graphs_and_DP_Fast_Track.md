# Fast Track: Backtracking, Graphs and Dynamic Programming

## 1. Backtracking

Use when exploring a decision tree of candidates:
1. Choose.
2. Recurse.
3. Undo the choice.

Typical problems: subsets, permutations, combination sum, Sudoku. Complexity is often exponential; explain pruning and output-size lower bounds.

## 2. Graphs

Representations:
- Adjacency list: O(V + E) space; efficient for sparse graphs.
- Adjacency matrix: O(V²) space; O(1) edge lookup.

Core algorithms:
- BFS: shortest path in unweighted graphs, O(V + E).
- DFS: reachability, components, cycle detection.
- Topological sort: dependency ordering in a DAG.
- Union-Find: connectivity and cycle detection in undirected graphs.

Clarify directed/undirected, weighted/unweighted, disconnected and cyclic cases.

## 3. Dynamic programming

DP is useful when a problem has:
- Overlapping subproblems.
- Optimal substructure / reusable state.

A disciplined derivation:
1. Define the state precisely.
2. Write the recurrence.
3. Identify base cases.
4. Choose evaluation order.
5. Determine whether state compression is valid.
6. Analyze time and space.

Example Fibonacci: `dp[i] = dp[i-1] + dp[i-2]`; O(n) time, O(1) space with rolling variables.

## 4. Pattern comparison

| Problem clue | Likely technique |
|---|---|
| Enumerate valid combinations | Backtracking |
| Reachability / connected components | BFS or DFS |
| Dependency ordering | Topological sort |
| Repeated subproblems with recurrence | DP |
| Connectivity under edge additions | Union-Find |

## 5. Common mistakes

- Using DFS for shortest path in an unweighted graph without tracking levels.
- Forgetting to undo state in backtracking.
- Defining DP state too vaguely.
- Memoizing a state that omits a variable affecting future decisions.
- Assuming every recursive problem is DP.

## Practice sequence

Number of Islands → Course Schedule → Subsets → Generate Parentheses → Climbing Stairs → Coin Change → Knapsack.
