# Prefix Sum

**Pattern:** Prefix Sum  
**Use cases:** Range sums, subarray sums, cumulative balances, frequency-by-prefix problems.

## 1. Concept

Define `prefix[i]` as the sum of elements before index `i`. Then the sum of inclusive range `[left, right]` is:

`prefix[right + 1] - prefix[left]`

A prefix array makes repeated range-sum queries O(1) after O(n) preprocessing.

## 2. Brute force range sum

```python
def range_sum_bruteforce(nums, left, right):
    return sum(nums[left:right + 1])
```

Each query costs O(k), up to O(n).

## 3. Optimized range-sum queries

```python
def build_prefix(nums):
    prefix = [0]
    for value in nums:
        prefix.append(prefix[-1] + value)
    return prefix

def range_sum(prefix, left, right):
    return prefix[right + 1] - prefix[left]
```

Build O(n), each query O(1), auxiliary space O(n).

## 4. Dry run

`nums = [2, -1, 3, 4]`

Prefix = `[0, 2, 1, 4, 8]`

Range `[1, 3]` = `prefix[4] - prefix[1] = 8 - 2 = 6`, matching `-1 + 3 + 4`.

## 5. Subarray sum equals K

A subarray ending at current position sums to K when an earlier prefix equals `current_prefix - K`.

```python
from collections import defaultdict

def count_subarrays_sum_k(nums, k):
    freq = defaultdict(int)
    freq[0] = 1
    prefix = 0
    count = 0

    for value in nums:
        prefix += value
        count += freq[prefix - k]
        freq[prefix] += 1

    return count
```

Expected O(n) time, O(n) space. Initialize frequency of prefix zero to one to count subarrays beginning at index zero.

## 6. Dry run: count

`nums = [1, 1, 1]`, k=2:
- Prefix 1: need -1, count 0; freq[1]=1.
- Prefix 2: need 0, count becomes 1; freq[2]=1.
- Prefix 3: need 1, count becomes 2; freq[3]=1.
- Answer 2: subarrays `[1,1]` at indices 0–1 and 1–2.

## 7. Complexity

- Build prefix: O(n) time, O(n) space.
- Range query: O(1).
- Count subarrays with hash map: expected O(n) time, O(n) space.

## 8. Edge cases and common mistakes

- Empty input and invalid range bounds.
- Negative values (prefix sum still works).
- Forgetting the leading zero.
- Mixing inclusive and exclusive boundaries.
- Using a set instead of frequency map when counting multiple matches.
- Integer overflow in fixed-width languages; Python integers are arbitrary precision.

## 9. Interviewer–candidate discussion

**Interviewer:** Why does sliding window fail for arbitrary integers?

**Candidate:** Negative values break the monotonic relationship between window expansion and sum. Prefix sums plus a frequency map do not require non-negative values.

## 10. Follow-ups

- Longest subarray with sum K.
- Subarray sum divisible by K (prefix remainder counts).
- 2D matrix region sums (2D prefix sum).
- Running account balance.
- Count subarrays with XOR equal to K (prefix XOR).

## 11. Tests

```python
p = build_prefix([2, -1, 3, 4])
assert range_sum(p, 1, 3) == 6
assert count_subarrays_sum_k([1, 1, 1], 2) == 2
assert count_subarrays_sum_k([1, -1, 0], 0) == 3
assert count_subarrays_sum_k([], 0) == 0
```

## Data Engineering connection

Prefix sums can support cumulative financial balances and partition-level aggregates. When computing globally across partitions, each partition's local prefix must be offset by the total of all preceding partitions.
