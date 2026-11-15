# Lesson 5.1 — Transaction Fact Tables

## 1. Purpose and mental model

A transaction fact table records individual business events at a declared grain. Examples include an order line being placed, a payment being captured, a page being viewed, a trip being completed, or a support interaction being created.

The most important principle is **one row represents one precisely defined event or measurement occurrence**. Grain is a business statement, not merely a list of foreign keys.

A transaction fact is often the most detailed and flexible analytical representation of a business process. It supports drill-down, reconciliation, event-level analysis, and the later construction of snapshots and aggregates.

### Learning objectives
- Declare and defend transaction grain.
- Distinguish event, transaction, line-item, and state-change grains.
- Select dimensions, measures, keys, and degenerate identifiers.
- Understand additive, semi-additive, and non-additive measures.
- Design for late data, corrections, duplicates, and high volume.
- Explain transaction fact design in a senior/principal interview.

---

## 2. Four-step dimensional design applied to facts

A reliable design sequence is:

1. **Select the business process.** Examples: order capture, payment settlement, trip completion, support case handling.
2. **Declare the grain.** Describe exactly what one row represents.
3. **Identify dimensions.** Determine the descriptive context valid at that grain.
4. **Identify facts.** Include only measurements that are true at that grain.

Do not start with a list of columns copied from source tables. First identify the business event and its measurement point.

### Example: e-commerce order

Possible grains:
- One row per order.
- One row per order line.
- One row per payment attempt.
- One row per shipment package.
- One row per refund line.

These are different business processes or event levels. An order-level total should not be repeated on every order-line row and then summed; that multiplies the amount by the number of lines.

A useful rule: if two measures are produced at different grains, they usually belong in different fact tables or require an explicit allocation/aggregation design.

## 3. Grain: the design decision that prevents most modelling errors

Consider a retail order with three products and two payment attempts. It may generate:
- 1 order header
- 3 order lines
- 2 payment attempts
- 1 shipment
- 1 refund later

A single wide “order fact” joining all these child records can produce 3 × 2 = 6 rows before shipment/refund joins. Order totals and payment amounts will be duplicated.

Instead, model separate processes at their natural grains:
- `fact_order_line`: one row per order line.
- `fact_payment_attempt`: one row per payment attempt.
- `fact_shipment`: one row per shipment/package.
- `fact_refund_line`: one row per refunded order line.

Use conformed dimensions and shared identifiers to analyse across processes. Do not join fact tables directly at incompatible grains and then aggregate duplicated measures.

### Grain declaration examples

| Business process | Good grain statement |
|---|---|
| Retail sales | One row per scanned or sold order line |
| Payment | One row per payment attempt or ledger posting (choose one) |
| Web analytics | One row per tracked event |
| Ride sharing | One row per completed trip |
| Support | One row per agent-customer interaction |
| Inventory movement | One row per stock movement line |

“Daily sales by product” is an aggregate grain, not the lowest event grain.

## 4. Typical transaction fact structure

A fact table usually contains:
- Dimension foreign keys.
- Numeric measures.
- Event timestamp or date key.
- Degenerate identifiers where useful.
- Technical lineage columns where operationally justified.

Example:

```sql
CREATE TABLE fact_order_line (
    order_line_id       VARCHAR(80) NOT NULL, -- degenerate business identifier
    order_id            VARCHAR(80) NOT NULL,
    order_date_sk       INTEGER NOT NULL,
    customer_sk         BIGINT NOT NULL,
    product_sk          BIGINT NOT NULL,
    store_sk            BIGINT NOT NULL,
    channel_sk          INTEGER NOT NULL,
    currency_sk         INTEGER NOT NULL,
    quantity            DECIMAL(18, 3) NOT NULL,
    unit_price          DECIMAL(18, 4),
    gross_amount        DECIMAL(18, 2) NOT NULL,
    discount_amount     DECIMAL(18, 2) NOT NULL,
    tax_amount          DECIMAL(18, 2) NOT NULL,
    net_amount          DECIMAL(18, 2) NOT NULL,
    event_ts            TIMESTAMP NOT NULL,
    source_system       VARCHAR(40) NOT NULL,
    source_event_id     VARCHAR(120),
    load_ts             TIMESTAMP NOT NULL
);
```

This is illustrative DDL. Identity/constraint syntax, clustering, partitioning, and enforcement vary by warehouse. In a platform where PK/FK constraints are informational, pipeline tests must enforce uniqueness and referential integrity.

### Should a transaction fact have a primary key?

The business event identifier, source event ID, or composite natural key may uniquely identify a row. A warehouse-generated fact surrogate key can be useful for lineage or update management, but is not automatically required. Many analytical fact tables are physically organized around partitioning and sort/clustering choices rather than a conventional OLTP primary key.

