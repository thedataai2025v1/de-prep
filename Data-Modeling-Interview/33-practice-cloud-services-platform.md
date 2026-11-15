# Lesson 33 — Practice: Cloud Services Platform

## Learning Objectives
- Define the entity hierarchy for a multi-tenant cloud platform
- Distinguish measured usage, rated cost, invoiced amounts, and financial adjustments
- Model effective-dated resource ownership and rate-card changes
- Reconcile estimated cost to final invoice amounts
- Handle shared resources, credits, and multi-currency billing

## Interview Prompt

Design an analytical data model for a cloud provider offering compute, storage, databases, and networking services. Customers create organizations and projects, provision resources, generate usage, receive bills, and may receive discounts or credits.

### 1. Clarify Business Questions

- What is a customer: organization, account, tenant, or billing entity?
- Can one organization have multiple billing accounts?
- Can projects share resources?
- How are resources created, resized, stopped, and deleted?
- What is the usage measurement interval?
- How are meters, units, rates, tiers, and discounts defined?
- Are costs estimated in near real time and finalized later?
- How are credits, taxes, and currency conversion handled?

### 2. Candidate Conceptual Diagram

```text
ORGANIZATION 1 ─────< PROJECT
ORGANIZATION 1 ─────< BILLING_ACCOUNT
PROJECT 1 ─────< RESOURCE
RESOURCE 1 ─────< RESOURCE_LIFECYCLE_EVENT
RESOURCE 1 ─────< RESOURCE_USAGE
SERVICE 1 ─────< METER
METER 1 ─────< RATE_VERSION
BILLING_ACCOUNT 1 ─────< INVOICE 1 ─────< INVOICE_LINE
INVOICE_LINE ─────< USAGE_ALLOCATION >──── RESOURCE_USAGE
```

This is an analytical view. Actual commercial and accounting models may have more detailed contract, entitlement, tax, and settlement structures.

### 3. Candidate Grains

| Fact | Grain |
|------|-------|
| Resource lifecycle event | One row per resource state transition |
| Resource inventory snapshot | One row per resource per snapshot time |
| Usage measurement | One row per resource, meter, and measurement interval |
| Rate version | One row per meter/rate/tier/effective period |
| Invoice line | One row per billed line item |
| Credit/adjustment | One row per credit or adjustment event |

Usage and invoice lines are not necessarily one-to-one. Invoice lines may aggregate many usage records, and usage can be adjusted or rated later.

### 4. Temporal and Financial Semantics

A rate may change over time. Preserve effective dates and associate usage with the applicable rate version. Clarify whether pricing is based on event time, billing period, contract version, or another rule.

Separate:
- Raw measured usage
- Rated usage or estimated cost
- Invoice/billed amount
- Credit and adjustment
- Recognized revenue, if required by finance

Do not label all of these as `cost` or `revenue`.

## Interview Dialogue

**Interviewer:** How do you model cloud cost per customer?

**Candidate:** I'd clarify the allocation policy and customer hierarchy. Usage may be measured at resource and meter interval grain, while billing lines may aggregate usage. I'd map usage to billing accounts and projects, apply effective-dated rates, and represent credits and adjustments separately.

**Interviewer:** A resource changes project ownership. What happens to historical cost?

**Candidate:** I'd preserve effective-dated resource ownership or record the ownership allocation at usage time. Otherwise, historical cost could be incorrectly attributed to the current project.

**Interviewer:** Why keep usage separate from invoice lines?

**Candidate:** Usage is a measurement process; invoice lines are a billing process. They may differ due to aggregation, minimum charges, rounding, credits, corrections, or contract terms. Keeping both enables reconciliation.

## Follow-Up Questions

1. How do you model resource hierarchy?
2. How do you track resource ownership changes?
3. How do you handle rate-card changes?
4. How do you reconcile estimated cost to invoice amount?
5. How do you allocate shared resources?
6. How do you support multi-currency and tax reporting?

## Principal-Level Discussion

- Define common customer, account, project, resource, and service semantics
- Govern cost allocation and shared-resource policies
- Build auditable usage-to-invoice lineage
- Handle late usage and invoice adjustments
- Separate high-volume telemetry from financial reporting

## Evaluation Rubric

| Criterion | Weight |
|-----------|--------|
| Domain discovery | 20% |
| Entity and hierarchy design | 20% |
| Usage and billing grains | 25% |
| Temporal pricing and reconciliation | 20% |
| Communication | 15% |

## Key Takeaways

| Concept | Key Point |
|---------|-----------|
| Entity hierarchy | Organisation → project → resource with effective-dated ownership |
| Usage vs billing | Keep raw measured usage separate from rated cost and invoiced amount |
| Rate versioning | Effective-dated rates; associate usage with the applicable rate version |
| Ownership changes | Preserve historical project ownership to avoid incorrect cost attribution |
| Credits and adjustments | Track separately from usage and invoice lines for reconciliation |
| Multi-currency | Record original currency, exchange rate, and converted amount |
| Telemetry vs finance | Separate high-volume telemetry from financial reporting grain |

### Related Lessons
- [Lesson 21](./21-slowly-changing-dimensions-types-1-4.md) — Slowly Changing Dimensions (Types 1–4)
- [Lesson 24](./24-fact-table-types.md) — Fact Table Types
- [Lesson 33](./33-practice-cloud-services-platform.md) — Practice: Cloud Services Platform
- [Lesson 34](./34-practice-online-advertising-platform.md) — Practice: Online Advertising Platform
