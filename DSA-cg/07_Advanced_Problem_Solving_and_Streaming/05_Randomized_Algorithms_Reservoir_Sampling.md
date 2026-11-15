# Randomized Algorithms: Reservoir Sampling

**Level:** Medium–Hard | **Patterns:** Uniform sampling from unknown-length streams

## 1. Concept and when to use it

Reservoir sampling selects a uniform random sample of `k` items from a stream whose length may be unknown, using only `O(k)` memory.

Use it when data arrives incrementally and cannot be stored in full: sampling logs, telemetry, records from a large file, or a stream with no known final count.

## 2. Approaches

### Brute force

Store every item, then sample `k` at the end. This uses `O(n)` memory.

### Optimized: Algorithm R

1. Put the first `k` items in the reservoir.
2. For the item at zero-based stream position `i >= k`, choose a random integer `j` from `[0, i]`.
3. If `j < k`, replace reservoir slot `j`.

```python
import random

def reservoir_sample(iterable, k, rng=None):
    if k < 0:
        raise ValueError("k must be non-negative")
    rand = rng if rng is not None else random
    reservoir = []

    for i, item in enumerate(iterable):
        if i < k:
            reservoir.append(item)
        else:
            j = rand.randint(0, i)
            if j < k:
                reservoir[j] = item

    return reservoir
```

## 3. Detailed dry run

Suppose `k=2` and stream is `A,B,C,D`.
- A and B fill the reservoir.
- For C at position 2, choose `j` uniformly from 0..2. If `j=1`, replace B with C.
- For D at position 3, choose `j` uniformly from 0..3. If `j=0`, replace A.
Each of the four items has the same final inclusion probability `2/4`.

Proof sketch: an item at position `i` enters with probability `k/(i+1)` and must survive each later replacement. The product of survival probabilities yields final inclusion probability `k/n`.

## 4. Complexity

For `n` stream items: `O(n)` time and `O(k)` memory. The algorithm is one-pass and does not require the stream length in advance.

## 5. Edge cases and common mistakes

- `k=0` returns an empty sample.
- `k>n` returns all available items.
- Reproducible tests should inject a seeded random generator.
- Sampling is uniform over stream positions; duplicate values are distinct records.
- Do not use `randint(0, i-1)`; position `i` must be included.
- This is not cryptographically secure sampling.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is every item equally likely to remain?

**Candidate:** Every new item has probability `k/(i+1)` of entering the reservoir. Once an item is in, each later step retains it with probability `1-k/(i+1)`. Multiplying those conditional probabilities gives a final inclusion probability of `k/n`.

**Interviewer:** What if I need a weighted sample?

**Candidate:** Basic Algorithm R assumes equal weights. Weighted reservoir sampling uses different algorithms and must define whether sampling is with or without replacement.

## 7. Follow-ups

- Sample one item uniformly from an unknown-length stream.
- Merge independent reservoirs from partitions, preserving uniformity.
- Produce a deterministic sample based on stable record IDs and a hash threshold.
- Discuss privacy and sampling bias in production telemetry.

## 8. Practical exercises and tests

```python
import random

assert reservoir_sample([], 3, random.Random(1)) == []
assert reservoir_sample([1, 2], 5, random.Random(1)) in ([1, 2], [2, 1])
assert reservoir_sample(range(100), 0, random.Random(1)) == []
sample = reservoir_sample(range(1000), 10, random.Random(42))
assert len(sample) == 10
assert len(set(sample)) == 10
assert all(0 <= value < 1000 for value in sample)
```

**Data-platform connection:** Reservoir sampling is useful for bounded-memory inspection of large event streams. Sampling policy must still account for tenant, time, geography, and other dimensions if a globally uniform sample would hide rare but important segments.
