# Set Matrix Zeroes

**Difficulty:** Medium  
**Topics:** Array, Matrix, In-place Algorithm  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 04

## 1. Problem Statement

If a matrix cell is zero, set its entire row and column to zero. Modify the matrix in place.

**Example:**
```text
matrix = [[1,1,1],[1,0,1],[1,1,1]]
Output = [[1,0,1],[0,0,0],[1,0,1]]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Record all zero-bearing row and column indices in separate sets, then zero matching cells.

```python
def set_zeroes(matrix):
    if not matrix or not matrix[0]:
        return
    rows, cols = set(), set()
    for r in range(len(matrix)):
        for c in range(len(matrix[0])):
            if matrix[r][c] == 0:
                rows.add(r); cols.add(c)
    for r in range(len(matrix)):
        for c in range(len(matrix[0])):
            if r in rows or c in cols:
                matrix[r][c] = 0
```

- **Complexity:** O(mn) time; O(m+n) auxiliary space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use the first row and first column as markers, plus two flags recording whether either originally contained zero.

```python
def set_zeroes(matrix):
    if not matrix or not matrix[0]:
        return
    m, n = len(matrix), len(matrix[0])
    first_row = any(matrix[0][c] == 0 for c in range(n))
    first_col = any(matrix[r][0] == 0 for r in range(m))
    for r in range(1, m):
        for c in range(1, n):
            if matrix[r][c] == 0:
                matrix[r][0] = matrix[0][c] = 0
    for r in range(1, m):
        for c in range(1, n):
            if matrix[r][0] == 0 or matrix[0][c] == 0:
                matrix[r][c] = 0
    if first_row:
        for c in range(n): matrix[0][c] = 0
    if first_col:
        for r in range(m): matrix[r][0] = 0
```

- **Complexity:** O(mn) time; O(1) auxiliary space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use the first row and first column as markers, plus two flags recording whether either originally contained zero.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

The center zero marks row 1 at matrix[1][0] and column 1 at matrix[0][1]. The inner matrix is zeroed from these markers; the original first-row/first-column flags are applied last.

## 4. Interview Conversation

**Interviewer:** Why not zero immediately when a zero is found?

**Candidate:** A newly written zero could be mistaken for an original zero and cascade incorrectly. I need a marking pass first.

**Interviewer:** Can you do it in constant extra space?

**Candidate:** Yes. Reuse the first row and column as marker storage, while preserving their original zero state in two Boolean flags.

**Interviewer:** Why apply the flags last?

**Candidate:** Because the first row and column contain markers until the inner cells have been processed.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Empty matrix: return early. Single row/column: flags handle them. Explain that the algorithm mutates input. Rectangular matrices are supported.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Why two flags are necessary; how marker writes differ from original zeros; why order of passes matters.

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

In-place mutation is useful under memory pressure, but production transformations often prefer immutable stages for lineage and reproducibility.

## 9. Final Interview Summary

**One-sentence explanation:** Use the first row and first column as markers, plus two flags recording whether either originally contained zero.

**Complexity to say aloud:** O(mn) time; O(1) auxiliary space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #04 in the supplied study plan.*