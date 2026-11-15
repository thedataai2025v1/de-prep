# Generate Parentheses

**Difficulty:** Medium  
**Topics:** Backtracking, validity pruning, Catalan numbers

## 1. Problem statement

Generate all combinations of n pairs of parentheses that are balanced and correctly nested.

## 2. Approaches

### Brute force

Generate all 2^(2n) strings of length 2n, then validate each. This wastes time generating invalid prefixes.

### Optimized: build only valid prefixes

```python
def generate_parentheses(n):
    if n < 0:
        raise ValueError("n must be non-negative")

    results = []
    path = []

    def backtrack(open_count, close_count):
        if len(path) == 2 * n:
            results.append("".join(path))
            return

        if open_count < n:
            path.append("(")
            backtrack(open_count + 1, close_count)
            path.pop()

        if close_count < open_count:
            path.append(")")
            backtrack(open_count, close_count + 1)
            path.pop()

    backtrack(0, 0)
    return results
```

There are Catalan-number \(C_n\) outputs. Time is at least O(C_n·n) to materialize them; auxiliary recursion/path space O(n), excluding output.

## 3. Dry run

For n=2:
- Start `""`; add `(`.
- Add another `(`; now `"(("`.
- Add `)` because close count is below open count.
- Add `)` to form `"(())"`.
- Backtrack; from `"("`, add `)` to form `"()"`, then add `(` and `)` to form `"()()"`.
Results: `["(())", "()()"]`.

## 4. Interviewer–candidate discussion

**Interviewer:** Why can we add a closing parenthesis only when close_count < open_count?

**Candidate:** Otherwise the prefix would contain more closes than opens and can never be repaired by appending characters.

**Interviewer:** What is the output count?

**Candidate:** The nth Catalan number, \(C_n = \frac{1}{n+1}\binom{2n}{n}\).

## 5. Edge cases and mistakes

n=0 returns `[""]`; negative n rejected; ensure exactly n opens and closes. Forgetting to pop corrupts sibling branches.

## 6. Tests and variations

```python
assert generate_parentheses(0) == [""]
assert set(generate_parentheses(1)) == {"()"}
assert set(generate_parentheses(2)) == {"(())", "()()"}
assert len(generate_parentheses(3)) == 5
```

Follow-ups: multiple bracket types; generate only one valid sequence; rank/unrank Catalan objects; count solutions without materializing them.
