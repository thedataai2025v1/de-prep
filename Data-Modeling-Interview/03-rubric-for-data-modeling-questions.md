# Module 1: Overview

## Lesson 1.3 — Rubric for Data Modeling Questions

### How Interviewers Score You

Most companies use a rubric with 4–5 dimensions. Understanding it helps you self-evaluate.

---

### Dimension 1: Requirements Gathering (20%)

| Score | What It Looks Like |
|---|---|
| **1 — Poor** | Jumps into design without asking questions; ignores business context |
| **2 — Fair** | Asks a few questions but misses key ones (e.g., no volume/latency questions) |
| **3 — Good** | Asks 4–6 relevant questions covering business, volume, latency, retention |
| **4 — Strong** | Asks comprehensive questions, summarizes understanding, adapts questions based on answers |
| **5 — Outstanding** | Asks insightful questions that reveal hidden requirements; identifies trade-offs proactively |

**Key questions that score high:**
- "What decisions will this data drive?"
- "What's the SLA on data freshness?"
- "Are there compliance/regulatory requirements? (GDPR, HIPAA)"
- "Do we need to support multi-tenancy?"

---

### Dimension 2: Schema Design (30%)

| Score | What It Looks Like |
|---|---|
| **1 — Poor** | No clear schema; confused about grain; mixes fact and dimension concepts |
| **2 — Fair** | Basic schema with obvious grain issues; missing key dimensions |
| **3 — Good** | Clear star schema with correct grain; reasonable set of dimensions |
| **4 — Strong** | Well-structured star schema; correct SCD types; handles edge cases |
| **5 — Outstanding** | Elegant schema with justified trade-offs; handles late data, schema evolution, multi-grain analysis |

**What to get right:**
- [ ] **Grain** is clearly defined for every fact table
- [ ] **Primary keys** use surrogate keys, not natural keys
- [ ] **Dimensions** are properly normalized (but not over-normalized)
- [ ] **SCD type** is chosen and justified (Type 1, 2, or 7)
- [ ] **Fact table type** matches the use case (transaction, periodic snapshot, accumulating snapshot)

---

### Dimension 3: Depth of Knowledge (20%)

| Score | What It Looks Like |
|---|---|
| **1 — Poor** | Cannot explain basic concepts (star schema, SCD, grain) |
| **2 — Fair** | Knows basics but struggles with nuances |
| **3 — Good** | Explains concepts clearly; handles follow-up questions |
| **4 — Strong** | Deep knowledge; connects concepts (e.g., links SCD to business needs) |
| **5 — Outstanding** | Expert-level knowledge; discusses advanced topics naturally (bridge tables, degenerate dimensions, role-playing dimensions) |

**Concepts that separate levels:**

| Junior | Mid | Senior | Staff+ |
|---|---|---|---|
| Star schema, basic SCD | SCD Type 2, fact table types | SCD Type 7, bridge tables | Data vault, conformed dimensions |
| Normalization | Partitioning | Materialized views | Multi-model architectures |
| Primary/foreign keys | Clustering | Data quality frameworks | Data mesh, domain-driven design |

---

### Dimension 4: Communication & Structure (15%)

| Score | What It Looks Like |
|---|---|
| **1 — Poor** | Disorganized; cannot explain reasoning; rambling |
| **2 — Fair** | Some structure but hard to follow; talks too fast |
| **3 — Good** | Clear structure; explains trade-offs; invites feedback |
| **4 — Strong** | Crisp, structured, easy to follow; checks for understanding |
| **5 — Outstanding** | Consultative style; adapts to interviewer's level; uses diagrams effectively |

**Pro tips:**
- "Let me start with the high-level picture, then we can dive into each table."
- "Does this direction make sense so far?"
- "I'm making this assumption about X — does that match your understanding?"

---

### Dimension 5: Problem-Solving & Adaptability (15%)

| Score | What It Looks Like |
|---|---|
| **1 — Poor** | Stuck when challenged; cannot adapt the design |
| **2 — Fair** | Defends original design even when wrong |
| **3 — Good** | Incorporates feedback; adjusts design reasonably |
| **4 — Strong** | Gracefully handles curveballs; explains old vs new design |
| **5 — Outstanding** | Turns challenges into teaching moments; shows depth under pressure |

**Typical curveballs:**
- "Now we also need to track data at the sub-product level."
- "The business wants real-time dashboards."
- "We found a bug: some dimension attributes arrive late."
- "Legal requires 7-year data retention."

---

### Self-Evaluation Checklist

Before the interview, score yourself:

- [ ] I can ask 6+ relevant clarifying questions
- [ ] I can draw a star schema from memory
- [ ] I can explain SCD Types 1, 2, and 7 with examples
- [ ] I know the 3 fact table types and when to use each
- [ ] I can discuss partitioning and indexing trade-offs
- [ ] I can handle a curveball without panicking

---

### Interviewer Questions You'll Face

1. "How would you describe your data modeling process?"
2. "Tell me about a data model you designed. What trade-offs did you make?"
3. "How do you handle disagreement with a stakeholder on the model design?"

---

### Top-Up Questions (Senior-Level Probes)

- "How do you measure the quality of a data model?"
- "What's the difference between dimensional modeling and data vault? When would you use each?"
- "How do you balance query performance vs storage cost?"
- "Describe how you'd handle a situation where two business units have conflicting dimensional needs."

---

### Related Lessons

- ⬅️ [Lesson 2: How to Answer Data Modeling Questions](./02-how-to-answer-data-modeling-questions.md) — the answering framework
- ➡️ [Lesson 4: Data Modeling Fundamentals](./04-data-modeling-fundamentals.md) — the technical concepts the rubric evaluates
- ➡️ [Lesson 5: Introduction to Gathering Business Requirements](./05-introduction-to-gathering-business-requirements.md) — start the requirements-gathering module

