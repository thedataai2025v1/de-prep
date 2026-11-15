# Data Modeling Interview — Reference & Recommendations

Curated resources, key takeaways, and a study guide for the 27-module Data Modeling interview prep series.

---

## Module Quick Reference

| Module | Topic | Key Concepts |
|---|---|---|
| 01 | Introduction to Data Modeling | Star schema vs snowflake, dimensional vs normalized, when to use which |
| 02 | How to Answer Data Modeling Questions | Structured response approach, clarifying questions, showing trade-offs |
| 03 | Rubric for Data Modeling Questions | Scoring dimensions: correctness, depth, communication, real-world grounding |
| 04 | Data Modeling Fundamentals | Facts, dimensions, grains, surrogate keys, role-playing dimensions |
| 05 | Gathering Business Requirements | Elicitation techniques, stakeholder mapping, KPI identification |
| 06 | Recognizing the Core Business Problem | Problem framing, root cause analysis, translating business to data |
| 07 | Analyzing Metrics | KPIs, derived metrics, metric definitions, avoiding measurement traps |
| 08 | Analyzing Query Patterns | Scan patterns, filter selectivity, join frequency, access paths |
| 09 | Defining Latency Requirements | Real-time vs batch, SLA negotiation, consistency vs availability |
| 10 | Data Volume & Scalability | Partitioning strategies, data lifecycle, growth planning |
| 11 | Data Retention & Historical Data | Retention policies, archiving, legal/compliance constraints |
| 12 | Data Quality & Validation | Profiling, anomaly detection, quality rules, monitoring |
| 13 | High-Level Model Diagrams | Entity-relationship diagrams, logical vs physical models |
| 14 | Evolving Models | Schema migration, backward compatibility, iterative design |
| 15 | Practice: E-Commerce Platform | Orders, products, customers, returns, promotions |
| 16 | Practice: Social Media Analytics | Posts, engagement, users, followers, content types |
| 17 | Practice: Video Streaming | Views, sessions, content, subscriptions, bandwidth |
| 18 | Practice: Rideshare Platform | Rides, drivers, passengers, payments, geospatial |
| 19 | Practice: Financial Transactions | Accounts, transactions, ledgers, compliance, auditing |
| 20 | Practice: IoT Sensor Data | Time-series, telemetry, device management, aggregations |
| 21 | Slowly Changing Dimensions (Types 0–7) | Type 1–7, hybrid SCDs, additive tracking, decision framework |
| 22 | Degenerate & Junk Dimensions | Transaction IDs as dimensions, flag aggregation, junk dim design |
| 23 | Dimension Design Practice | Natural vs surrogate keys, conformed dimensions, role-playing dims |
| 24 | Fact Table Types | Transaction, periodic snapshot, accumulating snapshot, factless facts |
| 25 | Additive, Semi-Additive & Non-Additive Measures | Aggregation rules, level-based additivity, semi-additive examples |
| 26 | Multi-Fact Query Strategies | Conformed dimensions, bridge tables, fact consolidation |
| 27 | Fact Modeling Practice | Grain selection, fact table design, handling edge cases |

---

## Key Interview Concepts (Senior-Level)

### Must-Know Patterns
- **Conformed Dimensions** — shared dimensions across multiple fact tables to enable cross-fact analysis
- **Bridge Tables** — handle many-to-many relationships in star schemas
- **Role-Playing Dimensions** — one physical dimension used in multiple roles (e.g., `dim_date` as `order_date`, `ship_date`)
- **Factless Fact Tables** — track events or coverage without measures (e.g., class attendance, promotions active)
- **Surrogate Key Generation** — why and how: database sequences, hash-based keys, identity columns
- **Type 6 & 7 SCDs** — hybrid approaches combining multiple SCD strategies for complex requirements
- **Degenerate Dimensions** — fact-level attributes with no separate dimension table (e.g., transaction ID)
- **Junk Dimensions** — grouping unrelated flags/indicators into a single small dimension

### Common Interview Traps
- Confusing grain levels between fact tables in multi-fact queries
- Merging unrelated facts into a single table (leads to NULL proliferation)
- Using natural keys as surrogate keys (breaks on source system changes)
- Ignoring latency requirements when choosing between batch and real-time models
- Designing dimensions without considering conformance across fact tables

---

## Recommended Books

| Book | Why Read It |
|---|---|
| *The Data Warehouse Toolkit* by Ralph Kimball | The definitive guide to dimensional modeling, SCDs, and fact table design |
| *Star Schema: The Complete Reference* by Ralph Kimball | Deep dive into star schema design and implementation |
| *Designing Data-Intensive Applications* by Martin Kleppmann | Architecture, consistency, and scalability for modern data systems |
| *Fundamentals of Data Engineering* by Joe Reis & Matt Housley | End-to-end data engineering lifecycle, including modeling |
| *Data Mesh* by Zhamak Dehghani | Decentralized data ownership, domain-driven data products |
| *Lean Analytics* by Alistair Croll & Ben Yoskovitz | Connecting metrics to business outcomes |

