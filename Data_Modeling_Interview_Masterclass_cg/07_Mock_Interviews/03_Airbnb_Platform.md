# Principal Data Engineer Mock Interview — Short-Term Rental Marketplace (Airbnb-like)

**Level:** Senior / Staff / Principal Data Engineer  
**Duration:** 90 minutes (plus 30–45 minutes for SQL exercise)  
**Case type:** End-to-end source-to-warehouse data modeling  

## How to use this case in an interview

Treat this as a 75–90 minute interview, not a schema memorization exercise. First state assumptions, then ask clarifying questions, then inventory business processes and source systems. Do not design a single universal wide table.

**Important:** The source schemas below are realistic, intentionally simplified teaching fixtures. They are not representations of private production schemas at the named companies. Source-system tables are usually normalized for transactional writes; warehouse facts and dimensions are designed for analytical access.

## 1. Interview prompt and success criteria

> A two-sided lodging marketplace needs metrics for search-to-book conversion, listing availability, occupancy, booking value, host payouts, cancellations, guest refunds, pricing, quality and supply health across local markets.

A strong Principal-level answer must demonstrate:
- ability to clarify ambiguous business language and establish metric contracts;
- understanding of operational source entities, keys, cardinalities and update semantics;
- grain-first modeling and separation of independent business processes;
- correct historical, event-time, financial and temporal semantics;
- robust ingestion, CDC, deduplication, late-arrival and replay strategy;
- scalable physical design, observability, governance and cross-team ownership.

## 2. Interview conversation: requirement discovery

Use this as a natural conversation, not a checklist recitation.

| Candidate asks | Interviewer response / working assumption | Design consequence |
|---|---|---|
| Who consumes the data? | Product, operations, finance, leadership and data science. | Separate operational freshness marts from certified financial reporting. |
| What decisions are made? | Funnel, operational performance, customer outcomes and financial reconciliation. | Model multiple business processes, not one fact. |
| What is the authoritative definition of the core entity? | The source domain owner owns the canonical identifier and lifecycle. | Preserve source IDs and define cross-system identity resolution. |
| Are source rows overwritten or append-only? | Both patterns exist. | Combine CDC current-state capture with immutable event history. |
| Can one parent have multiple children or attempts? | Yes; retries, reassignment, partial fulfillment and adjustments occur. | Explicit one-to-many / many-to-many tables; never assume one-to-one. |
| Do attributes change historically? | Yes, some dimensions and policies change. | SCD Type 2 or effective-dated reference data where past attribution matters. |
| What are freshness and availability expectations? | Operational aggregates in minutes; certified finance in daily close windows. | Distinct SLAs, pipelines and quality gates. |
| What is the expected scale? | High-volume events plus lower-volume entities. | Separate hot event facts, snapshots, aggregates and raw retention tiers. |
| What is the meaning of a reporting day? | Local business day for user-facing operations; UTC for event transport. | Store UTC instants plus business timezone/date derivations. |
| What corrections/deletions are required? | Late corrections, privacy deletion and financial reversals occur. | Versioned corrections, tombstones, lineage and policy-driven retention. |

**Candidate synthesis:** “I will model the core entities and lifecycle events independently, preserve source identifiers and event history, then publish conformed dimensions and facts at explicit grains. Before finalizing, I will validate every metric against its source-of-truth definition and reconciliation owner.”

## 3. Source-system inventory and source table structures

The following tables are the starting operational layer. For each source, identify: primary key, foreign keys, row grain, mutable columns, event timestamps, ingestion/update timestamps, deletion semantics and expected CDC behavior.

### `identity.users`
**Grain / purpose:** One row per platform user; user may act as guest and host.

```text
user_id PK | created_at | home_country | account_status | verification_level | updated_at
U10 | 2024-05-01 | IN | active | verified | 2026-11-14
U11 | 2024-07-02 | US | active | verified | 2026-11-13
```

### `catalog.listings`
**Grain / purpose:** One row per listing; mutable current listing attributes.

```text
listing_id PK | host_user_id FK | property_type | room_type | city_id | timezone | max_guests | current_status | created_at | updated_at
L100 | U11 | apartment | entire_home | BLR | Asia/Kolkata | 4 | active | 2025-01-01 | 2026-11-14
```

