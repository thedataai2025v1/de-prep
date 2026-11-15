# Top K Frequent Elements

**Difficulty:** Medium  
**Topics:** Array, Hash Table, Heap, Bucket Sort  
**Study-plan section:** Trees and Heaps  
**Problem ID:** 30

## 1. Problem Statement

Given an integer array nums and integer k, return the k most frequent elements. Output order does not matter.

**Example:**
```text
nums = [1,1,1,2,2,3], k = 2
Output = [1,2]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Count frequencies, then sort all distinct values by descending frequency.

```python
from collections import Counter
def top_k_frequent(nums, k):
    counts = Counter(nums)
    return [x for x, _ in counts.most_common(k)]
```

- **Complexity:** O(n + u log u) typical sorting; O(u) space, u = distinct values

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use a min-heap of size k, or bucket values by frequency. Bucket sort gives O(n) time because frequency is at most n.

```python
from collections import Counter
def top_k_frequent(nums, k):
    counts = Counter(nums)
    buckets = [[] for _ in range(len(nums)+1)]
    for value, freq in counts.items():
        buckets[freq].append(value)
    out = []
    for freq in range(len(buckets)-1, 0, -1):
        for value in buckets[freq]:
            out.append(value)
            if len(out) == k:
                return out
    return out
```

- **Complexity:** O(n) time; O(n+u) space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use a min-heap of size k, or bucket values by frequency. Bucket sort gives O(n) time because frequency is at most n.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Counts are 1→3, 2→2, 3→1. Bucket 3 contains [1], bucket 2 contains [2]; scanning buckets downward returns [1,2].

## 4. Interview Conversation

**Interviewer:** What is the baseline?

**Candidate:** Count frequencies, sort distinct values by frequency and take k. That is O(n+u log u).

**Interviewer:** Can you avoid sorting all distinct values?

**Candidate:** A min-heap of size k costs O(u log k), or bucket by frequency for O(n) time because no frequency exceeds n.

**Interviewer:** What if ties occur at the kth position?

**Candidate:** The prompt allows any valid ordering, but if deterministic tie-breaking is required, I must define and implement it.

**Interviewer:** Which would you choose?

**Candidate:** For small k relative to u, a size-k heap is memory efficient. For this constraint, bucket sort is linear and straightforward.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Heap approach: O(u log k) time, O(u+k) space including counts. Streaming top-k requires approximate heavy-hitter algorithms or maintaining bounded candidates. Distributed top-k needs local aggregation then global merge.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Clarify tie-breaking and whether output order matters. Test k=1, k=u and all values equally frequent.

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

Top-k is common in telemetry, logs and usage analytics. Distributed systems use local combiners and global aggregation; skew and approximate algorithms may matter.

## 9. Final Interview Summary

**One-sentence explanation:** Use a min-heap of size k, or bucket values by frequency. Bucket sort gives O(n) time because frequency is at most n.

**Complexity to say aloud:** O(n) time; O(n+u) space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #30 in the supplied study plan.*