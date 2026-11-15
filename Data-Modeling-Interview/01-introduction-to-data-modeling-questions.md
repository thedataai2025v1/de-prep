# Module 1: Overview

## Lesson 1.1 — Introduction to Data Modeling Questions

### What Is a Data Modeling Interview?

A data modeling interview tests your ability to design structured data systems that serve real business needs. Interviewers typically present a scenario (e.g., "Design a data warehouse for a ride-sharing platform") and evaluate how you:

- **Gather and clarify requirements**
- **Translate business needs into a structured data model**
- **Choose appropriate schema types** (star, snowflake, normalized)
- **Handle edge cases** (late-arriving data, historical tracking, scalability)
- **Optimize for query performance**

### Why Companies Ask These Questions

| What They're Looking For | Why It Matters |
|---|---|
| Structured thinking | Can you break down ambiguity into a clean plan? |
| Business acumen | Do you ask the right questions before designing? |
| Depth of knowledge | Do you know SCDs, fact types, partitioning strategies? |
| Communication | Can you explain trade-offs clearly? |
| Real-world experience | Have you dealt with real-scale data problems? |

### What to Expect in the Interview

1. **Problem statement** — The interviewer describes a business domain.
2. **Q&A phase** — You ask clarifying questions about metrics, volume, latency, users.
3. **Design phase** — You sketch entities, relationships, and schema on a whiteboard or diagramming tool.
4. **Deep-dive** — The interviewer picks specific tables and asks about SCD strategy, partitioning, indexing, edge cases.
5. **Evolution** — "Now the business wants to add X — how would you change the model?"

### Key Principles to Remember

- **Requirements first, schema second.** Never jump into drawing tables without understanding the business.
- **Dimensional modeling** is the industry standard for data warehouses and analytics.
- **There is no single correct answer.** Interviewers evaluate your reasoning, not a specific schema.
- **Think about data consumers.** Analysts, ML engineers, and dashboards all have different query patterns.

---

### Interviewer Questions You'll Face

1. "Walk me through how you'd design a data warehouse for [X]."
2. "What's the first thing you'd do?"
3. "How would you track historical changes in [dimension]?"
4. "What fact tables would you create and why?"
5. "How would you handle [edge case]?"
6. "What's the expected data volume and how does it affect your design?"

---

### Questions YOU Should Ask the Interviewer (Conversation Starters)

| Category | Questions to Ask |
|---|---|
| **Business** | "What are the key metrics the business cares about?" "Who are the primary data consumers?" |
| **Volume** | "How much data do we process per day/month?" "Is the data growing rapidly?" |
| **Latency** | "What's the acceptable freshness? Real-time, near-real-time, or batch is fine?" |
| **Retention** | "How long do we need to keep historical data?" |
| **Query Patterns** | "What are the most common queries? Time range, granularity?" |
| **Constraints** | "Any technology constraints? (e.g., BigQuery, Snowflake, Redshift)" |

---

### Top-Up Questions (Senior-Level Probes)

- "How would your design change if daily volume increased 10x?"
- "How do you handle schema evolution when the upstream source changes?"
- "What's your strategy for data quality checks in the pipeline?"
- "How do you handle late-arriving data in an accumulating snapshot fact table?"
- "Would you denormalize further? Where do you draw the line?"

---

### Modern Data Stack Context

Today's data modeling rarely happens in a vacuum. Most companies use a **modern data stack** that shapes how you design:

| Layer | Tools | Impact on Data Modeling |
|---|---|---|
| **Storage** | Snowflake, BigQuery, Redshift, Databricks | Cloud DWs handle scale; focus on logical design, not physical tuning |
| **Transformation** | dbt, Spark, Airflow | dbt promotes modular, testable models with materialization choices (table, view, incremental) |
| **Orchestration** | Airflow, Dagster, dbt Core | Pipeline scheduling affects refresh frequency and model dependencies |
| **Serving** | Metabase, Looker, Tableau, Superset | BI tools expect star schemas; understand your consumers' query patterns |
| **Quality** | dbt tests, Great Expectations, Soda Core | Data quality checks are part of the model, not afterthought |

> **Key insight:** In the modern stack, data modeling shifts from heavy physical design (partitioning, clustering, indexing) to **logical modeling** (grain, relationships, business logic) — the warehouse handles the physical layer. dbt turns data models into version-controlled, testable SQL files.

---

### Practice Exercise

> **Prompt:** You are asked to design a data warehouse for an e-commerce platform. Write down:
> 1. The top 5 clarifying questions you'd ask the interviewer.
> 2. A rough list of dimensions and fact tables you'd expect.
> 3. One potential scalability concern and how you'd address it.

---

### Related Lessons

- ➡️ [Lesson 2: How to Answer Data Modeling Questions](./02-how-to-answer-data-modeling-questions.md) — the step-by-step answering framework
- ➡️ [Lesson 3: Rubric for Data Modeling Questions](./03-rubric-for-data-modeling-questions.md) — what interviewers score you on
- ➡️ [Lesson 4: Data Modeling Fundamentals](./04-data-modeling-fundamentals.md) — core concepts (grain, dimensions, facts, normalization)

