# Tips for Acing Technical Coding Interviews

**Type:** Interview preparation lesson  
**Audience:** Senior / Principal Data Engineer  
**Outcome:** Demonstrate structured problem solving, not just fast coding.

## 1. What the interviewer is evaluating

1. Problem decomposition and clarification.
2. Choice of data structures and algorithms.
3. Correctness reasoning and handling of edge cases.
4. Code quality, naming, modularity and testability.
5. Complexity analysis and trade-off awareness.
6. Communication, adaptability and response to hints.

At senior levels, interviewers also look for judgment: when a simple solution is preferable, how the approach behaves at scale, and what assumptions break in production.

## 2. A repeatable interview workflow

| Stage | What to do | Example language |
|---|---|---|
| Clarify | Confirm inputs, outputs, constraints and semantics | “Can the input contain duplicates or be empty?” |
| Restate | Explain the task in your own words | “I need to return the earliest…” |
| Baseline | Give a simple correct solution | “The direct approach compares every pair…” |
| Optimize | Identify repeated work or exploitable structure | “A hash map lets me avoid rescanning…” |
| Prove | State the invariant / correctness argument | “Before processing index i, the map contains…” |
| Code | Implement in small, readable steps | Explain meaningful decisions |
| Test | Use normal, boundary and adversarial cases | Empty, singleton, duplicates, extremes |
| Analyze | State time and auxiliary space | Define variables and assumptions |
| Reflect | Discuss alternatives and scale | “If the data does not fit in memory…” |

## 3. Communication rules

- Think aloud, but keep the narration structured.
- Do not narrate every keystroke.
- If stuck, identify what is known and isolate the uncertainty.
- Ask for a hint after making a concrete attempt, not after going silent.
- If the interviewer changes a requirement, state which part of the design changes.
- Never claim an operation is O(1) without qualifying average, amortized or worst-case behavior.

## 4. Test strategy

For each problem, test:
- Minimum valid input.
- Empty input if permitted.
- Duplicates and repeated values.
- Negative values and zero where relevant.
- Already sorted / reverse-sorted inputs.
- No-solution and multiple-solution cases.
- Maximum constraints and overflow / recursion-depth concerns.

## 5. Principal-level follow-ups

Expect questions such as:
- What if the input is too large for memory?
- Can the algorithm be streamed or parallelized?
- What is the partitioning or shuffle cost?
- How do you make processing idempotent?
- What happens with skew, late data, retries or partial failure?
- What metric would you monitor in production?

Do not force distributed systems into every answer. First solve the stated problem correctly; then explain when a distributed adaptation is justified.

## 6. Practice exercise

Take Two Sum and explain it in 90 seconds:
1. Clarify whether indices or values are required.
2. State O(n²) brute force.
3. Derive the hash-map solution.
4. Explain why lookup happens before insertion.
5. Test `[3, 3]`, target `6`.
6. State average O(n) time and O(n) space.

## Interview conversation

**Interviewer:** Why should I believe your optimized solution is correct?

**Candidate:** Before processing index `i`, the map contains every value from earlier indices and its index. If the complement exists, it must be at an earlier distinct index, so returning that pair is valid. If it does not exist, storing the current value preserves the invariant for the next iteration.

## Final checklist

- [ ] Clarified contract and constraints.
- [ ] Presented a correct baseline.
- [ ] Explained the optimization.
- [ ] Stated an invariant or correctness argument.
- [ ] Tested edge cases.
- [ ] Analyzed time and auxiliary space.
- [ ] Discussed only relevant trade-offs.
