# Module 2: Gathering Business Requirements
## Lesson 2.8 — Example: Business Requirements Gathering

> *"The best data models don't start with tables — they start with conversations."* This lesson walks through a complete, end-to-end requirements-gathering scenario so you can see the entire process in action before you face it in an interview.

---

### 1. The Scenario

You're interviewing for a **Senior Data Engineer** role at a ride-sharing company (think Uber/Lyft). The interviewer opens with:

> *"We need a data warehouse for our ride-sharing platform. Walk me through how you'd figure out what to build."*

This is a deliberately vague prompt. Your goal is to **ask the right questions**, **capture the requirements**, **summarize them back**, and **show how they drive design decisions** — all within a 20-minute interview window.

---

### 2. The Conversation — A Walkthrough

Below is a realistic back-and-forth. Your questions are in **bold**; the interviewer's responses are in *italics*.

#### Phase 1 — Business Context

**"Let me start by understanding the business problem. What decisions will this data warehouse enable? Is this focused on a specific area — operations, finance, marketing, or all of the above?"**

*"We need it to support executives understanding the health of the business daily, the operations team optimizing driver allocation in each city, and the finance team reconciling revenue and payouts."*

**"So three stakeholder groups: executives, operations, and finance. Are there any other consumers — product, marketing, data science?"**

*"Data science needs it for pricing models and demand forecasting. Marketing wants to track the impact of promotions on ride volume."*

**"Got it. What are the top 3–4 business outcomes you're trying to drive with this warehouse?"**

*"First, reducing the time between a rider requesting a ride and a driver arriving — that's ETA accuracy. Second, optimizing driver utilization so we don't have surges without supply or supply without demand. Third, accurately tracking revenue and cost per ride for finance. And fourth, understanding rider retention — we're losing too many riders after their first month."*

#### Phase 2 — Data Consumers & Their Needs

**"You've mentioned five consumer groups. Let me clarify each one's needs. Starting with executives — what does the daily health dashboard look like? What metrics are on it?"**

*"Executives need: total rides completed, gross booking value, rider retention rate, average ETA, and city-level breakdowns. They check it every morning, so daily data is fine."*

**"Operations — you mentioned optimizing driver allocation. How real-time does that need to be?"**

*"Operations uses live dashboards showing active drivers, wait times, and ride requests by city zone. That needs to be near real-time — under 5 minutes of latency. They make decisions throughout the day."*

**"Finance — what's the granularity they need for revenue and payouts?"**

*"They need per-ride detail for 24 months for audit and reconciliation. Monthly aggregates forever. They also need to track promotions and driver incentives back to individual rides."*

**"Data science — what models are they running, and what data access do they need?"**

*"Pricing models need historical ride data with time, location, price, and weather. Demand forecasting needs rider signup, ride frequency, and cancellation data. They'll run custom queries, so they need access to raw data."*

#### Phase 3 — Data Sources

**"Where does all this data currently live? What systems are producing it?"**

*"We have four main systems. The Ride Service tracks every ride lifecycle: requested, accepted, started, completed, or cancelled. The Driver Service manages driver profiles, status, location updates, and shifts. The Payment Service handles rider charges, driver payouts, tips, and promotions. And the User Service has rider profiles, signups, and account status."*

**"Are these all internal databases, or are there external sources?"**

*"Most are internal PostgreSQL databases. We also ingest GPS telemetry from drivers' phones — about a location ping every 30 seconds. And we get weather data from a third-party API."*

**"What's the reliability of these sources? Any known data quality issues?"**

*"GPS telemetry is the messiest — we get duplicates, gaps when a driver loses signal, and the occasional phantom location jump. The core transactional systems are reliable but sometimes have late-arriving data — like a ride that's marked completed 30 minutes after the driver ended it."*

**"Is there an existing data pipeline, or are we building from scratch?"**

*"We have a basic ETL using Airflow that loads daily snapshots into a data lake on S3. It works but is slow and doesn't support the real-time needs of operations."*

#### Phase 4 — Volume, Latency & Retention

**"Let's quantify the scale. How many rides per day? How many drivers?"**

*"About 2 million rides per day across our active cities. We have roughly 500,000 active drivers. GPS telemetry is the biggest volume — each driver sends a ping every 30 seconds when active."**

