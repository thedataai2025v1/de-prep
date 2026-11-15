# Lesson 1: Introduction to Data Modeling Questions

> **Principal Mentor Note:** At the Senior level, interviewers test whether you *know* dimensional modeling. At the Staff and Principal levels, they test whether you know **when dimensional modeling breaks**, how your logical schema maps to physical cloud storage (Iceberg, Delta, Snowflake, BigQuery), and how your model survives 3 years of organizational and product evolution without requiring a full rewrite.

---

## 1. What Interviewers Are Actually Evaluating

When an interviewer asks *"Design a data model for Uber / Stripe / Airbnb"*, they are **not** looking for a single textbook Entity-Relationship Diagram (ERD). In fact, there is no single "correct" schema. Instead, they are evaluating four distinct engineering muscles:

1. **Ambiguity Navigation & Product Sense:** Can you translate vague stakeholder complaints (*"Leadership doesn't trust our churn numbers"*) into crisp business processes, clear entity boundaries, and unambiguous metrics?
2. **Grain Discipline:** Do you lock down the exact atomic meaning of a single row in every table *before* listing columns? (90% of data modeling failures stem from mixing grains in a single fact table.)
3. **Read vs. Write & Cost Trade-Offs:** Do you understand the physical consequences of your logical model? Every join, `MERGE` (upsert), SCD Type 2 history chain, or unpartitioned scan translates directly into compute dollars and SLA breaches.
4. **Evolvability & Operational Resilience:** Can your schema handle late-arriving facts, retroactive dimension corrections, schema evolution, PII/GDPR deletions, and idempotent backfills without breaking downstream BI dashboards or ML feature pipelines?

---

## 2. Types of Data Modeling Interviews

Not all data modeling rounds are identical. Recognize which variant you are in within the first 3 minutes:

| Interview Archetype | Primary Focus | Typical Prompt Style | Key Artifacts Expected |
| :--- | :--- | :--- | :--- |
| **Analytical / Warehouse Modeling (Kimball / Lakehouse)** | BI reporting, executive KPIs, metric consistency, historical tracking | *"Design the analytics data warehouse for a food delivery app."* | Bus matrix, Star/Snowflake schema, Fact types (Transaction/Snapshot/Accumulating), SCD strategy, Partitioning. |
| **Operational / OLTP Modeling** | Transactional integrity, concurrency, locking, low-latency point lookups | *"Design the database schema for a double-entry ledger or ticket booking system."* | 3NF / BCNF tables, foreign keys, composite indexes, isolation levels, optimistic/pessimistic locking. |
| **Product / Event & ML Feature Modeling** | High-throughput telemetry, sessionization, point-in-time correctness | *"Model user clickstream events to power both funnel analytics and real-time recommendations."* | Semi-structured event envelopes, sessionization logic, One Big Table (OBT) vs. normalized event layers, PIT joins. |
| **Hybrid / End-to-End Platform Modeling** | Ingestion from OLTP $\rightarrow$ Lakehouse $\rightarrow$ Serving | *"How do you model order state changes from microservices into a gold layer?"* | CDC ingestion patterns, Bronze/Silver/Gold contracts, late-arriving data handling, data quality SLAs. |

---

## 3. Core Interview Questions & Model Answers

### Q1: "Why do we still care about dimensional modeling when modern columnar warehouses (Snowflake, BigQuery, Databricks) have massive compute and cheap storage?"

**Weak / Junior Answer:**
> *"Because star schemas reduce storage space by removing duplicate strings and make SQL queries easier to write."*

