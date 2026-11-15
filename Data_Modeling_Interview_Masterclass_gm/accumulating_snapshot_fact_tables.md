# Lesson 26: Accumulating Snapshot Fact Tables

> **Principal Mentor Note:** Whenever an interviewer asks *"How do we measure conversion bottlenecks, stage-to-stage lag times, and SLA breaches across a multi-step workflow?"*—such as an **E-Commerce Order Pipeline**, **Loan Underwriting Process**, **Hiring/Recruiting Funnel**, or **Insurance Claim Lifecycle**—the textbook Kimball answer is an **Accumulating Snapshot Fact Table**. At the Principal level, you must also explain how to handle **non-linear workflows (loops/retries/cancellations)** and **high-volume `MERGE` write amplification**.

---

## 1. Core Definition & Architectural Properties

An **Accumulating Snapshot Fact Table** models a **finite, multi-stage lifecycle** with a clear beginning and end. Unlike a Transaction Fact (append-only) or Periodic Snapshot (new row every day), an Accumulating Snapshot creates **one single row per lifecycle instance** and **updates that row in-place (`MERGE`)** as the entity passes through each milestone.

| Architectural Dimension | Accumulating Snapshot Fact Table |
| :--- | :--- |
| **Grain** | **One row per lifecycle instance** (e.g., 1 row per `order_id`, `loan_application_id`, `support_ticket_id`, `claim_id`). |
| **Mutability** | **Mutable (`INSERT` on creation, `UPDATE` / `MERGE` as milestones complete)** until reaching a terminal state (`DELIVERED`, `CLOSED`, `REJECTED`). |
| **Signature Columns** | Multiple **Role-Playing Date/Timestamp Foreign Keys** (one per milestone: `submitted_at`, `approved_at`, `funded_at`), **Pre-Computed Milestone Lag Measures** (`submit_to_approve_hours`), and **Stage Counters** (`rework_count`). |
| **Why Not Just Query Raw Events?** | Calculating end-to-end funnel durations (`Stage 1 -> Stage 4`) on a raw event table requires expensive 4-way self-joins or `MAX(CASE WHEN stage = ...)` window scans across billions of rows on every BI dashboard refresh. Accumulating Snapshots pay that join cost **once at write time**. |

---

## 2. Production DDL Pattern: Mortgage / Loan Underwriting Funnel

```sql
-- ============================================================================
-- ACCUMULATING SNAPSHOT FACT: LOAN APPLICATION LIFECYCLE
-- Grain: One row per loan_application_id (updated via MERGE as stages complete)
-- ============================================================================
CREATE TABLE fct_loan_application_lifecycle (
    loan_application_id     VARCHAR(64) PRIMARY KEY,
    -- 1. Role-Playing Milestone Date Keys (FK -> dim_date; -1 sentinel if not yet reached!)
    started_date_sk         INT NOT NULL,          -- Partition key
    submitted_date_sk       INT NOT NULL DEFAULT -1,
    docs_verified_date_sk   INT NOT NULL DEFAULT -1,
    underwritten_date_sk    INT NOT NULL DEFAULT -1,
    approved_date_sk        INT NOT NULL DEFAULT -1,
    funded_date_sk          INT NOT NULL DEFAULT -1,
    terminal_date_sk        INT NOT NULL DEFAULT -1,
    -- 2. Exact UTC Milestone Timestamps (for sub-day hour/minute SLA precision)
    started_at_utc          TIMESTAMP NOT NULL,
    submitted_at_utc        TIMESTAMP,
    docs_verified_at_utc    TIMESTAMP,
    underwritten_at_utc     TIMESTAMP,
    approved_at_utc         TIMESTAMP,
    funded_at_utc           TIMESTAMP,
    rejected_or_withdrawn_at_utc TIMESTAMP,
    -- 3. Conformed Dimension Foreign Keys
    applicant_sk            BIGINT NOT NULL,
    loan_product_sk         INT NOT NULL,          -- 30YR_FIXED, FHA, HELOC
    underwriter_sk          BIGINT NOT NULL DEFAULT -1,
    acquisition_channel_sk  INT NOT NULL,
    -- 4. Current Lifecycle State
    current_stage_code      VARCHAR(32) NOT NULL,  -- STARTED, SUBMITTED, DOCS_VERIFIED, UNDERWRITING, APPROVED, FUNDED, REJECTED
    is_terminal_state       BOOLEAN NOT NULL,      -- TRUE when FUNDED, REJECTED, or WITHDRAWN (stops future MERGE scans!)
    -- 5. Pre-Computed Stage-to-Stage Lag Measures (Hours — Additive for AVG/SUM denominators!)
    start_to_submit_hours   DECIMAL(10,2),
    submit_to_docs_hours    DECIMAL(10,2),
    docs_to_underwrite_hours DECIMAL(10,2),
    underwrite_to_approve_hours DECIMAL(10,2),
    approve_to_fund_hours   DECIMAL(10,2),
    end_to_end_cycle_hours  DECIMAL(10,2),
    -- 6. Loop / Rework Counters & Financial Measures
    doc_resubmission_count  INT NOT NULL DEFAULT 0,-- Captures backward loops cleanly!
    requested_principal_usd DECIMAL(14,2) NOT NULL,
    approved_principal_usd  DECIMAL(14,2)
)
PARTITION BY (started_date_sk)
CLUSTER BY (is_terminal_state, current_stage_code);
```

---

## 3. Handling Real-World Workflow Edge Cases

