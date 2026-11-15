# Streaming Median with Two Heaps

**Level:** Medium–Hard | **Patterns:** Two heaps, online order statistics

## 1. Concept and when to use it

Maintain the lower half of values in a max-heap and the upper half in a min-heap. Python's `heapq` is a min-heap, so negate values for the lower half.

Use this when values arrive one at a time and you need the median after each insertion: monitoring latencies, online metrics, and interactive analytics.

## 2. Approaches

### Brute force

Keep all values, sort after every insertion, and read the middle. Each update costs `O(n log n)`.

### Optimized: two heaps

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.lower = []  # max-heap encoded as negatives
        self.upper = []  # min-heap

    def add_num(self, value):
        if not self.lower or value <= -self.lower[0]:
            heapq.heappush(self.lower, -value)
        else:
            heapq.heappush(self.upper, value)

        # Keep lower equal in size to upper, or one larger.
        if len(self.lower) > len(self.upper) + 1:
            heapq.heappush(self.upper, -heapq.heappop(self.lower))
        elif len(self.upper) > len(self.lower):
            heapq.heappush(self.lower, -heapq.heappop(self.upper))

    def find_median(self):
        if not self.lower:
            raise ValueError("No values have been added")
        if len(self.lower) > len(self.upper):
            return float(-self.lower[0])
        return (-self.lower[0] + self.upper[0]) / 2
```

## 3. Detailed dry run

Insert `5, 2, 10, 4`.
- Add 5: lower `[-5]`, upper `[]`; median 5.
- Add 2: lower `[-5,-2]` (conceptually values 5,2); rebalance moves 5 to upper; lower contains 2, upper contains 5; median 3.5.
- Add 10: goes to upper; rebalance moves 5 to lower; lower values `[2,5]`, upper `[10]`; median 5.
- Add 4: goes to lower, then rebalance moves its largest value to upper; lower `[2,4]`, upper `[5,10]`; median 4.5.

## 4. Complexity

Insertion is `O(log n)`; median lookup is `O(1)`. Space is `O(n)` because exact median requires retaining the values (or equivalent order-statistic state).

## 5. Edge cases and common mistakes

- Empty median query should have a defined behavior; this implementation raises.
- Even count: average the two middle values.
- Rebalance after insertion.
- Maintain ordering invariant: every lower value is no greater than every upper value.
- Python integers can be negated safely, but be aware of numeric types and overflow in other languages.
- Removing arbitrary old values (sliding median) needs lazy deletion or an order-statistics tree; this implementation only inserts.

## 6. Interviewer–candidate discussion

**Interviewer:** What are the two invariants?

**Candidate:** The lower heap has either the same number of elements as upper or one extra. Also, the largest lower value is no greater than the smallest upper value.

**Interviewer:** Can this support a rolling median over the last 1,000 values?

**Candidate:** Not directly, because arbitrary removal is not supported. I would add delayed deletion and heap-size accounting, or use an ordered multiset/order-statistics tree where available.

## 7. Follow-ups

- Return median after every insertion.
- Support deletion of a value.
- Maintain rolling median for a fixed-size window.
- Compare exact median with approximate quantile sketches for distributed streams.

## 8. Practical exercises and tests

```python
finder = MedianFinder()
try:
    finder.find_median()
    raise AssertionError("Expected empty structure to fail")
except ValueError:
    pass

finder.add_num(5)
assert finder.find_median() == 5.0
finder.add_num(2)
assert finder.find_median() == 3.5
finder.add_num(10)
assert finder.find_median() == 5.0
finder.add_num(4)
assert finder.find_median() == 4.5
```

**Data-platform connection:** Online medians are useful for latency and payload-size monitoring. At high volume, exact global median is expensive to maintain across partitions; approximate quantile sketches are common.
