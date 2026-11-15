# Mock Interview 2 — Customer Support Analytics Platform

**Interview level:** Senior / Staff / Principal Data Engineer  
**Suggested duration:** 75–90 minutes  
**Primary challenge:** Model ticket lifecycles, conversations, messages, assignments, queues, SLA clocks, agent productivity, customer experience, and reopen/escalation behavior without confusing event history with current state.

## 1. Interview brief

> Design a warehouse for a global customer-support organization. Leaders need backlog, first response, resolution time, SLA attainment, reopen rate, contact reasons, agent workload, channel mix, customer satisfaction, and cost-to-serve reporting.

The interviewer expects you to identify that a "ticket" is not the same as a "conversation", "message", "assignment", or "SLA interval". A ticket may contain multiple conversations and messages, move across teams, be reopened, and have multiple SLA clocks.

## 2. Requirement discovery

Ask:
1. What creates a ticket—email, chat, phone, social, API, or an agent?
2. Can multiple contacts be merged into one ticket? Can one conversation create multiple tickets?
3. Are messages customer-authored, agent-authored, bot-authored, or system-generated?
4. Does "first response" include an automated acknowledgement?
5. Does SLA pause outside business hours or while waiting for the customer?
6. Can a ticket be reopened after resolution? Does the SLA clock restart?
7. Are assignments one-to-one or can multiple agents collaborate?
8. Are queues and teams hierarchical and time-varying?
9. Which calendar defines business hours—customer locale, support locale, or contract?
10. Are CSAT responses optional and delayed?

Assumptions: one canonical ticket may contain multiple conversations; each conversation contains messages; tickets can have multiple assignment intervals and status transitions; SLA policies vary by contract, priority, channel, and region.

## 3. Business processes and grains

| Process | Fact | Grain |
|---|---|---|
| Ticket lifecycle | `fact_ticket_snapshot` | One row per ticket, current or periodic snapshot |
| Ticket transitions | `fact_ticket_event` | One row per status/priority/reopen event |
| Message activity | `fact_message` | One row per message |
| Agent assignment | `fact_ticket_assignment` | One row per ticket × assignee/team assignment interval |
| SLA milestones | `fact_sla_milestone` | One row per ticket × SLA type × milestone occurrence |
| SLA time accounting | `fact_sla_interval` | One row per ticket × SLA clock × active/pause interval |
| CSAT | `fact_csat_response` | One row per survey response |
| Backlog | `fact_ticket_daily_snapshot` | One row per date × ticket |

Do not sum ticket counts after joining to messages: a ticket with 40 messages becomes 40 joined rows. Aggregate messages to ticket grain or use distinct ticket IDs for ticket-level counts.

## 4. Conceptual model

```text
CUSTOMER ──< TICKET ──< CONVERSATION ──< MESSAGE
                 │
                 ├──< TICKET_EVENT
                 ├──< TICKET_ASSIGNMENT >── AGENT / TEAM / QUEUE
                 ├──< SLA_MILESTONE
                 ├──< SLA_INTERVAL
                 └──< CSAT_RESPONSE

TICKET ── CHANNEL / PRIORITY / REASON / PRODUCT / REGION
AGENT ──< AGENT_TEAM_MEMBERSHIP (effective-dated)
```

A ticket may move between queues. Team membership and agent attributes change over time. Preserve the assignment interval so historical productivity is attributed to the correct team and agent.

## 5. Dimensions

- `dim_customer`: customer segment, account tier, region; minimize direct PII.
- `dim_agent`: agent role, location, employment/skill cohort; Type 2 where history matters.
- `dim_team`, `dim_queue`: support organization hierarchy; retain effective dates.
- `dim_channel`: email, chat, phone, social, API, in-product.
- `dim_ticket_reason`: standardized taxonomy, with source-to-canonical mapping.
- `dim_priority`, `dim_status`, `dim_product`, `dim_language`.
- `dim_date` and `dim_business_calendar`: local holidays, support hours, contractual calendars.
- `dim_sla_policy`: policy version, target duration, pause rules, eligible statuses.

Use role-playing dates for created, first response, resolved, closed, and reopened timestamps.

## 6. Core fact design

