# How to Answer Any Coding Interview Question

**Type:** Reusable speaking framework

## 1. The 8-step answer

1. **Clarify:** Inputs, outputs, constraints, mutation, ordering and edge cases.
2. **Restate:** Summarize the task and confirm shared understanding.
3. **Baseline:** Offer the simplest correct approach.
4. **Optimize:** Identify the bottleneck and select a suitable pattern.
5. **Prove:** Explain an invariant, recurrence or correctness argument.
6. **Implement:** Write readable code with deliberate variable names.
7. **Validate:** Dry-run normal and adversarial examples.
8. **Analyze:** State time, auxiliary space and trade-offs.

## 2. Reusable phrases

- “Before coding, I want to confirm the input constraints.”
- “The straightforward approach is correct but costs…”
- “The repeated work is…”
- “I can maintain the following invariant…”
- “Let me test the smallest valid input and a duplicate-heavy case.”
- “The time complexity is…, and auxiliary space excludes the output.”
- “If the data exceeds memory, the design would need…”

## 3. Example: Valid Parentheses

**Clarify:** Are only `()[]{}` present? Can other characters occur?

**Baseline:** Repeatedly remove adjacent valid pairs; potentially O(n²).

**Optimized:** Use a stack. Push opening brackets. On a closing bracket, verify the top matches. At end, stack must be empty.

```python
def is_valid(s):
    pairs = {")": "(", "]": "[", "}": "{"}
    stack = []

    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False
        else:
            return False  # adjust if non-bracket characters are allowed

    return not stack
```

Time O(n), space O(n).

## 4. If you get stuck

- Write a small example and manually track state.
- Identify what information must be remembered.
- Ask whether input order or sorting can help.
- Look for a monotonic property, repeated state or matching key.
- State a simpler partial solution before attempting a complex one.

## 5. Principal-level extension

After solving, discuss only relevant operational extensions: data size, streaming, partitioning, skew, retries, idempotency, observability and failure handling. Do not derail the coding answer with an architecture lecture.

## Self-assessment rubric

Score 1–5 for clarification, algorithm choice, correctness, implementation, testing, complexity and communication. Revisit any dimension below 4.
