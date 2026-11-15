# Lesson 32: Mock Interview 2 — Design a Data Warehouse Schema for Customer Support

> **Principal Mentor Note:** Customer Support & Contact Center data models (Zendesk, Intercom, Salesforce Service Cloud, Amazon Connect) look deceptively simple on the surface (*"Tickets and Agents"*), but hide **four classic Principal-level traps**: **(1) Non-linear ticket lifecycles** (tickets get reopened 3 times, escalated across 4 queues, and merged), **(2) Business-hours SLA math** (pausing the SLA clock when waiting on the customer or over weekends), **(3) AI Chatbot deflection vs. Human Agent handoff**, and **(4) Agent shift capacity vs. ticket handling utilization**.

---

## 1. Phase 1: Clarifying Questions & Scope Alignment (Minutes 0–6)

### Interviewer Prompt
> *"Design an analytical data warehouse for our global Customer Support organization. Leadership wants to measure First Contact Resolution (FCR), SLA breach rates, AI bot deflection efficiency, agent productivity, and top drivers of customer contacts."*

### Candidate's 4-Bucket Discovery Questions
1. **Channels & AI Deflection:** *"Are we modeling omnichannel support—in-app live chat, email, phone voice calls, and automated AI chatbot sessions—including conversations resolved purely by the AI bot without ever creating a human ticket?"*
   * *Interviewer:* *"Yes—capture both AI bot conversations (deflected vs. escalated) and human-handled support tickets."*
2. **Lifecycle & SLA Rules:** *"Can tickets be reopened after being marked solved, transferred across multiple specialist queues (Tier 1 $\rightarrow$ Tier 2 $\rightarrow$ Trust & Safety), and does our SLA clock pause while waiting on customer replies?"*
   * *Interviewer:* *"Yes to all three. SLA timers pause in `PENDING_CUSTOMER` status and account for regional business hours."*
3. **Agent Capacity & CSAT:** *"Do we also track agent scheduled shift hours (for utilization/occupancy) and post-resolution Customer Satisfaction (CSAT / NPS) surveys?"*
   * *Interviewer:* *"Yes, agent occupancy and CSAT scores by contact reason are top executive KPIs."*
4. **Scale & Latency:** *"What volume of conversations/tickets per day, freshness SLA, and compliance rules (e.g., PII redaction in transcripts/metadata) apply?"*
   * *Interviewer:* *"Assume 3M support interactions/day, 15M ticket events/day, 30-minute freshness for WFM (Workforce Management) dashboards, 5-year retention, and strict PII isolation."*

---

## 2. Phase 2: Enterprise Bus Matrix & Grain Contract (Minutes 6–12)

### Enterprise Bus Matrix
```
Business Process (Fact Table)              | Date | Time | Customer | Agent | Queue/Team | Contact Reason | Channel | SLA Policy
-------------------------------------------+------+------+----------+-------+------------+----------------+---------+-----------
fct_support_tickets (Accumulating Snap)    |  X   |  X   |    X     |   X   |     X      |       X        |    X    |     X
fct_ticket_state_events (Transaction)      |  X   |  X   |    X     |   X   |     X      |       X        |    X    |
fct_ticket_messages (Transaction)          |  X   |  X   |    X     |   X   |            |                |    X    |
fct_agent_daily_capacity (Periodic Snap)   |  X   |      |          |   X   |     X      |                |         |
```

### Explicit Grain Declarations
1. **`fct_support_tickets` (Accumulating Snapshot):** One row per unique support ticket/conversation (`ticket_id`), tracking end-to-end lifecycle timestamps, net active SLA handling time (excluding `PENDING_CUSTOMER` hold time), reopen count, AI deflection flag, and CSAT score.
2. **`fct_ticket_state_events` (Transaction Fact):** One row per discrete ticket state transition, queue transfer, or escalation (`event_id`), preserving complete auditability for multi-hop transfers and reopenings.
3. **`fct_agent_daily_capacity` (Periodic Snapshot):** One row per `(agent_sk, shift_date_sk)`, capturing scheduled shift minutes, active handle minutes, idle available minutes, and tickets solved.

---

## 3. Phase 3: High-Level Constellation Diagram & DDL (Minutes 12–26)

```
+---------------------+       +----------------------+       +-----------------------+
| dim_customer [SCD2] |       |  dim_date [CONFORM]  |       | dim_agent [SCD2]      |
| PK: customer_sk     |       |  PK: date_sk         |       | PK: agent_sk          |
| NK: customer_id     |       +----------+-----------+       | NK: agent_id          |
| ltv_tier, country   |            ^            ^            | team_name, site_city, |
+----------+----------+            |            |            | tenure_band, bpo_vendor|
           | 1                     |            |            +-----+-------------+---+
           |                       |            |                  | 1           | 1
           | N             +-------+----+  +----+--------------+   |             |
           +-------------->| fct_support|  | fct_ticket_state_ |<--+             | N
                           | _tickets   |  | events [TXN]      |                 v
+---------------------+    | [ACC-SNAP] |  | Grain: 1 row /    |    +-----------------------+
| dim_contact_reason  |--->| Grain: 1 / |  | state transition  |    | fct_agent_daily_      |
| [SCD2 - Flattened]  |1:N | ticket_id  |  +-------------------+    | capacity [PER-SNAP]   |
| category_l1/l2/l3   |    +----+-------+                           | Grain: 1/(agent,date) |
+---------------------+         ^                                   +-----------------------+
                                | 1:N
                    +-----------+-----------+
                    | dim_sla_policy [SCD2] |
                    | tier, target_frt_mins |
                    | target_ttr_mins       |
                    +-----------------------+
```