**"So for GPS: 500,000 drivers × 480 pings per day per driver — that's about 240 million telemetry events daily. Is that right?"**

*"Yes, and it's growing. We're adding two new cities per quarter, which adds roughly 15% more volume each time."*

**"What about the other sources? Estimated daily rows?"**

*"Rides: 2 million rows. Payments: roughly 3–4 million (some rides have multiple charges for tips and surcharges). User events like signups are about 50,000 per day. Ride lifecycle events — if we track each status change — about 6 million per day since each ride has roughly 3 status changes."*

**"What are your latency requirements for each consumer?"**

*"Executives: daily batch is fine. Finance: daily, with ability to drill into any day for the past 24 months. Operations: under 5 minutes, preferably under 2. Data science: within 24 hours is fine — they batch-train models weekly."*

**"Data retention?"**

*"GPS telemetry: 90 days rolling — after that it's too expensive and not useful. Ride, payment, and user data: 5 years for compliance and audit. Aggregated monthly metrics: indefinite."*

**"Any compliance or security constraints?"**

*"Yes. Rider phone numbers and addresses are PII — must be encrypted at rest. We operate in Europe, so GDPR applies — we need the ability to delete a rider's data on request. Driver bank account details for payouts must be masked in the warehouse."*

---

### 3. Structured Requirements Document

After the conversation, you'd organize everything like this:

#### 3.1 Business Objectives

| # | Objective | Priority |
|---|-----------|----------|
| 1 | Reduce ETA inaccuracy to improve rider satisfaction | High |
| 2 | Optimize driver utilization to balance supply and demand | High |
| 3 | Accurately track revenue, costs, and payouts per ride for finance | High |
| 4 | Understand and improve rider retention, especially Month 1 | Medium |

#### 3.2 Key Metrics & Definitions

| Metric | Formula | Grain | Additive? |
|--------|---------|-------|-----------|
| **Total Rides Completed** | `COUNT(ride_id)` WHERE status = 'completed' | Per city, per day | ✅ Yes |
| **Gross Booking Value (GBV)** | `SUM(ride_total_fare)` | Per ride | ✅ Yes |
| **ETA Accuracy** | `AVG(ABS(estimated_arrival - actual_arrival))` in minutes | Per ride | ❌ No (ratio/avg) |
| **Driver Utilization** | `SUM(active_trip_minutes) / SUM(shift_minutes)` per driver | Per driver, per day | ❌ No (ratio) |
| **Rider Retention (Month 1)** | `(riders with ≥1 ride in month 2) / (riders with ≥1 ride in month 1)` | Monthly | ❌ No (semi-additive) |
| **Avg Fare per Ride** | `SUM(fare) / COUNT(ride_id)` | Per city, per day | ❌ No (ratio) |
| **Cancellation Rate** | `COUNT(cancelled_rides) / COUNT(total_rides)` | Per city, per day | ❌ No (ratio) |

#### 3.3 Data Consumers & Needs

| Consumer | Use Case | Latency Need | Query Pattern |
|----------|----------|-------------|---------------|
| **Executives** | Daily health dashboard (5 KPIs) | Daily batch | Pre-aggregated summaries, filtered by city |
| **Operations** | Live driver/ride supply-demand dashboards | < 5 min | High-cardinality filters by city zone; frequent refresh |
| **Finance** | Revenue reconciliation, payout audit | Daily | Drill-down to per-ride detail; 24-month lookback |
| **Data Science** | Pricing models, demand forecasting | < 24 hours | Ad-hoc queries; joins across rides, weather, user data |
| **Marketing** | Promotion effectiveness tracking | Daily | Cohort analysis, A/B test comparison |

#### 3.4 Data Sources

| Source | Type | Daily Volume | Format | Notes |
|--------|------|-------------|--------|-------|
| **Ride Service** | PostgreSQL (transactions) | ~6M lifecycle events | Relational | Status changes per ride |
| **Driver Service** | PostgreSQL | ~500K driver rows | Relational | Profiles, shifts, status |
| **Payment Service** | PostgreSQL | ~3–4M payment records | Relational | Charges, payouts, tips |
| **User Service** | PostgreSQL | ~50K user events | Relational | Signups, account changes |
| **GPS Telemetry** | Event stream (Kafka) | ~240M location pings | JSON | High volume; duplicates & gaps |
| **Weather API** | REST API | ~10K hourly readings | JSON | External; per city zone |

