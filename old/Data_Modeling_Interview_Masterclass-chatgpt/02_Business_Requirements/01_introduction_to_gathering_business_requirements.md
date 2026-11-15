# Lesson 1: Introduction to Gathering Business Requirements

## Learning objectives

-   Explain why requirements discovery precedes schema design.
-   Identify stakeholders, decisions, metrics, constraints, and system
    boundaries.
-   Separate stated requirements from assumptions and inferred needs.
-   Lead a structured discovery conversation.

## 1. Why requirements gathering matters

A data model is successful only when it represents the right business
concepts and supports the decisions its consumers need to make. If the
team misunderstands the business problem, a technically elegant schema
can still produce incorrect metrics, poor adoption, or expensive rework.

Requirements gathering is therefore a design activity, not an
administrative phase. It helps determine: - What business process is
being represented? - Which decisions will the data support? - Who uses
the data, and how? - What does each metric mean? - What level of detail
is needed? - How fresh, complete, and historically accurate must the
data be? - Which legal, security, operational, and cost constraints
apply?

## 2. Categories of requirements

### Business requirements

The outcomes the organization wants, such as reducing customer churn or
improving driver utilization.

### Analytical requirements

Questions, metrics, dimensions, filters, drill paths, cohorts, and
reports that users need.

### Data requirements

Required entities, attributes, identifiers, relationships, history,
source systems, quality expectations, and lineage.

### Non-functional requirements

Latency, availability, throughput, scale, concurrency, reliability,
recoverability, security, privacy, and cost.

### Governance and lifecycle requirements

Ownership, access policies, retention, deletion, auditability,
regulatory obligations, and metric definitions.

## 3. Stakeholder discovery

Typical stakeholders include: - Business sponsor: desired outcomes and
success criteria. - Domain expert: process definitions, exceptions, and
terminology. - Analyst or BI developer: metrics, dimensions, and query
patterns. - Application or source-system owner: event semantics and
source limitations. - Data engineer: ingestion, transformation, quality,
and operations. - Security, privacy, and compliance: access, retention,
and permitted use. - Platform team: supported capabilities, reliability,
and cost constraints.

Do not assume one stakeholder knows the full process. Compare
perspectives and resolve conflicting definitions explicitly.

## 4. A practical discovery sequence

1.  Ask what decision the data should improve.
2.  Identify the business process and its start/end points.
3.  Ask for the top five questions or KPIs.
4.  Clarify the definition and calculation of each KPI.
5.  Identify the required grain and drill-down level.
6.  Understand filters, joins, and common query patterns.
7.  Establish freshness and historical requirements.
8.  Estimate volume, growth, and concurrency.
9.  Identify source systems, data ownership, and quality risks.
10. Confirm privacy, retention, and access requirements.
11. Summarize assumptions and ask stakeholders to validate them.

## 5. Requirements traceability

Maintain a simple mapping from business need to implementation and
validation.

  -----------------------------------------------------------------------
  Business question Metric /          Data requirement  Validation
                    dimension                           
  ----------------- ----------------- ----------------- -----------------
  What was daily    Net sales, date,  Order-line sales  Reconcile to
  net sales by      product           and return        approved sales
  product?                            treatment         definition

  Which customers   Active status,    Activity events   Compare against
  became inactive?  customer, time    and inactivity    agreed cohort
                                      rule              

  How long did      Resolution        Case lifecycle    Validate sample
  support cases     duration, case,   timestamps        cases with
  take?             team                                support
  -----------------------------------------------------------------------

This traceability prevents a common failure: building tables that look
complete but do not answer the original questions.

## 6. Interviewer--candidate conversation

**Interviewer:** Design a data warehouse for a subscription business.
What do you ask first?

**Candidate:** I'd clarify the decisions the warehouse should support.
For example, do we need revenue recognition, subscription growth, churn,
failed-payment analysis, customer lifetime value, or operational billing
monitoring? Those use cases influence the processes and grains we model.

**Interviewer:** The business says it wants a churn dashboard. Is that
enough?

**Candidate:** Not yet. I'd ask what counts as churn: cancellation
request, subscription expiration, non-renewal, or a payment failure
after a grace period. I'd also clarify whether churn is logo-based,
subscription-based, or revenue-based, and how reactivations are treated.

**Interviewer:** What do you do with conflicting answers?

**Candidate:** I'd document the competing definitions, identify the
business owner accountable for the metric, show how each definition
changes the result, and get an explicit decision. I would not silently
choose one in a transformation.

## 7. Common mistakes

-   Starting with source columns instead of business decisions.
-   Treating a KPI name as a complete definition.
-   Asking many low-value questions without prioritization.
-   Ignoring consumers outside the analytics team.
-   Failing to record assumptions and decisions.
-   Confusing a stakeholder request with a verified requirement.

## 8. Practice questions

1.  What is the difference between a business requirement and a data
    requirement?
2.  Who should participate in requirements discovery?
3.  How do you resolve conflicting definitions of revenue?
4.  How do you know when requirements are sufficiently clear to begin
    modeling?
5.  How would you create traceability from a KPI to source data and
    validation?

## 9. Principal-level follow-ups

-   How do you establish accountability for enterprise metrics?
-   How do you prevent multiple domains from defining customer or
    revenue differently?
-   How do you handle a sponsor who wants a delivery date before
    requirements are understood?
-   How do you prioritize requirements when cost and time are
    constrained?
-   How do you preserve decisions and assumptions as teams and
    requirements change?

## 10. Revision checklist

-   [ ] I can identify stakeholder groups.
-   [ ] I distinguish business, analytical, data, and non-functional
    requirements.
-   [ ] I can define a discovery sequence.
-   [ ] I connect requirements to data and validation.
-   [ ] I document assumptions, owners, and unresolved questions.

## Key takeaway

**Gather requirements around business decisions and measurable outcomes;
do not mistake a list of requested columns for a data-model
specification.**
