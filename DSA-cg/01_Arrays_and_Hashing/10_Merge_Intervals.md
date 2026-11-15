# Merge Intervals

**Difficulty:** Medium  
**Topics:** Array, Sorting, Intervals  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 10

## 1. Problem Statement

Merge all overlapping intervals and return a set of non-overlapping intervals covering the same ranges.

**Example:**
```text
intervals = [[1,3],[2,6],[8,10],[15,18]]
Output = [[1,6],[8,10],[15,18]]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Repeatedly compare and merge overlapping pairs until no overlap remains.

```python
def merge(intervals):
    pending = [list(x) for x in intervals]
    changed = True
    while changed:
        changed = False
        out = []
        while pending:
            cur = pending.pop()
            for i, other in enumerate(pending):
                if cur[0] <= other[1] and other[0] <= cur[1]:
                    pending[i] = [min(cur[0],other[0]), max(cur[1],other[1])]
                    changed = True
                    break
            else:
                out.append(cur)
        pending = out
    return sorted(pending)
```

- **Complexity:** Can be O(n³) in this repeated-scan implementation; O(n) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Sort by start; compare each interval to the last merged interval and extend its end when they overlap.

```python
def merge(intervals):
    if not intervals: return []
    intervals.sort(key=lambda x: x[0])
    out = [intervals[0][:]]
    for start, end in intervals[1:]:
        if start <= out[-1][1]:
            out[-1][1] = max(out[-1][1], end)
        else:
            out.append([start, end])
    return out
```

- **Complexity:** O(n log n) time; O(n) output space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Sort by start; compare each interval to the last merged interval and extend its end when they overlap.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Start with [1,3]. [2,6] overlaps, so extend to [1,6]. [8,10] and [15,18] do not overlap and are appended.

## 4. Interview Conversation

**Interviewer:** What is the overlap condition?

**Candidate:** For closed intervals sorted by start, the next start is <= the current end. If intervals are half-open, endpoint semantics may change.

**Interviewer:** Why is comparing only with the last output interval enough?

**Candidate:** Sorting means earlier intervals cannot start later than the current one; the last merged interval has the furthest relevant end among processed intervals.

**Interviewer:** What if one interval is contained in another?

**Candidate:** Taking max of the end points preserves the containing interval.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Insert interval into sorted list: O(n). Meeting rooms: sort events or use a min-heap. Streaming merge requires intervals to arrive sorted, otherwise buffering/sorting is needed.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Clarify whether [1,2] and [2,3] merge; that depends on closed versus half-open interval semantics.

Additional checks:
- Minimum valid input size.
- Maximum input size and performance.
- Duplicate values / repeated characters / repeated edges where applicable.
- Boundary values and absent target/result.

## 7. Common Interview Mistakes

- Coding before confirming constraints and output semantics.
- Explaining complexity without stating what `n`, `m`, `k` or other variables represent.
- Ignoring empty inputs, duplicates, boundary conditions or mutation requirements.
- Claiming a hash operation is worst-case O(1); it is generally expected/average O(1).
- Giving an optimized algorithm without explaining its invariant or correctness argument.

## 8. Data Engineering Connection

Used for SLA windows, sessionization, validity periods and maintenance windows; temporal boundary semantics must be explicit.

## 9. Final Interview Summary

**One-sentence explanation:** Sort by start; compare each interval to the last merged interval and extend its end when they overlap.

**Complexity to say aloud:** O(n log n) time; O(n) output space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #10 in the supplied study plan.*