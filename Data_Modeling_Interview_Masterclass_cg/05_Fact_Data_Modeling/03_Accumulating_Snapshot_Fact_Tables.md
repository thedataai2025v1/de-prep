# Lesson 5.3 — Accumulating Snapshot Fact Tables

## 1. Purpose

An accumulating snapshot fact table tracks a business process that has a predictable sequence of milestones and a reasonably well-defined beginning and end. A row is created when the process begins and updated as milestones occur.

Examples:
- Order fulfilment: order placed → approved → picked → shipped → delivered.
- Insurance claim: submitted → assessed → approved/denied → paid → closed.
- Loan application: submitted → verified → underwritten → approved → disbursed.
- Support case: opened → assigned → first response → resolved → closed.

It provides an at-a-glance view of each process occurrence and makes cycle-time analysis convenient.

### Learning objectives
- Recognize workflows suitable for accumulating snapshots.
- Declare process-instance grain.
- Model milestone dates, status, durations, and role-playing date dimensions.
- Handle loops, reopened states, late events, and history limitations.
- Combine accumulating snapshots with transaction facts or effective-dated versions.

## 2. Grain

Typical grain:

> One row per order line fulfilment process instance.

If each order line can be split into multiple shipments, the grain may need to be one row per order-line shipment rather than one row per order line. The grain must match the actual process instance.

A process instance identifier (order line, claim, application, case, shipment) identifies the row. The row is updated as the process progresses.

## 3. Structure and example

```sql
CREATE TABLE fact_order_fulfilment (
    fulfilment_id          VARCHAR(100) NOT NULL,
    order_date_sk          INTEGER NOT NULL,
    approval_date_sk       INTEGER,
    pick_date_sk           INTEGER,
    ship_date_sk           INTEGER,
    delivery_date_sk       INTEGER,
    customer_sk            BIGINT NOT NULL,
    product_sk             BIGINT NOT NULL,
    warehouse_sk           BIGINT,
    carrier_sk             BIGINT,
    current_status_sk      INTEGER NOT NULL,
    ordered_qty            DECIMAL(18, 3),
    fulfilled_qty          DECIMAL(18, 3),
    order_to_ship_hours    DECIMAL(18, 2),
    ship_to_delivery_hours DECIMAL(18, 2),
    total_cycle_hours      DECIMAL(18, 2),
    is_complete            BOOLEAN NOT NULL,
    last_event_ts          TIMESTAMP NOT NULL,
    load_ts                TIMESTAMP NOT NULL
);
```

The date keys are role-playing uses of `dim_date`. Null milestone keys can mean “not reached yet,” but should not be confused with unknown or invalid timestamps. Some models use explicit not-applicable/not-yet-reached members rather than nulls, depending on platform and semantic conventions.

## 4. Why update a row?

Suppose an order line moves through five milestones. Rather than creating five accumulating-snapshot rows, maintain one row and populate milestone dates as the process advances.

Benefits:
- Easy to calculate elapsed time between milestones.
- Easy to find current pipeline bottlenecks.
- One row per process instance simplifies current-state reporting.
- The table is usually smaller than an event table for processes with multiple milestones.

Costs:
- Updates can create write amplification in append-oriented lakehouse systems.
- Intermediate state transitions may be lost if only the latest row is retained.
- Reopened or repeated milestones do not fit neatly into one date column per milestone.
- Historical “what did we know at a past point in time?” questions need additional history.

## 5. Duration measures

Duration can be:
- Calendar elapsed time.
- Business hours.
- Business days.
- Time excluding weekends/holidays.
- Time spent in a particular state.

These definitions differ. A simple timestamp subtraction calculates elapsed time, not necessarily business time.

Example:

```sql
SELECT
    fulfilment_id,
    ship_ts,
    delivery_ts,
    EXTRACT(EPOCH FROM (delivery_ts - ship_ts)) / 3600.0
        AS ship_to_delivery_hours
FROM fulfilment_process;
```

SQL date-difference syntax varies by engine. Business-time calculations need a governed working calendar and possibly operating-hours schedules.

Prefer retaining milestone timestamps (or date keys plus timestamps) so durations can be recalculated if business definitions change. Persisted duration columns can improve performance but should be derived and quality-checked.

