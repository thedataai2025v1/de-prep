# Hash Tables: Fundamentals and Interview Reasoning

## 1. Concept

A hash table maps keys to values using a hash function and collision-resolution strategy. Python `dict` and `set` are hash-table-backed structures.

Expected average operations:
- Lookup: O(1).
- Insert: O(1).
- Delete: O(1).

Worst-case behavior can degrade due to collisions. Hash keys must be hashable; mutable lists and dictionaries cannot be keys.

## 2. When to use

- Membership checks.
- Frequency counts.
- Mapping a value to an index or metadata.
- Grouping records by key.
- Detecting duplicates.
- Memoization.

Do not use a hash table when sorted order, range queries or prefix search are central requirements unless combined with another structure.

## 3. Example: first unique character

### Brute force

For each character, count occurrences with repeated scans: O(n²).

### Optimized frequency map

```python
from collections import Counter

def first_unique_char(s):
    counts = Counter(s)
    for i, ch in enumerate(s):
        if counts[ch] == 1:
            return i
    return -1
```

Time O(n), space O(k), where k is distinct character count.

## 4. Dry run

`"swiss"`:
- Counts: s=3, w=1, i=1.
- Scan in order: s is repeated; w occurs once.
- Return index 1.

## 5. Common mistakes

- Using a mutable object as a key.
- Assuming worst-case O(1).
- Forgetting hash maps consume memory.
- Losing order when output order matters.
- Confusing missing key with a stored `None`.
- Mutating a dictionary while iterating over it.
- Failing to define case sensitivity or normalization.

## 6. Interviewer–candidate discussion

**Interviewer:** What is the trade-off between sorting and hashing for duplicate detection?

**Candidate:** A set gives expected O(n) time and O(n) space. Sorting can give O(n log n) time and may use implementation-dependent extra memory, but can be useful when the data is already sorted or memory constraints favor an in-place sort. Sorting also changes order unless a copy is used.

## 7. Follow-ups

- Group anagrams.
- Top K frequent elements.
- Longest consecutive sequence.
- Count subarrays with sum K.
- Implement an LRU cache using a dictionary plus doubly linked list.
- Discuss hash flooding and untrusted keys.

## 8. Tests

```python
assert first_unique_char("swiss") == 1
assert first_unique_char("aabb") == -1
assert first_unique_char("z") == 0
assert first_unique_char("") == -1
```

## Data Engineering connection

Hash-based deduplication is common in event pipelines. The key must represent business identity, and deduplication requires a retention horizon, deterministic conflict policy and handling of late arrivals. A process-local set does not provide global deduplication across workers.