### `availability.calendar_days`
**Grain / purpose:** One row per listing × local date; may be regenerated from rules and overrides.

```text
listing_id FK | local_date | available_flag | nightly_price_minor | min_nights | blocked_reason | updated_at
L100 | 2026-12-01 | true | 650000 | 2 | NULL | 2026-11-14
L100 | 2026-12-02 | false | NULL | 2 | host_block | 2026-11-14
```

### `booking.reservations`
**Grain / purpose:** One row per reservation; mutable current reservation state.

```text
reservation_id PK | guest_id FK | listing_id FK | booked_at | checkin_local_date | checkout_local_date | guest_count | status | quoted_total_minor | currency | updated_at
B900 | U10 | L100 | 2026-11-14 10:00 | 2026-12-01 | 2026-12-04 | 2 | confirmed | 2100000 | INR | 2026-11-14 10:01
```

### `booking.reservation_changes`
**Grain / purpose:** One row per reservation change request/version.

```text
change_id PK | reservation_id FK | version_no | changed_at | old_checkin | old_checkout | new_checkin | new_checkout | old_total_minor | new_total_minor | change_status
CH1 | B900 | 2 | 2026-11-15 09:00 | 2026-12-01 | 2026-12-04 | 2026-12-02 | 2026-12-05 | 2100000 | 2250000 | accepted
```

### `payments.payment_transactions`
**Grain / purpose:** One row per payment attempt/transaction; multiple attempts per reservation possible.

```text
payment_id PK | reservation_id FK | payment_type | status | amount_minor | currency | provider_ref | created_at | settled_at
P1 | B900 | guest_charge | succeeded | 2100000 | INR | PSP1 | 2026-11-14 10:01 | 10:02
```

### `payments.host_payouts`
**Grain / purpose:** One row per host payout instruction/transfer.

```text
payout_id PK | host_id FK | reservation_id FK | amount_minor | currency | status | initiated_at | paid_at | provider_ref
PO1 | U11 | B900 | 1800000 | INR | paid | 2026-12-04 12:00 | 2026-12-05 09:00 | PSPP1
```

### `reviews.listing_reviews`
**Grain / purpose:** One row per submitted review.

```text
review_id PK | reservation_id FK | reviewer_id | reviewee_id | rating | submitted_at | category_scores
RV1 | B900 | U10 | U11 | 5 | 2026-12-05 10:00 | {cleanliness:5,accuracy:5}
```

### Source profiling questions the interviewer may ask

1. Which source is authoritative when two systems disagree?
2. Is the primary key globally unique or only unique within tenant/marketplace/country?
3. Can a natural key be reused after deletion or account closure?
4. Does `updated_at` change on every mutation, and is it reliable enough for incremental extraction?
5. Are deletes hard deletes, soft deletes, tombstones or status transitions?
6. Are timestamps UTC, local time or timezone-naive?
7. Is a child table append-only, mutable, or both?
8. What happens when the parent arrives after the child due to asynchronous ingestion?
9. Which fields are PII, payment-sensitive, location-sensitive or otherwise restricted?
10. What source-side constraints are real guarantees versus application conventions?

## 4. Source ER model and cardinality

```text
User (guest/host role), listing, listing-night availability, reservation, reservation change/version, reservation-night, payment transaction, refund, host payout, payout allocation, review, market/city, pricing rule.
```

State cardinality explicitly. A parent-to-child relationship is not a reason to flatten the child into the parent. For every relationship, discuss optionality, lifecycle, deletion behavior and whether the child can exist before the parent is visible in the warehouse.

## 5. Source-to-warehouse mapping

| Source object | Source grain | Target model | Transformation / modeling decision |
|---|---|---|---|
| `users` | One row per platform user | Silver `users` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `listings` | One row per listing | Silver `listings` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `calendar_days` | One row per listing × local date | Silver `calendar_days` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `reservations` | One row per reservation | Silver `reservations` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `reservation_changes` | One row per reservation change request/version. | Silver `reservation_changes` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `payment_transactions` | One row per payment attempt/transaction | Silver `payment_transactions` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `host_payouts` | One row per host payout instruction/transfer. | Silver `host_payouts` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `listing_reviews` | One row per submitted review. | Silver `listing_reviews` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |

