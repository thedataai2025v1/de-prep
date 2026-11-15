# Trapping Rain Water

| Category | Details |
|---|---|
| **Difficulty** | Hard |
| **Topics** | Array, Two Pointers, Stack, Dynamic Programming |
| **Companies** | Google, Amazon, Facebook, Microsoft, Bloomberg, Uber, Apple |
| **Interview Frequency** | Very High (Top 10 Most-Asked) |

---

## 1. Problem Statement

Given `n` non-negative integers representing an elevation map where the width of each bar is 1, compute **how much water it can trap after raining**.

**Example:**

```
Input:  height = [0,1,0,2,1,0,1,3,2,1,2,1]
Output: 6

Explanation: The elevation map is represented by array [0,1,0,2,1,0,1,3,2,1,2,1].
Water trapped = 6 units (visualize as 6 blocks of water between bars).
```

**Constraints (LeetCode):**
- `1 <= n <= 2 * 10^4`
- `0 <= height[i] <= 10^5`

---

## 2. Approaches

### Approach 1: Two Pointers with Left/Right Max (⭐ Optimal)

**Idea:** Use two pointers scanning inward. Track the maximum height seen from the left and from the right. At each step, process the side with the **shorter** max height — the water at that position is determined by the shorter max, because the taller max on the other side will hold the water.

```python
def trap(height):
    if not height:
        return 0

    left, right = 0, len(height) - 1
    left_max = right_max = 0
    water = 0

    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                water += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                water += right_max - height[right]
            right -= 1

    return water
```

| Metric | Value |
|---|---|
| **Time** | O(n) — single pass |
| **Space** | O(1) — constant |

**Pros:**
- Optimal time and space
- No extra arrays or stacks needed
- Handles all cases elegantly

**Cons:**
- Requires careful explanation of why the shorter max determines water

---

### Approach 2: Pre-compute Left/Right Max Arrays

**Idea:** Pre-compute two arrays: `left_max[i]` = max height from 0..i, and `right_max[i]` = max height from i..n-1. Then water at index `i` = `min(left_max[i], right_max[i]) - height[i]`.

```python
def trap(height):
    n = len(height)
    if n == 0:
        return 0

    left_max = [0] * n
    right_max = [0] * n

    left_max[0] = height[0]
    for i in range(1, n):
        left_max[i] = max(left_max[i - 1], height[i])

    right_max[n - 1] = height[n - 1]
    for i in range(n - 2, -1, -1):
        right_max[i] = max(right_max[i + 1], height[i])

    water = 0
    for i in range(n):
        water += min(left_max[i], right_max[i]) - height[i]

    return water
```

| Metric | Value |
|---|---|
| **Time** | O(n) — three passes |
| **Space** | O(n) — two arrays |

**Pros:**
- Very intuitive — directly computes the "water level" at each index
- Easy to understand and explain

**Cons:**
- Uses O(n) extra space
- Three passes over the array

---

### Approach 3: Monotonic Stack

**Idea:** Use a stack to track indices with decreasing heights. When a taller bar appears, pop from the stack and compute the water trapped between the current bar and the bar before the popped one.

```python
def trap(height):
    stack = []
    water = 0

    for i, h in enumerate(height):
        while stack and h > height[stack[-1]]:
            bottom = stack.pop()
            if not stack:
                break
            left = stack[-1]
            width = i - left - 1
            bounded_height = min(height[left], h) - height[bottom]
            water += width * bounded_height
        stack.append(i)

    return water
```

| Metric | Value |
|---|---|
| **Time** | O(n) — each index pushed/popped at most once |
| **Space** | O(n) — stack |

**Pros:**
- Elegant stack-based approach
- Good for understanding the bounded-height concept

**Cons:**
- More complex logic
- O(n) space for the stack

---

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Two Pointers** ⭐ | O(n) | O(1) | **Interview choice** — optimal |
| **2. Left/Right Max Arrays** | O(n) | O(n) | Easiest to explain; good follow-up |
| **3. Monotonic Stack** | O(n) | O(n) | Good to mention as alternative pattern |

