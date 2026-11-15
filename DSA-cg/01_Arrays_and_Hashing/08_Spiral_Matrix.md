# Spiral Matrix

**Difficulty:** Medium  
**Topics:** Array, Matrix, Simulation  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 08

## 1. Problem Statement

Return all elements of an m×n matrix in clockwise spiral order, starting at the top-left.

**Example:**
```text
matrix = [[1,2,3],[4,5,6],[7,8,9]]
Output = [1,2,3,6,9,8,7,4,5]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Simulate direction changes while tracking visited cells.

```python
def spiral_order(matrix):
    if not matrix or not matrix[0]: return []
    m, n = len(matrix), len(matrix[0])
    seen = [[False]*n for _ in range(m)]
    dirs = [(0,1),(1,0),(0,-1),(-1,0)]
    r = c = d = 0
    out = []
    for _ in range(m*n):
        out.append(matrix[r][c]); seen[r][c] = True
        nr, nc = r+dirs[d][0], c+dirs[d][1]
        if not (0 <= nr < m and 0 <= nc < n and not seen[nr][nc]):
            d = (d+1) % 4
            nr, nc = r+dirs[d][0], c+dirs[d][1]
        r, c = nr, nc
    return out
```

- **Complexity:** O(mn) time; O(mn) auxiliary space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Maintain top, bottom, left and right boundaries; traverse each edge and shrink the corresponding boundary.

```python
def spiral_order(matrix):
    if not matrix or not matrix[0]: return []
    top, bottom = 0, len(matrix)-1
    left, right = 0, len(matrix[0])-1
    out = []
    while top <= bottom and left <= right:
        for c in range(left, right+1): out.append(matrix[top][c])
        top += 1
        for r in range(top, bottom+1): out.append(matrix[r][right])
        right -= 1
        if top <= bottom:
            for c in range(right, left-1, -1): out.append(matrix[bottom][c])
            bottom -= 1
        if left <= right:
            for r in range(bottom, top-1, -1): out.append(matrix[r][left])
            left += 1
    return out
```

- **Complexity:** O(mn) time; O(1) auxiliary space excluding output

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Maintain top, bottom, left and right boundaries; traverse each edge and shrink the corresponding boundary.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Emit top row [1,2,3], right edge [6,9], bottom reversed [8,7], left upward [4], then inner layer [5]. Each boundary moves inward after its edge is consumed.

## 4. Interview Conversation

**Interviewer:** How do you avoid revisiting cells?

**Candidate:** A visited matrix works, but four shrinking boundaries avoid the extra O(mn) memory.

**Interviewer:** Why guard the bottom and left traversals?

**Candidate:** A final layer may contain only one row or one column. Without guards, I could emit cells twice.

**Interviewer:** What about an empty matrix?

**Candidate:** Return an empty list before accessing matrix[0].

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Generate a spiral matrix by assigning values instead of appending. Complexity is O(mn). For huge matrices, discuss storage layout and non-sequential access.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Test 1×N, N×1, 1×1, rectangular and empty inputs.

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

Boundary traversal is useful in grid transforms; physical partitioning matters if the matrix is distributed.

## 9. Final Interview Summary

**One-sentence explanation:** Maintain top, bottom, left and right boundaries; traverse each edge and shrink the corresponding boundary.

**Complexity to say aloud:** O(mn) time; O(1) auxiliary space excluding output

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #08 in the supplied study plan.*