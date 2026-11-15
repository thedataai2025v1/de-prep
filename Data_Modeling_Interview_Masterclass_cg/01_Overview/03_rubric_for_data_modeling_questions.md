# Lesson 3: Rubric for Data Modeling Questions

## Learning objectives

-   Understand the dimensions on which interviewers evaluate a modeling
    answer.
-   Recognize the difference between a plausible schema and a strong
    engineering design.
-   Self-assess answers and identify gaps.
-   Demonstrate Senior Staff and Principal-level judgment.

## 1. Evaluation rubric

Use this 100-point rubric as a practice framework. Actual interview
rubrics vary by company and interviewer.

  ------------------------------------------------------------------------
  Evaluation area                             Weight What strong
                                                     performance looks
                                                     like
  --------------------- ---------------------------- ---------------------
  Requirement discovery                           15 Clarifies business
                                                     goals, users,
                                                     metrics, constraints

  Business                                        10 Identifies processes,
  understanding                                      entities,
                                                     definitions, edge
                                                     cases

  Grain and correctness                           15 Defines grain
                                                     precisely; avoids
                                                     fan-out and
                                                     double-counting

  Model design                                    15 Selects appropriate
                                                     facts, dimensions,
                                                     keys, relationships

  History and lifecycle                           10 Handles changes, late
                                                     data, corrections,
                                                     retention

  Query and performance                           10 Connects model to
  reasoning                                          query patterns and
                                                     physical design

  Scale and reliability                           10 Considers volume,
                                                     freshness,
                                                     incremental loads,
                                                     recovery

  Communication                                    5 Structured, concise,
                                                     clear, collaborative

  Trade-offs and                                  10 Explains
  evolution                                          alternatives, risks,
                                                     and future change

  **Total**                                  **100** 
  ------------------------------------------------------------------------

## 2. Performance bands

  ------------------------------------------------------------------------
                         Score Interpretation        Next step
  ---------------------------- --------------------- ---------------------
                       90--100 Excellent             Sharpen concise
                                                     explanations and edge
                                                     cases

                        80--89 Strong                Improve trade-off
                                                     depth and operational
                                                     details

                        70--79 Competent             Practice grain,
                                                     history, and
                                                     requirement discovery

                        60--69 Developing            Revisit fundamentals
                                                     and structured
                                                     problem solving

                      Below 60 Needs more practice   Rebuild the design
                                                     from requirements
                                                     upward
  ------------------------------------------------------------------------

These are self-assessment bands, not a claim about any specific
company's hiring thresholds.

## 3. What weak, good, and excellent answers look like

### Requirement discovery

-   **Weak:** Starts drawing tables immediately.
-   **Good:** Asks about KPIs, users, freshness, and scale.
-   **Excellent:** Prioritizes questions by how much they could change
    the model, identifies conflicting definitions, and states
    assumptions.

### Grain

-   **Weak:** Says "one row per order" without checking the use case.
-   **Good:** Chooses order-line grain for product sales analysis.
-   **Excellent:** Separates order, order-line, payment, refund, and
    shipment processes where their grains differ; explains safe ways to
    combine them.

### History

-   **Weak:** Says "we keep history."
-   **Good:** Identifies Type 1 or Type 2 based on reporting needs.
-   **Excellent:** Explains effective dates, current-row handling,
    late-arriving dimensions, corrections, event time, and as-of
    reporting.

### Performance

-   **Weak:** Says "add indexes."
-   **Good:** Considers partitioning and aggregates.
-   **Excellent:** Connects physical layout to platform, selectivity,
    data size, concurrency, query patterns, maintenance cost, and
    measured performance.

### Principal-level judgment

-   **Weak:** Focuses only on a local schema.
-   **Good:** Considers scalability and downstream consumers.
-   **Excellent:** Addresses ownership, shared definitions, contracts,
    migration, governance, operational cost, and organizational
    adoption.

## 4. Red flags interviewers may notice

