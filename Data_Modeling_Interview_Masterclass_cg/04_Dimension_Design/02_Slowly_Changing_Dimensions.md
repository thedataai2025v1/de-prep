# Lesson 4.2 — Slowly Changing Dimensions (SCDs)

## 1. The problem SCDs solve

Business attributes change. A customer moves city, an employee changes department, a product moves category, or a merchant changes risk tier. The warehouse must decide whether reports should show the current value, the value at the time of the event, or both.

Slowly Changing Dimension (SCD) techniques are patterns for managing those changes in dimensional models. “Slowly” is a historical name; the same principles can apply to rapidly changing attributes, although a classic Type 2 dimension may not be the right implementation for every high-frequency attribute.

### Learning outcomes
- Explain SCD Types 0, 1, 2, and 3, and recognize Types 4 and 6.
- Design validity intervals and current-row indicators.
- Join facts to the correct dimension version.
- Handle late-arriving facts, inferred members, and retroactive corrections.
- Discuss CDC, idempotency, and operational safeguards.

---

## 2. Choose history semantics before choosing an SCD type

Ask the business:
1. Should reports restate history using the latest attribute value?
2. Must reports preserve the value that was true when the event occurred?
3. Do analysts need both current and historical values?
4. Is the change legally or operationally auditable?
5. How often does the attribute change, and what is the expected row growth?

Example: A customer changes region from North to South on 1 July.
- A report of last quarter's sales **as they were understood at the time** should use North.
- A report that asks **how much revenue came from customers currently in South** may intentionally use South for all historical sales.
- Both are valid questions, but they are not the same metric.

The correct design follows the required interpretation, not a preference for a particular SCD number.

## 3. Type 0 — retain original value

Type 0 preserves the original attribute value and does not update it. It is appropriate for attributes that should remain fixed for the analytical purpose, such as an original acquisition channel, where the business explicitly defines it as immutable.

Do not use Type 0 merely because updates are inconvenient. Confirm the business rule and preserve source history if corrections may be needed.

## 4. Type 1 — overwrite

Type 1 overwrites the existing value. No historical version is retained in the dimension.

Example:

```sql
UPDATE dim_customer
SET preferred_language = 'en'
WHERE customer_bk = 'C-1042'
  AND is_current = TRUE;
```

Use when:
- The old value is incorrect and should not appear in reporting.
- History is not required for that attribute.
- A correction should restate historical analysis.

Advantages: simple, compact, easy for current-state reporting.

Risks: historical reports can change after a correction; audit requirements may need a separate change log.

## 5. Type 2 — create a new version

Type 2 preserves history by inserting a new dimension row when a tracked attribute changes. Facts reference the surrogate key for the version that was valid for the event.

Example rows:

| customer_sk | customer_bk | region | valid_from | valid_to | is_current |
|---:|---|---|---|---|---|
| 501 | C-1042 | North | 2024-01-01 | 2025-07-01 | false |
| 982 | C-1042 | South | 2025-07-01 | 9999-12-31 | true |

The date boundary convention must be consistent. A robust convention is a half-open interval: `valid_from <= event_time AND event_time < valid_to`. This avoids overlapping boundary timestamps.

### Typical Type 2 columns

```sql
CREATE TABLE dim_customer (
    customer_sk  BIGINT NOT NULL,
    customer_bk  VARCHAR(100) NOT NULL,
    region       VARCHAR(80),
    segment      VARCHAR(50),
    valid_from   TIMESTAMP NOT NULL,
    valid_to     TIMESTAMP NOT NULL,
    is_current   BOOLEAN NOT NULL,
    change_hash  VARCHAR(64),
    source_updated_at TIMESTAMP,
    load_ts      TIMESTAMP NOT NULL
);
```

A uniqueness rule should ensure at most one current row per business key. Also validate that validity intervals do not overlap.

### Load pattern (conceptual SQL)

The following illustrates the logic; syntax and transaction semantics differ by warehouse.

```sql
-- 1. Close the current version when tracked attributes changed.
UPDATE dim_customer d
SET valid_to = s.effective_at,
    is_current = FALSE
FROM staging_customer s
WHERE d.customer_bk = s.customer_bk
  AND d.is_current = TRUE
  AND (
       COALESCE(d.region, '') <> COALESCE(s.region, '')
    OR COALESCE(d.segment, '') <> COALESCE(s.segment, '')
  );

-- 2. Insert a new current version for new or changed keys.
INSERT INTO dim_customer (
    customer_sk, customer_bk, region, segment,
    valid_from, valid_to, is_current, load_ts
)
SELECT
    generate_surrogate_key(), s.customer_bk, s.region, s.segment,
    s.effective_at, TIMESTAMP '9999-12-31 00:00:00', TRUE, CURRENT_TIMESTAMP
FROM staging_customer s
LEFT JOIN dim_customer d
  ON d.customer_bk = s.customer_bk
 AND d.is_current = TRUE
WHERE d.customer_bk IS NULL;
```

