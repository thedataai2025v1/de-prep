# Lesson 2: How to Answer Data Modeling Questions

## Learning objectives

-   Use a consistent structure for open-ended interview questions.
-   Ask high-value clarifying questions without turning the interview
    into an interrogation.
-   Explain grain, facts, dimensions, relationships, and trade-offs.
-   Handle interviewer challenges and evolving requirements.

## 1. The 8-step interview framework

Use this sequence as a mental checklist. Adapt it to the problem rather
than reciting it mechanically.

### Step 1 --- Clarify the objective

Ask what decision or workflow the data model must support.

Examples: - Revenue and profitability reporting? - Operational
monitoring? - Customer behavior analysis? - Regulatory reporting? -
Product experimentation?

### Step 2 --- Identify users and use cases

Understand who will use the data and how: - Executives: trends and
KPIs. - Analysts: slice-and-dice exploration. - Data scientists:
features and historical labels. - Applications: low-latency lookups or
operational decisions.

### Step 3 --- Gather requirements and constraints

Cover: - Required metrics and their definitions. - Query patterns and
common filters. - Freshness and latency. - Data volume, growth, and
concurrency. - History and retention. - Privacy, access, compliance, and
data residency. - Source systems, quality, and ownership.

### Step 4 --- Identify business processes and entities

Separate processes such as order placement, payment capture, refund,
shipment, and return. Identify descriptive entities such as customer,
product, seller, location, and date.

### Step 5 --- Define grain

State one sentence for each fact table:

> "One row represents one \[business event or measurement\] for each
> \[business key\] at \[time or process stage\]."

Examples: - One row per order line. - One row per payment transaction. -
One row per driver per service day. - One row per account per day.

If the grain is unclear, stop and resolve it before listing measures.

### Step 6 --- Design facts and dimensions

For each fact: - Declare the grain. - Identify foreign keys and
degenerate identifiers. - Classify measures as additive, semi-additive,
or non-additive. - Identify dimensions needed for filtering and
grouping. - Check whether multiple processes require separate fact
tables.

### Step 7 --- Explain history and data lifecycle

Discuss: - Type 1 versus Type 2 history where relevant. - Event time
versus processing time. - Late-arriving facts and dimensions. -
Corrections, deletes, and reprocessing. - Retention, archival, and
deletion requirements.

### Step 8 --- Validate and optimize

Walk through representative queries and test whether the model answers
them correctly. Then discuss physical design: partitioning, clustering,
indexing, aggregates, incremental processing, and cost.

## 2. Clarifying questions: prioritize, don't overwhelm

A useful set of opening questions:

1.  What are the top business questions this model must answer?
2.  Is this an operational model or an analytical warehouse?
3.  What is the required level of detail---order, order line, event,
    customer-day?
4.  Which metrics need agreed definitions?
5.  How fresh must the data be?
6.  What are the approximate volume and growth?
7.  How much history must be retained?

Ask the questions most likely to change the design. State reasonable
assumptions for details that do not materially affect the first draft.

## 3. A whiteboard structure

Organize the board in this order:

1.  Scope and assumptions.
2.  Business processes.
3.  Grain statement for each fact.
4.  Fact tables and measures.
5.  Dimensions and relationships.
6.  Historical behavior.
7.  Example queries and validation.
8.  Physical and operational considerations.

Draw the model only after explaining the grain. Label relationship
cardinality and indicate whether keys are natural, surrogate, or
degenerate where useful.

## 4. How to explain trade-offs

Use this pattern:

> "I would choose **A** because it supports **requirement X**. The cost
> is **trade-off Y**. If **condition Z** becomes more important, I would
> consider **alternative B**."

Example:

> "I'd use a Type 2 customer dimension because analysts need to report
> using the customer's historical segment. This increases row count and
> requires effective-date joins. If the requirement is only to show the
> current segment, Type 1 is simpler and avoids historical versions."

This is stronger than saying one design is universally best.

## 5. Model validation through queries

