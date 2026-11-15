# Sliding Window

**Pattern:** Sliding Window  
**Use cases:** Contiguous subarrays/substrings, fixed-size aggregates, longest/shortest valid range.

## 1. Concept and when to use it

Maintain a window `[left, right]` and update its state as boundaries move. The technique avoids recomputing every contiguous range from scratch.

Two common forms:
- Fixed-size window: size k is predetermined.
- Variable-size window: expand and shrink while maintaining a validity condition.

A sliding window is appropriate when the state can be updated efficiently and pointer movement is logically safe.

## 2. Example: maximum sum of any window of size k

### Brute force

```python
def max_sum_bruteforce(nums, k):
    if k <= 0 or k > len(nums):
        raise ValueError("k must be between 1 and len(nums)")
    return max(sum(nums[i:i + k]) for i in range(len(nums) - k + 1))
```

Time O(nk) due to repeated summation; temporary slices also allocate memory.

### Optimized fixed window

```python
def max_sum_window(nums, k):
    if k <= 0 or k > len(nums):
        raise ValueError("k must be between 1 and len(nums)")

    window_sum = sum(nums[:k])
    best = window_sum

    for right in range(k, len(nums)):
        window_sum += nums[right] - nums[right - k]
        best = max(best, window_sum)

    return best
```

Time O(n), auxiliary space O(1).

## 3. Dry run

`nums=[2, 1, 5, 1, 3, 2]`, k=3:
- Initial `[2,1,5]` sum 8.
- Slide: remove 2, add 1 → 7.
- Slide: remove 1, add 3 → 9.
- Slide: remove 5, add 2 → 6.
- Maximum = 9.

## 4. Variable window example: longest substring without repeating characters

```python
def longest_unique_substring(s):
    last_seen = {}
    left = 0
    best = 0

    for right, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= left:
            left = last_seen[ch] + 1
        last_seen[ch] = right
        best = max(best, right - left + 1)

    return best
```

Expected O(n) time and O(min(n, alphabet-size)) state. The window invariant is that it contains no repeated character.

## 5. Edge cases and common mistakes

- k=0 or k larger than n.
- Negative values: fixed-size maximum sum still works.
- Empty string.
- Repeated character before the current left boundary.
- Updating left backwards when a character was seen in an old window.
- Applying sum-based shrink logic to arrays with negative values without proof.
- Off-by-one window length.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is the unique-substring algorithm linear?

**Candidate:** The right pointer advances once per character. The left pointer only moves forward, never backward. Each character is inserted into the last-seen map once, so total pointer movement and map operations are linear in expectation.

## 7. Follow-ups

- Minimum window substring.
- Longest substring with at most K distinct characters.
- Count anagrams of a pattern.
- Maximum in each sliding window (monotonic deque).
- Minimum-size subarray with sum at least target; clarify positivity assumptions.

## 8. Tests

```python
assert max_sum_window([2, 1, 5, 1, 3, 2], 3) == 9
assert max_sum_window([-5, -2, -8], 2) == -7
assert longest_unique_substring("abcabcbb") == 3
assert longest_unique_substring("bbbbb") == 1
assert longest_unique_substring("") == 0
assert longest_unique_substring("abba") == 2
```

## Data Engineering connection

A streaming window may be event-time or processing-time based. Real systems must additionally define watermarks, late-event handling, state expiry and whether a window is tumbling, sliding or session-based.
