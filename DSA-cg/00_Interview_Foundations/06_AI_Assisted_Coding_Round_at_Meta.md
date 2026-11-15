# Bonus: AI-Assisted Coding Round at Meta

**Type:** Awareness and preparation lesson  
**Important:** Interview formats and permitted tools can change. Confirm the exact instructions in the current recruiter communication; do not assume AI tools are allowed in a live assessment.

## 1. What this lesson prepares you for

An AI-assisted coding exercise may evaluate whether you can:
- Define a precise problem and constraints.
- Use an AI assistant to accelerate exploration without outsourcing judgment.
- Detect incorrect, incomplete or inefficient generated code.
- Test edge cases and validate complexity.
- Explain and defend the final implementation.

This is a preparation framework, not a claim about a universal or current Meta interview policy.

## 2. A safe workflow when AI assistance is explicitly permitted

1. Read the task and write down the contract.
2. Attempt a short independent decomposition.
3. Ask the assistant for alternatives or edge cases, not merely a final answer.
4. Inspect every generated line.
5. Run tests against examples and adversarial cases.
6. Analyze complexity independently.
7. Be ready to explain and modify the code without AI.

## 3. Example prompt for code review

> Review this Python solution for correctness. Identify edge cases and complexity issues. Do not rewrite it immediately. First explain the invariant and provide tests that could falsify the implementation.

This prompt encourages verification rather than blind acceptance.

## 4. Risks to watch

- Plausible code with an off-by-one error.
- Incorrect assumptions about input constraints.
- Hashing or sorting used where ordering semantics differ.
- Hidden quadratic operations.
- Tests that merely repeat the example.
- Leaked confidential company or candidate information.
- Use of prohibited tools in an assessment.

## 5. Interview conversation

**Interviewer:** The assistant suggested this solution. How do you know it is correct?

**Candidate:** I treat the suggestion as a hypothesis. I derive the invariant, test boundary cases, compare it with a simple baseline, and independently analyze complexity. I can explain and change the implementation without relying on the assistant.

## 6. Principal-level perspective

AI-assisted development changes the speed of drafting, not accountability. The engineer remains responsible for correctness, security, privacy, maintainability, testing and production consequences.

## Checklist

- [ ] Confirm tool policy before using AI.
- [ ] Never paste confidential data or secrets.
- [ ] Independently understand the algorithm.
- [ ] Test adversarial cases.
- [ ] Verify complexity and library behavior.
- [ ] Be able to reproduce the solution unaided.