---

## 3. Edge Cases

> **Always mention these in an interview to show thoroughness.**

| Edge Case | Input | Expected Behavior |
|---|---|---|
| Empty | `[]` | Return 0 |
| Single element | `[5]` | Return 0 (no space between bars) |
| Two elements | `[3, 7]` | Return 0 |
| All ascending | `[0,1,2,3,4,5]` | Return 0 |
| All descending | `[5,4,3,2,1,0]` | Return 0 |
| Plateau | `[3,3,3,3]` | Return 0 |
| Valley | `[3,0,0,2]` | Water = min(3,2) × 2 - 0 - 0 = 4 |
| Single dip | `[5,0,5]` | Water = 5 (min(5,5) - 0) × 1 = 5 |
| Mountain shape | `[1,2,3,2,1]` | Return 0 (no valleys) |

---

## 4. Interview Talking Points

- **"The two-pointer approach is optimal: O(n) time and O(1) space. The key insight is that water at any position is determined by the shorter of the tallest bars to its left and right — and we can track this with two pointers moving inward."**
- **Data Engineering Connection:** This problem models **workload distribution and buffer management** — like balancing data across partitions where the "shorter max" is the bottleneck. The trapped water represents underutilized capacity or data skew between processing nodes.

---

## 5. Interview Follow-Up Questions

**Q1. Prove that the two-pointer approach is correct.**
→ At any point, `left_max` is the max seen from the left, and `right_max` from the right. If `height[left] < height[right]`, then `left_max < right_max` (because `height[left]` is the current left candidate). So the water at `left` is bounded by `left_max`. The taller `right_max` guarantees water won't spill right, so we can safely compute water at `left` and advance.

**Q2. How would you modify this for 3D trapping (surface of a 2D grid)?**
→ Use a min-heap (priority queue) starting from the border, expanding inward. This is a harder problem (LeetCode 407. Trapping Rain Water II, Hard). It requires a BFS-like approach with a heap.

**Q3. What if the bars are placed at irregular intervals with different widths?**
→ Each bar has both a height and a width. The area calculation becomes `width * min(left_max, right_max)`. The algorithm generalizes but is no longer O(n).

**Q4. Can you solve this with constant space but without two pointers?**
→ The two-pointer approach already achieves O(1) space. No other O(n) time + O(1) space solution exists.

**Q5. How does this relate to Container With Most Water?**
→ Both involve two-pointer strategies with height constraints. Container looks for **two bars** forming a container; Trapping Rain Water looks at **all positions** between bars to capture water. Container is a maximization problem; Trapping Rain Water is a summation problem.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `max` instead of `min` for water level | Water level is determined by the shorter wall | `water += min(left_max, right_max) - height[i]` |
| Forgetting to subtract `height[i]` | The bar itself displaces water | Always subtract the current bar height |
| Not handling empty or single-element arrays | IndexError on array access | Early return 0 for n < 3 |
| Stack approach: forgetting `if not stack: break` | After popping the last element, no left boundary exists | Check stack emptiness before computing width |
| Using `<` instead of `<=` in pointer comparison | Both directions work; subtle difference in behavior on equal heights | Either is fine; be consistent |
| Adding water before updating max in two-pointer | If current height is a new max, no water is trapped there yet | Update max first, then compute water |

---

## 7. Related Problems

| Problem | Key Concept | Difficulty |
|---|---|---|
| [Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Two-pointer max area | Medium |
| [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | Monotonic stack for max area | Hard |
| [Trapping Rain Water II](https://leetcode.com/problems/trapping-rain-water-ii/) | 3D water trapping with heap | Hard |
| [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Prefix/suffix arrays | Medium |
| [Max Chunks To Make Sorted](https://leetcode.com/problems/max-chunks-to-make-sorted/) | Partitioning with prefix max | Medium |

---

*Last updated: 2025 — for data engineering interview preparation.*