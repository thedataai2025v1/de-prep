# Serialize and Deserialize Strings

**Difficulty:** Medium  
**Topics:** Encoding, delimiters, length-prefix format

## 1. Problem statement

Design `encode(list_of_strings) -> str` and `decode(str) -> list_of_strings` such that arbitrary strings—including empty strings and delimiter characters—round-trip without ambiguity.

## 2. Approaches

### Fragile baseline: delimiter join

```python
def encode_bad(items):
    return "|".join(items)

def decode_bad(payload):
    return payload.split("|")
```
This fails if an item contains `|`; escaping rules add complexity.

### Optimized: length-prefix encoding

Format each item as decimal length, `#`, then exact content.

```python
def encode(items):
    chunks = []
    for item in items:
        chunks.append(str(len(item)))
        chunks.append("#")
        chunks.append(item)
    return "".join(chunks)

def decode(payload):
    result = []
    i = 0
    while i < len(payload):
        j = payload.find("#", i)
        if j == -1:
            raise ValueError("missing length delimiter")
        length_text = payload[i:j]
        if not length_text.isdigit():
            raise ValueError("invalid length")
        length = int(length_text)
        start = j + 1
        end = start + length
        if end > len(payload):
            raise ValueError("truncated payload")
        result.append(payload[start:end])
        i = end
    return result
```
O(T) time and O(T) output/storage, where T is total encoded length.

## 3. Dry run

`["cat", "", "a#b"]` encodes as `3#cat0#3#a#b`. Decoder reads length 3 and `cat`; length 0 and empty string; length 3 and `a#b`. The `#` inside content is safe because the decoder consumes exactly the declared number of characters.

## 4. Interviewer–candidate discussion

**Interviewer:** Why is the delimiter safe inside the content?

**Candidate:** The delimiter is only searched for in the length prefix. After parsing the length, the decoder advances by exactly that many characters, so content is not parsed as metadata.

**Interviewer:** What about Unicode?

**Candidate:** This implementation measures Python string length in Unicode code points. If the protocol is byte-oriented, encode to bytes and prefix the UTF-8 byte length instead.

## 5. Edge cases and mistakes

Empty list, empty string, delimiter in content, multiple digits in length, malformed prefix, truncated payload, Unicode. Do not confuse character length and byte length. Validate untrusted payload sizes to avoid excessive allocation or denial-of-service.

## 6. Tests

```python
cases = [[], [""], ["cat", "", "a#b"], ["😀", "naïve"]]
for case in cases:
    assert decode(encode(case)) == case
assert decode("") == []
```

## 7. Follow-ups

1. Add versioning and schema evolution.
2. Make the format streaming rather than loading the full payload.
3. Add maximum item/total length limits.
4. Compare length-prefix, escaped delimiters, JSON and Protocol Buffers.

**Principal-level connection:** This is a miniature wire protocol. Discuss compatibility, validation, framing, encoding, resource limits and versioning.
