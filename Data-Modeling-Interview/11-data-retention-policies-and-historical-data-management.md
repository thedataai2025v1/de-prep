# Lesson 2.7 — Data Retention Policies & Historical Data Management

> **Where are we?** Module 2 (Gathering Business Requirements) → Lesson 2.7. This lesson bridges business compliance requirements with physical data-model design. It is one of the most frequently tested areas for senior data-modeling roles.

---

## 1. What Data Retention Means — And Why It Matters

**Data retention** is the policy governing *how long* data is kept, *where* it is stored over time, and *when/how* it is disposed of. It sits at the intersection of three business imperatives:

| Imperative | Why It Drives Retention Policy |
|---|---|
| **Compliance** | Laws (GDPR, HIPAA, SOX, CCPA) mandate minimum/maximum retention windows. Non-compliance can trigger fines up to 4% of global revenue. |
| **Cost** | Storing unnecessary data wastes cloud or on-prem storage. Tiered storage (hot → cold → archive) can reduce costs by **60–80%**. |
| **Performance** | Bloated tables degrade query speed. Archiving old data keeps "hot" tables lean, improving OLTP response times and BI dashboard latency. |

> **Key phrase for interviews:** *"Retention isn't just a legal requirement — it's a performance and cost optimization strategy."*

### What a Retention Policy Document Looks Like

A well-defined policy answers five questions:

```
1. WHAT data is covered?        (PII, transaction logs, user activity, etc.)
2. HOW LONG must we keep it?    (e.g., 7 years for financial records)
3. WHERE should it live?        (hot → warm → cold tiers)
4. HOW do we dispose of it?     (secure delete, anonymize, or encrypt-at-rest)
5. WHO authorizes exceptions?   (compliance officer, legal, data steward)
```

---

## 2. Retention Tiers: Hot, Warm, and Cold

Modern data platforms use a three-tier storage model mapped to access frequency and data age.

| Tier | Age Range | Access Frequency | Storage Medium | Typical Latency | Cost Multiplier |
|---|---|---|---|---|---|
| **Hot** | 0–3 months | Daily / high | SSD, in-memory, cloud DB | < 10 ms | 1.0x (baseline) |
| **Warm** | 3–24 months | Weekly / moderate | HDD, columnar store, data warehouse | 50–200 ms | 0.3x |
| **Cold (Archive)** | 2+ years | Rare / on-demand | Object storage (S3 Glacier), tape | Minutes–hours | 0.05x |

### Visual Model

```
       ┌─────────────────────────────────────────────┐
       │              DATA INGESTION PIPELINE         │
       └──────────────────┬──────────────────────────┘
                          │
                   ┌──────▼──────┐
                   │   HOT TIER   │  ← Active tables, current partitions
                   │ (SSD/OLTP)   │     Partition key = month/year
                   └──────┬──────┘
                          │  (after 3 months)
                   ┌──────▼──────┐
                   │   WARM TIER  │  ← Data warehouse, columnar store
                   │ (HDD/DW)     │     Pre-aggregations may be added
                   └──────┬──────┘
                          │  (after 24 months)
                   ┌──────▼──────┐
                   │  COLD TIER   │  ← Object store, Glacier
                   │ (Archive)    │     Point-in-time snapshots
                   └──────┬──────┘
                          │  (policy expiry)
                   ┌──────▼──────┐
                   │   DISPOSAL   │  ← Secure delete or anonymization
                   └─────────────┘
```

---

## 3. Legal & Compliance Drivers

Every jurisdiction imposes different rules. A senior data modeler must know the **top five regulatory frameworks**:

| Regulation | Jurisdiction | Key Retention Rules | Penalty for Violation |
|---|---|---|---|
| **GDPR** | EU / EEA | Right to be forgotten (Art. 17); no excessive retention | Up to €20M or 4% of global revenue |
| **HIPAA** | US (healthcare) | Minimum 6 years for audit logs; PHI retention set by state law | Up to $1.5M per violation category |
| **SOX** | US (public companies) | 7-year retention for financial records & audit trails | Criminal penalties + fines |
| **CCPA** | US (California) | Right to deletion; limits on data collection purpose | Up to $7,500 per intentional violation |
| **FINRA/SEC** | US (financial services) | 6–7 years for trade records, communications, transaction data | Fines, license revocation |

> **Interview tip:** You don't need to memorize every law. *Show you know how to find the right rules and translate them into data-model constraints.*

### How to Translate Regulation into Model Design