This is illustrative, not a complete production merge. A production implementation must handle:
- New keys and changed keys in a deterministic order.
- Duplicate source events and out-of-order events.
- Null-safe attribute comparisons.
- Concurrent loads for the same business key.
- Reprocessing and idempotency.
- Surrogate key generation and transaction boundaries.
- Effective time versus ingestion time.

### Fact-to-dimension lookup

For a fact event at `event_ts`, resolve the dimension version with:

```sql
SELECT d.customer_sk
FROM dim_customer d
WHERE d.customer_bk = :customer_bk
  AND :event_ts >= d.valid_from
  AND :event_ts <  d.valid_to;
```

Then store that `customer_sk` in the fact. If the warehouse resolves the dimension at query time instead, ensure the temporal join is correct and performant.

## 6. Type 3 — retain limited previous value

Type 3 stores a limited amount of history in additional columns, for example `current_region` and `previous_region`.

| customer_bk | current_region | previous_region |
|---|---|---|
| C-1042 | South | North |

It can support a narrowly defined “current versus previous” comparison, but it does not preserve an unlimited timeline. Repeated changes overwrite the previous value.

Use it only when the business explicitly needs a small fixed number of states. Do not choose it as a substitute for a complete audit history.

## 7. Types 4 and 6 — useful variants

- **Type 4:** keep current attributes in a current dimension and historical versions in a separate history table. This can isolate current-state access from history, but adds join and governance complexity.
- **Type 6:** hybrid approach combining Type 1, Type 2, and Type 3 ideas. It may retain historical rows while also exposing current attributes on those rows. It is powerful but can confuse consumers unless semantics are documented carefully.

Interview answers should prioritize Types 1 and 2, then describe other types when a use case justifies them.

## 8. Late-arriving data

### Late-arriving dimension

A fact arrives before its dimension record. For example, a trip event references a newly registered driver whose profile has not yet landed.

Options:
1. Create an inferred/placeholder dimension row using the known business key.
2. Load the fact against a governed “not yet received” member and later restate the foreign key.
3. Delay fact publication if the business requires complete dimensional context.

An inferred member can later be enriched in place if no historical versioning is needed, or replaced/closed and versioned according to the effective-time policy.

### Late-arriving fact

A fact arrives after the dimension has changed. Do not automatically attach it to the current row. Resolve the dimension version using the fact's business event time and the dimension's effective interval.

If a fact is loaded for 15 June but arrives on 10 July, and the customer moved on 1 July, the fact should normally reference the pre-move version if the analysis is “customer region at event time.”

## 9. Retroactive corrections and out-of-order changes

Suppose the warehouse has:
- North valid from January to July.
- South valid from July onward.

Later, the source reports that the move actually happened on 15 June. This is not a simple “close current, insert new row” operation. The timeline must be corrected, and facts in the affected period may need their surrogate keys re-resolved.

A safe correction process:
1. Identify the business key and corrected effective timestamp.
2. Reconstruct the full valid-time timeline from authoritative events.
3. Detect gaps and overlaps.
4. Rebuild or split affected Type 2 intervals.
5. Re-key impacted facts or use a temporal lookup layer.
6. Reconcile counts and measures before publishing.
7. Record who/what caused the correction and when it was applied.

For regulated or audit-heavy domains, distinguish **valid time** (when the attribute was true in the business) from **system/transaction time** (when the warehouse learned or recorded it). This is a bitemporal concern.

## 10. CDC and production implementation

CDC streams often provide operation type, source key, source commit position, and source event timestamp. They do not always provide a trustworthy business-effective timestamp. Establish which timestamp controls the SCD timeline.

Production design checklist:
- Deduplicate by source event ID or log position.
- Preserve event ordering per business key where required.
- Use a deterministic attribute comparison or hash, with collision risk understood.
- Make retries idempotent.
- Handle deletes as explicit business events (soft-delete flag, end-date, or tombstone policy).
- Track source, ingestion, and effective timestamps separately.
- Validate one current row per business key.
- Validate non-overlapping validity ranges.
- Monitor late-arrival rate and re-key volume.
- Publish a clear contract for downstream consumers.

