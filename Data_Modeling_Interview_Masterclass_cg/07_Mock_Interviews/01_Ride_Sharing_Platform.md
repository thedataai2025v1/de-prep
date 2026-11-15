# Principal Data Engineer Mock Interview — Ride-Sharing Marketplace

**Level:** Senior / Staff / Principal Data Engineer  
**Duration:** 90 minutes (plus 30–45 minutes for SQL exercise)  
**Case type:** End-to-end source-to-warehouse data modeling  

## How to use this case in an interview

Treat this as a 75–90 minute interview, not a schema memorization exercise. First state assumptions, then ask clarifying questions, then inventory business processes and source systems. Do not design a single universal wide table.

**Important:** The source schemas below are realistic, intentionally simplified teaching fixtures. They are not representations of private production schemas at the named companies. Source-system tables are usually normalized for transactional writes; warehouse facts and dimensions are designed for analytical access.

## 1. Interview prompt and success criteria

> A multi-country ride-hailing marketplace needs trusted metrics for request conversion, dispatch efficiency, trip completion, driver utilization, ETA accuracy, gross bookings, driver earnings, incentives and contribution margin.

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

### `rider_service.riders`
**Grain / purpose:** One row per rider; mutable profile master.

```text
rider_id PK | created_at | home_city_id | signup_channel | rider_status | country_code | updated_at
R1001 | 2026-01-03 10:00 | BLR | organic | active | IN | 2026-11-14 08:00
R1002 | 2026-02-10 11:30 | HYD | referral | active | IN | 2026-11-13 20:00
```

### `driver_service.drivers`
**Grain / purpose:** One row per driver; current operational attributes only.

```text
driver_id PK | onboarded_at | current_tier | current_status | home_city_id | updated_at
D501 | 2025-09-01 | gold | active | BLR | 2026-11-14 07:00
D502 | 2025-10-10 | silver | suspended | BLR | 2026-11-14 09:00
```

### `dispatch.ride_requests`
**Grain / purpose:** One row per request; mutable current state.

```text
request_id PK | rider_id FK | requested_at | pickup_lat | pickup_lon | pickup_zone_id | drop_zone_id | product_code | quoted_fare_minor | currency | status | cancelled_by | updated_at
Q9001 | R1001 | 2026-11-14 09:00:00 | 12.97 | 77.59 | Z12 | Z18 | economy | 18500 | INR | completed | NULL | 2026-11-14 09:42
Q9002 | R1002 | 2026-11-14 09:05:00 | 12.96 | 77.60 | Z12 | Z21 | premium | 32000 | INR | rider_cancelled | rider | 2026-11-14 09:08
```

### `dispatch.driver_offers`
**Grain / purpose:** One row per offer attempt; a request may fan out to many drivers and retry.

```text
offer_id PK | request_id FK | driver_id FK | attempt_no | offered_at | response_at | response_code | offer_distance_m
O1 | Q9001 | D501 | 1 | 09:00:03 | 09:00:08 | accepted | 450
O2 | Q9001 | D502 | 1 | 09:00:04 | 09:00:10 | rejected | 700
O3 | Q9002 | D501 | 1 | 09:05:03 | NULL | expired | 900
```

### `trip_service.trips`
**Grain / purpose:** One row per canonical vehicle trip; operational row can be updated as milestones occur.

```text
trip_id PK | request_id FK | assigned_driver_id | vehicle_id | accepted_at | arrived_at | pickup_at | dropoff_at | trip_status | distance_m | duration_sec | final_fare_minor | currency | updated_at
T7001 | Q9001 | D501 | V88 | 09:00:08 | 09:11:00 | 09:13:00 | 09:42:00 | completed | 12400 | 1740 | 21200 | INR | 09:43
```

### `trip_service.trip_events`
**Grain / purpose:** Append-only lifecycle events; repeated event types and corrections are possible.

