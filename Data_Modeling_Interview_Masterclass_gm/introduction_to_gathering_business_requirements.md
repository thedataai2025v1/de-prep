# Lesson 5: Introduction to Gathering Business Requirements

> **Principal Mentor Note:** Junior engineers treat requirements gathering like a checklist (*"How many rows? What's the latency?"*). Principal Data Engineers treat it like **risk discovery**. Every question you ask in the first 6 minutes should eliminate an architectural trap—such as mixed grains, retroactive metric restatements, unscalable `MERGE` amplification, or silent timezone drift—before you draw a single table.

---

## 1. The Principal Requirements Discovery Funnel

When an interviewer gives an open-ended prompt (*"Design a data model for an online marketplace"*), use the **4-Layer Discovery Funnel** to extract technical constraints from business language:

```
Layer 1: BUSINESS OUTCOMES & CONSUMERS
  └─> Who consumes the data? What decisions/SLAs fail if this model is slow or wrong?

Layer 2: METRICS, GRAIN & SEMANTICS
  └─> What exact KPIs are computed? What is the finest atomic event? Where do edge cases hide?

Layer 3: ACCESS PATTERNS & TEMPORAL RULES
  └─> How will users slice/filter? Do we need point-in-time historical state (SCD2) or current state?

Layer 4: SCALE, LATENCY & LIFECYCLE
  └─> Rows/day, peak burst, freshness SLA, late-arriving data window, retention & GDPR rules.
```

---

## 2. Translating Vague Stakeholder Language into Technical Specs

In real interviews (and real Principal roles), interviewers answer in **product language**, not database language. You must translate immediately:

| What the Interviewer Says | Hidden Data Modeling Implication | Follow-Up Question to Lock It Down |
| :--- | :--- | :--- |
| *"Marketing wants to see conversion funnels."* | Ordered event sequence + session/attribution windows + un-converted drop-offs. | *"Is conversion measured within a single session, or across a 7-day / 30-day multi-touch window?"* |
| *"Finance needs to reconcile payouts every month."* | Immutable ledger entries, multi-currency FX rates, refunds/chargebacks, strict auditability. | *"Are refunds modeled as negative reversing transactions or updates to the original order?"* |
| *"Sales reps switch territories every quarter, and VPs compare YoY regional performance."* | Classic **SCD Type 2 vs. Type 1** dilemma (`as-was` vs. `as-is` reporting). | *"When a rep moves from West to East, should Q1 revenue stay with West (`as-was`) or move to East (`as-is`), or do leadership dashboards need both?"* |
| *"Our app works offline and syncs when riders get cell service."* | Out-of-order / **late-arriving facts** (`event_time` $\ll$ `ingestion_time`). | *"What is the 99th-percentile sync delay—minutes, hours, or up to 7 days—before we close a billing period?"* |
| *"Data scientists want to train churn models on this warehouse."* | **Point-in-Time (PIT) correctness** (zero data leakage from future dimension updates). | *"Do ML pipelines need exact feature snapshots as of the prediction timestamp?"* |

---

## 3. High-Leverage Question Bank (Memorize These Patterns)

Instead of asking 20 scattered questions, ask **4–5 crisp, compound questions** that demonstrate domain mastery:

1. **Consumer & Workload Split:**
   > *"Are we optimizing primarily for executive BI dashboards with predictable slice-and-dice filters, ad-hoc product analyst exploration, or downstream ML feature generation?"*
2. **Core Business Process & Edge Cases:**
   > *"Should we scope the v1 model to the core happy-path transaction, or must we also capture lifecycle state transitions like cancellations, partial refunds, and disputes?"*
3. **Historical Truth (`As-Was` vs. `As-Is`):**
   > *"When key entity attributes change—such as a merchant's fee tier or a user's subscription plan—do we need to preserve historical state at the time of the transaction (SCD Type 2), or always report under current attributes?"*
4. **Scale, Freshness & Late Data:**
   > *"Roughly what daily event volume and peak concurrency are we designing for, what is our freshness SLA (e.g., hourly vs. daily), and how late can upstream events arrive?"*

---

## 4. Real Interview Questions & Model Answers

### Q1: "Why not just capture every possible column and event from the source database so we never have to go back to stakeholders for requirements?"

**Principal-Level Answer:**
> *"That 'dump everything and figure it out later' approach works in the **Bronze raw ingestion layer**, and I always advocate preserving 100% of raw payloads there. However, pushing uncurated source schemas directly into the **Gold analytical model** creates three severe failures:*
> 1. ***Semantic Ambiguity:** Source OLTP databases encode business logic implicitly in application code—for example, `status = 4` might mean 'Canceled by Driver' in one microservice and 'Expired' in another. Without upfront business alignment, analysts bake conflicting `CASE` statements into every dashboard.*
> 2. ***Grain Corruption:** Copying normalized OLTP tables 1:1 leaves analysts joining 10 tables and accidentally triggering Fan Traps that double-count revenue.*
> 3. ***Compute & Maintenance Debt:** Building SCD Type 2 tracking, complex `MERGE` pipelines, and clustered partitions on 200 unused columns burns compute budget on every batch run and locks us into rigid downstream contracts."*

---

### Q2: "How do you handle conflicting requirements between two stakeholder teams—say, Finance wants strict immutable accounting snapshots while Product wants retroactive attribution updates?"

**Principal-Level Answer:**
> *"Never force two conflicting truth definitions into a single mutable table. Instead, separate them cleanly in the Gold layer:*
> * *For **Finance**, I model an **append-only accounting ledger / periodic snapshot fact** (`fct_finance_monthly_close`) where closed accounting periods are locked, and any late adjustments or chargebacks appear as new adjustment entries in the current open period.*
> * *For **Product**, I model a **restatable operational fact** (`fct_product_attribution`) with a sliding lookback window (e.g., re-processing the trailing 14 days on each run) so conversion cohorts reflect true user behavior.*
> *Both tables derive from the same validated Silver event stream, guaranteeing shared entity keys while honoring each team's business contract."*

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "What if the interviewer answers 'I don't know' to your volume or latency question?"**
   * **Expected Insight:** Propose realistic industry-standard numbers based on the company archetype, state them explicitly, and move forward. Example: *"If we're modeling a mid-to-large consumer marketplace, let's anchor on ~100M events/day (~15–20 GB/day compressed Parquet), a 1-hour micro-batch SLA for operational dashboards, and 5 years of retention. That puts our largest fact table around ~180B rows over 5 years, which makes partition pruning and incremental `MERGE` strategies essential."*
2. **Top-Up 2: "How do you know when to stop asking clarifying questions and start modeling?"**
   * **Expected Insight:** Watch the clock (aim for **4–6 minutes max**). The moment you know: (a) the **primary business process**, (b) the **top 2–3 KPIs**, (c) the **historical tracking need (SCD1 vs. SCD2)**, and (d) the **order of magnitude of scale/latency**, summarize your assumptions, ask *"Does this scope sound good to start?"*, and begin designing.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Asks closed-ended, hypothesis-driven questions** (*"Should partial refunds be modeled as negative line items or separate adjustment events?"*). | Asks open-ended, aimless questions (*"So... what columns do you want in the database?"*). |
| **Identifies cross-team semantic conflicts** (`as-was` vs. `as-is`, gross vs. net, Finance vs. Product) within the first 5 minutes. | Assumes every team defines "Revenue", "Active User", and "Order Date" identically. |
| **Summarizes requirements cleanly** and gets explicit interviewer alignment before moving to schema design. | Interrogates the interviewer for 15 minutes straight without writing anything down on the board. |