Define the deduplication identity explicitly. A timestamp alone is almost never a safe unique key.

## 5. Measures: additive behaviour

### Additive measures

Can be summed across all relevant dimensions:
- Quantity sold.
- Extended sales amount.
- Discount amount.
- Number of units shipped.

### Semi-additive measures

Can be summed across some dimensions but not others:
- Account balance can be summed across accounts at one point in time, but not across dates.
- Inventory on hand can be summed across products and locations at a snapshot, but not across daily snapshots.

### Non-additive measures

Should not be summed:
- Unit price.
- Percentage or rate.
- Conversion rate.
- Temperature.
- Average handling time as a precomputed average.

Store additive components whenever possible. For example, store `discount_amount` and `gross_amount`, then calculate:

```sql
SUM(discount_amount) / NULLIF(SUM(gross_amount), 0) AS discount_rate
```

Do not use `AVG(discount_rate)` across rows unless the business explicitly wants an unweighted average of row-level rates. A weighted rate is usually calculated from aggregated numerator and denominator.

### Currency and units

Amounts in different currencies are not directly additive. Either:
- Keep original amount and currency key.
- Store standardized reporting-currency amount using an approved FX rate and conversion timestamp.
- Preserve both values and document the conversion policy.

Likewise, quantities in different units require conversion to a common unit before aggregation.

## 6. Event time, processing time, and business time

A production event pipeline may have several timestamps:
- `event_ts`: when the event occurred in the source/business.
- `source_commit_ts`: when the source committed it.
- `ingest_ts`: when the platform received it.
- `process_ts`: when a transformation processed it.
- `effective_ts`: when the business says the state became effective.

These are not interchangeable. For event analytics, business event time is usually the analytical timestamp. For latency monitoring, ingestion and processing timestamps are also valuable.

Store timezone semantics clearly. A timestamp without timezone interpretation can create incorrect daily and hourly aggregations, especially across daylight-saving changes or globally distributed users.

## 7. Corrections, reversals, and mutable source events

The statement “transaction facts are immutable” is a modelling ideal, not a guarantee that source data never changes. Operational systems can:
- Correct an amount.
- Cancel an order.
- Reverse a payment.
- Delete an event.
- Send duplicate or out-of-order CDC records.

Choose a policy based on audit and accounting needs:
1. **In-place correction:** update the erroneous row; simple, but loses prior warehouse state unless audited elsewhere.
2. **Reversal event:** retain the original event and add an offsetting event. Often preferable for financial/ledger semantics.
3. **Versioned event:** preserve revisions with event version or valid-time metadata.
4. **Restatement:** rebuild affected partitions or date ranges from authoritative source data.

Do not treat a cancellation as a deletion by default. A cancellation is often a business event with its own timestamp and reason.

## 8. Late-arriving facts and dimensions

A late-arriving fact is received after its business event time. It should normally be assigned to the event's date and the dimension versions valid at event time, not the ingestion date or current dimension row.

A late-arriving dimension means the fact's referenced entity is not yet available. Use a governed inferred/unknown member or delay publication according to the data contract. Later enrichment must preserve key integrity.

Operational checks:
- Event uniqueness/deduplication.
- Late-arrival distribution and maximum lateness.
- Missing dimension key rate.
- Reversal/correction rate.
- Measure reconciliation to source.
- Currency and unit consistency.
- Event-time partition backfill strategy.

## 9. Worked example: payment lifecycle

Do not collapse payment authorization, capture, settlement, refund, and chargeback into one “payment amount” row if the business needs to analyse each event.

A clean event model might have one row per payment lifecycle event:

```sql
CREATE TABLE fact_payment_event (
    payment_event_id   VARCHAR(120) NOT NULL,
    payment_id         VARCHAR(100) NOT NULL,
    event_type_sk      INTEGER NOT NULL,
    event_date_sk      INTEGER NOT NULL,
    customer_sk        BIGINT NOT NULL,
    merchant_sk        BIGINT NOT NULL,
    payment_method_sk  INTEGER NOT NULL,
    currency_sk        INTEGER NOT NULL,
    event_amount       DECIMAL(18, 2),
    event_ts           TIMESTAMP NOT NULL,
    source_event_id    VARCHAR(120) NOT NULL
);
```

This supports counts and amounts by event type. However, summing all event types may not represent net cash movement: an authorization, capture, refund, and chargeback have different financial meanings. The semantic layer should define which event types contribute to each metric.

An alternative is separate facts for payment attempts, ledger postings, and settlement batches when their grains and accounting semantics differ.

## 10. Query patterns and fact-to-fact integration

To compare sales and payments by common dimensions, aggregate each fact independently to the same conformed dimensional grain, then join the aggregates.

