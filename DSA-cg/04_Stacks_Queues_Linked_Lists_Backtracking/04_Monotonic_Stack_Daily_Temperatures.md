# Monotonic Stack — Daily Temperatures

**Difficulty:** Medium  
**Topics:** Next greater element, decreasing stack, amortized analysis

## 1. Concept and when to use

A monotonic stack keeps values (or indices) in increasing or decreasing order. It is useful for next greater/smaller element, stock spans, histogram rectangles and temperature waiting-time problems.

For each day, find how many days until a warmer temperature. Store unresolved day indices in a decreasing-temperature stack.

## 2. Approaches

### Brute force

For each day, scan right until a warmer day appears: O(n²) worst case.

### Optimized monotonic stack

```python
def daily_temperatures(temperatures):
    answer = [0] * len(temperatures)
    stack = []  # indices with unresolved warmer day

    for i, temp in enumerate(temperatures):
        while stack and temperatures[stack[-1]] < temp:
            previous = stack.pop()
            answer[previous] = i - previous
        stack.append(i)

    return answer
```

O(n) time because every index is pushed and popped at most once; O(n) space.

## 3. Dry run

Input `[73,74,75,71,69,72,76,73]`:
- Day 0 (73) waits.
- Day 1 (74) resolves day 0: answer[0]=1.
- Day 2 (75) resolves day 1: answer[1]=1.
- Days 3 and 4 (71,69) remain.
- Day 5 (72) resolves day 4 after 1 day and day 3 after 2 days.
- Day 6 (76) resolves day 5, day 2 and remaining unresolved indices.
- Day 7 (73) does not resolve 76.
Result: `[1,1,4,2,1,1,0,0]`.

## 4. Interviewer–candidate discussion

**Interviewer:** The nested `while` looks quadratic. Why is it linear?

**Candidate:** Across the entire loop each index is pushed once and popped at most once, so total stack operations are O(n).

**Interviewer:** Why store indices instead of temperatures?

**Candidate:** The answer needs the distance between days, and indices let us retrieve the corresponding temperature.

## 5. Edge cases and mistakes

Empty input, strictly increasing, strictly decreasing, equal temperatures. The problem asks for *warmer*, so comparison is `<`, not `<=`.

## 6. Tests and variations

```python
assert daily_temperatures([73,74,75,71,69,72,76,73]) == [1,1,4,2,1,1,0,0]
assert daily_temperatures([30,40,50,60]) == [1,1,1,0]
assert daily_temperatures([60,50,40]) == [0,0,0]
assert daily_temperatures([70,70]) == [0,0]
```

Follow-ups: next greater element; circular next greater element; largest rectangle in histogram; stock span; next smaller element.
