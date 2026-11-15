# Lesson 2: How to Answer Data Modeling Questions (The 45-Minute Execution Framework)

> **Principal Mentor Note:** Most candidates fail data modeling interviews not because they lack SQL or Kimball knowledge, but because of **time mismanagement**. They spend 25 minutes debating minor dimension attributes and run out of time before discussing partitioning, late-arriving data, or answering the interviewer's actual analytical queries. Use a strict time-boxed execution blueprint.

---

## 1. The 45-Minute Time-Boxed Blueprint

```
[00 - 06 min] Phase 1: Clarify Scope, Users & Business Questions
[06 - 12 min] Phase 2: Define Entities, Grain & Bus Matrix
[12 - 26 min] Phase 3: Core Dimensional / Physical Schema Design
[26 - 33 min] Phase 4: Query Validation (Dry-Run the Hardest KPIs)
[33 - 41 min] Phase 5: Scale, Partitioning, CDC & Edge Cases
[41 - 45 min] Phase 6: Trade-Offs, Evolution & Wrap-Up
```

---

## 2. Step-by-Step Execution Guide

### Phase 1: Clarify Scope, Users & Business Questions (Minutes 0–6)
Never guess requirements. Ask targeted, high-leverage questions grouped into **4 buckets**:
1. **Consumers & Use Cases:** *"Who are the primary consumers—executive BI dashboards, product analysts running funnel queries, operational alerting, or ML feature pipelines?"*
2. **Core KPIs & Metrics:** *"What are the top 3–5 decisions or metrics this model must power? (e.g., Net Revenue, Take Rate, Rider Utilization, 7-day Retention)."*
3. **Scale & Latency:** *"What is the event volume (e.g., 5M vs. 5B events/day), and what is the freshness SLA—daily batch, hourly micro-batch, or sub-minute streaming?"*
4. **History & Compliance:** *"How many years of history must be retained, do we need point-in-time historical state (SCD Type 2), and are there GDPR/CCPA right-to-be-forgotten requirements?"*

> **Pro-Tip:** After asking, **summarize your assumptions out loud** and get verbal buy-in from the interviewer before moving on. *"Great—so we are designing an analytical warehouse for BI and product analytics, handling ~200M events/day with a 1-hour freshness SLA and 5 years of retention. Let's dive into the core business processes."*

---

### Phase 2: Define Entities, Grain & Bus Matrix (Minutes 6–12)
Before writing column names, establish the **conceptual backbone**:
* **Identify Core Business Processes:** e.g., *Trip Request*, *Trip Execution*, *Payment Settlement*, *Driver Payout*.
* **Declare the Exact Grain of Each Fact Table:** Write it explicitly on the whiteboard/doc.
  * *Bad:* "Grain: Trips"
  * *Principal:* "Grain: One row per completed or canceled ride request (`trip_id`), captured at terminal state transition."
* **Draft a Quick Enterprise Bus Matrix:** Show which dimensions conform across which facts.

```
Business Process / Fact          | Date | User | Driver | Vehicle | Geo/City | Promo
---------------------------------+------+------+--------+---------+----------+------
fct_trip_requests (Transaction)  |  X   |  X   |   X    |    X    |    X     |  X
fct_driver_daily_snap (Periodic) |  X   |      |   X    |    X    |    X     |
fct_payments (Transaction)       |  X   |  X   |   X    |         |    X     |  X
```

---

