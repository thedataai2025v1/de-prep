# Search in Rotated Sorted Array

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Binary Search |
| **Companies** | Google, Meta, Amazon, Microsoft, Bloomberg |
| **Interview Frequency** | High |

---

## 1. Problem Statement
There is an integer array `nums` sorted in ascending order (with no duplicates) that has been rotated between 1 and `n` times. Given `nums` and an integer `target`, return the index of `target` if it is in `nums`, or `-1` if not. The algorithm must run in `O(log n)` time.

**Example:**
```
Input: nums = [4, 5, 6, 7, 0, 1, 2], target = 0
Output: 4
```

---

## 2. Approaches

### Approach 1: Modified Binary Search (Optimal)
At each step, at least one half of the array `[left..mid]` or `[mid..right]` is always sorted. Determine which half is sorted by comparing `nums[left]` with `nums[mid]`, then check if the target lies in the sorted half. If not, search the other half.

**Pros:**
- Single pass through the search space — all logic in one loop
- O(1) space — no auxiliary data structures or extra passes
- Handles both rotated and unrotated arrays with the same uniform logic

**Cons:**
- More complex conditional branching — easy to mix up the sorted/unsorted half checks
- Requires understanding which half is sorted at each step — harder to explain under pressure

```python
def search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if nums[mid] == target:
            return mid

        # Left half [left..mid] is sorted
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # Right half [mid..right] is sorted
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1
```

| Metric | Value |
|---|---|
| **Time** | O(log n) |
| **Space** | O(1) |

### Approach 2: Two-Step Approach (Alternative)
First find the rotation pivot (minimum element) using binary search, then perform standard binary search on the correct half.

**Pros:**
- Separates concerns — pivot finding and standard binary search are independent, well-known subproblems
- Easier to reason about and debug — each step is a standard algorithm
- Reusable: the pivot-finding step is the solution to "Find Minimum in Rotated Sorted Array"

**Cons:**
- Two separate O(log n) passes — slightly more total operations than Approach 1
- More lines of code overall — can feel verbose in an interview

```python
def search(nums: list[int], target: int) -> int:
    # Step 1: Find the pivot (smallest element)
    left, right = 0, len(nums) - 1
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] > nums[right]:
            left = mid + 1
        else:
            right = mid
    pivot = left

    # Step 2: Binary search in the correct half
    if nums[pivot] <= target <= nums[-1]:
        left, right = pivot, len(nums) - 1
    else:
        left, right = 0, pivot - 1

    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

| Metric | Value |
|---|---|
| **Time** | O(log n) — two O(log n) passes |
| **Space** | O(1) |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| Not rotated (already sorted): `[1, 2, 3, 4, 5]` | Works — the "sorted half" logic still applies |
| Single element `[5]`, target = 5 | Returns `0` |
| Single element `[5]`, target = 3 | Returns `-1` |
| Target not present in array | Returns `-1` |
| Rotation by 1 (last element moved to front): `[5, 1, 2, 3, 4]` | Works correctly |
| Rotation by n-1 (first element moved to end): `[2, 3, 4, 5, 1]` | Works correctly |
| Array with 2 elements: `[3, 1]` | Works — mid calculation and boundary checks hold |

---

## 4. Interview Talking Points
- "This problem tests whether you can **adapt binary search** when the total order is broken. The key insight: even in a rotated array, at least one half is always sorted — this preserves the binary search invariant."
- "DE connection: Searching in **circular buffers** and **ring buffers** in stream processing is analogous. Log rotation (e.g., rotated log files), Kafka partition offsets, and time-windowed data all involve searching in logically cyclic structures."
- "The comparison `nums[left] <= nums[mid]` (with `<=`) correctly handles the case where `left == mid` (two-element array or single remaining element)."
- "This pattern generalizes: any time you have a structure where a global property is broken but a local invariant holds in one partition, you can still binary search."

---

## 5. Interview Follow-Up Questions
**Q1. What if the array can contain duplicates? Can you still achieve O(log n)?**
-> No — in the worst case (e.g., `[1, 0, 1, 1, 1]`), you cannot distinguish which half is sorted. The best you can do is O(n) in the worst case, though average case remains O(log n) by skipping duplicates when `nums[left] == nums[mid] == nums[right]`.

**Q2. How would you find the minimum element in a rotated sorted array?**
-> Binary search comparing `nums[mid]` with `nums[right]`. If `nums[mid] > nums[right]`, the minimum is in `[mid+1, right]`; otherwise it's in `[left, mid]`. Runs in O(log n).

**Q3. How does this relate to real-world data engineering?**
-> Rotated arrays model **circular buffers** used in streaming systems (Kafka, Flink state stores), log rotation schemes, and ring buffers in time-series databases. Efficient search in these structures enables fast lookups of events within a window.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Using `nums[left] < nums[mid]` (strict `<`) | Fails when `left == mid` (two-element subarray) — left half incorrectly labeled unsorted | Use `nums[left] <= nums[mid]` (non-strict) |
| Confusing `<=` with `<` in target range checks | Off-by-one: `nums[left] < target < nums[mid]` misses the case where `target == nums[left]` | Use `nums[left] <= target < nums[mid]` |
| Forgetting the `else` branch for sorted right half | Skips searching the unsorted half when the sorted half doesn't contain the target | Always handle both sorted and unsorted branches |
| Assuming rotation always happens — not checking unrotated case | Logic may fail on an already-sorted array | The `nums[left] <= nums[mid]` check naturally handles the unrotated case |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| Find Minimum in Rotated Sorted Array | Finds the pivot point — often step 1 of this problem |
| Search in Rotated Sorted Array II (with duplicates) | Same logic with O(n) worst-case for duplicate-heavy inputs |
| Binary Search (21) | Foundation — this is binary search on a "partially ordered" array |
| Find Peak Element | Uses similar "slope-based" binary search reasoning |
| Koko Eating Bananas | Binary search on answer space — same divide-and-conquer spirit |

---

*Last updated: 2025*
