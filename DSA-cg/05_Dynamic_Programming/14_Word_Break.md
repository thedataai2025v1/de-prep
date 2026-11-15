# Word Break

**Difficulty:** Medium | **Pattern:** String DP, prefix reachability

## Problem

Given a string and a dictionary of words, determine whether the string can be segmented into a sequence of dictionary words. Words may be reused.

## Approaches

Brute force tries every split recursively and repeats suffix work.

DP state `dp[i]` means prefix `text[:i]` can be segmented. `dp[0]=True`.

```python
def word_break(text, words):
    word_set = set(words)
    if not text:
        return True
    if not word_set:
        return False

    max_len = max(map(len, word_set))
    dp = [False] * (len(text) + 1)
    dp[0] = True

    for end in range(1, len(text) + 1):
        start_min = max(0, end - max_len)
        for start in range(start_min, end):
            if dp[start] and text[start:end] in word_set:
                dp[end] = True
                break
    return dp[-1]
```

Let n be text length and L maximum word length. This implementation checks at most L candidate suffixes per position; slicing and hashing can cost up to O(L), giving a conservative O(n·L²) bound. Space O(n + dictionary size).

## Dry run

Text `"leetcode"`, words `{"leet","code"}`:
- dp[0]=true.
- At end 4, `"leet"` is found from reachable prefix 0; dp[4]=true.
- At end 8, `"code"` is found from reachable prefix 4; dp[8]=true.
Return true.

## Interviewer–candidate discussion

**Interviewer:** Why use a set?

**Candidate:** Average O(1) membership checks, rather than scanning a list of dictionary words for each split.

**Interviewer:** Why limit the start positions?

**Candidate:** No dictionary word is longer than L, so a valid final word must begin within the previous L characters.

## Edge cases and mistakes

Empty text convention; empty dictionary; empty string inside dictionary (ignore it); repeated words; Unicode; do not infer segmentation from character frequency alone.

## Tests

```python
assert word_break("leetcode", {"leet", "code"})
assert word_break("applepenapple", {"apple", "pen"})
assert not word_break("catsandog", {"cats", "dog", "sand", "and", "cat"})
assert word_break("", set())
```

Follow-ups: return one segmentation; count segmentations; trie optimization; minimum number of words; sentence generation with memoization.