## 6. State transitions and repeated milestones

A simple accumulating snapshot assumes each milestone occurs once in a predictable sequence. Real workflows may:
- Move from approved back to pending.
- Be denied, appealed, and approved later.
- Be assigned and reassigned multiple times.
- Have multiple shipment attempts.
- Be reopened after resolution.

A single `approved_date_sk` cannot explain multiple approval cycles. There are three common strategies:

1. **Accumulating snapshot + event transaction fact.** Keep the latest milestone summary in the accumulating row; record every transition in a separate event fact.
2. **Versioned accumulating snapshot.** Insert a new effective-dated version when the process state changes, preserving “as-of” views.
3. **State interval fact.** Record one row per process instance and state interval, with start/end timestamps. This is useful for time-in-state analytics.

For complex workflows, the event/transition fact should usually be the detailed source of truth, with the accumulating snapshot derived as a convenient current summary.

## 7. Pairing with a transaction fact

Example:
- `fact_case_event`: one row per case state transition or interaction.
- `fact_case_accumulating`: one row per case, updated with current milestones and status.

The event fact answers:
- How many times was a case reopened?
- Who changed its status?
- What was the sequence of states?
- How long did it remain in each state?

The accumulating snapshot answers:
- How many cases are currently open?
- What is the average time from opening to first response?
- Which milestone is currently blocking each case?
- What is the total elapsed time for completed cases?

Do not expect the accumulating snapshot alone to answer every historical workflow question.

## 8. Handling late and out-of-order events

Suppose a shipment event arrives after the delivery event due to source delay. Do not blindly set milestone columns in ingestion order. Use the event's business timestamp and transition rules.

Recommended pipeline:
1. Store raw events immutably with source event ID and event time.
2. Deduplicate by stable event identity.
3. Order events per process instance using source sequence/commit metadata where reliable.
4. Validate allowed transitions.
5. Recompute the current milestone summary deterministically.
6. Update the accumulating snapshot idempotently.
7. Alert on impossible sequences or missing required milestones.

If two events have the same event time, use source sequence or a documented tie-breaker. Processing timestamp should not silently determine business chronology.

## 9. Current-state versus historical pipeline reporting

A standard accumulating snapshot represents the latest known state. It does not inherently preserve the state at every past date.

If the business asks “How many applications were in underwriting on 31 March, as we knew it then?”, options include:
- A periodic snapshot of pipeline state.
- A transaction event fact plus state reconstruction.
- Effective/expiration timestamps on accumulating snapshot versions.
- A state-interval fact.

Kimball's guidance describes pairing an accumulating snapshot with a transaction fact when the intermediate transitions and historical evolution matter. A time-stamped accumulating snapshot can also preserve versions, but adds storage and query complexity.

## 10. Worked example: loan application

Grain: one row per loan application.

Milestones:
- Application submitted.
- Documents completed.
- Verification completed.
- Underwriting completed.
- Decision made.
- Funds disbursed.

Measures:
- Requested amount.
- Approved amount.
- Days from submit to verification.
- Days from verification to decision.
- Total time to disbursement.

Dimensions:
- Applicant/customer.
- Product.
- Branch/channel.
- Loan officer.
- Decision reason.
- Role-playing dates for milestones.

A separate `fact_application_event` stores every status transition and document event. The accumulating snapshot is refreshed from the ordered event stream.

Important questions:
- Can an application be resubmitted?
- Can it be withdrawn and reopened?
- Can multiple applicants be associated with one application?
- Can a loan be disbursed in tranches?
- Is “decision date” the initial decision or final decision after appeal?

These questions can change the grain.

## 11. SQL pattern: derive current milestone state from events

The following is conceptual SQL. The exact event ordering and transition logic depend on source semantics.