-   No business questions before design.
-   No grain statement.
-   Mixing different grains in one fact table.
-   Treating every measure as additive.
-   Confusing a natural key with a surrogate key.
-   Using SCD Type 2 for every attribute without justification.
-   Ignoring deletes, corrections, late-arriving data, and duplicates.
-   Joining multiple one-to-many tables and creating fan-out.
-   Claiming partitioning or indexing always improves performance.
-   Ignoring privacy, retention, or access controls.
-   Being unable to explain why the design meets the stated queries.

## 5. Self-assessment scorecard

After each mock interview, score yourself from 1 to 5 in each area.

  -----------------------------------------------------------------------
  Area              1                 3                 5
  ----------------- ----------------- ----------------- -----------------
  Requirements      Assumed           Asked key         Prioritized and
                    everything        questions         resolved
                                                        ambiguity

  Grain             Unclear           Mostly correct    Precise and
                                                        defended

  Model             Incomplete        Works for main    Handles edge
                                      use case          cases and
                                                        multiple
                                                        processes

  History           Ignored           Basic SCD         Correct temporal
                                      awareness         and late-data
                                                        strategy

  Performance       Generic           Some workload     Evidence-driven
                                      awareness         platform-aware
                                                        design

  Communication     Disorganized      Understandable    Structured,
                                                        concise,
                                                        interactive

  Trade-offs        One answer only   Some alternatives Clear decision
                                                        criteria and
                                                        consequences

  Principal         Table-level only  System-level      Domain,
  judgment                            awareness         governance,
                                                        migration, cost,
                                                        ownership
  -----------------------------------------------------------------------

A score of 5 should require concrete evidence from your answer, not
confidence alone.

## 6. Worked scoring example

**Prompt:** Design a warehouse for subscription billing.

Candidate response: - Clarifies monthly recurring revenue, churn,
renewals, failed payments, and refunds. - Separates subscription
lifecycle events, invoices, invoice lines, and payment transactions. -
Defines one row per invoice line for billed item analysis and one row
per payment attempt for payment analysis. - Explains that current
subscription state and subscription event history answer different
questions. - Discusses late-arriving payment confirmations, idempotent
ingestion, and reconciliation. - Proposes date-based pruning only after
understanding query patterns and platform behavior.

Illustrative score: - Requirements: 14/15 - Business understanding:
9/10 - Grain: 14/15 - Model: 13/15 - History: 9/10 - Performance: 8/10 -
Scale/reliability: 9/10 - Communication: 4/5 - Trade-offs/evolution:
9/10 - **Total: 89/100**

The score is illustrative. The important output is the feedback: the
candidate should validate physical performance and explain semantic
definitions such as MRR, refunds, and churn more precisely.

## 7. Interviewer--candidate conversation

**Interviewer:** Your schema looks reasonable. How do I know it is
correct?

**Candidate:** I'd validate it against the business questions and
invariants. For example, order counts should not change when joining
order lines; revenue should reconcile to the agreed financial
definition; and historical segment reporting should use the segment
effective at the relevant business time.

**Interviewer:** What if the query results are duplicated?

**Candidate:** I'd inspect the grain and join cardinality first. A
one-to-many join can multiply rows. I'd verify key uniqueness, aggregate
child processes at the correct grain, or keep separate facts and combine
their aggregates at a shared dimensional level.

**Interviewer:** What makes this a Principal-level design?

**Candidate:** Beyond correctness and performance, I'd define ownership
and shared semantics, document contracts, plan schema evolution and
migration, establish quality and reconciliation controls, and make the
model usable across teams without creating conflicting definitions.

## 8. Mock interview scoring template

Use this after each practice session:

-   Problem:
-   Assumptions made:
-   Clarifying questions asked:
-   Fact grains stated:
-   Facts and dimensions identified:
-   History strategy:
-   Queries validated:
-   Trade-offs explained:
-   Important edge cases missed:
-   Score by rubric:
-   Three improvements for next attempt:

## 9. Revision checklist

-   [ ] I know the main evaluation dimensions.
-   [ ] I can recognize common red flags.
-   [ ] I can distinguish good from excellent reasoning.
-   [ ] I can score my own mock interviews with evidence.
-   [ ] I address organizational and operational concerns at Principal
    level.

## Key takeaway

**A strong model is correct, explainable, workload-aware, evolvable, and
governed---not merely normalized or fast.**
