# Data Modeling Interview Masterclass — References & Recommendations

This file consolidates the external reading links cited in the course lessons and provides a practical guide for using them during interview preparation.

> **How to use this index:** Treat the course lessons as the primary curriculum. Use the references to validate platform-specific behavior, deepen a topic, and prepare evidence-based trade-off discussions. Product documentation changes; verify current syntax and feature availability for the engine/version in the interview scenario.

## 1. Recommended study approach

1. **Learn the concept before memorizing a schema.** For every case, identify business processes, actors, lifecycle states, metrics, and query patterns.
2. **State the grain explicitly.** Write one sentence for every fact table: “One row represents …”. Then identify its natural key and expected cardinality.
3. **Separate business events from current state.** Preserve event history where it matters; derive current snapshots deterministically.
4. **Define metrics before writing SQL.** Specify numerator, denominator, time basis, exclusions, currency, and late-arrival/correction policy.
5. **Draw conceptual relationships before dimensional models.** Distinguish entities, many-to-many relationships, event facts, snapshots, and bridges.
6. **Explain operational versus analytical models.** Normalize transactional write models where appropriate; design analytical facts/dimensions for reporting and aggregation.
7. **Defend trade-offs.** Discuss freshness, correctness, cost, concurrency, retention, privacy, query patterns, and operational complexity.
8. **Practise aloud under time limits.** Use 75–90 minutes per case: discovery, model, pipelines, SQL, scale/governance, and follow-up questions.
9. **Use a metric fanout check.** Before joining facts, compare grains and aggregate child facts to compatible grains to prevent duplicated measures.
10. **Close with principal-level ownership.** Cover data contracts, domain ownership, migration, observability, SLOs, adoption, and decision records.

## 2. Suggested mock-interview sequence

| Order | Case | What to emphasize |
|---:|---|---|
| 1 | Ride-sharing | Request vs offer vs trip; event lifecycle; telemetry and marketplace metrics |
| 2 | Customer support | Ticket/message/assignment grains; SLA calendars; reopen and merge history |
| 3 | Airbnb-like marketplace | Listing-night availability; reservation changes; stay-night and payment separation |
| 4 | Stripe-like payments | Payment intent vs attempt vs balance movement; immutable postings; reconciliation |
| 5 | Instagram-like social platform | Impression vs reach; social graph; event scale, skew, deletion and privacy |
| 6 | Amazon-like marketplace | Product vs seller offer; order vs line; split shipments; inventory and returns |

## 3. Reference links cited in the course

