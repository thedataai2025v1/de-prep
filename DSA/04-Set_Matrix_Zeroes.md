# Set Matrix Zeroes

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Matrix, In-place Algorithms |
| **Companies** | Amazon, Google, Microsoft |
| **Interview Frequency** | Medium-High |

---

## 1. Problem Statement

Given an `m x n` matrix, if an element is `0`, set its entire row and column to `0`. Do it **in place**.

**Example:**
```
Input:
[
  [1, 1, 1],
  [1, 0, 1],
  [1, 1, 1]
]
Output:
[
  [1, 0, 1],
  [0, 0, 0],
  [1, 0, 1]
]
```

---

## 2. Approaches

### Approach 1: First Row and First Column as Markers (⭐ Optimal)

Use the first row and first column of the matrix itself as markers. If `matrix[i][j] == 0`, mark `matrix[i][0]` and `matrix[0][j]` as `0`. Two separate booleans track whether the first row and first column themselves need to be zeroed. Finally, iterate through the matrix again (skipping row 0 and column 0) and zero out based on markers. Then handle the first row and column.

**Pros:**
- True O(1) extra space — uses the matrix itself as storage
- Two clean passes — first to mark, second to apply
- Works for any m x n matrix
- Elegant use of existing memory — impressive in interviews

**Cons:**
- Two flags needed for the first row/column — easy to forget
- Modifies the matrix in place — destroys the original data
- More complex logic than the separate-arrays approach — harder to implement under pressure
- Order of operations matters: must read markers BEFORE overwriting

```python
def setZeroes(matrix: list[list[int]]) -> None:
    m, n = len(matrix), len(matrix[0])
    first_row_zero = any(matrix[0][j] == 0 for j in range(n))
    first_col_zero = any(matrix[i][0] == 0 for i in range(m))

    # Mark first row/col
    for i in range(1, m):
        for j in range(1, n):
            if matrix[i][j] == 0:
                matrix[i][0] = 0
                matrix[0][j] = 0

    # Zero out rows/cols
    for i in range(1, m):
        for j in range(1, n):
            if matrix[i][0] == 0 or matrix[0][j] == 0:
                matrix[i][j] = 0

    # Handle first row
    if first_row_zero:
        for j in range(n):
            matrix[0][j] = 0

    # Handle first column
    if first_col_zero:
        for i in range(m):
            matrix[i][0] = 0
```

| Metric | Value |
|---|---|
| **Time** | O(m × n) |
| **Space** | O(1) |

### Approach 2: Separate Row and Column Sets (Alternative)

Use two sets: one for rows that need to be zeroed, one for columns. Iterate the matrix once to collect zeros, then iterate again to apply the zeroes.

**Pros:**
- Very clear and easy to understand
- Simple two-pass logic that's hard to get wrong
- No tricky edge cases with first row/column

**Cons:**
- Uses O(m + n) extra space for the sets
- Two separate data structures to maintain
- Doesn't meet the strict O(1) space constraint
- Extra overhead of set operations (hashing)

```python
def setZeroes(matrix: list[list[int]]) -> None:
    m, n = len(matrix), len(matrix[0])
    zero_rows = set()
    zero_cols = set()

    # Identify zero positions
    for i in range(m):
        for j in range(n):
            if matrix[i][j] == 0:
                zero_rows.add(i)
                zero_cols.add(j)

    # Apply zeroes
    for i in zero_rows:
        for j in range(n):
            matrix[i][j] = 0
    for j in zero_cols:
        for i in range(m):
            matrix[i][j] = 0
```

| Metric | Value |
|---|---|
| **Time** | O(m × n) |
| **Space** | O(m + n) |

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. First Row/Col as Markers** | O(m × n) | O(1) | Interviews — optimal space |
| **2. Sets for Rows/Cols** | O(m × n) | O(m + n) | Quick implementation, readability |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| All zeros | Entire matrix becomes zeros — both approaches handle it |
| No zeros | No changes made |
| First row or first column has zeros | Boolean flags handle this correctly |
| 1 × 1 matrix with zero | Sets the single element to 0 |
| Row or column is entirely zeros | Entire cross of row + column becomes zeros |

---

## 4. Interview Talking Points

- "I use the first row and first column as markers to avoid using extra arrays. This is a classic space optimization technique where we repurpose existing memory to store state."
- "In data engineering, this is similar to using a bitmap for sparse data representation. Instead of storing which rows/cols are zeroed, we store the information in-place, reducing memory overhead significantly for large matrices — important in distributed systems where memory is expensive."

---

## 5. Interview Follow-Up Questions

**Q1: What if we can't modify the first row and first column of the matrix?**
-> Use two separate boolean flags for the first row and first column, and use rows 1..m-1 and columns 1..n-1 as markers. This is essentially what we already do with `first_row_zero` and `first_col_zero`.

**Q2: How would this change for a sparse matrix (mostly zeros)?**
-> For sparse matrices, the set-based approach is better: only store the rows and columns that actually need zeroing. You could also use a sparse matrix representation (e.g., COO, CSR) to avoid processing non-zero elements.

**Q3: Can this be done in a single pass?**
-> Not in general — you need at least two passes: one to identify which rows/cols to zero, and one to apply the changes. Unless you delay all writes, which is equivalent to a second pass.

**Q4: What if the matrix is very large (doesn't fit in memory)?**
-> Process it in chunks/blocks. Store row and column zero markers in a separate data structure or on disk. This is similar to how databases handle external sorting.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Zeroing immediately when a 0 is found | Subsequent cells become 0, causing a chain reaction — the whole matrix becomes zeros | Collect markers first, then apply zeroes in a second pass |
| Forgetting the first row/column booleans | If the first row/col has zeros, they get overwritten as markers before being processed | Save `first_row_zero` and `first_col_zero` before using row/col 0 as markers |
| Iterating from 0 in the marking pass | `matrix[0][0]` is used as a marker for both row 0 and column 0 — ambiguity | Mark from index 1, handle 0 separately with the boolean flags |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Product of Array Except Self](DSA/02-Product_of_Array_Except_Self.md) | Uses prefix/suffix pattern for similar elimination logic |
| [Rotate Image](https://leetcode.com/problems/rotate-image/) | Another in-place matrix transformation |
| [Spiral Matrix](https://leetcode.com/problems/spiral-matrix/) | Matrix traversal pattern |

---

*Last updated: 2025 — for data engineering interview preparation.*
