# Rotate Image

**Difficulty:** Medium  
**Topics:** Array, Matrix, In-place Algorithm  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 09

## 1. Problem Statement

Rotate an n×n matrix 90 degrees clockwise in place.

**Example:**
```text
matrix = [[1,2,3],[4,5,6],[7,8,9]]
Output = [[7,4,1],[8,5,2],[9,6,3]]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Map each source cell (r,c) to (c,n-1-r) in a new matrix, then copy it back.

```python
def rotate(matrix):
    n = len(matrix)
    out = [[0]*n for _ in range(n)]
    for r in range(n):
        for c in range(n):
            out[c][n-1-r] = matrix[r][c]
    for r in range(n):
        matrix[r] = out[r]
```

- **Complexity:** O(n²) time; O(n²) extra space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Transpose across the main diagonal, then reverse each row.

```python
def rotate(matrix):
    n = len(matrix)
    for r in range(n):
        for c in range(r+1, n):
            matrix[r][c], matrix[c][r] = matrix[c][r], matrix[r][c]
    for row in matrix:
        row.reverse()
```

- **Complexity:** O(n²) time; O(1) extra space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Transpose across the main diagonal, then reverse each row.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Transpose to [[1,4,7],[2,5,8],[3,6,9]], then reverse each row to get [[7,4,1],[8,5,2],[9,6,3]].

## 4. Interview Conversation

**Interviewer:** What is the coordinate mapping?

**Candidate:** Clockwise rotation maps (r,c) to (c,n-1-r). A new matrix can implement it directly.

**Interviewer:** How can you do it in place?

**Candidate:** Transpose the matrix, then reverse each row.

**Interviewer:** How do you know the direction is clockwise?

**Candidate:** The transpose swaps axes; reversing each row produces the clockwise orientation. Reversing columns instead would produce counter-clockwise rotation.

**Interviewer:** What if the matrix is rectangular?

**Candidate:** The stated problem guarantees square dimensions. A rectangular rotation changes shape and needs a different in-place strategy.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Counter-clockwise: transpose then reverse each column. 180 degrees: reverse row order and each row. Four clockwise rotations should restore the original.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Check 1×1 and 2×2 matrices, and ensure transpose swaps only c>r to avoid swapping twice.

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

Coordinate transformations appear in image and geospatial pipelines; document row/column orientation explicitly.

## 9. Final Interview Summary

**One-sentence explanation:** Transpose across the main diagonal, then reverse each row.

**Complexity to say aloud:** O(n²) time; O(1) extra space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #09 in the supplied study plan.*