Mapping rules:
- Preserve the source natural key and the source system identifier in the integration layer.
- Generate warehouse surrogate keys for dimensions when historical versions are needed.
- Do not silently drop orphan records; route them to an inferred/unknown member and reconcile later.
- Preserve raw payloads and source metadata in the bronze layer; do not treat a parsed current-state table as the only history.
- Keep header, line, event, snapshot and financial posting grains separate.

## 6. Target dimensional model

### Proposed facts

| Fact | Grain | Key attributes / measures |
|---|---|---|
| `fact_reservation` | One row per reservation ID/current canonical version; lifecycle accumulating snapshot. | reservation_id; guest/listing, booking timestamp, current stay dates, status, booked value and current outcome. |
| `fact_reservation_event` | One row per reservation state/change event. | event_id; reservation, event type, event timestamp, version, actor, old/new values. |
| `fact_listing_night_snapshot` | One row per listing × local stay date × snapshot time. | listing, local date, availability, nightly price, min nights; snapshot time explicit. |
| `fact_stay_night` | One row per occupied reservation × local night (derived after stay). | reservation, listing, guest, local date, realized occupancy, allocated nightly revenue. |
| `fact_payment_transaction` | One row per payment/refund transaction. | payment ID, reservation, transaction type, amount, currency, status, provider reference, settlement time. |
| `fact_host_payout` | One row per payout allocation. | payout ID, host, reservation, amount, fee/tax adjustments, payout date. |

### Conformed dimensions

dim_listing (Type 2 for property/room type, market classification); dim_user_role or role-playing guest/host user dimensions; dim_market/city with IANA timezone; dim_date; dim_cancellation_reason; dim_booking_channel; dim_currency; dim_review_category.

### Star-schema relationship sketch

```text
                dim_date (role-playing)
                      |
dim_actor ---- fact_core_process ---- dim_product_or_service
     |                |               |
dim_geo ------------- | -------------- dim_status
                      |
                dim_reason

Independent child/event/financial facts share conformed dimensions
but are not joined raw at incompatible grains.
```

The sketch is deliberately generic: use the case-specific fact list above to draw separate stars. A conformed dimension shares meaning and keys across facts; it does not mean all facts should be physically joined into one table.

## 7. Physical source DDL: representative operational schema

Below is an illustrative relational source definition for the main entity and one child table. It is not the warehouse DDL.

```sql
-- Operational source example; syntax is PostgreSQL-like.
CREATE TABLE core_entity (
    entity_id       VARCHAR(64) PRIMARY KEY,
    actor_id        VARCHAR(64) NOT NULL,
    business_status VARCHAR(32) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    source_version  BIGINT NOT NULL DEFAULT 1
);

CREATE TABLE core_entity_event (
    event_id        VARCHAR(64) PRIMARY KEY,
    entity_id       VARCHAR(64) NOT NULL,
    event_type      VARCHAR(64) NOT NULL,
    event_ts        TIMESTAMPTZ NOT NULL,
    source_sequence BIGINT,
    payload         JSONB,
    CONSTRAINT fk_event_entity
      FOREIGN KEY (entity_id) REFERENCES core_entity(entity_id)
);
```

Discuss why the OLTP schema may be normalized and why the warehouse should not blindly copy its table layout. At production scale, source constraints may not be enforced in every system; validate rather than assume.

## 8. Warehouse DDL: fact grain and dimensional keys

```sql
-- Illustrative analytical target. Adapt types, identity strategy and syntax
-- to Snowflake, Redshift, Databricks/Delta or the chosen warehouse.
CREATE TABLE dim_actor (
    actor_key       BIGINT NOT NULL,
    actor_id        VARCHAR(64) NOT NULL,
    segment         VARCHAR(64),
    region          VARCHAR(64),
    valid_from      TIMESTAMP NOT NULL,
    valid_to        TIMESTAMP NOT NULL,
    is_current      BOOLEAN NOT NULL,
    source_system   VARCHAR(64),
    record_hash     VARCHAR(128)
);

CREATE TABLE fact_core_process (
    process_key       BIGINT NOT NULL,
    process_id        VARCHAR(64) NOT NULL,
    actor_key         BIGINT,
    process_date_key  INTEGER,
    created_ts        TIMESTAMP NOT NULL,
    terminal_ts       TIMESTAMP,
    process_status    VARCHAR(32),
    source_updated_ts TIMESTAMP,
    warehouse_load_ts TIMESTAMP NOT NULL
);
```

