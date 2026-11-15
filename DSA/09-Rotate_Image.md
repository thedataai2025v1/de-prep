# Rotate Image

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Matrix |
| **Companies** | Google, Amazon, Microsoft, Apple, Facebook |
| **Interview Frequency** | Medium |

---

## 1. Problem Statement

You are given an `n x n` 2D matrix representing an image. **Rotate the image 90 degrees clockwise in-place.**

**Example:**

```
Input:  matrix = [[1,2,3],[4,5,6],[7,8,9]]
Output: [[7,4,1],[8,5,2],[9,6,3]]
```

**Constraints (LeetCode):**
- `1 <= n <= 20`
- `-1000 <= matrix[i][j] <= 1000`

---

## 2. Approaches

### Approach 1: Transpose Then Reverse Each Row (⭐ Optimal)

**Idea:** A 90-degree clockwise rotation can be achieved in two steps:
1. **Transpose** the matrix (swap `matrix[i][j]` with `matrix[j][i]`) — converts rows to columns.
2. **Reverse each row** — the columns now appear in reversed (clockwise) order.

This works in-place because both operations swap elements within the matrix.

```python
def rotate(matrix):
    n = len(matrix)
    if n == 0:
        return

    # Step 1: Transpose (swap across diagonal)
    for i in range(n):
        for j in range(i + 1, n):
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]

    # Step 2: Reverse each row
    for i in range(n):
        matrix[i].reverse()
```

| Metric | Value |
|---|---|
| **Time** | O(n²) — each element visited twice (once in transpose, once in reverse) |
| **Space** | O(1) — in-place |

**Pros:**
- Simple and intuitive — easy to derive the two-step transformation
- In-place, constant extra space
- Hardly any edge-case logic needed

**Cons:**
- Requires two passes over the matrix
- Reverse step modifies rows in-place (acceptable)

---

### Approach 2: Four-Cell Cyclic Rotation

**Idea:** Rotate elements in groups of four (one per corner). For each layer from outermost inward, rotate the four corresponding cells in a cycle: `top-left → top-right → bottom-right → bottom-left → top-left`. Perform this for each cell along one edge of the current layer.

```python
def rotate(matrix):
    n = len(matrix)
    if n == 0:
        return

    for i in range(n // 2):
        for j in range(i, n - i - 1):
            # Save top-left
            temp = matrix[i][j]

            # Bottom-left → top-left
            matrix[i][j] = matrix[n - 1 - j][i]

            # Bottom-right → bottom-left
            matrix[n - 1 - j][i] = matrix[n - 1 - i][n - 1 - j]

            # Top-right → bottom-right
            matrix[n - 1 - i][n - 1 - j] = matrix[j][n - 1 - i]

            # Saved temp → top-right
            matrix[j][n - 1 - i] = temp
```

| Metric | Value |
|---|---|
| **Time** | O(n²) — each element moved once |
| **Space** | O(1) — in-place |

**Pros:**
- Single pass over the matrix
- Direct rotation without intermediate transformations
- Good for understanding the geometric transformation

**Cons:**
- More complex indexing — easy to make off-by-one errors
- Harder to explain in an interview setting
- Index derivation can be confusing under time pressure

---

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Transpose + Reverse** ⭐ | O(n²) | O(1) | **Interview choice** — simple, memorable |
| **2. Four-Cell Cycle** | O(n²) | O(1) | When a single-pass solution is preferred |

---

## 3. Edge Cases

> **Always mention these in an interview to show thoroughness.**

| Edge Case | Input / Description | Expected Behavior |
|---|---|---|
| Empty matrix | `[]` | Return `[]` |
| 1×1 matrix | `[[5]]` | No change |
| 2×2 matrix | `[[1,2],[3,4]]` | `[[3,1],[4,2]]` |
| All same values | `[[1,1],[1,1]]` | Unchanged (no effect) |
| Single row/col | N/A (must be n×n, so min 1×1) | Already handled |

---

## 4. Interview Talking Points

- **"The transpose + reverse approach is optimal: O(n²) time is required because we must touch every element, and O(1) space by doing it in-place."**
- **Data Engineering Connection:** Matrix rotations model coordinate transformations in geo-spatial data processing (e.g., rotating satellite imagery or map tiles) and image preprocessing pipelines for computer vision.

---

## 5. Interview Follow-Up Questions

**Q1. Can you rotate the matrix 90 degrees counter-clockwise?**
→ Yes. Transpose the matrix, then **reverse each column** instead of each row. Or use the four-cell approach but shift values in the opposite direction.

**Q2. Can you do this without the transpose step?**
→ Use the four-cell cyclic approach directly (Approach 2). It performs rotation in one pass but requires careful index arithmetic.

**Q3. How would you rotate by 180 degrees?**
→ Reverse each row, then reverse each column (or reverse the entire matrix). This is equivalent to `matrix[i][j] → matrix[n-1-i][n-1-j]`.

**Q4. What if the matrix were m×n (rectangular) instead of n×n?**
→ In-place rotation of a rectangular matrix is more complex and typically requires a buffer. LeetCode's Rotate Image constraint specifies n×n specifically.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Transposing with `for j in range(n)` instead of `range(i+1, n)` | Swaps elements twice, reverting the transpose | Start `j` from `i+1` to swap each pair once |
| Forgetting `n // 2` in cyclic approach | Outer loop iterates full matrix, re-rotating inner layers | Only iterate `n // 2` layers |
| Off-by-one in inner loop range | `range(i, n-i)` instead of `range(i, n-i-1)` | Inner j must go to `n-i-1` to avoid last corner |
| Modifying input without checking constraints | Fails on empty or 1×1 matrices | Add early return for edge cases |

---

## 7. Related Problems

| Problem | Key Concept | Difficulty |
|---|---|---|
| [Spiral Matrix](https://leetcode.com/problems/spiral-matrix/) | Matrix traversal in spiral order | Medium |
| [Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes/) | In-place matrix modification | Medium |
| [Transpose Matrix](https://leetcode.com/problems/transpose-matrix/) | Simple matrix transpose | Easy |
| [Image Smoother](https://leetcode.com/problems/image-smoother/) | Convolution-like image transformation | Easy |

---

*Last updated: 2025 — for data engineering interview preparation.*