Interviewers love poking holes in Accumulating Snapshots with messy operational edge cases. Here are the **three standard curveballs** and their exact fixes:

### Edge Case 1: Backward Loops & Rework (`UNDERWRITING -> DOCS_VERIFIED -> UNDERWRITING`)
* **The Problem:** An applicant submits blurry ID documents; the underwriter kicks the application back to `DOCS_PENDING` twice before approving it.
* **The Fix:**
  1. Decide explicitly on timestamp semantics: store **`first_docs_verified_at_utc`** (or `latest_docs_verified_at_utc`) consistently on the snapshot row.
  2. Add an additive **loop counter** (`doc_resubmission_count`) on the accumulating snapshot to measure rework friction.
  3. Keep the underlying **Transaction Fact** (`fct_loan_stage_events`) in Silver/Gold if compliance auditors need to see every single backward bounce timestamp.

### Edge Case 2: Skipped Milestones (`EXPRESS_APPROVAL` skips `DOCS_VERIFIED`)
* **The Problem:** Automated instant-approval loans jump directly from `SUBMITTED` to `APPROVED`, leaving `docs_verified_at_utc = NULL`.
* **The Fix:** Never let `NULL` in an intermediate milestone break end-to-end duration math! Compute `end_to_end_cycle_hours` directly from `approved_at_utc - submitted_at_utc` rather than summing individual step lags (`step1 + step2 + step3`), which would evaluate to `NULL` if `step2` is `NULL`!

### Edge Case 3: Bounding `MERGE` Cost on Multi-Year Tables
* **The Problem:** If `fct_loan_application_lifecycle` holds 5 years of applications (`500M rows`) and 50,000 applications update today, running an unbounded `MERGE` rewrites old partitions.
* **The Fix:**
  1. Partition by `started_date_sk` (immutable cohort date—never partition by `updated_at`, which moves rows across partitions!).
  2. Bound the `MERGE` target scan using **both** a cohort window AND `target.is_terminal_state = FALSE` (e.g., `WHERE target.started_date_sk >= 20260501 AND target.is_terminal_state = FALSE`), leveraging Parquet min/max pruning on `is_terminal_state` so closed historical cohorts are skipped in milliseconds!

---

## 4. Validation Query (Stage Conversion Funnel & Bottleneck Analysis)

```sql
-- Query: Q3 Loan Origination Funnel Conversion & Median/Avg Stage Bottlenecks
SELECT
    lp.product_name,
    COUNT(*)                                                                AS total_started,
    COUNT(submitted_at_utc)                                                 AS total_submitted,
    COUNT(approved_at_utc)                                                  AS total_approved,
    COUNT(funded_at_utc)                                                    AS total_funded,
    ROUND(COUNT(funded_at_utc) * 100.0 / NULLIF(COUNT(submitted_at_utc), 0), 2) AS submit_to_fund_pct,
    ROUND(AVG(submit_to_docs_hours), 1)                                     AS avg_hrs_submit_to_docs,
    ROUND(AVG(docs_to_underwrite_hours), 1)                                 AS avg_hrs_docs_to_uw,
    ROUND(AVG(underwrite_to_approve_hours), 1)                              AS avg_hrs_uw_to_approve,
    ROUND(AVG(end_to_end_cycle_hours), 1)                                   AS avg_total_cycle_hours
FROM fct_loan_application_lifecycle f
JOIN dim_date d ON f.started_date_sk = d.date_sk
JOIN dim_loan_product lp ON f.loan_product_sk = lp.loan_product_sk
WHERE d.calendar_quarter = '2026-Q3'
GROUP BY 1
ORDER BY 5 DESC;
```

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Why use `-1` (`Unknown / Not Yet Reached`) instead of `NULL` for unreached milestone foreign keys like `funded_date_sk`?"**
   * **Expected Insight:** If an analyst writes `SELECT d.calendar_month, COUNT(*) FROM fct_loan_application_lifecycle f JOIN dim_date d ON f.funded_date_sk = d.date_sk GROUP BY 1`, `NULL` foreign keys are **silently dropped** by `INNER JOIN`, hiding the exact count of in-flight/unfunded loans! With a `-1` row in `dim_date` (`date_sk = -1, calendar_month = 'Not Yet Reached'`), `INNER JOIN` preserves 100% of rows and surfaces in-flight backlog explicitly in BI charts.
2. **Top-Up 2: "When is a process NOT a good fit for an Accumulating Snapshot?"**
   * **Expected Insight:** When a process has **no defined endpoint or unbounded recurring states**—for example, a customer support chat thread with 500 back-and-forth messages, or a bank account's lifetime deposits/withdrawals. Unbounded processes belong in **Transaction Fact Tables** (for individual messages/transfers) paired with **Periodic Snapshot Fact Tables** (for daily/monthly state).

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Uses Accumulating Snapshots specifically for finite, milestone-driven workflows** and pairs them with Transaction Facts when audit history of backward loops is needed. | Uses an Accumulating Snapshot for unbounded event streams (like clickstream or recurring bank deposits). |
| **Partitions by immutable creation date (`started_date_sk`)** and filters `is_terminal_state = FALSE` during `MERGE` to minimize write amplification. | Partitions an Accumulating Snapshot by `last_updated_date`, causing rows to physically jump partitions on every milestone update. |
| **Maps unreached milestone date FKs to `-1` sentinel rows** in `dim_date` while keeping raw `TIMESTAMP` columns `NULL` until reached. | Leaves milestone date FKs `NULL`, causing `INNER JOIN` BI queries to silently drop all in-progress records. |