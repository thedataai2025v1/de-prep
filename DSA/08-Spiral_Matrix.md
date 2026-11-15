# Spiral Matrix

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Matrix, Simulation |
| **Companies** | Google, Amazon, Microsoft, Apple, Bloomberg |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an `m x n` matrix, return **all elements of the matrix in spiral order** (clockwise: top-left, go right, down, left, up, repeat inward).

**Example:**

```
Input:  matrix = [[1,2,3],[4,5,6],[7,8,9]]
Output: [1,2,3,6,9,8,7,4,5]
```

**Constraints (LeetCode):**
- `1 <= m, n <= 10`
- `-100 <= matrix[i][j] <= 100`

---

## 2. Approaches

### Approach 1: Layer-by-Layer Traversal with Boundaries (⭐ Optimal)

**Idea:** Maintain four boundaries (`top`, `bottom`, `left`, `right`) that shrink as we traverse each layer. Move right across `top` row, then down the `right` column, then left across `bottom` row (if still valid), then up the `left` column (if still valid). Repeat until boundaries cross.

```python
def spiralOrder(matrix):
    if not matrix or not matrix[0]:
        return []

    res = []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1

    while top <= bottom and left <= right:
        # Move right across top row
        for col in range(left, right + 1):
            res.append(matrix[top][col])
        top += 1

        # Move down the right column
        for row in range(top, bottom + 1):
            res.append(matrix[row][right])
        right -= 1

        # Move left across bottom row (if rows remain)
        if top <= bottom:
            for col in range(right, left - 1, -1):
                res.append(matrix[bottom][col])
            bottom -= 1

        # Move up the left column (if columns remain)
        if left <= right:
            for row in range(bottom, top - 1, -1):
                res.append(matrix[row][left])
            left += 1

    return res
```

| Metric | Value |
|---|---|
| **Time** | O(m × n) — each element visited exactly once |
| **Space** | O(1) — excluding output array |

**Pros:**
- Intuitive layer-by-layer logic
- Constant extra space (only pointers)
- Handles all matrix shapes uniformly

**Cons:**
- Boundary conditions must be carefully checked after each direction change
- Slightly verbose but very clear

---

### Approach 2: Direction Vector with Visited Set

**Idea:** Use a direction array (`right → down → left → up → right…`) and a visited boolean matrix. Change direction when hitting a boundary or a visited cell. Continue until all cells are visited.

```python
def spiralOrder(matrix):
    if not matrix or not matrix[0]:
        return []

    m, n = len(matrix), len(matrix[0])
    visited = [[False] * n for _ in range(m)]
    res = []

    # Direction vectors: right, down, left, up
    dr = [0, 1, 0, -1]
    dc = [1, 0, -1, 0]

    r = c = d = 0

    for _ in range(m * n):
        res.append(matrix[r][c])
        visited[r][c] = True

        nr, nc = r + dr[d], c + dc[d]
        if nr < 0 or nr >= m or nc < 0 or nc >= n or visited[nr][nc]:
            d = (d + 1) % 4  # change direction
            nr, nc = r + dr[d], c + dc[d]

        r, c = nr, nc

    return res
```

| Metric | Value |
|---|---|
| **Time** | O(m × n) — each element visited once |
| **Space** | O(m × n) — visited matrix |

**Pros:**
- Elegant direction-switching pattern
- Easy to extend to other spiral patterns (e.g., counter-clockwise)

**Cons:**
- Requires O(m × n) extra space for visited tracking
- Slightly less intuitive than boundary approach

---

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Layer-by-Layer (Boundaries)** ⭐ | O(m × n) | O(1) | **Interview choice** — minimal space |
| **2. Direction Vector + Visited** | O(m × n) | O(m × n) | Situations where directional logic is preferred |

---

## 3. Edge Cases

> **Always mention these in an interview to show thoroughness.**

| Edge Case | Input / Description | Expected Behavior |
|---|---|---|
| Empty matrix | `[]` | Return `[]` |
| Single row | `[[1,2,3,4]]` | Return `[1,2,3,4]` (just go right) |
| Single column | `[[1],[2],[3]]` | Return `[1,2,3]` |
| 1×1 matrix | `[[5]]` | Return `[5]` |
| 2×2 matrix | `[[1,2],[3,4]]` | Return `[1,2,4,3]` |
| 3×4 rectangular | `[[1,2,3,4],[5,6,7,8],[9,10,11,12]]` | Return `[1,2,3,4,8,12,11,10,9,5,6,7]` |

---

## 4. Interview Talking Points

- **"I can solve this in O(m×n) time with O(1) extra space using four boundary pointers. This is optimal because each element must be visited at least once."**
- **Data Engineering Connection:** Spiral traversal mirrors sequential I/O patterns when processing multi-dimensional batch data — reading data in a structured sweep across partitions, or iterating over time-series blocks stored in a grid layout.

---

## 5. Interview Follow-Up Questions

**Q1. What if the matrix is rectangular (not square)?**
→ The boundary approach handles any m×n shape naturally. The `top <= bottom` and `left <= right` guards after each direction ensure correct behavior on non-square matrices.

**Q2. How would you generate a spiral matrix (fill values in spiral order)?**
→ Same boundary approach but assign values 1..m×n in spiral order instead of reading them. Initialize a matrix with zeros, then follow the same traversal assigning incrementing values.

**Q3. What if the spiral direction were counter-clockwise?**
→ Adjust the order: move down first, then right, then up, then left. Swap row and column iteration directions accordingly.

**Q4. Can you do this without the extra column/row guard checks?**
→ The `if top <= bottom` and `if left <= right` guards are essential for rectangular matrices. Without them, a single-row matrix would double-count elements when traversing back left.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Forgetting `if top <= bottom` before left traversal | On single-row matrices, re-traverses the same row backward | Always check boundaries before each direction |
| Forgetting `if left <= right` before up traversal | On single-column matrices, re-traverses the same column upward | Add the guard before the up loop |
| Off-by-one in boundary updates | Using `range(left, right)` instead of `range(left, right + 1)` misses corner elements | Inclusive ranges: `right + 1` and `left - 1` |
| Not handling empty matrix input | `matrix[0]` throws IndexError | Check `if not matrix or not matrix[0]` |
| Using list as visited flag | Lists are mutable and unhashable | Use a boolean matrix or set of tuples |

---

## 7. Related Problems

| Problem | Key Concept | Difficulty |
|---|---|---|
| [Spiral Matrix II](https://leetcode.com/problems/spiral-matrix-ii/) | Generate n×n matrix in spiral order | Medium |
| [Spiral Matrix III](https://leetcode.com/problems/spiral-matrix-iii/) | Spiral traversal starting from center | Medium |
| [Rotate Image](https://leetcode.com/problems/rotate-image/) | Rotate matrix 90 degrees clockwise in-place | Medium |
| [Matrix Diagonal Sum](https://leetcode.com/problems/matrix-diagonal-sum/) | Sum primary and secondary diagonals | Easy |
| [Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes/) | In-place matrix modification with markers | Medium |

---

*Last updated: 2025 — for data engineering interview preparation.*