# Principal Data Engineer Mock Interview — Customer Support / Contact Centre

**Level:** Senior / Staff / Principal Data Engineer  
**Duration:** 90 minutes (plus 30–45 minutes for SQL exercise)  
**Case type:** End-to-end source-to-warehouse data modeling  

## How to use this case in an interview

Treat this as a 75–90 minute interview, not a schema memorization exercise. First state assumptions, then ask clarifying questions, then inventory business processes and source systems. Do not design a single universal wide table.

**Important:** The source schemas below are realistic, intentionally simplified teaching fixtures. They are not representations of private production schemas at the named companies. Source-system tables are usually normalized for transactional writes; warehouse facts and dimensions are designed for analytical access.

## 1. Interview prompt and success criteria

> A SaaS company wants analytics for ticket volume, first response, resolution time, SLA compliance, agent productivity, channel mix, backlog, reopen rate, bot containment, CSAT and customer support cost.

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

### `crm.customers`
**Grain / purpose:** One row per customer account; mutable CRM master.

```text
customer_id PK | account_id FK | created_at | segment | region | plan | status | updated_at
C100 | A10 | 2025-01-01 | enterprise | APAC | premium | active | 2026-11-14
C101 | A11 | 2025-04-02 | SMB | EMEA | basic | active | 2026-11-13
```

### `support.tickets`
**Grain / purpose:** One row per ticket; current state is overwritten.

```text
ticket_id PK | customer_id FK | created_at | channel | initial_priority | current_priority | current_status | current_assignee_id | category_id | closed_at | updated_at
TK1 | C100 | 2026-11-14 08:00 | email | high | high | closed | AG7 | BILLING | 2026-11-14 10:30 | 10:31
TK2 | C101 | 2026-11-14 08:05 | chat | low | medium | open | AG8 | LOGIN | NULL | 09:00
```

### `support.conversations`
**Grain / purpose:** One row per conversation/session; a ticket may have multiple conversations.

```text
conversation_id PK | ticket_id FK | channel | started_at | ended_at | bot_session_id | language
CV1 | TK1 | email | 08:00 | 10:30 | NULL | en
CV2 | TK2 | chat | 08:05 | 08:25 | BOT9 | en
```

### `support.messages`
**Grain / purpose:** One row per message; bot and human messages share the table.

```text
message_id PK | conversation_id FK | sender_type | sender_id | sent_at | message_type | body_ref | is_public
M1 | CV1 | customer | C100 | 08:00 | text | OBJ1 | true
M2 | CV1 | agent | AG7 | 08:18 | text | OBJ2 | true
M3 | CV2 | bot | BOT9 | 08:06 | prompt | OBJ3 | true
```

### `support.ticket_assignments`
**Grain / purpose:** One row per assignment interval; reassignment creates a new row.

```text
assignment_id PK | ticket_id FK | agent_id | team_id | assigned_at | unassigned_at | assignment_reason
AS1 | TK1 | AG6 | BILLING | 08:01 | 08:15 | initial
AS2 | TK1 | AG7 | BILLING | 08:15 | 10:31 | escalation
```

### `support.ticket_events`
**Grain / purpose:** Append-only ticket state transitions.

```text
event_id PK | ticket_id FK | event_type | event_ts | actor_type | actor_id | old_value | new_value | source_sequence
TE1 | TK1 | created | 08:00 | customer | C100 | NULL | open | 1
TE2 | TK1 | priority_changed | 08:10 | system | SYS | low | high | 2
TE3 | TK1 | reopened | 11:00 | customer | C100 | closed | open | 3
```

### `support.sla_policy_versions`
**Grain / purpose:** One row per policy version and effective interval.

```text
policy_version_id PK | plan | priority | channel | calendar_id | first_response_target_sec | resolution_target_sec | valid_from | valid_to
PV1 | premium | high | email | APAC_BIZ | 900 | 14400 | 2026-01-01 | 9999-12-31
```

### `support.csat_responses`
**Grain / purpose:** One row per survey response; multiple responses may exist per ticket.