**Do not copy this generic DDL as your answer.** Replace `actor`, `process` and the generic measures with the case entities, and state the grain in a sentence before adding columns. The key design is: one fact row corresponds to exactly one declared business event/process instance at that grain.

## 9. Transformation and loading strategy

### Layered architecture

```text
Operational DBs / APIs / Event producers
        |
        +--> CDC (snapshot + change log) ----+
        |                                    |
        +--> Domain event streams -----------+--> Bronze: immutable raw + metadata
        |                                    |
        +--> Periodic source snapshots ------+ 
                                             |
                                  Silver: typed, deduped, conformed
                                  - source key normalization
                                  - event ordering and version resolution
                                  - data quality and quarantine
                                  - identity and SCD processing
                                             |
                                  Gold: facts, dimensions, aggregates
                                  - certified metric definitions
                                  - finance/operations marts
                                  - access-controlled data products
```

### Required ingestion metadata

Every landed record should carry, where available:
`source_system`, `source_table_or_topic`, `source_primary_key`, `source_event_id`, `source_version`, `source_commit_ts`, `event_ts`, `ingested_at`, `operation_type` (insert/update/delete), `schema_version`, `kafka_topic`, `kafka_partition`, `kafka_offset`, `batch_id` and `payload_hash`.

Do not confuse Kafka offset with a business event ID. Offsets are transport coordinates scoped to a topic-partition. Use source event IDs, entity versions or deterministic business keys for idempotency.

### Incremental loading pattern

1. Bootstrap with a consistent snapshot or source-supported snapshot/CDC handoff.
2. Capture changes with transaction/commit metadata where supported.
3. Land raw changes append-only.
4. Deduplicate by source identity and version; quarantine conflicting duplicates.
5. Apply deletes as tombstones or governed deletion actions, not accidental disappearance.
6. Build current-state silver tables using deterministic latest-version logic.
7. Build event facts from immutable event sources; never synthesize historical events from a current row unless clearly labeled as inferred.
8. Process late arrivals with event-time lookback/replay windows.
9. Reconcile source counts, key counts, financial totals and state distributions.
10. Publish gold tables only after critical quality checks pass.

## 10. Worked data problem: from source rows to facts

Use the sample source records above. Explain, step by step:
- which source records represent separate business events;
- which IDs are natural keys and which are warehouse surrogate keys;
- which source rows are updates versus new facts;
- which attributes need Type 2 history;
- which measures can be summed and which cannot;
- what happens when a child row arrives before its parent;
- how corrections and reversals are represented;
- how the target fact rows are produced without fanout.

### SQL pattern: latest source state

```sql
WITH ranked AS (
  SELECT s.*,
         ROW_NUMBER() OVER (
           PARTITION BY source_system, source_pk
           ORDER BY source_version DESC, source_commit_ts DESC, ingested_at DESC
         ) AS rn
  FROM silver_source_changes s
)
SELECT *
FROM ranked
WHERE rn = 1
  AND operation_type <> 'DELETE';
```

This pattern is valid only if the ordering columns represent a trustworthy source version order. If source versions can tie or are absent, define a deterministic tie-breaker and document the limitation.

### SQL pattern: prevent fanout

```sql
WITH child_agg AS (
  SELECT parent_id, COUNT(*) AS child_count
  FROM fact_child_event
  GROUP BY parent_id
)
SELECT p.parent_id,
       p.parent_amount,
       COALESCE(c.child_count, 0) AS child_count
FROM fact_parent p
LEFT JOIN child_agg c
  ON p.parent_id = c.parent_id;
```

Never sum a parent-grain measure after joining to multiple child rows unless it has first been aggregated to a compatible grain.

## 11. Metric contracts and reconciliation

