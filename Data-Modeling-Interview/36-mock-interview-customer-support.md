# Lesson 36 — Mock Interview: Customer Support Platform

**Level:** Senior / Staff / Principal Data Engineer
**Duration:** 60-90 minutes
**Case type:** End-to-end source-to-warehouse data modeling

## How to use this case in an interview

Treat this as a full interview exercise. Focus on modeling event-driven processes with state changes, handling complex hierarchies, and designing for both operational and analytical needs.

## Learning Objectives

- Model state machine processes (ticket lifecycle with transitions)
- Handle 1:M and M:N relationships in dimensional models
- Design factless fact tables for tracking compliance
- Build time-intelligence metrics (aging, SLA breaches, resolution time)
- Model agent hierarchies and team assignments

## Interview Dialogue

**Interviewer:** We're building a data platform for a customer support / contact center company like Zendesk. We need to model tickets, interactions, agents, customers and SLA compliance. Design the data warehouse.

**Candidate:** I'd start by identifying the core business processes:

1. **Ticket Lifecycle** — creation, assignment, status transitions, resolution
2. **Interactions** — each interaction (email, chat, phone, SMS) within a ticket
3. **SLA Compliance** — meeting time-based SLAs based on ticket priority, customer tier, and channel
4. **Agent Performance** — handle time, resolution rate, customer satisfaction

**Interviewer:** Walk me through the source system design.

**Candidate:** The operational systems likely have:

**Source Systems:**

**Tickets Table**
- ticket_id, customer_id, subject, description, channel_type
- status, priority, created_at, updated_at, assigned_agent_id

**Interactions Table**
- interaction_id, ticket_id, interaction_type (email, phone, chat, SMS)
- initiated_by (customer, agent), started_at, duration_seconds, agent_id

**Agents Table**
- agent_id, email, first_name, last_name
- team_id, role, hire_date, status

**Customers Table**
- customer_id, email, account_id, tier (free, pro, enterprise)
- created_at, region

**Teams Table**
- team_id, team_name, manager_id, department

**SLA Policies Table**
- sla_id, ticket_type, priority, first_response_minutes
- resolution_hours, escalation_minutes, active_from, active_to

**Interviewer:** How would you design the warehouse fact and dimension tables?

**Candidate:** I'd model each process at its natural grain:

**Core Dimensions:**

**Dim_Ticket**
- ticket_sk (surrogate key), ticket_id, customer_sk
- subject, channel_sk, priority_sk
- created_date_sk, created_time_sk
- assigned_agent_sk, team_sk, status
- current_flag, start_date, end_date (SCD Type 2 for status changes)

**Dim_Customer**
- customer_sk, customer_id, account_sk, tier_sk
- region_sk, created_date_sk

**Dim_Agent**
- agent_sk, agent_id, team_sk, role_sk
- hire_date_sk, status, manager_sk

**Dim_Date** — standard date dimension
**Dim_Time** — granular time dimension for SLA calculations

**Core Fact Tables:**

**Fact_Ticket_Events** (transactional fact, one row per status change)
- ticket_sk, date_sk, time_sk, agent_sk
- previous_status_sk, new_status_sk, event_timestamp
- wait_time_before_assignment_seconds, time_in_status_hours
- sla_breached_flag, sla_breach_type_sk

**Fact_Interactions** (transactional fact, one row per interaction)
- interaction_sk, ticket_sk, date_sk, time_sk
- interaction_type_sk, agent_sk, duration_seconds
- initiated_by_sk, resolution_flag

**Fact_Ticket_SLA** (periodic snapshot fact, daily)
- ticket_sk, date_sk, sla_policy_sk
- first_response_time_minutes, resolution_time_hours
- sla_met_flag, sla_breach_count
- total_interactions_count, total_wait_time_hours

**Factless Fact Tables:**

**Fact_SLA_Compliance** (factless fact table)
- ticket_sk, date_sk, sla_policy_sk
- tracks when tickets met/did not meet specific SLAs

**Interviewer:** How do you handle the ticket lifecycle state machine?

**Candidate:** I'd track each state transition explicitly in Fact_Ticket_Events:
- **New → Opened** → Assigned → In Progress → Resolved → Closed
- **New → Closed** (immediate close)
- Any status → Escalated → status (re-open)

Each transition captures:
- The timestamp of transition
- The agent who triggered it
- The time spent in the previous state
- Whether any SLA was breached

**Interviewer:** How do you handle the agent hierarchy?

**Candidate:** Agents report to teams, teams have managers who are also agents. This creates a recursive hierarchy. In the dimensional model, I'd add:
- team_sk directly in Dim_Agent
- manager_sk in Dim_Agent (pointing back to another agent row)
- Department level via the team hierarchy

For roll-ups, I can create a separate hierarchy table or use recursive CTEs in queries.

**Interviewer:** How do you handle the M:N relationship between tickets and tags/categories?

**Candidate:** If a ticket can have multiple tags, I'd use a bridge table:

**Bridge_Ticket_Tags**
- ticket_sk, tag_sk

This preserves the M:N relationship. For reporting, I'd join this bridge table to fact tables when tag-level filtering is needed.

## Key Takeaways

| Concept | Application in this case |
|---------|--------------------------|
| State machine tracking | Fact_Ticket_Events captures all status transitions |
| Factless fact tables | Track SLA compliance without measures |
| Periodic snapshot facts | Fact_Ticket_SLA for daily aging and SLA metrics |
| Bridge tables | Handle M:N relationships (ticket ↔ tags) |
| Recursive hierarchies | Agent → Team → Manager → Department |
| Time intelligence | SLA breach detection, resolution time calculation |
| SCD Type 2 | Track ticket status changes with history |
| Role-playing dimensions | Customer vs agent in interactions |
| Multiple grain levels | Interaction-level vs ticket-level facts |
| Data quality | Validation of SLA calculations against source system |

## Interview Scoring Rubric

| Level | What to look for |
|-------|------------------|
| **Junior** | Can identify basic fact and dimension tables |
| **Senior** | Models state transitions, understands grain, uses bridge tables |
| **Staff** | Handles recursive hierarchies, factless facts, SLA time-intelligence |
| **Principal** | Addresses data quality, multi-channel attribution, compliance reporting |

### Related Lessons

- [Lesson 3](./03-identifying-the-right-business-processes.md) — Identifying business processes
- [Lesson 4](./04-determining-grain-of-fact-table.md) — Determining grain of fact tables
- [Lesson 14](./14-evolving-models-based-on-changing-requirements.md) — Evolving models
- [Lesson 21](./21-slowly-changing-dimensions-types-1-4.md) — SCD Types 1-4
- [Lesson 24](./24-fact-table-types.md) — Fact table types
- [Lesson 32](./32-bridge-tables-and-factless-facts.md) — Bridge tables and factless facts
