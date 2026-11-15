# Merge Two Sorted Lists

**Difficulty:** Easy  
**Topics:** Linked List, Two Pointers  
**Study-plan section:** Linked List  
**Problem ID:** 24

## 1. Problem Statement

Merge two sorted singly linked lists into one sorted list by reusing the nodes and return its head.

**Example:**
```text
list1 = 1→2→4; list2 = 1→3→4
Output = 1→1→2→3→4→4
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Copy values into an array, sort them and build a new linked list.

```python
def merge_two_lists(list1, list2):
    values = []
    while list1:
        values.append(list1.val); list1 = list1.next
    while list2:
        values.append(list2.val); list2 = list2.next
    values.sort()
    dummy = cur = ListNode(0)
    for x in values:
        cur.next = ListNode(x); cur = cur.next
    return dummy.next
```

- **Complexity:** O((m+n) log(m+n)) time; O(m+n) space

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** Use a dummy head and two pointers. Append the smaller current node, then attach the remaining suffix.

```python
def merge_two_lists(list1, list2):
    dummy = ListNode(0)
    tail = dummy
    while list1 and list2:
        if list1.val <= list2.val:
            tail.next = list1; list1 = list1.next
        else:
            tail.next = list2; list2 = list2.next
        tail = tail.next
    tail.next = list1 or list2
    return dummy.next
```

- **Complexity:** O(m+n) time; O(1) auxiliary space (reusing nodes)

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Which approach should I choose?

**Recommended:** Use a dummy head and two pointers. Append the smaller current node, then attach the remaining suffix.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

Compare heads 1 and 1, append one; then compare 2 and 1, append 1; continue until one list ends, then append the remaining sorted suffix.

## 4. Interview Conversation

**Interviewer:** Why use a dummy node?

**Candidate:** It avoids a special case for initializing the head of the result. I always append after tail.

**Interviewer:** Why can you attach the remaining list directly?

**Candidate:** The remaining input list is already sorted and every value in it is no smaller than the last appended value.

**Interviewer:** Are you allocating new nodes?

**Candidate:** No. This implementation reuses existing nodes. If inputs must remain unchanged, I would create copies.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Unequal lengths are handled by attaching the remainder. Clarify whether node reuse is allowed. Recursive merge is possible but uses O(m+n) stack.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Test empty list, one empty list, duplicates and disjoint value ranges.

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

Merging sorted streams is a core ETL operation. Distributed merges require partition/order guarantees and careful handling of skew.

## 9. Final Interview Summary

**One-sentence explanation:** Use a dummy head and two pointers. Append the smaller current node, then attach the remaining suffix.

**Complexity to say aloud:** O(m+n) time; O(1) auxiliary space (reusing nodes)

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #24 in the supplied study plan.*