# N-Queens and Constraint Backtracking

**Difficulty:** Hard  
**Topics:** Constraint satisfaction, pruning, bitmasks

## 1. Problem statement

Place n queens on an n×n chessboard so that no two queens share a row, column or diagonal. Return all valid boards.

## 2. Approaches

### Brute force

Try all ways to place n queens among n² squares and validate conflicts. The raw search space is enormous.

### Optimized: one queen per row with column/diagonal sets

```python
def solve_n_queens(n):
    if n < 0:
        raise ValueError("n must be non-negative")
    if n == 0:
        return [[]]

    board = [["."] * n for _ in range(n)]
    columns = set()
    diag_down = set()  # row - col
    diag_up = set()    # row + col
    results = []

    def backtrack(row):
        if row == n:
            results.append(["".join(line) for line in board])
            return

        for col in range(n):
            d1, d2 = row - col, row + col
            if col in columns or d1 in diag_down or d2 in diag_up:
                continue

            columns.add(col)
            diag_down.add(d1)
            diag_up.add(d2)
            board[row][col] = "Q"

            backtrack(row + 1)

            board[row][col] = "."
            columns.remove(col)
            diag_down.remove(d1)
            diag_up.remove(d2)

    backtrack(0)
    return results
```

The search is exponential; commonly bounded by O(n!) candidate placements with O(n) state (excluding output). Actual work is reduced by pruning.

## 3. Dry run

For n=4, place a queen in row 0. Each attempted column constrains the next row. Branches that reuse a column or diagonal are rejected immediately. The valid solutions are two distinct boards; the algorithm explores only conflict-free partial placements.

## 4. Interviewer–candidate discussion

**Interviewer:** Why is it safe to place exactly one queen per row?

**Candidate:** Any valid solution has one queen in each row because there are n queens and no two may share a row. We can construct row by row without missing solutions.

**Interviewer:** How can bitmasks help?

**Candidate:** Columns and diagonals can be represented as bits, making conflict checks and state updates efficient for moderate n.

## 5. Edge cases and mistakes

n=0 convention, n=1, n=2 and n=3 have no solutions; restore every set and board mutation after recursion; save a board snapshot rather than the mutable board object.

## 6. Tests

```python
assert len(solve_n_queens(1)) == 1
assert solve_n_queens(2) == []
assert solve_n_queens(3) == []
assert len(solve_n_queens(4)) == 2
for solution in solve_n_queens(4):
    assert all(row.count("Q") == 1 for row in solution)
```

Follow-ups: count solutions without storing boards; bitmask implementation; symmetry reduction; Sudoku solver; branch-and-bound and constraint propagation.