```text
event_id PK | entity_type | entity_id | event_type | event_ts | source_sequence | actor_type | reason_code | payload_version
E1 | trip | T7001 | driver_arrived | 09:11:00 | 4 | driver | NULL | 1
E2 | trip | T7001 | pickup | 09:13:00 | 5 | driver | NULL | 1
E3 | trip | T7001 | dropoff | 09:42:00 | 6 | system | NULL | 1
```

### `finance.trip_ledger_entries`
**Grain / purpose:** One row per financial posting/adjustment; immutable postings preferred.

```text
ledger_entry_id PK | trip_id FK | component_code | amount_minor | currency | direction | posted_at | source_txn_id | reverses_entry_id
L1 | T7001 | rider_base_and_distance | 20000 | INR | credit | 09:43 | TX1 | NULL
L2 | T7001 | rider_tip | 1200 | INR | credit | 09:50 | TX2 | NULL
L3 | T7001 | platform_fee | 3000 | INR | debit | 09:44 | TX3 | NULL
```

### `driver_app.driver_supply_events`
**Grain / purpose:** One row per online/offline transition; intervals are derived.

```text
supply_event_id PK | driver_id FK | event_type | event_ts | zone_id | device_id
S1 | D501 | online | 08:30 | Z12 | DEV9
S2 | D501 | offline | 10:00 | Z12 | DEV9
```

### `telemetry.driver_location_pings`
**Grain / purpose:** High-volume telemetry; one row per ping, not per trip.

```text
ping_id PK | driver_id | device_id | ping_ts | lat | lon | accuracy_m | trip_id nullable
P1 | D501 | DEV9 | 09:12:10 | 12.971 | 77.591 | 8 | T7001
P2 | D501 | DEV9 | 09:12:20 | 12.972 | 77.592 | 7 | T7001
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
Rider, request, offer attempt, assignment, trip, trip event, driver, vehicle, driver-vehicle assignment, service zone, city, supply interval, ledger posting, location ping.
```

State cardinality explicitly. A parent-to-child relationship is not a reason to flatten the child into the parent. For every relationship, discuss optionality, lifecycle, deletion behavior and whether the child can exist before the parent is visible in the warehouse.

## 5. Source-to-warehouse mapping

| Source object | Source grain | Target model | Transformation / modeling decision |
|---|---|---|---|
| `riders` | One row per rider | Silver `riders` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `drivers` | One row per driver | Silver `drivers` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `ride_requests` | One row per request | Silver `ride_requests` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `driver_offers` | One row per offer attempt | Silver `driver_offers` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `trips` | One row per canonical vehicle trip | Silver `trips` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `trip_events` | Append-only lifecycle events | Silver `trip_events` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `trip_ledger_entries` | One row per financial posting/adjustment | Silver `trip_ledger_entries` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `driver_supply_events` | One row per online/offline transition | Silver `driver_supply_events` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `driver_location_pings` | High-volume telemetry | Silver `driver_location_pings` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |

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
| `fact_ride_request` | One row per ride request (accumulating snapshot). | request_id; request_ts, terminal_ts; rider, city, pickup/drop zone, product; status; quoted fare; matched/completed/cancelled flags. |
| `fact_driver_offer` | One row per request × driver × offer attempt. | offer_id; request, driver, offer_ts, response_ts; response_seconds; accepted/rejected/expired flags; dispatch rank. |
| `fact_trip` | One row per canonical vehicle trip. | trip_id; request, rider, driver, vehicle, zones, milestone timestamps, distance, duration, final fare; trip outcome. |
| `fact_trip_event` | One row per unique lifecycle event. | event_id; trip, event type, event time, actor, source sequence; no trip-level fare repeated. |
| `fact_trip_ledger` | One row per financial ledger posting. | ledger_entry_id; trip, component, signed amount, currency, posting timestamp, reversal reference. |
| `fact_driver_supply_interval` | One row per driver × continuous online interval × zone. | driver, zone, online_start, online_end, online_seconds; derive occupied seconds by overlap with trips. |
| `fact_driver_location_ping` | One row per raw ping; retained separately from core marts. | ping_id, driver, trip, ping_ts, geo, accuracy; restricted access and short retention. |

