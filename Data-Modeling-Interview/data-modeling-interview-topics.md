# Data Modeling Interview — 27-Module Curriculum

Structured preparation for data modeling interview questions, organized in learning order from fundamentals through advanced practice scenarios.

---

## Part 1 — How to Approach Data Modeling Questions

| # | Module | What You'll Learn |
|---|---|---|
| 01 | [Introduction to Data Modeling](./01-introduction-to-data-modeling-questions.md) | What data modeling interviews test, the scope and expectations |
| 02 | [How to Answer Data Modeling Questions](./02-how-to-answer-data-modeling-questions.md) | Step-by-step response framework: clarify → diagram → design → refine |
| 03 | [Interview Scoring Rubric](./03-rubric-for-data-modeling-questions.md) | How interviewers evaluate: correctness, depth, communication, trade-offs |
| 04 | [Data Modeling Fundamentals](./04-data-modeling-fundamentals.md) | Star schema, snowflake schema, grain, facts, dimensions, surrogate keys |

## Part 2 — Requirements Analysis

| # | Module | What You'll Learn |
|---|---|---|
| 05 | [Gathering Business Requirements](./05-introduction-to-gathering-business-requirements.md) | Elicitation techniques, stakeholder mapping, translating business needs to data |
| 06 | [Recognizing the Core Business Problem](./06-recognizing-the-core-business-problem.md) | Problem framing, identifying what the business actually needs to measure |
| 07 | [Analyzing Metrics](./07-analyzing-metrics.md) | KPI definitions, derived metrics, metric dependencies, avoiding measurement traps |
| 08 | [Analyzing Query Patterns](./08-analyzing-query-patterns.md) | Scan patterns, filter selectivity, join frequency, how queries shape the model |
| 09 | [Defining Latency Requirements](./09-defining-latency-requirements.md) | Real-time vs batch, SLA negotiation, consistency vs availability trade-offs |
| 10 | [Data Volume & Scalability](./10-data-volume-and-scalability-considerations.md) | Partitioning strategies, data lifecycle management, growth planning |
| 11 | [Data Retention & Historical Data](./11-data-retention-policies-and-historical-data-management.md) | Retention policies, archiving strategies, legal and compliance constraints |
| 12 | [Data Quality & Validation](./12-data-quality-rules-and-validation-strategies.md) | Profiling, anomaly detection, quality rules, monitoring and alerting |

## Part 3 — Design & Iteration

| # | Module | What You'll Learn |
|---|---|---|
| 13 | [High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md) | Entity-relationship diagrams, logical vs physical models, diagramming tools |
| 14 | [Evolving Models Over Time](./14-evolving-models-based-on-changing-requirements.md) | Schema migration, backward compatibility, iterative design, versioning |

## Part 4 — Practice Scenarios (End-to-End Modeling)

| # | Module | What You'll Learn |
|---|---|---|
| 15 | [Practice: E-Commerce Platform](./15-practice-ecommerce-platform.md) | Orders, products, customers, returns, promotions, multi-table design |
| 16 | [Practice: Social Media Analytics](./16-practice-social-media-analytics.md) | Posts, engagement, users, followers, content-type hierarchies |
| 17 | [Practice: Video Streaming Service](./17-practice-video-streaming-service.md) | Views, sessions, content catalog, subscriptions, bandwidth tracking |
| 18 | [Practice: Rideshare Platform](./18-practice-ridesharing-platform.md) | Rides, drivers, passengers, payments, geospatial dimensions |
| 19 | [Practice: Financial Transactions](./19-practice-financial-transactions.md) | Accounts, ledgers, transaction auditing, compliance requirements |
| 20 | [Practice: IoT Sensor Data](./20-practice-iot-sensor-data.md) | Time-series modeling, telemetry, device management, roll-ups |

## Part 5 — Dimensional Modeling (Deep Dive)

| # | Module | What You'll Learn |
|---|---|---|
| 21 | [Slowly Changing Dimensions (Types 0–7)](./21-slowly-changing-dimensions-types-1-4.md) | SCD Type 0–4, Type 5 (additive), Type 6 (hybrid), Type 7 (double), decision framework |
| 22 | [Degenerate & Junk Dimensions](./22-degenerate-and-junk-dimensions.md) | Transaction IDs as dimensions, flag aggregation, when junk dims help vs hurt |
| 23 | [Dimension Design Practice](./23-dimension-design-practice.md) | Natural vs surrogate keys, conformed dimensions, role-playing dimensions |

## Part 6 — Fact Table Design (Deep Dive)

| # | Module | What You'll Learn |
|---|---|---|
| 24 | [Fact Table Types](./24-fact-table-types.md) | Transaction facts, periodic snapshots, accumulating snapshots, factless facts |
| 25 | [Additive, Semi-Additive & Non-Additive Measures](./25-additive-semi-additive-non-additive-measures.md) | Aggregation rules, level-based additivity, semi-additive measures (balances, rates) |
| 26 | [Multi-Fact Query Strategies](./26-multi-fact-query-strategies.md) | Conformed dimensions, bridge tables, combining multiple fact tables in queries |
| 27 | [Fact Modeling Practice](./27-fact-modeling-practice.md) | Grain selection, fact table design, handling edge cases, final integrated exercise |

---

## Suggested Study Order

Follow the numbered sequence 01 → 27. Each part builds on the previous:

1. **Part 1** (01–04): Learn the framework and fundamentals
2. **Part 2** (05–12): Learn requirements analysis
3. **Part 3** (13–14): Learn design and iteration
4. **Part 4** (15–20): Practice with real scenarios
5. **Part 5** (21–23): Deep-dive into dimensional modeling
6. **Part 6** (24–27): Deep-dive into fact table design

**Optional reinforcement:** After finishing all 27 modules, revisit the practice scenarios (15–20, 23, 27) with fresh knowledge of SCDs and fact table types.

---

## Reference Materials

- [📖 Reference & Recommendations](./reference.md) — curated books, resources, and interview prep checklist