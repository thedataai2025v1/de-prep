# Linked List Fundamentals and Node Operations

**Difficulty:** Foundation–Medium  
**Topics:** Singly linked list, pointers, sentinel/dummy node

## 1. Concept and when to use

A singly linked list stores nodes where each node points to the next. Unlike arrays, nodes need not be contiguous in memory. Insertion after a known node is O(1), but indexing and searching are O(n).

Use linked lists when pointer manipulation is the core requirement or when the problem explicitly provides nodes. In production Python, built-in lists/deques are usually preferable unless linked-list semantics are required.

## 2. Node and common operations

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next


def build_linked_list(values):
    dummy = ListNode()
    tail = dummy
    for value in values:
        tail.next = ListNode(value)
        tail = tail.next
    return dummy.next


def to_list(head):
    values = []
    while head:
        values.append(head.val)
        head = head.next
    return values


def find_node(head, target):
    while head:
        if head.val == target:
            return head
        head = head.next
    return None


def insert_after(node, value):
    if node is None:
        raise ValueError("node cannot be None")
    node.next = ListNode(value, node.next)
```

Building and converting are O(n); search O(n); insertion after a known node O(1).

## 3. Brute force versus pointer-based operations

For inserting at index i, an array-like approach may shift O(n) values. A linked list must still traverse to index i (O(n)), but the pointer insertion itself is O(1). A dummy head simplifies operations that may change the head.

## 4. Dry run

List `10 → 20 → 40`. Insert 30 after node 20:
- New node points to 40.
- Node 20 points to new node 30.
- Result: `10 → 20 → 30 → 40`.

The order of pointer assignments matters: save the old `node.next` before overwriting it.

## 5. Interviewer–candidate discussion

**Interviewer:** Why use a dummy node?

**Candidate:** It removes special-case code for deleting or inserting at the head. The returned head is `dummy.next`.

**Interviewer:** What is the cost of accessing the kth element?

**Candidate:** O(k) traversal, unlike an array's O(1) indexing.

## 6. Edge cases and mistakes

Empty list, one node, head/tail insertion, duplicate values, accidental lost references, cycles. Do not assume a linked list is acyclic unless the problem says so.

## 7. Tests and exercises

```python
head = build_linked_list([10, 20, 40])
middle = find_node(head, 20)
insert_after(middle, 30)
assert to_list(head) == [10, 20, 30, 40]
assert to_list(None) == []
assert find_node(head, 99) is None
```

Follow-ups: doubly linked lists; sentinel head and tail; delete a node; compare locality and memory overhead with arrays.
