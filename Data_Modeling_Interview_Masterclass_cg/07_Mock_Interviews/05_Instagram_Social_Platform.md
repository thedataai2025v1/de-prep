# Principal Data Engineer Mock Interview — Social Media Platform (Instagram-like)

**Level:** Senior / Staff / Principal Data Engineer  
**Duration:** 90 minutes (plus 30–45 minutes for SQL exercise)  
**Case type:** End-to-end source-to-warehouse data modeling  

## How to use this case in an interview

Treat this as a 75–90 minute interview, not a schema memorization exercise. First state assumptions, then ask clarifying questions, then inventory business processes and source systems. Do not design a single universal wide table.

**Important:** The source schemas below are realistic, intentionally simplified teaching fixtures. They are not representations of private production schemas at the named companies. Source-system tables are usually normalized for transactional writes; warehouse facts and dimensions are designed for analytical access.

## 1. Interview prompt and success criteria

> A social platform needs analytics for content creation, impressions, reach, engagement, follows, retention, creator performance, feed ranking experiments, abuse/fraud and privacy deletion across very high-volume event streams.

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
**Grain / purpose:** One row per user; mutable profile and account state.

```text
user_id PK | created_at | country_code | account_type | account_status | privacy_setting | updated_at
U1 | 2024-01-01 | IN | creator | active | public | 2026-11-14
U2 | 2024-02-01 | US | personal | active | private | 2026-11-14
```

### `content.posts`
**Grain / purpose:** One row per post/reel; current post metadata.

```text
post_id PK | creator_id FK | content_type | created_at | visibility | audio_id | deleted_at | updated_at
P100 | U1 | reel | 2026-11-14 08:00 | public | AU8 | NULL | 08:00
```

### `content.media_assets`
**Grain / purpose:** One row per media asset; binary content stored separately.

```text
asset_id PK | post_id FK | media_type | object_uri | duration_ms | width | height | created_at
A1 | P100 | video | s3://private/obj1 | 30000 | 1080 | 1920 | 08:00
```

### `social.follow_edges`
**Grain / purpose:** Current edge table; updates on follow/unfollow.

```text
follower_id PK-part | followee_id PK-part | edge_status | created_at | updated_at
U2 | U1 | active | 2026-11-14 08:10 | 08:10
```

### `events.feed_impressions`
**Grain / purpose:** One row per delivered impression candidate/render; dedup semantics must be defined.

```text
impression_id PK | viewer_id | post_id | surface | event_ts | session_id | experiment_id | rank_position | delivered_flag
I1 | U2 | P100 | home_feed | 08:11:00 | S9 | EXP3 | 2 | true
```

### `events.engagement_events`
**Grain / purpose:** Append-only likes/comments/shares/saves and reversals.

```text
event_id PK | actor_id | post_id | event_type | event_ts | client_event_id | session_id | payload
EV1 | U2 | P100 | like | 08:12:00 | CE1 | S9 | {}
EV2 | U2 | P100 | unlike | 08:15:00 | CE2 | S9 | {}
```

### `events.video_playback`
**Grain / purpose:** One row per playback progress/heartbeat event; very high volume.

```text
playback_event_id PK | viewer_id | post_id | playback_session_id | event_type | event_ts | watched_ms | completion_pct
V1 | U2 | P100 | PS1 | start | 08:11:02 | 0 | 0
V2 | U2 | P100 | PS1 | progress | 08:11:12 | 10000 | 33.3
```

### `experiments.assignments`
**Grain / purpose:** One row per user × experiment assignment/version.

```text
assignment_id PK | user_id | experiment_id | variant | assigned_at | exposure_ts
XA1 | U2 | EXP3 | B | 2026-11-14 08:00 | 08:11
```

### `privacy.deletion_requests`
**Grain / purpose:** One row per subject deletion request and processing status.

```text
request_id PK | subject_id | request_type | requested_at | due_at | status | completed_at
DR1 | U2 | account_delete | 2026-11-14 12:00 | 2026-12-14 | processing | NULL
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
User, post, media asset, follow edge, impression, engagement event, playback session/event, comment, hashtag, creator, experiment/variant assignment, report/moderation event, deletion request.
```

