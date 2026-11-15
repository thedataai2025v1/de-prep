# LRU Cache

**Difficulty:** Medium–Hard  
**Topics:** Hash map, doubly linked list, O(1) eviction

## 1. Problem statement

Design a cache with fixed capacity. `get(key)` returns the value or -1 and marks the key most recently used. `put(key,value)` inserts/updates and evicts the least recently used item when capacity is exceeded. Both operations should be O(1) average time.

## 2. Approaches

### Baseline

Use a list of key-value pairs. Lookup and moving an item to the front take O(n).

### Optimized: hash map + doubly linked list

Python's `OrderedDict` is a concise production option. The explicit implementation below demonstrates the data structures.

```python
class _Node:
    def __init__(self, key=None, value=None):
        self.key, self.value = key, value
        self.prev = self.next = None


class LRUCache:
    def __init__(self, capacity):
        if capacity < 0:
            raise ValueError("capacity must be non-negative")
        self.capacity = capacity
        self.nodes = {}
        self.left = _Node()   # least-recent side
        self.right = _Node()  # most-recent side
        self.left.next = self.right
        self.right.prev = self.left

    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _insert_most_recent(self, node):
        previous = self.right.prev
        previous.next = node
        node.prev = previous
        node.next = self.right
        self.right.prev = node

    def _make_recent(self, node):
        self._remove(node)
        self._insert_most_recent(node)

    def get(self, key):
        if key not in self.nodes:
            return -1
        node = self.nodes[key]
        self._make_recent(node)
        return node.value

    def put(self, key, value):
        if self.capacity == 0:
            return
        if key in self.nodes:
            node = self.nodes[key]
            node.value = value
            self._make_recent(node)
            return

        node = _Node(key, value)
        self.nodes[key] = node
        self._insert_most_recent(node)

        if len(self.nodes) > self.capacity:
            lru = self.left.next
            self._remove(lru)
            del self.nodes[lru.key]
```

Average O(1) get/put; O(capacity) space.

## 3. Dry run

Capacity 2:
- put(1,A): order LRU→MRU `[1]`
- put(2,B): `[1,2]`
- get(1): returns A; order `[2,1]`
- put(3,C): evicts 2; order `[1,3]`
- get(2): returns -1

## 4. Interviewer–candidate discussion

**Interviewer:** Why do we need both structures?

**Candidate:** The hash map gives O(1) average lookup by key. The doubly linked list gives O(1) removal and movement when the node is already known.

**Interviewer:** Why doubly linked rather than singly linked?

**Candidate:** We need to unlink an arbitrary known node in O(1), which requires its predecessor. A doubly linked node stores both links.

## 5. Edge cases and mistakes

Capacity zero, capacity one, update existing key, get changes recency, repeated eviction, keys with value 0. Do not evict before inserting unless carefully managing the capacity invariant.

## 6. Tests and follow-ups

```python
cache = LRUCache(2)
cache.put(1, 1); cache.put(2, 2)
assert cache.get(1) == 1
cache.put(3, 3)
assert cache.get(2) == -1
cache.put(4, 4)
assert cache.get(1) == -1
assert cache.get(3) == 3
assert cache.get(4) == 4

zero = LRUCache(0)
zero.put(1, 1)
assert zero.get(1) == -1
```

Follow-ups: TTL expiration; LFU policy; concurrency and locking; sharding; cache stampede; memory accounting and eviction observability.
