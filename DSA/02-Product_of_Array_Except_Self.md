# Product of Array Except Self

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Prefix/Suffix |
| **Companies** | Amazon, Google, Microsoft, Meta |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an integer array `nums`, return an array `output` where `output[i]` is the product of all the elements of `nums` except `nums[i]`. The product of any prefix or suffix of `nums` is guaranteed to fit in an integer. You must write an algorithm that runs in `O(n)` time and **does not use the division operation**.

**Example:**
```
Input:  nums = [1, 2, 3, 4]
Output: [24, 12, 8, 6]
```

---

## 2. Approaches

### Approach 1: Prefix and Suffix Passes (⭐ Optimal)

Make two passes over the array. First pass builds prefix products into the output array. Second pass multiplies in suffix products on the fly using a running variable.

**Pros:**
- Two simple, clean passes with very readable logic
- Handles zeros naturally — no special case needed
- Constant extra space beyond the result — meets the O(1) constraint
- Optimal O(n) time complexity

**Cons:**
- Requires understanding the prefix/suffix pattern
- Must reuse the output array to achieve O(1) extra space — non-obvious
- Two passes (though still O(n))

```python
def productExceptSelf(nums: list[int]) -> list[int]:
    n = len(nums)
    output = [1] * n

    # First pass: prefix products
    prefix = 1
    for i in range(n):
        output[i] = prefix
        prefix *= nums[i]

    # Second pass: multiply in suffix products
    suffix = 1
    for i in range(n - 1, -1, -1):
        output[i] *= suffix
        suffix *= nums[i]

    return output
```

| Metric | Value |
|---|---|
| **Time** | O(n) |
| **Space** | O(1) extra (output array doesn't count) |

### Approach 2: Separate Left and Right Arrays (Alternative)

Build two full arrays: `left[i]` = product of all elements to the left of `i`, and `right[i]` = product of all elements to the right of `i`. Then multiply element-wise.

**Pros:**
- Easy to understand and debug — each array is independent
- Visually clear how the final answer is computed
- Good for explaining the concept before optimizing

**Cons:**
- Uses O(n) extra space beyond output — doesn't meet the strict O(1) constraint
- Requires three passes (left, right, then combine) instead of two
- Not the expected answer in an interview setting

```python
def productExceptSelf(nums: list[int]) -> list[int]:
    n = len(nums)
    left = [1] * n
    right = [1] * n
    output = [1] * n

    # Build left products
    for i in range(1, n):
        left[i] = left[i - 1] * nums[i - 1]

    # Build right products
    for i in range(n - 2, -1, -1):
        right[i] = right[i + 1] * nums[i + 1]

    # Multiply left and right
    for i in range(n):
        output[i] = left[i] * right[i]

    return output
```

| Metric | Value |
|---|---|
| **Time** | O(n) |
| **Space** | O(n) extra (two auxiliary arrays) |

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Prefix/Suffix Passes** | O(n) | O(1) extra | Interviews, production code |
| **2. Left/Right Arrays** | O(n) | O(n) extra | Explaining the concept first |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| Array with 2 elements | Works correctly: `[a, b]` → `[b, a]` |
| Array with zeros | Handled naturally; zeros propagate through prefix/suffix |
| All zeros | Returns array of zeros |
| Negative numbers | Handled correctly — signs propagate naturally |
| Single element | Not valid per problem constraints (n ≥ 2) |

---

## 4. Interview Talking Points

- "I use the prefix/suffix pattern here because it avoids division entirely. This is important when zeros are present — division would cause a divide-by-zero error."
- "In data engineering, this pattern maps directly to window aggregations: computing running totals, exponential decay features, or prefix sums in time-series pipelines. The same two-pass technique is used in SQL window functions like `SUM() OVER (ORDER BY)`."

---

## 5. Interview Follow-Up Questions

**Q1: What if we're allowed to use division?**
-> We could compute the total product and divide each element out. However, this fails when any element is zero. We'd need a special case: count zeros, and if more than one zero exists, all outputs are zero.

**Q2: Can this be done in parallel?**
-> Yes. Both the prefix and suffix passes can be parallelized using a parallel prefix-sum (scan) algorithm, which computes the full prefix array in O(log n) time with O(n) threads.

**Q3: What if we also need to handle floating-point numbers?**
-> The algorithm works identically — it doesn't depend on integer arithmetic. Only concern is floating-point precision accumulation, which is a general issue, not specific to this approach.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using division (`total // nums[i]`) | Fails with zeros; also doesn't meet the constraint | Use prefix/suffix two-pass |
| Forgetting to initialize output with 1s | Results in zero-filled output | `output = [1] * n` |
| Off-by-one in loop bounds | Misses first or last element | Use `range(n)` for forward, `range(n-1, -1, -1)` for backward |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Set Matrix Zeroes](DSA/04-Set_Matrix_Zeroes.md) | Uses prefix/suffix-like marking for row/column operations |
| [Maximum Subarray](DSA/07-Maximum_Subarray.md) | Another running-aggregation pattern on arrays |
| Running Product / Prefix Sum | Same two-pass prefix/suffix technique |

---

*Last updated: 2025 — for data engineering interview preparation.*