```sql
WITH sales AS (
    SELECT order_date_sk, customer_sk, SUM(net_amount) AS sales_amount
    FROM fact_order_line
    GROUP BY order_date_sk, customer_sk
),
payments AS (
    SELECT payment_date_sk AS order_date_sk,
           customer_sk,
           SUM(captured_amount) AS captured_amount
    FROM fact_payment_capture
    GROUP BY payment_date_sk, customer_sk
)
SELECT
    COALESCE(s.order_date_sk, p.order_date_sk) AS date_sk,
    COALESCE(s.customer_sk, p.customer_sk) AS customer_sk,
    s.sales_amount,
    p.captured_amount
FROM sales s
FULL OUTER JOIN payments p
  ON s.order_date_sk = p.order_date_sk
 AND s.customer_sk = p.customer_sk;
```

This “drill across” approach avoids raw fact-to-fact fan-out. It still requires aligned definitions, compatible dimensions, and careful treatment of unmatched rows.

## 11. Interview conversation

**Interviewer:** Design a fact table for an e-commerce platform.

**Candidate:** I would first confirm the business process and grain. If the main requirement is product sales analysis, I would choose one row per order line. I would then identify order date, customer, product, seller/store, channel, promotion, and currency dimensions, and define measures such as quantity, gross amount, discount, tax, and net amount.

**Interviewer:** Why not one row per order?

**Candidate:** An order can contain multiple products, categories, and sellers. Order-line grain preserves product-level analysis. I would use a separate order-header fact if header-level measures or lifecycle analysis are required. I would not repeat order-level totals on every line.

**Interviewer:** The order can have multiple payments. How do you join payments?

**Candidate:** Payment attempts are a separate process and have a different grain. I would model one row per payment attempt or payment event, depending on the questions. For reporting across sales and payments, I would aggregate each fact to a common dimensional grain and drill across rather than join raw facts.

**Interviewer:** What happens when a payment is refunded?

**Candidate:** I would preserve the refund as a distinct event or reversal/ledger posting. The metric definition must state whether it reports gross captured, refunded, net captured, or settled cash. A single mutable amount would obscure the lifecycle and audit trail.

**Interviewer:** How do you make the load idempotent?

**Candidate:** Define a stable event identity, deduplicate before merge/append, and use source event IDs or log positions where available. Reprocessing the same source window should not create duplicate business events. I would test replay behaviour and reconcile counts and amounts to the source.

## 12. Common interview questions

1. What is a transaction fact table?
2. Why is grain the first design decision?
3. What is the difference between order grain and order-line grain?
4. Explain additive, semi-additive, and non-additive measures.
5. Should unit price be stored in a fact table?
6. How do you model multiple payments per order?
7. How do you avoid fact-to-fact join fan-out?
8. How do you handle late-arriving facts?
9. Are fact tables always immutable?
10. How do you model cancellations, refunds, and reversals?
11. What is a degenerate dimension?
12. How would you design a transaction fact with billions of rows?

### Follow-up / top-up questions

- What if the source has no unique event ID?
- What if two events have the same timestamp and business key?
- How do you deduplicate CDC events while preserving legitimate repeated events?
- How do you handle a correction that arrives months later?
- How do you reconcile source totals when source and warehouse currencies differ?
- What is the difference between a transaction fact and an event log?
- When would you keep raw event payloads outside the dimensional fact?
- How do you choose partition columns when event time and ingestion time differ?
- What if a dimension key is unknown at initial load?
- How do you handle a many-to-many promotion allocation?

### Principal-level probes

- Design a canonical event-to-fact ingestion architecture for multiple source systems.
- How do you provide exactly-once business semantics over at-least-once delivery?
- How do you govern metric definitions across domains?
- How would you support both financial auditability and low-latency analytics?
- What is your strategy for backfills without disrupting downstream consumers?
- How do you decide whether a process deserves one fact table or multiple complementary facts?

## 13. Practice exercise

Design transaction facts for a ride-sharing platform:
- Trip requested, accepted, driver arrived, trip started, trip completed.
- Fare estimate, final fare, toll, tax, tip, promotion, refund.
- Multiple payment attempts and possible chargebacks.

Deliver:
1. Grain statement for each proposed fact.
2. Dimension keys and measures.
3. Event-time and ingestion-time policy.
4. Deduplication identity and correction strategy.
5. A query for completed trips and net revenue by city and month.

**Evaluation rubric (10 points):**
- Grain correctness: 2
- Separation of distinct processes: 2
- Measures and additivity: 2
- Late/correction/idempotency design: 2
- Query and operational reasoning: 2

## 14. Key takeaways

- A transaction fact captures an event at a declared grain.
- Grain must be a business definition, not just a technical key list.
- Separate processes with different grains instead of joining them into a fan-out-prone table.
- Store additive components and define currency/unit semantics.
- Late data, corrections, and reversals require explicit business and technical policies.