#### 3.5 Volume Estimates

| Data Source | Daily Rows | Annual Estimate (×365) | Notes |
|-------------|-----------|----------------------|-------|
| GPS Telemetry | 240M | ~87.6B | 90-day retention → ~21.6B active |
| Ride Lifecycle | 6M | ~2.19B | Full 5-year retention |
| Payments | 4M | ~1.46B | Full 5-year retention |
| Drivers | 500K | ~182M (mostly updates) | SCD-2 tracking needed |
| Users | 50K | ~18M | SCD-2 for retention analysis |
| **Total** | **~250.5M** | **~91.5B** | Before dedup / aggregation |

#### 3.6 Retention Policy

| Data Category | Retention | Storage Tier |
|---------------|-----------|-------------|
| GPS Telemetry | 90 days rolling | Hot storage (high-performance) |
| Ride & Payment Detail | 5 years | Hot for 24 months, cold for 3 more |
| User & Driver History | 5 years | Hot storage |
| Monthly Aggregates | Indefinite | Warm storage |
| Raw Source Snapshots | 30 days | Cold / archival (for debugging) |

#### 3.7 Compliance Requirements

| Requirement | Scope | Modeling Impact |
|-------------|-------|----------------|
| **GDPR Right to Delete** | Rider PII (name, phone, address) | Must support row-level deletes; soft-delete with audit trail |
| **PII Encryption** | All rider personal data | Encrypted columns; separate PII fact/dimension |
| **Bank Account Masking** | Driver payout accounts | Masked view in warehouse; raw data in secure payment vault |
| **Data Residency (EU)** | All European rider/driver data | Separate EU storage cluster; geo-partitioned |

---

### 4. The Summary Statement

Before you start designing, you present this back to the interviewer:

> *"Based on our discussion, here's what I understand:*
>
> *We're building a data warehouse for a ride-sharing platform serving ~2 million daily rides and 500,000 active drivers. Five consumer groups need data at different latencies: executives get daily dashboards, operations needs sub-5-minute freshness for supply-demand optimization, finance needs per-ride audit detail for 24 months, data science gets daily refresh for model training, and marketing tracks promotion performance.*
>
> *Six data sources feed the system, dominated by GPS telemetry at ~240M daily events (90-day retention) and transactional data from ride, driver, payment, and user services (5-year retention). We must handle GDPR-compliant deletion, PII encryption, and driver bank data masking.*
>
> *The core KPIs are ETA accuracy, driver utilization, rider retention, and gross booking value. Based on these requirements, I'd design three fact tables — an accumulating-snapshot fact for rides, a transaction fact for payments, and a periodic-snapshot fact for daily driver utilization — supported by dimensions for time, city/zone, driver, rider, and ride type. GPS telemetry would be aggregated to zone-level summaries to manage volume."*

> **Interview tip:** This summary does three things simultaneously: it proves you listened, it demonstrates structure, and it previews your design — all before you've drawn a single table.

---

### 5. Requirements → Design Decisions Mapping

Here's how each requirement drives a concrete modeling choice:

