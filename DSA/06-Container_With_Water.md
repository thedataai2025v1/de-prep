# Container With Most Water

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Two Pointers, Greedy |
| **Companies** | Amazon, Google, Meta, Microsoft |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an array `height` where `height[i]` is the height of a vertical line at position `i`, find the two lines that together with the x-axis form a container that holds the most water. Return the maximum amount of water.

**Example:**
```
Input:  height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
Output: 49
```

---

## 2. Approaches

### Approach 1: Brute Force (Naive)

Try every pair of lines. For each pair `(i, j)`, compute the area as `min(height[i], height[j]) * (j - i)`. Track the maximum.

**Pros:**
- Trivial to implement — no thought required
- Conceptually simple — enumerates all possibilities
- Good starting point for exploring optimizations

**Cons:**
- O(n²) time — too slow for large n
- Checks redundant pairs that can't possibly yield a larger area
- Will not pass for arrays with 10^5 elements

```python
def maxArea(height: list[int]) -> int:
    max_water = 0
    n = len(height)
    for i in range(n):
        for j in range(i + 1, n):
            water = min(height[i], height[j]) * (j - i)
            max_water = max(max_water, water)
    return max_water
```

| Metric | Value |
|---|---|
| **Time** | O(n²) |
| **Space** | O(1) |

### Approach 2: Two Pointers (⭐ Optimal)

Start with two pointers at both ends of the array. The area is limited by the shorter line. Move the pointer at the shorter line inward because moving the taller line cannot increase the area (the width decreases and the height is still capped by the shorter line).

**Pros:**
- O(n) time — single pass with optimal two-pointer logic
- O(1) space — minimal memory footprint
- Greedy yet provably optimal: moving the shorter pointer is always safe
- Elegant and simple to implement once understood

**Cons:**
- Requires a non-obvious insight: why moving the shorter line is correct
- Greedy reasoning must be explained carefully in interviews
- Only works for this specific problem — not generalizable to arbitrary area problems
- All duplicates of the same height are still processed

```python
def maxArea(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    max_water = 0

    while left < right:
        water = min(height[left], height[right]) * (right - left)
        max_water = max(max_water, water)

        if height[left] < height[right]:
            left += 1
        else:
            right -= 1

    return max_water
```

| Metric | Value |
|---|---|
| **Time** | O(n) |
| **Space** | O(1) |

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Brute Force** | O(n²) | O(1) | Understanding the problem, verifying correctness |
| **2. Two Pointers** | O(n) | O(1) | Interviews, production code |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| Two lines only | Area computed directly |
| All equal heights (e.g., `[5, 5, 5, 5]`) | Maximum area is the largest width × 5 |
| Heights are decreasing (e.g., `[5, 4, 3, 2, 1]`) | First and last lines form the maximum area |
| Heights are increasing (e.g., `[1, 2, 3, 4, 5]`) | First and last lines form the maximum area |
| Very tall one side, very short other side | Short side limits the area; move the short side in |
| Large heights (up to 10⁴) | Works within integer bounds |

---

## 4. Interview Talking Points

- "The key insight is that the container is limited by the shorter line. Moving the pointer from the shorter line inward is the only move that can potentially increase the area — the width shrinks, but we hope to find a taller line."
- "In data engineering, this greedy two-pointer approach appears in sliding-window aggregations over time-series data and in merge-join algorithms where we advance the smaller key."

---

## 5. Interview Follow-Up Questions

**Q1: What if we need to find the three lines that form the maximum triangle or trapezoid area?**
-> That's a fundamentally different problem. For a triangle with three lines as vertices, the maximum area is found by maximizing the base and the vertical midpoint. This problem is specifically about two vertical lines forming a rectangle on the x-axis.

**Q2: Can the two-pointer approach be proven correct?**
-> Yes. At any step, the area is `min(h[left], h[right]) × (right - left)`. If `h[left] < h[right]`, any pair with `left` and some index between `left` and `right` will have a smaller width and a height ≤ `h[left]`, so moving `left` right is always safe.

**Q3: What if heights can be floating-point numbers?**
-> The algorithm works identically — the comparisons and area calculations are the same. Floating-point precision is not an issue because we only do comparisons and multiplications.

**Q4: What if we need the three containers (lines) that collectively hold the most water?**
-> This would be a combinatorial optimization problem — likely NP-hard in general. The single-container case has the efficient two-pointer solution because of the monotonic optimal substructure.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Moving the taller line instead of the shorter line | Misses the opportunity to find a taller bound | Move only the pointer pointing to the shorter line |
| Updating `max_water` only when `min(left, right)` is larger | Area depends on both height and width — width always decreases | Update max at every step |
| Using `abs(left - right)` for width | Redundant but correct — just use `right - left` since left < right |
| Checking height equality incorrectly with `if` on both pointers | Both conditions branch incorrectly | Use `<` vs `>=` or `<=` to handle equality deterministically |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/) | Similar two-pointer approach but multiple containers |
| [3Sum](DSA/05-3Sum.md) | Two-pointer technique on a sorted array |
| [Maximum Subarray](DSA/07-Maximum_Subarray.md) | Another array-based optimization with O(n) greedy approach |

---

*Last updated: 2025 — for data engineering interview preparation.*