# Find the Duplicate Number

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Linked List, Two Pointers |
| **Companies** | Google, Amazon, Meta, Microsoft |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an array of `n + 1` integers where each integer is between `1` and `n` (inclusive), prove that at least one duplicate number must exist and find it. You must solve the problem **without modifying the array** and using only **O(1) extra space**.

**Example:**
```
Input:  nums = [1, 3, 4, 2, 2]
Output: 2

Input:  nums = [3, 1, 3, 4, 2]
Output: 3
```

---

## 2. Approaches

### Approach 1: Floyd's Cycle Detection (Tortoise and Hare) (⭐ Optimal)

Treat the array as a linked list where `nums[i]` points to the next node at index `nums[i]`. Because there is a duplicate, two different indices point to the same next index — creating a cycle. Use Floyd's cycle detection algorithm: a slow pointer moves 1 step and a fast pointer moves 2 steps. When they meet, reset one pointer to the start and move both 1 step at a time. The meeting point is the duplicate.

**Pros:**
- O(n) time, O(1) space — meets all constraints perfectly
- Clever and impressive in an interview
- Doesn't modify the input array
- Generalizable to any cycle detection problem

**Cons:**
- Very non-obvious — the array-as-linked-list mapping is not intuitive
- Hard to explain clearly under pressure
- Doesn't work if the array can contain 0 (since 0 is a valid index)
- Only returns one duplicate even if there are multiple

```python
def findDuplicate(nums: list[int]) -> int:
    # Phase 1: Find intersection point in the cycle
    slow = nums[0]
    fast = nums[0]
    while True:
        slow = nums[slow]
        fast = nums[nums[fast]]
        if slow == fast:
            break

    # Phase 2: Find entrance to the cycle (the duplicate)
    slow2 = nums[0]
    while slow != slow2:
        slow = nums[slow]
        slow2 = nums[slow2]

    return slow
```

| Metric | Value |
|---|---|
| **Time** | O(n) |
| **Space** | O(1) |

### Approach 2: Binary Search on Value Range

Binary search on the range `[1, n]`. For a midpoint `mid`, count how many elements in the array are ≤ `mid`. If the count is greater than `mid`, the duplicate must be in the left half (pigeonhole principle).

**Pros:**
- Easier to understand and explain than cycle detection
- O(1) space — meets the constraint
- Doesn't modify the input array
- Works even when there are multiple duplicates

**Cons:**
- O(n log n) time — slower than Floyd's algorithm
- Two nested loops can be error-prone to implement
- Doesn't scale well for very large arrays with tight time limits

```python
def findDuplicate(nums: list[int]) -> int:
    left, right = 1, len(nums) - 1

    while left < right:
        mid = left + (right - left) // 2
        count = sum(1 for x in nums if x <= mid)

        if count > mid:
            right = mid
        else:
            left = mid + 1

    return left
```

| Metric | Value |
|---|---|
| **Time** | O(n log n) |
| **Space** | O(1) |

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Floyd's Cycle Detection** | O(n) | O(1) | Interviews, optimal solution |
| **2. Binary Search on Values** | O(n log n) | O(1) | When the cycle trick is too hard to explain |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| All elements the same (e.g., `[2, 2, 2, 2]`) | Both approaches return 2 |
| Duplicate is the first element | Works — cycle detection doesn't depend on position |
| Two different duplicates (e.g., `[1, 1, 2, 2]`) | Both approaches return one of the duplicates |
| Array of size 2 (`[1, 1]`) | Smallest valid input; both approaches work |

---

## 4. Interview Talking Points

- "The key insight is treating the array as a linked list where `nums[i]` is the pointer to the next node. This transforms a duplicate-finding problem into a cycle-detection problem."
- "In data engineering, cycle detection is useful for finding loops in DAGs, detecting circular dependencies in ETL pipelines, and identifying self-referential foreign keys in relational data."

---

## 5. Interview Follow-Up Questions

**Q1: What if we're allowed to modify the array?**
-> We could sort it in O(n log n) time and scan for adjacent duplicates, or use negative marking (negate `nums[abs(nums[i])]` to mark seen values) in O(n) time, O(1) space.

**Q2: What if we're allowed O(n) extra space?**
-> Use a hash set to track seen elements — O(n) time, O(n) space. Simplest solution but violates the space constraint.

**Q3: What if the array contains 0?**
-> Floyd's algorithm fails because 0 is a valid index and breaks the 1-to-n range assumption. Binary search on values still works, or we can offset all values by +1.

**Q4: Can Floyd's algorithm find ALL duplicates, not just one?**
-> No — it finds the entrance to the cycle, which is one duplicate. To find all duplicates, use the binary search approach or a frequency count (if O(n) space is allowed).

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Starting slow and fast at `nums[0]` vs index `0` | The pointers represent **values** used as indices, not array indices | Start both at `nums[0]` (the value at index 0) |
| Not resetting one pointer to `nums[0]` in phase 2 | You won't find the cycle entrance | After detecting the cycle, reset one pointer to `nums[0]` |
| Using binary search on indices instead of values | The values are not sorted by index | Binary search on the **value range** `[1, n]` |
| Counting elements `< mid` instead of `≤ mid` | Off-by-one: misses the boundary case | Use `count = sum(1 for x in nums if x <= mid)` |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | Same Floyd's cycle detection algorithm |
| [Happy Number](https://leetcode.com/problems/happy-number/) | Another cycle detection problem |
| [First Missing Positive](https://leetcode.com/problems/first-missing-positive/) | O(n) time, O(1) space on an array |

---

*Last updated: 2025 — for data engineering interview preparation.*
