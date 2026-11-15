# Lesson 2: Recognizing the Core Business Problem

## Learning objectives

-   Translate a broad request into a specific business problem.
-   Identify users, decisions, outcomes, and process boundaries.
-   Separate symptoms from root causes.
-   Define scope and success criteria before modeling.

## 1. Request versus problem

A stakeholder may ask: "Build a dashboard showing customer activity."

That is a requested solution, not yet a well-defined problem. The
underlying need might be: - Identify customers at risk of churn. -
Understand which product features drive engagement. - Measure onboarding
completion. - Detect suspicious account activity. - Evaluate campaign
effectiveness.

Each problem requires different events, definitions, grains, history,
and latency.

## 2. The problem-framing canvas

Use these prompts:

  Prompt              What to discover
  ------------------- --------------------------------------------
  Who?                Users, decision-makers, affected customers
  What decision?      Action the consumer will take
  Why now?            Business driver and urgency
  Which process?      Process being measured and its boundaries
  What outcome?       KPI and success threshold
  At what level?      Entity, event, transaction, or time grain
  When needed?        Freshness and decision latency
  What constraints?   Privacy, quality, cost, systems, retention

A concise problem statement:

> "For \[user\], provide \[information or capability\] at \[grain and
> freshness\] so they can \[decision/action\], measured by \[success
> criteria\], subject to \[constraints\]."

## 3. Distinguish symptoms from root causes

Example: "Our customer retention is falling."

Possible underlying questions: - Is the decline concentrated in a
customer segment? - Did acquisition mix change? - Is onboarding
completion lower? - Are customers experiencing service failures? - Did
pricing or product packaging change? - Has the churn definition
changed? - Is the apparent decline caused by missing or delayed events?

The data model should make it possible to investigate plausible drivers,
but it should not encode an unverified causal explanation.

## 4. Define the process boundary

For an order process, clarify whether the scope starts at: - Cart
creation, - Checkout initiation, - Order submission, - Payment
authorization, - Payment capture, - Fulfillment, - Delivery, - Or final
settlement.

These are distinct events. A model that treats them as one "order" event
can obscure failed attempts, retries, cancellations, and partial
fulfillment.

## 5. Scope and prioritization

Classify requirements as: - **Must have:** Required for the primary
decision or compliance. - **Should have:** Material value, but not
essential for first release. - **Could have:** Useful enhancement. -
**Out of scope:** Explicitly excluded for this release.

Also identify dependencies and risks. Do not promise a single universal
model for unrelated operational and analytical workloads without
explaining the architecture boundary.

## 6. Example: customer support

Broad request: "We need support analytics."

Clarify whether the main problem is: - Reduce first response time. -
Improve resolution time. - Understand contact reasons. - Measure agent
productivity. - Detect repeat contacts. - Monitor SLA breaches. -
Understand customer satisfaction.

Potential processes: - Case creation and lifecycle. - Message or
interaction events. - Assignment and reassignment. - SLA milestones. -
Customer feedback.

A case-level accumulating snapshot may help analyze lifecycle
milestones, while interaction-level transaction facts may support
message and contact analysis. They answer different questions.

## 7. Interviewer--candidate conversation

**Interviewer:** A product manager asks for a customer 360 model. How do
you start?

**Candidate:** I'd ask which decisions the customer view must
support---service resolution, marketing segmentation, product
personalization, risk, or executive reporting. "Customer 360" can mean
different things across those domains.

**Interviewer:** The product manager says, "Everything about the
customer."

**Candidate:** I'd narrow the scope by users, priority journeys, data
sensitivity, and measurable outcomes. I'd also identify the customer
identity-resolution rules and authoritative sources. I would avoid
creating an unrestricted table containing every available attribute.

**Interviewer:** What if they insist on all data in one place?

**Candidate:** I'd explain the risks of mixed semantics, privacy
exposure, and unclear ownership. I'd propose governed domain datasets
and a curated access layer, with purpose-based views or products for
approved use cases.

## 8. Common mistakes

-   Accepting solution language without discovering the problem.
-   Treating correlation as causation.
-   Ignoring process start and end definitions.
-   Designing for every imaginable use case in the first release.
-   Omitting explicit exclusions and success measures.

## 9. Practice questions

1.  Turn "build a customer 360" into a testable problem statement.
2.  A retailer says sales are down. What hypotheses and data questions
    do you explore?
3.  How would you scope a first release for support analytics?
4.  Why must a process boundary be explicit?
5.  How do you distinguish a data-quality issue from a business trend?

## 10. Principal-level follow-ups

-   How do you challenge an executive's requested solution
    constructively?
-   How do you balance a domain-specific use case against enterprise
    reuse?
-   How do you prevent a "customer 360" initiative from becoming an
    uncontrolled data lake?
-   How do you prioritize when stakeholders have incompatible
    objectives?

## Revision checklist

-   [ ] I can reframe a solution request as a business problem.
-   [ ] I identify users, decisions, outcomes, and process boundaries.
-   [ ] I separate symptoms, hypotheses, and verified causes.
-   [ ] I define scope and success criteria.

## Key takeaway

**Model the business process and decision that matter---not the vague
label used in the initial request.**