| Requirement | Design Decision | Rationale |
|-------------|----------------|-----------|
| 2M rides/day, multiple status changes | **Accumulating snapshot fact** for rides (one row per ride_id, updated on each status change) | A single transaction fact can't hold `request_time`, `accept_time`, `start_time`, `end_time`, and all the derived intervals (wait time, ride duration) |
| GPS: 240M events/day | **Aggregate GPS at zone + hour granularity** before loading into the warehouse | Raw telemetry is too granular for most queries; pre-aggregating cuts volume by ~95% |
| Finance needs per-ride detail | **Separate transaction fact for payments** (grain: one row per charge/line item) | Payments have a different grain (multiple charges per ride) — a conformed dimension on `ride_id` joins them |
| Driver utilization over time | **Periodic snapshot fact** (one row per driver per day) | Tracks driver shifts, active minutes, and rides completed per day — a semi-additive metric |
| Operations needs < 5 min latency | **Separate near-real-time layer** (Kafka → streaming ETL → materialized aggregates) | The daily batch warehouse can't serve operations; a streaming pipeline feeds live dashboards |
| Riders churned in Month 1 | **SCD Type 2 on rider dimension** + periodic snapshot fact for monthly retention | Need to track rider status over time to measure cohort retention accurately |
| GDPR deletion | **Soft-delete flag in rider dimension** + ability to cascade to facts | Hard-delete breaks referential integrity; soft-delete preserves audit trail while honoring deletion requests |
| PII encryption | **Separate rider_PII sub-fact or encrypted columns** in rider dimension | Keeps PII isolated; non-PII fields (rider_id, join_date, city) remain queryable by all consumers |
| 5-year retention, 90-day GPS | **Partition by date** + automated partition drop for GPS; hot/cold tiering for rides | Date partitioning enables efficient pruning, archival, and deletion |
| Marketing: promotion tracking | **Degenerate dimension** `promotion_id` on the rides fact | No separate dimension table needed for simple promotion codes; filters rides by promo |

#### Preview: Fact & Dimension Tables

```
FACT Tables:
├── fact_ride_accumulating_snapshot
│   ├── ride_key (PK, surrogate)
│   ├── date_key (FK → dim_date)           -- date of request
│   ├── driver_key (FK → dim_driver)
│   ├── rider_key (FK → dim_rider)
│   ├── city_zone_key (FK → dim_city_zone)
│   ├── ride_type_key (FK → dim_ride_type)
│   ├── promotion_id (degenerate dim)
│   ├── requested_at / accepted_at / started_at / completed_at
│   ├── estimated_fare / actual_fare / tip / surcharge
│   ├── status
│   └── derived: wait_time, ride_duration, eta_error
│
├── fact_payment_transaction
│   ├── payment_key (PK)
│   ├── date_key (FK → dim_date)
│   ├── ride_key (FK → fact_ride)           -- conformed grain link
│   ├── payment_type (charge, payout, refund)
│   ├── amount, currency
│   └── status
│
└── fact_driver_daily_snapshot
    ├── driver_key (FK → dim_driver)
    ├── date_key (FK → dim_date)
    ├── shift_start / shift_end
    ├── rides_completed
    ├── active_minutes
    └── utilization_rate (computed at query time)

DIMENSION Tables:
├── dim_date           (standard calendar)
├── dim_city_zone      (city + zone + geo bounds)
├── dim_driver         (SCD-2: profile, status changes)
├── dim_rider          (SCD-2: join_date, status, soft_delete_flag)
├── dim_ride_type      (shared, XL, comfort, etc.)
└── dim_weather        (hourly by city zone, 90-day retention)
```

---

### Interviewer Questions You'll Face

| # | Question | What They're Testing |
|---|----------|---------------------|
| 1 | *"How would you handle the GPS telemetry data without overloading the warehouse?"* | Your ability to manage high-volume, high-cardinality data through pre-aggregation and tiered storage |
| 2 | *"Why an accumulating snapshot for rides instead of a transaction fact with one row per status change?"* | Understanding of Kimball patterns and their trade-offs: accumulating captures the full lifecycle in one row; transaction facts lose interval metrics |
| 3 | *"How would you model the fact that a single ride can have multiple payment events (fare, tip, surge, promo refund)?"* | Recognition that different grains require different fact tables, connected by conformed dimensions |
| 4 | *"If operations needs real-time data but finance needs daily accuracy, how do you avoid having two inconsistent datasets?"* | Understanding of medallion architecture: a single source of truth with different serving layers (bronze → silver → gold) at different latencies |

---

### Top-Up Questions (Senior-Level Probes)