```sql
WITH ranked_events AS (
    SELECT
        application_id,
        event_type,
        event_ts,
        source_sequence,
        ROW_NUMBER() OVER (
            PARTITION BY application_id, event_type
            ORDER BY event_ts DESC, source_sequence DESC
        ) AS rn
    FROM fact_application_event
),
milestones AS (
    SELECT
        application_id,
        MAX(CASE WHEN event_type = 'SUBMITTED' AND rn = 1
                 THEN event_ts END) AS submitted_ts,
        MAX(CASE WHEN event_type = 'VERIFIED' AND rn = 1
                 THEN event_ts END) AS verified_ts,
        MAX(CASE WHEN event_type = 'DECIDED' AND rn = 1
                 THEN event_ts END) AS decided_ts,
        MAX(CASE WHEN event_type = 'DISBURSED' AND rn = 1
                 THEN event_ts END) AS disbursed_ts
    FROM ranked_events
    GROUP BY application_id
)
SELECT *
FROM milestones;
```

This simplified query selects the latest occurrence of each event type. It is not sufficient when repeated events have different meanings (e.g., multiple disbursements or approval cycles). In those cases, model process stages/attempt IDs explicitly.

## 12. Interview conversation

**Interviewer:** Design a fact table for order fulfilment.

**Candidate:** I would use an accumulating snapshot if the process has a predictable set of milestones. The grain could be one row per order-line fulfilment instance. It would have role-playing date keys for order, approval, pick, ship, and delivery, plus customer, product, warehouse, carrier, quantities, status, and milestone durations.

**Interviewer:** What if one order line ships in two packages?

**Candidate:** Then one row per order line is not the correct process-instance grain for fulfilment. I would use one row per order-line shipment/package, or define a parent order-line process with a child shipment fact. The grain must represent the actual unit that moves through the milestones.

**Interviewer:** Can this table tell us every time an order was delayed and resumed?

**Candidate:** Not by itself. The accumulating row captures the latest milestone summary and can lose intermediate transitions. I would pair it with a transaction fact containing each status transition or use a state-interval model if time-in-state analysis is important.

**Interviewer:** The delivery event arrives before the shipping event. What do you do?

**Candidate:** I would retain the raw events, order them using business event time and source sequence, validate the transition rules, and recompute the process summary deterministically. I would not assume ingestion order equals business order.

## 13. Common interview questions

1. What is an accumulating snapshot fact table?
2. How does it differ from a transaction fact?
3. What is its grain?
4. Why does it have multiple date keys?
5. What measures can it contain?
6. How do you calculate cycle time?
7. What happens when a process is reopened?
8. How do you preserve intermediate state transitions?
9. How do you handle out-of-order events?
10. When would you use a periodic snapshot instead?

### Follow-up / top-up questions

- What if a process has optional milestones?
- What if the process can branch into parallel paths?
- What if one order line has multiple shipments?
- How do you model retries and repeated approvals?
- How do you calculate business-hours SLA?
- How do you report pipeline state as of a historical date?
- What if a late event changes a previously calculated duration?
- How do you handle updates efficiently in an append-oriented lakehouse?
- Can an accumulating snapshot be rebuilt from an event log?
- What is the difference between a milestone fact and a state-transition fact?

### Principal-level probes

- Design a canonical process-event model that supports multiple downstream workflow summaries.
- How do you ensure event replay produces the same accumulating snapshot?
- How would you model parallel and optional workflow stages?
- How do you preserve auditability while serving low-latency current-state dashboards?
- What is the appropriate balance between event facts, state intervals, periodic snapshots, and accumulating snapshots?
- How do you handle a workflow definition that changes over time?

## 14. Practice exercise

Design an accumulating snapshot for customer support cases:
- Cases can be assigned and reassigned.
- First response and resolution SLAs differ by priority.
- Cases can be reopened.
- A case can be escalated to another team.
- Some cases are merged.

Specify grain, milestones, dimensions, duration measures, event-history companion table, late-event handling, and historical as-of strategy.

**Evaluation rubric (10 points):**
- Correct process-instance grain: 2
- Milestone and dimension design: 2
- Reopen/loop handling: 2
- Historical and late-event strategy: 2
- SLA and operational semantics: 2

## 15. Key takeaways

- Accumulating snapshots summarize a process instance as it advances through milestones.
- The grain is one row per process occurrence, not one row per event.
- Multiple milestone dates are role-playing date dimensions.
- Repeated transitions and historical state require a companion event or state-history model.
- Build the snapshot deterministically from events when replayability and auditability matter.
