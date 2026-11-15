# Boggle Board

**Difficulty:** Medium  
**Topics:** Hash Set / Trie, DFS, Backtracking, Grid Search

## 1. Problem statement

Given a 2D board of characters and a list of words, return the words that can be formed by tracing adjacent cells. A move can go horizontally or vertically (some variants allow diagonals); a cell cannot be reused within one word.

Clarify:
- Are diagonal moves allowed?
- Is the board rectangular?
- Can a word use a cell more than once?
- Are letters case-sensitive?
- Should duplicate dictionary words appear once?

This lesson assumes four-direction adjacency, no cell reuse within a word, and unique output words.

## 2. Brute-force approach

For each dictionary word, search from every board cell using DFS. If word length is L and each cell has up to 4 neighbors, naive search is exponential in L in the worst case. It is acceptable for tiny boards but repeats work across words.

## 3. Optimized approach: trie + DFS

A trie shares prefixes across dictionary words. DFS follows board paths only while the current path is a valid dictionary prefix.

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.word = None

def find_words(board, words):
    if not board or not board[0]:
        return []

    root = TrieNode()
    for word in words:
        node = root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.word = word

    rows, cols = len(board), len(board[0])
    found = set()

    def dfs(r, c, parent):
        ch = board[r][c]
        node = parent.children.get(ch)
        if node is None:
            return

        if node.word is not None:
            found.add(node.word)
            node.word = None  # avoid reporting duplicates

        board[r][c] = None  # mark visited
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] is not None:
                dfs(nr, nc, node)
        board[r][c] = ch  # restore

        if not node.children and node.word is None:
            parent.children.pop(ch, None)

    for r in range(rows):
        for c in range(cols):
            dfs(r, c, root)

    return sorted(found)
```

The implementation temporarily mutates and restores the board. It assumes a rectangular board and that board cells are strings (not `None`).

## 4. Dry run

Board:
```text
o a a n
e t a e
i h k r
i f l v
```
Words: `oath`, `pea`, `eat`, `rain`.

- Starting at `o`, DFS follows `o → a → t → h`; `oath` is found.
- Prefix `p` is absent from the trie, so searches beginning with p are pruned immediately.
- `eat` can be found along an adjacent path.
- `rain` is not present as a valid path.
- Result contains `eat` and `oath`.

## 5. Complexity

Let B be the number of board cells, W total characters across dictionary words, and L maximum word length. Trie construction is O(W). Search is data-dependent; worst-case DFS can be exponential in L per starting cell, but trie-prefix pruning and shared prefixes often reduce work substantially. Trie space is O(W); recursion stack O(L); found output O(number of matches).

Do not claim a universal O(B × W) bound for all boards and dictionaries.

## 6. Edge cases and mistakes

- Empty board or empty word list.
- One-cell board.
- Word longer than number of cells.
- Duplicate words.
- Diagonal rule mismatch.
- Reusing a cell.
- Forgetting to restore board after DFS.
- Mutating shared trie state incorrectly.
- Ragged (non-rectangular) board.

## 7. Interviewer–candidate discussion

**Interviewer:** Why is a trie better than searching every word independently?

**Candidate:** It shares common prefixes and lets DFS stop as soon as the current path is not a prefix of any remaining word. That avoids repeating the same board traversal for words with shared prefixes.

## 8. Follow-ups

- Eight-direction adjacency.
- Return paths / coordinates, not only words.
- Support wildcard tiles.
- Very large dictionary: memory-aware trie or compact representation.
- Parallelize by starting cells, while controlling duplicate results and shared mutable trie state.

## 9. Tests

Use the sample board above and verify:
- `oath` and `eat` are returned.
- `pea` and `rain` are absent.
- Empty board returns `[]`.
- One cell `[['a']]` with words `['a', 'aa']` returns only `['a']`.

## Data Engineering connection

Trie prefix pruning is analogous to predicate pushdown: eliminate branches early when they cannot produce a valid result. In distributed search, partitioning and duplicate-result reconciliation become additional concerns.
