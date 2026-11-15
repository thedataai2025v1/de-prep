# Container With Most Water

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Two Pointers, Greedy |
| **Companies** | Google, Amazon, Facebook, Microsoft, Bloomberg, Uber |
| **Interview Frequency** | High |

---

## 1. Problem Statement

You are given an integer array `height` of length `n`. There are `n` vertical lines drawn such that the two endpoints of the `i`-th line are `(i, 0)` and `(i, height[i])`.

Find two lines that together with the x-axis form a container, such that the container contains the **most water**.

Return the **maximum amount of water** a container can store.

**Note:** You may not slant the container.

**Example:**

```
Input:  height = [1,8,6,2,5,4,8,3,7]
Output: 49

Explanation: The vertical lines at indices 1 and 8 (heights 8 and 7) form
a container of width 7 and height min(8,7) = 7, giving area = 7 × 7 = 49.
```

**Constraints (LeetCode):**
- `2 <= n <= 10^5`
- `0 <= height[i] <= 10^4`

---

## 2. Approaches

### Approach 1: Two Pointers (⭐ Optimal)

**Idea:** Start with two pointers at the two ends of the array. The area is `min(height[left], height[right]) * (right - left)`. Move the pointer pointing to the **shorter** line inward — because moving the taller line would never increase the area (the width shrinks and the height is still bounded by the shorter line). Track the maximum area encountered.

```python
def maxArea(height):
    left, right = 0, len(height) - 1
    max_water = 0

    while left < right:
        # Calculate area
        width = right - left
        h = min(height[left], height[right])
        max_water = max(max_water, width * h)

        # Move the pointer with the shorter height
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1

    return max_water
```

| Metric | Value |
|---|---|
| **Time** | O(n) — single pass |
| **Space** | O(1) — only two pointers |

**Pros:**
- Linear time, constant space — optimal
- Intuitive greedy reasoning
- Easy to implement correctly

**Cons:**
- Proof of correctness (why moving shorter pointer works) requires careful explanation

---

### Approach 2: Brute Force (Alternative)

**Idea:** Check every possible pair of lines `(i, j)` and compute the area. Keep the maximum.

```python
def maxArea(height):
    n = len(height)
    max_water = 0

    for i in range(n):
        for j in range(i + 1, n):
            width = j - i
            h = min(height[i], height[j])
            max_water = max(max_water, width * h)

    return max_water
```

| Metric | Value |
|---|---|
| **Time** | O(n²) — all pairs |
| **Space** | O(1) |

**Pros:**
- Correct and straightforward
- No pointer logic needed

**Cons:**
- For n = 10⁵, this is 5 billion iterations — too slow
- Never acceptable in an interview for the final answer

---

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Two Pointers** ⭐ | O(n) | O(1) | **Interview choice** — optimal |
| **2. Brute Force** | O(n²) | O(1) | Only for small arrays (n < 1000) |

---

## 3. Edge Cases

> **Always mention these in an interview to show thoroughness.**

| Edge Case | Input | Expected Behavior |
|---|---|---|
| Minimum size (2 elements) | `[1, 1]` | Area = min(1,1) × 1 = 1 |
| All equal heights | `[5, 5, 5, 5]` | Max area at widest pair: 5 × 3 = 15 |
| Strictly increasing | `[1, 2, 3, 4, 5]` | Max area = min(1,5) × 4 = 4 |
| Strictly decreasing | `[5, 4, 3, 2, 1]` | Max area = min(5,1) × 4 = 4 |
| Ascending then descending | `[1, 2, 5, 2, 1]` | Max area at the tallest bar on both sides |
| All zeros | `[0, 0, 0]` | Area = 0 |
| One zero in middle | `[1, 0, 1]` | Area = min(1,1) × 2 = 2 |

---

## 4. Interview Talking Points

- **"The two-pointer approach achieves O(n) time and O(1) space, which is optimal because we must examine each element at least once. The key insight is that moving the shorter pointer never misses a larger area."**
- **Data Engineering Connection:** This problem models resource allocation optimization — finding the optimal partition boundaries in data shuffling, maximizing throughput of data pipelines given capacity constraints (the "shorter wall" being the bottleneck).

---

## 5. Interview Follow-Up Questions

**Q1. Prove that moving the shorter pointer is correct.**
→ The area is bounded by `min(height[l], height[r])`. If we move the taller pointer inward, the new min height is still ≤ the old min height (the shorter one), and the width decreases, so the area can only decrease or stay the same. Therefore, the larger area cannot come from moving the taller pointer — we must move the shorter one.

**Q2. What if the heights can be floating-point numbers?**
→ The algorithm works identically — comparison and min/max work for floats.

**Q3. How would you find the top K maximum containers, not just the max?**
→ Use a max-heap to store candidate areas, or a more complex two-pointer approach that tracks the K largest. Brute force O(n²) + heap could work for small n.

**Q4. Can you solve this with 3D containers (volume instead of area)?**
→ That becomes a different, more complex problem (3D water container). The two-pointer technique does not generalize directly.

**Q5. What if you need to output the indices of the two lines instead of the area?**
→ Track `left` and `right` when `max_water` is updated, and return them alongside the area.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Moving the taller pointer instead of shorter | Misses the optimal solution; the proof hinges on moving the shorter one | Always move the pointer with the smaller height |
| Forgetting the width is `right - left` | Using `right - left + 1` or a stale value | Calculate width fresh each iteration |
| Using `max(height[left], height[right])` for height | The water level is limited by the **shorter** line | Use `min(height[left], height[right])` |
| Not handling the case `height[left] == height[right]` | Both pointers are equal; moving either is fine | The code handles this naturally (`else` branch moves right) |
| Brute force for large n | O(n²) for n=10⁵ is 10¹⁰ operations — impossible | Use two-pointer approach |

---

## 7. Related Problems

| Problem | Key Concept | Difficulty |
|---|---|---|
| [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/) | Water trapped in elevation map | Hard |
| [Three Sum](https://leetcode.com/problems/3sum/) | Two-pointer technique with sorting | Medium |
| [Max Area of Island](https://leetcode.com/problems/max-area-of-island/) | Grid area with DFS/BFS | Medium |
| [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | Stack-based maximum area | Hard |
| [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Two-pointer on strings | Easy |

---

*Last updated: 2025 — for data engineering interview preparation.*