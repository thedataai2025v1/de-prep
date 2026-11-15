# Most Common Words

**Difficulty:** Easy–Medium  
**Topics:** Tokenization, hash maps, heap, top-k

## 1. Problem statement

Given text and integer k, return the k most frequent words. Clarify tokenization, case sensitivity, punctuation, and tie-breaking. This lesson uses lowercase alphabetic tokens and alphabetical tie-breaking.

## 2. Approaches

### Brute force: count and repeatedly select

```python
def top_words_brute(text, k):
    words = text.lower().split()
    counts = {w: words.count(w) for w in set(words)}
    result = []
    for _ in range(min(k, len(counts))):
        best = min(counts, key=lambda w: (-counts[w], w))
        result.append(best)
        del counts[best]
    return result
```
Counting with repeated `list.count` is O(nu); repeated selection adds up to O(ku). Space O(u).

### Optimized: Counter + heap

```python
import re
import heapq
from collections import Counter

def most_common_words(text, k):
    if k <= 0:
        return []
    words = re.findall(r"[a-z]+", text.lower())
    counts = Counter(words)
    return [word for word, count in heapq.nsmallest(
        min(k, len(counts)),
        counts.items(),
        key=lambda item: (-item[1], item[0])
    )]
```
Tokenization/counting O(n); selecting k from u unique words with heap is O(u log k) in the typical bounded-k formulation. The helper's implementation details may use O(u) storage for counts.

## 3. Dry run

Text `"Data data pipelines need data quality"` normalizes to `data,data,pipelines,need,data,quality`. Counts: data=3; others=1. For k=2, output `["data","need"]` because alphabetical order breaks the tie.

## 4. Interviewer–candidate discussion

**Interviewer:** Why not sort all unique words?

**Candidate:** Sorting costs O(u log u). If k is much smaller than u, a heap can reduce selection work to O(u log k). If k is close to u, sorting may be simpler.

**Interviewer:** What if the text is too large for memory?

**Candidate:** Use distributed aggregation or map-side combiners, then merge partial counts. For very high cardinality, discuss approximate heavy-hitter algorithms and error guarantees.

## 5. Edge cases and mistakes

Empty text, k<=0, k larger than unique count, punctuation, apostrophes, Unicode, case folding, deterministic ties. Do not leave tokenization rules implicit.

## 6. Tests and variations

```python
assert most_common_words("", 3) == []
assert most_common_words("a b a c b a", 2) == ["a", "b"]
assert most_common_words("Data, data! AI.", 2) == ["data", "ai"]
assert most_common_words("x y", 5) == ["x", "y"]
```
Variation: top-k events by customer, approximate streaming heavy hitters, stop-word filtering, locale-aware tokenization.
