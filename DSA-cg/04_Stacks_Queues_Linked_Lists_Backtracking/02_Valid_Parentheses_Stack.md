# Valid Parentheses

**Difficulty:** Easy–Medium  
**Topics:** Stack, matching delimiters, parsing

## 1. Problem and pattern

Given a string containing `()[]{}`, determine whether brackets are correctly matched and nested. A stack is natural because the most recently opened bracket must close first.

## 2. Approaches

### Brute force

Repeatedly remove adjacent pairs `()`, `[]`, `{}` until no change remains. This is simple but repeated string creation/scanning can cost O(n²).

### Optimized stack

```python
def is_valid_parentheses(text):
    opening = {"(", "[", "{"}
    match = {")": "(", "]": "[", "}": "{"}
    stack = []

    for ch in text:
        if ch in opening:
            stack.append(ch)
        elif ch in match:
            if not stack or stack.pop() != match[ch]:
                return False
        else:
            raise ValueError(f"unexpected character: {ch}")

    return not stack
```

Time O(n), auxiliary space O(n).

## 3. Dry run

Input `"{[()]}"`:
- `{` → stack `[{]`
- `[` → `[{, []`
- `(` → `[{, [, (]`
- `)` matches `(`; pop.
- `]` matches `[`; pop.
- `}` matches `{`; pop.
- Stack empty: valid.

For `"([)]"`, `)` expects `(` but top is `[`, so return false immediately.

## 4. Interviewer–candidate discussion

**Interviewer:** Why isn't counting opening and closing brackets enough?

**Candidate:** Counts ignore order. `([)]` has balanced counts but invalid nesting.

**Interviewer:** What should happen for non-bracket characters?

**Candidate:** Clarify the input contract. This implementation raises `ValueError`; another valid contract might ignore other characters.

## 5. Edge cases and mistakes

Empty string (valid), one bracket, early closing bracket, wrong type, unclosed opening bracket, unexpected characters. Do not pop before checking the stack is nonempty.

## 6. Tests and follow-ups

```python
assert is_valid_parentheses("")
assert is_valid_parentheses("()[]{}")
assert is_valid_parentheses("{[()]}")
assert not is_valid_parentheses("(]")
assert not is_valid_parentheses("([)]")
assert not is_valid_parentheses("(((")
```

Follow-ups: support quotes and escaped delimiters; validate HTML-like tags; report the first error position; build a parser for nested expressions.