### Conformed dimensions

dim_rider (pseudonymous; Type 1 for current contact/profile attributes); dim_driver (Type 2 for tier/region attributes affecting historical reporting); dim_vehicle (Type 2 or effective-dated assignment bridge); dim_city (timezone, currency, country); dim_service_zone (zone hierarchy/geospatial cell); dim_ride_product; dim_cancellation_reason; dim_date and role-playing timestamps; dim_currency.

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

1. **How do you calculate request conversion if one request can create multiple offers and one trip can include multiple riders?**
2. **Driver accepts, then cancels before pickup; another driver is assigned. Is this one trip or two? What is the canonical trip identity?**
3. **How would you calculate driver utilization across overlapping online intervals, breaks, multiple concurrent app sessions and trip intervals?**
4. **How do you model pooled rides where one vehicle trip serves multiple rider requests? Would you introduce a trip-request bridge and what is its grain?**
5. **How do you reconstruct surge multiplier at quote time if pricing rules change later?**
6. **How do you reconcile trip fare, rider charge, driver earnings, platform fee, incentives, tolls, tips and refunds?**
7. **GPS events arrive at 10-second cadence for millions of drivers. What do you retain raw, aggregate, index/cluster and expire?**
8. **How would you model city-local business date around daylight-saving transitions and midnight-crossing trips?**
9. **How do you migrate from a mutable trips table to event-sourced trip lifecycle without double counting?**
10. **What operational SLOs, ownership boundaries and data contracts would you establish across dispatch, trip, finance and telemetry teams?**

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

