# Number of Islands

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, DFS, BFS, Union Find, Grid |
| **Companies** | Google, Meta, Amazon, Microsoft, Uber, Airbnb |
| **Interview Frequency** | Very High |

---

## 1. Problem Statement
Given an `m x n` 2D binary grid `grid` where `1` represents land and `0` represents water, count the number of islands. An island is formed by connecting adjacent lands horizontally or vertically. Assume all four edges of the grid are surrounded by water.

**Example:**
```
Input: grid = [
  ["1","1","1","1","0"],
  ["1","1","0","1","0"],
  ["1","1","0","0","0"],
  ["0","0","1","0","0"]
]
Output: 3
```

---

## 2. Approaches

### Approach 1: DFS (Depth-First Search)
Iterate through every cell. When you find an unvisited land cell (`"1"`), start a DFS to sink the entire island — mark all connected land cells as visited (change to `"0"`). Each DFS call increments the island count by 1.

**Pros:**
- Simple and concise — the DFS subroutine is only 3-4 lines
- Natural for "flood fill" style problems — easy to generalize to 8-directional connectivity
- Modifies the grid in-place — no extra visited set needed

**Cons:**
- Mutates the input grid — may be unacceptable if the caller needs the original data
- O(m × n) recursion depth in the worst case (a snake-shaped island) — can stack overflow for very large grids

```python
def numIslands(grid: list[list[str]]) -> int:
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])

    def dfs(r: int, c: int) -> None:
        # Mark current cell as visited by sinking it
        grid[r][c] = "0"
        # Explore all 4 directions
        for dr, dc in [(1, 0), (-1, 0), (0, 1), (0, -1)]:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == "1":
                dfs(nr, nc)

    count = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == "1":
                dfs(r, c)
                count += 1

    return count
```

| Metric | Value |
|---|---|
| **Time** | O(m × n) — each cell visited at most once |
| **Space** | O(m × n) — worst-case DFS recursion depth |

### Approach 2: BFS (Breadth-First Search)
Same idea as DFS but use a queue instead of recursion. When you find a land cell, enqueue it and process neighbors iteratively — marking them as visited when dequeued.

**Pros:**
- No recursion depth limit — uses a queue on the heap, safe for arbitrarily large grids
- Still modifies grid in-place — no extra visited set needed
- Intuitive "wave" expansion — easier to visualize as flood propagation

**Cons:**
- Queue can hold up to O(m × n) elements in the worst case (a full row of an island boundary)
- Slightly more boilerplate than DFS — need to manage an explicit queue and coordinate pairs

```python
from collections import deque

def numIslands(grid: list[list[str]]) -> int:
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    count = 0

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == "1":
                # Start BFS to sink the entire island
                grid[r][c] = "0"
                queue = deque([(r, c)])
                while queue:
                    cr, cc = queue.popleft()
                    for dr, dc in [(1, 0), (-1, 0), (0, 1), (0, -1)]:
                        nr, nc = cr + dr, cc + dc
                        if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == "1":
                            grid[nr][nc] = "0"
                            queue.append((nr, nc))
                count += 1

    return count
```

| Metric | Value |
|---|---|
| **Time** | O(m × n) — each cell visited at most once |
| **Space** | O(min(m, n)) — queue width at most the shorter dimension |

### Approach 3: Union-Find (Disjoint Set)
Treat each land cell as an element in a disjoint set. For every land cell, union it with its right and bottom neighbors if they are also land. The number of distinct sets equals the number of islands.

**Pros:**
- Does not mutate the input grid — preserves the original data
- Naturally extensible to dynamic grids — can add/remove cells and update the count incrementally
- Path compression + union by rank gives near-constant amortized time per operation

**Cons:**
- Significantly more boilerplate — need a full Union-Find class with find/union methods
- Higher constant factors — path compression and union by rank add overhead vs simple BFS/DFS
- O(m × n) space for the parent and rank arrays — heavier memory footprint than in-place DFS/BFS