```
Regulatory Requirement      →    Data Model Action
───────────────────────────────────────────────────
"Keep for 7 years"           →    Partition by year; set archival policy
"Right to be forgotten"      →    Soft-delete flag + background purging job
"Audit trail required"       →    Append-only audit table with timestamp & actor
"Data must be encrypted"     →    Column-level encryption or TDE
"No longer than necessary"   →    TTL column; scheduled cleanup job
```

---

## 4. How Retention Affects Data Model Design

### 4a. Partitioning Aligned with Retention Windows

Partition tables along the axis that maps to your retention policy — usually **time**:

```sql
-- Example: Monthly partitioning for a financial transactions table
CREATE TABLE transactions (
    transaction_id   BIGINT PRIMARY KEY,
    account_id       BIGINT NOT NULL,
    amount           DECIMAL(15,2) NOT NULL,
    transaction_date DATE NOT NULL,
    ...
)
PARTITION BY RANGE (transaction_date) (
    PARTITION p202501 VALUES LESS THAN ('2025-02-01'),
    PARTITION p202502 VALUES LESS THAN ('2025-03-01'),
    -- ... continue monthly
);

-- Dropping old partitions is FAST (no row-by-row DELETE)
ALTER TABLE transactions DROP PARTITION p201803;
```

**Interview advantage:** Emphasize that *dropping a partition is O(1) — it just removes a metadata pointer, not individual rows.*

### 4b. Data Archival Strategies

| Strategy | How It Works | Best For |
|---|---|---|
| **Partition drop + export** | Export partition to cold storage, then DROP | Large OLTP tables |
| **ETL to data warehouse** | Periodically load into a columnar DW | Analytics workloads |
| **Snapshot to object store** | Periodic full or incremental snapshots (Parquet/ORC) | Compliance archives |
| **In-database tiering** | Cloud DBs (Snowflake, BigQuery) auto-tier | Managed cloud platforms |

### 4c. TTL (Time-to-Live) Policies

Some databases support automatic expiration:

```sql
-- Cassandra / DynamoDB-style TTL
ALTER TABLE sessions ADD TTL 86400;  -- expire after 24 hours

-- MongoDB TTL index
db.sessions.createIndex( { "lastAccessed": 1 }, { expireAfterSeconds: 86400 } )
```

### 4d. Deletion vs. Anonymization

```
┌────────────────────────────────────────────────────────────┐
│              WHEN TO DELETE vs. ANONYMIZE                  │
├──────────────────┬─────────────────────────────────────────┤
│ DELETE           │ • Data is truly obsolete               │
│                  │ • GDPR "right to be forgotten" applies  │
│                  │ • No legal hold or audit requirement    │
├──────────────────┼─────────────────────────────────────────┤
│ ANONYMIZE        │ • Need to retain for analytics/stats   │
│                  │ • Regulators require the data to exist  │
│                  │ • Remove PII, keep non-sensitive fields │
└──────────────────┴─────────────────────────────────────────┘
```

Common anonymization techniques:
- **Hashing** (one-way): `SHA256(user_email)` → retains uniqueness, hides identity
- **Masking**: `john@example.com` → `j***@example.com`
- **Generalization**: exact birthdate → birth year only
- **K-anonymity**: ensure at least *k* records share the same quasi-identifier values

---

## 5. Historical Data Management Strategies

### 5a. Full History vs. Latest State

| Approach | Storage | Use Case | Trade-off |
|---|---|---|---|
| **Latest state only** (overwrites) | Minimal | Caching, active sessions | No ability to reconstruct past |
| **Full append history** | Large | Audit trails, regulatory compliance | High storage cost |
| **Hybrid** (current + archive) | Medium | Most production systems | Requires partitioning strategy |

### 5b. Slowly Changing Dimensions (SCD)

SCDs are the standard technique for tracking attribute changes over time in dimensional models.

| SCD Type | How It Works | Storage Impact | When to Use |
|---|---|---|---|
| **Type 0** | Never change | Minimal | Fixed attributes (SSN, product code) |
| **Type 1** | Overwrite | Minimal | Non-audited corrections |
| **Type 2** | New row with start/end dates | **Highest** | Full history required (most common for compliance) |
| **Type 3** | Add previous-value column | Low | Track last-change only |

**SCD Type 2 Example (Customer Address History):**

