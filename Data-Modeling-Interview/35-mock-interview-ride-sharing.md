# Lesson 35 — Mock Interview: Ride Sharing Platform

**Level:** Senior / Staff / Principal Data Engineer
**Duration:** 60-90 minutes
**Case type:** End-to-end source-to-warehouse data modeling

## How to use this case in an interview

Treat this as a full interview exercise. State assumptions, ask clarifying questions, inventory business processes and source systems, then design the warehouse model. The key is demonstrating first-principles thinking rather than memorizing a "correct" schema.

## Learning Objectives

- Model a complex multi-sided marketplace with multiple event types
- Handle role-playing dimensions, surrogate keys and SCD dimensions
- Design independent business processes with separate fact tables
- Address operational freshness vs. certified analytics
- Handle multi-tenant data and cross-tenant visibility

## Interview Dialogue

**Interviewer:** We're building a data platform for a ride-sharing company like Uber. We need to model everything from driver and rider onboarding through rides, payments and loyalty programs. Design the data warehouse.

**Candidate:** First, I'd identify the core business processes:

1. **Driver & Rider Onboarding** — registration, verification, background checks, document uploads
2. **Ride Lifecycle** — request, matching, acceptance, start, end, cancellation
3. **Payments & Settlements** — ride charges, promotions, tips, driver payouts, platform commissions
4. **Geographic Operations** — surge pricing, zones, time-based availability
5. **Loyalty & Ratings** — points, tier levels, driver/rider ratings

**Interviewer:** Great. Walk me through the source system design and the warehouse model.

**Candidate:** The operational systems likely have:

**Source Systems:**

**Riders Table**
- rider_id, email, phone, first_name, last_name
- registration_date, status, tier_level

**Drivers Table**
- driver_id, email, phone, first_name, last_name
- license_number, vehicle_info, background_check_status
- registration_date, status, tier_level

**Ride Requests Table**
- ride_request_id, rider_id, pickup_lat, pickup_lon, dropoff_lat, dropoff_lon
- requested_at, status (pending, matched, cancelled)

**Ride Events Table**
- event_id, ride_id, event_type (matched, started, ended, cancelled)
- event_timestamp, driver_id, details (JSON)

**Payments Table**
- payment_id, ride_id, rider_id, amount, currency, payment_method
- paid_at, status

**Driver Payouts Table**
- payout_id, driver_id, ride_id, gross_amount, commission, net_amount
- payout_date, status

**Interviewer:** Good start. Now model the warehouse. How would you design the fact and dimension tables?

**Candidate:** I'd model each business process at its natural grain:

**Core Dimensions:**

**Dim_Rider**
- rider_sk (surrogate key), rider_id (natural key), email, phone
- first_name, last_name, registration_date, status
- tier_level, tier_effective_date
- current_flag, start_date, end_date (SCD Type 2 for tier changes)

**Dim_Driver**
- driver_sk (surrogate key), driver_id (natural key), email, phone
- first_name, last_name, license_number, vehicle_type
- registration_date, status, tier_level
- current_flag, start_date, end_date (SCD Type 2 for tier changes)

**Dim_Date** — standard date dimension
**Dim_Time** — granular time dimension for surge zones
**Dim_Location** — zone-based location (using a junk dimension for pickup/dropoff zone combinations)

**Core Fact Tables:**

**Fact_Ride_Events** (transactional fact, one row per event per ride)
- ride_sk, rider_sk, driver_sk, date_sk, time_sk
- event_type_sk, event_timestamp
- pickup_zone_sk, dropoff_zone_sk
- duration_seconds, distance_km
- status, cancellation_reason_sk

**Fact_Payments** (transactional fact, one row per payment)
- payment_sk, ride_sk, rider_sk, driver_sk, date_sk
- gross_amount, commission_amount, net_rider_amount
- payment_method_sk, currency_sk, promo_applied_flag
- tip_amount

**Fact_Driver_Payouts** (transactional fact, one row per payout)
- payout_sk, driver_sk, date_sk
- gross_amount, commission_deduction, net_payout
- payout_method_sk, status_sk

**Fact_Rider_Requests** (transactional fact)
- request_sk, rider_sk, date_sk, time_sk
- pickup_zone_sk, dropoff_zone_sk
- matched_flag, wait_time_seconds, cancelled_flag

**Interviewer:** How do you handle role-playing dimensions?

**Candidate:** In the ride events fact table, the rider and driver play different roles. The same person could be both rider and driver in different rides. We'd use role-playing by having separate foreign keys to the same physical dimension table conceptually, though in practice we create separate dimension tables (Dim_Rider, Dim_Driver) because the attributes and lifecycle are different.

**Interviewer:** What about SCD handling?

**Candidate:** I'd use SCD Type 2 for dimensions where historical tracking matters:
- **Dim_Rider tier level** — track when riders move between loyalty tiers
- **Dim_Driver tier level** — track driver tier changes
- **Dim_Driver vehicle info** — track vehicle changes

For dimensions where only current state matters (e.g., reference data like payment methods), I'd use SCD Type 1.

**Interviewer:** How would you handle operational freshness vs. certified analytics?

**Candidate:** I'd create separate pipelines:
- **Operational pipeline**: near-real-time ingestion (minutes) for dashboards monitoring active rides, driver availability, and immediate operational decisions
- **Certified analytics pipeline**: daily full processing with complete SCD handling, data quality validation, and financial-grade reconciliation for revenue reporting

## Key Takeaways

| Concept | Application in this case |
|---------|--------------------------|
| Multiple business processes | Separate fact tables for rides, payments, payouts |
| SCD Type 2 | Track rider/driver tier changes over time |
| Role-playing dimensions | Different roles (rider vs driver) in same system |
| Event tracking | Fact_Ride_Events captures the full ride lifecycle |
| Financial reconciliation | Fact_Payments + Fact_Driver_Payouts must net to platform revenue |
| Operational freshness | Minutes-level for ops dashboards vs daily for certified analytics |
| Surrogate keys | Required for SCD Type 2 and handling natural key reuse |
| Junk dimensions | Combine sparse flags (matched, cancelled, promo applied) |
| Late-arriving facts | Handle delayed event data from mobile clients |
| Multi-tenant operations | Geographic zones with time-based surge pricing |

## Interview Scoring Rubric

| Level | What to look for |
|-------|------------------|
| **Junior** | Can identify basic fact and dimension tables |
| **Senior** | Separates business processes correctly, uses SCD Type 2, understands grain |
| **Staff** | Handles role-playing, operational vs certified pipelines, financial reconciliation |
| **Principal** | Addresses late-arriving facts, cross-system identity resolution, data governance |

### Related Lessons

- [Lesson 3](./03-identifying-the-right-business-processes.md) — Identifying business processes
- [Lesson 4](./04-determining-grain-of-fact-table.md) — Determining grain of fact tables
- [Lesson 21](./21-slowly-changing-dimensions-types-1-4.md) — SCD Types 1-4
- [Lesson 24](./24-fact-table-types.md) — Fact table types
- [Lesson 25](./25-additive-semi-additive-non-additive-measures.md) — Additive, semi-additive, non-additive measures
- [Lesson 30](./30-aggregates-and-materialized-views.md) — Aggregates and materialized views
