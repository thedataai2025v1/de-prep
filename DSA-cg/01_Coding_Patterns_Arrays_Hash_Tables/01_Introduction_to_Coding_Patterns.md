# Introduction to Coding Patterns

**Type:** Core DSA lesson  
**Audience:** Senior / Principal Data Engineer  
**Outcome:** Recognize reusable algorithmic structures and justify when they apply.

## 1. What is a coding pattern?

A coding pattern is a reusable way to organize state and work. It is not a memorized code template. The right pattern follows from the problem's constraints, input structure and required output.

Examples:
- Hash map: remember previously seen values or frequencies.
- Two pointers: coordinate positions in a sequence.
- Sliding window: maintain a contiguous range.
- Prefix sum: answer cumulative/range-sum questions.
- Tortoise and hare: reason about cycles or relative pointer speed.
- Monotonic stack: maintain ordered candidates for next-greater/smaller queries.
- Binary search: exploit sorted order or a monotonic predicate.
- Heap: repeatedly select an extreme element.
- Backtracking: explore and undo choices.
- Dynamic programming: reuse overlapping subproblem results.

## 2. Pattern selection workflow

1. Identify whether the problem is about a value, pair, range, ordering, path, connectivity or sequence of choices.
2. Check constraints: sorted? contiguous? duplicates? positive-only? online or offline?
3. Establish a brute-force solution.
4. Ask what work is repeated.
5. Identify the smallest state needed to avoid repeating it.
6. Prove the invariant or recurrence.
7. Test the optimized approach against edge cases.

## 3. Pattern decision table

| Problem clue | Candidate pattern | Important qualification |
|---|---|---|
| Pair lookup among many values | Hash map / set | Define duplicate and index rules |
| Sorted sequence, pair from ends | Two pointers | Usually needs sorted order |
| Contiguous range with maintainable condition | Sliding window | Sum windows often need non-negative values |
| Many cumulative/range-sum queries | Prefix sum | Clarify inclusive/exclusive boundaries |
| Cycle in linked list / repeated state | Tortoise & hare | Requires deterministic next transition |
| Next greater element | Monotonic stack | Decide strict vs non-strict comparison |
| Values in known range 1..n | Cyclic placement | Validate domain and duplicates |
| Repeated min/max selection | Heap | Choose min/max and tie-breaking |
| Explore all valid combinations | Backtracking | Prune safely |
| Repeated subproblems | DP / memoization | State must capture all future-relevant information |

## 4. Worked example: Two Sum

Brute force checks all pairs: O(n²) time, O(1) extra space.

Optimized approach stores values already visited:
```python
def two_sum(nums, target):
    seen = {}
    for i, value in enumerate(nums):
        complement = target - value
        if complement in seen:
            return [seen[complement], i]
        seen[value] = i
    return []
```

Expected O(n) time and O(n) space. Looking up before inserting prevents using the same element twice.

## 5. Common interview mistakes

- Selecting a pattern based only on a keyword.
- Applying sliding window when negative values break monotonicity.
- Assuming hash lookups have guaranteed worst-case O(1).
- Forgetting duplicate semantics.
- Giving complexity without counting sorting or data-structure construction.
- Optimizing before defining the exact output contract.

## 6. Interviewer–candidate discussion

**Interviewer:** How do you decide between a sliding window and prefix sums?

**Candidate:** I first check whether the range condition can be maintained as the left and right boundaries move. If non-negative values make the sum monotonic, a variable sliding window may work. If values can be negative or I need arbitrary range-sum queries, prefix sums are a more general fit.

## 7. Practice exercises

For each, name the pattern, state the invariant, and explain why alternatives are less suitable:
- Two Sum
- Move Zeros to End
- Contiguous Subarray Sum
- Longest Substring Without Repeating Characters
- Linked List Cycle
- Find the Duplicate Number
- Daily Temperatures
- Kth Largest Element

## Test checklist

- [ ] Can explain the pattern without quoting a template.
- [ ] Can identify assumptions that make it valid.
- [ ] Can derive brute force first.
- [ ] Can state the invariant.
- [ ] Can explain time and auxiliary space.