```sql
CREATE TABLE dim_customer (
    customer_sk       BIGINT PRIMARY KEY,          -- surrogate key
    customer_nk       VARCHAR(36) NOT NULL,         -- natural key
    first_name        VARCHAR(50),
    last_name         VARCHAR(50),
    address_line_1    VARCHAR(100),
    city              VARCHAR(50),
    state_province    VARCHAR(50),
    postal_code       VARCHAR(20),
    effective_from    DATE NOT NULL,                -- history start
    effective_to      DATE DEFAULT '9999-12-31',    -- history end (9999 = current)
    is_current       BOOLEAN DEFAULT TRUE           -- quick lookup for "latest"
);

-- When address changes, old row gets effective_to, new row gets current dates
-- Querying history for a specific date:
SELECT * FROM dim_customer
WHERE customer_nk = 'C-12345'
  AND '2023-06-15' BETWEEN effective_from AND effective_to;
```

> **Interview ace move:** Mention that SCD Type 2 is the *only* type that satisfies both "what was the state at time T?" and regulatory "show us the change trail."

### 5c. Audit Tables for Regulatory Compliance

```sql
CREATE TABLE audit_trail (
    audit_id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    table_name       VARCHAR(64) NOT NULL,
    record_id        VARCHAR(64) NOT NULL,
    action           ENUM('INSERT','UPDATE','DELETE') NOT NULL,
    old_values       JSON,                          -- snapshot before change
    new_values       JSON,                          -- snapshot after change
    changed_by       VARCHAR(100) NOT NULL,         -- user / service account
    changed_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address       VARCHAR(45),
    reason_code      VARCHAR(20)                    -- e.g., 'CORRECTION', 'MERGE'
);

-- Immutable: no UPDATE or DELETE allowed on this table
-- Retention: typically 7+ years, never purged
```

---

## 6. Practical Examples by Domain

| Domain | Retention Window | Strategy | Key Constraint |
|---|---|---|---|
| **Financial Services** | 7 years minimum | Full append history; never delete; partition by year | SOX/FINRA audit trail; every transaction must be reconstructable |
| **Healthcare** | 6–10+ years | HIPAA-compliant encryption at rest & in transit; PHI anonymized for analytics | Consent tracking; right to access but not always right to delete |
| **E-commerce** | 2 years detailed; aggregate beyond | Detailed orders in hot/warm; roll up to monthly aggregates for cold | GDPR right to be forgotten; order details can be anonymized after warranty |
| **Social Media** | Selective (6–24 months) | Anonymize user-generated content after X years; retain analytics aggregates | CCPA/GDPR deletion requests; content removal ≠ deletion (may cache) |

### Domain-Specific Design Pattern: Financial Services

```
┌───────────────────────────────────────────────────────────┐
│              FINANCIAL SYSTEM — DATA RETENTION             │
├────────────┬──────────────────────────────────────────────┤
│ TABLE      │ RETENTION + MODELING DECISION                │
├────────────┼──────────────────────────────────────────────┤
│ transactions│ Monthly partitions; keep 7 years in DW;     │
│             │ archive to cold storage; NEVER delete       │
├────────────┼──────────────────────────────────────────────┤
│ accounts    │ SCD Type 2 for all attribute changes;       │
│ (dim)       │ 7-year history in analytics layer           │
├────────────┼──────────────────────────────────────────────┤
│ audit_log   │ Append-only; no deletes; 10-year retention; │
│             │ separate schema for isolation               │
├────────────┼──────────────────────────────────────────────┤
│ sessions    │ TTL = 24 hours; hard delete after expiry    │
└────────────┴──────────────────────────────────────────────┘
```

---

## Interview Questions You'll Face

1. **"We have a 2-billion-row transactions table. Query performance is suffering. How do you design a retention strategy without losing compliance?"**
   - *Strong answer:* Propose time-based partitioning (monthly), archival of partitions older than 24 months to a data warehouse, and a scheduled job to DROP old partitions after export. Mention that partition drops are metadata-only operations.

2. **"A customer requests their data be deleted under GDPR. The data is in a 5-year-old archived dataset. What do you do?"**
   - *Strong answer:* Explain that GDPR deletion rights *do* apply to archived data. Describe locating the customer's records across all tiers, securely deleting or anonymizing them, and updating any derived aggregates. Note the tension with SOX/financial retention — legal review is needed.

3. **"How would you model historical product price changes so analysts can see prices at any point in time?"**
   - *Strong answer:* SCD Type 2 on `dim_product` with `effective_from`/`effective_to` columns. Demonstrate the query pattern using `BETWEEN`. Mention Type 3 as a lighter alternative if only "previous vs. current" is needed.

4. **"Explain how data retention ties into your choice of database technology."**
   - *Strong answer:* Connect TTL features (Cassandra, MongoDB), native partitioning (PostgreSQL, MySQL), auto-tiering (Snowflake, BigQuery), and append-only constraints (DynamoDB with on-demand backup). Emphasize that the *policy* drives the *technology choice*, not the other way around.

---

## Questions YOU Should Ask the Interviewer

