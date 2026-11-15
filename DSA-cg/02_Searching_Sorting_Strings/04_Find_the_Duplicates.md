# Find the Duplicates

**Difficulty:** Easy–Medium  
**Topics:** Hash sets, sorting, frequency counting

## 1. Problem statement

Return the distinct values that occur more than once. Clarify output order: first duplicate appearance, sorted order, or arbitrary order.

## 2. Approaches

### Brute force: compare every pair

```python
def duplicates_bruteforce(nums):
    found = set()
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] == nums[j]:
                found.add(nums[i])
    return found
```
O(n²) time, O(d) output space.

### Optimized: one-pass set tracking

```python
def find_duplicates(nums):
    seen, duplicates = set(), set()
    for value in nums:
        if value in seen:
            duplicates.add(value)
        else:
            seen.add(value)
    return duplicates
```
Expected O(n) time, O(n) space.

### Alternative: frequency map

```python
from collections import Counter

def find_duplicates_count(nums):
    return {value for value, count in Counter(nums).items() if count > 1}
```
Expected O(n) time and O(u) space, where `u` is the number of distinct values.

## 3. Dry run

For `[4, 2, 4, 3, 2, 2]`: first 4 and 2 enter `seen`; second 4 enters `duplicates`; 3 enters `seen`; second 2 enters `duplicates`; third 2 remains there. Result `{2, 4}`.

## 4. Interview discussion

**Interviewer:** Can you preserve order of first repeated appearance?

**Candidate:** Use a set for membership and a list for output, adding a value only the first time it repeats.

```python
def duplicates_in_repeat_order(nums):
    seen, emitted, result = set(), set(), []
    for x in nums:
        if x in seen and x not in emitted:
            result.append(x)
            emitted.add(x)
        seen.add(x)
    return result
```

**Interviewer:** Can you do it in O(1) extra space?

**Candidate:** Only under additional constraints, such as values restricted to `1..n` and permission to mutate the array. Otherwise a general-purpose solution needs storage or a sorting pass.

## 5. Complexity and pitfalls

Hash approach: expected O(n) time, O(n) space. Sorting approach: O(n log n), with sort's memory/mutation details depending on implementation. Do not confuse repeated occurrences with distinct duplicate values. Do not assume hash operations have strict worst-case O(1).

## 6. Edge cases and tests

```python
assert find_duplicates([]) == set()
assert find_duplicates([1]) == set()
assert find_duplicates([1, 1, 1]) == {1}
assert find_duplicates([1, 2, 3]) == set()
assert find_duplicates([4, 2, 4, 3, 2, 2]) == {2, 4}
```

**Follow-ups:** Return values appearing exactly `k` times; find the most frequent duplicate; handle a stream where all history cannot fit in memory. For streaming, discuss approximate sketches or external state with explicit accuracy guarantees.
