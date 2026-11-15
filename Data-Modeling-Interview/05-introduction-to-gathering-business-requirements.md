# Module 2: Gathering Business Requirements
## Lesson 2.1 — Introduction to Gathering Business Requirements

---

### Why Requirements Gathering Is the Most Critical Step

In data modeling interviews, your technical skill with normalization, star schemas, or partitioning is irrelevant if you build the **wrong** model. Requirements gathering is where you demonstrate that you can *think like an engineer* before you *act like a modeler*.

> **Interviewers are not testing whether you can draw an ERD. They are testing whether you can discover what the ERD should model.**

When you walk into a data modeling interview, the first 10–15 minutes are almost always a requirements-gathering conversation. The interviewer will describe a vague business scenario — "We run an e-commerce platform" or "We need a system to track customer behavior" — and your job is to turn that vagueness into a precise specification before touching a whiteboard.

**Why this matters:**
- A model built on incomplete requirements will fail in production, no matter how elegant.
- Interviewers use your questions to gauge your depth of experience: juniors ask about tables; seniors ask about SLAs, data consumers, and failure modes.
- The best models emerge from understanding the *business problem*, not the *technical puzzle*.

---

### The Consequences of Skipping or Rushing This Phase

| Mistake | Consequence | Real-World Example |
|---------|-------------|-------------------|
| Assuming the data sources | Model can't be populated; pipeline fails | Building a star schema that expects `customer_id` when the source system only has email |
| Ignoring data volume | Poor partitioning, slow queries | A fact table grows to 50 billion rows without partition strategy — daily refresh takes 6 hours |
| Overlooking retention policy | Storage costs explode; compliance violations | Storing PII indefinitely in a marketing analytics DB → GDPR breach |
| Not identifying all consumers | Model optimizes for one use case, breaks others | Building a transactional model that can't serve real-time dashboard needs |
| Skipping non-functional requirements | System meets "what" but fails "how well" | A model works for 100 users but crashes at 10,000 concurrent queries |

> **Rule of thumb:** 40% of data model failures in production trace back to unclear or missing requirements, not technical flaws.

---

### Functional vs. Non-Functional Requirements

Understanding this distinction is fundamental. Interviewers expect you to articulate it clearly.

#### Functional Requirements — *What the system does*

These describe the **capabilities** of the system: what data it stores, what operations it supports, what outputs it produces.

| Category | Example |
|----------|---------|
| **Data entities** | "We need to store products, customers, orders, and shipments" |
| **Business operations** | "The model must support calculating monthly revenue by region" |
| **Data relationships** | "Each order belongs to one customer but may contain multiple products" |
| **Reporting needs** | "We need a daily summary of new sign-ups, active users, and churn" |
| **Integration needs** | "The model must feed a BI tool and a machine-learning pipeline" |

#### Non-Functional Requirements — *How well the system does it*

These describe the **constraints** and **quality attributes** of the system.

| Category | Example |
|----------|---------|
| **Performance / Latency** | "Queries must return in under 2 seconds for the live dashboard" |
| **Volume / Scale** | "We expect 10 million events per day, growing 20% month-over-month" |
| **Availability** | "The analytics database must be available 99.9% of the time" |
| **Retention** | "Raw events kept for 90 days; aggregated data kept for 5 years" |
| **Compliance / Security** | "PII must be encrypted at rest; GDPR right-to-delete must be supported" |
| **Cost** | "Total storage and compute budget capped at $50K/month" |

> **Interview tip:** Always ask about non-functional requirements. Juniors stop at "what data do you have?" Seniors ask "how much, how fast, how long, and under what constraints?"

---

### A Structured Approach to Gathering Requirements in an Interview

Use this **five-phase framework** during any data modeling interview. It keeps you organized and ensures you don't miss critical dimensions.

```
Phase 1: Context       →  "What business are we solving for?"
Phase 2: Consumers     →  "Who uses this data and how?"
Phase 3: Sources       →  "Where does the data come from?"
Phase 4: Metrics       →  "What do they need to calculate?"
Phase 5: Constraints   →  "What limits do we work within?"
```

#### Phase 1 — Business Context
Start broad. Understand the domain, the industry, and the core business problem.

> *"Before we talk about tables, can you help me understand: what business decision is this data model enabling?"*

#### Phase 2 — Data Consumers
Identify every person, system, or process that will read from this model.

> *"Who are the primary consumers of this data? Executives? Data scientists? A real-time API? Each will need a different optimization strategy."*

#### Phase 3 — Data Sources
Map the upstream systems. Know the source, the format, the refresh frequency, and the reliability.

> *"What systems produce this data? Are they transactional databases, event streams, third-party APIs? What's the data quality like?"*

#### Phase 4 — Business Metrics
Pinpoint the KPIs, reports, and analyses that must be derivable from the model.

> *"What are the top 5 metrics or reports that this model must support? What grain of detail do each of them need?"*

#### Phase 5 — Constraints
Close with non-functional requirements: volume, latency, retention, compliance, cost.

> *"Are there any constraints I should be aware of — query latency targets, data retention policies, compliance requirements, or budget limits?"*

---

### Common Categories of Requirements

