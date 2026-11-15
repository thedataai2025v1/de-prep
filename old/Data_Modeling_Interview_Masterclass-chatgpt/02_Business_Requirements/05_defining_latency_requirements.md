# Lesson 5: Defining Latency Requirements

## Learning objectives

-   Distinguish data freshness, processing latency, query latency, and
    end-to-end decision latency.
-   Translate vague freshness requests into measurable service
    objectives.
-   Understand batch, micro-batch, and streaming trade-offs.
-   Identify how latency affects model and platform choices.

## 1. Latency is not one number

Clarify which latency the stakeholder means:

-   **Source delay:** Time between the business event and its
    availability in the source.
-   **Ingestion latency:** Time from source availability to landing in
    the data platform.
-   **Transformation latency:** Time spent processing and publishing
    curated data.
-   **Data freshness:** Age of the latest relevant event visible to a
    consumer.
-   **Query latency:** Time for a query or dashboard to return.
-   **Decision latency:** Time from event occurrence to an actionable
    decision.

A dashboard that loads in two seconds may still show data that is six
hours old.

## 2. Convert vague statements into an SLO

"Near real time" is ambiguous. Ask: - Is the target 5 seconds, 1 minute,
or 15 minutes? - Is the target average, median, or percentile? - Does it
apply continuously or only during business hours? - What is the
acceptable completeness level? - How should late or corrected events
appear? - What happens during source outages or backfills?

Example objective:

> 99% of eligible payment events should be available in the risk
> analytics serving dataset within 2 minutes of source publication,
> measured over a rolling 30-day window, excluding documented source
> outages.

The exact target must be agreed with stakeholders and validated against
platform capabilities and cost.

## 3. Batch, micro-batch, and streaming

  -----------------------------------------------------------------------
  Pattern                 Typical fit             Trade-offs
  ----------------------- ----------------------- -----------------------
  Batch                   Daily/weekly reporting  Lower operational
                                                  complexity; higher
                                                  freshness delay

  Micro-batch             Minutes-level freshness Balanced latency and
                                                  throughput;
                                                  scheduling/processing
                                                  overhead

  Streaming               Seconds or sub-seconds  Lower latency; more
                                                  operational complexity
                                                  and state management
  -----------------------------------------------------------------------

Streaming is not automatically better. Choose it when the business
decision genuinely requires the latency and the organization can operate
the system reliably.

## 4. Latency and modeling

Latency can affect: - Whether facts are append-only events or
periodically reconciled snapshots. - Whether dimensions are available
before facts. - How late-arriving data is handled. - Whether consumers
need provisional versus finalized metrics. - Whether aggregates are
updated incrementally. - How duplicate events and retries are managed.

Do not change a fact's business grain simply to meet a refresh target.
Solve ingestion and serving latency at the appropriate architectural
layer.

## 5. Event time and processing time

-   **Event time:** When the business event occurred.
-   **Ingestion/processing time:** When the platform received or
    processed it.

A trip may occur at 10:00, arrive from a mobile device at 10:07, and be
processed at 10:08. Reporting by event time and monitoring pipeline
delay answer different questions.

For event-time processing, consider watermarks, allowed lateness,
correction windows, and retractions or updates where supported by the
processing framework.

## 6. Example: fraud analytics

A fraud team may need a risk signal within seconds, while finance
reporting can be finalized daily.

A sensible architecture may have: - A low-latency event-processing path
for risk decisions. - A durable raw event history for replay and
audit. - A curated analytical model for investigation and trend
analysis. - A reconciled financial dataset for settled outcomes.

These datasets can share business identifiers and definitions without
forcing every consumer onto one serving path.

## 7. Interviewer--candidate conversation

**Interviewer:** The business wants real-time sales reporting. What
architecture do you choose?

**Candidate:** I'd clarify the decision that needs real-time data and
the actual freshness target. I'd also ask whether the metric must be
provisional or financially reconciled. A seconds-level operational view
and a finalized revenue report may need different paths.

**Interviewer:** Why not stream every table?

**Candidate:** Streaming adds state, ordering, replay, monitoring, and
operational complexity. If a 15-minute micro-batch meets the decision
need, it may be more reliable and cost-effective. I'd choose the
simplest architecture that meets the agreed SLO.

**Interviewer:** How do late events affect daily sales?

**Candidate:** I'd retain event time and ingestion time, define a
lateness and correction policy, and make the reporting semantics
explicit. Recent periods may be provisional and then restated as late
events arrive, with reconciliation and auditability.

## 8. Common mistakes

-   Treating query speed as data freshness.
-   Accepting "real time" without a measurable target.
-   Assuming streaming guarantees correctness or exactly-once business
    effects.
-   Ignoring late events and source delays.
-   Failing to distinguish provisional from finalized metrics.

## 9. Practice questions

1.  Explain the difference between freshness and query latency.
2.  When would micro-batch be preferable to streaming?
3.  How would you define a freshness SLO?
4.  How do event time and processing time differ?
5.  How do you handle late events in daily aggregates?

## 10. Principal-level follow-ups

-   How do you negotiate latency versus cost and reliability?
-   How do you define SLOs across source, platform, and consumer teams?
-   How do you provide both low-latency and reconciled reporting?
-   How do you measure freshness for sparse or irregular event streams?

## Revision checklist

-   [ ] I distinguish the different latency measures.
-   [ ] I translate vague freshness requests into SLOs.
-   [ ] I justify batch, micro-batch, or streaming.
-   [ ] I understand event time, processing time, and late data.
-   [ ] I distinguish provisional and finalized metrics.

## Key takeaway

**Choose latency targets from business decisions, then design ingestion
and serving paths to meet them reliably.**