| Metric | Grain / numerator | Denominator / exclusions | Time basis | Owner / reconciliation |
|---|---|---|---|---|
| Primary volume metric | Distinct qualifying business IDs | Eligible population, not raw rows | Event/business date | Product or operations |
| Conversion | Distinct entities reaching defined outcome | Eligible entities entering funnel | Cohort or event date | Product analytics |
| Time-to-outcome | Outcome timestamp − start timestamp | Exclude invalid/negative durations; define pause policy | Elapsed or business time | Operations |
| Financial amount | Signed postings or allocated transaction amounts | Currency and status scope explicit | Posting/settlement date | Finance |
| Current balance/state | Latest valid snapshot at an as-of time | No summing across snapshot dates | Snapshot timestamp | Operations/finance |

For every metric, document: business definition, SQL owner, source-of-truth table, numerator, denominator, filters, timezone, currency, late-arrival policy, correction policy, refresh SLA and expected reconciliation tolerance.

## 12. Data quality, failure modes and recovery

| Failure mode | Detection | Handling |
|---|---|---|
| Duplicate delivery | Duplicate source event ID / same entity version | Idempotent merge; retain raw evidence |
| Out-of-order event | Sequence/time regression | Reorder by source sequence; recompute affected entity |
| Late-arriving dimension | Fact references unknown natural key | Inferred member, then resolve |
| Parent/child mismatch | Orphan foreign key | Quarantine or inferred parent; alert owner |
| Schema evolution | Registry/contract incompatibility | Compatibility gate, versioned parser, quarantine |
| Source delete | CDC tombstone or deletion request | Apply retention/privacy policy; preserve audit metadata where allowed |
| Backfill overlap | Duplicate keys across batch ranges | Idempotent deterministic merge |
| Metric drift | Source-to-gold reconciliation delta | Stop certification, identify changed definition or missing data |
| Partial pipeline failure | Checkpoint/manifest mismatch | Replay from durable offset/watermark; no partial publish |

## 13. Interviewer–candidate deep-dive dialogue

**Interviewer:** “Why not create one denormalized table with all attributes and child details?”

**Candidate:** “Because the source has independent grains and cardinalities. Flattening creates row multiplication, duplicates parent measures, increases update complexity and makes lifecycle history ambiguous. I would publish a core fact plus separate event/child facts, then expose curated views or aggregates for specific use cases.”

**Interviewer:** “What if the source has only a mutable current-state table and no event log?”

**Candidate:** “I would first ask whether CDC can capture row changes from the database log. If only periodic snapshots are available, I can infer observed state changes between snapshots, but I must label the history as sampled/inferred and cannot claim to know intermediate transitions.”

**Interviewer:** “A fact arrives before its dimension. Do you drop it?”

**Candidate:** “No. I preserve the fact and use an inferred dimension member keyed by the natural key, then enrich or remap when the full dimension arrives. The approach depends on whether the surrogate key can be updated safely and whether downstream tables have already consumed it.”

**Interviewer:** “How do you handle a late correction to a completed process?”

**Candidate:** “Keep raw change history, identify whether the correction changes descriptive state or an additive financial/event fact, and apply a versioned correction. For finance, use reversal/adjustment postings rather than overwriting the original movement. Recompute only impacted aggregates and expose both event time and posting time.”

**Interviewer:** “How would you prove the warehouse is correct?”

**Candidate:** “Define reconciliation at the same grain as the source: row counts, distinct business IDs, status distributions, amounts by currency and posting date, and selected entity-level traces. Use freshness, completeness, uniqueness, referential integrity and metric-level checks. Certification should be blocked on material unexplained variance.”

## 14. Principal-level follow-up questions

1. **How do you derive bookable nights when availability is represented by rules plus overrides rather than a fully materialized calendar?**
2. **Reservation check-in is Dec 1 and checkout Dec 4. How many occupied nights and which local dates?**
3. **How do you model a reservation date change while preserving original booking value and incremental price difference?**
4. **How do you prevent double booking under concurrent reservation attempts, and what does the analytics warehouse need to know?**
5. **How do you calculate occupancy when listings are temporarily paused, blocked by hosts or unavailable for regulatory reasons?**
6. **Guest payment succeeds but host payout fails. Which business metrics should change and which should not?**
7. **How do you handle cancellation policies that vary by listing, market, booking date and policy version?**
8. **How do you calculate host earnings when fees, taxes, currency conversion and payout timing differ?**
9. **How do you preserve local timezone semantics for properties across daylight-saving changes?**
10. **What data contracts would you establish between listing, availability, booking, payments and trust/safety domains?**