```python
def numIslands(grid: list[list[str]]) -> int:
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    parent = {}  # (r, c) -> parent node
    rank = {}
    island_count = 0

    def find(cell: tuple) -> tuple:
        if parent[cell] != cell:
            parent[cell] = find(parent[cell])  # path compression
        return parent[cell]

    def union(cell1: tuple, cell2: tuple) -> bool:
        root1, root2 = find(cell1), find(cell2)
        if root1 != root2:
            # Union by rank
            if rank[root1] > rank[root2]:
                parent[root2] = root1
            elif rank[root1] < rank[root2]:
                parent[root1] = root2
            else:
                parent[root2] = root1
                rank[root1] += 1
            return True  # merged two separate sets
        return False

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == "1":
                cell = (r, c)
                parent[cell] = cell
                rank[cell] = 0
                island_count += 1
                # Union with right neighbor
                if c + 1 < cols and grid[r][c + 1] == "1":
                    if union(cell, (r, c + 1)):
                        island_count -= 1
                # Union with bottom neighbor
                if r + 1 < rows and grid[r + 1][c] == "1":
                    if union(cell, (r + 1, c)):
                        island_count -= 1

    return island_count
```

| Metric | Value |
|---|---|
| **Time** | O(m × n × α(m × n)) — nearly linear; α is the inverse Ackermann function |
| **Space** | O(m × n) — parent and rank arrays |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| Empty grid `[]` | Returns `0` |
| All water `[["0","0"]]` | Returns `0` |
| All land `[["1","1"]]` | Returns `1` |
| Single cell `["1"]` | Returns `1` |
| Single cell `["0"]` | Returns `0` |
| Checkerboard pattern | Each isolated land cell is a separate island |
| Island touching all 4 borders | Counts as 1 island |
| Grid with 1 column or 1 row | Works — neighbor checks stay within bounds |

---

## 4. Interview Talking Points
- "This is the canonical **connected components** problem on a grid. The same pattern applies to flood fill in image processing, region detection, and graph connectivity analysis."
- "DE connection: Connected components on grids are used in **geospatial analytics** (counting contiguous regions in satellite imagery, flood detection maps), **anomaly detection in time-series heatmaps** (finding contiguous blocks of anomalous readings), and **cluster detection** in spatial data."
- "The DFS approach is the most common and easiest to implement. BFS avoids recursion limits. Union-Find is useful when the grid changes dynamically — you can add/remove cells and update the count in near-constant time."
- "Marking visited cells by sinking them (`grid[r][c] = "0"`) is an **in-place optimization** that avoids a separate visited set. This is a standard trick for grid problems where you can modify the input."

---

## 5. Interview Follow-Up Questions
**Q1. How would you handle 8-directional connectivity (including diagonals)?**
-> Add 4 more direction pairs to the neighbor list: `[(1,1), (1,-1), (-1,1), (-1,-1)]`. The algorithm structure stays identical.

**Q2. How would you count the size of each island instead of just the number?**
-> Track the size during the DFS/BFS traversal: increment a local counter for each cell visited, then append/store it when the traversal ends.

**Q3. How does this scale to very large grids (e.g., satellite imagery)?**
-> Process the grid in tiles/chunks using a parallel or distributed approach (e.g., Spark). Use Union-Find for incremental merging across tile boundaries. For real-time streaming, maintain a Union-Find structure and process events as they arrive.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Not marking cells as visited | Infinite recursion or double-counting — the same island is revisited | Always mark when you visit: `grid[r][c] = "0"` or use a separate visited set |
| Marking too late in BFS (after dequeuing) | The same cell gets added to the queue multiple times through different paths | Mark cells when you **enqueue** them, not when you dequeue |
| Off-by-one in boundary checks | IndexError when checking neighbors at the grid edge | Use `0 <= nr < rows and 0 <= nc < cols` |
| Union-Find: not checking both directions | Misses vertical or horizontal connections | Check at least 2 directions (right + bottom) to cover all adjacencies without double-checking |
| Not handling empty grid | IndexError on `len(grid[0])` when grid is `[]` | Add guard clause: `if not grid: return 0` |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| Max Area of Island | Same traversal — track area instead of counting |
| Pacific Atlantic Water Flow | Bidirectional BFS/DFS from ocean edges |
| Surrounded Regions | DFS from boundary to mark non-surrounded regions |
| Rotting Oranges | BFS with levels (minutes) on a grid |
| Clone Graph | Same traversal pattern but on a general graph |
| Redundant Connection | Union-Find application on a graph |

---

*Last updated: 2025*