- [Amazon Redshift table design best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html) — cited in `07_Mock_Interviews/01_Ride_Sharing_Platform.md`.
- [Fulfillment Platform re-architecture](https://www.uber.com/us/en/blog/fulfillment-platform-rearchitecture/) — cited in `07_Mock_Interviews/01_Ride_Sharing_Platform.md`.
- [Redshift best practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html) — cited in `07_Mock_Interviews/01_Ride_Sharing_Platform.md`, `07_Mock_Interviews/02_Customer_Support_Platform.md`.
- [Dimensional modeling in Amazon Redshift](https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/) — cited in `07_Mock_Interviews/02_Customer_Support_Platform.md`, `07_Mock_Interviews/03_Airbnb_Platform.md`, `07_Mock_Interviews/06_Amazon_Marketplace.md`.
- [Scaling beyond one: evolving data architecture for a multi-product world](https://airbnb.tech/uncategorized/scaling-beyond-one-how-airbnb-evolved-its-data-architecture-for-a-multi-product-world/) — cited in `07_Mock_Interviews/03_Airbnb_Platform.md`.
- [Capture a PaymentIntent](https://docs.stripe.com/api/payment_intents/capture) — cited in `07_Mock_Interviews/04_Stripe_Payments_Platform.md`.
- [Dispute object](https://docs.stripe.com/api/disputes/object) — cited in `07_Mock_Interviews/04_Stripe_Payments_Platform.md`.
- [Events API](https://docs.stripe.com/api/events) — cited in `07_Mock_Interviews/04_Stripe_Payments_Platform.md`.
- [Payouts API](https://docs.stripe.com/api/payouts) — cited in `07_Mock_Interviews/04_Stripe_Payments_Platform.md`.
- [Automating data removal](https://engineering.fb.com/2023/10/31/data-infrastructure/automating-data-removal/) — cited in `07_Mock_Interviews/05_Instagram_Social_Platform.md`.
- [How Meta understands data at scale](https://engineering.fb.com/2025/04/28/security/how-meta-understands-data-at-scale/) — cited in `07_Mock_Interviews/05_Instagram_Social_Platform.md`.
- [Migrating Data Ingestion Systems at Meta Scale](https://engineering.fb.com/2026/05/12/data-infrastructure/migrating-data-ingestion-systems-at-meta-scale/) — cited in `07_Mock_Interviews/05_Instagram_Social_Platform.md`.
- [Redshift table design best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html) — cited in `07_Mock_Interviews/06_Amazon_Marketplace.md`.
- [Choosing the best sort key](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-sort-key.html) — cited in `README.md`.
- [Clustering keys and clustered tables](https://docs.snowflake.com/en/user-guide/tables-clustering-keys) — cited in `README.md`.
- [Data skipping](https://docs.databricks.com/aws/en/tables/data-skipping) — cited in `README.md`.
- [Factors affecting query performance](https://docs.aws.amazon.com/redshift/latest/dg/c-query-performance.html) — cited in `README.md`.
- [Liquid clustering](https://docs.databricks.com/aws/en/tables/clustering) — cited in `README.md`.
- [Optimizing query performance](https://docs.snowflake.com/en/user-guide/performance-query-options) — cited in `README.md`.
- [Search Optimization Service](https://docs.snowflake.com/en/user-guide/search-optimization-service) — cited in `README.md`.
- [When to partition tables](https://docs.databricks.com/aws/en/tables/partitions) — cited in `README.md`.

## 4. Platform-specific validation checklist

- **Amazon Redshift:** Validate sort/distribution choices, compression, workload patterns, concurrency, and query plans against current Redshift documentation. Avoid assuming traditional OLTP indexes are available in the same way as row-store databases.
- **Snowflake:** Validate micro-partition pruning, clustering, Search Optimization, materialized views, and dynamic-table behavior against the actual workload and current product documentation.
- **Databricks / Delta Lake:** Validate partitioning versus liquid clustering, data skipping, file sizing, table maintenance, and streaming/CDC semantics for the runtime and table format in use.
- **Kafka / CDC:** Validate delivery guarantees, ordering scope, key selection, schema compatibility, replay behavior, and sink idempotency. Kafka offsets are transport coordinates, not universal business-event identifiers.
- **SQL dialects:** Treat DDL and date/time functions in lessons as illustrative. Confirm syntax, timestamp semantics, decimal precision, constraints, and merge behavior for the chosen engine.
- **Financial systems:** Confirm currency minor units, FX source/time, posting semantics, settlement definitions, and reconciliation rules with Finance and the payment provider.

## 5. Interview answer checklist

Before finalizing a design, check that you can answer:

- What business decision does this model support?
- What is the grain of every fact and bridge?
- What are the natural keys and surrogate keys?
- Which dimensions are conformed, role-playing, or slowly changing?
- Which measures are additive, semi-additive, or non-additive?
- What happens on retries, duplicates, out-of-order events, late arrivals, corrections, and deletions?
- How are facts reconciled to operational and financial sources?
- What are the freshness, availability, quality, and recovery SLOs?
- How does the model scale in volume, cardinality, skew, and concurrency?
- What data is sensitive, who can access it, and how is retention enforced?
- What is the migration and adoption plan across teams?

## 6. Important modelling cautions

- A fact table's grain must be consistent; do not mix header, line, event, and snapshot rows in one table without an explicit design reason.
- A bridge table models a relationship (often many-to-many); a factless fact table records an event or coverage occurrence. They are related patterns, not interchangeable terms.
- An accumulating snapshot is useful for lifecycle milestones but does not replace immutable event history when auditability or repeated transitions matter.
- Inventory and balance snapshots are generally semi-additive over time; do not sum daily balances across dates as if they were transaction amounts.
- A business object state (for example, payment succeeded or reservation confirmed) does not necessarily prove settlement, occupancy, delivery, or cash movement.
- Distinct counts, attribution, conversion, SLA, occupancy, and revenue metrics require explicit business definitions; SQL alone cannot settle ambiguous semantics.

## 7. Course map

The masterclass contains seven modules: interview fundamentals; business requirements; high-level model diagrams and practice domains; dimension design; fact modeling; performance optimization; and six end-to-end mock interviews.

Use the individual lesson files for the detailed explanations, diagrams, worked examples, SQL, dialogue, exercises, and scoring rubrics. This reference index is a companion, not a replacement for the lessons.

## Module 7 source-modeling note

The six mock interviews now include illustrative operational source schemas and sample records. These are educational fixtures, not proprietary schemas from the named companies. Validate exact provider object relationships and current API semantics against official documentation. Each case also includes case-specific relationship diagrams, target DDL examples and SQL exercises.


## Additional references used to strengthen Module 7

- [The Data Modeling Interview Framework](https://www.datainterview.com/courses/data-modeling/playbook-framework) — discovery-first interview dialogue and grain-first design.
- [Ride-sharing warehouse practice case](https://www.datainterview.com/courses/cloud-data-warehousing/rideshare-warehouse) — worked ride-sharing warehouse example.
- [DataDriven ride-sharing schema practice](https://datadriven.io/problems/ride_sharing_platform_schema) — source entities, keys and relationship exercise.
- [Uber Schemaless: scalable datastore using MySQL](https://www.uber.com/us/en/blog/schemaless-part-one-mysql-datastore/) — evolving trip records and late-arriving data.
- [Meta: Migrating Data Ingestion Systems at Meta Scale](https://engineering.fb.com/2026/05/12/data-infrastructure/migrating-data-ingestion-systems-at-meta-scale/) — CDC snapshot/delta migration and reliability.
- [Meta: How Meta understands data at scale](https://engineering.fb.com/2025/04/28/security/how-meta-understands-data-at-scale/) — schemas, annotations, data classification and privacy-aware assets.
- [Meta: Data lineage at scale](https://engineering.fb.com/2025/01/22/security/how-meta-discovers-data-flows-via-lineage-at-scale/) — source-to-sink lineage and privacy controls.