For each follow-up, answer in this order: clarify the requirement, identify the affected grain, state the design choice, explain an alternative, discuss failure modes, and define how you would validate it.

## 15. Common weak answers

- Drawing a star schema before understanding source entities and business processes.
- Listing columns without declaring grain and keys.
- Treating source `updated_at` as a complete event history.
- Joining independent facts directly and multiplying measures.
- Using one timestamp for event time, ingestion time, business date and posting date.
- Summing balances or distinct counts across time.
- Treating a status field as an immutable event log.
- Assuming primary/foreign-key constraints are enforced in every source.
- Omitting currency, timezone, deletion, retention and late-arrival policies.
- Giving a technology answer (“use Kafka/Spark”) without explaining semantics and recovery.

## 16. Candidate practice assignment

Complete these tasks on a whiteboard or in a Markdown file:

1. Draw the source ER diagram and label PK/FK/cardinality/optionality.
2. Identify all source grains and mutable versus append-only tables.
3. Write the target fact grain statement for each fact.
4. Draw separate star schemas and identify conformed dimensions.
5. Write DDL for the three most important target facts and dimensions.
6. Write two analytical SQL queries and one reconciliation query.
7. Explain CDC bootstrap, replay, deduplication, late data and delete handling.
8. Identify one metric that is easy to miscalculate and show how you prevent it.
9. Explain physical design for a selected platform (Snowflake, Redshift or Databricks).
10. Present a migration, ownership and adoption plan as the Principal Engineer.

### Scoring rubric (100 points)

| Area | Points | What excellent performance looks like |
|---|---:|---|
| Discovery and metric semantics | 10 | Asks targeted questions; resolves ambiguous definitions |
| Source model and lineage | 15 | Correct source grains, keys, cardinalities and mutation semantics |
| Fact/dimension design | 20 | Grain-first, conformed dimensions, history and no fanout |
| SQL and correctness | 10 | Correct joins, time logic, deduplication and reconciliation |
| Ingestion and CDC | 15 | Snapshot/CDC handoff, idempotency, replay, late data and deletes |
| Scale and performance | 10 | Workload-driven partitioning/clustering/aggregation and cost controls |
| Governance and quality | 10 | Privacy, lineage, contracts, ownership and SLOs |
| Principal leadership | 10 | Trade-offs, migration, stakeholder alignment and measurable adoption |

**Interpretation:** 85–100 = strong Principal-level signal; 70–84 = strong Senior/Staff with gaps to close; 55–69 = design fundamentals need strengthening; below 55 = revisit source modeling and grain fundamentals.

## 17. Further reading and validation