```text
response_id PK | ticket_id FK | submitted_at | score | survey_version | respondent_type
CS1 | TK1 | 2026-11-14 10:40 | 5 | v2 | customer
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
Customer/account, ticket, conversation, message, agent, team, assignment interval, ticket event, SLA policy version, business calendar, CSAT response, channel, category.
```

State cardinality explicitly. A parent-to-child relationship is not a reason to flatten the child into the parent. For every relationship, discuss optionality, lifecycle, deletion behavior and whether the child can exist before the parent is visible in the warehouse.

## 5. Source-to-warehouse mapping

| Source object | Source grain | Target model | Transformation / modeling decision |
|---|---|---|---|
| `customers` | One row per customer account | Silver `customers` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `tickets` | One row per ticket | Silver `tickets` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `conversations` | One row per conversation/session | Silver `conversations` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `messages` | One row per message | Silver `messages` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `ticket_assignments` | One row per assignment interval | Silver `ticket_assignments` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `ticket_events` | Append-only ticket state transitions. | Silver `ticket_events` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `sla_policy_versions` | One row per policy version and effective interval. | Silver `sla_policy_versions` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `csat_responses` | One row per survey response | Silver `csat_responses` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |

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
| `fact_ticket_lifecycle` | One row per ticket (accumulating snapshot). | ticket_id; created, first human response, resolved, closed, latest reopen timestamps; current status; lifecycle durations. |
| `fact_ticket_event` | One row per ticket state-change event. | event_id; ticket, event type, actor, event timestamp, old/new state; immutable history. |
| `fact_message` | One row per message. | message_id; conversation, ticket, sender, channel, sent timestamp, bot/human indicator; body stored outside analytics. |
| `fact_agent_assignment_interval` | One row per assignment interval. | assignment_id; ticket, agent, team, assigned/unassigned timestamps; handling seconds and handoff count. |
| `fact_sla_milestone` | One row per ticket × SLA milestone × policy version. | ticket, milestone type, target seconds, elapsed business seconds, breach flag, pause intervals. |
| `fact_csat_response` | One row per survey response. | response_id; ticket, score, survey version, submission time; response rate denominator defined separately. |

### Conformed dimensions

dim_customer (Type 2 for segment/plan/region history); dim_agent (Type 2 for team, location, employment state); dim_team; dim_channel; dim_ticket_category; dim_priority; dim_date/time; dim_sla_policy_version; dim_business_calendar (working windows, holidays, timezone).

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

1. **How do you compute first response when a bot sends an instant greeting and a human responds later?**
2. **Ticket is closed, reopened three times and merged into another ticket. How do you count resolution and ticket volume?**
3. **An agent is assigned for 12 minutes, then reassigned. Does the first agent receive the full handling time?**
4. **How do you calculate SLA with weekends, holidays, timezones, pause states and policy changes during ticket lifecycle?**
5. **How do you distinguish time-to-first-response, time-to-first-resolution and total time-to-close?**
6. **How do you measure bot containment without counting a bot-handled ticket that escalates later?**
7. **How do you avoid double counting CSAT if multiple survey responses exist for one ticket?**
8. **How do you model a customer changing from SMB to enterprise while preserving historical SLA attribution?**
9. **How would you store message content securely while supporting operational analytics and deletion requests?**
10. **What would your support data product contract and freshness/quality SLO look like?**

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

