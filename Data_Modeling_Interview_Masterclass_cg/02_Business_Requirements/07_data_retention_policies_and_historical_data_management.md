# Lesson 7: Data Retention Policies and Historical Data Management

## Learning objectives

-   Distinguish retention, archival, deletion, and historical modeling.
-   Define time semantics and historical reporting requirements.
-   Understand late-arriving data, corrections, and restatements.
-   Include privacy, legal, and governance constraints in model design.

## 1. Retention is a business and governance decision

A retention policy specifies how long data is kept, where it is kept,
who can access it, and what happens at the end of the retention period.

Clarify: - Legal or regulatory requirements. - Contractual
obligations. - Business and analytical value. - Privacy and purpose
limitation. - Storage and operational cost. - Audit and investigation
needs. - Deletion and backup-expiry behavior. - Data residency and
access restrictions.

Do not assume that keeping everything forever is acceptable or that
deleting old data is always safe.

## 2. Distinguish lifecycle actions

-   **Hot data:** Frequently accessed, optimized for current workloads.
-   **Warm data:** Retained and queryable but less frequently accessed.
-   **Cold/archive data:** Lower-cost storage, potentially slower to
    retrieve.
-   **Deletion:** Removal or irreversible de-identification according to
    policy.
-   **Legal hold:** Data whose deletion is suspended under an authorized
    requirement.

Archiving is not the same as deletion. A policy must also consider
replicas, extracts, caches, downstream copies, backups, and derived
datasets.

## 3. Historical truth: what question must be answered?

Common requirements: - Current state: What is true now? - As-was
reporting: What did we believe or know at that time? - As-of business
reporting: What was valid in the business at a specified time? - Event
history: What sequence of events occurred? - Audit history: Who changed
what, when, and through which process?

These are different temporal questions.

## 4. Event time, effective time, and system time

-   **Event time:** When the business event occurred.
-   **Effective/valid time:** When an attribute or relationship is true
    in the business domain.
-   **System/recorded time:** When the platform recorded the
    information.

A customer address correction may be entered today but be effective from
last month. If the model only stores the latest value, it cannot answer
every historical question.

A bitemporal model may preserve both valid time and system time when
audit and correction requirements justify the complexity.

## 5. Historical dimension management

Type 1 overwrites an attribute and does not preserve its prior value in
the dimension. Type 2 adds a new version with validity metadata so facts
can be associated with the appropriate historical version.

For a Type 2 dimension, common columns include: - Surrogate key. -
Business key. - Effective start timestamp/date. - Effective end
timestamp/date. - Current-row indicator. - Tracked descriptive
attributes.

Define boundary semantics carefully, commonly using half-open intervals
such as `[effective_from, effective_to)` to avoid overlap at adjacent
boundaries.

## 6. Late-arriving and corrected data

Examples: - A trip event arrives hours after the trip. - A customer
dimension is created after a fact arrives. - A source corrects a
transaction amount. - A refund is recorded in a later accounting period
but relates to an earlier sale.

Define: - Whether reports use event date or posting date. - Whether
prior periods are restated. - How provisional values are identified. -
How corrections are audited. - Whether consumers receive change
notifications. - How reprocessing avoids duplicate effects.

## 7. Retention by data class

A useful policy matrix:

  Data class           Questions to decide
  -------------------- ------------------------------------------------------
  Raw events           Replay, audit, source retention, privacy
  Curated facts        Analytical value, correction window, reporting rules
  Dimensions           Historical reporting and legal needs
  Aggregates           Rebuildability, retention, deletion propagation
  Logs and telemetry   Security needs, operational value, cost
  Personal data        Purpose, lawful basis, access, deletion obligations

The actual retention duration must be set by authorized business, legal,
privacy, and security owners---not guessed by the data engineering team.

## 8. Interviewer--candidate conversation

**Interviewer:** The company wants five years of customer history. How
would you design it?

**Candidate:** I'd clarify which history is required: customer activity
events, changes to customer attributes, account state, or audit history.
I'd also identify regulatory and privacy constraints, then define
retention separately for each data class.

**Interviewer:** Why not just use SCD Type 2?

**Candidate:** Type 2 tracks selected dimension attribute changes. It
does not automatically preserve every event or every correction. If we
need to know both when a fact was valid and when the organization
learned it, we may need event history or bitemporal tracking.

**Interviewer:** A customer requests deletion. What happens to derived
tables?

**Candidate:** I'd follow the approved privacy workflow and data lineage
to identify affected raw, curated, aggregate, and downstream copies. The
deletion or de-identification behavior must be propagated and verified,
subject to legal holds and applicable policy. Backups and external
extracts need explicit treatment too.

## 9. Common mistakes

-   Treating retention as only a storage-cost decision.
-   Confusing archive with delete.
-   Keeping personal data indefinitely without purpose.
-   Using SCD Type 2 as a substitute for event history.
-   Ignoring late-arriving corrections and period restatement.
-   Failing to propagate deletion to derived data.
-   Leaving time-boundary semantics ambiguous.

## 10. Practice questions

1.  What is the difference between event time, valid time, and system
    time?
2.  When is SCD Type 2 insufficient?
3.  How do you handle late-arriving facts?
4.  How do you decide whether to restate historical reports?
5.  What is the difference between archival and deletion?
6.  How would you design a data deletion workflow across derived
    datasets?

## 11. Principal-level follow-ups

-   How do you create enterprise retention standards across domains?
-   How do you prove deletion has propagated through lineage?
-   How do you balance auditability with privacy obligations?
-   How do you manage legal holds and immutable storage?
-   How do you handle a change in retention policy for already-published
    data products?

## Revision checklist

-   [ ] I distinguish retention, archival, and deletion.
-   [ ] I understand temporal semantics.
-   [ ] I can choose between current-state, event, SCD, and bitemporal
    history.
-   [ ] I plan for late data and corrections.
-   [ ] I include downstream copies and privacy governance.

## Key takeaway

**Historical data design must preserve the kind of truth the business
needs while respecting retention, privacy, and legal constraints.**