## 11. Worked example: subscription plan change

A subscriber moves from Basic to Premium on 2026-03-10. The business wants revenue and usage attributed to the plan in effect at the event time.

`dim_subscription` uses Type 2:
- `subscription_sk`
- `subscription_bk`
- `plan_name`
- `valid_from`
- `valid_to`
- `is_current`

The usage fact stores `subscription_sk` resolved using the usage event timestamp. If the plan change is effective at midnight, define the timezone and boundary convention. If events arrive late, resolve against the effective-time timeline, not the current plan.

A separate current-state view can expose the latest plan for operational reporting. That avoids forcing every consumer to interpret historical rows.

---

## 12. Interview conversation

**Interviewer:** A customer's city changes. How would you model it?

**Candidate:** I would first ask whether historical sales should remain associated with the city that was valid at transaction time. If yes, I would use Type 2 for the city attribute or customer profile version. If only current-state analysis matters, Type 1 may be sufficient.

**Interviewer:** We need to know both where the customer lived when they purchased and where they live now.

**Candidate:** I would retain Type 2 history and expose a current customer view. The fact's customer surrogate key resolves to the historical version, while the current view joins to the current dimension row using the business key. I would label the two interpretations clearly so analysts do not mix them.

**Interviewer:** A transaction from last month arrives today. The customer moved yesterday. Which key do you use?

**Candidate:** I use the transaction's event timestamp and the dimension's effective-time intervals. I do not use today's current row. I would also verify that the event timestamp is trustworthy and that the validity intervals have no gaps or overlaps.

**Interviewer:** What if the source sends a correction saying the move happened two weeks earlier?

**Candidate:** That is a retroactive correction. I would rebuild the affected validity timeline and identify facts whose dimension version changes. Depending on scale and platform, I would re-key those facts in a controlled backfill or use a temporal mapping layer. I would reconcile before publishing and preserve an audit record.

## 13. Interview questions and follow-ups

### Core questions
1. Explain SCD Type 1 and Type 2 with examples.
2. When would you use Type 0 or Type 3?
3. What is the purpose of `valid_from`, `valid_to`, and `is_current`?
4. How do facts join to Type 2 dimensions?
5. How do you handle a late-arriving dimension?
6. How do you handle a late-arriving fact?
7. What is an inferred member?
8. How do you detect changed attributes efficiently?

### Follow-up / “top-up” questions
- What if two updates for the same business key arrive out of order?
- What if a source sends duplicate CDC events?
- What if a correction changes the effective date of a historical event?
- How do you prevent overlapping validity intervals?
- Should a deleted customer dimension row be physically deleted?
- How do you handle a fact whose event timestamp is missing?
- What is the difference between event time, effective time, ingestion time, and processing time?
- How would you implement SCD Type 2 at billions of rows?
- Would you use a hash to detect changes? What are the limitations?
- How would you handle a dimension attribute that changes hundreds of times per day?

### Principal-level probes
- Define a bitemporal model for a regulated customer attribute.
- How would you make a Type 2 pipeline replayable and idempotent?
- How do you estimate and control fact re-keying cost?
- What is your strategy for concurrent updates to the same business key?
- How do you expose historical and current semantics safely to self-service analytics?

## 14. Practice exercise

A ride-sharing company receives driver profile updates from a mobile app and a compliance platform. A driver changes home city, vehicle class, and compliance status. Trips may arrive late by up to 72 hours. Compliance corrections may be retroactive.

Design:
1. Which attributes use Type 1, Type 2, or a separate history model?
2. Which timestamp defines validity for each attribute?
3. How do trips resolve the correct driver and vehicle versions?
4. How do you process duplicate and out-of-order CDC events?
5. How do you correct previously published facts after a retroactive compliance update?

**Evaluation rubric (10 points):**
- Correct business semantics and SCD choice: 2
- Validity interval and temporal join design: 2
- Late-arrival and inferred-member handling: 2
- Replay, idempotency, and data quality: 2
- Operational cost, auditability, and trade-offs: 2

## 15. Key takeaways

- SCD choice is a business semantics decision, not merely an ETL implementation detail.
- Type 1 overwrites; Type 2 preserves versions; Type 3 preserves limited prior state.
- Facts should resolve to the dimension version valid at event time when historical context matters.
- Late and retroactive data require timeline reconstruction and sometimes fact re-keying.
- Production SCD pipelines need ordering, idempotency, interval validation, lineage, and clear consumer contracts.