### Phase 3: Core Schema Design (Minutes 12–26)
Build the schema **inside-out** (Fact tables first, then Dimensions):
1. **Fact Tables:**
   * Primary Key / Degenerate Dimensions (e.g., `trip_id`, `invoice_number`).
   * Foreign Keys to Conformed Dimensions (`user_sk`, `driver_sk`, `date_sk`, `city_sk`).
   * Role-Playing Date/Time Keys (`requested_date_sk`, `completed_date_sk`, plus UTC `TIMESTAMP` columns).
   * Additive Measures (`gross_fare_usd`, `platform_fee_usd`, `distance_meters`, `duration_seconds`).
   * Semi-Additive / Non-Additive Measures (explicitly flag ratios or balances so analysts don't `SUM()` them blindly).
2. **Dimension Tables:**
   * Surrogate Key (`_sk`) vs. Natural/Durable Business Key (`_id`).
   * Descriptive attributes used for filtering (`WHERE`) and grouping (`GROUP BY`).
   * Explicitly mark **SCD Type 1** (overwrite) vs. **SCD Type 2** (`effective_from`, `effective_to`, `is_current`) per attribute group.

---

### Phase 4: Query Validation — Dry-Run the Hardest KPIs (Minutes 26–33)
**Do not wait for the interviewer to ask for SQL.** Proactively write 1 or 2 non-trivial SQL queries against your own schema to prove it works.
* Pick the trickiest metric discussed in Phase 1 (e.g., *Month-over-Month Net Take Rate by Driver Tier at the time of the trip*).
* Writing this query immediately exposes missing foreign keys, wrong grain, or broken SCD Type 2 joins before the interviewer catches them.

---

### Phase 5: Scale, Partitioning, CDC & Edge Cases (Minutes 33–41)
Transition from logical modeling to **physical data engineering**:
* **Partitioning & Clustering:** How is data laid out in Iceberg/Delta/Snowflake? (e.g., `PARTITIONED BY (DATE(completed_at_utc))`, `CLUSTER BY (city_id, driver_id)`).
* **Ingestion & Idempotency:** How do upstream CDC streams (Debezium/Kafka) populate these tables without duplicates on pipeline retries? (`MERGE INTO` keyed on natural key + watermark/sequence number).
* **Edge Cases:**
  * Late-arriving facts (a trip completed at 23:58 UTC arrives at 01:15 UTC the next day).
  * Late-arriving dimensions (a fact arrives with `driver_id = 999` before `dim_driver` has ingested driver 999 $\rightarrow$ inferred/placeholder member row).
  * Multi-currency normalization (`local_amount`, `currency_code`, `usd_exchange_rate`, `usd_amount`).

---

### Phase 6: Trade-Offs, Evolution & Wrap-Up (Minutes 41–45)
Conclude like a Principal owner by stating what you deliberately **deferred** and how the model evolves:
* *"Right now, we normalized `dim_driver` and `dim_vehicle` separately because drivers switch rental vehicles frequently. If BI query latency on executive dashboards exceeds our 3-second SLA, we can materialize a daily Gold OBT aggregate `agg_city_daily_kpis`."*

---

## 3. Worked Mini-Scenario: Demonstrating the Framework

**Prompt:** *"Design a data model for a subscription SaaS product like Notion or Figma."*

### Clarifying Dialogue (Simulated)
* **Candidate:** *"Before drawing tables, I'd like to clarify three things: (1) Are we modeling B2B workspace subscriptions, individual consumer plans, or both? (2) Are we primarily tracking recurring revenue metrics (ARR/MRR, churn, expansion) or product usage telemetry, or connecting the two? (3) Do workspaces change pricing tiers mid-billing cycle?"*
* **Interviewer:** *"Focus on B2B workspaces, specifically ARR/MRR movements—new business, upgrades, downgrades, and churn—and how they correlate with active seat counts. Upgrades happen mid-cycle."*

### Grain & Model Decision
* **Grain Trap Avoided:** If you only have a `fct_invoices` table, you cannot easily see active ARR on an arbitrary day or track zero-dollar churn events where no invoice is generated.
* **Principal Solution:** Pair a **Transaction Fact** (`fct_subscription_events` — one row per subscription state change: `CREATED`, `UPGRADED`, `DOWNGRADED`, `RENEWED`, `CANCELED`) with a **Monthly Periodic Snapshot Fact** (`fct_workspace_mrr_monthly` — one row per active or newly churned workspace per month-end, capturing `ending_mrr`, `net_mrr_change`, `mrr_movement_type`, `licensed_seats`, and `active_seats_30d`).

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "What if the interviewer gives you zero requirements and says, 'You're the Principal Data Engineer—you tell me what matters'?"**
   * **Response Strategy:** Don't stall. State a realistic, opinionated baseline: *"In a B2C marketplace, the three lifeblood loops are Supply Acquisition, Demand Conversion, and Unit Economics/Take Rate. I will scope our v1 warehouse around those three processes, starting with Demand Conversion and Unit Economics. Stop me if you'd like me to pivot to Supply."*
2. **Top-Up 2: "How do you handle a mid-interview scope pivot—e.g., at minute 30 they say, 'Actually, we now have 100x the data volume and need real-time fraud detection on this model'?"**
   * **Response Strategy:** Explicitly separate the **OLAP Analytical Plane** from the **Low-Latency Serving Plane**. Keep the Kimball warehouse model for historical reconciliation and analytics, and introduce an upstream streaming state store (Flink + RocksDB / Redis / Pinot) consuming the same Kafka topic for sub-second fraud scoring, sharing the exact same canonical event schema contract.

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Time-boxes proactively** and checks the clock at the 15-minute and 30-minute marks. | Spends 35 minutes listing 40 columns of `dim_user` and never designs a fact table. |
| **Validates the model with SQL** against the hardest metric before being prompted. | Leaves the whiteboard as a static diagram without verifying join paths or aggregation math. |
| **Explains the "Why" behind every modeling choice** in terms of query patterns and maintenance cost. | Dogmatically recites Kimball rules without connecting them to the specific business domain. |