| Question | What It Signals |
|---|---|
| *"What data retention policies does the organization currently enforce?"* | You think about compliance before schema |
| *"Are there any upcoming regulatory changes affecting our data lifecycle?"* | Forward-thinking; risk-aware |
| *"How do we currently handle archival and data disposal?"* | You're already planning implementation |
| *"Do you have a data governance team that owns retention policy?"* | You understand organizational structure |
| *"Are there any 'never-delete' datasets we should be aware of?"* | You prioritize risk mitigation |

---

## Top-Up Questions (Senior-Level Probes)

1. **"How do you balance GDPR's 'right to be forgotten' against SOX's 7-year retention mandate for the same dataset?"**
   - *Pro tip:* The answer involves **anonymization** (not deletion) for the analytics copy, maintaining the original in a legally sealed archive accessible only to auditors, and documenting the legal basis for retention in each tier.

2. **"Walk me through how you'd design a time-travel query engine on top of SCD Type 2 data for a data warehouse."**
   - *Pro tip:* Discuss `BETWEEN effective_from AND effective_to` query patterns, materialized views for common time-snapshots, and the performance implications of large SCD tables. Mention Delta Lake / Apache Hudi as modern alternatives that provide native time-travel.

3. **"What's the impact of SCD Type 2 on data volume over 10 years, and how do you mitigate it?"**
   - *Pro tip:* Estimate: if 1% of customers change an address per year, after 10 years you have ~1.1x the original rows — manageable. But for high-churn tables, propose **partitioned SCD tables** or moving old history to a separate "history" table.

4. **"How would you implement a global data retention policy across a multi-region deployment with different local laws?"**
   - *Pro tip:* Design a **policy table** that maps `(data_category, region) → (retention_days, disposal_action)`. Use this as a lookup in your archival pipeline. Emphasize the tension between global consistency and local compliance.

---

## Practice Exercise

### Scenario: FinTech Transaction Platform

You're the lead data modeler for a fintech startup that processes **500,000 transactions/day** across 12 countries. The CEO wants to know: *"Can we delete data older than 2 years to save costs?"*

**Your tasks:**

1. **Identify the regulatory constraints** for at least three of the 12 countries.
2. **Design a partitioning strategy** that supports both performance and compliance.
3. **Propose an archival flow**: what data goes where, and when?
4. **Handle the GDPR deletion request** for a user who also has SOX-relevant data.
5. **Sketch the schema** for at least one table with a retention-aware design.

**Hint:** Your answer should demonstrate that you understand you *cannot* simply delete 2-year-old data in financial services — but you *can* move it to cheaper storage and keep it queryable by auditors.

<details>
<summary>🔑 Solution Outline (click to expand)</summary>

1. **Constraints:** EU (GDPR — right to be forgotten, 4% fine); US (SOX — 7-year financial retention, FINRA — 6-year communications); UK (FCA — 5–7 years for trade records).
2. **Partitioning:** Monthly partitions on `transaction_date`. Current 24 months in hot/warm tier; 2–7 years in cold archive; 7+ years flagged for legal-review deletion.
3. **Archival flow:** At T+24 months, export partition to Parquet on S3 Glacier; DROP from primary DB. Keep a metadata catalog mapping `(partition, archive_location)`.
4. **GDPR + SOX:** Do NOT delete. Anonymize PII in the analytics copy; keep original in a sealed, access-controlled archive. Document the legal basis for retention override.
5. **Schema:** See `transactions` table above, with added columns: `region_code`, `is_anonymized BOOLEAN DEFAULT FALSE`, `retention_expiry_date`.

</details>

---

## Key Takeaways

- **Retention policy is a first-class design constraint**, not an afterthought. Ask about it in every requirements-gathering session.
- **Partitioning is your best friend** for aligning physical storage with retention windows.
- **SCD Type 2 is the gold standard** for auditable history; know its trade-offs.
- **Anonymization is the compromise** when laws conflict (GDPR delete vs. SOX keep).
- **Always document the "why"** — your design decisions must be explainable to auditors and regulators.

> *"The best data models aren't just logically correct — they're legally defensible."*

---

### Related Lessons

- ⬅️ [Lesson 10: Data Volume and Scalability Considerations](./10-data-volume-and-scalability-considerations.md) — volume and retention go together
- ➡️ [Lesson 12: Example Business Requirements Gathering](./12-example-business-requirements-gathering.md) — requirements gathering walk-through
- ➡️ [Lesson 21: Slowly Changing Dimensions Types 1-4](./21-slowly-changing-dimensions-types-1-4.md) — SCD Type 2 vs. retention implications
