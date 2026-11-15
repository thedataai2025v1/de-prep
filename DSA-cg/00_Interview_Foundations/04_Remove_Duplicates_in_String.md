# Remove Duplicates in String

**Difficulty:** Medium  
**Topics:** Strings, Hash Set, Two Pointers / Stable Filtering

## 1. Problem statement

Given a string, remove repeated characters while retaining only the first occurrence of each character and preserving original order.

**Example:** `"programming"` → `"progamin"`

Clarify whether comparison is case-sensitive, whether Unicode normalization matters, and whether the output should preserve first or last occurrences. The solution below preserves the first occurrence and uses exact code-point equality.

## 2. Approaches

### Baseline: membership in output

```python
def remove_duplicates_baseline(s):
    result = ""
    for ch in s:
        if ch not in result:
            result += ch
    return result
```

This is simple but repeated string membership and concatenation can lead to O(n²) time.

### Optimized: set plus list

```python
def remove_duplicates(s):
    seen = set()
    result = []

    for ch in s:
        if ch not in seen:
            seen.add(ch)
            result.append(ch)

    return "".join(result)
```

Expected O(n) time and O(k) auxiliary space, where k is the number of distinct characters.

## 3. Dry run

`"programming"`:
- Keep `p`, `r`, `o`, `g`, `a`, `m`, `i`, `n`.
- Skip later `r`, `m`, `g`.
- Result: `"progamin"`.

## 4. Correctness argument

A character is appended exactly when it has not appeared earlier. Therefore, the output contains each distinct character once, in the order of its first occurrence.

## 5. Interview conversation

**Interviewer:** Why not use `set(s)`?

**Candidate:** A set does not express the required stable first-occurrence order as clearly as a set-plus-list approach. I use the set only for membership and the list to preserve order.

**Interviewer:** What if case-insensitive deduplication is required?

**Candidate:** Normalize the comparison key, such as `ch.casefold()`, while retaining the original character for output. Unicode normalization may also be required depending on the business definition.

## 6. Edge cases

- Empty string.
- All characters unique.
- All characters identical.
- Mixed case (`"AaA"`).
- Spaces and punctuation.
- Unicode composed/decomposed forms.

## 7. Follow-ups

- Keep the last occurrence instead.
- Remove adjacent duplicates only.
- Remove duplicates from a stream with bounded memory.
- Return both the deduplicated string and duplicate counts.

## Data Engineering connection

Stable deduplication resembles event compaction by business key. In distributed systems, “first” must be defined using an ordering field (event time plus deterministic tie-breaker); partition-local sets alone cannot guarantee global first occurrence.