- AWS, [Dimensional modeling in Amazon Redshift](https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/).
- Meta Engineering, [How Meta understands data at scale](https://engineering.fb.com/2025/04/28/security/how-meta-understands-data-at-scale/) — schema and semantic annotations as governance foundations.
- Kimball Group, [Fact tables](https://www.kimballgroup.com/2008/11/fact-tables/).

Also validate warehouse-specific syntax and capabilities against the current vendor documentation. The SQL here is illustrative and intentionally not tied to one engine.

## 18. Case-specific relationship diagram

```text
customers 1 ─── N tickets 1 ─── N conversations 1 ─── N messages
                    │
                    ├── 1 ─── N ticket_events
                    ├── 1 ─── N ticket_assignments N ─── 1 agents
                    ├── N ─── 1 SLA policy version (resolved by effective time)
                    └── 0..N CSAT responses
teams 1 ─── N agents
business_calendars 1 ─── N SLA policy versions
```

A ticket can have multiple conversations and assignments. Do not assume one ticket equals one conversation or one agent.

## 19. Case-specific target DDL

```sql
CREATE TABLE fact_agent_assignment_interval (
  assignment_key BIGINT NOT NULL,
  assignment_id VARCHAR(64) NOT NULL,
  ticket_id VARCHAR(64) NOT NULL,
  agent_key BIGINT,
  team_key BIGINT,
  assigned_ts TIMESTAMP NOT NULL,
  unassigned_ts TIMESTAMP,
  assigned_seconds BIGINT,
  is_current BOOLEAN NOT NULL
);

CREATE TABLE fact_sla_milestone (
  ticket_id VARCHAR(64) NOT NULL,
  milestone_type VARCHAR(32) NOT NULL,
  policy_version_key BIGINT NOT NULL,
  start_ts TIMESTAMP NOT NULL,
  stop_ts TIMESTAMP,
  target_business_seconds BIGINT NOT NULL,
  elapsed_business_seconds BIGINT,
  paused_business_seconds BIGINT,
  breached_flag BOOLEAN
);
```

## 20. Case-specific SQL exercises

**A. First human response, excluding bots**

```sql
WITH first_human AS (
  SELECT c.ticket_id, MIN(m.sent_at) AS first_human_response_ts
  FROM support.conversations c
  JOIN support.messages m ON m.conversation_id = c.conversation_id
  WHERE m.sender_type = 'agent'
    AND m.message_type NOT IN ('internal_note', 'system')
    AND m.is_public = TRUE
  GROUP BY c.ticket_id
)
SELECT t.ticket_id,
       EXTRACT(EPOCH FROM (h.first_human_response_ts - t.created_at)) AS first_response_seconds
FROM support.tickets t
LEFT JOIN first_human h ON h.ticket_id = t.ticket_id;
```

**B. Assignment handoffs**

```sql
SELECT ticket_id,
       COUNT(*) AS assignment_intervals,
       COUNT(DISTINCT agent_id) AS distinct_agents,
       SUM(EXTRACT(EPOCH FROM (COALESCE(unassigned_at, CURRENT_TIMESTAMP) - assigned_at))) AS assigned_seconds
FROM support.ticket_assignments
GROUP BY ticket_id;
```

This is assignment elapsed time, not necessarily active handling time. Active handling requires agent activity/availability signals.

**C. CSAT average with response count**

```sql
SELECT DATE_TRUNC('week', submitted_at) AS response_week,
       COUNT(*) AS response_count,
       AVG(score) AS avg_csat
FROM support.csat_responses
GROUP BY 1;
```

Do not infer satisfaction among all tickets from survey respondents alone; report response rate and sampling bias.

## 21. Detailed source-to-target mapping matrix

| Source table | Source columns used | Target model | Transformation / semantic rule |
|---|---|---|---|
| `customers` | customer_id, account_id, segment, region, plan | dim_customer (SCD2); dim_account | Preserve historical segment/plan attribution. |
| `tickets` | ticket_id, customer_id, created_at, channel, priority, status, assignee | fact_ticket_lifecycle; dim_customer/channel/priority/category | Current row alone cannot preserve all state transitions. |
| `conversations` | conversation_id, ticket_id, channel, start/end | fact_conversation or degenerate conversation key on message fact | Ticket can have multiple conversations. |
| `messages` | message_id, conversation_id, sender, sent_at, type, public flag | fact_message; dim_sender_type; dim_channel | Exclude bot/system/internal messages from human-response metric. |
| `ticket_assignments` | assignment_id, ticket_id, agent, team, assigned/unassigned | fact_agent_assignment_interval; dim_agent/team | Interval grain; supports handoffs and time allocation. |
| `ticket_events` | event_id, ticket_id, type, timestamp, old/new state | fact_ticket_event; fact_ticket_lifecycle | Append-only transitions drive reconstructed state history. |
| `sla_policy_versions` | policy_version_id, plan, priority, channel, calendar, targets, validity | dim_sla_policy_version; fact_sla_milestone | Resolve policy effective at ticket/milestone time. |
| `csat_responses` | response_id, ticket_id, score, submitted_at | fact_csat_response; dim_survey_version | One row per response; include response-rate denominator. |
