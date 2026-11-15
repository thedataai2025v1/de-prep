# Fast Track: Arrays, Two Pointers, Stacks and Sliding Window

**Type:** Pattern overview lesson

## 1. Arrays

Use arrays when order and indexed access matter. Typical operations:
- Index access: O(1).
- Search unsorted: O(n).
- Append: amortized O(1).
- Insert/delete near the front or middle: O(n).

Interview signals: contiguous range, in-place update, stable compaction, pair/triplet search.

## 2. Two pointers

Use two indices to reduce nested scanning:
- Opposite ends: sorted pair sum, container area.
- Same direction: remove duplicates, move zeros.
- Fast/slow: cycle detection, middle of linked list.

Example: sorted pair sum is O(n) time and O(1) extra space.

## 3. Stacks

A stack supports last-in-first-out behavior. Use it for:
- Matching brackets.
- Parsing nested structures.
- Monotonic next-greater / next-smaller problems.
- Iterative DFS.

Each element is often pushed and popped at most once, yielding O(n) time.

## 4. Sliding window

Use for contiguous ranges when the condition can be maintained as boundaries move.
- Fixed window: add entering item, remove leaving item.
- Variable window: expand right; shrink left while the invariant is violated.

Caution: a sum-based variable window often requires non-negative values. Negative values can invalidate the monotonicity assumption.

## 5. Pattern selection questions

| Clue | Candidate technique |
|---|---|
| Sorted array, pair from both ends | Two pointers |
| Longest/shortest valid contiguous range | Sliding window, if condition is maintainable |
| Next greater element | Monotonic stack |
| In-place stable filtering | Read/write pointers |
| Repeated range-sum queries | Prefix sum |
| Nested delimiters | Stack |

## 6. Practice sequence

1. Move Zeros to End.
2. Valid Parentheses.
3. Daily Temperatures.
4. Longest Substring Without Repeating Characters.
5. Container With Most Water.
6. Minimum Window Substring.

For each: explain brute force, derive the pattern, state the invariant and test edge cases.
