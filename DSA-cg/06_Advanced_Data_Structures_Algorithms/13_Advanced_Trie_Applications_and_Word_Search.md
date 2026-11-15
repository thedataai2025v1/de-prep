# Advanced Trie Applications and Word Search

**Level:** Medium–Hard | **Patterns:** Prefix tree, DFS + pruning

## 1. Concept and when to use it

A trie shares common prefixes across strings. Beyond basic insert/search/prefix lookup, tries can support autocomplete, dictionary filtering, and multi-word board search. In Word Search II, a trie prunes board DFS as soon as the current character path is not a prefix of any remaining word.

Use this combination when searching many words against a shared structure. Searching each word independently repeats work.

## 2. Approaches

### Brute force

Run a DFS from every board cell for every word. If the board has `R×C` cells and maximum word length `L`, the rough worst-case work can grow with the number of words times `R C 4^L`.

### Optimized: trie-guided DFS

```python
def find_words(board, words):
    if not board or not board[0] or not words:
        return []

    trie = {}
    terminal = "#"
    for word in words:
        node = trie
        for char in word:
            node = node.setdefault(char, {})
        node[terminal] = word

    rows, cols = len(board), len(board[0])
    found = []

    def dfs(r, c, parent):
        char = board[r][c]
        node = parent.get(char)
        if node is None:
            return

        word = node.pop(terminal, None)
        if word is not None:
            found.append(word)

        board[r][c] = None
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] is not None:
                dfs(nr, nc, node)
        board[r][c] = char

        if not node:
            parent.pop(char)

    for r in range(rows):
        for c in range(cols):
            dfs(r, c, trie)

    return found
```

## 3. Dry run

Board:
`[['o','a'], ['e','t']]`, words `["oat", "eat", "tea"]`.
- Trie shares branches where prefixes overlap.
- DFS from `o` follows `o→a`; no `t` adjacent to `a` in this board, so it stops.
- DFS from `e` can follow `e→a` only if adjacency and trie prefix permit it; no complete word is found in this small example.
- The board cell is restored after each DFS path, allowing other paths to reuse it.

A more useful positive test is board `[['o','a','t'], ['e','t','a']]` with words `["oat", "eat"]`; `o→a→t` finds `"oat"`.

## 4. Complexity

Trie construction is `O(total characters in words)`. Search is data-dependent; worst-case DFS can still be exponential in maximum word length per starting cell, but prefix pruning substantially reduces repeated exploration. Trie space is `O(total characters)`.

## 5. Edge cases and mistakes

- Empty board, empty first row, and empty word list.
- Rectangular board assumption: this code uses the first row's width; validate rectangular input if it is not guaranteed.
- Restore board cells after recursive exploration.
- Do not reuse a cell in the same path.
- Duplicate words are returned once because the terminal marker is removed.
- Mutating the trie to prune exhausted branches is an optimization; ensure it does not remove a branch still needed elsewhere.

## 6. Interviewer–candidate discussion

**Interviewer:** Why is a trie better than a set of words?

**Candidate:** A set checks completed strings. The trie tells DFS immediately whether the current prefix can lead to any target word, pruning dead paths early.

**Interviewer:** Why remove the terminal marker after finding a word?

**Candidate:** It prevents duplicate results when the same word can be formed along multiple paths. It also lets us remove exhausted trie branches.

## 7. Follow-ups

- Return all board paths, not just words.
- Support diagonal movement.
- Add wildcard characters.
- Implement autocomplete with top-k frequency ranking.
- Discuss memory trade-offs: dictionary nodes versus compact arrays or radix tries.

## 8. Tests

```python
board = [["o", "a", "t", "x"], ["e", "a", "t", "x"]]
result = set(find_words(board, ["oat", "eat", "tea", "oat"]))
assert "oat" in result
assert "eat" in result
assert len(result) == len(set(result))
assert find_words([], ["a"]) == []
assert find_words([["a"]], ["a", "b"]) == ["a"]
```

**Data-platform connection:** Tries are useful for prefix-based routing, autocomplete over metadata catalogs, and hierarchical key lookup. For large catalogs, discuss memory locality, compression, and update concurrency.