For a sales model, validate: - Revenue by day and product. - Revenue by
customer segment as of sale date. - Number of orders versus number of
order lines. - Return amount by original sale period. - Payment totals
versus order totals.

Look for fan-out joins. If an order has three lines and two payment
attempts, joining both child tables at order level can produce six rows.
Aggregate or model each process at its own grain before combining
results.

## 6. Interviewer--candidate conversation

**Interviewer:** Design a warehouse for ride-sharing analytics.

**Candidate:** I'll assume the goal is business analytics rather than
the live dispatch application. I'd clarify the core KPIs---completed
trips, gross bookings, driver earnings, cancellations, wait time, and
utilization---and the required freshness.

**Interviewer:** Start drawing the schema.

**Candidate:** First I'll define the central process. For trip
performance, one row per trip is a reasonable initial grain. I'd have a
trip fact with trip identifiers, rider and driver keys, pickup and
drop-off location keys, request and completion date-time keys, status,
distance, duration, fare components, and relevant flags. Driver-day
earnings or driver availability would be separate facts if their grains
differ.

**Interviewer:** Why not put driver earnings in the trip fact?

**Candidate:** If earnings are fully attributable to each trip,
trip-level earnings can be a measure there. But payouts, adjustments,
bonuses, and settlements may occur at different grains and times. I'd
model those financial events separately and reconcile them to trips
where the business rules allow it.

**Interviewer:** What if the company wants a dashboard refreshed every
minute?

**Candidate:** I'd clarify the latency and consistency requirements. The
warehouse model can remain analytically sound, while ingestion and
serving architecture may use streaming or micro-batches, incremental
aggregates, and a serving layer. I would not compromise the grain just
to meet a refresh target.

## 7. Handling pushback

When challenged: 1. Confirm the concern. 2. Restate the requirement that
drove your choice. 3. Explain the trade-off. 4. Offer an alternative and
its consequences. 5. Ask whether the changed requirement should alter
the design.

Do not defend a choice after its assumptions have changed.

## 8. Common weak answers and stronger alternatives

  -----------------------------------------------------------------------
  Weak answer                         Stronger answer
  ----------------------------------- -----------------------------------
  "I'll create a fact table."         "I'll define the business process
                                      and grain, then choose the fact
                                      structure."

  "We'll use SCD Type 2."             "I'll use Type 2 for attributes
                                      whose historical state must be
                                      preserved; other attributes may use
                                      Type 1."

  "Partition by date."                "I'll choose partitioning based on
                                      common filters, data volume, file
                                      sizing, and platform behavior."

  "Use a star schema because it is    "A star schema simplifies
  faster."                            analytical joins and semantic
                                      consistency; I'll validate
                                      performance against actual
                                      workloads."

  "Put everything in one table."      "I'll keep different grains
                                      separate and provide a curated wide
                                      view if a consumer workload
                                      benefits."
  -----------------------------------------------------------------------

## 9. Practice prompts

For each prompt, speak for 5--8 minutes and draw a model: 1. Design a
sales warehouse. 2. Design a subscription analytics model. 3. Design
customer support analytics. 4. Design a ride-sharing analytics platform.
5. Design a social media engagement warehouse.

For each, explicitly state assumptions, grain, facts, dimensions,
history, and trade-offs.

## 10. Principal-level evaluation

A Principal-level answer should additionally cover: - Domain boundaries
and ownership. - Shared metrics and conformed dimensions. - Data
contracts and schema evolution. - Backfills, replay, and idempotency. -
Quality controls and reconciliation. - Cost and operational burden. -
Migration and consumer compatibility. - Governance, privacy, and
retention.

## 11. Revision checklist

-   [ ] I follow a structured approach without sounding scripted.
-   [ ] I ask questions that materially affect the design.
-   [ ] I state assumptions explicitly.
-   [ ] I define grain before measures.
-   [ ] I validate with realistic queries.
-   [ ] I explain trade-offs and alternatives.
-   [ ] I can respond constructively to pushback.

## Key takeaway

**The interviewer is evaluating your reasoning process as much as the
final schema.**