### Production DDL (Snowflake / Apache Iceberg)

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE gold.dim_agent (
    agent_sk                BIGINT PRIMARY KEY,
    agent_id                VARCHAR(64) NOT NULL,
    agent_type              VARCHAR(16) NOT NULL,  -- HUMAN, AI_BOT_V3
    team_name               VARCHAR(64) NOT NULL,  -- BILLING_T1, FRAUD_T2, VIP_CONCIERGE
    site_location           VARCHAR(64) NOT NULL,  -- MANILA_BPO, BENGALURU_HUB, AUSTIN_HQ, CLOUD_BOT
    bpo_vendor_name         VARCHAR(64),           -- 'In-House' vs vendor name for cost/QA tracking
    manager_name            VARCHAR(128),          -- Flattened hierarchy
    hire_date               DATE,
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

CREATE TABLE gold.dim_contact_reason (
    contact_reason_sk       INT PRIMARY KEY,
    reason_code             VARCHAR(64) NOT NULL,
    reason_l1               VARCHAR(64) NOT NULL,  -- e.g., 'PAYMENTS_BILLING'
    reason_l2               VARCHAR(64) NOT NULL,  -- e.g., 'REFUND_REQUEST'
    reason_l3               VARCHAR(64) NOT NULL,  -- e.g., 'DUPLICATE_CHARGE'
    is_self_serve_eligible  BOOLEAN NOT NULL
);

-- ============================================================================
-- FACT 1: SUPPORT TICKETS LIFECYCLE (Accumulating Snapshot)
-- Grain: One row per support ticket / conversation (ticket_id)
-- ============================================================================
CREATE TABLE gold.fct_support_tickets (
    ticket_id               VARCHAR(64) PRIMARY KEY,
    linked_order_id         VARCHAR(64),           -- Degenerate FK to core business transaction!
    created_date_sk         INT NOT NULL,          -- Partition key
    first_agent_reply_date_sk INT NOT NULL DEFAULT -1,
    latest_solved_date_sk   INT NOT NULL DEFAULT -1,
    closed_date_sk          INT NOT NULL DEFAULT -1,
    created_at_utc          TIMESTAMP NOT NULL,
    first_bot_response_utc  TIMESTAMP,
    escalated_to_human_utc  TIMESTAMP,
    first_human_reply_utc   TIMESTAMP,
    latest_solved_at_utc    TIMESTAMP,
    closed_at_utc           TIMESTAMP,
    -- Dimension Foreign Keys
    customer_sk             BIGINT NOT NULL,
    initial_agent_sk        BIGINT NOT NULL,       -- Often AI_BOT_V3
    solving_agent_sk        BIGINT NOT NULL DEFAULT -1, -- Final owner who solved the ticket
    solving_queue_sk        INT NOT NULL DEFAULT -1,
    contact_reason_sk       INT NOT NULL,
    channel_sk              INT NOT NULL,          -- IN_APP_CHAT, VOICE, EMAIL, SOCIAL
    sla_policy_sk           INT NOT NULL,
    -- Current State & Flags
    current_status          VARCHAR(24) NOT NULL,  -- OPEN, PENDING_CUSTOMER, SOLVED, CLOSED
    is_terminal_closed      BOOLEAN NOT NULL,      -- TRUE after 7-day post-solve reopen window expires!
    is_bot_deflected        BOOLEAN NOT NULL,      -- TRUE if solved by AI bot with zero human handoff
    is_first_contact_resolved BOOLEAN NOT NULL,    -- TRUE if solved in <=1 agent touch & 0 reopens
    -- Pre-Computed SLA & Handle Time Measures (Active Business Seconds — excludes customer wait!)
    first_response_bus_sec  INT,                   -- Business-hours seconds to first human reply
    net_resolution_bus_sec  INT,                   -- Active agent/queue time excluding PENDING_CUSTOMER
    total_agent_handle_sec  INT NOT NULL DEFAULT 0,-- Sum of active agent screen/talk time
    -- Reopen & Transfer Counters
    reopen_count            INT NOT NULL DEFAULT 0,
    queue_transfer_count    INT NOT NULL DEFAULT 0,
    agent_touch_count       INT NOT NULL DEFAULT 0,
    -- SLA Breach & CSAT Measures
    is_frt_sla_breached     BOOLEAN NOT NULL DEFAULT FALSE,
    is_ttr_sla_breached     BOOLEAN NOT NULL DEFAULT FALSE,
    csat_score              TINYINT,               -- 1..5 scale (NULL if survey not answered)
    csat_responded_flag     TINYINT NOT NULL DEFAULT 0
)
PARTITION BY (created_date_sk)
CLUSTER BY (is_terminal_closed, solving_queue_sk);

-- ============================================================================
-- FACT 2: TICKET STATE & TRANSFER EVENTS (Transaction Fact)
-- Grain: One row per discrete state transition, queue hop, or reopen (event_id)
-- ============================================================================
CREATE TABLE gold.fct_ticket_state_events (
    event_id                VARCHAR(64) PRIMARY KEY,
    ticket_id               VARCHAR(64) NOT NULL,
    event_date_sk           INT NOT NULL,
    occurred_at_utc         TIMESTAMP NOT NULL,
    actor_agent_sk          BIGINT NOT NULL,
    from_status             VARCHAR(24),
    to_status               VARCHAR(24) NOT NULL,
    from_queue_sk           INT,
    to_queue_sk             INT NOT NULL,
    time_in_prior_state_sec INT NOT NULL,          -- Exact duration spent in previous state/queue
    is_business_hours_sec   INT NOT NULL           -- Pre-computed using site calendar for SLA audit
)
PARTITION BY (event_date_sk)
CLUSTER BY (ticket_id);
```

---

## 4. Phase 4: Query Validation (Minutes 26–33)

```sql
-- Query: Weekly AI Bot Deflection Rate, First Contact Resolution (FCR), SLA Compliance & CSAT
SELECT
    d.year_week,
    cr.reason_l1                                                    AS contact_category,
    COUNT(t.ticket_id)                                              AS total_contacts,
    ROUND(
        SUM(CASE WHEN t.is_bot_deflected THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(t.ticket_id), 0),
    2)                                                              AS bot_deflection_pct,
    ROUND(
        SUM(CASE WHEN t.is_first_contact_resolved THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(t.ticket_id), 0),
    2)                                                              AS fcr_pct,
    ROUND(
        SUM(CASE WHEN t.is_ttr_sla_breached = FALSE THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(t.ticket_id), 0),
    2)                                                              AS sla_compliance_pct,
    ROUND(AVG(t.csat_score), 2)                                     AS avg_csat_score
FROM gold.fct_support_tickets t
JOIN gold.dim_date d ON t.created_date_sk = d.date_sk
JOIN gold.dim_contact_reason cr ON t.contact_reason_sk = cr.contact_reason_sk
WHERE d.calendar_date >= DATE '2026-10-01'
  AND t.current_status IN ('SOLVED', 'CLOSED')
GROUP BY 1, 2
ORDER BY 3 DESC;
```

---

## 5. Phase 5: Principal Curveballs & "Top-Up" Questions (Minutes 33–45)

1. **Curveball 1 (Calculating Business-Hours SLA Excluding `PENDING_CUSTOMER` Wait Time):** *"How do you compute `net_resolution_bus_sec` accurately without writing nightmarish SQL calendar joins in BI?"*
   * **Principal Answer:** Never compute business-hours SLA math in BI queries! Compute it incrementally in the **Silver-to-Gold event stream processor** as state transitions arrive in `fct_ticket_state_events`. Every time a ticket transitions out of an active state (`OPEN -> PENDING_CUSTOMER` or `OPEN -> SOLVED`), a deterministic calendar function (`calc_business_seconds(prior_ts, current_ts, sla_calendar_id)`) calculates the exact business-hour delta and increments `net_resolution_bus_sec` on `fct_support_tickets`. When the ticket is in `PENDING_CUSTOMER`, the increment is `0`.
2. **Curveball 2 (Ticket Reopenings Breaking FCR & SLA):** *"A ticket is marked `SOLVED` on Monday (so `is_first_contact_resolved = TRUE`), and on Thursday the angry customer replies 'This still isn't fixed!' reopening the ticket. How does your model handle this?"*
   * **Principal Answer:** Keep `is_terminal_closed = FALSE` for a **7-day post-solve grace window** before locking the ticket as `CLOSED`. When the customer replies on Thursday, the `MERGE` pipeline updates `fct_support_tickets`: increments `reopen_count += 1`, flips `is_first_contact_resolved = FALSE`, updates `latest_solved_at_utc = NULL`, and resumes accumulating `net_resolution_bus_sec` until final resolution.
3. **Curveball 3 (Ticket Merging & Duplicate Contacts):** *"A customer sends 3 separate emails about the same lost package, and an agent merges Tickets #2 and #3 into Master Ticket #1. How do you avoid inflating ticket counts and FCR?"*
   * **Principal Answer:** Add `merged_into_ticket_id` and `is_merged_duplicate` flag on `fct_support_tickets` (and set `current_status = 'MERGED_CLOSED'`). Governed semantic views exclude `is_merged_duplicate = TRUE` from unique issue counts and FCR denominators, while keeping them visible for raw agent email volume metrics.