- Airbnb Engineering, [Scaling beyond one: evolving data architecture for a multi-product world](https://airbnb.tech/uncategorized/scaling-beyond-one-how-airbnb-evolved-its-data-architecture-for-a-multi-product-world/).
- AWS, [Dimensional modeling in Amazon Redshift](https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/).
- Kimball Group, [Slowly Changing Dimensions](https://www.kimballgroup.com/2008/08/slowly-changing-dimensions/).

Also validate warehouse-specific syntax and capabilities against the current vendor documentation. The SQL here is illustrative and intentionally not tied to one engine.

## 18. Case-specific relationship diagram

```text
users (guest/host roles)
  ├── hosts 1 ─── N listings 1 ─── N listing_calendar_days
  └── guests 1 ─── N reservations N ─── 1 listings
                         │
                         ├── 1 ─── N reservation_changes
                         ├── 1 ─── N reservation_nights (derived)
                         ├── 1 ─── N payment_transactions
                         ├── 1 ─── N refunds
                         ├── 1 ─── N payout_allocations N ─── 1 host_payouts
                         └── 0..N reviews
```

A user can be both host and guest. Use role-playing relationships or a shared user dimension with explicit role keys. Do not duplicate the person as two unrelated identities unless the source domain does so.

## 19. Case-specific target DDL

```sql
CREATE TABLE fact_listing_night_snapshot (
  listing_key BIGINT NOT NULL,
  local_date_key INTEGER NOT NULL,
  snapshot_ts TIMESTAMP NOT NULL,
  is_available BOOLEAN NOT NULL,
  nightly_price_minor BIGINT,
  currency_code CHAR(3),
  min_nights INTEGER,
  blocked_reason_key BIGINT
);

CREATE TABLE fact_stay_night (
  reservation_id VARCHAR(64) NOT NULL,
  listing_key BIGINT NOT NULL,
  guest_key BIGINT,
  local_stay_date_key INTEGER NOT NULL,
  occupied_night_count INTEGER NOT NULL DEFAULT 1,
  allocated_nightly_amount_minor BIGINT,
  currency_code CHAR(3)
);
```

## 20. Case-specific SQL exercises

**A. Expand a reservation into occupied local nights**

```sql
-- PostgreSQL-style illustration; checkout date is exclusive.
SELECT r.reservation_id,
       r.listing_id,
       d::date AS local_stay_date
FROM booking.reservations r
CROSS JOIN LATERAL generate_series(
  r.checkin_local_date::timestamp,
  (r.checkout_local_date - INTERVAL '1 day')::timestamp,
  INTERVAL '1 day'
) AS d
WHERE r.status IN ('confirmed', 'completed');
```

**B. Occupancy by market and stay date**

```sql
SELECT market_key, local_date_key,
       SUM(CASE WHEN is_occupied THEN 1 ELSE 0 END) AS occupied_listing_nights,
       SUM(CASE WHEN is_bookable THEN 1 ELSE 0 END) AS bookable_listing_nights,
       1.0 * SUM(CASE WHEN is_occupied THEN 1 ELSE 0 END)
         / NULLIF(SUM(CASE WHEN is_bookable THEN 1 ELSE 0 END), 0) AS occupancy_rate
FROM mart_listing_night
GROUP BY market_key, local_date_key;
```

The denominator must exclude dates/listings that are not eligible to be booked. Do not count every listing-calendar date as available supply.

**C. Booking value versus cash and payout**

```sql
SELECT reservation_id,
       SUM(CASE WHEN transaction_type = 'guest_charge' THEN amount_minor ELSE 0 END) AS guest_charged_minor,
       SUM(CASE WHEN transaction_type = 'refund' THEN amount_minor ELSE 0 END) AS refunded_minor
FROM fact_payment_transaction
GROUP BY reservation_id;
```

Host payout is a separate fact and should not be inferred as booking value minus a guessed fee.

## 21. Detailed source-to-target mapping matrix

| Source table | Source columns used | Target model | Transformation / semantic rule |
|---|---|---|---|
| `users` | user_id, created_at, country, status, verification | dim_user; role-playing guest/host keys | One identity can act in multiple marketplace roles. |
| `listings` | listing_id, host_user_id, property/room type, city, timezone | dim_listing (SCD2); dim_host; dim_market | Preserve historical listing classification and timezone. |
| `calendar_days` | listing_id, local_date, available, nightly price, min nights | fact_listing_night_snapshot | Grain includes snapshot time; materialized calendar may be derived from rules. |
| `reservations` | reservation_id, guest, listing, booked_at, stay dates, status, quote | fact_reservation; dim_guest/listing/date | Current reservation row is not sufficient for change history. |
| `reservation_changes` | change_id, reservation, version, old/new dates and totals | fact_reservation_event / reservation version history | Keep before/after values and accepted/rejected status. |
| `payment_transactions` | payment_id, reservation, type, status, amount, currency | fact_payment_transaction; dim_currency | Charge, refund and settlement states remain separate. |
| `host_payouts` | payout_id, host, reservation, amount, status, dates | fact_host_payout and payout allocation bridge | Payout can bundle or split multiple reservation balances. |
| `listing_reviews` | review_id, reservation, reviewer/reviewee, rating | fact_review; dim_review_category | One row per review; category scores may need child fact. |
