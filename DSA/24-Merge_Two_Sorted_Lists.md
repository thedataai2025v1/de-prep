# Merge Two Sorted Lists

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Linked List, Recursion, Two Pointers |
| **Companies** | Google, Meta, Amazon, Microsoft, Apple |
| **Interview Frequency** | High |

---

## 1. Problem Statement
Given the heads of two sorted linked lists, merge them into one sorted linked list by splicing together the nodes. Return the head of the merged list.

**Example:**
```
Input: list1 = [1, 2, 4], list2 = [1, 3, 4]
Output: [1, 1, 2, 3, 4, 4]
```

---

## 2. Approaches

### Approach 1: Iterative — Two Pointers (Optimal)
Use a dummy sentinel node to simplify head management. Compare the current nodes of both lists; attach the smaller one to the result and advance that pointer. When one list is exhausted, append the remainder of the other.

**Pros:**
- O(1) extra space — no call stack or sentinel node overhead beyond the dummy
- No recursion limit risk — safe for very long lists (10⁵+ nodes)
- Easy to explain step-by-step in an interview with a dry run on paper

**Cons:**
- Requires a dummy/sentinel node — adds a small setup step that candidates sometimes forget
- Slightly more verbose than the recursive version (4-5 lines of pointer management vs 3-4 lines)

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def mergeTwoLists(list1: ListNode, list2: ListNode) -> ListNode:
    # Sentinel/dummy node — avoids special-casing the head
    dummy = ListNode(0)
    tail = dummy  # tail tracks the last node of the merged list

    while list1 and list2:
        if list1.val <= list2.val:
            tail.next = list1
            list1 = list1.next
        else:
            tail.next = list2
            list2 = list2.next
        tail = tail.next  # advance tail

    # Attach the remaining nodes (only one of these is non-None)
    tail.next = list1 if list1 else list2

    return dummy.next  # skip the sentinel
```

| Metric | Value |
|---|---|
| **Time** | O(m + n) — each node visited exactly once |
| **Space** | O(1) — only a few pointer variables and a sentinel |

### Approach 2: Recursive (Alternative)
Compare the heads of both lists. The smaller node becomes the current node, and its `next` is set to the result of recursively merging the rest.

**Pros:**
- Concise — 3-4 lines of logic, directly expresses the "pick the smaller, merge the rest" idea
- No explicit sentinel node needed — the recursive base case handles empty lists naturally
- Demonstrates recursive thinking — valued for algorithmic elegance

**Cons:**
- O(m + n) call stack space — risks stack overflow on very long lists (Python default limit ~1000)
- The recursive structure is harder to trace during a live interview without drawing the call stack
- Function call overhead per node — slower in practice than iterative

```python
def mergeTwoLists(list1: ListNode, list2: ListNode) -> ListNode:
    # Base case: if either list is empty, return the other
    if list1 is None:
        return list2
    if list2 is None:
        return list1

    if list1.val <= list2.val:
        list1.next = mergeTwoLists(list1.next, list2)
        return list1
    else:
        list2.next = mergeTwoLists(list1, list2.next)
        return list2
```

| Metric | Value |
|---|---|
| **Time** | O(m + n) — each node compared once |
| **Space** | O(m + n) — call stack depth equals total merged length |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| Both lists empty (`None`, `None`) | Returns `None` |
| One list empty, other non-empty | Returns the non-empty list |
| Both lists have one node | Returns 2-node list in sorted order |
| All values in one list are smaller | Returns concatenation: all of list1 then all of list2 |
| Identical lists `[1, 2, 3]` and `[1, 2, 3]` | Merges to `[1, 1, 2, 2, 3, 3]` |
| Lists with duplicate values | Handles correctly — `<=` ensures stability |

---

## 4. Interview Talking Points
- "This is the core **merge step of merge sort** — the same algorithm applied to linked lists instead of arrays. For arrays, merge requires O(m+n) temporary space; for linked lists, it's O(1) because we splice nodes directly."
- "DE connection: Merging sorted streams is fundamental in data engineering: **merge joins** in SQL databases, **sorted stream merging** in Apache Spark (partition-level merging), and **compaction** in LSM trees (RocksDB, Cassandra, HBase). The linked-list version models merge without materializing an intermediate buffer."
- "The dummy/sentinel node pattern is a **standard trick** for linked list problems: it eliminates the special case of creating the head node. Always keep the dummy; return `dummy.next` at the end."
- "This problem generalizes to merging **k sorted lists** — a common follow-up that introduces a min-heap (priority queue) for O(n log k) time."

---

## 5. Interview Follow-Up Questions
**Q1. How would you merge k sorted linked lists?**
-> Use a min-heap (priority queue) of size k. Insert the head of each list, then repeatedly extract the smallest and push its next node. Runs in O(N log k) where N is the total number of nodes.

**Q2. How does this compare to merging two sorted arrays?**
-> For arrays, merge requires O(m+n) auxiliary space for the result. For linked lists, we splice existing nodes — O(1) extra space. The logic is identical (compare and advance).

**Q3. How does this relate to real-world data engineering?**
-> Merging sorted streams is used in **merge joins** (database query optimizers), **sorted file merging** (Parquet partition combines), and **stream union** operators (Flink, Spark). The linked-list merge is the zero-copy version of this pattern.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Returning `dummy` instead of `dummy.next` | Returns the sentinel node with `val = 0` — extra bogus node in the output | Always return `dummy.next` |
| Forgetting to advance `tail` after attaching a node | Creates a cycle — all new nodes point to the same `tail` | Always do `tail = tail.next` after linking |
| Recursive: not checking both base cases | Crashes on `None.next` when one list is empty | Check `if list1 is None: return list2` AND `if list2 is None: return list1` |
| Recursive: using `list1.next = ...` but returning `list2` | Mismatches — if list1 was chosen, must return list1 | Return the same node whose `.next` was updated |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| Merge k Sorted Lists | Generalizes this to k lists using a min-heap |
| Sort List | Merge sort on a linked list — uses this as the merge step |
| Merge Two Sorted Arrays | Same logic but requires O(m+n) auxiliary space |
| Reverse Linked List (23) | Another foundational linked-list manipulation |
| Linked List Cycle | Tests understanding of linked list traversal patterns |
| Add Two Numbers | Similar two-pointer iteration on linked lists |

---

*Last updated: 2025*
