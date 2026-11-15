# Two Sum

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Array, Hash Map |
| **Companies** | Google, Amazon, Microsoft, Meta |
| **Interview Frequency** | Very High |

---

## 1. Problem Statement

Given an array of integers `nums` and an integer `target`, return the indices of the two numbers that add up to `target`. You may assume each input has exactly one solution, and you may not use the same element twice.

**Example:**
```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
```

---

## 2. Approaches

### Approach 1: Brute Force (Naive)

Check every pair of elements. For each element, scan the rest of the array for a complement that sums to the target.

**Pros:**
- Simplest approach to implement — no extra data structures
- Easy to explain and understand under pressure
- No pre-processing overhead

**Cons:**
- O(n²) time — fails on large inputs
- Redundant checks (checks both (i,j) and (j,i))
- Will not pass interview for anything beyond trivial inputs

```python
def twoSum(nums: list[int], target: int) -> list[int]:
    n = len(nums)
    for i in range(n):
        for j in range(i + 1, n):
            if nums[i] + nums[j] == target:
                return [i, j]
    return []
```

| Metric | Value |
|---|---|
| **Time** | O(n²) |
| **Space** | O(1) |

### Approach 2: Hash Map (One-Pass) (⭐ Optimal)

Use a dictionary to store each number and its index as we iterate. For each element, check if its complement (`target - nums[i]`) already exists in the map. If yes, return the stored index and the current index immediately.

**Pros:**
- O(n) time — single pass through the array
- O(1) lookup for complements via hash map
- Returns as soon as the pair is found — no wasted work
- This is the expected answer in interviews

**Cons:**
- Uses O(n) extra space for the hash map
- Slightly more code than brute force
- Hash collisions could degrade performance (rare in practice)

```python
def twoSum(nums: list[int], target: int) -> list[int]:
    seen = {}  # value -> index
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

| Metric | Value |
|---|---|
| **Time** | O(n) |
| **Space** | O(n) |

### Complexity Comparison

| Approach | Time | Space | Best For |
|---|---|---|---|
| **1. Brute Force** | O(n²) | O(1) | Understanding the problem first |
| **2. Hash Map (One-Pass)** | O(n) | O(n) | Interviews, production code |

---

## 3. Edge Cases

| Edge Case | Behavior |
|---|---|
| Duplicate values (e.g., `[3, 3]`, target = 6) | Hash map stores first index; second 3 finds complement in map |
| Negative numbers | Handled naturally — hash map works with any integer |
| Target is zero | Works: looks for pairs summing to 0 |
| Large arrays (10⁵ elements) | Brute force TLEs; hash map handles it in ~10ms |
| Same element used twice | Not allowed — `j` starts from `i+1` (brute force) or the complement is looked up before inserting current element (hash map) |

---

## 4. Interview Talking Points

- "I use a hash map here because it gives O(1) average-case lookup. For each element, I check if the complement (`target - num`) has already been seen. This reduces the problem from O(n²) to O(n)."
- "In data engineering, hash maps are the go-to structure for joining datasets by key, deduplication, and caching. The same one-pass pattern is used in streaming joins: maintain a buffer of seen keys and match incoming records against it."

---

## 5. Interview Follow-Up Questions

**Q1: What if there could be multiple valid pairs?**
-> Return all pairs by continuing the scan after finding the first match. Store results in a list. For the hash map approach, you'd need to handle duplicate values carefully — store lists of indices per value.

**Q2: What if the array is sorted?**
-> Use the two-pointer technique: one pointer at the start, one at the end. If the sum is too small, move the left pointer right; if too large, move the right pointer left. O(n) time, O(1) space — but you lose the original indices unless you store them with the values before sorting.

**Q3: What if you need to find k-sum (e.g., 3-sum) instead of 2-sum?**
-> Generalize recursively: sort the array, fix one element, then solve (k-1)-sum on the remainder. For 3-sum this becomes O(n²). See [3Sum](DSA/05-3Sum.md).

**Q4: What if the input is a stream of numbers?**
-> Maintain the hash map of seen values. For each new number, check if its complement is in the map. Same one-pass logic works naturally for streaming data.

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Checking `if complement in seen` AFTER inserting current element | May use the same element twice | Check BEFORE inserting `seen[num] = i` |
| Returning values instead of indices | Problem asks for indices, not values | Store and return indices from the map |
| Using `nums.index(complement)` | O(n) per lookup, making overall O(n²) | Use the hash map directly for O(1) lookup |
| Not handling duplicate values | If two elements have the same value, the second overwrites the first in the map | The overwrite is actually correct — we only need the earliest index |

---

## 7. Related Problems

| Problem | Connection |
|---|---|
| [3Sum](DSA/05-3Sum.md) | Extension to finding triplets that sum to zero |
| [4Sum](https://leetcode.com/problems/4sum/) | Further generalization of k-sum problems |
| [Two Sum II — Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) | Uses two-pointer technique instead of hash map |

---

*Last updated: 2025 — for data engineering interview preparation.*