State cardinality explicitly. A parent-to-child relationship is not a reason to flatten the child into the parent. For every relationship, discuss optionality, lifecycle, deletion behavior and whether the child can exist before the parent is visible in the warehouse.

## 5. Source-to-warehouse mapping

| Source object | Source grain | Target model | Transformation / modeling decision |
|---|---|---|---|
| `users` | One row per user | Silver `users` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `posts` | One row per post/reel | Silver `posts` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `media_assets` | One row per media asset | Silver `media_assets` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `follow_edges` | Current edge table | Silver `follow_edges` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `feed_impressions` | One row per delivered impression candidate/render | Silver `feed_impressions` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `engagement_events` | Append-only likes/comments/shares/saves and reversals. | Silver `engagement_events` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `video_playback` | One row per playback progress/heartbeat event | Silver `video_playback` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `assignments` | One row per user × experiment assignment/version. | Silver `assignments` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `deletion_requests` | One row per subject deletion request and processing status. | Silver `deletion_requests` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |

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
| `fact_post_creation` | One row per post creation/version where history is needed. | post, creator, content type, created time, visibility, media attributes. |
| `fact_impression` | One row per valid delivered impression event. | impression ID, viewer, post, surface, timestamp, rank, experiment; define dedup/window. |
| `fact_engagement_event` | One row per immutable engagement action/reversal. | event ID, actor, post, type, event time, client event ID; unlike is a separate event. |
| `fact_playback_session` | One row per viewer × post × playback session. | session ID, viewer, post, start/end, watched duration, completion; derive from heartbeat stream. |
| `fact_follow_event` | One row per follow/unfollow edge transition. | follower, followee, event type/time; current graph is a separate snapshot/bridge. |
| `fact_experiment_exposure` | One row per user × experiment × exposure. | assignment, variant, exposure time, eligible population and exposure definition. |

### Conformed dimensions

dim_user pseudonymous with privacy labels and effective-dated account/creator attributes; dim_post/content; dim_surface; dim_experiment/variant; dim_geo coarse-grained; dim_date/time; dim_audio/hashtag where licensed and useful.

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

1. **How do you define a valid impression, and how do you deduplicate retries without removing legitimate repeat views?**
2. **Why is reach non-additive across days, and what approximate distinct strategy would you use?**
3. **How do you compute watch time from start/progress/heartbeat/complete events with missing and duplicated heartbeats?**
4. **How do you represent follow/unfollow history and the current social graph without scanning the entire event history?**
5. **How do you handle celebrity skew where a single post generates an extreme concentration of events?**
6. **How do you attribute engagement to experiment variants when assignment and exposure occur at different times?**
7. **How do privacy deletion requests propagate to raw events, derived aggregates, feature stores, backups and downstream consumers?**
8. **How do you distinguish content deletion, account deactivation, privacy setting changes and legal retention holds?**
9. **How would you sample or aggregate high-volume telemetry while preserving model-training and audit needs?**
10. **What governance metadata, lineage and schema annotation capabilities would you standardize across product teams?**

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