### `fact_ticket_snapshot` — accumulating snapshot
**Grain:** one row per canonical ticket.  
Milestones: created, first human response, first resolution, final resolution, closed, latest reopen. Measures: message count (derived carefully), total active handling seconds, customer-wait seconds, agent-wait seconds, reopen count, transfer count. This table is convenient for current lifecycle reporting but should not replace the event history.

### `fact_message`
**Grain:** one row per message.  
Attributes: message ID, ticket ID, conversation ID, sender type, sender key (nullable/controlled), message channel, created timestamp, attachment count, bot flag, public/internal flag, language, response linkage. Avoid storing message body in broad analytics tables; text content has higher privacy and retention risk. Use secured systems for text analytics and publish only approved derived features.

### `fact_ticket_assignment`
**Grain:** one row per assignment interval.  
Fields: ticket, assigned agent/team/queue, start/end timestamps, assignment reason, assignment source, transfer flag. An assignment can end because of reassignment, resolution, or queue movement. Derive workload duration from interval intersections with the relevant business calendar.

### SLA model
Do not store a single `sla_met` boolean without its definition. One ticket can have first-response and resolution SLAs, each with policy versions and pauses. Store each milestone and the clock intervals that contributed to elapsed business time.

Example:
```sql
CREATE TABLE fact_sla_milestone (
  ticket_id            VARCHAR(64),
  sla_type             VARCHAR(32), -- FIRST_RESPONSE / RESOLUTION
  policy_key           BIGINT,
  milestone_number     INTEGER,
  target_seconds       BIGINT,
  elapsed_business_seconds BIGINT,
  achieved_ts          TIMESTAMP,
  breached_flag        BOOLEAN,
  pause_seconds        BIGINT,
  calculation_version  INTEGER
);
```

## 7. SLA calculation: the trap

**Interviewer:** "Ticket opened Friday at 5:55 PM. Support hours end at 6 PM and resume Monday at 9 AM. The first response arrives Monday at 9:10 AM. Is the response time 65 hours or 15 minutes?"

**Candidate:** "It depends on the contractual calendar and whether the clock pauses outside service hours. I would not calculate `response_ts - created_ts` and label it SLA elapsed time. I would intersect the ticket's active SLA intervals with the applicable business calendar, apply pause rules, and record the policy version. The wall-clock duration and SLA duration are separate measures."

A robust calculation uses:
1. policy selected at ticket creation or effective policy-change time;
2. local timezone and holiday calendar;
3. start/stop events for active and paused clock intervals;
4. deterministic handling of reopen and policy changes;
5. a versioned calculation so historical results can be reproduced.

## 8. Example SQL

### First human response (exclude bot acknowledgement)
```sql
WITH first_response AS (
  SELECT ticket_id, MIN(message_ts) AS first_human_response_ts
  FROM fact_message
  WHERE sender_type = 'AGENT'
    AND is_public = TRUE
  GROUP BY ticket_id
)
SELECT
  t.ticket_id,
  t.created_ts,
  r.first_human_response_ts,
  DATEDIFF(second, t.created_ts, r.first_human_response_ts)
      AS wall_clock_first_response_seconds
FROM fact_ticket_snapshot t
LEFT JOIN first_response r ON r.ticket_id = t.ticket_id;
```
This is wall-clock time, not necessarily SLA time. The engine-specific date-difference syntax may vary.

### Daily backlog snapshot
```sql
SELECT snapshot_date, queue_key, priority_key,
       COUNT(*) AS open_ticket_count,
       SUM(age_seconds) AS total_open_age_seconds
FROM fact_ticket_daily_snapshot
WHERE is_open = TRUE
GROUP BY snapshot_date, queue_key, priority_key;
```

### Agent workload without assignment fanout
First aggregate assignment intervals to agent/day, then join to agent dimension. Define whether collaborative work is credited fully, fractionally, or by primary owner; do not silently count one ticket as multiple completed tickets.

## 9. Data pipeline and change capture

- Ingest ticket, message, assignment, status, and survey events independently.
- Preserve source IDs, event IDs, source update versions, ingestion timestamps, and deletion/tombstone markers.
- Use CDC for mutable ticket/agent/team records, and event streams for append-only activity where available.
- Normalize status/reason taxonomies in a governed mapping layer.
- Build event facts before deriving ticket snapshots.
- Recompute affected tickets when late messages, merged tickets, corrected timestamps, or delayed survey responses arrive.
- Keep raw payloads access-controlled and apply retention to message content and attachments separately.