| # | Question | What It Demonstrates |
|---|----------|---------------------|
| 1 | *"The GPS telemetry has duplicates and gaps — at what layer would you clean that, and what deduplication strategy?"* | Pipeline architecture awareness: dedup at the bronze/raw layer using event_id + timestamp window; gap-fill at the silver/aggregation layer |
| 2 | *"How would you handle the late-arriving ride completion events that the finance team flagged?"* | Late-data patterns: backfill strategy with watermark-based processing; accumulating snapshot handles this naturally since the row updates in place |
| 3 | *"What happens when a rider deletes their account under GDPR — do you delete from facts, and how does that affect historical metrics?"* | GDPR-aware modeling: soft-delete in dimension; fact rows become orphaned but queryable with a flag; historical aggregates pre-computed before deletion |
| 4 | *"We're adding 15% volume per quarter — at what point does the current architecture break, and what would you build differently?"* | Capacity planning and architectural foresight: identify the breaking point (GPS volume), propose event streaming with incremental materialization, and plan for partition pruning |

---

### Practice Exercise

**Scenario:** You're interviewing for a Senior Data Engineer role at *FoodDash*, a food delivery platform. The interviewer says:

> *"We deliver 500,000 orders per day across 100 cities. We have restaurants, drivers, and customers. We need a data warehouse so our operations team can optimize delivery times, our finance team can track commissions, and our marketing team can measure the impact of discounts. Walk me through the requirements you'd gather."*

**Your task:** Write out:

1. **8–10 questions** you'd ask, organized by phase (Business Context, Consumers, Sources, Metrics, Constraints).
2. **A structured requirements summary** modeled on Section 3 above — at minimum: business objectives, key metrics with formulas, data consumers and latency needs, and data sources with volume estimates.
3. **A 4–5 sentence summary statement** you'd give back to the interviewer, previewing your design.
4. **At least 3 specific design decisions** driven by your gathered requirements (e.g., "I'd use an accumulating snapshot for orders because...").

<details>
<summary>💡 Click to reveal a hint</summary>

Think about these food-delivery-specific challenges:
- **Order lifecycle** is like a ride: placed → assigned → picked up → delivered → rated (use an accumulating snapshot).
- **Driver positioning** between deliveries creates idle-time metrics (periodic snapshot).
- **Restaurant performance** (acceptance rate, prep time) is a separate dimension from driver performance.
- **Commission calculations** involve both the restaurant cut and the delivery fee — potentially two payment lines per order.
- **Delivery time** decomposes into three intervals: assignment-to-pickup + pickup-to-delivery. Can you think of which fact table captures this?

</details>

<details>
<summary>💡 Click to reveal a model answer outline</summary>

**Key metrics to identify:**
- Average delivery time (assignment → pickup + pickup → delivery)
- Order fulfillment rate (delivered / total placed)
- Commission revenue per city per day
- Driver utilization (delivered orders / active hours)
- Restaurant acceptance rate (accepted orders / assigned orders)
- Customer retention (repeat order rate)

**Fact tables you'd propose:**
- `fact_order_accumulating` (one row per order, updated through lifecycle)
- `fact_payment` (one row per payment line: restaurant commission, driver fee, customer charge)
- `fact_driver_daily` (periodic snapshot: orders delivered, miles, active time)
- Dimensions: `dim_date`, `dim_city`, `dim_restaurant` (SCD-2), `dim_driver` (SCD-2), `dim_customer`, `dim_discount`

**Design decision examples:**
1. "I'd use an accumulating snapshot for orders because we need to measure multiple intervals within a single order lifecycle — assignment-to-pickup and pickup-to-delivery — and a transaction fact would require joining multiple rows per order to compute these."
2. "I'd separate payment into its own fact table because each order generates three payment events (customer charge, restaurant commission, driver fee) at different grains, and the finance team needs to reconcile each independently."
3. "I'd use SCD Type 2 on the restaurant dimension because restaurant details (name, address, commission rate) change over time, and finance needs to know which rate applied at order time, not just the current rate."

</details>

---

> **Key takeaway from this lesson:** Requirements gathering is not a checklist — it's a **conversation strategy**. Every question you ask should have a clear purpose: either it uncovers a business constraint, quantifies a scale dimension, or reveals a design tension. The better your questions, the better your model.

---

### Related Lessons

- ⬅️ [Lesson 11: Data Retention and Historical Data Management](./11-data-retention-policies-and-historical-data-management.md) — non-functional requirements context
- ➡️ [Lesson 13: Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md) — translate requirements into visuals
- ➡️ [Lessons 15-20: Practice Scenarios](./15-practice-ecommerce-platform.md) — apply the requirements from this session
