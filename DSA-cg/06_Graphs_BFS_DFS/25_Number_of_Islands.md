# Number of Islands

**Difficulty:** Medium  
**Topics:** Array, Matrix, Graph, DFS, BFS  
**Study-plan section:** Graphs BFS DFS  
**Problem ID:** 25

## 1. Problem Statement

Given a grid of '1' land and '0' water, count connected islands using horizontal and vertical adjacency.

**Example:**
```text
grid = [["1","1","0"],["1","0","0"],["0","0","1"]]
Output = 2
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** For every land cell, explore its connected component using DFS/BFS and mark visited.

```python
def num_islands(grid):
    if not grid: return 0
    m, n = len(grid), len(grid[0])
    seen = set()
    def dfs(r, c):
        if r<0 or r>=m or c<0 or c>=n or grid[r][c] != '1' or (r,c) in seen:
            return
        seen.add((r,c))
        dfs(r+1,c); dfs(r-1,c); dfs(r,c+1); dfs(r,c-1)
    count = 0
    for r in range(m):
        for c in range(n):
            if grid[r][c] == '1' and (r,c) not in seen:
                count += 1; dfs(r,c)
    return count
```

- **Complexity:** O(mn) time; O(mn) visited/recursion space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use iterative DFS or BFS to avoid recursion-depth limits; mark cells when enqueued/pushed.

```python
from collections import deque
def num_islands(grid):
    if not grid: return 0
    m, n = len(grid), len(grid[0])
    seen = set()
    islands = 0
    for r in range(m):
        for c in range(n):
            if grid[r][c] != '1' or (r,c) in seen: continue
            islands += 1
            q = deque([(r,c)]); seen.add((r,c))
            while q:
                x, y = q.popleft()
                for nx, ny in ((x+1,y),(x-1,y),(x,y+1),(x,y-1)):
                    if 0 <= nx < m and 0 <= ny < n and grid[nx][ny] == '1' and (nx,ny) not in seen:
                        seen.add((nx,ny)); q.append((nx,ny))
    return islands
```

- **Complexity:** O(mn) time; O(mn) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use iterative DFS or BFS to avoid recursion-depth limits; mark cells when enqueued/pushed.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Scan row by row. The first unvisited land cell starts an island; BFS marks every horizontally/vertically connected land cell. The isolated land cell later starts the second island.

## 4. Interview Conversation

**Interviewer:** What does one DFS/BFS represent?

**Candidate:** It visits one connected component of land, so I increment the island count once per unvisited land component.

**Interviewer:** When should you mark visited?

**Candidate:** When pushing or enqueuing a cell, not when removing it, to prevent duplicate queue entries.

**Interviewer:** What if the grid is huge?

**Candidate:** Recursive DFS risks stack overflow. Iterative traversal is safer; distributed connected components require iterative graph processing and cross-partition communication.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

8-direction adjacency changes the answer. Mutating the grid can replace a visited set if allowed. Complexity is O(rows×cols).

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Use bounds checks before indexing. Mark on enqueue. Recursion depth can reach O(mn).

Additional checks:
- Minimum valid input size.
- Maximum input size and performance.
- Duplicate values / repeated characters / repeated edges where applicable.
- Boundary values and absent target/result.

## 7. Common Interview Mistakes

- Coding before confirming constraints and output semantics.
- Explaining complexity without stating what `n`, `m`, `k` or other variables represent.
- Ignoring empty inputs, duplicates, boundary conditions or mutation requirements.
- Claiming a hash operation is worst-case O(1); it is generally expected/average O(1).
- Giving an optimized algorithm without explaining its invariant or correctness argument.

## 8. Data Engineering Connection

Connected-component detection appears in entity resolution and graph analytics; distributed components require iterative propagation or specialized graph engines.

## 9. Final Interview Summary

**One-sentence explanation:** Use iterative DFS or BFS to avoid recursion-depth limits; mark cells when enqueued/pushed.

**Complexity to say aloud:** O(mn) time; O(mn) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #25 in the supplied study plan.*