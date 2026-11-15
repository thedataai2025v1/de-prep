# Binary Search

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Array, Binary Search |
| **Companies** | Google, Amazon, Microsoft, Meta, Apple |
| **Interview Frequency** | High |

---

## 1. Problem Statement
Given a sorted array of integers `nums` in ascending order and an integer `target`, return the index of `target` if it is found in `nums`. Otherwise, return `-1`. The algorithm must have `O(log n)` runtime complexity.

**Example:**
```
Input: nums = [-1, 0, 3, 5, 9, 12], target = 9
Output: 4
```

---

## 2. Approaches

### Approach 1: Standard Binary Search (Optimal)
Use two pointers `left` and `right` to narrow the search range. At each step, compare the middle element with the target: if equal, return the index; if smaller, search the right half; if larger, search the left half.

**Pros:**
- O(1) space — no call stack or auxiliary data structures
- Simple loop with clear invariants — easy to explain in an interview
- No recursion limit risk — safe for arbitrarily large arrays

**Cons:**
- Requires careful boundary management (`left <= right`, `mid + 1`, `mid - 1`) — off-by-one errors are common
- Slightly less intuitive for candidates more comfortable with recursive thinking

```python
def search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2  # prevents integer overflow

        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1  # target not found
```

| Metric | Value |
|---|---|
| **Time** | O(log n) |
| **Space** | O(1) |

### Approach 2: Recursive Binary Search (Alternative)
Same logic as iterative but expressed recursively. Each recursive call halves the search space by passing updated boundaries.

**Pros:**
- Directly expresses the divide-and-conquer nature of binary search
- Fewer explicit state variables — left/right are passed as function arguments
- Easier to adapt for problems that naturally recurse (e.g., find first/last occurrence)

**Cons:**
- O(log n) call stack space — risks stack overflow on very large arrays in languages with limited stack depth
- Slightly more verbose due to the nested helper function and base case check

```python
def search(nums: list[int], target: int) -> int:
    def binary_search(left: int, right: int) -> int:
        if left > right:
            return -1

        mid = left + (right - left) // 2

        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            return binary_search(mid + 1, right)
        else:
            return binary_search(left, mid - 1)

    return binary_search(0, len(nums) - 1)
```

| Metric | Value |
|---|---|
| **Time** | O(log n) |
| **Space** | O(log n) — call stack depth |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| Empty array `[]` | Returns `-1` |
| Single element `[5]`, target = 5 | Returns `0` |
| Single element `[5]`, target = 3 | Returns `-1` |
| Target at first index | Returns `0` |
| Target at last index | Returns `len(nums) - 1` |
| Target not in array | Returns `-1` |
| Even-length array | Works correctly — `mid` truncates down |
| Odd-length array | Works correctly — `mid` truncates down |

---

## 4. Interview Talking Points
- "Binary search requires a **sorted** dataset — the key invariant. In data engineering, we exploit this in sorted partitioned tables, B-tree indexes, and range scans."
- "The `mid = left + (right - left) // 2` formula avoids integer overflow in languages like C/C++/Java where `(left + right)` can exceed `INT_MAX`. Python handles arbitrary integers, but the pattern is still best practice."
- "DE connection: Binary search is used in sorted data lookup — index scans on partitioned tables, searching within sorted log entries, and efficient key-value lookups in key-value stores."
- "The loop condition `left <= right` is critical: using `left < right` causes off-by-one errors that skip the last remaining element."

---

## 5. Interview Follow-Up Questions
**Q1. What if the array contains duplicates? How do you find the first or last occurrence?**
-> Use modified binary search: when `nums[mid] == target`, don't return immediately. Instead, if searching for the first occurrence, set `right = mid - 1` and remember `mid` as a candidate. For the last occurrence, set `left = mid + 1`. This still runs in O(log n).

**Q2. How would you adapt binary search for a nearly sorted array (each element at most k positions from its sorted position)?**
-> For small k, linear search from the expected position is O(k). For larger k, you can use a modified binary search with adjusted bounds or a min-heap approach.

**Q3. How does binary search apply to data engineering at scale?**
-> Binary search on sorted partitions enables efficient range queries in columnar stores (Parquet, ORC) and index skip scans in databases. It's the foundation of B-tree indexing.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Using `mid = (left + right) // 2` without overflow protection | Can overflow in C/C++/Java when both are large | Use `left + (right - left) // 2` |
| Using `left < right` instead of `left <= right` | Misses the last single-element case | Use `left <= right` |
| Forgetting `+1` or `-1` in boundary update (`left = mid` instead of `left = mid + 1`) | Causes infinite loop when `mid` equals `left` or `right` | Always add/subtract 1 to ensure progress |
| Not handling empty array | Causes IndexError on `len(nums) - 1` or `nums[mid]` | Add guard clause: `if not nums: return -1` |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| Find Minimum in Rotated Sorted Array | Extends binary search to rotated arrays |
| Search in Rotated Sorted Array | Binary search with twist — determine sorted half |
| Koko Eating Bananas | Binary search on answer space |
| Sqrt(x) | Binary search for the largest integer whose square ≤ x |
| Find First and Last Position of Element in Sorted Array | Binary search with duplicate handling |
| Peak Element | Binary search using slope instead of value comparison |

---

*Last updated: 2025*
