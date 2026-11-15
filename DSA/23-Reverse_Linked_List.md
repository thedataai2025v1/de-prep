# Reverse Linked List

| Category | Details |
|---|---|
| **Difficulty** | Easy |
| **Topics** | Linked List, Recursion |
| **Companies** | Google, Amazon, Microsoft, Meta, Apple, Bloomberg |
| **Interview Frequency** | High |

---

## 1. Problem Statement
Given the `head` of a singly linked list, reverse the list and return the new head of the reversed list.

**Example:**
```
Input: head = [1, 2, 3, 4, 5]
Output: [5, 4, 3, 2, 1]
```

---

## 2. Approaches

### Approach 1: Iterative — Three Pointers (Optimal)
Walk the list with three pointers: `prev` (initially `None`), `curr` (starts at `head`), and `next_node` (to save the next reference before overwriting `curr.next`). At each step, reverse the link by setting `curr.next = prev`, then advance all three pointers.

**Pros:**
- O(1) space — only three pointer variables, no call stack
- No recursion limit risk — safe for very long lists (e.g., 10⁵+ nodes)
- Better cache locality — sequential pointer traversal without function call overhead

**Cons:**
- Requires manual management of three pointers — easy to forget saving `next_node` before reassigning
- Slightly more verbose than the recursive version

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def reverseList(head: ListNode) -> ListNode:
    prev = None
    curr = head

    while curr is not None:
        next_node = curr.next  # save next
        curr.next = prev       # reverse link
        prev = curr            # advance prev
        curr = next_node       # advance curr

    return prev  # prev is the new head
```

| Metric | Value |
|---|---|
| **Time** | O(n) — one pass through the list |
| **Space** | O(1) — only three pointer variables |

### Approach 2: Recursive (Alternative)
Reverse the sublist starting from `head.next`, then attach `head` to the end of the reversed sublist. The base case is a single node or empty list.

**Pros:**
- Concise and elegant — directly expresses "reverse the rest, then attach" logic
- Fewer explicit state variables — the call stack manages position naturally
- Demonstrates understanding of recursion — valued in interviews for algorithmic thinking

**Cons:**
- O(n) call stack space — risks stack overflow on long lists (Python default limit ~1000)
- The `head.next.next = head` link reversal is harder to explain under pressure
- Not suitable for production code where list length is unbounded

```python
def reverseList(head: ListNode) -> ListNode:
    # Base case: empty list or single node
    if head is None or head.next is None:
        return head

    # Recursively reverse the rest of the list
    new_head = reverseList(head.next)

    # Make the next node point back to head
    head.next.next = head
    head.next = None  # break old link

    return new_head
```

| Metric | Value |
|---|---|
| **Time** | O(n) — one node processed per recursive call |
| **Space** | O(n) — call stack depth equals list length |

---

## 3. Edge Cases
| Edge Case | Behavior |
|---|---|
| Empty list (`head = None`) | Returns `None` |
| Single node `[1]` | Returns `[1]` — already reversed |
| Two nodes `[1, 2]` | Returns `[2, 1]` |
| Very long list (10⁵ nodes) | Iterative: no stack overflow. Recursive: may exceed recursion limit |
| All values identical `[3, 3, 3]` | Works — reversal depends on structure, not values |

---

## 4. Interview Talking Points
- "Reversing a linked list is a **fundamental building block** for many linked list algorithms: palindrome check (reverse half and compare), reorder list, rotate list by k places."
- "DE connection: Reversing data flow is useful in **pipeline backtracking** (undo operations in a DAG), **replaying events in reverse** for debugging, and **undo/redo patterns** in stateful stream processing (e.g., rolling back a window in Flink)."
- "The iterative approach is preferred in production: no recursion limit risk, constant space, and better cache behavior. The recursive approach is elegant but risks stack overflow on long lists."
- "In Python, linked lists are less common than arrays, but the pattern of pointer manipulation is critical for systems programming and understanding how languages like C handle memory."

---

## 5. Interview Follow-Up Questions
**Q1. How would you reverse a linked list in groups of k nodes?**
-> Use the single reverse as a building block. Keep track of the `prev_tail` (node before the current group) and the new `head` and `tail` of each reversed group. Requires careful pointer bookkeeping.

**Q2. How would you check if a linked list is a palindrome?**
-> Use the two-pointer technique to find the middle, reverse the second half, then compare node by node with the first half. Optionally restore the list by reversing the second half again.

**Q3. What happens to memory when you reverse a linked list iteratively?**
-> No memory is allocated or freed — only pointer values are reassigned. The total heap footprint remains constant. The garbage collector is not involved.

---

## 6. Common Mistakes
| Mistake | Why Wrong | Fix |
|---|---|---|
| Forgetting to save `curr.next` before reversing (`curr.next = prev` without saving) | Loses the reference to the rest of the list — list is broken after the first step | Always store `next_node = curr.next` before reassigning `curr.next` |
| Returning `curr` instead of `prev` as the new head | `curr` is `None` after the loop ends — returns an empty list | Return `prev`, which points to the last visited node (new head) |
| Recursive: forgetting `head.next = None` | Creates a cycle — `head` still points forward, forming a loop | Always set `head.next = None` after reversing the link |
| Recursive: base case only checks `head is None` | Misses the single-node case — recursion calls `head.next.next` on `None` | Check `head is None or head.next is None` |

---

## 7. Related Problems
| Problem | Connection |
|---|---|
| Reverse Linked List II (reverse from m to n) | Extends this to a sub-range of the list |
| Reverse Nodes in k-Group | Reverses the list in fixed-size chunks |
| Palindrome Linked List | Uses reverse to compare first and second halves |
| Merge Two Sorted Lists (24) | Linked list manipulation — similar pointer management |
| Reorder List | Reverse second half, then interleave with first half |
| Swap Nodes in Pairs | Reverses adjacent pairs — same pointer-swapping logic |

---

*Last updated: 2025*