- Meta Engineering, [Migrating Data Ingestion Systems at Meta Scale](https://engineering.fb.com/2026/05/12/data-infrastructure/migrating-data-ingestion-systems-at-meta-scale/).
- Meta Engineering, [How Meta understands data at scale](https://engineering.fb.com/2025/04/28/security/how-meta-understands-data-at-scale/).
- Meta Engineering, [Automating data removal](https://engineering.fb.com/2023/10/31/data-infrastructure/automating-data-removal/).
- Meta Engineering, [Data lineage at scale](https://engineering.fb.com/2025/01/22/security/how-meta-discovers-data-flows-via-lineage-at-scale/).

Also validate warehouse-specific syntax and capabilities against the current vendor documentation. The SQL here is illustrative and intentionally not tied to one engine.

## 18. Case-specific relationship diagram

```text
users 1 ─── N posts 1 ─── N media_assets
users N ─── N users (follow_edges; directed, temporal)
users 1 ─── N impressions N ─── 1 posts
users 1 ─── N engagement_events N ─── 1 posts
users 1 ─── N playback_events N ─── 1 posts
users 1 ─── N experiment_assignments
users 1 ─── N privacy_deletion_requests
```

The current follow graph is a derived state. The follow event fact is the history. Keep both only when there are consumers for both.

## 19. Case-specific target DDL

```sql
CREATE TABLE fact_impression (
  impression_id VARCHAR(64) NOT NULL,
  viewer_key BIGINT,
  post_key BIGINT NOT NULL,
  surface_key BIGINT,
  event_ts TIMESTAMP NOT NULL,
  session_id VARCHAR(64),
  experiment_key BIGINT,
  rank_position INTEGER,
  delivered_flag BOOLEAN NOT NULL,
  dedup_key VARCHAR(128) NOT NULL
);

CREATE TABLE fact_engagement_event (
  event_id VARCHAR(64) NOT NULL,
  actor_key BIGINT,
  post_key BIGINT NOT NULL,
  event_type VARCHAR(32) NOT NULL,
  event_ts TIMESTAMP NOT NULL,
  client_event_id VARCHAR(64),
  session_id VARCHAR(64),
  is_reversal BOOLEAN NOT NULL DEFAULT FALSE
);
```

## 20. Case-specific SQL exercises

**A. Engagement events by type**

```sql
SELECT DATE_TRUNC('day', event_ts) AS event_day,
       event_type,
       COUNT(DISTINCT event_id) AS event_count
FROM fact_engagement_event
GROUP BY 1, 2;
```

**B. Reach is a distinct count over the requested window**

```sql
SELECT post_key,
       COUNT(DISTINCT viewer_key) AS unique_reach
FROM fact_impression
WHERE delivered_flag = TRUE
  AND event_ts >= :window_start
  AND event_ts < :window_end
GROUP BY post_key;
```

Do not add daily reach values to produce weekly reach. Recompute distinct viewers over the full week or use an explicitly approximate mergeable sketch.

**C. Current follow graph from latest edge event**

```sql
WITH ranked AS (
  SELECT follower_id, followee_id, event_type, event_ts,
         ROW_NUMBER() OVER (
           PARTITION BY follower_id, followee_id
           ORDER BY event_ts DESC, event_id DESC
         ) AS rn
  FROM fact_follow_event
)
SELECT follower_id, followee_id
FROM ranked
WHERE rn = 1 AND event_type = 'follow';
```

The ordering must use a trustworthy sequence/version when timestamps can collide or events can be delayed.

## 21. Detailed source-to-target mapping matrix

| Source table | Source columns used | Target model | Transformation / semantic rule |
|---|---|---|---|
| `users` | user_id, account type/status, privacy setting | dim_user (privacy-aware SCD2) | PII and restricted attributes separated from analytics key. |
| `posts` | post_id, creator, type, visibility, created/deleted | dim_post; fact_post_creation | Deletion and visibility changes need history/events. |
| `media_assets` | asset_id, post, media type, URI, dimensions/duration | dim_media_asset or post media bridge | Store metadata; secure binary media outside warehouse. |
| `follow_edges` | follower, followee, status, timestamps | current bridge_social_graph; fact_follow_event | Current edge and event history are different products. |
| `feed_impressions` | impression_id, viewer, post, surface, time, rank, experiment | fact_impression; dim_surface/experiment | Define delivered/viewable and dedup semantics. |
| `engagement_events` | event_id, actor, post, type, time, client event | fact_engagement_event | Unlike/unshare/reversal are separate events. |
| `video_playback` | playback event, viewer, post, session, watched time | fact_playback_event; derived fact_playback_session | Deduplicate heartbeats and derive sessions with bounded inactivity logic. |
| `experiment_assignments` | assignment, user, experiment, variant, assignment/exposure | dim_experiment/variant; fact_experiment_exposure | Assignment does not imply exposure. |
| `deletion_requests` | request, subject, type, requested/due/completed | privacy control plane and deletion audit fact | Track propagation across raw, derived and downstream stores. |
