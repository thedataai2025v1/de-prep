# Module 1: Overview

## Lesson 1.2 — How to Answer Data Modeling Questions

### The 5-Step Framework

Use this consistent structure in every data modeling interview. It keeps you organized and signals seniority.

#### Step 1: Clarify Requirements (5–8 minutes)

Ask targeted questions in this order:

1. **Business goals** — "What decisions will this data support?"
2. **Key metrics** — "What are the top KPIs the stakeholders track daily?"
3. **Data consumers** — "Who will query this? BI tools, ML models, ad-hoc analysts?"
4. **Volume & growth** — "What's current data volume? Growth rate?"
5. **Latency requirements** — "Real-time, near-real-time, or is batch acceptable?"
6. **Retention** — "How far back do we need to query?"
7. **Technology constraints** — "Any preferred tools or platforms?"

> **Tip:** Don't just ask questions — summarize your understanding. "So to confirm, you need daily transaction-level analytics with 2-year retention, primarily consumed by BI dashboards."

#### Step 2: Sketch High-Level Design (5–10 minutes)

- Draw the main **fact tables** and **dimension tables**.
- Show **relationships** (primary keys → foreign keys).
- Identify **grain** of each fact table.
- Label approximate **data volumes** per table.

```
Example Star Schema:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   fact_sales │─────│ dim_product  │
│              │─────│ dim_customer │
│              │─────│ dim_date     │
│              │─────│ dim_location │
└──────────────┘     └──────────────┘
```

#### Step 3: Detail the Schema (10–15 minutes)

For each table, define:

- **Columns** (name, type, description)
- **Grain** (what one row represents)
- **Primary key / surrogate key**
- **Slowly Changing Dimension strategy** (if applicable)
- **Partitioning and clustering strategy**

#### Step 4: Address Edge Cases (5 minutes)

- Late-arriving data
- Missing dimension members
- Schema evolution in source systems
- Data quality issues (duplicates, nulls, invalid values)

#### Step 5: Optimize & Discuss Trade-offs (5 minutes)

- **Partitioning:** Which column? (Usually date/time)
- **Clustering/Sorting:** What filters are most common?
- **Aggregates:** Are there pre-aggregated tables for heavy queries?
- **Trade-offs:** "I chose a star schema over snowflake because query simplicity outweighs storage."

---

### Common Mistakes & How to Avoid Them

| Mistake | Fix |
|---|---|
| Jumping straight into schema design | Always start with clarifying questions |
| Not defining grain | Explicitly state the grain of every fact table |
| Ignoring historical tracking | Mention SCD strategy for key dimensions |
| Over-normalizing | Data warehouses are denormalized; use star schema |
| Forgetting about data volume | Estimate row counts and storage implications |
| Not discussing edge cases | Always address late data, nulls, duplicates |

---

### Sample Interview Dialogue

**Interviewer:** "Design a data warehouse for a video streaming service."

**You:**
> "Great. Before I design anything, I'd like to understand the business better:
>
> 1. What are the primary metrics? — Watch time, completion rate, churn, recommendations CTR?
> 2. Who are the consumers? — Product team dashboards, content licensing analysts, recommendation ML models?
> 3. What's the expected daily volume? — Number of view events, user count?
> 4. What latency is needed? — Can we do nightly batch or do we need near-real-time dashboards?
> 5. How long do we need to retain data? — 1 year, 3 years?
> 6. Any technology preferences — BigQuery, Snowflake, Redshift?
>
> Once I have that, I'll sketch the high-level model."

---

### Interviewer Questions You'll Face

1. "Walk me through your approach before you start drawing."
2. "Why did you choose this grain for the fact table?"
3. "How would you handle a dimension attribute that changes frequently?"
4. "What would you do if the source system adds a new attribute?"

---

### Top-Up Questions (Senior-Level Probes)

- "How would you version your schema as the business evolves?"
- "Describe a time you had to refactor a data model. What went wrong and how did you fix it?"
- "How do you decide between a periodic snapshot and an accumulating snapshot?"
- "What's your approach to monitoring data quality in production?"

---

### Practice Exercise

> **Prompt:** You're interviewing for a Senior Data Engineer role at a fintech company. They ask: "Design a data model for transaction fraud analysis."
>
> Write out:
> 1. Your 6 clarifying questions.
> 2. A summary statement restating the requirements.
> 3. A rough sketch of 2–3 fact tables and their dimensions.

---

### Related Lessons

- ⬅️ [Lesson 1: Introduction to Data Modeling Questions](./01-introduction-to-data-modeling-questions.md) — context and overview
- ➡️ [Lesson 3: Rubric for Data Modeling Questions](./03-rubric-for-data-modeling-questions.md) — scoring criteria interviewers use
- ➡️ [Lesson 4: Data Modeling Fundamentals](./04-data-modeling-fundamentals.md) — the core technical concepts you'll apply