For merged tickets, preserve a canonical ticket identity plus a ticket-relationship mapping (`merged_into_ticket_id`, effective time, merge reason). Historical facts must remain traceable to original source tickets.

## 10. Quality and governance

| Control | Expected invariant |
|---|---|
| Ticket identity | Canonical ticket IDs resolve to one current snapshot |
| Message uniqueness | Message source ID is not duplicated |
| Assignment validity | End timestamp is after start timestamp |
| SLA reproducibility | Same input events + policy version yield same result |
| Status history | Every current status can be traced to events |
| CSAT integrity | Response links to a valid survey/ticket |
| Privacy | Message content is restricted and retention enforced |
| Metric consistency | Bot acknowledgements excluded consistently from human-response KPI |

PII includes email, phone, names, message content, and attachments. Prefer pseudonymous customer keys in analytics. Apply row/column controls, purpose limitation, audited access, deletion propagation, and aggregated reporting.

## 11. Realistic interview dialogue

**Interviewer:** "Support says there are 100,000 tickets, but the dashboard says 127,000."  
**Candidate:** "I'd inspect the join path and grain. If ticket records are joined to messages or assignment intervals, counts can fan out. I'd compare canonical ticket counts before and after each join and aggregate child facts independently."

**Interviewer:** "An agent sends an automated greeting. Is that first response?"  
**Candidate:** "Only if the business definition says so. I'd expose separate `first_auto_response_ts` and `first_human_response_ts`, then define the KPI denominator and exclusions in a metric contract."

**Interviewer:** "A ticket is resolved, reopened, and resolved again. Which resolution time do you report?"  
**Candidate:** "There are multiple valid measures: time to first resolution, time to final resolution, cumulative active resolution time, and number of reopen cycles. I would publish them distinctly rather than overwrite the first resolution timestamp."

**Interviewer:** "Can we rank agents by tickets closed?"  
**Candidate:** "Not without adjusting for complexity, assignment duration, queue mix, collaboration, and quality outcomes. Raw closure counts can incentivize premature closure. I'd present workload and quality measures together and make attribution rules transparent."

## 12. Principal-level probes

1. Who owns the definition of SLA: Support Operations, Legal, or Engineering?
2. How do you model a policy change without rewriting history?
3. How do you handle ticket merges and split operations?
4. How would you migrate from vendor A to vendor B while retaining longitudinal reporting?
5. How do you expose trusted semantic metrics to BI and experimentation teams?
6. How do you protect customer message content while enabling approved AI quality analysis?
7. What are the SLOs for ticket freshness, SLA recalculation, and data completeness?
8. How would you detect a vendor integration silently dropping status events?

## 13. Common mistakes
- Treating ticket and message as the same grain.
- Counting bot acknowledgements as human responses without agreement.
- Using wall-clock duration for a business-hours SLA.
- Overwriting status history.
- Attributing all ticket work to the final assigned agent.
- Ignoring reopens, merges, policy versions, and time zones.
- Exposing message content broadly to analysts.

## 14. Practice exercise

A ticket is assigned to Team A for 20 minutes, Team B for 2 hours, then reopened and assigned to Team C. It has 8 customer messages, 5 agent messages, 2 bot messages, and two SLA policies during its life.

Design the model and explain:
- first response versus first human response;
- ownership attribution;
- first resolution versus final resolution;
- SLA policy versioning;
- ticket count and message count without fanout;
- how an agent-level dashboard avoids misleading comparisons.

### Scoring rubric (10 points)
| Dimension | Points |
|---|---:|
| Requirements and KPI definitions | 1.5 |
| Grain and lifecycle model | 2.0 |
| SLA and business-calendar correctness | 2.0 |
| Pipeline, late data, and CDC | 1.0 |
| Privacy and quality | 1.0 |
| Performance and semantic layer | 1.0 |
| Principal leadership and trade-offs | 1.5 |

## 15. Further reading
- AWS, [Redshift best practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html).
- AWS, [Dimensional modeling in Amazon Redshift](https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/) — grain and fact/dimension design.
