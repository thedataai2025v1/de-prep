# Group Anagrams

**Difficulty:** Medium  
**Topics:** Array, String, Hash Table, Sorting  
**Study-plan section:** Arrays and Hashing  
**Problem ID:** 11

## 1. Problem Statement

Given an array of strings, group all anagrams together. Output order does not matter. Anagrams contain the same characters with the same frequencies.

**Example:**
```text
strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
Output = [["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]
```

**What I would clarify before coding:**
- Are inputs empty or null, and what should be returned?
- May I modify the input?
- Are duplicates, negative values or ties possible?
- Does output order matter?

## 2. Approaches

### Approach 1: Baseline / Brute Force

**Idea:** Sort each word and use the sorted word as a canonical dictionary key.

```python
from collections import defaultdict
def group_anagrams(strs):
    groups = defaultdict(list)
    for word in strs:
        groups[''.join(sorted(word))].append(word)
    return list(groups.values())
```

- **Complexity:** O(n·k log k) time; O(n·k) space including output/keys

**Pros:**
- Directly mirrors the problem statement and is a useful correctness baseline.
- Usually easy to explain and validate with small examples.

**Cons:**
- Often performs repeated work and may not scale to the maximum constraints.

### Approach 2: Optimized Approach

**Idea:** For lowercase a-z, count 26 character frequencies and use the tuple as the key.

```python
from collections import defaultdict
def group_anagrams(strs):
    groups = defaultdict(list)
    for word in strs:
        counts = [0] * 26
        for ch in word:
            counts[ord(ch)-ord('a')] += 1
        groups[tuple(counts)].append(word)
    return list(groups.values())
```

- **Complexity:** O(n·(k+26)) = O(nk) time; O(n·(k+26)) space including output/keys

**Pros:**
- Uses the problem's structure to reduce unnecessary work.
- Suitable for the expected interview constraints.

**Cons / trade-offs:**
- Requires a clear invariant or data-structure choice.
- Additional memory or input mutation may be involved; state the contract explicitly.

### Approach 3: Python Counter

**Idea:** Use `Counter` to count characters and turn the sorted `(character, frequency)` pairs into an immutable key. This supports arbitrary characters without a fixed 26-character alphabet.

```python
from collections import defaultdict, Counter

def group_anagrams(strs):
    groups = defaultdict(list)
    for word in strs:
        key = tuple(sorted(Counter(word).items()))
        groups[key].append(word)
    return list(groups.values())
```

- **Time:** O(n × (k + u log u)), where `u` is the number of distinct characters in a word.
- **Space:** O(n × k), including output and dictionary keys.

**Pros:**
- Readable and concise.
- Supports arbitrary characters.
- Does not require a manually maintained fixed-size alphabet.

**Cons:**
- Sorting the frequency entries adds overhead.
- Counter and tuple construction create additional objects.
- For lowercase a-z, a fixed-size frequency array is more efficient.

### Which approach should I choose?

**Recommended:** For lowercase a-z, count 26 character frequencies and use the tuple as the key. For arbitrary character sets, use sorting or Counter.

Choose the optimized approach after explaining the baseline and proving why the optimization is correct. If the interviewer values simplicity over asymptotic improvement for tiny inputs, explain that trade-off.

## 3. Step-by-Step Dry Run

`eat`, `tea` and `ate` all produce the same frequency vector: one a, one e, one t. They therefore share a key. `tan`/`nat` share another key; `bat` forms its own group.

## 4. Interview Conversation

**Interviewer:** How do you group anagrams?

**Candidate:** I need a canonical representation that is identical for equal character multisets. Sorting each word provides one.

**Interviewer:** Can you make it linear in the word length?

**Candidate:** If characters are limited to lowercase a-z, count into a 26-element array and use its tuple as the key.

**Interviewer:** Why a tuple?

**Candidate:** A list is mutable and unhashable; a tuple is immutable and hashable, so it can be a dictionary key.

**Interviewer:** What if input is Unicode?

**Candidate:** The fixed 26-slot array no longer applies. I would use sorting or a frequency map with a stable key, and clarify normalization and case sensitivity.

## 5. Interview Follow-up Questions

**Q1. What variations or constraints would you discuss next?**

Why sorting works: identical multisets yield identical sorted sequences. Millions of rows: partition by canonical key; in Spark, groupBy can shuffle and skewed keys can become hot. Clarify whether spaces and punctuation count.

**Q2. What changes if the input size or execution environment changes?**

Revisit the time/space trade-off, mutation contract, data distribution and whether the algorithm can be streamed or partitioned. State any assumptions before proposing a distributed adaptation.

## 6. Edge Cases and Test Cases

Case-insensitive input requires normalization. Unicode may require NFC/NFKC normalization depending on business semantics. Group Anagrams differs from Valid Anagram because it groups many strings rather than comparing two.

Additional checks:
- Minimum valid input size.
- Maximum input size and performance.
- Duplicate values / repeated characters / repeated edges where applicable.
- Boundary values and absent target/result.

## 7. Common Interview Mistakes

- Coding before confirming constraints and output semantics.
- Explaining complexity without stating what `n`, `m`, `k` or other variables represent.
- Ignoring empty inputs, duplicates, boundary conditions or mutation requirements.
- Claiming a hash operation is worst-case O(1); it is generally expected/average O(1).
- Giving an optimized algorithm without explaining its invariant or correctness argument.

## 8. Data Engineering Connection

Canonical keys are useful in deduplication and grouping. Distributed groupBy incurs shuffle; profile skew and avoid assuming uniform key distribution.

## 9. Final Interview Summary

**One-sentence explanation:** For lowercase a-z, count 26 character frequencies and use the tuple as the key.

**Complexity to say aloud:** O(n·(k+26)) = O(nk) time; O(n·(k+26)) space including output/keys

**Principal-level signal:** Explain not only the algorithm, but also the input contract, correctness invariant, operational constraints, data volume assumptions and alternative trade-offs.

---

*Prepared for Principal Data Engineer interview practice. Problem #11 in the supplied study plan.*