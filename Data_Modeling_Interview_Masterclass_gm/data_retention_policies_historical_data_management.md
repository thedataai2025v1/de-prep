# Lesson 11: Data Retention Policies, GDPR/Compliance & Historical Data Management

> **Principal Mentor Note:** Senior engineers focus on loading data *into* the warehouse; Principal engineers equally design how data **ages, rests, gets corrected, and gets legally purged**. In regulated domains (Fintech, Healthcare, AdTech, Consumer Apps), ignoring retention tiers, bi-temporal auditability, or GDPR/CCPA erasure can expose the company to multi-million-dollar fines or force painful multi-month table rewrites.

---

## 1. Multi-Tier Data Lifecycle & Retention Architecture

Not all layers of your data platform should have the same retention policy. Align retention with **business value, legal mandate, and storage/compute cost**:

| Data Layer | Typical Retention | Storage Tier & Format | Purpose & Governance Rule |
| :--- | :--- | :--- | :--- |
| **Bronze (Raw Event / CDC Archive)** | `90 days – 7 years` (Audit dependent) | S3/GCS/ADLS Standard $\rightarrow$ **Glacier / Archive** after 90d (Parquet/Avro) | Disaster recovery, deterministic replay, legal audit trail. Compressed with ZSTD; zero interactive BI queries. |
| **Silver (Canonical Cleaned Layer)** | `1 – 3 years` | Iceberg / Delta Lake (Warm Object Storage) | Rebuilding Gold marts when business definitions change without re-parsing raw JSON. |
| **Gold Atomic Facts (`fct_*`)** | `2 – 5 years` (rolling hot window) | Iceberg / Snowflake / BigQuery (Active Storage) | Granular analytical slicing, cohort analysis, ML model training. |
| **Gold Rollups & Financial Ledgers** | `7 – 10+ years` (Statutory compliance) | Active Warehouse Storage (Tiny footprint) | Multi-year YoY executive trends, tax/SOX/GAAP audit compliance (`agg_*`, `fct_ledger_*`). |

---

## 2. Historical Data Management: Bi-Temporal Modeling

In financial, insurance, healthcare, and enterprise SaaS systems, single-timeline SCD Type 2 (`valid_from`, `valid_to`) is often **not enough**. You must distinguish **two independent timelines**:

1. **Business / Valid Time (`valid_from`, `valid_to`):** When a fact or attribute was **true in the real world** (e.g., *"Customer's new pricing contract took effect retroactively on January 1st"*).
2. **System / Transaction Time (`recorded_from`, `recorded_to`):** When the data warehouse **learned about and recorded** that version (e.g., *"Sales Ops entered the signed contract into Salesforce on February 10th"*).

### Why Bi-Temporal Modeling Matters in Interviews
Suppose on **Feb 1**, Finance closed January's revenue report showing Customer X on the `Basic` tier (`$1,000`). On **Feb 10**, Sales Ops backdates Customer X's upgrade to `Enterprise` (`$5,000`) effective **Jan 1**.
* **Single-Timeline SCD Type 2:** Overwrites the Jan 1 row. Now if Finance re-runs the Jan report on Feb 15, the numbers no longer match what was submitted to auditors on Feb 1!
* **Bi-Temporal Table:**
  * Row 1: `tier = 'Basic'`, `valid = [Jan 1, ∞)`, `recorded = [Jan 1, Feb 10)`
  * Row 2: `tier = 'Basic'`, `valid = [Jan 1, Jan 1)` *(voided)* / `tier = 'Enterprise'`, `valid = [Jan 1, ∞)`, `recorded = [Feb 10, ∞)`
  * Querying `WHERE valid_time = 'Jan 15' AND recorded_time = 'Feb 01'` reproduces the **exact audit report** submitted on Feb 1, while `WHERE valid_time = 'Jan 15' AND recorded_time = CURRENT_TIMESTAMP` gives the **restated operational truth**!

---

## 3. GDPR / CCPA "Right to Be Forgotten" & PII Isolation Patterns

Deleting one user's personal data across 50 billion immutable Parquet rows is an operational nightmare if PII is scattered across fact tables. Use one of **three Principal architectural patterns**:

### Pattern 1: PII Vault / Outrigger Isolation (Recommended Default)
* Strip all PII (`full_name`, `email`, `phone`, `ip_address`, `exact_lat_lon`, `street_address`) out of all fact tables and core analytical dimensions.
* Store PII exclusively in a tightly access-controlled **PII Vault table** (`dim_user_pii`) keyed by a non-reversible surrogate key (`user_sk = UUID()` or salted HMAC).
* Core fact tables (`fct_trips`, `fct_orders`) only contain `user_sk` and non-PII analytical attributes (`city`, `country`, `signup_cohort_month`, `user_tier`).
* **On GDPR Erasure Request:** Delete/nullify the single row in `dim_user_pii` (and Bronze raw logs if within retention, or rely on Pattern 2 below). All historical trip counts and revenue totals remain 100% intact and anonymized!

