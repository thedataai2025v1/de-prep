# Trie (Prefix Tree)

**Difficulty:** Medium  
**Topics:** Prefix search, shared prefixes, dictionary tree

## 1. Concept and when to use

A trie stores keys character by character. It supports prefix lookup efficiently and is useful for autocomplete, dictionary search and routing prefixes. For a word of length L, insert/search costs O(L) expected time with hash-map children.

## 2. Approaches

### Baseline: scan a word list

```python
def starts_with_scan(words, prefix):
    return [word for word in words if word.startswith(prefix)]
```
O(total characters scanned) per query; simple for small static lists.

### Optimized: trie

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_word = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_word = True

    def search(self, word):
        node = self._walk(word)
        return node is not None and node.is_word

    def starts_with(self, prefix):
        return self._walk(prefix) is not None

    def _walk(self, text):
        node = self.root
        for ch in text:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node
```
Insert/search/prefix: O(L) expected time; space O(total characters across inserted words) in the worst case.

## 3. Dry run

Insert `"car"` and `"cat"`: root has child `c`; node c has child a; node a has children r and t. Search `"ca"` succeeds as a prefix but is not a complete word unless `is_word` is true at that node.

## 4. Interviewer–candidate discussion

**Interviewer:** Why is a trie better than scanning all words?

**Candidate:** Prefix lookup depends mainly on prefix length, not on the number of stored words, though memory overhead is higher.

**Interviewer:** How would you reduce memory?

**Candidate:** Use compact child arrays for a fixed alphabet, radix/compressed tries, ternary search tries, or a static minimized automaton depending on workload.

## 5. Edge cases and mistakes

Empty word, duplicate insert, prefix that is also a complete word, Unicode, case normalization. Do not equate “prefix exists” with “word exists”.

## 6. Tests and variations

```python
trie = Trie()
trie.insert("apple")
assert trie.search("apple")
assert not trie.search("app")
assert trie.starts_with("app")
trie.insert("app")
assert trie.search("app")
assert trie.starts_with("")
```
Follow-ups: delete a word; return top-k autocomplete suggestions; wildcard search; support concurrent reads/writes.
