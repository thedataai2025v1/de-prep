# Bloom Filter: Probabilistic Membership

**Level:** Medium–Hard | **Patterns:** Bit array, multiple hashes, probabilistic data structure

## 1. Concept and when to use it

A Bloom filter answers “possibly present” or “definitely absent.” It can produce false positives, but a correctly implemented standard Bloom filter has no false negatives for inserted items (assuming no deletion and no corruption).

Use it as a compact pre-check before expensive storage lookups: checking whether a key may exist in a remote database, reducing unnecessary disk reads, or filtering duplicate candidates.

## 2. Approaches

### Brute force

Store all keys in a set. Membership is exact but memory grows with the number and size of keys.

### Optimized: bit array with k hash positions

A production Bloom filter should use robust hash derivation and parameter sizing. The following educational implementation uses double hashing from Python's process-local `hash`; do not persist its bit array across processes unless you replace it with stable, specified hashes.

```python
class BloomFilter:
    def __init__(self, size, hash_count):
        if size <= 0 or hash_count <= 0:
            raise ValueError("size and hash_count must be positive")
        self.size = size
        self.hash_count = hash_count
        self.bits = bytearray((size + 7) // 8)

    def _positions(self, item):
        text = str(item).encode("utf-8")
        h1 = hash(text)
        h2 = hash(b"salt:" + text) | 1
        for i in range(self.hash_count):
            yield (h1 + i * h2) % self.size

    def add(self, item):
        for position in self._positions(item):
            self.bits[position // 8] |= 1 << (position % 8)

    def might_contain(self, item):
        return all(
            self.bits[position // 8] & (1 << (position % 8))
            for position in self._positions(item)
        )
```

## 3. Detailed dry run

With a small bit array, adding `"customer-17"` sets several bit positions.
- Querying `"customer-17"` checks those positions; all are set, so the answer is “possibly present.”
- Querying an unseen key may find at least one zero bit and return “definitely absent.”
- If all its positions happen to be set by other keys, it is a false positive.

## 4. Complexity

Insertion and membership are `O(k)` hash probes, where `k` is the number of hash functions. Space is `O(m)` bits, where `m` is the bit-array size.

For expected false-positive probability `p` with `n` inserted items, the standard approximate optimal sizing is:
- `m ≈ -n ln(p) / (ln 2)^2`
- `k ≈ (m/n) ln 2`

## 5. Edge cases and common mistakes

- False positives are possible; false negatives should not occur under the standard assumptions.
- A Bloom filter cannot confirm existence; always perform the authoritative lookup on “possibly present.”
- Hash functions must be stable if filters are persisted or shared across machines.
- Standard Bloom filters do not support safe arbitrary deletion; counting Bloom filters use counters at extra cost.
- Overfilling increases false-positive probability.
- Never use it as the sole source of truth for correctness-critical existence checks.

## 6. Interviewer–candidate discussion

**Interviewer:** How do you use a Bloom filter to reduce database reads?

**Candidate:** If it says definitely absent, skip the database lookup. If it says possibly present, query the database. False positives cost an extra lookup; false negatives would be unsafe, so the filter must be maintained correctly.

**Interviewer:** Can it tell me which key caused a bit to be set?

**Candidate:** No. It stores membership evidence, not key identity or provenance.

## 7. Follow-ups

- Derive optimal bit count and number of hash functions.
- Compare Bloom filters with exact sets and Cuckoo filters.
- Explain counting Bloom filters and deletion trade-offs.
- Discuss rebuilding and versioning filters during data refresh.

## 8. Practical exercises and tests

```python
bf = BloomFilter(size=1000, hash_count=4)
bf.add("customer-17")
assert bf.might_contain("customer-17") is True
# An absent item may be a false positive, so do not assert False here.
try:
    BloomFilter(0, 2)
    raise AssertionError("Expected invalid size")
except ValueError:
    pass
```

**Data-platform connection:** Bloom filters are used in storage engines and data systems to skip reads for keys that are definitely absent. Correctness depends on stable hashing, lifecycle management, and treating positive results as candidates rather than proof.