**Principal-Level Answer:**
> *"Storage is cheap, so we don't do dimensional modeling to save bytes anymore—we do it to enforce **semantic consistency**, **predictable query performance**, and **blast-radius containment**.*
>
> *First, without conformed dimensions, five different teams will define 'Active Enterprise Customer' five different ways in ad-hoc OBT (One Big Table) pipelines. Second, while modern MPP engines handle wide scans well, updating a mutable attribute—like reassigning a sales territory—across a 50-billion-row denormalized table requires rewriting massive partitions, whereas updating a 5-million-row SCD Type 2 dimension is sub-second. Finally, in a lakehouse architecture, we often expose **Star Schema in the Silver/Gold core** for governance and reuse, and selectively materialize **One Big Table (OBT)** views or tables in the consumption layer for BI tools like Tableau or Looker where join elimination isn't guaranteed."*

---

### Q2: "Walk me through the difference between an Operational (OLTP) data model and an Analytical (OLAP) data model for the exact same business domain—say, an E-commerce Checkout."

**Principal-Level Answer:**
> *"In an **OLTP checkout database**, the workload is high-concurrency, small-payload reads and writes (`INSERT`, `UPDATE`) keyed by `order_id` or `user_id`. We normalize to **3NF** to eliminate write anomalies—if a user updates their shipping address before fulfillment, we update one row in `users` or `order_addresses` without locking historical order items. We optimize for B-tree/LSM index point lookups and ACID transactions.*
>
> *In an **OLAP warehouse**, the workload is low-concurrency, high-throughput sequential scans and aggregations across millions of rows (`SUM(net_revenue) BY region, product_category, quarter`). Joins across 12 normalized tables kill columnar scan efficiency and confuse analysts. Here, we denormalize attributes into **Dimensions** (`dim_product`, `dim_customer`, `dim_date`) and capture immutable business events in **Fact tables** (`fct_order_line_items`), partitioned by date and clustered by high-cardinality filter keys."*

---

## 4. Conversational & "Top-Up" Follow-Up Questions

Interviewers frequently use these rapid-fire follow-ups during the opening minutes to calibrate your seniority:

1. **Top-Up 1: "When would you intentionally choose One Big Table (OBT) over a Star Schema?"**
   * **Expected Insight:** When serving a single, well-defined BI dashboard or ML training dataset where the join topology is fixed, dimensions rarely undergo retroactive historical restatements, and the BI engine (e.g., PowerBI DirectQuery, Apache Druid, ClickHouse) penalizes runtime joins heavily. Emphasize that OBT should be a **downstream materialization** built *from* a clean dimensional core, not the primary source of truth.
2. **Top-Up 2: "What happens when a single business entity—like an Order—has both immutable line-item events and mutable lifecycle timestamps?"**
   * **Expected Insight:** Do not force them into one table. Model a **Transaction Fact** (`fct_order_items`) at the order-line grain for revenue/quantity math, and a separate **Accumulating Snapshot Fact** (`fct_order_fulfillment`) at the order grain to track milestone timestamps (`ordered_at`, `paid_at`, `shipped_at`, `delivered_at`) and milestone lag durations.
3. **Top-Up 3: "How do you handle a stakeholder who insists on putting every single JSON attribute from an upstream Kafka event into the core fact table?"**
   * **Expected Insight:** Apply the **Curated Core + Semi-Structured Extension** pattern. Promote high-value, SLA-backed, frequently filtered attributes to strongly typed top-level columns, and retain the rest in a `VARIANT` / `JSONB` / `MAP` `extended_attributes` column. This prevents schema churn from breaking downstream contracts while preserving 100% data fidelity for ad-hoc exploration.

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Starts with business processes & metrics** before drawing a single box or table. | Immediately writes `CREATE TABLE users (...)` without asking what queries or KPIs matter. |
| **Explicitly states the grain** of every fact table in plain English (e.g., *"One row per rider trip leg"*). | Mixes header-level metrics (e.g., `order_shipping_fee`) into a line-item fact table without allocation rules. |
| **Connects logical design to physical execution** (partition pruning, file compaction, `MERGE` amplification). | Treats the data warehouse like an infinite-speed magic box with zero physical constraints. |
| **Proactively surfaces edge cases** (cancellations, refunds, late events, timezone boundaries, currency conversion). | Designs only for the "happy path" where every order succeeds cleanly and arrives on time. |