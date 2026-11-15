# Top K Frequent Elements

| Category | Details |
|---|---|
| **Difficulty** | Medium |
| **Topics** | Array, Hash Table, Heap, Quickselect |
| **Companies** | Google, Amazon, Meta, Bloomberg |
| **Interview Frequency** | High |

---

## 1. Problem Statement
Given an integer array `nums` and an integer `k`, return the `k` most frequent elements. You may return the answer in any order.

**Example:**
```
Input: nums = [1, 1, 1, 2, 2, 3], k = 2
Output: [1, 2]
Explanation: 1 appears 3 times, 2 appears 2 times, 3 appears 1 time. Top 2 are [1, 2].
```

**Example 2:**
```
Input: nums = [1], k = 1
Output: [1]
```

---

## 2. Approaches

### Approach 1: Min-Heap of Size K (Optimal)
First count frequencies using a hash map. Then maintain a min-heap of size `k`. For each unique element, push its `(frequency, element)` tuple into the heap; if the heap exceeds size `k`, pop the smallest frequency (the heap root). After processing all elements, the heap contains the top `k` most frequent.

**Pros:**
- Single pass over the hash map after counting — no full sort needed
- Heap size is bounded by k, so space is small even when k << number of unique elements

**Cons:**
- O(n log k) due to heap operations — slower than bucket sort for large n with small k but acceptable for most cases
- Requires importing `heapq` and managing heap invariant manually

```python
import heapq
from collections import Counter

def topKFrequent(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)  # {element: count}

    # Min-heap: stores (frequency, element)
    heap = []
    for num, count in freq.items():
        heapq.heappush(heap, (count, num))
        if len(heap) > k:
            heapq.heappop(heap)  # Remove element with lowest frequency

    return [num for count, num in heap]
```

| Metric | Value |
|---|---|
| **Time** | O(n log k) — counting is O(n), each of up to n unique elements does a heap push/pop of O(log k) |
| **Space** | O(n) — hash map stores all unique elements, heap stores k |

### Approach 2: Bucket Sort by Frequency (Fastest — O(n))
After counting frequencies, create buckets where index `i` holds a list of all elements that appear `i` times. Iterate buckets from highest frequency to lowest, collecting elements until we have `k`. Since the maximum frequency is at most `n`, this is bounded.

**Pros:**
- O(n) time overall — no sorting or heap overhead; just linear bucket creation + iteration
- Simple logic once the bucket structure is set up

**Cons:**
- Bucket array uses O(n) space — can be wasteful if k is very small compared to n
- Wasteful when frequency distribution is sparse (many empty buckets)

```python
from collections import Counter

def topKFrequent(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)
    n = len(nums)

    # Buckets: index = frequency, value = list of elements with that frequency
    buckets = [[] for _ in range(n + 1)]
    for num, count in freq.items():
        buckets[count].append(num)

    # Collect top k from highest frequency down
    result = []
    for count in range(n, 0, -1):
        for num in buckets[count]:
            result.append(num)
            if len(result) == k:
                return result

    return result
```

| Metric | Value |
|---|---|
| **Time** | O(n) — counting + bucketing + collecting all in O(n) |
| **Space** | O(n) — hash map + buckets |

### Approach 3: Quickselect (Alternative)
Partition the unique elements by frequency using the quickselect algorithm to find the k-th most frequent element, then collect all elements with frequency >= that threshold.

**Pros:**
- O(n) average time without requiring bucket-like auxiliary structures proportional to input size
- In-place partitioning of the elements list avoids extra data structures beyond the frequency map

**Cons:**
- O(n²) worst-case time is possible (rare with random pivot selection, but hard to guarantee)
- Modifies the input elements list as a side effect

```python
import random
from collections import Counter

def topKFrequent(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)
    elements = list(freq.keys())

    def quickselect(left: int, right: int, k_smallest: int) -> None:
        if left == right:
            return

        pivot_idx = random.randint(left, right)
        pivot_freq = freq[elements[pivot_idx]]

        # Move pivot to end
        elements[pivot_idx], elements[right] = elements[right], elements[pivot_idx]

        # Partition: greater frequencies to the left
        store = left
        for i in range(left, right):
            if freq[elements[i]] > pivot_freq:
                elements[store], elements[i] = elements[i], elements[store]
                store += 1

        # Move pivot to sorted position
        elements[store], elements[right] = elements[right], elements[store]

        if store == k_smallest:
            return
        elif store > k_smallest:
            quickselect(left, store - 1, k_smallest)
        else:
            quickselect(store + 1, right, k_smallest)

    quickselect(0, len(elements) - 1, k)
    return elements[:k]
```

| Metric | Value |
|---|---|
| **Time** | O(n) average, O(n²) worst case (rare with random pivot) |
| **Space** | O(n) — hash map + elements list |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| `k = 1` | Return single most frequent element |
| `k = len(set(nums))` | Return all unique elements |
| All elements have the same frequency | Return any `k` elements |
| Single element array (`nums = [1], k = 1`) | Return `[1]` |
| Empty array (`nums = []`) | Return `[]` (though constraints typically guarantee `1 <= k <= len(nums)`) |
| Negative numbers in `nums` | Handled correctly by Counter |

---

## 4. Interview Talking Points
- "The **heap approach** is the most practical: O(n log k) is fast enough for any realistic input, and it's easy to implement and explain."
- "The **bucket sort approach** achieves optimal O(n) time and is the best theoretical answer. It works because frequencies are bounded integers in `[1, n]`."
- "DE connection: This is exactly **top-N analysis in data engineering** — finding the most active users, top-selling products, most frequent error codes in logs, or Pareto analysis (80/20 rule). The bucket sort approach scales especially well for log frequency analysis."

---

## 5. Interview Follow-Up Questions
**Q1. How would you handle a streaming input (elements arrive one at a time)?**
-> Maintain a running Counter and the min-heap of size k. For each new element, update its count in O(1) and adjust the heap if needed.

**Q2. What if `k` is very close to the total number of unique elements?**
-> The heap approach is still O(n log k). If k ≈ n, consider sorting or quickselect for better practical performance.

**Q3. What if you need the top k *in order of frequency*?**
-> The heap returns in arbitrary order. Sort the result by frequency: `sorted(heap, key=lambda x: -x[0])`.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Using a max-heap and popping k times | Works but O(n log n) — pushing all elements then popping k | Use a min-heap of size k for O(n log k) |
| Forgetting to count frequencies first | Can't determine frequency without counting | Always start with `Counter(nums)` or manual frequency map |
| Using `heapq` without understanding it's a min-heap | `(count, num)` works because we want smallest count at the top to discard it | Store `(count, num)` — smallest count is popped first |
| Returning before collecting exactly k elements | May return fewer or more than k | Check `if len(result) == k` after each addition |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| [Top K Frequent Words](https://leetcode.com/problems/top-k-frequent-words/) | Same approach but with alphabetical tiebreaker |
| [Sort Characters By Frequency](https://leetcode.com/problems/sort-characters-by-frequency/) | Same frequency counting, return all elements sorted by frequency |
| [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Uses the same quickselect or heap pattern |
| [Wiggle Sort II](https://leetcode.com/problems/wiggle-sort-ii/) | Uses median finding (related to quickselect) |

---

*Last updated: 2025*
