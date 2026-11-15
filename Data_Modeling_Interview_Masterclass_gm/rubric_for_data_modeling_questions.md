# Lesson 3: Principal vs. Senior Evaluation Rubric

> **Principal Mentor Note:** Understanding the grading rubric behind the curtain changes how you communicate. When hiring committees debrief on a Data Modeling loop, they score candidates across **five core competencies**. Knowing what separates a *Senior (L5)* bar from a *Staff/Principal (L6/L7)* bar allows you to hit the exact signals interviewers write down in their feedback packets.

---

## 1. The 5-Axis Interview Grading Rubric

| Competency | Needs Improvement (No Hire) | Senior Bar (L5 Hire) | Principal / Staff Bar (L6/L7 Strong Hire) |
| :--- | :--- | :--- | :--- |
| **1. Problem Framing & Requirements** | Jumps straight into table creation; misses key business edge cases (refunds, cancellations, timezones). | Asks good clarifying questions about metrics, entities, and volume before modeling. | Identifies **hidden business ambiguities** (e.g., gross vs. net revenue recognition, attribution windows), prioritizes by business value, and defines clear domain boundaries. |
| **2. Grain & Dimensional Rigor** | Mixes grains in a single table; confuses facts and dimensions; uses natural strings as FKs without thought. | Correctly separates facts and dimensions; defines clear grain; applies standard Star Schema patterns. | Selects optimal fact patterns (**Transaction vs. Periodic Snapshot vs. Accumulating Snapshot vs. Bridge**); enforces strict additivity rules; prevents fan-out/chasm traps. |
| **3. Historical & Temporal Modeling** | Overwrites mutable data (SCD Type 1 everywhere) or creates duplicate rows breaking `COUNT(*)`. | Implements SCD Type 2 accurately with effective timestamps and surrogate keys. | Masters **bi-temporal modeling** (`valid_time` vs. `transaction_time`), handles **late-arriving facts & dimensions** cleanly, and chooses between SCD Type 2, Type 6, and snapshot facts based on query ergonomics. |
| **4. Physical Storage & Scale Architecture** | Ignores partitioning, clustering, file sizes, or `MERGE` cost; designs models that trigger full-table scans. | Recommends basic date partitioning and understands columnar vs. row-oriented storage. | Connects logical schema to **physical layout** (Iceberg hidden partitioning, Z-ordering/Liquid Clustering, small-file compaction, skew mitigation, incremental `MERGE` strategies). |
| **5. Data Governance, Evolution & Ops** | Treats schema as static; ignores PII/GDPR, backfills, or downstream breaking changes. | Mentions data quality checks and basic schema evolution (adding nullable columns). | Designs for **zero-downtime evolution**, crypto-shredding/PII isolation for GDPR compliance, idempotent backfill pipelines, and semantic layer governance. |

---

## 2. Deep Dive: How Calibration Plays Out in Live Answers

### Calibration Example A: Handling Many-to-Many Relationships
* **Prompt:** *"In our hospital data warehouse, a single patient encounter (`encounter_id`) can have multiple diagnosis codes (`ICD-10`), and each diagnosis can have a weighting factor."*
* **Senior Answer:** *"I'll create a bridge table `bridge_encounter_diagnosis` between `fct_encounters` and `dim_diagnosis` with a `weighting_factor` column so we don't duplicate the encounter cost."*
* **Principal Answer:** *"We have a classic multivalued dimension problem. If we join `fct_encounters` directly to `dim_diagnosis`, `SUM(total_billed_amount)` will fan out and overcount revenue. I will introduce a **Bridge Table** (`bridge_encounter_diagnosis`) with a `weighting_factor` summing to `1.0` per encounter, plus a `primary_diagnosis_flag`. Even better, for executive dashboards that only care about the primary diagnosis and don't want analysts to forget the weighting factor—causing accidental double-counting—I'll also expose `primary_diagnosis_sk` directly on `fct_encounters` and provide a governed semantic view for weighted attribution."*

### Calibration Example B: Handling GDPR "Right to Be Forgotten" in an Immutable Lakehouse
* **Prompt:** *"A user requests full deletion under GDPR. How does your data model support this without rewriting 5 years of Parquet files?"*
* **Senior Answer:** *"We run a nightly `DELETE FROM` or `MERGE` job on all tables where `user_id = X`."*
* **Principal Answer:** *"Running row-level `DELETE` across 5 years of partitioned fact tables triggers massive copy-on-write/merge-on-read file rewrites and breaks immutable audit aggregates. Instead, we isolate all PII in a dedicated, mutable **PII Vault / Outrigger Dimension** (`dim_user_pii`) keyed by a non-reversible surrogate `user_sk` (or encrypt PII columns per-user using **crypto-shredding**, storing the user's encryption key in a KMS/key table). Fact tables only store the anonymized `user_sk`. When a GDPR deletion request arrives, we purge the single row in `dim_user_pii` (or destroy the user's encryption key). All historical fact metrics—like total rides in Berlin in 2023—remain intact and accurate, while the individual is permanently de-identified."*

---

## 3. Self-Audit Checklist Before Concluding Any Interview

Before saying *"I'm ready for questions"*, mentally run through these 6 checks:
1. **Grain Check:** Did I state the grain of every fact table explicitly?
2. **Fan-Out Check:** Can any join between my fact table and dimensions multiply fact measures unexpectedly?
3. **Time Check:** Did I distinguish between *event occurrence time* (`event_ts`) and *warehouse ingestion time* (`ingested_at`), and standardize on UTC?
4. **History Check:** If an entity attribute changes tomorrow (e.g., customer changes plan, product changes category), does my model preserve historical truth?
5. **Scale Check:** Which table is the largest, how many rows/day does it grow by, and how are queries pruned so they don't scan the entire table?
6. **Failure/Ops Check:** If yesterday's pipeline fails and re-runs twice today, will my tables contain duplicate rows?

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "What is the single most common reason a strong Senior engineer gets down-leveled in a Principal data modeling interview?"**
   * **Answer:** **Over-indexing on logical elegance while ignoring operational and economic reality.** For example, designing a pristine 18-table Data Vault or pure 3NF model for a 40-person analytics team on Snowflake, or recommending SCD Type 2 on a 2-billion-row high-churn dimension without calculating the `MERGE` compute cost and join explosion.
2. **Top-Up 2: "How much SQL should I write during a 45-minute data modeling interview?"**
   * **Answer:** Write **just enough SQL to prove your design**—typically 1–2 validation queries (15–25 lines total) covering your trickiest metric (e.g., cohort retention, SCD Type 2 point-in-time join, or funnel conversion). Spend 75% of your time on requirements, grain, entity relationships, physical layout, and edge cases.