---

## Recommended Resources

| Resource | Type | What It Covers |
|---|---|---|
| [Kimball Group Website](https://www.kimballgroup.com) | Articles & white papers | Dimensional modeling, SCDs, data warehouse patterns |
| [DBT Docs — Data Modeling](https://docs.getdbt.com/docs/build/materializations) | Documentation | Practical data modeling in modern data stacks |
| [Apache Pinot](https://pinot.apache.org/) | Documentation | Real-time analytics data models |
| [Materialize](https://materialize.com/) | Documentation | Streaming materialized views, real-time aggregation |
| [Snowflake Data Modeling Guide](https://community.snowflake.com/s/article/Best-Practices-for-Data-Modeling-for-Snowflake) | Guide | Cloud-native data modeling best practices |
| [Mode Analytics Data Warehouse Guide](https://mode.com/sql-tutorial/) | Tutorial | SQL-based data modeling fundamentals |

---

## Interview Preparation Checklist

### Before the Interview
- [ ] Review the 27 modules at least once
- [ ] Practice 3+ full end-to-end modeling exercises (modules 15–20, 23, 27)
- [ ] Be able to explain the trade-offs between each SCD type (Type 1–7)
- [ ] Know the difference between the 4 fact table types and when to use each
- [ ] Practice explaining your design decisions out loud (not just writing SQL)

### During the Interview
- [ ] Start with clarifying questions — never jump straight to tables
- [ ] Explicitly state your assumptions
- [ ] Draw a high-level diagram before writing detailed DDL
- [ ] Call out trade-offs: "I chose X over Y because…"
- [ ] Mention data quality, latency, and scalability even if not asked
- [ ] End with how you'd evolve the model over time

### Red Flags (Avoid These)
- ❌ Drawing tables before understanding requirements
- ❌ Merging unrelated facts into one table
- ❌ Using natural keys as primary keys in dimensional models
- ❌ Ignoring the grain when joining fact tables
- ❌ Designing without considering conformance dimensions
- ❌ Not mentioning how the model handles historical data

---

## Common Interview Question Patterns

1. **"Design a data model for [platform]"** — follow the 6-step process (modules 02, 05–10)
2. **"How would you handle [SCD scenario]"** — pick the right SCD type with justification (module 21)
3. **"What type of fact table for [scenario]"** — match the scenario to transaction, snapshot, accumulating or factless (module 24)
4. **"How do you join these fact tables?"** — use conformed dimensions, bridge tables (module 26)
5. **"How does your model handle [historical tracking / compliance]"** — SCDs + retention policies (modules 11, 21)

---

## Further Reading by Topic

| If You Want to Go Deeper On… | Read… |
|---|---|
| Slowly Changing Dimensions | Kimball Group articles on SCD types 5–7 |
| Real-time Data Modeling | Apache Pinot docs, Materialize docs |
| Data Quality at Scale | Great Expectations docs, dbt tests |
| Dimensional Modeling Edge Cases | Kimball Group Q&A forum |
| Cloud Data Modeling | Snowflake, BigQuery, Redshift official guides |
| Data Product Thinking | *Data Mesh* by Zhamak Dehghani |

## Part 7 — Performance Optimization

- [28 — Date-Based Partitioning](./28-date-based-partitioning.md)
- [29 — Indexing Strategies](./29-indexing-strategies.md)
- [30 — Aggregates & Materialized Views](./30-aggregates-and-materialized-views.md)

## Part 8 — Advanced Patterns & Practice

- [31 — Advanced Dimension Design](./31-advanced-dimension-design.md)
- [32 — Bridge Tables & Factless Facts](./32-bridge-tables-and-factless-facts.md)
- [33 — Practice: Cloud Services Platform](./33-practice-cloud-services-platform.md)
- [34 — Practice: Online Advertising Platform](./34-practice-online-advertising-platform.md)

## Part 9 — Mock Interviews & Principal-Level Prep

- [35 — Mock Interview: Ride-Sharing Platform](./35-mock-interview-ride-sharing.md)
- [36 — Mock Interview: Customer Support Platform](./36-mock-interview-customer-support.md)
- [37 — Mock Interview: Airbnb-Style Platform](./37-mock-interview-airbnb-style.md)
- [38 — Mock Interview: Stripe Billing](./38-mock-interview-stripe-billing.md)
- [39 — Mock Interview: Instagram Analytics](./39-mock-interview-instagram-analytics.md)
- [40 — Mock Interview: Amazon Orders](./40-mock-interview-amazon-orders.md)
- [41 — Principal-Level Cheat Sheet](./41-principal-level-cheat-sheet.md)
- [42 — Architect Audit Criteria](./42-architect-audit-criteria.md)

---

*Last updated: 2025-01 — aligned with the 42-module curriculum.*
