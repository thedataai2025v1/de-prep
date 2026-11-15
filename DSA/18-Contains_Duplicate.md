# Contains Duplicate

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Array, Hash Table |
| **Companies** | Amazon, Google, Meta, Microsoft, Apple |
| **Interview Frequency** | High |

---

## 1. Problem Statement

Given an integer array `nums`, return `true` if any value appears at least twice in the array, and return `false` if every element is distinct.

**Example:**
```
Input:  nums = [1,2,3,1]
Output: true

Input:  nums = [1,2,3,4]
Output: false

Input:  nums = [1,1,1,3,3,4,3,2,4,2]
Output: true
```

---

## 2. Approaches

### Approach 1: Hash Set (⭐ Optimal / Interview Choice)

**Idea:** Insert each element into a set. If an element is already in the set, a duplicate exists.

**Pros:**
- Optimal O(n) average time with early exit on first duplicate found
- Simple, clean logic that's easy to implement under pressure
- Works regardless of whether input is sorted or unsorted
- Scalable to large datasets — processes elements incrementally

**Cons:**
- O(n) worst-case space — stores all elements if no duplicates exist
- Hash collisions (rare) could degrade to O(n²) in pathological cases
- Not suitable when memory is extremely constrained
- Does not tell you which elements are duplicated (just whether any are)

```python
def containsDuplicate(nums: list[int]) -> bool:
    seen = set()
    for num in nums:
        if num in seen:
            return True
        seen.add(num)
    return False
```

| Metric | Value |
|---|---|
| **Time** | O(n) — each insertion and lookup is O(1) average |
| **Space** | O(n) — worst case stores all elements |

---

### Approach 2: Sort Then Compare Adjacent (Alternative)

**Idea:** Sort the array first, then check if any adjacent elements are equal.

**Pros:**
- No extra data structure needed beyond the sort (in-place for some languages)
- O(1) extra space in languages with in-place sort (though Python's Timsort uses O(n))
- Deterministic worst-case performance — no hash collision concerns
- Can short-circuit as soon as a duplicate pair is found

**Cons:**
- O(n log n) time — slower than hash set for large inputs
- Modifies the input array (may not be allowed)
- Sorting overhead outweighs benefits for small to medium arrays
- Doesn't benefit from early exits in the unsorted portion before sorting completes

```python
def containsDuplicate(nums: list[int]) -> bool:
    nums.sort()
    for i in range(1, len(nums)):
        if nums[i] == nums[i - 1]:
            return True
    return False
```

| Metric | Value |
|---|---|
| **Time** | O(n log n) — sorting dominates |
| **Space** | O(1) or O(n) depending on sort implementation (Timsort uses O(n) in Python) |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| Empty array `[]` | Return `False` — no elements to duplicate |
| Single element `[1]` | Return `False` — only one occurrence |
| All unique `[1,2,3,4]` | Return `False` |
| All same `[5,5,5]` | Return `True` |
| Large array with late duplicate | Hash set detects it in O(n) regardless of position |

---

## 4. Interview Talking Points

- "The hash set approach is the go-to for deduplication in real-time processing. It's the foundation of how data engineers detect duplicates in streaming pipelines."
- "In data engineering, this problem maps directly to data quality checks: detecting duplicate records in ETL, finding anomaly patterns in log data, and ensuring referential integrity in warehouse loads. The sort-then-compare approach is useful when memory is constrained and you can process data in external sort (like MapReduce)."

---

## 5. Interview Follow-Up Questions

> **Be prepared for these — they differentiate junior from senior candidates.**

**Q1. What if you need to find the duplicate element, not just detect it?**
→ Return the element when `num in seen` is True. Or use Floyd's Cycle Detection if the array has specific constraints (values in range 1 to n-1).

**Q2. What if the input is already sorted?**
→ Skip the hash set entirely and use the two-pointer adjacent comparison approach for O(n) time with O(1) space.

**Q3. Why not use `len(nums) != len(set(nums))`?**
→ It works but creates a full set copy of all elements before comparing lengths — it can't short-circuit. The incremental `seen` set returns immediately on the first duplicate.

**Q4. What if memory is very limited (O(1) space required)?**
→ Sort the array in-place (if modification is allowed), then scan for adjacent duplicates. In Python, Timsort uses O(n) auxiliary space, so this trade-off is language-dependent.

**Q5. What if the values are in a known small range (e.g., 0 to 99)?**
→ Use a boolean array of size 100 instead of a hash set. This gives guaranteed O(1) lookups with no collision risk and lower constant factors.

---

## 6. Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Using nested loops O(n²) | Too slow for large inputs | Use hash set for O(n) or sort for O(n log n) |
| Forgetting to return `False` at end | Implicit None may cause issues | Explicitly return `False` after the loop |
| Using `len(nums) != len(set(nums))` | Works but creates a full set copy — less efficient for early detection | Check incrementally with `seen` set and return early |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [Remove Duplicates from Sorted Array](./15-Remove_Duplicates.md) | In-place deduplication variant |
| [Find All Duplicates in an Array](https://leetcode.com/problems/find-all-duplicates-in-an-array/) | Extended version requiring the actual duplicate values |
| [Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii/) | Adds a distance constraint between duplicates |

---

*Last updated: 2025 — for data engineering interview preparation.*