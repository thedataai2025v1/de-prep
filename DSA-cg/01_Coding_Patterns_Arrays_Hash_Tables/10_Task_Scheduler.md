# Task Scheduler

**Difficulty:** Medium  
**Topics:** Arrays, Hash Map, Greedy, Heap / Counting  
**Problem:** Given tasks represented by uppercase letters and a cooldown n, find the minimum CPU intervals needed to execute every task. The same task must have at least n intervals between executions. Idle intervals are allowed.

## 1. Clarify

- Is every task one unit of time?
- Can different task types execute in any order?
- Is the output minimum total intervals or an actual schedule?
- Is cooldown measured as n full intervals between same-type executions?

## 2. Brute-force direction

Enumerate possible valid schedules and choose the shortest. This becomes combinatorial and is not suitable for large inputs. A backtracking search can be useful only for tiny constraints or as a correctness oracle.

## 3. Optimized greedy formula

Let:
- `f_max` be the maximum task frequency.
- `count_max` be the number of task types with frequency `f_max`.

The minimum intervals are:

`max(len(tasks), (f_max - 1) * (n + 1) + count_max)`

```python
from collections import Counter

def least_interval(tasks, n):
    if n < 0:
        raise ValueError("cooldown must be non-negative")
    if not tasks:
        return 0

    frequencies = Counter(tasks)
    max_frequency = max(frequencies.values())
    count_max = sum(freq == max_frequency for freq in frequencies.values())

    frame_length = (max_frequency - 1) * (n + 1) + count_max
    return max(len(tasks), frame_length)
```

Time O(N + K), where N is task count and K distinct task types. Space O(K).

## 4. Dry run

Tasks `A A A B B B`, cooldown 2:
- max frequency = 3.
- number of tasks with max frequency = 2.
- Frame length = (3−1) × (2+1) + 2 = 8.
- There are 6 tasks, so answer = max(6, 8) = 8.
One valid schedule: `A B idle A B idle A B`.

## 5. Why the formula works

The most frequent task determines the minimum number of frames. There are `f_max - 1` gaps between its occurrences; each gap needs `n` cooldown slots plus the next frame position, giving `n+1` spacing. Other task types with the same maximum frequency occupy the final positions of the frame. If there are enough other tasks to fill all idle slots, total task count becomes the lower bound.

## 6. Common mistakes

- Returning idle slots rather than total intervals.
- Forgetting multiple task types can share the maximum frequency.
- Assuming a particular alphabetical ordering matters.
- Not handling empty input.
- Misinterpreting cooldown as requiring n idle intervals after the final task.

## 7. Interviewer–candidate discussion

**Interviewer:** Why take the maximum with total task count?

**Candidate:** The frame construction gives a lower bound caused by the most frequent task. But when there are many other tasks, they can fill the gaps, so the schedule cannot be shorter than the number of tasks themselves.

## 8. Follow-ups

- Return one valid schedule, not only its length.
- Different task durations.
- Multiple processors.
- Per-task cooldown values.
- Online arrival of tasks.
- Minimize weighted completion time.

## 9. Tests

```python
assert least_interval(["A", "A", "A", "B", "B", "B"], 2) == 8
assert least_interval(["A", "A", "A", "B", "B", "B"], 0) == 6
assert least_interval(["A"], 2) == 1
assert least_interval([], 2) == 0
assert least_interval(["A", "A", "A", "A", "B", "C", "D", "E"], 2) == 10
```

## Data Engineering connection

The cooldown resembles rate limiting or per-key throttling. Real schedulers must account for task durations, worker capacity, fairness, retries and time-based cooldown semantics; the interview formula assumes unit-duration tasks and one processor.