### Pattern 2: Crypto-Shredding (Best for Immutable Bronze/Kafka Archives)
* Generate a unique per-user symmetric encryption key (`user_dek`) stored in a dedicated **Key Management Table / KMS** (`user_encryption_keys(user_id, dek_ciphertext, status)`).
* Encrypt every PII field in Kafka/Bronze/Silver using that user's `user_dek` before writing to Parquet/Iceberg.
* **On GDPR Erasure Request:** Delete `user_dek` from the KMS table. All historical PII fields across 5 years of immutable S3 backups and Iceberg snapshots instantly become **cryptographically unrecoverable ciphertext**, with zero file rewrites!

### Pattern 3: Partition-Aligned Erasure & Vacuuming (Iceberg / Delta)
* Remember that running `DELETE FROM fct_events WHERE user_id = '123'` in Apache Iceberg or Delta Lake creates a new snapshot marking the row deleted, **but the old Parquet file containing the PII still sits on S3** until:
  1. `rewrite_data_files` compacts the file physically without the deleted row, AND
  2. `expire_snapshots` + `remove_orphan_files` purges the old Parquet files past the time-travel retention window (e.g., 7 days).
* *Mentioning snapshot expiration during a GDPR discussion is a massive Principal-level signal!*

---

## 4. Real Interview Questions & Model Answers

### Q1: "We are designing a data warehouse for a European fintech company. Regulators require us to retain 7 years of immutable transaction history for anti-money laundering (AML) audits, while GDPR requires us to erase a customer's personal data within 30 days of an erasure request. How do you resolve this direct conflict in your data model?"

**Principal-Level Answer:**
> *"This is a classic regulatory intersection between **AML/Financial Record-Keeping** (which legally supersedes GDPR erasure for core financial ledger facts) and **GDPR Data Minimization** (which requires purging non-essential PII and de-identifying behavioral telemetry).*
>
> *I resolve this structurally using **Domain Separation + Crypto-Shredding**:*
> 1. ***Financial Ledger Separation:** Model `fct_ledger_transactions` with immutable monetary amounts, timestamps, merchant IDs, and a pseudonymous `account_sk`. Under AML regulations, these financial records are retained for 7 years in an append-only ledger.*
> 2. ***PII Vault & Crypto-Shredding:** Isolate all personal identifiers (`name`, `email`, `address`, `gov_id_hash`, `device_fingerprint`) in a dedicated `dim_customer_pii` table encrypted with a per-customer Data Encryption Key (`customer_dek`) managed in KMS.*
> 3. ***Erasure Workflow:** When a GDPR deletion request arrives, we check if the customer is under an active AML legal hold. If not, we purge their non-essential behavioral clickstream rows, delete their row in `dim_customer_pii`, and **destroy their `customer_dek`** in KMS—rendering any historical PII across 7 years of Bronze/Silver archives permanently unreadable while preserving 100% of aggregate financial auditability."*

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "When you purge old granular fact data after 2 years (e.g., `fct_clickstream`), how do you prevent downstream dashboards from suddenly losing historical baselines?"**
   * **Expected Insight:** Before running TTL partition drops (`ALTER TABLE DROP PARTITION`), ensure **lifetime rollup tables** (`agg_user_monthly_engagement`, `agg_product_daily_kpis`) are populated and verified. Also stamp **lifetime milestone attributes** (`first_order_date`, `first_active_channel`, `lifetime_order_count_pre_cutoff`) onto `dim_user` so cohort retention queries don't break when atomic events older than 2 years are purged.
2. **Top-Up 2: "How do you handle a full historical backfill (e.g., fixing a bug in `net_revenue` calculation over the past 18 months) without breaking production BI queries during the rebuild?"**
   * **Expected Insight:** Use the **Write-Audit-Publish (WAP)** pattern (natively supported via **Apache Iceberg Branches** or blue/green table swapping): run the backfill into an isolated staging branch (`audit_branch`), execute automated row-count, null, and financial reconciliation checks against the old partition totals, and atomically fast-forward `main` only after all assertions pass.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Separates PII from analytical facts** using a PII Vault or Crypto-Shredding. | Sprinkles `email`, `full_name`, and `ip_address` directly onto every fact table. |
| **Knows that Iceberg/Delta `DELETE` leaves PII in old snapshots** until compaction + snapshot expiration run. | Assumes a SQL `DELETE` statement immediately erases bytes from cloud object storage. |
| **Preserves cohort/lifetime attributes on dimensions** before expiring atomic event partitions. | Drops partitions older than 1 year and accidentally erases `user_signup_date` for all existing customers. |