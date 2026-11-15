# Remove Duplicates from Sorted Array

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Array, Two Pointers |
| **Companies** | Amazon, Google, Meta, Microsoft, Apple |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an integer array `nums` sorted in non-decreasing order, remove the duplicates in-place such that each unique element appears only once. The relative order of the elements should be kept the same.

Return the number of elements in the array after removing the duplicates. The first `k` elements of `nums` must contain the unique elements in sorted order. Do not allocate extra space for another array.

**Example:**
```
Input:  nums = [1,1,2]
Output: 2 (nums becomes [1,2,_])

Input:  nums = [0,0,1,1,1,2,2,3,3,4]
Output: 5 (nums becomes [0,1,2,3,4,_,_,_,_,_])
```

---

## 2. Approaches

### Approach 1: Two Pointers (⭐ Optimal / Interview Choice)

**Idea:** Use a slow pointer to mark the position of the next unique element and a fast pointer to scan through the array. When the fast pointer finds a new unique element, place it at the slow pointer position.

**Pros:**
- Optimal O(n) time with a single pass through the array
- Zero extra space — fully in-place modification
- Simple logic that's easy to explain under pressure
- Works for any comparable element type (integers, strings, etc.)

**Cons:**
- Only works directly on sorted arrays — duplicates must be adjacent
- Modifies the input array, which may not be allowed in all interview variants
- Trailing elements beyond the unique count are left as garbage values

```python
def removeDuplicates(nums: list[int]) -> int:
    if not nums:
        return 0

    slow = 0
    for fast in range(1, len(nums)):
        if nums[fast] != nums[slow]:
            slow += 1
            nums[slow] = nums[fast]

    return slow + 1
```

| Metric | Value |
|---|---|
| **Time** | O(n) — single pass through the array |
| **Space** | O(1) — in-place modification |

---

### Approach 2: Set-Based Deduplication (Alternative)

**Idea:** Convert to a set to get unique values, then copy back. Simpler but uses extra space and doesn't guarantee in-place operation as required.

**Pros:**
- Extremely concise — fewer lines of code
- Handles unsorted input naturally without pre-sorting
- Easy to verify correctness at a glance

**Cons:**
- Violates the in-place constraint (O(n) extra space)
- Requires an intermediate data structure (set or dict)
- Copies data twice (build unique list, then copy back to nums)
- Not the expected answer in an interview asking for in-place solution

```python
def removeDuplicates(nums: list[int]) -> int:
    if not nums:
        return 0

    # Get unique elements while preserving order (sorted input guarantees order)
    unique = list(dict.fromkeys(nums))

    for i in range(len(unique)):
        nums[i] = unique[i]

    return len(unique)
```

| Metric | Value |
|---|---|
| **Time** | O(n) — to build and copy unique elements |
| **Space** | O(n) — extra list for unique elements |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| Empty array `[]` | Return 0 |
| Single element `[5]` | Return 1 |
| All same elements `[3,3,3]` | Return 1, array becomes `[3,_,_]` |
| No duplicates `[1,2,3]` | Return 3, array unchanged |

---

## 4. Interview Talking Points

- "Two pointer technique is ideal for sorted arrays where we can exploit the ordering to detect duplicates in a single pass."
- "In data engineering, this pattern mirrors deduplication in streaming pipelines: the fast pointer is the incoming stream, the slow pointer is the compacted output. Same idea applies when compacting sorted Parquet partitions or removing duplicate events in log aggregation."

---

## 5. Interview Follow-Up Questions

> **Be prepared for these — they differentiate junior from senior candidates.**

**Q1. What if the array is not sorted?**
→ Sorting first takes O(n log n) time, then apply the same two-pointer approach. Alternatively, use a hash set for O(n) time but O(n) space.

**Q2. What if each element may appear up to twice instead of once?**
→ Track a second slow pointer or maintain a count per element. The key change: advance slow only when `nums[fast] != nums[slow - 1]` (comparing two steps back to allow one duplicate).

**Q3. Why return `slow + 1` instead of `slow`?**
→ `slow` is a zero-based index pointing to the last unique element. Adding 1 gives the count of unique elements (i.e., the number of positions from 0 to slow inclusive).

**Q4. Can this approach work on a linked list?**
→ Yes — use a similar two-pointer pattern. Keep a `prev` pointer and skip nodes where `node.val == prev.val`. Adjust `prev.next` to skip the duplicate.

**Q5. What if you must not modify the input array?**
→ Use a set-based approach or create a copy of the array first. The interviewer will likely clarify whether in-place modification is required.

---

## 6. Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Using `list.pop()` in a loop | Shifts elements each time, making it O(n²) | Use two pointers to overwrite in-place |
| Forgetting to return `slow + 1` | Returns one less than the count of unique elements | Return `slow + 1` since slow is a zero-based index |
| Creating a new list | Violates the in-place constraint | Modify `nums` directly using index assignment |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Remove Element](https://leetcode.com/problems/remove-element/) | Same two-pointer pattern, different condition |
| [Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii/) | Variant allowing up to 2 occurrences |
| [Contains Duplicate](./18-Contains_Duplicate.md) | Similar deduplication concept without in-place requirement |

---

*Last updated: 2025 — for data engineering interview preparation.*