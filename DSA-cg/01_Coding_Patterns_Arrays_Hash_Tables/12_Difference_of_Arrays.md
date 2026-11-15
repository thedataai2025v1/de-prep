# Difference of Arrays

**Difficulty:** Easy  
**Topics:** Hash Sets, Arrays, Membership

## 1. Problem statement

Given two arrays, return the distinct values that appear in the first array but not the second, and the distinct values that appear in the second but not the first.

Example: `a=[1,2,3]`, `b=[2,4,6]` → `[[1,3],[4,6]]`.

Clarify whether duplicates should be retained. This version returns distinct values in arbitrary set-derived order; a stable-order variant is also shown.

## 2. Brute force

```python
def array_difference_bruteforce(a, b):
    only_a = []
    only_b = []

    for x in a:
        if x not in b and x not in only_a:
            only_a.append(x)

    for x in b:
        if x not in a and x not in only_b:
            only_b.append(x)

    return [only_a, only_b]
```

Time can be O(nm + n² + m²), space O(n+m) for outputs.

## 3. Optimized with sets

```python
def array_difference(a, b):
    set_a = set(a)
    set_b = set(b)
    return [list(set_a - set_b), list(set_b - set_a)]
```

Expected O(n+m) time and O(n+m) space. Output order is not guaranteed.

## 4. Stable-order variant

```python
def array_difference_stable(a, b):
    set_a = set(a)
    set_b = set(b)

    only_a = []
    emitted_a = set()
    for x in a:
        if x not in set_b and x not in emitted_a:
            only_a.append(x)
            emitted_a.add(x)

    only_b = []
    emitted_b = set()
    for x in b:
        if x not in set_a and x not in emitted_b:
            only_b.append(x)
            emitted_b.add(x)

    return [only_a, only_b]
```

Expected O(n+m) time, O(n+m) space.

## 5. Dry run

`a=[1,2,2,3]`, `b=[2,4,4]`:
- Unique values in a: {1,2,3}.
- Unique values in b: {2,4}.
- a-only: {1,3}; b-only: {4}.
- Stable output: `[[1,3],[4]]`.

## 6. Edge cases and mistakes

- Empty arrays.
- Identical arrays.
- No overlap.
- Repeated values.
- Unhashable elements (e.g. lists) cannot be inserted directly into a set.
- Output ordering requirements not clarified.

## 7. Interviewer–candidate discussion

**Interviewer:** Why use two sets?

**Candidate:** Membership in a list is linear. Converting both arrays to sets allows expected constant-time membership and naturally removes duplicates, matching the stated distinct-value requirement.

## 8. Follow-ups

- Preserve duplicate counts (multiset difference).
- Return intersection and symmetric difference.
- Support unhashable records using a chosen key.
- Process two sorted arrays in O(n+m) time and O(1) auxiliary space with two pointers.

## 9. Tests

```python
assert set(array_difference([1, 2, 3], [2, 4, 6])[0]) == {1, 3}
assert set(array_difference([1, 2, 3], [2, 4, 6])[1]) == {4, 6}
assert array_difference([], [1]) == [[], [1]]
assert array_difference([1, 1], [1]) == [[], []]
assert array_difference_stable([1, 2, 2, 3], [2, 4, 4]) == [[1, 3], [4]]
```
