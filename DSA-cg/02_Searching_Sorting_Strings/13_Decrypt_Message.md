# Decrypt Message — Reverse Words

**Difficulty:** Easy–Medium  
**Topics:** String parsing, two pointers, transformation rules

## 1. Problem statement

A common interview variant asks to decrypt a sentence by reversing the order of words while preserving each word's characters. Example: `"data engineering rocks"` becomes `"rocks engineering data"`. Confirm whether repeated spaces and punctuation must be preserved. This lesson normalizes whitespace to single spaces.

## 2. Approaches

### Brute force

```python
def decrypt_brute(message):
    words = message.split()
    result = ""
    for word in reversed(words):
        result += word + " "
    return result.strip()
```
O(n²) worst-case due to repeated concatenation; O(n) output space.

### Optimized

```python
def decrypt_message(message):
    return " ".join(reversed(message.split()))
```
O(n) time and O(n) space for token list/output.

## 3. Dry run

`"  data   engineering rocks "` → `split()` gives `["data","engineering","rocks"]`; reverse to `["rocks","engineering","data"]`; join gives `"rocks engineering data"`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why does split without an argument help?

**Candidate:** It treats runs of whitespace as separators and ignores leading/trailing whitespace, matching the normalized-output contract.

**Interviewer:** What if exact spacing must be preserved?

**Candidate:** Then tokenization must retain separators, or I can reverse token spans while preserving whitespace. I would not use this normalized implementation.

## 5. Complexity and pitfalls

O(n) time, O(n) space. Clarify whether “decrypt” means reverse words, reverse characters, Caesar shift, or another cipher; the name alone is ambiguous. Do not invent a cipher rule.

## 6. Tests and variations

```python
assert decrypt_message("") == ""
assert decrypt_message("hello") == "hello"
assert decrypt_message("data engineering rocks") == "rocks engineering data"
assert decrypt_message("  data   engineering  ") == "engineering data"
```
Variation: preserve exact whitespace; reverse characters in each word; apply a documented Caesar shift. Ask for the encryption rule before coding.