| Category | Key Questions to Ask | Why It Matters for the Model |
|----------|---------------------|------------------------------|
| **Business Metrics** | What KPIs must the model support? What aggregations are common? | Determines grain of fact tables, choice of dimensions, need for aggregates or materialized views |
| **Data Consumers** | Who queries this data? What tools do they use? How many concurrent users? | Determines optimization strategy: row-based for ad-hoc SQL vs. columnar for BI vs. pre-aggregated for dashboards |
| **Data Sources** | What systems feed this model? Structured, semi-structured, or unstructured? Batch or streaming? | Determines ETL/ELT strategy, staging layer design, and schema flexibility |
| **Latency** | How fresh must the data be? Real-time, near-real-time (minutes), or daily batch acceptable? | Determines streaming vs. batch architecture, choice of database engine |
| **Volume** | How many rows per day? Growth rate? Peak load? | Determines partitioning strategy, indexing, clustering, and hardware sizing |
| **Retention** | How long must raw and aggregated data be kept? Is archival needed? | Determines tiered storage strategy, partition pruning, and cleanup automation |
| **Compliance** | Any PII, HIPAA, GDPR, SOX, or PCI requirements? Data residency constraints? | Determines encryption, access controls, data masking, and geographic storage requirements |

---

### Interviewer Questions You'll Face

1. *"Walk me through how you would gather requirements for a data model. What's your process?"*
2. *"A stakeholder says 'we need a dashboard showing sales by region.' What questions do you ask before you start modeling?"*
3. *"How do you handle a situation where two stakeholders have conflicting requirements?"*
4. *"What would you do if the business requirements keep changing mid-project?"*

---

### Questions YOU Should Ask the Interviewer

| # | Question | What It Reveals |
|---|----------|-----------------|
| 1 | *"What business decisions will this data model enable?"* | Strategic thinking; business alignment |
| 2 | *"Who are the primary and secondary consumers of this data?"* | Awareness of multi-audience needs |
| 3 | *"What's the expected data volume and growth trajectory?"* | Scalability mindset |
| 4 | *"Are there any compliance or security constraints I should design for?"* | Security-aware engineering |
| 5 | *"What's the acceptable latency for data freshness?"* | Real-time vs. batch architecture decisions |
| 6 | *"What existing data sources or systems will this model pull from?"* | Integration awareness |
| 7 | *"Are there existing reports or metrics that must be reproducible?"* | Legacy compatibility and migration planning |

---

### Top-Up Questions (Senior-Level Probes)

These demonstrate depth and experience. Use 2–3 of these to elevate your interview performance.

1. *"What's the current data quality situation — are there known issues with completeness, consistency, or timeliness that the model needs to account for?"*
2. *"How does the organization currently handle slow-changing dimensions? Is there a need for SCD Type 2 tracking, or is current-state sufficient?"*
3. *"What's the incident response process when a pipeline fails or data quality degrades? Should the model include data quality flags or SLA monitoring?"*
4. *"Is there a data governance framework in place — who owns data definitions, and how are changes to business requirements communicated?"*
5. *"What's the cost of failure here — is this model supporting a mission-critical revenue system, or is it an exploratory analytics project?"*

---

### Practice Exercise

**Scenario:** You're interviewing for a Senior Data Engineer role at *StreamMart*, a subscription-based video streaming platform. The interviewer says:

> *"We need a data model to track subscriber behavior. We want to understand when people watch content, when they pause or skip, and when they cancel their subscription. Our marketing team wants to use this to personalize recommendations and reduce churn."*

**Your task:** Write down all the questions you would ask before designing the model. Group them using the five-phase framework.

<details>
<summary>💡 Click to reveal a model answer</summary>

#### Phase 1 — Business Context
- What specific business outcomes are we targeting: reducing churn, increasing watch time, improving retention?
- Is this for all users or a specific segment (e.g., trial users, lapsed subscribers)?

#### Phase 2 — Data Consumers
- Besides marketing, will product, engineering, or finance teams use this model?
- How will the marketing team consume the data — SQL, BI tool, API, ML pipeline?
- How many concurrent users will query this model?

#### Phase 3 — Data Sources
- Where are these events currently being logged? Is there an existing event tracking system?
- What's the event schema — do we have `user_id`, `content_id`, `event_type`, `timestamp`, `device`?
- Are there any third-party data sources (e.g., payment systems for cancellation data)?

#### Phase 4 — Business Metrics
- What specific metrics are needed: daily active users, session length, skip rate, churn rate?
- Do we need cohort analysis — e.g., behavior of users who signed up in January vs. February?
- What's the grain of analysis: individual user sessions or aggregated daily/monthly stats?

#### Phase 5 — Constraints
- How many subscribers? How many events per user per day?
- Is real-time behavior tracking needed, or is daily batch acceptable?
- How long should raw event data be retained? Any GDPR considerations for EU subscribers?
- Is there a budget constraint for storage and compute?

</details>

---

### Key Takeaways

| Do | Don't |
|----|-------|
| Ask about consumers before sources | Start drawing tables immediately |
| Quantify everything (volume, latency, retention) | Assume requirements are complete |
| Ask about compliance early | Treat non-functional requirements as optional |
| Confirm requirements before designing | Build the first model you think of |

> **Remember:** A great data modeler is first a great interviewer. Your questions are your first and most powerful design tool.

---

### Related Lessons

- ⬅️ [Lesson 4: Data Modeling Fundamentals](./04-data-modeling-fundamentals.md) — core concepts before gathering requirements
- ➡️ [Lesson 6: Recognizing the Core Business Problem](./06-recognizing-the-core-business-problem.md) — peel back the layers
- ➡️ [Lesson 7: Analyzing Metrics](./07-analyzing-metrics.md) — translating business KPIs into measures
