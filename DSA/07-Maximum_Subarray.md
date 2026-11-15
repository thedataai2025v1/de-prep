# Maximum Subarray

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Dynamic Programming |
| **Companies** | Google, Amazon, Microsoft, Meta |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an integer array `nums`, find the contiguous subarray (containing at least one number) which has the largest sum and return its sum.

**Example:**
```
Input:  nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Output: 6
```

The contiguous subarray `[4, -1, 2, 1]` has the largest sum = 6.

---

## 2. Approaches

### Approach 1: Kadane's Algorithm (⭐ Optimal)

Track two variables: `current_sum` (the best subarray ending at the current position) and `max_sum` (the best overall). At each element, either extend the previous subarray or start fresh from the current element. Reset `current_sum` to 0 (or restart) when it goes negative.

**Pros:**
- Single pass, constant space — optimal on all dimensions
- Handles all negative numbers correctly (returns the least-negative element)
- The `max(nums[i], current_sum + nums[i])` line captures the core DP decision beautifully
- Trivial to implement in any language

**Cons:**
- Non-obvious on first look — the DP recurrence isn't immediately clear
- Only works for sums, not for products (sign flips break the greedy assumption)
- Returns only the sum, not the subarray itself (though easily extended)

```python
def maxSubArray(nums: list[int]) -> int:
    current_sum = nums[0]
    max_sum = nums[0]

    for i in range(1, len(nums)):
        # Either extend the previous subarray or start a new one
        current_sum = max(nums[i], current_sum + nums[i])
        max_sum = max(max_sum, current_sum)

    return max_sum
```

| Metric | Value |
|---|---|
| **Time** | O(n) |
| **Space** | O(1) |

### Approach 2: Divide and Conquer (Alternative)

Split the array in half. The maximum subarray is either entirely in the left half, entirely in the right half, or crosses the midpoint. Recursively find the best in each half, and combine.

**Pros:**
- Demonstrates the divide-and-conquer pattern — good to showcase alternative strategies
- The cross-midpoint computation reveals structure of the problem
- Parallelizable — each half can be computed independently
- O(log n) recursion depth is acceptable for moderate n

**Cons:**
- Slower than Kadane's — O(n log n) vs O(n)
- More complex to implement correctly — off-by-one errors are common
- O(log n) recursion stack — not truly constant space
- The cross-midpoint pass is still O(n) per level — less elegant than Kadane's

```python
def maxSubArray(nums: list[int]) -> int:
    def helper(left: int, right: int) -> int:
        if left == right:
            return nums[left]

        mid = (left + right) // 2

        # Max subarray entirely in left half
        left_max = helper(left, mid)
        # Max subarray entirely in right half
        right_max = helper(mid + 1, right)
        # Max subarray crossing the midpoint
        mid_left_max = float('-inf')
        mid_right_max = float('-inf')

        current = 0
        for i in range(mid, left - 1, -1):
            current += nums[i]
            mid_left_max = max(mid_left_max, current)

        current = 0
        for i in range(mid + 1, right + 1):
            current += nums[i]
            mid_right_max = max(mid_right_max, current)

        cross_max = mid_left_max + mid_right_max

        return max(left_max, right_max, cross_max)

    return helper(0, len(nums) - 1)
```

| Metric | Value |
|---|---|
| **Time** | O(n log n) |
| **Space** | O(log n) (recursion stack) |

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Kadane's Algorithm** | O(n) | O(1) | Interviews — optimal, concise |
| **2. Divide and Conquer** | O(n log n) | O(log n) | When asked to compare approaches |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| All negative numbers | Returns the maximum (least negative) single element |
| Single element | Returns that element |
| Mix of positive and negative | Correctly finds the peak-sum subarray |
| All positive numbers | Returns the sum of the entire array |
| Array with zeros | Zeros don't hurt; they're neutral in the sum |

---

## 4. Interview Talking Points

- "Kadane's algorithm is a classic example of dynamic programming where we reduce the state to a single variable. The key insight: at each position, we either extend the best subarray ending at the previous position or start a new subarray from the current element. This greedy choice works because any subarray that ends at position `i` either includes `nums[i-1]` or doesn't."
- "In data engineering, this pattern appears in anomaly detection on time-series data: finding peak activity windows, detecting consecutive spikes in sensor data, or identifying the maximum consecutive error streak in a log pipeline. The same running-sum technique is used for detecting data quality degradation over time."

---

## 5. Interview Follow-Up Questions

**Q1: Can you return the subarray indices, not just the sum?**
-> Yes. Track `start`, `end`, and `temp_start` indices. Update `temp_start = i` whenever we restart `current_sum`, and update `start, end` whenever `max_sum` is updated.

**Q2: How would you handle a circular array?**
-> The answer is the maximum of: (a) the standard Kadane's result, and (b) the total sum minus the minimum subarray sum (the "wrap-around" case). Special case: if all numbers are negative, just return the standard Kadane's result.

**Q3: What if the input is streaming (online algorithm)?**
-> Kadane's algorithm is naturally online — it processes one element at a time and maintains only two variables. It's the ideal streaming solution for finding the maximum subarray sum in real time.

**Q4: How does this change for maximum product subarray?**
-> Multiplication changes the game: a negative number times another negative becomes positive, so you must track both the max and min (most negative) running product. This makes the problem harder — it requires tracking two states instead of one.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Setting `current_sum = 0` initially | Fails for all-negative arrays — returns 0 instead of the max element | Initialize with `nums[0]` |
| Using `current_sum = max(0, current_sum + nums[i])` | Loses the all-negative case; returns 0 incorrectly | Use `max(nums[i], current_sum + nums[i])` |
| Forgetting the `at least one element` constraint | An empty subarray has sum 0, but the problem requires ≥ 1 element | Initialize both variables with `nums[0]` |
| Not iterating from index 1 | Double-counts `nums[0]` in the loop | Start the loop at index 1 |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Product of Array Except Self](DSA/02-Product_of_Array_Except_Self.md) | Both use running-aggregation patterns on arrays |
| [Container With Water](DSA/06-Container_With_Water.md) | Another array optimization problem |
| Maximum Product Subarray | Same pattern, but with multiplication (signs flip behavior) |

---

*Last updated: 2025 — for data engineering interview preparation.*