- Uber Engineering, [Designing Schemaless, Uber’s scalable datastore using MySQL](https://www.uber.com/us/en/blog/schemaless-part-one-mysql-datastore/) — trip data can evolve through late-arriving notes/updates.
- Uber, [Fulfillment Platform re-architecture](https://www.uber.com/us/en/blog/fulfillment-platform-rearchitecture/).
- DataDriven, [Ride-sharing schema practice](https://datadriven.io/problems/ride_sharing_platform_schema).
- AWS, [Redshift table design best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html).

Also validate warehouse-specific syntax and capabilities against the current vendor documentation. The SQL here is illustrative and intentionally not tied to one engine.

## 18. Case-specific relationship diagram

```text
riders 1 ─── N ride_requests 1 ─── N driver_offers N ─── 1 drivers
                    │
                    └── 0..1 / 1 trip (confirm pooled/split product semantics)
                                  │
                                  ├── 1 ─── N trip_events
                                  ├── 1 ─── N trip_ledger_entries
                                  └── 1 ─── N trip_stops

drivers 1 ─── N driver_supply_events
drivers 1 ─── N driver_vehicle_assignments N ─── 1 vehicles
drivers 1 ─── N location_pings
cities 1 ─── N service_zones
```

The `ride_requests` to `trips` cardinality is a product decision. For a solo ride it is often 1:0..1; pooled rides can require a `trip_request_bridge` at one row per vehicle trip × rider request.

## 19. Case-specific target DDL

```sql
CREATE TABLE fact_driver_offer (
  offer_key BIGINT NOT NULL,
  offer_id VARCHAR(64) NOT NULL,
  request_id VARCHAR(64) NOT NULL,
  driver_key BIGINT,
  offer_attempt_no INTEGER NOT NULL,
  offered_ts TIMESTAMP NOT NULL,
  response_ts TIMESTAMP,
  response_code VARCHAR(32),
  response_seconds INTEGER,
  is_accepted BOOLEAN,
  is_expired BOOLEAN,
  source_event_id VARCHAR(64),
  warehouse_load_ts TIMESTAMP NOT NULL
);

CREATE TABLE fact_trip_ledger (
  ledger_entry_key BIGINT NOT NULL,
  ledger_entry_id VARCHAR(64) NOT NULL,
  trip_id VARCHAR(64) NOT NULL,
  component_code VARCHAR(64) NOT NULL,
  amount_minor BIGINT NOT NULL,
  currency_code CHAR(3) NOT NULL,
  direction VARCHAR(16) NOT NULL,
  posted_ts TIMESTAMP NOT NULL,
  reverses_entry_id VARCHAR(64),
  source_txn_id VARCHAR(64)
);
```

## 20. Case-specific SQL exercises

**A. Request funnel without offer fanout**

```sql
WITH request_outcome AS (
  SELECT request_id,
         MAX(CASE WHEN trip_status = 'completed' THEN 1 ELSE 0 END) AS completed_flag
  FROM fact_trip
  GROUP BY request_id
)
SELECT COUNT(*) AS requests,
       SUM(COALESCE(completed_flag, 0)) AS completed_requests,
       1.0 * SUM(COALESCE(completed_flag, 0)) / NULLIF(COUNT(*), 0) AS request_to_completion_rate
FROM fact_ride_request r
LEFT JOIN request_outcome t USING (request_id)
WHERE r.request_ts >= :start_ts AND r.request_ts < :end_ts;
```

**B. Offer acceptance by dispatch attempt**

```sql
SELECT DATE_TRUNC('day', offered_ts) AS offer_day,
       COUNT(*) AS offers,
       SUM(CASE WHEN response_code = 'accepted' THEN 1 ELSE 0 END) AS accepted_offers,
       1.0 * SUM(CASE WHEN response_code = 'accepted' THEN 1 ELSE 0 END)
           / NULLIF(COUNT(*), 0) AS offer_acceptance_rate
FROM fact_driver_offer
GROUP BY 1;
```

**C. Ledger net by component and currency**

```sql
SELECT component_code, currency_code,
       SUM(CASE WHEN direction = 'debit' THEN -amount_minor ELSE amount_minor END) AS signed_amount_minor
FROM fact_trip_ledger
GROUP BY component_code, currency_code;
```

Never aggregate across currencies without a governed FX conversion table and rate policy.

## 21. Detailed source-to-target mapping matrix

| Source table | Source columns used | Target model | Transformation / semantic rule |
|---|---|---|---|
| `riders` | rider_id, created_at, home_city_id, signup_channel | dim_rider; dim_city; signup date role | PII separated; current profile Type 1 unless historical segmentation is required. |
| `drivers` | driver_id, onboarded_at, current_tier, home_city_id | dim_driver (SCD2), dim_city | Current-only source cannot reconstruct old tiers without CDC/history. |
| `ride_requests` | request_id, rider_id, requested_at, pickup/drop zones, product, quote, status | fact_ride_request; dim_rider; dim_city/zone; dim_ride_product | Retain request grain and lifecycle version; quote is not final fare. |
| `driver_offers` | offer_id, request_id, driver_id, attempt_no, offered_at, response | fact_driver_offer; dim_driver | One row per offer attempt; do not collapse to one row per request. |
| `trips` | trip_id, request_id, assigned_driver_id, milestones, distance, final_fare | fact_trip; dim_driver; dim_vehicle; role-playing zone/date dims | Accumulating snapshot; use CDC or events to preserve milestone changes. |
| `trip_events` | event_id, trip_id, event_type, event_ts, sequence | fact_trip_event; dim_event_type; dim_actor | Append-only event fact; deduplicate by event identity/version. |
| `trip_ledger_entries` | ledger_entry_id, trip_id, component, signed amount, currency, reversal | fact_trip_ledger; dim_currency; dim_component | Immutable postings; reversal references original entry. |
| `driver_supply_events` | supply_event_id, driver_id, online/offline, event_ts, zone | fact_driver_supply_interval; dim_driver; dim_zone | Derive intervals from paired transitions; flag unmatched transitions. |
| `driver_location_pings` | ping_id, driver_id, ping_ts, coordinates, trip_id | restricted raw telemetry; aggregated geo/time fact | Separate retention/access domain; avoid joining raw pings to trip marts. |
