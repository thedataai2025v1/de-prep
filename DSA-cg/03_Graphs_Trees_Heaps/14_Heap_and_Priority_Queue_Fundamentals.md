# Heap and Priority Queue Fundamentals

**Difficulty:** Foundation–Medium  
**Topics:** Min-heap, max-heap, priority queue, heap operations

## 1. Concept and when to use

A heap is a complete binary tree represented compactly in an array. In a min-heap, the smallest element is at index 0. Use a heap when repeatedly extracting the smallest/largest item, maintaining top-k items, merging sorted streams or scheduling by priority.

## 2. Approaches

### Baseline: sort before every extraction

Sorting n items costs O(n log n); extracting the next item is then O(1) from the end. Repeating after every insertion is wasteful.

### Optimized: Python heapq

```python
import heapq

def heap_demo(values):
    heap = values[:]
    heapq.heapify(heap)             # O(n)
    smallest = heapq.heappop(heap) # O(log n)
    heapq.heappush(heap, 10)       # O(log n)
    return smallest, heap
```
Heapify O(n); push/pop O(log n); peek O(1). Space O(n).

Python `heapq` is a min-heap. A max-heap can be simulated by pushing negative numeric priorities, or by storing reversed comparable keys.

## 3. Dry run

Input `[7,2,5,1]` heapifies with 1 at root. Pop returns 1; push 3; the next pop returns 2. The exact internal array layout can vary while preserving the heap property.

## 4. Interviewer–candidate discussion

**Interviewer:** Why is heapify O(n), not O(n log n)?

**Candidate:** Most nodes are near the leaves and require little or no sifting. Summing the work by node height gives a linear total.

**Interviewer:** When is a heap not the right choice?

**Candidate:** If I need full sorted order, range queries, or fast arbitrary deletion/update, a sorted structure or balanced tree may be better.

## 5. Complexity and pitfalls

Heapify O(n); push/pop O(log n); peek O(1). Common mistakes: assuming the whole array is sorted, using mutable priorities without updating the heap, and forgetting tie-breakers for non-comparable payloads.

## 6. Tests and variations

```python
import heapq
h = [7, 2, 5, 1]
heapq.heapify(h)
assert heapq.heappop(h) == 1
heapq.heappush(h, 3)
assert heapq.heappop(h) == 2
```
Follow-ups: lazy deletion; stable priority queues using a sequence number; bounded top-k; merge k sorted streams; discuss backpressure in task schedulers.
