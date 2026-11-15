# Reverse Linked List

**Difficulty:** Easy  
**Topics:** Linked List, Recursion, Iteration  
**Study-plan section:** Linked List  
**Problem ID:** 23

## 1. Problem Statement

Given the head of a singly linked list, reverse the list and return the new head.

**Example:**
```text
1 → 2 → 3 → 4 → 5 → None
Output: 5 → 4 → 3 → 2 → 1 → None
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Recursively reverse the rest of the list and attach the current node at the tail.

```python
def reverse_list(head):
    if head is None or head.next is None:
        return head
    new_head = reverse_list(head.next)
    head.next.next = head
    head.next = None
    return new_head
```

- **Complexity:** O(n) time; O(n) call-stack space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Iteratively maintain previous, current and next pointers; reverse one link at a time.

```python
def reverse_list(head):
    prev, cur = None, head
    while cur:
        nxt = cur.next
        cur.next = prev
        prev = cur
        cur = nxt
    return prev
```

- **Complexity:** O(n) time; O(1) auxiliary space

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Iteratively maintain previous, current and next pointers; reverse one link at a time.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

For 1→2→3, save next=2, point 1.next to None, move prev to 1 and cur to 2. Repeat until cur is None; prev is the new head 3.

## 4. Interview Conversation

**Interviewer:** What pointers do you need?

**Candidate:** Previous, current and next. I save next before reversing current.next so I do not lose the rest of the list.

**Interviewer:** What is the loop invariant?

**Candidate:** The list from prev is already reversed; cur is the head of the not-yet-processed suffix.

**Interviewer:** What are the edge cases?

**Candidate:** An empty list and a one-node list return unchanged.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Recursive solution uses O(n) stack. Reverse a subrange: locate boundaries, reverse that segment and reconnect. Detect cycles before operations if input guarantees are absent.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Never overwrite next before saving it. Ensure the final old head points to None.

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

Pointer manipulation is less common in distributed data engineering but tests state invariants, mutation safety and algorithmic fundamentals.

## 9. Final Interview Summary

**One-sentence explanation:** Iteratively maintain previous, current and next pointers; reverse one link at a time.

**Complexity to say aloud:** O(n) time; O(1) auxiliary space

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #23 in the supplied study plan.*